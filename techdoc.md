# Vesper Analytics Platform - Technical Design Document

**Document Version:** 1.0
**Date:** 2026-02-20
**Classification:** Internal - Confidential
**Author:** Engineering Team
**Audience:** CTO, Engineering Leadership

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Architecture Overview](#2-system-architecture-overview)
3. [Multi-Tenant Architecture](#3-multi-tenant-architecture)
4. [Database Design](#4-database-design)
5. [User Management & RBAC System](#5-user-management--rbac-system)
6. [Authentication & Identity](#6-authentication--identity)
7. [Backend Architecture (FastAPI)](#7-backend-architecture-fastapi)
8. [Frontend Architecture (React)](#8-frontend-architecture-react)
9. [Caching, Messaging & Broker Architecture](#9-caching-messaging--broker-architecture)
10. [Security Architecture](#10-security-architecture)
11. [Azure Deployment & Infrastructure](#11-azure-deployment--infrastructure)
12. [API Design & Versioning](#12-api-design--versioning)
13. [Appendix: Entity Relationship Diagrams](#13-appendix-entity-relationship-diagrams)

---

## 1. Executive Summary

Vesper Analytics is a **multi-tenant SaaS platform** enabling organizations to launch branded analytics portals under their own custom domains. The platform provides a sophisticated, **industry-grade dynamic user management system** with granular role-based access control (RBAC) that governs visibility and actions down to the page, section, and individual UI control level.

### Key Design Principles

| Principle | Description |
|-----------|-------------|
| **Tenant Isolation** | Single database, logical isolation via `tenant_id` foreign keys with Row-Level Security (RLS) |
| **Zero-Trust Permissions** | UI is fully server-driven; every page, section, and action is authorized by the backend |
| **Identity Federation** | Microsoft Entra ID (OIDC) integration with automatic user provisioning and group sync |
| **Configuration over Code** | Permissions, themes, branding, and features are configurable per tenant without deployments |
| **Cloud-Native** | Containerized microservices on Azure with auto-scaling, Redis caching, and event-driven messaging |

### Technology Stack

| Layer | Technology | Azure Service |
|-------|-----------|---------------|
| Frontend | React 18+ (TypeScript) | Azure Static Web Apps |
| Backend API | FastAPI (Python 3.12+) | Azure Container Apps |
| Database | SQL Server | Azure SQL Database |
| Cache | Redis 7+ | Azure Cache for Redis |
| Message Broker | RabbitMQ / Kafka | Azure Service Bus / Event Hubs |
| Identity Provider | Microsoft Entra ID | Azure Active Directory |
| CDN/DNS | - | Azure Front Door + DNS Zones |
| Secrets | - | Azure Key Vault |
| Monitoring | - | Azure Monitor + Application Insights |

---

## 2. System Architecture Overview

```
                         ┌─────────────────────────────────────────────┐
                         │           Azure Front Door (CDN + WAF)      │
                         │    ┌──────────────────────────────────────┐  │
                         │    │  Custom Domain Routing & SSL Offload │  │
                         │    └──────────┬───────────┬───────────────┘  │
                         └───────────────┼───────────┼─────────────────┘
                                         │           │
                              ┌──────────▼──┐   ┌───▼──────────────┐
                              │ Static Web  │   │  Container Apps  │
                              │   App       │   │  (API Gateway)   │
                              │  (React)    │   │                  │
                              └─────────────┘   └───┬──────────────┘
                                                    │
                         ┌──────────────────────────┼──────────────────────┐
                         │          Internal Virtual Network (VNET)        │
                         │                          │                      │
                         │    ┌─────────────────────┼──────────────────┐   │
                         │    │         Container Apps Environment      │   │
                         │    │                     │                   │   │
                         │    │  ┌──────────┐ ┌─────▼─────┐ ┌────────┐ │   │
                         │    │  │ Auth     │ │ Core API  │ │ Tenant │ │   │
                         │    │  │ Service  │ │ Service   │ │ Mgmt   │ │   │
                         │    │  └────┬─────┘ └─────┬─────┘ └───┬────┘ │   │
                         │    │       │             │           │      │   │
                         │    │  ┌────▼─────┐ ┌─────▼─────┐ ┌──▼────┐ │   │
                         │    │  │ User     │ │ Analytics │ │ Notif │ │   │
                         │    │  │ Mgmt     │ │ Engine    │ │ Svc   │ │   │
                         │    │  └──────────┘ └───────────┘ └───────┘ │   │
                         │    └────────────────────────────────────────┘   │
                         │                     │                          │
                         │    ┌────────────────┼──────────────────────┐   │
                         │    │                │                      │   │
                         │    │  ┌─────────┐ ┌─▼───────┐ ┌─────────┐ │   │
                         │    │  │ Azure   │ │ Azure   │ │ Azure   │ │   │
                         │    │  │ SQL DB  │ │ Redis   │ │ Service │ │   │
                         │    │  │         │ │ Cache   │ │ Bus     │ │   │
                         │    │  └─────────┘ └─────────┘ └─────────┘ │   │
                         │    └───────────────────────────────────────┘   │
                         └────────────────────────────────────────────────┘
```

### Request Flow

```
User (tenant-a.vesper.com)
  │
  ▼
Azure Front Door ─── Resolves custom domain ──► Tenant Routing
  │
  ├─► Static Web App (React SPA)
  │     │
  │     └─► Fetches /api/tenant/resolve?domain=tenant-a.vesper.com
  │           │
  │           └─► Returns: tenant config, theme, branding, auth method
  │
  └─► API Requests (Bearer Token)
        │
        ▼
  API Gateway (Container App)
        │
        ├─► JWT Validation + Tenant Context Extraction
        ├─► Permission Middleware (RBAC check)
        ├─► Rate Limiting (Redis-backed)
        └─► Route to Microservice
```

---

## 3. Multi-Tenant Architecture

### 3.1 Tenant Resolution Strategy

Each tenant is assigned one or more custom domains. When a user visits a tenant portal, the system resolves the tenant through a **Domain Lookup Chain**:

```
1. Browser → https://analytics.acme-corp.com
2. DNS CNAME → vesper-platform.azurefd.net (Azure Front Door)
3. Front Door routes to Static Web App
4. React app calls: GET /api/v1/tenants/resolve?domain=analytics.acme-corp.com
5. Backend looks up domain in `tenant_domains` table
6. Returns tenant configuration (theme, auth config, features, branding)
7. React app initializes with tenant context
```

### 3.2 Tenant Configuration Model

Each tenant has a comprehensive configuration that drives the entire UX:

```json
{
  "tenant_id": "uuid",
  "tenant_slug": "acme-corp",
  "display_name": "Acme Corporation",
  "domains": ["analytics.acme-corp.com", "acme.vesper-analytics.com"],
  "auth_config": {
    "allowed_methods": ["entra", "form"],    // Tenant chooses: form, entra, or both
    "entra_tenant_id": "azure-ad-tenant-id",
    "entra_client_id": "app-registration-client-id",
    "enforce_mfa": true,
    "session_timeout_minutes": 480,
    "password_policy": { "min_length": 12, "require_special": true }
  },
  "branding": {
    "logo_url": "https://cdn.vesper.com/tenants/acme/logo.svg",
    "favicon_url": "https://cdn.vesper.com/tenants/acme/favicon.ico",
    "primary_color": "#1A73E8",
    "secondary_color": "#F4F6F9",
    "font_family": "Inter, sans-serif",
    "login_background_url": "...",
    "custom_css": "..."
  },
  "features": {
    "analytics_module": true,
    "reporting_module": true,
    "export_enabled": true,
    "max_users": 500
  },
  "status": "active"
}
```

### 3.3 Logical Tenant Isolation (Single Database)

All tables carry a `tenant_id` column. Isolation is enforced at three levels:

| Level | Mechanism |
|-------|-----------|
| **Database** | SQL Row-Level Security (RLS) policies filter by `tenant_id` using session context |
| **API** | FastAPI middleware injects `tenant_id` from JWT into every DB session |
| **Application** | Repository pattern always filters by tenant; no raw SQL escapes tenant boundary |

#### SQL Row-Level Security Implementation

```sql
-- Enable RLS on every tenant-scoped table
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Create security policy
CREATE SECURITY POLICY tenant_filter
  ADD FILTER PREDICATE dbo.fn_tenant_access_predicate(tenant_id) ON dbo.users,
  ADD BLOCK PREDICATE dbo.fn_tenant_access_predicate(tenant_id) ON dbo.users;

-- Predicate function
CREATE FUNCTION dbo.fn_tenant_access_predicate(@tenant_id UNIQUEIDENTIFIER)
RETURNS TABLE WITH SCHEMABINDING
AS RETURN
  SELECT 1 AS result
  WHERE @tenant_id = CAST(SESSION_CONTEXT(N'tenant_id') AS UNIQUEIDENTIFIER)
     OR CAST(SESSION_CONTEXT(N'is_super_admin') AS BIT) = 1;
```

---

## 4. Database Design

### 4.1 Complete Entity Relationship Model

The database is organized into five logical domains:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          TENANT MANAGEMENT DOMAIN                          │
│                                                                            │
│  ┌──────────────┐     ┌──────────────────┐     ┌────────────────────────┐  │
│  │   tenants    │────▶│  tenant_domains  │     │  tenant_configurations │  │
│  └──────┬───────┘     └──────────────────┘     └────────────────────────┘  │
│         │                                                                  │
└─────────┼──────────────────────────────────────────────────────────────────┘
          │
          │ tenant_id (FK on all tables below)
          │
┌─────────▼──────────────────────────────────────────────────────────────────┐
│                        USER MANAGEMENT DOMAIN                              │
│                                                                            │
│  ┌──────────────┐     ┌──────────────────┐     ┌────────────────────────┐  │
│  │    users     │────▶│ user_group_map   │◀────│       groups          │  │
│  └──────┬───────┘     └──────────────────┘     └────────┬───────────────┘  │
│         │                                               │                  │
│         │         ┌──────────────────────┐              │                  │
│         └────────▶│  user_role_map      │              │                  │
│                   └──────────┬───────────┘              │                  │
│                              │                          │                  │
│                   ┌──────────▼───────────┐              │                  │
│                   │  group_role_map      │◀─────────────┘                  │
│                   └──────────┬───────────┘                                 │
│                              │                                             │
└──────────────────────────────┼─────────────────────────────────────────────┘
                               │
┌──────────────────────────────▼─────────────────────────────────────────────┐
│                          RBAC / PERMISSION DOMAIN                          │
│                                                                            │
│                   ┌──────────────────────┐                                 │
│                   │       roles          │                                 │
│                   └──────────┬───────────┘                                 │
│                              │                                             │
│                   ┌──────────▼───────────┐     ┌────────────────────────┐  │
│                   │ role_permission_map  │────▶│     permissions       │  │
│                   └──────────────────────┘     └────────┬───────────────┘  │
│                                                         │                  │
│                                               ┌─────────▼──────────────┐   │
│                                               │   permission_scopes   │   │
│                                               │  (page / section /    │   │
│                                               │   action granularity) │   │
│                                               └────────────────────────┘  │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
                               │
┌──────────────────────────────▼─────────────────────────────────────────────┐
│                        IDENTITY FEDERATION DOMAIN                          │
│                                                                            │
│  ┌──────────────────────┐     ┌──────────────────────────────────────────┐ │
│  │  entra_connections   │     │  entra_group_mappings                   │ │
│  │  (OIDC config/tenant)│     │  (Entra Group ↔ Internal Group)        │ │
│  └──────────────────────┘     └──────────────────────────────────────────┘ │
│                                                                            │
│  ┌──────────────────────┐     ┌──────────────────────────────────────────┐ │
│  │  user_identities     │     │  sync_audit_log                        │ │
│  │  (external ID map)   │     │  (Entra sync history)                  │ │
│  └──────────────────────┘     └──────────────────────────────────────────┘ │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
                               │
┌──────────────────────────────▼─────────────────────────────────────────────┐
│                          AUDIT & SYSTEM DOMAIN                             │
│                                                                            │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐  │
│  │  audit_logs      │  │  login_history   │  │  api_rate_limits        │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────────────┘  │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Complete Table Definitions

#### 4.2.1 Tenant Management Domain

```sql
-- ═══════════════════════════════════════════════════════════════
-- TENANT MANAGEMENT
-- ═══════════════════════════════════════════════════════════════

CREATE TABLE tenants (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    slug                VARCHAR(63)      NOT NULL UNIQUE,       -- URL-safe identifier
    display_name        NVARCHAR(255)    NOT NULL,
    status              VARCHAR(20)      NOT NULL DEFAULT 'active'
                          CHECK (status IN ('active','suspended','provisioning','deactivated')),
    subscription_tier   VARCHAR(20)      NOT NULL DEFAULT 'standard'
                          CHECK (subscription_tier IN ('free','standard','professional','enterprise')),
    max_users           INT              NOT NULL DEFAULT 50,
    auth_methods        VARCHAR(50)      NOT NULL DEFAULT 'form'
                          CHECK (auth_methods IN ('form','entra','both')),
    created_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    updated_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    created_by          UNIQUEIDENTIFIER NULL
);

CREATE TABLE tenant_domains (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    tenant_id           UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    domain              VARCHAR(255)     NOT NULL UNIQUE,       -- e.g. analytics.acme.com
    is_primary          BIT              NOT NULL DEFAULT 0,
    ssl_status          VARCHAR(20)      NOT NULL DEFAULT 'pending'
                          CHECK (ssl_status IN ('pending','provisioning','active','failed')),
    verified_at         DATETIME2        NULL,
    created_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME()
);
CREATE UNIQUE INDEX uq_tenant_primary_domain
    ON tenant_domains(tenant_id) WHERE is_primary = 1;

CREATE TABLE tenant_configurations (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    tenant_id           UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    config_key          VARCHAR(100)     NOT NULL,              -- e.g. 'branding.logo_url'
    config_value        NVARCHAR(MAX)    NOT NULL,              -- JSON or scalar
    config_type         VARCHAR(20)      NOT NULL DEFAULT 'string'
                          CHECK (config_type IN ('string','json','boolean','integer')),
    updated_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    updated_by          UNIQUEIDENTIFIER NULL,
    CONSTRAINT uq_tenant_config UNIQUE (tenant_id, config_key)
);

CREATE TABLE tenant_themes (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    tenant_id           UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    theme_name          VARCHAR(50)      NOT NULL DEFAULT 'default',
    primary_color       VARCHAR(9)       NOT NULL DEFAULT '#1A73E8',
    secondary_color     VARCHAR(9)       NOT NULL DEFAULT '#F4F6F9',
    accent_color        VARCHAR(9)       NULL,
    background_color    VARCHAR(9)       NULL,
    text_color          VARCHAR(9)       NULL,
    font_family         VARCHAR(100)     NOT NULL DEFAULT 'Inter, sans-serif',
    logo_url            VARCHAR(500)     NULL,
    favicon_url         VARCHAR(500)     NULL,
    custom_css          NVARCHAR(MAX)    NULL,                  -- Tenant-specific CSS overrides
    component_overrides NVARCHAR(MAX)    NULL,                  -- JSON: per-component style tokens
    is_active           BIT              NOT NULL DEFAULT 1,
    created_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT uq_tenant_theme UNIQUE (tenant_id, theme_name)
);
```

#### 4.2.2 User Management Domain

```sql
-- ═══════════════════════════════════════════════════════════════
-- USER MANAGEMENT
-- ═══════════════════════════════════════════════════════════════

CREATE TABLE users (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    tenant_id           UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    email               VARCHAR(320)     NOT NULL,
    display_name        NVARCHAR(255)    NOT NULL,
    first_name          NVARCHAR(100)    NULL,
    last_name           NVARCHAR(100)    NULL,
    password_hash       VARCHAR(255)     NULL,                  -- NULL for Entra-only users
    avatar_url          VARCHAR(500)     NULL,
    phone               VARCHAR(20)      NULL,
    status              VARCHAR(20)      NOT NULL DEFAULT 'active'
                          CHECK (status IN ('active','inactive','locked','pending_activation')),
    user_type           VARCHAR(20)      NOT NULL DEFAULT 'standard'
                          CHECK (user_type IN ('super_admin','tenant_admin','standard')),
    auth_provider       VARCHAR(20)      NOT NULL DEFAULT 'form'
                          CHECK (auth_provider IN ('form','entra','both')),
    mfa_enabled         BIT              NOT NULL DEFAULT 0,
    mfa_secret          VARCHAR(255)     NULL,
    last_login_at       DATETIME2        NULL,
    password_changed_at DATETIME2        NULL,
    failed_login_count  INT              NOT NULL DEFAULT 0,
    locked_until        DATETIME2        NULL,
    created_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    updated_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    created_by          UNIQUEIDENTIFIER NULL,
    CONSTRAINT uq_user_email_tenant UNIQUE (tenant_id, email)
);

-- Super admins: tenant_id is a special 'platform' tenant UUID
-- They have cross-tenant access enforced at the application layer

CREATE TABLE groups (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    tenant_id           UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    name                NVARCHAR(100)    NOT NULL,
    description         NVARCHAR(500)    NULL,
    group_type          VARCHAR(20)      NOT NULL DEFAULT 'custom'
                          CHECK (group_type IN ('custom','entra_synced','system')),
    is_default          BIT              NOT NULL DEFAULT 0,    -- Auto-assign new users
    status              VARCHAR(20)      NOT NULL DEFAULT 'active',
    created_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    updated_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    created_by          UNIQUEIDENTIFIER NULL,
    CONSTRAINT uq_group_name_tenant UNIQUE (tenant_id, name)
);

CREATE TABLE user_group_map (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    user_id             UNIQUEIDENTIFIER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    group_id            UNIQUEIDENTIFIER NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
    assigned_at         DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    assigned_by         UNIQUEIDENTIFIER NULL,
    source              VARCHAR(20)      NOT NULL DEFAULT 'manual'
                          CHECK (source IN ('manual','entra_sync','api','system')),
    CONSTRAINT uq_user_group UNIQUE (user_id, group_id)
);
```

#### 4.2.3 RBAC / Permission Domain (Core Design)

```sql
-- ═══════════════════════════════════════════════════════════════
-- RBAC SYSTEM - THE HEART OF THE PERMISSION ENGINE
-- ═══════════════════════════════════════════════════════════════

-- ROLES: Defined per tenant (tenant admins create/manage roles)
CREATE TABLE roles (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    tenant_id           UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    name                NVARCHAR(100)    NOT NULL,
    description         NVARCHAR(500)    NULL,
    is_system_role      BIT              NOT NULL DEFAULT 0,    -- Cannot be deleted
    priority            INT              NOT NULL DEFAULT 0,    -- Higher = more authority
    status              VARCHAR(20)      NOT NULL DEFAULT 'active',
    created_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    updated_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    created_by          UNIQUEIDENTIFIER NULL,
    CONSTRAINT uq_role_name_tenant UNIQUE (tenant_id, name)
);

-- RESOURCES: Represent pages/modules in the UI (global registry)
CREATE TABLE resources (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    resource_key        VARCHAR(100)     NOT NULL UNIQUE,       -- e.g. 'dashboard', 'reports.sales'
    display_name        NVARCHAR(200)    NOT NULL,
    resource_type       VARCHAR(20)      NOT NULL
                          CHECK (resource_type IN ('page','section','widget','api_endpoint')),
    parent_resource_id  UNIQUEIDENTIFIER NULL REFERENCES resources(id),  -- Hierarchical
    module              VARCHAR(50)      NOT NULL,              -- Logical module grouping
    sort_order          INT              NOT NULL DEFAULT 0,
    metadata            NVARCHAR(MAX)    NULL,                  -- JSON: icon, route, etc.
    is_active           BIT              NOT NULL DEFAULT 1,
    created_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME()
);
-- Example resources:
-- 'dashboard'                    (page)
-- 'dashboard.revenue_chart'      (section)
-- 'dashboard.export_button'      (action within section)
-- 'users'                        (page)
-- 'users.create_button'          (action)
-- 'users.bulk_import'            (action)

-- ACTIONS: What can be done on a resource
CREATE TABLE actions (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    action_key          VARCHAR(50)      NOT NULL UNIQUE,       -- e.g. 'view','create','edit','delete','export'
    display_name        NVARCHAR(100)    NOT NULL,
    description         NVARCHAR(300)    NULL,
    category            VARCHAR(30)      NOT NULL DEFAULT 'general'
                          CHECK (category IN ('general','data','admin','system')),
    created_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME()
);
-- Standard actions: view, create, edit, delete, export, import, approve, configure

-- PERMISSIONS: The intersection of Resource + Action
CREATE TABLE permissions (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    resource_id         UNIQUEIDENTIFIER NOT NULL REFERENCES resources(id),
    action_id           UNIQUEIDENTIFIER NOT NULL REFERENCES actions(id),
    description         NVARCHAR(300)    NULL,
    is_active           BIT              NOT NULL DEFAULT 1,
    created_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT uq_permission UNIQUE (resource_id, action_id)
);
-- Example: (resource='dashboard', action='view') = "Can view the dashboard page"
-- Example: (resource='dashboard.revenue_chart', action='view') = "Can see the revenue chart"
-- Example: (resource='users', action='create') = "Can create users"
-- Example: (resource='reports.sales', action='export') = "Can export sales reports"

-- ROLE-PERMISSION MAP: Which permissions each role has
CREATE TABLE role_permission_map (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    role_id             UNIQUEIDENTIFIER NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id       UNIQUEIDENTIFIER NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    grant_type          VARCHAR(10)      NOT NULL DEFAULT 'allow'
                          CHECK (grant_type IN ('allow','deny')),    -- Explicit deny overrides allow
    conditions          NVARCHAR(MAX)    NULL,                       -- JSON: conditional access rules
    assigned_at         DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    assigned_by         UNIQUEIDENTIFIER NULL,
    CONSTRAINT uq_role_permission UNIQUE (role_id, permission_id)
);

-- USER → ROLE direct mapping (user-level role assignment)
CREATE TABLE user_role_map (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    user_id             UNIQUEIDENTIFIER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id             UNIQUEIDENTIFIER NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    assigned_at         DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    assigned_by         UNIQUEIDENTIFIER NULL,
    expires_at          DATETIME2        NULL,                       -- Time-bound role assignments
    CONSTRAINT uq_user_role UNIQUE (user_id, role_id)
);

-- GROUP → ROLE mapping (group-level role assignment)
CREATE TABLE group_role_map (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    group_id            UNIQUEIDENTIFIER NOT NULL REFERENCES groups(id) ON DELETE CASCADE,
    role_id             UNIQUEIDENTIFIER NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    assigned_at         DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    assigned_by         UNIQUEIDENTIFIER NULL,
    CONSTRAINT uq_group_role UNIQUE (group_id, role_id)
);
```

#### 4.2.4 Permission Resolution Logic

The effective permissions for a user are computed by merging direct and group-inherited permissions:

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    PERMISSION RESOLUTION ALGORITHM                       │
│                                                                          │
│  Input: user_id, resource_key, action_key                                │
│                                                                          │
│  Step 1: Collect all roles for the user                                  │
│     direct_roles  = SELECT role_id FROM user_role_map                    │
│                     WHERE user_id = ? AND (expires_at IS NULL            │
│                            OR expires_at > GETUTCDATE())                 │
│                                                                          │
│     group_roles   = SELECT grm.role_id FROM group_role_map grm           │
│                     JOIN user_group_map ugm ON ugm.group_id = grm.group_id│
│                     WHERE ugm.user_id = ?                                │
│                                                                          │
│     all_roles     = direct_roles UNION group_roles                       │
│                                                                          │
│  Step 2: Find matching permission                                        │
│     permission_id = SELECT id FROM permissions p                         │
│                     JOIN resources r ON r.id = p.resource_id             │
│                     JOIN actions a ON a.id = p.action_id                 │
│                     WHERE r.resource_key = ? AND a.action_key = ?        │
│                                                                          │
│  Step 3: Check role-permission grants                                    │
│     grants = SELECT grant_type FROM role_permission_map                  │
│              WHERE role_id IN (all_roles)                                │
│              AND permission_id = ?                                       │
│                                                                          │
│  Step 4: Apply precedence rules                                          │
│     IF any grant has grant_type = 'deny'  → DENIED                      │
│     ELIF any grant has grant_type = 'allow' → ALLOWED                   │
│     ELSE → DENIED (default deny)                                         │
│                                                                          │
│  Step 5: Check hierarchical inheritance (parent resources)               │
│     IF denied AND resource has parent_resource_id:                       │
│       Recurse with parent resource (page-level grants cascade down)      │
│                                                                          │
│  Output: { allowed: boolean, source: 'direct'|'group', role: string }   │
└───────────────────────────────────────────────────────────────────────────┘
```

#### 4.2.5 Identity Federation Domain (Microsoft Entra ID)

```sql
-- ═══════════════════════════════════════════════════════════════
-- MICROSOFT ENTRA ID INTEGRATION
-- ═══════════════════════════════════════════════════════════════

CREATE TABLE entra_connections (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    tenant_id           UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    entra_tenant_id     VARCHAR(100)     NOT NULL,              -- Azure AD Tenant ID
    client_id           VARCHAR(100)     NOT NULL,              -- App Registration Client ID
    client_secret_ref   VARCHAR(200)     NOT NULL,              -- Key Vault reference (never raw secret)
    authority_url       VARCHAR(500)     NOT NULL,
    scopes              VARCHAR(500)     NOT NULL DEFAULT 'openid profile email',
    auto_provision      BIT              NOT NULL DEFAULT 1,    -- Auto-create users on first login
    auto_sync_groups    BIT              NOT NULL DEFAULT 1,    -- Sync Entra groups periodically
    sync_interval_min   INT              NOT NULL DEFAULT 30,
    last_sync_at        DATETIME2        NULL,
    status              VARCHAR(20)      NOT NULL DEFAULT 'active',
    created_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT uq_tenant_entra UNIQUE (tenant_id)
);

-- Maps an Entra group to an internal group
CREATE TABLE entra_group_mappings (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    tenant_id           UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    entra_group_id      VARCHAR(100)     NOT NULL,              -- Azure AD Group Object ID
    entra_group_name    NVARCHAR(255)    NOT NULL,              -- Display name (for admin UI)
    internal_group_id   UNIQUEIDENTIFIER NOT NULL REFERENCES groups(id),
    sync_direction      VARCHAR(20)      NOT NULL DEFAULT 'entra_to_internal'
                          CHECK (sync_direction IN ('entra_to_internal','bidirectional')),
    auto_remove         BIT              NOT NULL DEFAULT 1,    -- Remove from group if removed in Entra
    last_synced_at      DATETIME2        NULL,
    created_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT uq_entra_group_map UNIQUE (tenant_id, entra_group_id)
);

-- Links internal user to external identity (Entra object ID)
CREATE TABLE user_identities (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    user_id             UNIQUEIDENTIFIER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider            VARCHAR(20)      NOT NULL DEFAULT 'entra'
                          CHECK (provider IN ('entra','google','saml')),
    external_id         VARCHAR(255)     NOT NULL,              -- Entra Object ID / sub claim
    external_email      VARCHAR(320)     NULL,
    external_metadata   NVARCHAR(MAX)    NULL,                  -- Raw claims JSON
    linked_at           DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME(),
    last_used_at        DATETIME2        NULL,
    CONSTRAINT uq_user_identity UNIQUE (user_id, provider),
    CONSTRAINT uq_external_id UNIQUE (provider, external_id)
);

CREATE TABLE sync_audit_log (
    id                  BIGINT IDENTITY(1,1) PRIMARY KEY,
    tenant_id           UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    sync_type           VARCHAR(30)      NOT NULL,              -- 'group_sync','user_provision'
    source              VARCHAR(20)      NOT NULL,              -- 'entra','manual'
    action_performed    VARCHAR(30)      NOT NULL,              -- 'user_created','group_mapped','user_removed'
    target_entity_type  VARCHAR(20)      NOT NULL,              -- 'user','group'
    target_entity_id    UNIQUEIDENTIFIER NOT NULL,
    details             NVARCHAR(MAX)    NULL,
    performed_at        DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME()
);
```

#### 4.2.6 Audit & System Domain

```sql
-- ═══════════════════════════════════════════════════════════════
-- AUDIT & SYSTEM
-- ═══════════════════════════════════════════════════════════════

CREATE TABLE audit_logs (
    id                  BIGINT IDENTITY(1,1) PRIMARY KEY,
    tenant_id           UNIQUEIDENTIFIER NOT NULL,
    user_id             UNIQUEIDENTIFIER NULL,
    action              VARCHAR(50)      NOT NULL,              -- 'user.create','role.assign','login.success'
    entity_type         VARCHAR(50)      NOT NULL,              -- 'user','role','group','tenant'
    entity_id           UNIQUEIDENTIFIER NULL,
    old_values          NVARCHAR(MAX)    NULL,                  -- JSON snapshot before change
    new_values          NVARCHAR(MAX)    NULL,                  -- JSON snapshot after change
    ip_address          VARCHAR(45)      NULL,
    user_agent          VARCHAR(500)     NULL,
    created_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME()
);
CREATE INDEX ix_audit_tenant_date ON audit_logs(tenant_id, created_at DESC);
CREATE INDEX ix_audit_user_date ON audit_logs(user_id, created_at DESC);

CREATE TABLE login_history (
    id                  BIGINT IDENTITY(1,1) PRIMARY KEY,
    tenant_id           UNIQUEIDENTIFIER NOT NULL,
    user_id             UNIQUEIDENTIFIER NULL,
    email               VARCHAR(320)     NOT NULL,
    login_method        VARCHAR(20)      NOT NULL,              -- 'form','entra','api_key'
    status              VARCHAR(20)      NOT NULL,              -- 'success','failed','locked','mfa_required'
    failure_reason      VARCHAR(100)     NULL,
    ip_address          VARCHAR(45)      NULL,
    user_agent          VARCHAR(500)     NULL,
    session_id          VARCHAR(100)     NULL,
    created_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME()
);
CREATE INDEX ix_login_history_tenant ON login_history(tenant_id, created_at DESC);

CREATE TABLE sessions (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    user_id             UNIQUEIDENTIFIER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    tenant_id           UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    refresh_token_hash  VARCHAR(255)     NOT NULL,
    device_info         NVARCHAR(500)    NULL,
    ip_address          VARCHAR(45)      NULL,
    expires_at          DATETIME2        NOT NULL,
    revoked             BIT              NOT NULL DEFAULT 0,
    created_at          DATETIME2        NOT NULL DEFAULT SYSUTCDATETIME()
);
```

### 4.3 Key Database Indexes for Performance

```sql
-- High-frequency permission resolution query
CREATE INDEX ix_user_role_map_user ON user_role_map(user_id) INCLUDE (role_id, expires_at);
CREATE INDEX ix_group_role_map_group ON group_role_map(group_id) INCLUDE (role_id);
CREATE INDEX ix_user_group_map_user ON user_group_map(user_id) INCLUDE (group_id);
CREATE INDEX ix_role_perm_map_role ON role_permission_map(role_id) INCLUDE (permission_id, grant_type);
CREATE INDEX ix_permissions_resource ON permissions(resource_id) INCLUDE (action_id);
CREATE INDEX ix_resources_key ON resources(resource_key) INCLUDE (parent_resource_id);

-- Tenant resolution (every single request)
CREATE INDEX ix_tenant_domains_domain ON tenant_domains(domain) INCLUDE (tenant_id);

-- User lookups
CREATE INDEX ix_users_email ON users(email) INCLUDE (tenant_id, status, password_hash);
CREATE INDEX ix_users_tenant ON users(tenant_id) INCLUDE (status);
```

---

## 5. User Management & RBAC System

### 5.1 User Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           USER HIERARCHY                                    │
│                                                                            │
│  ┌─────────────┐                                                           │
│  │ SUPER ADMIN │  Platform-level user (special 'platform' tenant)          │
│  │             │  • Full access to ALL tenants                             │
│  │             │  • Manage tenant lifecycle (create/suspend/delete)        │
│  │             │  • View cross-tenant analytics & audit logs               │
│  │             │  • Impersonate any tenant admin                            │
│  └──────┬──────┘                                                           │
│         │ manages                                                          │
│  ┌──────▼──────┐                                                           │
│  │TENANT ADMIN │  Tenant-scoped administrator                              │
│  │             │  • Full access within their tenant                        │
│  │             │  • Create/manage users, groups, roles                     │
│  │             │  • Configure permissions matrix                           │
│  │             │  • Manage Entra ID integration                            │
│  │             │  • Customize branding & theme                             │
│  └──────┬──────┘                                                           │
│         │ assigns                                                          │
│  ┌──────▼──────┐   ┌──────────────┐                                        │
│  │  STANDARD   │──▶│   GROUPS     │  Logical grouping of users             │
│  │   USERS     │   │              │  • Department-based groups             │
│  │             │   │  ┌─────────┐ │  • Entra-synced groups                 │
│  │             │   │  │ Roles   │ │  • Custom groups                       │
│  │             │   │  └─────────┘ │                                        │
│  └─────────────┘   └──────────────┘                                        │
│                                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Permission Assignment Flow

```
    ┌─────────┐         ┌─────────────┐         ┌──────────┐
    │  User   │────────▶│  User-Role  │────────▶│   Role   │
    │ (Alice) │         │    Map      │         │(Analyst) │
    └────┬────┘         └─────────────┘         └────┬─────┘
         │                                           │
         │  member of                                │ has permissions
         ▼                                           ▼
    ┌─────────┐         ┌─────────────┐    ┌──────────────────────┐
    │  Group  │────────▶│ Group-Role  │    │  Role-Permission Map │
    │(Finance)│         │    Map      │    │                      │
    └─────────┘         └──────┬──────┘    │  Role: Analyst       │
                               │           │  ├─ dashboard:view   │
                               ▼           │  ├─ reports:view     │
                          ┌──────────┐     │  ├─ reports:export   │
                          │   Role   │     │  └─ users:view       │
                          │(Viewer)  │     │                      │
                          └──────────┘     │  Role: Viewer        │
                                           │  ├─ dashboard:view   │
                                           │  └─ reports:view     │
                                           └──────────────────────┘

    Alice's EFFECTIVE permissions = Union(Analyst perms, Viewer perms)
                                  minus any explicit DENY grants
```

### 5.3 Permission Granularity Levels

The system supports **three levels of UI permission granularity**, all configurable per role:

| Level | Resource Type | Example | UI Effect |
|-------|--------------|---------|-----------|
| **Page** | `page` | `reports` | Show/hide entire navigation menu item and page route |
| **Section** | `section` | `reports.filters_panel` | Show/hide a section/card within a page |
| **Action** | `widget` | `reports.export_button` | Enable/disable specific buttons, controls, fields |

#### Example: Full Permission Matrix for "Reports" Page

| Resource Key | Type | Actions | Analyst | Viewer | Manager |
|-------------|------|---------|---------|--------|---------|
| `reports` | page | view | allow | allow | allow |
| `reports.filters_panel` | section | view | allow | deny | allow |
| `reports.data_grid` | section | view | allow | allow | allow |
| `reports.data_grid` | section | edit | deny | deny | allow |
| `reports.export_button` | widget | view | allow | deny | allow |
| `reports.delete_button` | widget | view | deny | deny | allow |
| `reports.share_button` | widget | view | allow | deny | allow |
| `reports.bulk_actions` | widget | view | deny | deny | allow |

### 5.4 User Provisioning Workflows

#### 5.4.1 Manual User Creation (Form Auth)

```
Tenant Admin
    │
    ▼
POST /api/v1/users
    │
    ├── Validate: email unique in tenant
    ├── Create user record (status: pending_activation)
    ├── Generate activation token
    ├── Assign to default group (if configured)
    ├── Send activation email
    └── Audit log: 'user.create'
    │
    ▼
User clicks activation link
    │
    ├── Set password
    ├── Status → active
    └── Audit log: 'user.activated'
```

#### 5.4.2 Entra ID Auto-Provisioning (JIT)

```
User logs in via Microsoft Entra
    │
    ▼
OIDC Callback → /api/v1/auth/entra/callback
    │
    ├── Validate id_token with Entra
    ├── Extract: oid, email, name, groups claim
    │
    ├── Lookup user_identities by (provider=entra, external_id=oid)
    │     │
    │     ├── EXISTS → Update last_used_at, proceed to login
    │     │
    │     └── NOT EXISTS → Auto-provision:
    │           ├── Create internal user record
    │           ├── Create user_identities link
    │           ├── For each Entra group in token:
    │           │     ├── Lookup entra_group_mappings
    │           │     ├── If mapping exists → add to internal group
    │           │     └── If no mapping → skip (admin configures later)
    │           ├── Assign default role (if configured)
    │           └── Audit log: 'user.auto_provisioned'
    │
    ▼
Issue JWT tokens (access + refresh)
```

#### 5.4.3 Entra Group Sync (Background Job)

```
Scheduled Job (every N minutes per tenant)
    │
    ▼
For each entra_group_mapping in tenant:
    │
    ├── Call Microsoft Graph API:
    │   GET /groups/{entra_group_id}/members
    │
    ├── Compare with current internal group membership:
    │
    │   NEW members (in Entra, not in internal):
    │     ├── Lookup user_identities by external_id
    │     │     ├── EXISTS → Add to internal group
    │     │     └── NOT EXISTS → Auto-provision user + add to group
    │     └── Audit log: 'sync.user_added_to_group'
    │
    │   REMOVED members (in internal, not in Entra):
    │     ├── If auto_remove = true → Remove from internal group
    │     └── Audit log: 'sync.user_removed_from_group'
    │
    └── Update last_synced_at timestamp
```

---

## 6. Authentication & Identity

### 6.1 Authentication Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AUTHENTICATION FLOW                                  │
│                                                                            │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                     TENANT AUTH CONFIG                                │   │
│  │  auth_methods = 'form' │ 'entra' │ 'both'                          │   │
│  └────────────┬────────────────────────────┬───────────────────────────┘   │
│               │                            │                              │
│       ┌───────▼───────┐            ┌───────▼───────┐                      │
│       │  FORM AUTH    │            │  ENTRA OIDC   │                      │
│       │               │            │               │                      │
│       │ email/password│            │ Authorization │                      │
│       │ + optional MFA│            │ Code Flow     │                      │
│       │               │            │ + PKCE        │                      │
│       └───────┬───────┘            └───────┬───────┘                      │
│               │                            │                              │
│               └────────────┬───────────────┘                              │
│                            │                                              │
│                    ┌───────▼───────┐                                      │
│                    │ TOKEN SERVICE │                                      │
│                    │               │                                      │
│                    │ Issue JWT:    │                                      │
│                    │ • Access (15m)│                                      │
│                    │ • Refresh (7d)│                                      │
│                    └───────┬───────┘                                      │
│                            │                                              │
│                    ┌───────▼───────┐                                      │
│                    │  JWT PAYLOAD  │                                      │
│                    │               │                                      │
│                    │ sub: user_id  │                                      │
│                    │ tid: tenant_id│                                      │
│                    │ type: user_typ│                                      │
│                    │ roles: [...]  │                                      │
│                    │ groups: [...] │                                      │
│                    │ iat, exp, jti │                                      │
│                    └───────────────┘                                      │
│                                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 JWT Token Structure

```json
// Access Token (short-lived, 15 min)
{
  "sub": "user-uuid",
  "tid": "tenant-uuid",
  "email": "alice@acme.com",
  "name": "Alice Johnson",
  "type": "standard",              // super_admin | tenant_admin | standard
  "roles": ["analyst", "viewer"],
  "groups": ["finance", "reporting"],
  "auth_provider": "entra",
  "iat": 1708300000,
  "exp": 1708300900,
  "jti": "unique-token-id",
  "iss": "vesper-platform",
  "aud": "vesper-api"
}

// Refresh Token: opaque, stored hashed in sessions table
```

### 6.3 API Authentication Middleware Pipeline

```
Incoming Request
    │
    ▼
[1] Rate Limiter (Redis) ─── 429 Too Many Requests
    │
    ▼
[2] Tenant Resolution ─── Extract tenant from domain/header
    │                      Set SESSION_CONTEXT('tenant_id')
    ▼
[3] JWT Validation ─── Verify signature, expiry, audience
    │                  Extract user claims
    ▼
[4] Session Validation ─── Check session not revoked (Redis lookup)
    │
    ▼
[5] Permission Check ─── Evaluate RBAC for requested endpoint
    │                     Check resource + action against user roles
    │                     Cache resolved permissions in Redis (TTL: 5min)
    ▼
[6] Request Handler ─── Business logic with tenant-scoped DB context
    │
    ▼
[7] Audit Logger ─── Async log to audit_logs (via message broker)
```

---

## 7. Backend Architecture (FastAPI)

### 7.1 Project Structure

```
vesper-backend/
├── alembic/                          # Database migrations
│   ├── versions/
│   └── env.py
├── app/
│   ├── __init__.py
│   ├── main.py                       # FastAPI app factory
│   ├── config.py                     # Settings (pydantic-settings)
│   │
│   ├── api/                          # API Layer
│   │   ├── __init__.py
│   │   ├── deps.py                   # Shared dependencies (get_db, get_current_user)
│   │   │
│   │   ├── v1/                       # API Version 1
│   │   │   ├── __init__.py
│   │   │   ├── router.py             # Aggregates all v1 routers
│   │   │   ├── auth.py               # Login, logout, token refresh, OIDC callback
│   │   │   ├── tenants.py            # Tenant CRUD, domain management, resolve
│   │   │   ├── users.py              # User CRUD, activation, password reset
│   │   │   ├── groups.py             # Group CRUD, membership management
│   │   │   ├── roles.py              # Role CRUD, permission assignment
│   │   │   ├── permissions.py        # Permission matrix, bulk assignment
│   │   │   ├── resources.py          # Resource registry (pages/sections/actions)
│   │   │   ├── entra.py              # Entra OIDC config, group mappings, sync
│   │   │   ├── audit.py              # Audit log queries
│   │   │   └── health.py             # Health check endpoints
│   │   │
│   │   └── v2/                       # Future API version
│   │       └── ...
│   │
│   ├── core/                         # Core infrastructure
│   │   ├── __init__.py
│   │   ├── security.py               # JWT encode/decode, password hashing
│   │   ├── permissions.py            # Permission resolution engine
│   │   ├── exceptions.py             # Custom exception classes
│   │   ├── events.py                 # Application lifecycle events
│   │   └── logging.py               # Structured logging config
│   │
│   ├── middleware/                    # Middleware stack
│   │   ├── __init__.py
│   │   ├── tenant_context.py         # Tenant resolution middleware
│   │   ├── rate_limiter.py           # Redis-based rate limiting
│   │   ├── correlation_id.py         # Request tracing
│   │   ├── audit.py                  # Automatic audit logging
│   │   └── error_handler.py          # Global exception handler
│   │
│   ├── models/                       # SQLAlchemy ORM models
│   │   ├── __init__.py
│   │   ├── base.py                   # Base model with tenant_id mixin
│   │   ├── tenant.py
│   │   ├── user.py
│   │   ├── group.py
│   │   ├── role.py
│   │   ├── permission.py
│   │   ├── resource.py
│   │   ├── identity.py              # Entra connections, user identities
│   │   └── audit.py
│   │
│   ├── schemas/                      # Pydantic request/response schemas
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── tenant.py
│   │   ├── user.py
│   │   ├── group.py
│   │   ├── role.py
│   │   ├── permission.py
│   │   └── common.py                # Pagination, filters, etc.
│   │
│   ├── services/                     # Business logic layer
│   │   ├── __init__.py
│   │   ├── auth_service.py
│   │   ├── tenant_service.py
│   │   ├── user_service.py
│   │   ├── group_service.py
│   │   ├── role_service.py
│   │   ├── permission_service.py
│   │   ├── entra_service.py          # Microsoft Graph API integration
│   │   ├── sync_service.py           # Entra group sync worker
│   │   └── cache_service.py          # Redis cache abstractions
│   │
│   ├── repositories/                 # Data access layer
│   │   ├── __init__.py
│   │   ├── base.py                   # Generic CRUD repository
│   │   ├── tenant_repo.py
│   │   ├── user_repo.py
│   │   ├── role_repo.py
│   │   └── permission_repo.py
│   │
│   ├── workers/                      # Background tasks
│   │   ├── __init__.py
│   │   ├── entra_sync_worker.py      # Periodic Entra group sync
│   │   ├── notification_worker.py
│   │   └── audit_worker.py           # Async audit log writer
│   │
│   └── db/                           # Database
│       ├── __init__.py
│       ├── session.py                # Async session factory
│       └── rls.py                    # Row-Level Security helpers
│
├── tests/
│   ├── conftest.py
│   ├── test_auth.py
│   ├── test_permissions.py
│   └── ...
│
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml
├── alembic.ini
└── .env.example
```

### 7.2 API Versioning Strategy

```python
# app/main.py
from fastapi import FastAPI
from app.api.v1.router import router as v1_router

def create_app() -> FastAPI:
    app = FastAPI(
        title="Vesper Analytics API",
        version="1.0.0",
        docs_url="/api/docs",
    )

    # API Versioning via URL prefix
    app.include_router(v1_router, prefix="/api/v1")
    # Future: app.include_router(v2_router, prefix="/api/v2")

    return app
```

**Versioning Rules:**
- Breaking changes → new version (`/api/v2`)
- Additive changes → same version (new fields, new endpoints)
- Deprecated endpoints → `Sunset` header + 6-month deprecation window
- Version discovery → `GET /api/versions` returns available versions

### 7.3 Core Permission Engine (Backend)

```python
# app/core/permissions.py
from typing import Optional
from app.services.cache_service import CacheService

class PermissionEngine:
    """
    Central permission resolution engine.
    Called by middleware and decorators for every protected endpoint.
    """

    CACHE_TTL = 300  # 5 minutes

    def __init__(self, db_session, cache: CacheService):
        self.db = db_session
        self.cache = cache

    async def check_permission(
        self,
        user_id: str,
        tenant_id: str,
        resource_key: str,
        action_key: str,
    ) -> PermissionResult:
        """
        Resolve whether a user has permission to perform an action on a resource.
        Uses Redis cache with 5-min TTL to avoid repeated DB hits.
        """
        cache_key = f"perm:{tenant_id}:{user_id}:{resource_key}:{action_key}"
        cached = await self.cache.get(cache_key)
        if cached is not None:
            return PermissionResult.from_cache(cached)

        # Step 1: Get all effective roles (direct + group-inherited)
        roles = await self._get_effective_roles(user_id)

        # Step 2: Resolve permission for resource + action
        result = await self._resolve_permission(roles, resource_key, action_key)

        # Step 3: If denied, check parent resource hierarchy
        if not result.allowed:
            result = await self._check_hierarchy(roles, resource_key, action_key)

        # Cache result
        await self.cache.set(cache_key, result.to_cache(), ttl=self.CACHE_TTL)
        return result

    async def get_user_permission_matrix(
        self, user_id: str, tenant_id: str
    ) -> dict:
        """
        Returns the FULL permission matrix for a user.
        Called once after login; frontend uses this to render UI.

        Returns:
        {
            "dashboard": {"view": true},
            "dashboard.revenue_chart": {"view": true},
            "dashboard.export_button": {"view": false},
            "reports": {"view": true, "export": true},
            "reports.filters_panel": {"view": true},
            "users": {"view": true, "create": false, "delete": false},
            ...
        }
        """
        cache_key = f"perm_matrix:{tenant_id}:{user_id}"
        cached = await self.cache.get(cache_key)
        if cached:
            return cached

        roles = await self._get_effective_roles(user_id)
        matrix = await self._build_full_matrix(roles)

        await self.cache.set(cache_key, matrix, ttl=self.CACHE_TTL)
        return matrix

    async def invalidate_user_cache(self, user_id: str, tenant_id: str):
        """Called when roles/groups/permissions change for a user."""
        pattern = f"perm:{tenant_id}:{user_id}:*"
        await self.cache.delete_pattern(pattern)
        await self.cache.delete(f"perm_matrix:{tenant_id}:{user_id}")

    async def invalidate_role_cache(self, role_id: str, tenant_id: str):
        """Called when a role's permissions change. Invalidates all users with that role."""
        user_ids = await self._get_users_with_role(role_id)
        for uid in user_ids:
            await self.invalidate_user_cache(uid, tenant_id)
```

### 7.4 Permission Decorator for Endpoints

```python
# app/api/deps.py
from functools import wraps

def require_permission(resource: str, action: str):
    """
    Decorator for FastAPI endpoints.
    Checks if the current user has the required permission.
    """
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            request = kwargs.get("request") or args[0]
            user = request.state.user
            engine = request.state.permission_engine

            result = await engine.check_permission(
                user_id=user.id,
                tenant_id=user.tenant_id,
                resource_key=resource,
                action_key=action,
            )

            if not result.allowed:
                raise PermissionDeniedError(
                    resource=resource, action=action
                )

            return await func(*args, **kwargs)
        return wrapper
    return decorator

# Usage in endpoint:
@router.get("/reports")
@require_permission("reports", "view")
async def get_reports(request: Request, db: AsyncSession = Depends(get_db)):
    ...

@router.post("/reports/{id}/export")
@require_permission("reports.export_button", "view")
async def export_report(id: str, request: Request):
    ...

@router.delete("/users/{id}")
@require_permission("users", "delete")
async def delete_user(id: str, request: Request):
    ...
```

### 7.5 Key API Endpoints

#### Authentication & Identity

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `POST` | `/api/v1/auth/login` | Form-based login (email/password) | Public |
| `POST` | `/api/v1/auth/refresh` | Refresh access token | Refresh Token |
| `POST` | `/api/v1/auth/logout` | Revoke session | Bearer |
| `GET` | `/api/v1/auth/entra/authorize` | Initiate Entra OIDC flow | Public |
| `GET` | `/api/v1/auth/entra/callback` | Entra OIDC callback (auto-provision) | Public |
| `POST` | `/api/v1/auth/mfa/verify` | Verify MFA code | Partial |

#### Tenant Management (Super Admin)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/tenants` | List all tenants |
| `POST` | `/api/v1/tenants` | Create new tenant |
| `GET` | `/api/v1/tenants/{id}` | Get tenant details |
| `PUT` | `/api/v1/tenants/{id}` | Update tenant |
| `PATCH` | `/api/v1/tenants/{id}/status` | Activate/suspend tenant |
| `GET` | `/api/v1/tenants/resolve?domain=...` | Resolve tenant by domain (public) |
| `POST` | `/api/v1/tenants/{id}/domains` | Add custom domain |

#### User Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/users` | List users (tenant-scoped) |
| `POST` | `/api/v1/users` | Create user |
| `GET` | `/api/v1/users/{id}` | Get user details |
| `PUT` | `/api/v1/users/{id}` | Update user |
| `DELETE` | `/api/v1/users/{id}` | Deactivate user |
| `POST` | `/api/v1/users/{id}/groups` | Assign user to groups |
| `POST` | `/api/v1/users/{id}/roles` | Assign roles directly |
| `GET` | `/api/v1/users/me/permissions` | Get current user's permission matrix |

#### Groups

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/groups` | List groups |
| `POST` | `/api/v1/groups` | Create group |
| `PUT` | `/api/v1/groups/{id}` | Update group |
| `GET` | `/api/v1/groups/{id}/members` | List group members |
| `POST` | `/api/v1/groups/{id}/roles` | Assign roles to group |

#### Roles & Permissions

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/roles` | List roles |
| `POST` | `/api/v1/roles` | Create role |
| `PUT` | `/api/v1/roles/{id}` | Update role |
| `GET` | `/api/v1/roles/{id}/permissions` | Get role's permission matrix |
| `PUT` | `/api/v1/roles/{id}/permissions` | Bulk update role permissions |
| `GET` | `/api/v1/resources` | List all resources (for permission UI) |
| `GET` | `/api/v1/permissions/matrix` | Full permission matrix (for admin UI) |

#### Entra Integration

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/entra/config` | Get tenant's Entra config |
| `PUT` | `/api/v1/entra/config` | Update Entra OIDC settings |
| `GET` | `/api/v1/entra/groups` | List Entra group mappings |
| `POST` | `/api/v1/entra/groups/map` | Map Entra group to internal group |
| `POST` | `/api/v1/entra/sync` | Trigger manual sync |
| `GET` | `/api/v1/entra/sync/status` | Get last sync status |

### 7.6 Rate Limiting Strategy

```python
# Redis-backed rate limiting per tenant + per user
RATE_LIMITS = {
    "default":          {"requests": 100, "window": 60},    # 100 req/min
    "auth.login":       {"requests": 5,   "window": 60},    # 5 attempts/min (brute force protection)
    "auth.refresh":     {"requests": 10,  "window": 60},
    "export":           {"requests": 5,   "window": 300},   # 5 exports / 5 min
    "bulk_operations":  {"requests": 2,   "window": 60},
    "super_admin":      {"requests": 500, "window": 60},    # Higher limits for admins
}

# Redis key pattern: rate:{tenant_id}:{user_id}:{endpoint_group}:{window}
# Algorithm: Sliding window counter (Redis sorted sets)
```

---

## 8. Frontend Architecture (React)

### 8.1 Project Structure

```
vesper-frontend/
├── public/
│   └── index.html
├── src/
│   ├── index.tsx
│   ├── App.tsx                         # Root: TenantProvider → ThemeProvider → Router
│   │
│   ├── config/
│   │   ├── api.ts                      # Axios instance, interceptors, base URLs
│   │   ├── routes.ts                   # Route definitions with permission keys
│   │   └── constants.ts
│   │
│   ├── providers/                      # React Context Providers
│   │   ├── TenantProvider.tsx          # Resolves tenant, loads config
│   │   ├── AuthProvider.tsx            # Auth state, tokens, login/logout
│   │   ├── PermissionProvider.tsx      # Permission matrix context
│   │   └── ThemeProvider.tsx           # Dynamic theming per tenant
│   │
│   ├── hooks/                          # Custom Hooks
│   │   ├── useAuth.ts                  # Login, logout, token refresh
│   │   ├── usePermission.ts           # Permission checks for components
│   │   ├── useTenant.ts               # Tenant config access
│   │   └── useApi.ts                   # API call wrapper with error handling
│   │
│   ├── components/                     # Shared Components
│   │   ├── ui/                         # Design system primitives
│   │   │   ├── Button.tsx
│   │   │   ├── Card.tsx
│   │   │   ├── DataTable.tsx
│   │   │   ├── Modal.tsx
│   │   │   └── ...
│   │   │
│   │   ├── auth/
│   │   │   ├── LoginForm.tsx           # Form-based login
│   │   │   ├── EntraLoginButton.tsx    # Microsoft SSO button
│   │   │   ├── MfaVerification.tsx
│   │   │   └── ProtectedRoute.tsx      # Route guard with permission check
│   │   │
│   │   ├── permissions/
│   │   │   ├── PermissionGate.tsx      # Conditionally render children
│   │   │   ├── PermissionMatrix.tsx    # Admin: visual permission editor
│   │   │   └── RolePermissionEditor.tsx
│   │   │
│   │   ├── layout/
│   │   │   ├── AppShell.tsx            # Main layout (sidebar + header + content)
│   │   │   ├── Sidebar.tsx             # Permission-aware navigation
│   │   │   ├── Header.tsx
│   │   │   └── Breadcrumb.tsx
│   │   │
│   │   └── admin/
│   │       ├── UserManagement.tsx
│   │       ├── GroupManagement.tsx
│   │       ├── RoleManagement.tsx
│   │       ├── TenantSettings.tsx
│   │       └── EntraConfig.tsx
│   │
│   ├── pages/                          # Page-level components
│   │   ├── Dashboard.tsx
│   │   ├── Reports.tsx
│   │   ├── UserAdmin.tsx
│   │   ├── Settings.tsx
│   │   └── ...
│   │
│   ├── services/                       # API service layer
│   │   ├── authService.ts
│   │   ├── userService.ts
│   │   ├── roleService.ts
│   │   ├── tenantService.ts
│   │   └── permissionService.ts
│   │
│   ├── themes/                         # Theme infrastructure
│   │   ├── base.ts                     # Base design tokens
│   │   ├── generateTheme.ts           # Dynamic theme from tenant config
│   │   └── overrides.ts               # Component-level style overrides
│   │
│   ├── types/                          # TypeScript types
│   │   ├── auth.ts
│   │   ├── user.ts
│   │   ├── permission.ts
│   │   ├── tenant.ts
│   │   └── api.ts
│   │
│   └── utils/
│       ├── permissions.ts              # Permission helper functions
│       ├── storage.ts                  # Secure token storage
│       └── errorHandler.ts
│
├── package.json
├── tsconfig.json
├── vite.config.ts
└── .env.example
```

### 8.2 Permission-Driven UI System

#### 8.2.1 PermissionGate Component

The core building block for permission-controlled UI:

```tsx
// src/components/permissions/PermissionGate.tsx

interface PermissionGateProps {
  resource: string;             // e.g. 'reports.export_button'
  action?: string;              // defaults to 'view'
  fallback?: React.ReactNode;   // What to render if denied (null = hidden)
  children: React.ReactNode;
}

const PermissionGate: React.FC<PermissionGateProps> = ({
  resource,
  action = 'view',
  fallback = null,
  children,
}) => {
  const { hasPermission } = usePermission();

  if (!hasPermission(resource, action)) {
    return <>{fallback}</>;
  }

  return <>{children}</>;
};

// Usage examples:

// Page-level: hide entire page
<PermissionGate resource="reports">
  <ReportsPage />
</PermissionGate>

// Section-level: hide a card/panel
<PermissionGate resource="dashboard.revenue_chart">
  <RevenueChartCard data={revenueData} />
</PermissionGate>

// Action-level: hide specific buttons
<PermissionGate resource="reports.export_button">
  <Button onClick={handleExport}>Export PDF</Button>
</PermissionGate>

// With fallback (disabled state instead of hidden)
<PermissionGate
  resource="users"
  action="delete"
  fallback={<Button disabled>Delete (No Permission)</Button>}
>
  <Button onClick={handleDelete} variant="danger">Delete User</Button>
</PermissionGate>
```

#### 8.2.2 Permission-Aware Navigation

```tsx
// src/components/layout/Sidebar.tsx

const navigationItems = [
  {
    label: 'Dashboard',
    icon: DashboardIcon,
    path: '/dashboard',
    resource: 'dashboard',                    // Permission key
  },
  {
    label: 'Reports',
    icon: ReportsIcon,
    path: '/reports',
    resource: 'reports',
    children: [
      { label: 'Sales', path: '/reports/sales', resource: 'reports.sales' },
      { label: 'Analytics', path: '/reports/analytics', resource: 'reports.analytics' },
    ],
  },
  {
    label: 'User Management',
    icon: UsersIcon,
    path: '/admin/users',
    resource: 'admin.users',
  },
];

const Sidebar = () => {
  const { hasPermission } = usePermission();

  // Filter navigation items based on permissions
  const visibleItems = navigationItems.filter(item =>
    hasPermission(item.resource, 'view')
  );

  return (
    <nav>
      {visibleItems.map(item => (
        <NavItem key={item.path} {...item}>
          {item.children?.filter(child =>
            hasPermission(child.resource, 'view')
          ).map(child => (
            <NavItem key={child.path} {...child} />
          ))}
        </NavItem>
      ))}
    </nav>
  );
};
```

#### 8.2.3 Protected Route System

```tsx
// src/components/auth/ProtectedRoute.tsx

interface ProtectedRouteProps {
  resource: string;
  action?: string;
  children: React.ReactElement;
}

const ProtectedRoute = ({ resource, action = 'view', children }: ProtectedRouteProps) => {
  const { isAuthenticated } = useAuth();
  const { hasPermission, isLoading } = usePermission();

  if (!isAuthenticated) return <Navigate to="/login" />;
  if (isLoading) return <PageSkeleton />;
  if (!hasPermission(resource, action)) return <AccessDenied />;

  return children;
};

// Route configuration
// src/config/routes.ts
const routes = [
  { path: '/dashboard',      element: <Dashboard />,      resource: 'dashboard' },
  { path: '/reports',        element: <Reports />,        resource: 'reports' },
  { path: '/admin/users',    element: <UserAdmin />,      resource: 'admin.users' },
  { path: '/admin/roles',    element: <RoleAdmin />,      resource: 'admin.roles' },
  { path: '/admin/groups',   element: <GroupAdmin />,      resource: 'admin.groups' },
  { path: '/settings',       element: <Settings />,       resource: 'settings' },
];

// App.tsx
<Routes>
  {routes.map(({ path, element, resource }) => (
    <Route
      key={path}
      path={path}
      element={<ProtectedRoute resource={resource}>{element}</ProtectedRoute>}
    />
  ))}
</Routes>
```

### 8.3 Multi-Tenant Theming System

```tsx
// src/providers/ThemeProvider.tsx

const ThemeProvider = ({ children }: { children: React.ReactNode }) => {
  const { tenantConfig } = useTenant();

  // Generate MUI/CSS theme from tenant configuration
  const theme = useMemo(() => {
    const base = createBaseTheme();

    return deepMerge(base, {
      palette: {
        primary: { main: tenantConfig.branding.primary_color },
        secondary: { main: tenantConfig.branding.secondary_color },
        accent: { main: tenantConfig.branding.accent_color },
      },
      typography: {
        fontFamily: tenantConfig.branding.font_family,
      },
      // Component-level overrides from tenant config
      components: tenantConfig.branding.component_overrides
        ? JSON.parse(tenantConfig.branding.component_overrides)
        : {},
    });
  }, [tenantConfig]);

  return (
    <MuiThemeProvider theme={theme}>
      {/* Inject tenant custom CSS */}
      {tenantConfig.branding.custom_css && (
        <style>{tenantConfig.branding.custom_css}</style>
      )}
      {children}
    </MuiThemeProvider>
  );
};
```

### 8.4 Tenant-Specific Component Customization

```tsx
// Tenant-specific component overrides without code changes
// Stored in tenant_themes.component_overrides (JSON)

{
  "DataTable": {
    "density": "comfortable",           // compact | standard | comfortable
    "stripedRows": true,
    "showColumnFilters": true
  },
  "Sidebar": {
    "variant": "collapsed",             // expanded | collapsed | mini
    "position": "left"
  },
  "LoginPage": {
    "layout": "split",                  // centered | split | fullscreen
    "showSocialLogin": true,
    "customWelcomeText": "Welcome to Acme Analytics"
  },
  "Header": {
    "showBreadcrumbs": true,
    "showNotifications": true,
    "showUserAvatar": true
  }
}

// Components read these overrides:
const DataTable = ({ data, columns }) => {
  const { getComponentConfig } = useTenant();
  const config = getComponentConfig('DataTable');

  return (
    <MuiDataGrid
      rows={data}
      columns={columns}
      density={config.density ?? 'standard'}
      // ...
    />
  );
};
```

### 8.5 Permission Matrix Admin UI

The Tenant Admin sees a visual permission editor:

```
┌──────────────────────────────────────────────────────────────────────────┐
│  ROLE: Analyst                                              [Save] [×]  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Resource                    │ View │ Create │ Edit │ Delete │ Export   │
│  ═══════════════════════════╪══════╪════════╪══════╪════════╪════════  │
│  ▼ Dashboard                │  ☑   │   -    │  -   │   -    │   -     │
│    ├─ Revenue Chart         │  ☑   │   -    │  -   │   -    │   -     │
│    ├─ User Stats Widget     │  ☑   │   -    │  -   │   -    │   -     │
│    └─ Export Button          │  ☐   │   -    │  -   │   -    │   -     │
│  ▼ Reports                  │  ☑   │   ☑   │  ☐   │   ☐    │   ☑    │
│    ├─ Filters Panel         │  ☑   │   -    │  -   │   -    │   -     │
│    ├─ Data Grid             │  ☑   │   -    │  ☐   │   -    │   -     │
│    ├─ Share Button           │  ☑   │   -    │  -   │   -    │   -     │
│    └─ Delete Button          │  ☐   │   -    │  -   │   -    │   -     │
│  ▼ User Management          │  ☑   │   ☐   │  ☐   │   ☐    │   ☐    │
│    ├─ Create Button          │  ☐   │   -    │  -   │   -    │   -     │
│    ├─ Bulk Import            │  ☐   │   -    │  -   │   -    │   -     │
│    └─ Role Assignment        │  ☐   │   -    │  -   │   -    │   -     │
│  ▼ Settings                 │  ☐   │   -    │  ☐   │   -    │   -     │
│                                                                         │
│  ☑ = Allowed    ☐ = Denied    - = Not applicable for this resource     │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Caching, Messaging & Broker Architecture

### 9.1 Redis Caching Strategy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         REDIS CACHE LAYERS                                  │
│                                                                            │
│  Layer 1: Session Cache                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Key: session:{session_id}                                           │   │
│  │ Value: { user_id, tenant_id, roles, revoked }                      │   │
│  │ TTL: 15 minutes (matches access token)                             │   │
│  │ Purpose: Fast session validation without DB hit                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Layer 2: Permission Cache                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Key: perm_matrix:{tenant_id}:{user_id}                             │   │
│  │ Value: { full permission matrix JSON }                             │   │
│  │ TTL: 5 minutes                                                     │   │
│  │ Invalidation: On role/group/permission change                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Layer 3: Tenant Config Cache                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Key: tenant_config:{domain}                                        │   │
│  │ Value: { full tenant config + theme + branding }                   │   │
│  │ TTL: 10 minutes                                                    │   │
│  │ Invalidation: On tenant config update                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Layer 4: Rate Limiting                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Key: rate:{tenant_id}:{user_id}:{endpoint}                        │   │
│  │ Structure: Sorted set (sliding window)                             │   │
│  │ TTL: Auto-expire with window                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Layer 5: API Response Cache                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Key: api_cache:{tenant_id}:{endpoint_hash}:{params_hash}          │   │
│  │ TTL: Varies by endpoint (30s - 5min)                               │   │
│  │ Purpose: Expensive queries (reports, analytics aggregations)       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
│  Azure Service: Azure Cache for Redis (Premium tier, 6GB, cluster)        │
│                                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Message Broker Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     EVENT-DRIVEN ARCHITECTURE                               │
│                                                                            │
│  Broker: Azure Service Bus (SaaS) or RabbitMQ on Container Apps (IaaS)    │
│                                                                            │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │                         TOPICS / QUEUES                            │     │
│  │                                                                    │     │
│  │  ┌──────────────────────┐  Producers:     Consumers:              │     │
│  │  │  user.events         │  Auth Service → Notification Service    │     │
│  │  │  • user.created      │                 Audit Worker            │     │
│  │  │  • user.updated      │                 Entra Sync Worker       │     │
│  │  │  • user.deactivated  │                 Analytics Engine        │     │
│  │  │  • user.login        │                                        │     │
│  │  └──────────────────────┘                                        │     │
│  │                                                                    │     │
│  │  ┌──────────────────────┐  Producers:     Consumers:              │     │
│  │  │  tenant.events       │  Tenant Mgmt → Provisioning Worker     │     │
│  │  │  • tenant.created    │                 DNS Config Worker       │     │
│  │  │  • tenant.suspended  │                 Notification Service    │     │
│  │  │  • tenant.config_upd │                 Cache Invalidation     │     │
│  │  └──────────────────────┘                                        │     │
│  │                                                                    │     │
│  │  ┌──────────────────────┐  Producers:     Consumers:              │     │
│  │  │  permission.events   │  RBAC Service → Cache Invalidation     │     │
│  │  │  • role.updated      │                 Audit Worker            │     │
│  │  │  • perm.assigned     │                 WebSocket Notifier      │     │
│  │  │  • group.membership  │                 (push to frontend)      │     │
│  │  └──────────────────────┘                                        │     │
│  │                                                                    │     │
│  │  ┌──────────────────────┐  Producers:     Consumers:              │     │
│  │  │  audit.events        │  All Services → Audit Writer            │     │
│  │  │  • action.performed  │                 (bulk insert to SQL)    │     │
│  │  └──────────────────────┘                                        │     │
│  │                                                                    │     │
│  │  ┌──────────────────────┐  Producers:     Consumers:              │     │
│  │  │  entra.sync          │  Scheduler   → Entra Sync Worker       │     │
│  │  │  • sync.requested    │                                        │     │
│  │  │  • sync.completed    │                                        │     │
│  │  └──────────────────────┘                                        │     │
│  │                                                                    │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                            │
│  Benefits:                                                                 │
│  • Decoupled services: auth doesn't wait for audit writes                 │
│  • Resilience: if notification service is down, events queue up           │
│  • Scale independently: audit writer can batch-process                    │
│  • Cross-service communication without direct API coupling                │
│                                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 9.3 Inter-Service Communication Pattern

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              MICROSERVICE COMMUNICATION PATTERNS                            │
│                                                                            │
│  Synchronous (HTTP/gRPC):                                                  │
│  ┌──────────────────────────────────────────────────────────────────┐      │
│  │ • API Gateway → Auth Service (token validation)                  │      │
│  │ • API Gateway → Core API (business logic)                       │      │
│  │ • Core API → User Mgmt Service (user lookups)                   │      │
│  │ • Tenant Mgmt → Entra Service (OIDC verification)               │      │
│  │                                                                  │      │
│  │ Pattern: Service mesh via Azure Container Apps internal DNS      │      │
│  │ Format: REST (JSON) for simplicity, gRPC for high-throughput    │      │
│  └──────────────────────────────────────────────────────────────────┘      │
│                                                                            │
│  Asynchronous (Message Broker):                                            │
│  ┌──────────────────────────────────────────────────────────────────┐      │
│  │ • Audit logging (fire-and-forget)                                │      │
│  │ • Email/notification dispatch                                    │      │
│  │ • Entra group sync (scheduled + on-demand)                      │      │
│  │ • Cache invalidation broadcasts                                  │      │
│  │ • Analytics event ingestion                                     │      │
│  │                                                                  │      │
│  │ Pattern: Pub/Sub topics with durable subscriptions              │      │
│  │ Dead letter: Failed messages → DLQ for investigation            │      │
│  └──────────────────────────────────────────────────────────────────┘      │
│                                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. Security Architecture

### 10.1 Security Controls Matrix

| Layer | Control | Implementation |
|-------|---------|---------------|
| **Network** | VNET isolation | All services in Azure VNET; DB and Redis on private endpoints |
| **Network** | WAF | Azure Front Door WAF with OWASP Core Rule Set 3.2 |
| **Network** | DDoS | Azure DDoS Protection Standard |
| **Transport** | TLS 1.3 | End-to-end encryption; TLS termination at Front Door |
| **Authentication** | JWT RS256 | Asymmetric signing; public key rotation via JWKS endpoint |
| **Authentication** | MFA | TOTP-based; enforced per tenant policy |
| **Authentication** | Brute Force Protection | Account lockout after 5 failed attempts (30-min cooldown) |
| **Authorization** | RBAC | Granular permission engine with explicit deny |
| **Authorization** | SQL RLS | Row-Level Security on all tenant-scoped tables |
| **Data** | Encryption at Rest | Azure SQL TDE + Azure Storage SSE |
| **Data** | Encryption in Transit | TLS 1.3 everywhere |
| **Data** | Secret Management | Azure Key Vault; never store secrets in config/code |
| **Data** | Password Hashing | Argon2id (memory-hard) with per-user salt |
| **API** | Rate Limiting | Redis sliding window; per-tenant + per-user + per-endpoint |
| **API** | Input Validation | Pydantic schemas with strict validation |
| **API** | CORS | Per-tenant allowed origins |
| **API** | CSRF | SameSite=Strict cookies + CSRF token for form auth |
| **Frontend** | XSS Prevention | React auto-escaping + CSP headers |
| **Frontend** | Token Storage | Access token in memory; refresh token in HttpOnly cookie |
| **Audit** | Logging | All state-changing operations logged with before/after snapshots |
| **Audit** | Login History | All login attempts (success + failure) recorded with IP + UA |
| **Compliance** | Data Residency | Azure region selection per tenant (configurable) |

### 10.2 CORS Configuration (Per-Tenant)

```python
# Dynamic CORS based on tenant domains
@app.middleware("http")
async def cors_middleware(request: Request, call_next):
    origin = request.headers.get("origin")
    tenant = await resolve_tenant_from_origin(origin)

    if tenant and origin in tenant.allowed_origins:
        response = await call_next(request)
        response.headers["Access-Control-Allow-Origin"] = origin
        response.headers["Access-Control-Allow-Credentials"] = "true"
        response.headers["Access-Control-Allow-Methods"] = "GET,POST,PUT,DELETE,PATCH"
        response.headers["Access-Control-Allow-Headers"] = "Authorization,Content-Type,X-Tenant-ID"
        return response

    return await call_next(request)
```

### 10.3 Content Security Policy

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
  font-src 'self' https://fonts.gstatic.com;
  img-src 'self' data: https://cdn.vesper.com https://*.blob.core.windows.net;
  connect-src 'self' https://*.vesper-api.com https://login.microsoftonline.com;
  frame-ancestors 'none';
  base-uri 'self';
```

---

## 11. Azure Deployment & Infrastructure

### 11.1 Infrastructure Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AZURE INFRASTRUCTURE TOPOLOGY                            │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Resource Group: rg-vesper-production                               │   │
│  │                                                                     │   │
│  │  ┌───────────────────────────────────────────────────────────────┐  │   │
│  │  │  Azure Front Door (Premium)                                   │  │   │
│  │  │  • Custom domains (*.vesper.com + tenant custom domains)     │  │   │
│  │  │  • SSL certificate management (auto-provisioning)            │  │   │
│  │  │  • WAF policy (OWASP 3.2 + custom rules)                   │  │   │
│  │  │  • Global load balancing                                     │  │   │
│  │  │  • Caching for static assets                                 │  │   │
│  │  └───────────────┬─────────────────────┬─────────────────────────┘  │   │
│  │                  │                     │                            │   │
│  │       ┌──────────▼──────────┐  ┌───────▼───────────────────┐       │   │
│  │       │ Static Web App      │  │ Container Apps Env        │       │   │
│  │       │ (React Frontend)    │  │ (VNET-integrated)         │       │   │
│  │       │                     │  │                           │       │   │
│  │       │ • Auto CI/CD from   │  │ ┌─────────────────────┐  │       │   │
│  │       │   GitHub Actions    │  │ │ API Gateway (YARP)  │  │       │   │
│  │       │ • Global CDN        │  │ │ Scale: 2-20 replicas│  │       │   │
│  │       │ • Staging slots     │  │ └──────────┬──────────┘  │       │   │
│  │       └─────────────────────┘  │            │             │       │   │
│  │                                │ ┌──────────▼──────────┐  │       │   │
│  │                                │ │ Auth Service        │  │       │   │
│  │                                │ │ Scale: 2-10 replicas│  │       │   │
│  │                                │ └─────────────────────┘  │       │   │
│  │                                │ ┌─────────────────────┐  │       │   │
│  │                                │ │ Core API Service    │  │       │   │
│  │                                │ │ Scale: 3-30 replicas│  │       │   │
│  │                                │ └─────────────────────┘  │       │   │
│  │                                │ ┌─────────────────────┐  │       │   │
│  │                                │ │ User Mgmt Service   │  │       │   │
│  │                                │ │ Scale: 2-15 replicas│  │       │   │
│  │                                │ └─────────────────────┘  │       │   │
│  │                                │ ┌─────────────────────┐  │       │   │
│  │                                │ │ Background Workers  │  │       │   │
│  │                                │ │ (Entra sync, audit) │  │       │   │
│  │                                │ │ Scale: 1-5 replicas │  │       │   │
│  │                                │ └─────────────────────┘  │       │   │
│  │                                └───────────────────────────┘       │   │
│  │                                                                    │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │  Data Layer (Private Endpoints in VNET)                     │   │   │
│  │  │                                                             │   │   │
│  │  │  ┌─────────────────┐  ┌──────────────┐  ┌───────────────┐  │   │   │
│  │  │  │ Azure SQL DB    │  │ Azure Redis  │  │ Azure Service │  │   │   │
│  │  │  │ (Business Crit.)│  │ (Premium P1) │  │ Bus (Standard)│  │   │   │
│  │  │  │                 │  │              │  │               │  │   │   │
│  │  │  │ • Geo-replicas  │  │ • 6GB cache  │  │ • Topics +    │  │   │   │
│  │  │  │ • Auto-failover │  │ • Clustering │  │   Subscriptns │  │   │   │
│  │  │  │ • TDE enabled   │  │ • Persistence│  │ • Dead letter │  │   │   │
│  │  │  │ • Auditing on   │  │              │  │               │  │   │   │
│  │  │  └─────────────────┘  └──────────────┘  └───────────────┘  │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                                                                    │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │  Operations                                                 │   │   │
│  │  │                                                             │   │   │
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │   │   │
│  │  │  │ Key Vault    │  │ App Insights │  │ Log Analytics    │  │   │   │
│  │  │  │ (Secrets)    │  │ (APM)        │  │ Workspace        │  │   │   │
│  │  │  └──────────────┘  └──────────────┘  └──────────────────┘  │   │   │
│  │  │                                                             │   │   │
│  │  │  ┌──────────────┐  ┌──────────────┐                        │   │   │
│  │  │  │ Container    │  │ Azure DNS    │                        │   │   │
│  │  │  │ Registry     │  │ Zones        │                        │   │   │
│  │  │  └──────────────┘  └──────────────┘                        │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                                                                    │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 11.2 Auto-Scaling Configuration

| Service | Min | Max | Scale Trigger | Cooldown |
|---------|-----|-----|--------------|----------|
| API Gateway | 2 | 20 | CPU > 70% OR concurrent requests > 100 | 60s |
| Auth Service | 2 | 10 | CPU > 60% OR request latency P95 > 500ms | 60s |
| Core API | 3 | 30 | CPU > 70% OR concurrent requests > 200 | 90s |
| User Mgmt | 2 | 15 | CPU > 70% OR queue depth > 50 | 60s |
| Workers | 1 | 5 | Queue depth > 100 messages | 120s |

### 11.3 CI/CD Pipeline

```
┌────────────────────────────────────────────────────────────────────────┐
│                        CI/CD PIPELINE                                  │
│                                                                        │
│  GitHub Repository                                                     │
│       │                                                                │
│       ├──► PR Created / Push to main                                  │
│       │                                                                │
│       ▼                                                                │
│  GitHub Actions Workflow                                               │
│       │                                                                │
│       ├── Stage 1: Lint + Type Check                                  │
│       │   ├── Python: ruff, mypy                                      │
│       │   └── React: eslint, tsc --noEmit                             │
│       │                                                                │
│       ├── Stage 2: Unit Tests                                         │
│       │   ├── Python: pytest (90%+ coverage)                          │
│       │   └── React: vitest + testing-library                         │
│       │                                                                │
│       ├── Stage 3: Integration Tests                                  │
│       │   └── Docker Compose with test DB + Redis                     │
│       │                                                                │
│       ├── Stage 4: Security Scan                                      │
│       │   ├── Trivy (container scan)                                  │
│       │   ├── Bandit (Python security)                                │
│       │   └── npm audit                                               │
│       │                                                                │
│       ├── Stage 5: Build & Push                                       │
│       │   ├── Docker build → Azure Container Registry                 │
│       │   └── React build → artifact                                  │
│       │                                                                │
│       ├── Stage 6: Deploy to Staging                                  │
│       │   ├── Alembic migrations (auto)                               │
│       │   ├── Container Apps revision deploy                          │
│       │   └── Static Web App staging slot                             │
│       │                                                                │
│       ├── Stage 7: Smoke Tests on Staging                             │
│       │   └── Health checks + critical path E2E                      │
│       │                                                                │
│       └── Stage 8: Production Deploy (manual approval gate)           │
│           ├── Blue/green deployment (Container Apps revisions)        │
│           ├── Gradual traffic shift (10% → 50% → 100%)               │
│           └── Auto-rollback on error rate spike                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 11.4 Environment Strategy

| Environment | Purpose | Azure SQL | Redis | Infra |
|------------|---------|-----------|-------|-------|
| **Local** | Developer machine | Docker SQL Server | Docker Redis | docker-compose |
| **Dev** | Integration testing | Azure SQL (Basic) | Redis (Basic) | Container Apps (min scale) |
| **Staging** | Pre-production validation | Azure SQL (Standard) | Redis (Standard) | Container Apps (prod-like) |
| **Production** | Live traffic | Azure SQL (Business Critical) | Redis (Premium) | Container Apps (auto-scale) |

---

## 12. API Design & Versioning

### 12.1 API Design Standards

```
Standard Response Envelope:

{
  "status": "success",                    // "success" | "error"
  "data": { ... },                        // Response payload
  "meta": {                               // Pagination / context
    "page": 1,
    "per_page": 20,
    "total": 150,
    "total_pages": 8
  },
  "errors": null                          // Array of error objects when status = "error"
}

Error Response:

{
  "status": "error",
  "data": null,
  "errors": [
    {
      "code": "PERMISSION_DENIED",
      "message": "You do not have permission to access this resource.",
      "field": null,
      "details": {
        "resource": "users",
        "action": "delete",
        "required_role": "tenant_admin"
      }
    }
  ]
}

HTTP Status Codes:
  200 - Success
  201 - Created
  204 - No Content (delete)
  400 - Validation Error
  401 - Unauthenticated
  403 - Forbidden (permission denied)
  404 - Not Found
  409 - Conflict (duplicate)
  422 - Unprocessable Entity
  429 - Rate Limited
  500 - Internal Server Error
```

### 12.2 API Versioning Policy

| Strategy | Implementation |
|----------|---------------|
| **Method** | URL path prefix (`/api/v1/`, `/api/v2/`) |
| **Breaking Change** | New major version; old version supported for 12 months |
| **Additive Change** | Same version; new optional fields, new endpoints |
| **Sunset Header** | `Sunset: Sat, 01 Jan 2028 00:00:00 GMT` on deprecated endpoints |
| **Discovery** | `GET /api/versions` returns all available versions and deprecation status |

### 12.3 Pagination, Filtering & Sorting

```
GET /api/v1/users?page=1&per_page=20&sort=-created_at&status=active&search=alice

Query Parameters:
  page        - Page number (default: 1)
  per_page    - Items per page (default: 20, max: 100)
  sort        - Sort field; prefix with - for descending
  search      - Full-text search across name, email
  status      - Filter by status enum
  group_id    - Filter by group membership
  role_id     - Filter by role assignment
```

---

## 13. Appendix: Entity Relationship Diagrams

### 13.1 Complete Permission Resolution - Visual Flow

```
                            ┌──────────┐
                            │   USER   │
                            │  (Alice) │
                            └────┬─────┘
                                 │
                   ┌─────────────┼─────────────┐
                   │                           │
            ┌──────▼──────┐             ┌──────▼──────┐
            │ Direct Role │             │   Groups    │
            │ Assignment  │             │  Membership │
            └──────┬──────┘             └──────┬──────┘
                   │                           │
            ┌──────▼──────┐             ┌──────▼──────┐
            │   ROLE:     │             │   GROUP:    │
            │  "Analyst"  │             │  "Finance"  │
            └──────┬──────┘             └──────┬──────┘
                   │                           │
                   │                    ┌──────▼──────┐
                   │                    │   ROLE:     │
                   │                    │  "Viewer"   │
                   │                    └──────┬──────┘
                   │                           │
                   └───────────┬───────────────┘
                               │
                        ┌──────▼──────┐
                        │  EFFECTIVE  │
                        │   ROLES:    │
                        │  Analyst +  │
                        │  Viewer     │
                        └──────┬──────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
       ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
       │  Analyst    │ │  Analyst    │ │   Viewer    │
       │  Perms for  │ │  Perms for  │ │   Perms for │
       │  Dashboard  │ │  Reports    │ │   Dashboard │
       └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
              │                │                │
              ▼                ▼                ▼
  ┌─────────────────────────────────────────────────────┐
  │              MERGED PERMISSION MATRIX               │
  │                                                     │
  │  dashboard           → view: ✓                     │
  │  dashboard.revenue   → view: ✓                     │
  │  dashboard.export    → view: ✗ (analyst:deny)      │
  │  reports             → view: ✓                     │
  │  reports.filters     → view: ✓                     │
  │  reports.export      → view: ✓ (analyst:allow)     │
  │  reports.delete      → view: ✗ (no grant = deny)   │
  │  users               → view: ✗ (no grant)          │
  └─────────────────────────────────────────────────────┘
              │
              ▼
  Sent to React frontend via GET /api/v1/users/me/permissions
  Frontend uses PermissionGate to show/hide UI elements
```

### 13.2 Entra ID Integration Flow

```
┌──────────┐     ┌──────────┐     ┌──────────────┐     ┌──────────────┐
│  Browser │     │  React   │     │  FastAPI      │     │  Microsoft   │
│          │     │  App     │     │  Backend      │     │  Entra ID    │
└────┬─────┘     └────┬─────┘     └──────┬────────┘     └──────┬───────┘
     │                │                   │                     │
     │  Click "Login  │                   │                     │
     │  with Microsoft│                   │                     │
     │───────────────▶│                   │                     │
     │                │                   │                     │
     │                │ GET /auth/entra/  │                     │
     │                │ authorize?tenant= │                     │
     │                │──────────────────▶│                     │
     │                │                   │                     │
     │                │ Redirect URL with │                     │
     │                │ state + PKCE      │                     │
     │                │◀──────────────────│                     │
     │                │                   │                     │
     │ Redirect to    │                   │                     │
     │ login.microsoft│                   │                     │
     │ online.com     │                   │                     │
     │◀───────────────│                   │                     │
     │                                    │                     │
     │ User authenticates with Microsoft  │                     │
     │──────────────────────────────────────────────���────────▶│
     │                                    │                     │
     │ Auth code callback                 │                     │
     │────────────────────────────────────▶│                     │
     │                                    │                     │
     │                                    │ Exchange code for   │
     │                                    │ tokens              │
     │                                    │────────────────────▶│
     │                                    │                     │
     │                                    │ id_token + access   │
     │                                    │ token               │
     │                                    │◀────────────────────│
     │                                    │                     │
     │                                    │ Validate id_token   │
     │                                    │ Extract: oid, email │
     │                                    │ name, groups        │
     │                                    │                     │
     │                                    │ Lookup user by oid  │
     │                                    │ ┌────────────────┐  │
     │                                    │ │ User exists?   │  │
     │                                    │ │ YES → login    │  │
     │                                    │ │ NO → provision │  │
     │                                    │ │  • Create user │  │
     │                                    │ │  • Map groups  │  │
     │                                    │ │  • Set identity│  │
     │                                    │ └────────────────┘  │
     │                                    │                     │
     │ Set cookies (refresh token)        │                     │
     │ Return access token (JSON)         │                     │
     │◀───────────────────────────────────│                     │
     │                                    │                     │
     │ Load app with permissions          │                     │
     │───────────────▶│                   │                     │
```

### 13.3 Key Design Decisions Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Multi-tenant DB model | Single DB, shared schema with RLS | Cost-effective, simpler operations, tenant isolation via RLS + app layer |
| Permission model | Resource + Action matrix per Role | Industry-standard RBAC (similar to AWS IAM, Azure RBAC); granular yet manageable |
| Permission granularity | Page → Section → Action (hierarchical) | Enables UI control at any level without rigid structure |
| Token strategy | Short-lived JWT (15min) + refresh token | Balance between security and UX; stateless API validation |
| Identity federation | OIDC with JIT provisioning + periodic sync | Seamless Entra integration; internal user always exists for permission mapping |
| Caching | Redis with event-driven invalidation | Sub-millisecond permission checks; consistent invalidation via broker events |
| Frontend permissions | Server-sent permission matrix + client-side gates | Zero-trust: UI never decides permissions; backend is the authority |
| API versioning | URL path prefix | Simple, explicit, works with any HTTP client |
| Message broker | Azure Service Bus (SaaS) | Managed service, dead-letter support, AMQP 1.0, no infra overhead |

---

**Document End**

*This document serves as the architectural blueprint for the Vesper Analytics multi-tenant platform. All design decisions should be reviewed and approved before implementation begins. Implementation should proceed in phases, starting with the core tenant and user management system, followed by the RBAC engine, Entra integration, and finally the frontend permission system.*
