# Vesper Analytics Platform — Technical Design Document

**Document Version:** 2.0
**Date:** 2026-02-26
**Classification:** Internal — Confidential
**Author:** Engineering Team
**Audience:** CTO, Engineering Leadership, Backend Engineers, Frontend Engineers, DevOps

---

## How to Read This Document

This document is organised in seven logical parts, progressing from architecture concepts through security foundations, backend implementation, advanced platform features, frontend integration, and finally to API reference and operations. The progression mirrors the recommended build sequence for the platform.

**Audience guide — focus on these parts based on your role:**

| Role | Recommended Sections |
|------|---------------------|
| **Solutions Architect / CTO** | Parts I & II — Sections 1–5 |
| **Backend Engineer** | Parts III, IV & V — Sections 6–14 |
| **Frontend Engineer** | Parts I, VI & §18 — Sections 1–3, 16–17, 18 |
| **DevOps / Infrastructure** | Parts I, II & VII — Sections 1–5, 18 |
| **Security Reviewer** | Sections 4, 7, 10 in full |
| **New Team Member** | Read all parts in order |

**Document conventions:**
- *Tenant* and *organisation* are used interchangeably throughout.
- A *resource* in RBAC context means a permission target (page, section, button) — not an Azure cloud resource.
- All REST API paths use the `/api/v1/` prefix. WebSocket paths use `/ws/v1/`.
- Code blocks show canonical implementation patterns; adapt configuration to your environment.

---

## Table of Contents

**PART I — PLATFORM OVERVIEW**
1. [Executive Summary](#1-executive-summary)
2. [System Architecture Overview](#2-system-architecture-overview)
3. [Multi-Tenant Architecture](#3-multi-tenant-architecture)

**PART II — SECURITY & DATA FOUNDATION**
4. [Security Architecture](#4-security-architecture)
5. [Database Design](#5-database-design)

**PART III — USER MANAGEMENT & ACCESS CONTROL**
6. [User Management & RBAC System](#6-user-management--rbac-system)
7. [Authentication & Identity](#7-authentication--identity)

**PART IV — BACKEND IMPLEMENTATION**
8. [Backend Architecture (FastAPI)](#8-backend-architecture-fastapi)
9. [API Design & Versioning Standards](#9-api-design--versioning-standards)
10. [Middleware Implementation (Full Stack)](#10-middleware-implementation-full-stack)
11. [Caching, Messaging & Broker Architecture](#11-caching-messaging--broker-architecture)

**PART V — PLATFORM FEATURES**
12. [WebSocket Architecture (Notifications & Chat)](#12-websocket-architecture-notifications--chat)
13. [AI Chat Agent — LangChain & LangGraph](#13-ai-chat-agent--langchain--langgraph)
14. [Power BI Integration & Backend-Controlled Menu System](#14-power-bi-integration--backend-controlled-menu-system)
15. [File Upload — SharePoint & OneLake Integration](#15-file-upload--sharepoint--onelake-integration)

**PART VI — FRONTEND**
16. [Frontend Architecture (React)](#16-frontend-architecture-react)
17. [Frontend Integration Guide](#17-frontend-integration-guide)

**PART VII — REFERENCE & OPERATIONS**
18. [Complete API Reference — End-to-End](#18-complete-api-reference--end-to-end)
19. [Azure Deployment & Infrastructure](#19-azure-deployment--infrastructure)
20. [Appendix: Entity Relationship Diagrams](#20-appendix-entity-relationship-diagrams)

---

---

## PART I — PLATFORM OVERVIEW

*This part introduces the Vesper Analytics platform: its purpose, the complete high-level system architecture (including WebSocket, AI Agent, and Power BI channels), and the multi-tenancy strategy that underpins every design decision in this document. Read this part first regardless of your role.*

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



### 1.4 Implementation Roadmap

The platform is built incrementally in six phases. Each phase is independently deployable and testable before the next begins.

| Phase | Focus Area | Key Sections | Deliverables |
|-------|-----------|-------------|-------------|
| **Phase 1** | Foundation | §4, §5, §7 | Azure infrastructure, full database schema with RLS, JWT auth service (login / logout / refresh / Entra OIDC) |
| **Phase 2** | Multi-Tenancy & RBAC | §3, §5, §6 | Tenant domain resolution, user / group / role management APIs, permission engine with Redis cache |
| **Phase 3** | Backend Core | §8, §9, §10 | FastAPI project, 9-layer middleware pipeline, API versioning and response standards |
| **Phase 4** | Infrastructure | §11, §18 | Redis caching layers, Service Bus event topics, CI/CD pipeline, auto-scaling |
| **Phase 5** | Platform Features | §12, §13, §14 | WebSocket channels, AI Chat Agent (LangGraph multi-agent), Power BI embed + menu system |
| **Phase 6** | Frontend | §16, §17 | React app shell, permission-aware UI, full integration with all APIs and WebSockets |

> **Phase dependencies:** Phase 2 requires Phase 1's auth service. Phase 5's WebSocket auth reuses the JWT middleware from Phase 3. Phase 5's AI Agent uses the Service Bus from Phase 4. Phase 6 requires all previous phases to be complete.


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



### 2.3 Platform Feature Communication Channels

The base REST API architecture is extended by three additional communication channels. These are covered in depth in Part V and Part VI.

```
Browser / Client
  │
  ├── HTTPS REST ─────────────────► REST API Gateway (Container App)
  │                                       │
  │                                  [MW-1 → MW-9 Middleware Pipeline]
  │                                       │
  │                                  Route Handlers → Services → DB / Redis / Service Bus
  │
  ├── WSS WebSocket ────────────── WebSocket Gateway (same Container App)
  │   /ws/v1/notifications               │
  │   /ws/v1/chat                        ├── Notification Service  (server → client push)
  │                                      └── Chat Channel → AI Agent Service
  │                                                │
  │                                      LangGraph Orchestrator
  │                                      ├── Intent Classifier (LLM)
  │                                      ├── SQL Agent (tenant-scoped read queries)
  │                                      ├── BI Insights Agent (Power BI dataset)
  │                                      └── Response Synthesizer (LLM streaming)
  │
  └── Power BI <iframe> ──────────► app.powerbi.com  (Microsoft CDN)
      embed token provided by             Token obtained via:
      backend API call                    GET /api/v1/powerbi/reports/{id}/embed-token
                                          Backend → Power BI Embed API → embed_url + token
                                          Token is user-scoped (optional Power BI RLS applied)
```

**Channel summary:**

| Channel | Protocol | Auth Mechanism | Direction | Primary Use |
|---------|----------|---------------|-----------|-------------|
| REST API | HTTPS | `Authorization: Bearer <jwt>` header | Request / Response | All CRUD, menus, Power BI token APIs |
| WS Notifications | WSS | JWT in `?token=` query param | Server → Client | Alerts, permission changes, report-ready events |
| WS Chat | WSS | JWT in `?token=` query param | Bidirectional | AI assistant queries and streaming responses |
| Power BI Embed | HTTPS iframe | Short-lived embed token | Client → powerbi.com | Embedded analytics report rendering |


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

---

## PART II — SECURITY & DATA FOUNDATION

*Security controls and the complete data model are foundational — every feature in the platform is built on top of them. Security is presented before implementation details to establish the zero-trust design philosophy early. The database schema follows immediately as the single source of truth for all platform entities. All feature-specific tables (notifications, chat, menu, Power BI) are indexed here and fully defined in their respective feature sections.*

---

## 4. Security Architecture

### 4.1 Security Controls Matrix

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

### 4.2 CORS Configuration (Per-Tenant)

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

### 4.3 Content Security Policy

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

## 5. Database Design

### 5.1 Complete Entity Relationship Model

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

### 5.2 Complete Table Definitions

#### 5.2.1 Tenant Management Domain

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

#### 5.2.2 User Management Domain

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

#### 5.2.3 RBAC / Permission Domain (Core Design)

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

#### 5.2.4 Permission Resolution Logic

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

#### 5.2.5 Identity Federation Domain (Microsoft Entra ID)

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

#### 5.2.6 Audit & System Domain

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

### 5.3 Key Database Indexes for Performance

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


### 5.4 Feature Domain Tables

The following feature-specific tables belong to the same single database and follow identical tenant isolation patterns (`tenant_id` FK + SQL RLS). Their full SQL `CREATE TABLE` definitions are kept in the feature sections that introduce them — this keeps schema definitions co-located with their architectural context.

| Domain | Tables | Full Schema Defined In |
|--------|--------|----------------------|
| **Menu & Navigation** | `menu_items` | Section 14.2 |
| **Power BI Reporting** | `powerbi_reports`, `powerbi_tenant_config`, `powerbi_embed_tokens` | Section 14.2 |
| **Real-Time Notifications** | `notifications` | Section 12.4 |
| **AI Chat** | `chat_sessions`, `chat_messages`, `chat_agent_tools` | Section 13.2 |

> **Isolation guarantee:** All feature tables carry `tenant_id UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id)` and are covered by the same SQL Row-Level Security policy from Section 5.1. The `resource_key` column on `menu_items` directly references the `resources` table (Section 5.2.3), making RBAC the single control plane for both API access and UI menu visibility — no separate menu permission configuration is needed.

---

## PART III — USER MANAGEMENT & ACCESS CONTROL

*This part covers the user lifecycle, the RBAC permission model, and all authentication flows. Read these two sections before the backend implementation — every API request in the platform flows through the permission system described here, and the middleware pipeline (Part IV) builds directly on the auth model defined here.*

---

## 6. User Management & RBAC System

> **Resource keys — single permission gate for APIs and menus.** Every `resource_key` registered in the `resources` table serves two simultaneous purposes:
> 1. **API-level enforcement:** Endpoints decorated with `@require_permission(resource_key, action)` check this key against the user's permission matrix before the handler runs.
> 2. **Menu-level visibility:** Each `menu_items` record (Section 14) carries a `resource_key`. The `MenuService` filters the navigation tree by checking `view` permission on each item's key before returning the menu to the client. A single role permission change therefore propagates automatically to both API access and sidebar visibility — no separate menu configuration needed.

### 6.1 User Hierarchy


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

### 6.2 Permission Assignment Flow

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

### 6.3 Permission Granularity Levels

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

### 6.4 User Provisioning Workflows

#### 6.4.1 Manual User Creation (Form Auth)

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

#### 6.4.2 Entra ID Auto-Provisioning (JIT)

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

#### 6.4.3 Entra Group Sync (Background Job)

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

## 7. Authentication & Identity

### 7.1 Authentication Architecture

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

### 7.2 JWT Token Structure

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

### 7.3 API Authentication Middleware Pipeline

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

---

## PART IV — BACKEND IMPLEMENTATION

*This part covers the FastAPI backend in four sequential sections that should be read in order: project structure and conventions (§8), API request/response standards (§9), the nine-layer middleware security pipeline (§10), and the caching and messaging infrastructure (§11). API standards are placed before middleware because understanding what a valid request looks like is a prerequisite for understanding how requests are validated.*

---

## 8. Backend Architecture (FastAPI)

### 8.1 Project Structure

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

### 8.2 API Versioning Strategy

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

### 8.3 Core Permission Engine (Backend)

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

### 8.4 Permission Decorator for Endpoints

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

### 8.5 Key API Endpoints

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

### 8.6 Rate Limiting Strategy

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

## 9. API Design & Versioning

### 9.1 API Design Standards

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

### 9.2 API Versioning Policy

| Strategy | Implementation |
|----------|---------------|
| **Method** | URL path prefix (`/api/v1/`, `/api/v2/`) |
| **Breaking Change** | New major version; old version supported for 12 months |
| **Additive Change** | Same version; new optional fields, new endpoints |
| **Sunset Header** | `Sunset: Sat, 01 Jan 2028 00:00:00 GMT` on deprecated endpoints |
| **Discovery** | `GET /api/versions` returns all available versions and deprecation status |

### 9.3 Pagination, Filtering & Sorting

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

## 10. Middleware Implementation (Full Stack)

Every HTTP request to the Vesper API passes through an ordered middleware pipeline. The sections below describe what each middleware layer does, how to implement it, and where auth/authorization checks plug in.

### 10.1 Middleware Execution Order

```
Incoming HTTP Request
        │
        ▼
[MW-1]  CorrelationIdMiddleware    → Assign X-Correlation-ID for distributed tracing
        │
        ▼
[MW-2]  CORSMiddleware             → Dynamic per-tenant allowed origins
        │
        ▼
[MW-3]  RateLimiterMiddleware      → Redis sliding-window; 429 if exceeded
        │
        ▼
[MW-4]  TenantContextMiddleware    → Resolve tenant from domain/header; inject tenant_id
        │                            into request.state AND DB session context (RLS)
        ▼
[MW-5]  JWTAuthMiddleware          → Validate Bearer token (signature, expiry, audience)
        │                            Extract claims → request.state.user
        ▼
[MW-6]  SessionValidationMiddleware→ Confirm session not revoked (Redis lookup)
        │
        ▼
[MW-7]  PermissionMiddleware       → RBAC check via PermissionEngine (Redis-cached)
        │                            Decorators (@require_permission) on each endpoint
        ▼
[MW-8]  AuditMiddleware            → Async publish audit event after response
        │
        ▼
[MW-9]  ErrorHandlerMiddleware     → Catch all exceptions → structured JSON error
        │
        ▼
    Route Handler (FastAPI endpoint)
```

> **Skip Rules**: MW-4 through MW-7 are bypassed for public endpoints (`/api/v1/auth/login`, `/api/v1/tenants/resolve`, `/api/v1/health`, WebSocket handshake URL before token validation step).

---

### 10.2 MW-1: Correlation ID Middleware

**Purpose**: Attach a unique request ID to every request for distributed tracing across microservices and log correlation in Application Insights.

```python
# app/middleware/correlation_id.py
import uuid
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

class CorrelationIdMiddleware(BaseHTTPMiddleware):
    HEADER = "X-Correlation-ID"

    async def dispatch(self, request: Request, call_next):
        correlation_id = request.headers.get(self.HEADER) or str(uuid.uuid4())
        request.state.correlation_id = correlation_id

        response = await call_next(request)
        response.headers[self.HEADER] = correlation_id
        return response
```

**Registration** (`app/main.py`):
```python
app.add_middleware(CorrelationIdMiddleware)
```

---

### 10.3 MW-2: CORS Middleware (Per-Tenant Dynamic)

**Purpose**: Allow browser requests only from origins registered for a tenant. Prevents cross-tenant data leakage via browser-side attacks.

```python
# app/middleware/cors.py
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from app.services.tenant_service import TenantService

CORS_HEADERS = {
    "Access-Control-Allow-Credentials": "true",
    "Access-Control-Allow-Methods": "GET,POST,PUT,PATCH,DELETE,OPTIONS",
    "Access-Control-Allow-Headers": (
        "Authorization,Content-Type,X-Tenant-ID,X-Correlation-ID"
    ),
    "Access-Control-Max-Age": "600",
}

class TenantCORSMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        origin = request.headers.get("origin", "")

        # Preflight
        if request.method == "OPTIONS" and origin:
            tenant = await TenantService.resolve_by_origin(origin)
            if tenant and origin in tenant.allowed_origins:
                from starlette.responses import Response
                resp = Response(status_code=204)
                resp.headers["Access-Control-Allow-Origin"] = origin
                for k, v in CORS_HEADERS.items():
                    resp.headers[k] = v
                return resp

        response = await call_next(request)

        if origin:
            tenant = getattr(request.state, "tenant", None)
            if tenant and origin in tenant.allowed_origins:
                response.headers["Access-Control-Allow-Origin"] = origin
                for k, v in CORS_HEADERS.items():
                    response.headers[k] = v

        return response
```

**Tenant allowed origins** are stored in `tenant_configurations` with `config_key = 'cors.allowed_origins'` as a JSON array:
```json
["https://analytics.acme-corp.com", "https://acme.vesper-analytics.com"]
```

---

### 10.4 MW-3: Rate Limiter Middleware

**Purpose**: Prevent brute force, abuse, and overload. Uses Redis sorted sets for a sliding window algorithm.

```python
# app/middleware/rate_limiter.py
import time
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse
from app.services.cache_service import CacheService

RATE_LIMITS = {
    "auth.login":       {"requests": 5,   "window": 60},
    "auth.refresh":     {"requests": 10,  "window": 60},
    "ws.connect":       {"requests": 20,  "window": 60},
    "ai.chat":          {"requests": 30,  "window": 60},
    "export":           {"requests": 5,   "window": 300},
    "bulk_operations":  {"requests": 2,   "window": 60},
    "default":          {"requests": 100, "window": 60},
    "super_admin":      {"requests": 500, "window": 60},
}

def _classify_endpoint(path: str) -> str:
    if "/auth/login" in path:    return "auth.login"
    if "/auth/refresh" in path:  return "auth.refresh"
    if "/ws/" in path:           return "ws.connect"
    if "/chat" in path:          return "ai.chat"
    if "/export" in path:        return "export"
    if "/bulk" in path:          return "bulk_operations"
    return "default"

class RateLimiterMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, cache: CacheService):
        super().__init__(app)
        self.cache = cache

    async def dispatch(self, request: Request, call_next):
        # Identify caller — use IP before auth, user_id after
        user = getattr(request.state, "user", None)
        tenant = getattr(request.state, "tenant", None)
        identifier = (
            f"{tenant.id}:{user.id}" if (tenant and user)
            else request.client.host
        )

        endpoint_class = _classify_endpoint(request.url.path)
        # Super admins get elevated limits
        if user and user.user_type == "super_admin":
            endpoint_class = "super_admin"

        limit_cfg = RATE_LIMITS[endpoint_class]
        now = time.time()
        window_start = now - limit_cfg["window"]
        redis_key = f"rate:{identifier}:{endpoint_class}"

        # Sliding window: remove old entries, add current, count
        pipe = self.cache.pipeline()
        pipe.zremrangebyscore(redis_key, 0, window_start)
        pipe.zadd(redis_key, {str(now): now})
        pipe.zcard(redis_key)
        pipe.expire(redis_key, limit_cfg["window"])
        results = await pipe.execute()
        request_count = results[2]

        if request_count > limit_cfg["requests"]:
            return JSONResponse(
                status_code=429,
                content={
                    "status": "error",
                    "errors": [{"code": "RATE_LIMITED",
                                "message": "Too many requests. Please slow down."}]
                },
                headers={"Retry-After": str(limit_cfg["window"])},
            )

        response = await call_next(request)
        response.headers["X-RateLimit-Limit"] = str(limit_cfg["requests"])
        response.headers["X-RateLimit-Remaining"] = str(
            max(0, limit_cfg["requests"] - request_count)
        )
        return response
```

---

### 10.5 MW-4: Tenant Context Middleware

**Purpose**: Resolve which tenant this request belongs to. Injects tenant context into every DB session (required for SQL Row-Level Security).

```python
# app/middleware/tenant_context.py
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse
from app.services.tenant_service import TenantService

PUBLIC_PATHS = {
    "/api/v1/tenants/resolve",
    "/api/v1/auth/login",
    "/api/v1/auth/entra/authorize",
    "/api/v1/auth/entra/callback",
    "/api/v1/health",
    "/api/docs",
    "/api/openapi.json",
}

class TenantContextMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Skip for public paths
        if request.url.path in PUBLIC_PATHS:
            return await call_next(request)

        # Resolution priority:
        # 1. X-Tenant-ID header (internal service-to-service)
        # 2. Origin / Referer domain (browser requests)
        # 3. JWT claim 'tid' (extracted later in JWT middleware, but
        #    we do a best-effort resolution here for public endpoints)

        tenant_id = request.headers.get("X-Tenant-ID")
        tenant = None

        if tenant_id:
            tenant = await TenantService.get_by_id(tenant_id)
        else:
            origin = request.headers.get("origin") or request.headers.get("referer", "")
            if origin:
                domain = origin.split("//")[-1].split("/")[0].split(":")[0]
                tenant = await TenantService.resolve_by_domain(domain)

        if not tenant or tenant.status != "active":
            return JSONResponse(
                status_code=400,
                content={"status": "error",
                         "errors": [{"code": "TENANT_NOT_FOUND",
                                     "message": "Unable to resolve tenant context."}]},
            )

        request.state.tenant = tenant

        # Inject tenant_id into DB session context for SQL RLS
        # Done inside get_db() dependency by reading request.state.tenant
        return await call_next(request)
```

**DB Session with RLS Context** (`app/db/session.py`):
```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy import text

async def get_db(request: Request) -> AsyncSession:
    async with AsyncSessionLocal() as session:
        tenant = getattr(request.state, "tenant", None)
        user = getattr(request.state, "user", None)

        if tenant:
            # Set SQL Server session context for RLS policies
            await session.execute(
                text("EXEC sp_set_session_context @key=N'tenant_id', "
                     "@value=:tid, @read_only=1"),
                {"tid": str(tenant.id)}
            )
        if user and user.user_type == "super_admin":
            await session.execute(
                text("EXEC sp_set_session_context @key=N'is_super_admin', "
                     "@value=1, @read_only=1")
            )
        yield session
```

---

### 10.6 MW-5: JWT Authentication Middleware

**Purpose**: Validate the Bearer token on every protected request. Populates `request.state.user` with the decoded claims. No DB hit — purely cryptographic verification.

```python
# app/middleware/jwt_auth.py
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse
from app.core.security import JWTService

PUBLIC_PATHS = {
    "/api/v1/tenants/resolve",
    "/api/v1/auth/login",
    "/api/v1/auth/entra/authorize",
    "/api/v1/auth/entra/callback",
    "/api/v1/health",
}

class JWTAuthMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, jwt_service: JWTService):
        super().__init__(app)
        self.jwt = jwt_service

    async def dispatch(self, request: Request, call_next):
        if request.url.path in PUBLIC_PATHS:
            return await call_next(request)

        # WebSocket upgrade: token passed as query param ?token=...
        if request.headers.get("upgrade") == "websocket":
            token = request.query_params.get("token")
        else:
            auth_header = request.headers.get("Authorization", "")
            if not auth_header.startswith("Bearer "):
                return JSONResponse(
                    status_code=401,
                    content={"status": "error",
                             "errors": [{"code": "MISSING_TOKEN",
                                         "message": "Authorization header required."}]}
                )
            token = auth_header[7:]

        try:
            claims = self.jwt.decode_access_token(token)
        except TokenExpiredError:
            return JSONResponse(
                status_code=401,
                content={"status": "error",
                         "errors": [{"code": "TOKEN_EXPIRED",
                                     "message": "Access token has expired."}]}
            )
        except InvalidTokenError as e:
            return JSONResponse(
                status_code=401,
                content={"status": "error",
                         "errors": [{"code": "INVALID_TOKEN",
                                     "message": str(e)}]}
            )

        # Cross-verify tenant: token tid must match resolved tenant
        tenant = getattr(request.state, "tenant", None)
        if tenant and claims["tid"] != str(tenant.id):
            return JSONResponse(
                status_code=403,
                content={"status": "error",
                         "errors": [{"code": "TENANT_MISMATCH",
                                     "message": "Token tenant does not match request tenant."}]}
            )

        request.state.user = UserContext(**claims)
        return await call_next(request)
```

**JWT Service** (`app/core/security.py`):
```python
# app/core/security.py
import jwt
from datetime import datetime, timedelta, timezone
from cryptography.hazmat.primitives import serialization

class JWTService:
    ALGORITHM = "RS256"
    ACCESS_TOKEN_EXPIRE_MINUTES = 15
    REFRESH_TOKEN_EXPIRE_DAYS = 7

    def __init__(self, private_key: str, public_key: str):
        self._private_key = serialization.load_pem_private_key(
            private_key.encode(), password=None
        )
        self._public_key = serialization.load_pem_public_key(public_key.encode())

    def create_access_token(self, user: User, tenant: Tenant) -> str:
        now = datetime.now(timezone.utc)
        payload = {
            "sub":           str(user.id),
            "tid":           str(tenant.id),
            "email":         user.email,
            "name":          user.display_name,
            "type":          user.user_type,
            "roles":         [r.name for r in user.roles],
            "groups":        [g.name for g in user.groups],
            "auth_provider": user.auth_provider,
            "iat":           int(now.timestamp()),
            "exp":           int((now + timedelta(
                                 minutes=self.ACCESS_TOKEN_EXPIRE_MINUTES
                             )).timestamp()),
            "jti":           str(uuid.uuid4()),
            "iss":           "vesper-platform",
            "aud":           "vesper-api",
        }
        return jwt.encode(payload, self._private_key, algorithm=self.ALGORITHM)

    def decode_access_token(self, token: str) -> dict:
        return jwt.decode(
            token,
            self._public_key,
            algorithms=[self.ALGORITHM],
            audience="vesper-api",
            issuer="vesper-platform",
        )

    # JWKS endpoint — for Entra and external consumers to verify public key
    def get_jwks(self) -> dict:
        from jwt.algorithms import RSAAlgorithm
        return {"keys": [json.loads(RSAAlgorithm.to_jwk(self._public_key))]}
```

**Public key rotation**: Keys are stored in Azure Key Vault. Rotation is handled by publishing new `kid` (Key ID) to the JWKS endpoint at `GET /api/v1/auth/.well-known/jwks.json`. Old keys remain valid for their token lifetime (15 min) after rotation.

---

### 10.7 MW-6: Session Validation Middleware

**Purpose**: Ensure the session associated with the JWT has not been explicitly revoked (logout, admin action, security event). Redis lookup — O(1).

```python
# app/middleware/session_validation.py
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse
from app.services.cache_service import CacheService

class SessionValidationMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, cache: CacheService):
        super().__init__(app)
        self.cache = cache

    async def dispatch(self, request: Request, call_next):
        user = getattr(request.state, "user", None)
        if not user:
            return await call_next(request)

        # Check revocation list
        revoked_key = f"revoked_token:{user.jti}"
        if await self.cache.exists(revoked_key):
            return JSONResponse(
                status_code=401,
                content={"status": "error",
                         "errors": [{"code": "SESSION_REVOKED",
                                     "message": "Session has been revoked. Please log in again."}]}
            )

        # Verify session record exists in Redis (created at login)
        session_key = f"session:{user.sub}"
        session = await self.cache.get(session_key)
        if not session:
            return JSONResponse(
                status_code=401,
                content={"status": "error",
                         "errors": [{"code": "SESSION_NOT_FOUND",
                                     "message": "No active session found."}]}
            )

        return await call_next(request)

# On logout — revoke the current JTI so it cannot be reused within its TTL
async def revoke_token(jti: str, exp: int, cache: CacheService):
    ttl = max(0, exp - int(time.time()))
    await cache.set(f"revoked_token:{jti}", "1", ttl=ttl)
```

---

### 10.8 MW-7: Permission Middleware & Endpoint Decorators

**Purpose**: RBAC enforcement. Two complementary mechanisms work together:

1. **Middleware** loads the permission engine into `request.state`
2. **Endpoint decorator** `@require_permission(resource, action)` performs the actual check

```python
# app/middleware/permission.py
from starlette.middleware.base import BaseHTTPMiddleware
from app.core.permissions import PermissionEngine
from app.services.cache_service import CacheService

class PermissionMiddleware(BaseHTTPMiddleware):
    """
    Attaches a PermissionEngine instance to every request.
    The actual check is done by @require_permission decorators on endpoints.
    This avoids running the check for endpoints that don't need it.
    """
    def __init__(self, app, cache: CacheService, db_factory):
        super().__init__(app)
        self.cache = cache
        self.db_factory = db_factory

    async def dispatch(self, request: Request, call_next):
        async with self.db_factory() as db:
            request.state.permission_engine = PermissionEngine(db, self.cache)
            return await call_next(request)
```

```python
# app/api/deps.py
from functools import wraps
from fastapi import HTTPException, status

def require_permission(resource: str, action: str):
    """
    FastAPI endpoint decorator for RBAC enforcement.
    Usage:
        @router.get("/reports")
        @require_permission("reports", "view")
        async def list_reports(request: Request): ...
    """
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            request: Request = kwargs.get("request") or next(
                (a for a in args if isinstance(a, Request)), None
            )
            user = request.state.user
            engine: PermissionEngine = request.state.permission_engine

            result = await engine.check_permission(
                user_id=str(user.sub),
                tenant_id=str(user.tid),
                resource_key=resource,
                action_key=action,
            )

            if not result.allowed:
                raise HTTPException(
                    status_code=status.HTTP_403_FORBIDDEN,
                    detail={
                        "code": "PERMISSION_DENIED",
                        "message": f"You do not have '{action}' permission on '{resource}'.",
                        "resource": resource,
                        "action": action,
                    }
                )
            return await func(*args, **kwargs)
        return wrapper
    return decorator

def require_role(*roles: str):
    """Simpler role-based guard for admin endpoints."""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            request: Request = kwargs.get("request") or next(
                (a for a in args if isinstance(a, Request)), None
            )
            user = request.state.user
            if not any(r in user.roles for r in roles):
                raise HTTPException(status_code=403, detail={
                    "code": "ROLE_REQUIRED",
                    "message": f"Requires one of roles: {roles}",
                })
            return await func(*args, **kwargs)
        return wrapper
    return decorator

# Example endpoint combining both
@router.delete("/users/{user_id}")
@require_permission("admin.users", "delete")
async def delete_user(user_id: str, request: Request, db=Depends(get_db)):
    """Requires both 'admin.users:delete' permission AND tenant_admin role."""
    ...

@router.post("/tenants")
@require_role("super_admin")
async def create_tenant(request: Request, ...):
    """Super admin only endpoint."""
    ...
```

**Authorization Check Flow in Detail**:
```
@require_permission("reports.powerbi", "view")
    │
    ├─ Cache lookup: perm:tenant_id:user_id:reports.powerbi:view  (TTL 5 min)
    │    ├─ HIT  → return cached result (sub-millisecond)
    │    └─ MISS → DB query:
    │         ├─ Collect all role_ids for user (direct + group-inherited)
    │         ├─ Find permission_id for (resource=reports.powerbi, action=view)
    │         ├─ Check role_permission_map for ALLOW or DENY grants
    │         ├─ Apply precedence: DENY > ALLOW > default-deny
    │         └─ If still denied, check parent resource (reports → view)
    │
    └─ Result stored in Redis
```

---

### 10.9 MW-8: Audit Logger Middleware

**Purpose**: Asynchronously record all state-changing operations without blocking the response.

```python
# app/middleware/audit.py
from starlette.middleware.base import BaseHTTPMiddleware
import asyncio

AUDITABLE_METHODS = {"POST", "PUT", "PATCH", "DELETE"}

class AuditMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, broker):
        super().__init__(app)
        self.broker = broker  # Azure Service Bus / RabbitMQ publisher

    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)

        if request.method in AUDITABLE_METHODS and response.status_code < 400:
            user = getattr(request.state, "user", None)
            tenant = getattr(request.state, "tenant", None)
            if user and tenant:
                # Fire-and-forget: do not await, don't slow response
                asyncio.create_task(self._publish_audit(request, response, user, tenant))

        return response

    async def _publish_audit(self, request, response, user, tenant):
        event = {
            "event_type":     "audit.action",
            "tenant_id":      str(tenant.id),
            "user_id":        str(user.sub),
            "method":         request.method,
            "path":           str(request.url.path),
            "status_code":    response.status_code,
            "correlation_id": getattr(request.state, "correlation_id", None),
            "ip_address":     request.client.host,
            "user_agent":     request.headers.get("user-agent"),
            "timestamp":      datetime.utcnow().isoformat(),
        }
        await self.broker.publish("audit.events", event)
```

---

### 10.10 MW-9: Global Error Handler

**Purpose**: Convert all unhandled exceptions to the standard response envelope. No stack traces leak to clients.

```python
# app/middleware/error_handler.py
from fastapi import Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError
from app.core.exceptions import (
    PermissionDeniedError, TenantNotFoundError,
    ResourceNotFoundError, ConflictError
)

def register_exception_handlers(app):

    @app.exception_handler(RequestValidationError)
    async def validation_error(request: Request, exc: RequestValidationError):
        return JSONResponse(status_code=422, content={
            "status": "error",
            "errors": [
                {"code": "VALIDATION_ERROR", "field": ".".join(str(l) for l in e["loc"][1:]),
                 "message": e["msg"]}
                for e in exc.errors()
            ]
        })

    @app.exception_handler(PermissionDeniedError)
    async def permission_denied(request: Request, exc: PermissionDeniedError):
        return JSONResponse(status_code=403, content={
            "status": "error",
            "errors": [{"code": "PERMISSION_DENIED", "message": str(exc),
                        "resource": exc.resource, "action": exc.action}]
        })

    @app.exception_handler(ResourceNotFoundError)
    async def not_found(request: Request, exc: ResourceNotFoundError):
        return JSONResponse(status_code=404, content={
            "status": "error",
            "errors": [{"code": "NOT_FOUND", "message": str(exc)}]
        })

    @app.exception_handler(Exception)
    async def generic_error(request: Request, exc: Exception):
        # Log full traceback to Application Insights
        logger.exception("Unhandled exception", extra={
            "correlation_id": getattr(request.state, "correlation_id", ""),
        })
        return JSONResponse(status_code=500, content={
            "status": "error",
            "errors": [{"code": "INTERNAL_ERROR",
                        "message": "An internal error occurred. Please try again."}]
        })
```

---

## 11. Caching, Messaging & Broker Architecture

### 11.1 Redis Caching Strategy

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

### 11.2 Message Broker Architecture

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

### 11.3 Inter-Service Communication Pattern

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

---

## PART V — PLATFORM FEATURES

*This part covers four advanced platform capabilities: real-time WebSocket communication for notifications and AI chat (§12), the multi-agent AI Chat system built on LangChain and LangGraph (§13), the Power BI embedded analytics system with backend-controlled dynamic menus (§14), and the file upload service supporting both SharePoint and Azure OneLake destinations with per-page routing and chunked streaming (§15). Each section follows the same structure: architecture overview → database schema → backend service → API endpoints.*

---

## 12. WebSocket Architecture (Notifications & Chat)

### 12.1 Overview

Two distinct WebSocket channels are maintained per authenticated user:

| Channel | Path | Purpose |
|---------|------|---------|
| **Notification WS** | `/ws/v1/notifications` | Server-push: system alerts, permission changes, report updates |
| **Chat Agent WS** | `/ws/v1/chat` | Bidirectional: user queries → AI agent → streaming responses |

```
Browser
  │
  ├── REST API (HTTP)         → All CRUD, menu, Power BI token APIs
  │
  ├── WS /ws/v1/notifications → Server pushes JSON events (one-way from server)
  │
  └── WS /ws/v1/chat          → Full duplex: user sends query, server streams AI response
```

### 12.2 WebSocket Authentication Flow

WebSocket connections carry the access token in the URL query string because browser WebSocket API does not support custom headers.

```
Client → GET /ws/v1/notifications?token=<access_token>
              │
              ▼
         FastAPI WebSocket endpoint
              │
         [1] Extract token from query params
         [2] JWTService.decode_access_token(token) → claims
         [3] TenantService.resolve(claims.tid) → tenant
         [4] SessionValidation — check Redis for revocation
         [5] PermissionEngine.check("notifications", "view")
              │
         [OK] → Accept WebSocket upgrade; register connection
         [FAIL]→ Close(code=4001, reason="Unauthorized")
```

**Connection Manager** (shared across both channels):

```python
# app/core/websocket_manager.py
import asyncio
from collections import defaultdict
from typing import Dict, Set
from fastapi import WebSocket

class ConnectionManager:
    """
    Manages all active WebSocket connections.
    Connections are indexed by tenant_id and user_id for targeted delivery.
    """

    def __init__(self):
        # { tenant_id: { user_id: Set[WebSocket] } }
        self._connections: Dict[str, Dict[str, Set[WebSocket]]] = defaultdict(
            lambda: defaultdict(set)
        )
        self._lock = asyncio.Lock()

    async def connect(self, websocket: WebSocket, tenant_id: str, user_id: str):
        await websocket.accept()
        async with self._lock:
            self._connections[tenant_id][user_id].add(websocket)

    async def disconnect(self, websocket: WebSocket, tenant_id: str, user_id: str):
        async with self._lock:
            self._connections[tenant_id][user_id].discard(websocket)

    async def send_to_user(self, tenant_id: str, user_id: str, message: dict):
        """Push to all active connections of a specific user."""
        sockets = self._connections.get(tenant_id, {}).get(user_id, set()).copy()
        dead = set()
        for ws in sockets:
            try:
                await ws.send_json(message)
            except Exception:
                dead.add(ws)
        for ws in dead:
            await self.disconnect(ws, tenant_id, user_id)

    async def broadcast_to_tenant(self, tenant_id: str, message: dict):
        """Push to all users in a tenant (e.g., permission change broadcast)."""
        tenant_sockets = self._connections.get(tenant_id, {})
        for user_id in list(tenant_sockets.keys()):
            await self.send_to_user(tenant_id, user_id, message)

# Global singleton
ws_manager = ConnectionManager()
```

---

### 12.3 Notification WebSocket

**Endpoint** (`app/api/v1/websocket.py`):

```python
# app/api/v1/websocket.py
from fastapi import APIRouter, WebSocket, WebSocketDisconnect, Query
from app.core.websocket_manager import ws_manager
from app.core.security import JWTService
from app.services.session_service import SessionService

router = APIRouter()

@router.websocket("/ws/v1/notifications")
async def notifications_ws(
    websocket: WebSocket,
    token: str = Query(..., description="JWT access token"),
):
    # --- Auth ---
    try:
        claims = jwt_service.decode_access_token(token)
    except Exception:
        await websocket.close(code=4001, reason="Invalid or expired token")
        return

    if await session_service.is_revoked(claims["jti"]):
        await websocket.close(code=4001, reason="Session revoked")
        return

    tenant_id = claims["tid"]
    user_id   = claims["sub"]

    # --- Connect ---
    await ws_manager.connect(websocket, tenant_id, user_id)

    # Send connection acknowledgement
    await websocket.send_json({
        "type":    "connection.established",
        "user_id": user_id,
        "message": "Connected to Vesper Notification Service",
    })

    try:
        # Heartbeat loop — keep connection alive
        # Server only pushes; client sends ping to maintain connection
        while True:
            data = await websocket.receive_text()
            if data == "ping":
                await websocket.send_text("pong")
            # Ignore any other client messages on this channel

    except WebSocketDisconnect:
        await ws_manager.disconnect(websocket, tenant_id, user_id)
```

**Notification Payload Schema**:
```json
{
  "type": "notification.info | notification.warning | notification.error | permission.changed | report.ready | system.maintenance",
  "id": "uuid",
  "title": "Report Generated",
  "message": "Your Q4 Sales report is ready to download.",
  "action_url": "/reports/q4-sales-2025",
  "severity": "info | warning | error",
  "metadata": {},
  "timestamp": "2026-02-26T10:30:00Z",
  "read": false
}
```

**Notification Types**:

| Type | Trigger | Target |
|------|---------|--------|
| `notification.info` | Report ready, sync complete | Specific user |
| `notification.warning` | Password expiry, quota near | Specific user |
| `notification.error` | Sync failed, embed error | Specific user or admin |
| `permission.changed` | Role/group updated | Affected user(s) |
| `report.ready` | Power BI report embed ready | Specific user |
| `system.maintenance` | Scheduled downtime | Entire tenant |
| `chat.response` | AI agent response chunk | Specific user |

**Pushing from Services**:
```python
# Any backend service can push to a user via the manager
await ws_manager.send_to_user(
    tenant_id=tenant_id,
    user_id=user_id,
    message={
        "type": "report.ready",
        "id": str(uuid.uuid4()),
        "title": "Power BI Report Ready",
        "message": "Your embedded report token has been refreshed.",
        "metadata": {"report_id": report_id},
        "timestamp": datetime.utcnow().isoformat(),
    }
)
```

---

### 12.4 Database Schema for Notifications

```sql
-- ═══════════════════════════════════════════════════════════════
-- NOTIFICATION SYSTEM
-- ═══════════════════════════════════════════════════════════════

CREATE TABLE notifications (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    tenant_id       UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    user_id         UNIQUEIDENTIFIER NULL REFERENCES users(id),  -- NULL = broadcast to tenant
    notification_type VARCHAR(50)    NOT NULL,
    title           NVARCHAR(200)   NOT NULL,
    message         NVARCHAR(1000)  NOT NULL,
    action_url      VARCHAR(500)    NULL,
    severity        VARCHAR(20)     NOT NULL DEFAULT 'info'
                      CHECK (severity IN ('info','warning','error','success')),
    metadata        NVARCHAR(MAX)   NULL,        -- JSON: extra context
    is_read         BIT             NOT NULL DEFAULT 0,
    read_at         DATETIME2       NULL,
    expires_at      DATETIME2       NULL,        -- Auto-cleanup old notifications
    created_at      DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME()
);
CREATE INDEX ix_notifications_user ON notifications(tenant_id, user_id, is_read, created_at DESC);
```

**Notification REST APIs** (complement WebSocket push):

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/api/v1/notifications` | List user's notifications (paginated) | Bearer |
| `PATCH` | `/api/v1/notifications/{id}/read` | Mark notification as read | Bearer |
| `POST` | `/api/v1/notifications/read-all` | Mark all as read | Bearer |
| `DELETE` | `/api/v1/notifications/{id}` | Delete notification | Bearer |
| `GET` | `/api/v1/notifications/unread-count` | Unread badge count | Bearer |

---


### 12.7 WebSocket Token Expiry and Reconnection

WebSocket access tokens share the same 15-minute TTL as REST API tokens. Unlike REST requests — which silently refresh via an Axios interceptor — an open WebSocket connection cannot swap its authentication token mid-session. The client must proactively reconnect with a fresh token before expiry.

**Reconnection sequence:**

```
[T + 0m]  Client opens: WSS /ws/v1/notifications?token=<access_token>
           Server validates token at handshake → sends {type: "connection.established"}

[T +13m]  Client timer fires (2 min before expiry):
           1. POST /api/v1/auth/refresh  — HttpOnly refresh cookie sent automatically
           2. Receive new access_token   — expires_in: 900
           3. ws.close(1000, "token_refresh")  — graceful close
           4. ws = new WebSocket("/ws/v1/notifications?token=<new_access_token>")
           5. Server validates new token → sends {type: "connection.established"}

[T +28m]  Repeat the cycle
```

**Client-side pattern:**

```typescript
// Integrated into useNotificationSocket hook (see Section 16.4)
function scheduleWsTokenRefresh(
  expiresIn: number,
  reconnect: () => void
): ReturnType<typeof setTimeout> {
  const delayMs = Math.max(0, (expiresIn - 120) * 1000); // 2 min buffer
  return setTimeout(async () => {
    try {
      await authService.refresh();   // new access_token stored in memory
      reconnect();                    // close old WS, open fresh connection
    } catch {
      authService.logout();           // refresh failed → force re-login
    }
  }, delayMs);
}
```

**Server behaviour:** Token validation occurs only at the WebSocket handshake (`ws.accept()`). An already-authenticated, open connection is **not** forcibly closed when the token's expiry time passes — the connection continues until the client closes it or a network event drops it. A new connection attempt with an expired token will be rejected (code 4001).

## 13. AI Chat Agent — LangChain & LangGraph

### 13.1 Architecture Overview

The AI Chat Agent allows users to ask natural language questions about their business data. It routes queries through a multi-agent orchestration layer built with **LangGraph**, calls specialized sub-agents (SQL, BI, summarization), and streams responses back via WebSocket.

```
User (WebSocket)
    │  "What were the top 5 products by revenue last quarter?"
    ▼
Chat WebSocket Endpoint (/ws/v1/chat)
    │
    ▼
Chat Session Manager
    │  Load conversation history from Redis/DB
    ▼
LangGraph Orchestrator Agent
    │
    ├──► Intent Classifier Agent
    │       └─► Determines: "sql_query | powerbi_query | general_insights | system_help"
    │
    ├──► [if sql_query]    SQL Agent
    │       ├─ Generate SQL from natural language (LLM)
    │       ├─ Validate & sanitize (read-only, tenant-scoped)
    │       ├─ Execute against Azure SQL (via read replica)
    │       └─ Format results as table/JSON
    │
    ├──► [if powerbi_query] BI Insights Agent
    │       ├─ Identify relevant Power BI report/dataset
    │       ├─ Call Power BI REST API for embedded data
    │       └─ Summarize using LLM
    │
    ├──► [if general_insights] Analytics Aggregation Agent
    │       ├─ Query pre-computed analytics from Redis/SQL
    │       └─ Format insights with trend analysis
    │
    └──► Response Synthesizer Agent
            ├─ Combine all sub-agent outputs
            ├─ Generate natural language summary (LLM streaming)
            └─ Stream chunks via WebSocket → user

Each chunk pushed via ws_manager.send_to_user() as it is generated
```

### 13.2 Database Schema for Chat

```sql
-- ═══════════════════════════════════════════════════════════════
-- AI CHAT SESSIONS & HISTORY
-- ═══════════════════════════════════════════════════════════════

CREATE TABLE chat_sessions (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    tenant_id       UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    user_id         UNIQUEIDENTIFIER NOT NULL REFERENCES users(id),
    session_title   NVARCHAR(200)   NULL,           -- Auto-generated from first message
    status          VARCHAR(20)     NOT NULL DEFAULT 'active'
                      CHECK (status IN ('active','archived','deleted')),
    created_at      DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME(),
    updated_at      DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME()
);

CREATE TABLE chat_messages (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    session_id      UNIQUEIDENTIFIER NOT NULL REFERENCES chat_sessions(id) ON DELETE CASCADE,
    tenant_id       UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    user_id         UNIQUEIDENTIFIER NOT NULL REFERENCES users(id),
    role            VARCHAR(20)     NOT NULL CHECK (role IN ('user','assistant','system','tool')),
    content         NVARCHAR(MAX)   NOT NULL,
    tool_calls      NVARCHAR(MAX)   NULL,           -- JSON: tool invocations made by agent
    tool_results    NVARCHAR(MAX)   NULL,           -- JSON: results returned to agent
    tokens_used     INT             NULL,
    model_name      VARCHAR(100)    NULL,           -- e.g. 'claude-sonnet-4-6'
    latency_ms      INT             NULL,
    created_at      DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME()
);
CREATE INDEX ix_chat_messages_session ON chat_messages(session_id, created_at);

CREATE TABLE chat_agent_tools (
    id              UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    tenant_id       UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    tool_name       VARCHAR(100)    NOT NULL,        -- 'sql_query', 'powerbi_data', 'trend_analysis'
    is_enabled      BIT             NOT NULL DEFAULT 1,
    config          NVARCHAR(MAX)   NULL,            -- JSON: tool-specific config per tenant
    CONSTRAINT uq_tenant_tool UNIQUE (tenant_id, tool_name)
);
```

---

### 13.3 Chat WebSocket Endpoint

```python
# app/api/v1/chat_ws.py
from fastapi import APIRouter, WebSocket, WebSocketDisconnect, Query
from app.agents.orchestrator import ChatOrchestrator
from app.core.websocket_manager import ws_manager

router = APIRouter()

@router.websocket("/ws/v1/chat")
async def chat_ws(
    websocket: WebSocket,
    token: str = Query(...),
    session_id: str = Query(None, description="Resume existing chat session"),
):
    # --- Auth (same as notifications) ---
    try:
        claims = jwt_service.decode_access_token(token)
    except Exception:
        await websocket.close(code=4001, reason="Unauthorized")
        return

    # --- Permission check: user must have 'ai.chat:use' permission ---
    engine = PermissionEngine(await get_db_session(), cache)
    result = await engine.check_permission(
        user_id=claims["sub"],
        tenant_id=claims["tid"],
        resource_key="ai.chat",
        action_key="use",
    )
    if not result.allowed:
        await websocket.close(code=4003, reason="Permission denied: ai.chat:use")
        return

    tenant_id = claims["tid"]
    user_id   = claims["sub"]

    # Create or resume chat session
    session = await chat_service.get_or_create_session(
        tenant_id=tenant_id,
        user_id=user_id,
        session_id=session_id,
    )

    await ws_manager.connect(websocket, tenant_id, user_id)
    await websocket.send_json({
        "type":       "chat.session.started",
        "session_id": str(session.id),
        "message":    "Connected to Vesper AI Assistant",
    })

    orchestrator = ChatOrchestrator(
        tenant_id=tenant_id,
        user_id=user_id,
        session_id=str(session.id),
        websocket=websocket,
    )

    try:
        while True:
            raw = await websocket.receive_json()
            msg_type = raw.get("type")

            if msg_type == "chat.message":
                # Non-blocking: stream response chunks as they arrive
                await orchestrator.handle_message(
                    content=raw["content"],
                    attachments=raw.get("attachments", []),
                )

            elif msg_type == "chat.stop":
                await orchestrator.stop_generation()

            elif msg_type == "ping":
                await websocket.send_json({"type": "pong"})

    except WebSocketDisconnect:
        await ws_manager.disconnect(websocket, tenant_id, user_id)
```

**Message Protocol (Client → Server)**:
```json
{
  "type": "chat.message",
  "content": "What were the top 5 products by revenue last quarter?",
  "attachments": []
}
```

**Message Protocol (Server → Client, streaming)**:
```json
// Chunk (streamed as LLM generates)
{"type": "chat.chunk", "session_id": "...", "content": "The top 5 products", "done": false}
{"type": "chat.chunk", "session_id": "...", "content": " by revenue last quarter were:", "done": false}

// Tool invocation notification (visible to user as "Thinking...")
{"type": "chat.tool_call", "tool": "sql_query", "status": "running", "description": "Querying sales database..."}
{"type": "chat.tool_call", "tool": "sql_query", "status": "done", "result_summary": "7 rows returned"}

// Final completion
{"type": "chat.done", "session_id": "...", "message_id": "...", "tokens_used": 423}

// Error
{"type": "chat.error", "code": "TOOL_ERROR", "message": "Failed to query database"}
```

---

### 13.4 LangGraph Orchestrator

```python
# app/agents/orchestrator.py
from langchain_anthropic import ChatAnthropic
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from app.agents.tools import (
    sql_query_tool,
    powerbi_data_tool,
    trend_analysis_tool,
    general_knowledge_tool,
)

# ── State definition ──────────────────────────────────────────
from typing import TypedDict, Annotated, Sequence
from langchain_core.messages import BaseMessage, HumanMessage, AIMessage
import operator

class AgentState(TypedDict):
    messages:       Annotated[Sequence[BaseMessage], operator.add]
    tenant_id:      str
    user_id:        str
    session_id:     str
    intent:         str | None
    tool_outputs:   list
    final_response: str | None

# ── LLM ──────────────────────────────────────────────────────
llm = ChatAnthropic(
    model="claude-sonnet-4-6",
    api_key=settings.ANTHROPIC_API_KEY,
    streaming=True,
    max_tokens=4096,
)

# ── Tools ─────────────────────────────────────────────────────
tools = [
    sql_query_tool,
    powerbi_data_tool,
    trend_analysis_tool,
    general_knowledge_tool,
]
llm_with_tools = llm.bind_tools(tools)

# ── Graph Nodes ───────────────────────────────────────────────

async def intent_classifier(state: AgentState) -> AgentState:
    """Classify the user's intent to route to the right agent."""
    system = """You are a business analytics assistant for Vesper Analytics.
    Classify the user's intent as one of:
    - sql_query: Needs live data from the database
    - powerbi_insight: Needs insights from Power BI reports
    - trend_analysis: Needs trend/comparison analysis
    - general: General business question answerable from context

    Respond with just the classification label."""

    last_msg = state["messages"][-1].content
    classification = await llm.ainvoke([
        {"role": "system", "content": system},
        {"role": "user",   "content": last_msg},
    ])
    return {"intent": classification.content.strip()}


async def agent_node(state: AgentState) -> AgentState:
    """Main reasoning agent with access to all tools."""
    system = f"""You are Vesper AI, a business intelligence assistant.
    Tenant ID: {state['tenant_id']}
    You have access to tools to query the database, retrieve Power BI insights,
    and perform trend analysis. Always scope queries to the current tenant.
    Be concise, accurate, and cite your sources (tool results).
    Today's date: {datetime.utcnow().strftime('%Y-%m-%d')}"""

    messages = [{"role": "system", "content": system}] + [
        {"role": m.type, "content": m.content} for m in state["messages"]
    ]
    response = await llm_with_tools.ainvoke(messages)
    return {"messages": [response]}


def should_continue(state: AgentState) -> str:
    """Decide whether to call tools or finish."""
    last = state["messages"][-1]
    if hasattr(last, "tool_calls") and last.tool_calls:
        return "tools"
    return END


# ── Graph Assembly ─────────────────────────────────────────────
def build_agent_graph() -> StateGraph:
    graph = StateGraph(AgentState)

    graph.add_node("intent_classifier", intent_classifier)
    graph.add_node("agent",             agent_node)
    graph.add_node("tools",             ToolNode(tools))

    graph.set_entry_point("intent_classifier")
    graph.add_edge("intent_classifier", "agent")
    graph.add_conditional_edges("agent", should_continue, {
        "tools": "tools",
        END:     END,
    })
    graph.add_edge("tools", "agent")  # Loop: tools → agent → tools (until done)

    return graph.compile()

# ── Orchestrator class ─────────────────────────────────────────
class ChatOrchestrator:
    _graph = build_agent_graph()

    def __init__(self, tenant_id, user_id, session_id, websocket):
        self.tenant_id  = tenant_id
        self.user_id    = user_id
        self.session_id = session_id
        self.ws         = websocket

    async def handle_message(self, content: str, attachments: list):
        # Load conversation history
        history = await chat_service.get_history(self.session_id, limit=20)

        initial_state: AgentState = {
            "messages":       history + [HumanMessage(content=content)],
            "tenant_id":      self.tenant_id,
            "user_id":        self.user_id,
            "session_id":     self.session_id,
            "intent":         None,
            "tool_outputs":   [],
            "final_response": None,
        }

        # Stream through LangGraph
        async for chunk in self._graph.astream(initial_state):
            node_name = list(chunk.keys())[0]
            node_output = chunk[node_name]

            if node_name == "tools":
                # Notify client about tool invocations
                for tool_msg in node_output.get("messages", []):
                    await self.ws.send_json({
                        "type":        "chat.tool_call",
                        "tool":        tool_msg.name,
                        "status":      "done",
                        "result_summary": str(tool_msg.content)[:200],
                    })

            elif node_name == "agent":
                # Stream LLM response text chunks
                for msg in node_output.get("messages", []):
                    if hasattr(msg, "content") and msg.content:
                        await self.ws.send_json({
                            "type":    "chat.chunk",
                            "session_id": self.session_id,
                            "content": msg.content,
                            "done":    False,
                        })

        # Save to DB
        await chat_service.save_message(
            session_id=self.session_id,
            role="assistant",
            content=final_content,
        )

        await self.ws.send_json({
            "type":       "chat.done",
            "session_id": self.session_id,
        })
```

---

### 13.5 LangChain Tool Implementations

```python
# app/agents/tools.py
from langchain_core.tools import tool
from app.db.read_replica import get_read_db
from app.core.sql_validator import validate_safe_sql

@tool
async def sql_query_tool(
    query: str,
    tenant_id: str,
    description: str = "Execute a read-only SQL query against the tenant's business data.",
) -> str:
    """
    Execute a natural language query translated to safe, read-only SQL.
    Always scoped to the current tenant via tenant_id.

    Args:
        query: The SQL SELECT statement to execute (no DDL/DML allowed)
        tenant_id: Tenant scope for Row-Level Security

    Returns:
        JSON-formatted query results (max 1000 rows)
    """
    # Safety: validate SQL is SELECT-only, no system tables, no cross-tenant refs
    if not validate_safe_sql(query):
        return "Error: Only SELECT statements on business tables are permitted."

    async with get_read_db(tenant_id) as db:
        result = await db.execute(text(query))
        rows = result.mappings().fetchmany(1000)
        return json.dumps([dict(r) for r in rows], default=str)


@tool
async def powerbi_data_tool(
    report_name: str,
    metric: str,
    tenant_id: str,
    description: str = "Retrieve summary metrics from a Power BI report dataset.",
) -> str:
    """
    Fetch aggregated data from a specific Power BI report.

    Args:
        report_name: The Power BI report name (e.g., 'Sales Performance')
        metric: The metric to retrieve (e.g., 'total_revenue', 'top_products')
        tenant_id: Tenant scope

    Returns:
        JSON with metric value and context
    """
    report = await powerbi_service.get_report_by_name(tenant_id, report_name)
    if not report:
        return f"Report '{report_name}' not found."

    data = await powerbi_service.get_dataset_values(
        tenant_id=tenant_id,
        dataset_id=report.dataset_id,
        metric=metric,
    )
    return json.dumps(data, default=str)


@tool
async def trend_analysis_tool(
    metric: str,
    period: str,
    tenant_id: str,
    description: str = "Analyze trends for a business metric over a time period.",
) -> str:
    """
    Perform trend analysis on pre-computed analytics data.

    Args:
        metric: Metric name (e.g., 'monthly_revenue', 'user_signups')
        period: Time period ('last_7d', 'last_30d', 'last_quarter', 'last_year')
        tenant_id: Tenant scope

    Returns:
        JSON with trend data, percentage change, and statistical summary
    """
    data = await analytics_service.get_trend(tenant_id, metric, period)
    return json.dumps(data, default=str)
```

---

### 13.6 Chat REST APIs (Session Management)

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `GET` | `/api/v1/chat/sessions` | List user's chat sessions | Bearer |
| `POST` | `/api/v1/chat/sessions` | Create new chat session | Bearer |
| `GET` | `/api/v1/chat/sessions/{id}` | Get session details | Bearer |
| `DELETE` | `/api/v1/chat/sessions/{id}` | Archive/delete session | Bearer |
| `GET` | `/api/v1/chat/sessions/{id}/messages` | Get full message history | Bearer |
| `GET` | `/api/v1/chat/tools` | List available agent tools for tenant | Bearer |
| `PUT` | `/api/v1/chat/tools/{tool_name}` | Enable/disable tool (tenant admin) | Bearer + `admin.ai:configure` |

---

## 14. Power BI Integration & Backend-Controlled Menu System

### 14.1 Design Philosophy

The backend is the **sole authority** on:
1. Which menu items a user sees
2. Which Power BI reports map to each menu item
3. The embed token required to render each report
4. Sub-menu structure and ordering
5. Whether an item is visible, disabled, or conditionally shown

The frontend **never** hardcodes menu items or Power BI URLs. It renders whatever the backend returns.

```
Backend Controls:
  menu_items table  → defines the navigation tree
  powerbi_reports   → maps menu items to Power BI workspaces + report IDs
  role_permission_map → gates each menu item behind RBAC
  PowerBI API call  → generates time-limited embed tokens per user

Frontend Flow:
  GET /api/v1/navigation/menu
    → Returns: filtered menu tree (only items user can see)
    → Each menu item has: label, icon, route, type, sub_items[]
    → If type = 'powerbi': also includes embed_config reference

  GET /api/v1/powerbi/reports/{menu_item_id}/embed-token
    → Returns: embed_url, embed_token, token_expiry
    → Frontend renders <PowerBIEmbed> component with these values
```

---

### 14.2 Database Schema for Menu & Power BI

```sql
-- ═══════════════════════════════════════════════════════════════
-- MENU & NAVIGATION SYSTEM
-- ═══════════════════════════════════════════════════════════════

CREATE TABLE menu_items (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    tenant_id           UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    parent_id           UNIQUEIDENTIFIER NULL REFERENCES menu_items(id), -- NULL = top-level
    label               NVARCHAR(100)   NOT NULL,
    icon                VARCHAR(100)    NULL,         -- Icon name from icon library
    route               VARCHAR(200)    NULL,         -- Frontend route (e.g. '/reports/sales')
    item_type           VARCHAR(30)     NOT NULL DEFAULT 'page'
                          CHECK (item_type IN (
                              'page',          -- Standard React page
                              'powerbi',       -- Embedded Power BI report
                              'powerbi_group', -- Group of Power BI reports
                              'external_link', -- External URL
                              'divider',       -- Visual separator
                              'group_header'   -- Non-clickable section label
                          )),
    sort_order          INT             NOT NULL DEFAULT 0,
    is_active           BIT             NOT NULL DEFAULT 1,
    is_new_tab          BIT             NOT NULL DEFAULT 0,   -- Open in new browser tab
    badge_text          NVARCHAR(20)    NULL,                 -- e.g. 'NEW', 'BETA'
    tooltip             NVARCHAR(300)   NULL,
    -- RBAC: the resource_key that must have 'view' permission for this item to appear
    resource_key        VARCHAR(100)    NOT NULL,             -- Links to resources table
    created_at          DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME(),
    updated_at          DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME(),
    created_by          UNIQUEIDENTIFIER NULL
);
CREATE INDEX ix_menu_items_tenant ON menu_items(tenant_id, parent_id, sort_order)
    WHERE is_active = 1;

-- ═══════════════════════════════════════════════════════════════
-- POWER BI REPORT REGISTRY
-- ═══════════════════════════════════════════════════════════════

CREATE TABLE powerbi_reports (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    tenant_id           UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    menu_item_id        UNIQUEIDENTIFIER NOT NULL REFERENCES menu_items(id),
    display_name        NVARCHAR(200)   NOT NULL,
    description         NVARCHAR(500)   NULL,

    -- Power BI service references
    workspace_id        VARCHAR(100)    NOT NULL,    -- Power BI Workspace (Group) ID
    report_id           VARCHAR(100)    NOT NULL,    -- Power BI Report ID
    dataset_id          VARCHAR(100)    NULL,        -- Power BI Dataset ID (for Q&A)
    page_name           VARCHAR(100)    NULL,        -- Specific report page (optional)

    -- Embed configuration
    embed_type          VARCHAR(30)     NOT NULL DEFAULT 'report'
                          CHECK (embed_type IN ('report','dashboard','tile','visual')),
    filter_pane_enabled BIT             NOT NULL DEFAULT 1,
    nav_pane_enabled    BIT             NOT NULL DEFAULT 0,
    default_filters     NVARCHAR(MAX)   NULL,        -- JSON: pre-applied filters
    rls_roles           NVARCHAR(MAX)   NULL,        -- JSON: Power BI RLS roles to apply

    -- Token lifecycle
    token_ttl_minutes   INT             NOT NULL DEFAULT 60,
    is_active           BIT             NOT NULL DEFAULT 1,
    created_at          DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME(),
    updated_at          DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME(),
    CONSTRAINT uq_powerbi_menu_item UNIQUE (tenant_id, menu_item_id)
);

-- ═══════════════════════════════════════════════════════════════
-- POWER BI TENANT SERVICE CREDENTIALS
-- ═══════════════════════════════════════════════════════════════
-- Stored per tenant so each tenant uses their own Azure AD app registration

CREATE TABLE powerbi_tenant_config (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    tenant_id           UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id) UNIQUE,
    azure_tenant_id     VARCHAR(100)    NOT NULL,    -- Customer's Azure tenant
    client_id           VARCHAR(100)    NOT NULL,    -- App registration with PBI permissions
    client_secret_ref   VARCHAR(200)    NOT NULL,    -- Azure Key Vault reference
    service_account_upn VARCHAR(320)    NULL,        -- Optional: service account email
    embed_token_type    VARCHAR(20)     NOT NULL DEFAULT 'ServicePrincipal'
                          CHECK (embed_token_type IN ('ServicePrincipal','MasterUser')),
    is_active           BIT             NOT NULL DEFAULT 1,
    created_at          DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME()
);

-- ═══════════════════════════════════════════════════════════════
-- EMBED TOKEN CACHE (Track issued tokens for reuse within TTL)
-- ═══════════════════════════════════════════════════════════════

CREATE TABLE powerbi_embed_tokens (
    id                  UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWSEQUENTIALID(),
    tenant_id           UNIQUEIDENTIFIER NOT NULL REFERENCES tenants(id),
    report_id           UNIQUEIDENTIFIER NOT NULL REFERENCES powerbi_reports(id),
    user_id             UNIQUEIDENTIFIER NOT NULL REFERENCES users(id),
    token_value         VARCHAR(2000)   NOT NULL,    -- Encrypted embed token
    embed_url           VARCHAR(500)    NOT NULL,
    expires_at          DATETIME2       NOT NULL,
    created_at          DATETIME2       NOT NULL DEFAULT SYSUTCDATETIME()
);
CREATE INDEX ix_pbi_tokens_user_report ON powerbi_embed_tokens(
    tenant_id, user_id, report_id, expires_at DESC
);
```

---

### 14.3 Power BI Service (Token Generation)

```python
# app/services/powerbi_service.py
import aiohttp
from azure.identity.aio import ClientSecretCredential
from app.services.vault_service import VaultService
from app.services.cache_service import CacheService

POWER_BI_SCOPE = "https://analysis.windows.net/powerbi/api/.default"
POWER_BI_API   = "https://api.powerbi.com/v1.0/myorg"

class PowerBIService:

    def __init__(self, vault: VaultService, cache: CacheService):
        self.vault = vault
        self.cache = cache

    async def _get_access_token(self, config: PowerBITenantConfig) -> str:
        """Get Azure AD access token for Power BI API using Service Principal."""
        cache_key = f"pbi_access_token:{config.tenant_id}"
        cached = await self.cache.get(cache_key)
        if cached:
            return cached

        secret = await self.vault.get_secret(config.client_secret_ref)

        credential = ClientSecretCredential(
            tenant_id=config.azure_tenant_id,
            client_id=config.client_id,
            client_secret=secret,
        )
        token = await credential.get_token(POWER_BI_SCOPE)
        # Cache with buffer (expire 5 min before actual expiry)
        ttl = max(0, int(token.expires_on - time.time()) - 300)
        await self.cache.set(cache_key, token.token, ttl=ttl)
        return token.token

    async def generate_embed_token(
        self,
        tenant_id: str,
        report: PowerBIReport,
        user_id: str,
        user_email: str,
    ) -> dict:
        """
        Generate a Power BI embed token for a specific report and user.
        Applies Row-Level Security (RLS) if configured.
        """
        # Check for valid cached token first
        cache_key = f"pbi_embed:{tenant_id}:{report.id}:{user_id}"
        cached = await self.cache.get(cache_key)
        if cached:
            return cached

        config = await self.get_tenant_config(tenant_id)
        access_token = await self._get_access_token(config)

        headers = {
            "Authorization": f"Bearer {access_token}",
            "Content-Type":  "application/json",
        }

        # Build embed token request
        body = {
            "reports": [{"id": report.report_id}],
            "datasets": [{"id": report.dataset_id}] if report.dataset_id else [],
        }

        # Apply Power BI RLS if configured
        if report.rls_roles:
            rls_roles = json.loads(report.rls_roles)
            body["identities"] = [{
                "username": user_email,
                "roles":    rls_roles,
                "datasets": [report.dataset_id],
            }]

        async with aiohttp.ClientSession() as session:
            # Step 1: Get embed URL for the report
            async with session.get(
                f"{POWER_BI_API}/groups/{report.workspace_id}/reports/{report.report_id}",
                headers=headers
            ) as resp:
                report_meta = await resp.json()
                embed_url = report_meta["embedUrl"]

            # Step 2: Generate embed token
            async with session.post(
                f"{POWER_BI_API}/GenerateToken",
                headers=headers,
                json=body,
            ) as resp:
                token_resp = await resp.json()

        result = {
            "embed_url":     embed_url,
            "embed_token":   token_resp["token"],
            "token_id":      token_resp["tokenId"],
            "expiration":    token_resp["expiration"],
            "report_id":     report.report_id,
            "workspace_id":  report.workspace_id,
            "page_name":     report.page_name,
            "settings": {
                "filter_pane_enabled": report.filter_pane_enabled,
                "nav_pane_enabled":    report.nav_pane_enabled,
            },
            "default_filters": json.loads(report.default_filters) if report.default_filters else [],
        }

        # Cache the embed token (TTL = report.token_ttl_minutes - 5 min buffer)
        ttl = (report.token_ttl_minutes - 5) * 60
        await self.cache.set(cache_key, result, ttl=ttl)

        # Notify user via WebSocket when token is ready (useful for pre-fetching)
        await ws_manager.send_to_user(tenant_id, user_id, {
            "type":      "report.ready",
            "report_id": report.report_id,
            "expires_at": result["expiration"],
        })

        return result
```

---

### 14.4 Menu API Implementation

```python
# app/api/v1/navigation.py
from fastapi import APIRouter, Request, Depends
from app.services.menu_service import MenuService
from app.api.deps import get_current_user

router = APIRouter(prefix="/api/v1/navigation", tags=["Navigation"])

@router.get("/menu")
async def get_menu(
    request: Request,
    db=Depends(get_db),
):
    """
    Returns the complete navigation menu tree, filtered by the user's permissions.

    The frontend should call this once after login and cache the result locally.
    The menu is automatically filtered — the user only sees items they have access to.

    On permission changes, the backend pushes a 'permission.changed' WebSocket event
    and the frontend should re-fetch this endpoint.
    """
    user = request.state.user
    engine = request.state.permission_engine

    menu_service = MenuService(db, engine)

    # Returns hierarchical filtered menu
    menu = await menu_service.get_user_menu(
        tenant_id=user.tid,
        user_id=user.sub,
    )

    return {"status": "success", "data": menu}


@router.get("/menu/{item_id}/sub-menu")
async def get_sub_menu(
    item_id: str,
    request: Request,
    db=Depends(get_db),
):
    """
    Lazily load sub-menu items for a given parent menu item.
    Used for deep menu hierarchies where not all levels are loaded upfront.
    """
    user = request.state.user
    engine = request.state.permission_engine
    menu_service = MenuService(db, engine)

    sub_items = await menu_service.get_sub_menu(
        tenant_id=user.tid,
        user_id=user.sub,
        parent_id=item_id,
    )

    return {"status": "success", "data": sub_items}
```

**Menu Service — Permission-Filtered Tree**:

```python
# app/services/menu_service.py
class MenuService:

    async def get_user_menu(self, tenant_id: str, user_id: str) -> list:
        # Load all active menu items for tenant
        all_items = await self.db.execute(
            select(MenuItem)
            .where(MenuItem.tenant_id == tenant_id)
            .where(MenuItem.is_active == True)
            .order_by(MenuItem.parent_id, MenuItem.sort_order)
        )
        all_items = all_items.scalars().all()

        # Filter by user permissions (check each item's resource_key)
        visible_items = []
        for item in all_items:
            result = await self.permission_engine.check_permission(
                user_id=user_id,
                tenant_id=tenant_id,
                resource_key=item.resource_key,
                action_key="view",
            )
            if result.allowed:
                visible_items.append(item)

        # Build tree structure
        return self._build_tree(visible_items)

    def _build_tree(self, items: list) -> list:
        """Convert flat list to nested tree."""
        item_map = {str(i.id): self._serialize(i) for i in items}
        roots = []

        for item in items:
            serialized = item_map[str(item.id)]
            if item.parent_id is None:
                roots.append(serialized)
            elif str(item.parent_id) in item_map:
                parent = item_map[str(item.parent_id)]
                parent.setdefault("sub_items", []).append(serialized)

        return sorted(roots, key=lambda x: x["sort_order"])

    def _serialize(self, item: MenuItem) -> dict:
        return {
            "id":         str(item.id),
            "label":      item.label,
            "icon":       item.icon,
            "route":      item.route,
            "item_type":  item.item_type,
            "sort_order": item.sort_order,
            "badge_text": item.badge_text,
            "tooltip":    item.tooltip,
            "is_new_tab": item.is_new_tab,
            "sub_items":  [],
        }
```

**Menu API Response Example**:
```json
{
  "status": "success",
  "data": [
    {
      "id": "uuid-1",
      "label": "Dashboard",
      "icon": "grid-2x2",
      "route": "/dashboard",
      "item_type": "page",
      "sort_order": 1,
      "badge_text": null,
      "sub_items": []
    },
    {
      "id": "uuid-2",
      "label": "Analytics",
      "icon": "bar-chart",
      "route": null,
      "item_type": "group_header",
      "sort_order": 2,
      "sub_items": [
        {
          "id": "uuid-3",
          "label": "Sales Performance",
          "icon": "trending-up",
          "route": "/reports/sales",
          "item_type": "powerbi",
          "sort_order": 1,
          "badge_text": "NEW",
          "sub_items": []
        },
        {
          "id": "uuid-4",
          "label": "Customer Insights",
          "icon": "users",
          "route": "/reports/customers",
          "item_type": "powerbi",
          "sort_order": 2,
          "badge_text": null,
          "sub_items": []
        }
      ]
    }
  ]
}
```

---

### 14.5 Power BI Report APIs

| Method | Endpoint | Description | Auth | Permission |
|--------|----------|-------------|------|------------|
| `GET` | `/api/v1/powerbi/reports` | List all Power BI reports for tenant | Bearer | `powerbi:view` |
| `GET` | `/api/v1/powerbi/reports/{menu_item_id}` | Get report config for a menu item | Bearer | Per-item resource key |
| `GET` | `/api/v1/powerbi/reports/{menu_item_id}/embed-token` | Generate embed token | Bearer | Per-item resource key |
| `POST` | `/api/v1/powerbi/reports/{menu_item_id}/refresh-token` | Force refresh embed token | Bearer | Per-item resource key |
| `GET` | `/api/v1/admin/powerbi/reports` | Admin: list all reports | Bearer | `admin.powerbi:view` |
| `POST` | `/api/v1/admin/powerbi/reports` | Admin: create report mapping | Bearer | `admin.powerbi:create` |
| `PUT` | `/api/v1/admin/powerbi/reports/{id}` | Admin: update report mapping | Bearer | `admin.powerbi:edit` |
| `DELETE` | `/api/v1/admin/powerbi/reports/{id}` | Admin: remove report mapping | Bearer | `admin.powerbi:delete` |
| `GET` | `/api/v1/admin/powerbi/config` | Get Power BI service config | Bearer | `admin.powerbi:configure` |
| `PUT` | `/api/v1/admin/powerbi/config` | Update Power BI credentials | Bearer | `admin.powerbi:configure` |

**Get Embed Token Response**:
```json
{
  "status": "success",
  "data": {
    "report_id":     "pbi-report-uuid",
    "workspace_id":  "pbi-workspace-uuid",
    "embed_url":     "https://app.powerbi.com/reportEmbed?reportId=...&groupId=...",
    "embed_token":   "H4sIAAAAAAAEAy2...",
    "token_id":      "token-uuid",
    "expiration":    "2026-02-26T12:30:00Z",
    "page_name":     "ReportSection1",
    "settings": {
      "filter_pane_enabled": true,
      "nav_pane_enabled":    false
    },
    "default_filters": [
      {
        "$schema": "http://powerbi.com/product/schema#basic",
        "target": {"table": "Sales", "column": "Year"},
        "operator": "In",
        "values": [2025, 2026]
      }
    ]
  }
}
```

---

### 14.6 Token Auto-Refresh Strategy

Power BI embed tokens expire (default 60 min). The backend proactively refreshes them before expiry:

```
Strategy 1: Frontend-triggered refresh
  - Frontend tracks token expiry_at from embed-token response
  - 5 minutes before expiry, calls POST /api/v1/powerbi/reports/{id}/refresh-token
  - Backend returns new token, frontend updates <PowerBIEmbed> component

Strategy 2: Backend-pushed refresh (WebSocket)
  - Background worker scans powerbi_embed_tokens table
  - Finds tokens expiring within next 10 minutes
  - Pre-generates new tokens via Power BI API
  - Pushes 'report.token_refresh' event via WebSocket to each user
  - Frontend receives event and calls embed-token API with ?use_cached=true

Redis Cache Key: pbi_embed:{tenant_id}:{report_id}:{user_id}
  TTL: (token_ttl_minutes - 5) * 60 seconds
```

---

## 15. File Upload — SharePoint & OneLake Integration

This section covers the architecture, API design, and implementation for uploading files from any React page to backend-configured destinations (Microsoft SharePoint or Azure OneLake / Data Lake Gen2). The system is intentionally **page-agnostic on the frontend and destination-agnostic on the backend** — a single set of APIs handles all upload scenarios regardless of which page initiates the upload or which storage target receives the data.

### 15.1 Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Single API surface** | One endpoint handles all pages/destinations | Frontend pages do not embed destination details; all routing is resolved server-side by `page_key` |
| **Streaming upload** | FastAPI `UploadFile` streams directly to storage | No full-file buffering in memory; constant ~8 MB working memory per concurrent upload |
| **Chunked protocol** | Client splits large files (>5 MB) into chunks | Allows progress reporting, resume on failure, works within API Gateway body limits |
| **Destination abstraction** | `FileUploadService` wraps both SharePoint and OneLake APIs | Frontend and upload endpoint are destination-unaware; swap backends without changing API contracts |
| **Per-page destination config** | DB table `upload_page_config` maps `page_key` → destination | Any page can be re-pointed to a new path via DB config change, no code deployment needed |
| **Multi-file in one request** | Multipart `multipart/form-data` with N files | Reduces round-trips for batch uploads; each file streams independently |
| **Total size cap** | 200 MB default (configurable per tenant) | Prevents abuse; set per tenant in `upload_destinations` |

### 15.2 Upload Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                          React Page (any)                            │
│                                                                      │
│   Page sends X-Page-Key header (e.g. "energy_data_ingestion")       │
│   File is chunked client-side (8 MB chunks via File.slice())        │
│   Each chunk PUT to /api/v1/files/upload/{upload_id}/chunk          │
└──────────────────────────────┬───────────────────────────────────────┘
                               │  HTTPS / multipart or chunked
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      FastAPI Upload Endpoint                         │
│                                                                      │
│  1. Validate JWT + tenant context (middleware)                       │
│  2. Check RBAC: files:upload permission for resource = page_key     │
│  3. Resolve page_key → UploadDestination from DB / Redis cache      │
│  4. Validate file types, total size, per-file size limits           │
│  5. Stream each file chunk to FileUploadService                     │
│  6. Record upload_history row (async, after stream completes)       │
└──────────────┬───────────────────────────────────┬───────────────────┘
               │                                   │
               ▼                                   ▼
┌──────────────────────────┐          ┌────────────────────────────────┐
│  SharePoint Adapter      │          │  OneLake / ADLS Gen2 Adapter   │
│                          │          │                                │
│  Microsoft Graph API     │          │  azure-storage-file-datalake   │
│  Resumable Upload Session│          │  DataLakeFileClient            │
│  (files > 4 MB)          │          │  append_data() + flush_data()  │
│  Direct PUT (≤ 4 MB)     │          │                                │
└──────────────────────────┘          └────────────────────────────────┘
```

### 15.3 Database Schema

```sql
-- Defines available upload destinations (SharePoint sites, OneLake accounts)
-- One row per logical destination; shared across tenants or tenant-scoped
CREATE TABLE upload_destinations (
    destination_id      INT             IDENTITY(1,1) PRIMARY KEY,
    tenant_id           NVARCHAR(100)   NOT NULL,
    destination_name    NVARCHAR(200)   NOT NULL,           -- Human label
    destination_type    NVARCHAR(20)    NOT NULL,           -- 'sharepoint' | 'onelake'

    -- SharePoint fields (destination_type = 'sharepoint')
    sp_site_id          NVARCHAR(500),                      -- Graph API site ID
    sp_drive_id         NVARCHAR(500),                      -- Document library drive ID
    sp_base_folder_path NVARCHAR(1000),                     -- e.g. /Uploads/EnergyData

    -- OneLake / ADLS Gen2 fields (destination_type = 'onelake')
    adls_account_name   NVARCHAR(200),                      -- Storage account name
    adls_filesystem     NVARCHAR(200),                      -- Container / filesystem name
    adls_base_path      NVARCHAR(1000),                     -- e.g. /raw/energy_uploads

    -- Limits
    max_total_mb        INT             DEFAULT 200,        -- Total per request (configurable)
    max_file_mb         INT             DEFAULT 100,        -- Per file limit
    allowed_extensions  NVARCHAR(500)   DEFAULT '*',        -- Comma-separated, '*' = all
    allow_folder_create BIT             DEFAULT 1,          -- Can frontend create subfolders?

    is_active           BIT             DEFAULT 1,
    created_at          DATETIME2       DEFAULT GETUTCDATE(),
    updated_at          DATETIME2       DEFAULT GETUTCDATE(),

    INDEX IX_upload_dest_tenant (tenant_id)
);

-- Maps a React page_key to its configured upload destination + sub-path
CREATE TABLE upload_page_config (
    config_id           INT             IDENTITY(1,1) PRIMARY KEY,
    tenant_id           NVARCHAR(100)   NOT NULL,
    page_key            NVARCHAR(200)   NOT NULL,           -- Frontend page identifier
    destination_id      INT             NOT NULL
                        REFERENCES upload_destinations(destination_id),
    sub_path            NVARCHAR(1000)  DEFAULT '',         -- Appended to base_folder_path
    description         NVARCHAR(500),                      -- Internal note on purpose
    is_active           BIT             DEFAULT 1,
    created_at          DATETIME2       DEFAULT GETUTCDATE(),

    UNIQUE (tenant_id, page_key),
    INDEX IX_upload_page_tenant (tenant_id, page_key)
);

-- Audit trail of completed uploads
CREATE TABLE upload_history (
    upload_id           UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    tenant_id           NVARCHAR(100)   NOT NULL,
    user_id             UNIQUEIDENTIFIER NOT NULL,
    page_key            NVARCHAR(200)   NOT NULL,
    destination_id      INT             NOT NULL,
    destination_path    NVARCHAR(2000)  NOT NULL,           -- Full resolved path
    file_name           NVARCHAR(500)   NOT NULL,
    file_size_bytes     BIGINT          NOT NULL,
    file_extension      NVARCHAR(50),
    destination_url     NVARCHAR(2000),                     -- SharePoint/OneLake URL
    status              NVARCHAR(20)    DEFAULT 'completed', -- 'completed' | 'failed'
    error_message       NVARCHAR(MAX),
    upload_started_at   DATETIME2       DEFAULT GETUTCDATE(),
    upload_completed_at DATETIME2,
    duration_ms         INT,

    INDEX IX_upload_hist_tenant_user (tenant_id, user_id, upload_started_at DESC)
);

-- Tracks in-progress chunked uploads (TTL-cleaned by background job)
CREATE TABLE upload_sessions (
    session_id          UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    tenant_id           NVARCHAR(100)   NOT NULL,
    user_id             UNIQUEIDENTIFIER NOT NULL,
    page_key            NVARCHAR(200)   NOT NULL,
    file_name           NVARCHAR(500)   NOT NULL,
    file_size_bytes     BIGINT          NOT NULL,
    total_chunks        INT             NOT NULL,
    chunks_received     INT             DEFAULT 0,
    bytes_received      BIGINT          DEFAULT 0,
    -- Destination adapter state
    destination_type    NVARCHAR(20)    NOT NULL,
    adapter_session_url NVARCHAR(2000), -- SharePoint upload session URL
    adls_file_path      NVARCHAR(2000), -- OneLake file path being appended
    status              NVARCHAR(20)    DEFAULT 'in_progress',
    expires_at          DATETIME2       NOT NULL,           -- Auto-expire stale sessions
    created_at          DATETIME2       DEFAULT GETUTCDATE(),

    INDEX IX_upload_sessions_tenant (tenant_id, status)
);
```

### 15.4 Upload Protocol

The system supports two upload modes selected by the client based on file size:

#### Mode A — Direct Multipart Upload (≤ 5 MB per file)

All files in one `multipart/form-data` POST. Suitable for small document uploads from any page.

```
POST /api/v1/files/upload
Content-Type: multipart/form-data
Authorization: Bearer <token>
X-Page-Key: energy_data_ingestion

Body:
  files[]   = File1.xlsx (2 MB)
  files[]   = File2.csv  (1 MB)
  sub_path  = "2026/February"     ← optional: append to configured base path
  new_folder = ""                 ← optional: create this folder first, then upload into it
```

#### Mode B — Chunked Upload (> 5 MB per file, or when total > 5 MB)

Three-phase protocol for reliable large-file transfer:

```
Phase 1 — Initiate (once per file)
──────────────────────────────────
POST /api/v1/files/upload/init
{
  "page_key": "per_report_uploads",
  "file_name": "Q1_2025_Budget.xlsx",
  "file_size_bytes": 104857600,         ← 100 MB
  "total_chunks": 13,                   ← ceil(100MB / 8MB)
  "sub_path": "2026/Q1",
  "mime_type": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
}

Response:
{
  "upload_session_id": "uuid",
  "chunk_size_bytes": 8388608,          ← 8 MB recommended chunk size
  "expires_at": "2026-02-27T12:30:00Z"
}

Phase 2 — Send Chunks (repeat for each chunk)
──────────────────────────────────────────────
PUT /api/v1/files/upload/{upload_session_id}/chunk
Content-Range: bytes 0-8388607/104857600
Content-Type: application/octet-stream

<raw binary chunk data>

Response (200 OK):
{
  "chunks_received": 1,
  "bytes_received": 8388608,
  "percent_complete": 8
}

Phase 3 — Complete (once all chunks received)
──────────────────────────────────────────────
POST /api/v1/files/upload/{upload_session_id}/complete

Response:
{
  "upload_id": "uuid",
  "file_name": "Q1_2025_Budget.xlsx",
  "destination_url": "https://company.sharepoint.com/sites/...",
  "file_size_bytes": 104857600,
  "duration_ms": 4520
}
```

### 15.5 Backend Service Implementation

```python
# app/services/file_upload_service.py
from abc import ABC, abstractmethod
from typing import AsyncIterator, Optional
import aiohttp
from azure.storage.filedatalake.aio import DataLakeServiceClient
from app.core.config import settings


class UploadDestination:
    """Resolved destination config from DB."""
    destination_type: str      # 'sharepoint' | 'onelake'
    max_total_mb: int
    max_file_mb: int
    allowed_extensions: list[str]
    allow_folder_create: bool
    # SharePoint
    sp_site_id: Optional[str]
    sp_drive_id: Optional[str]
    sp_base_folder_path: Optional[str]
    # OneLake
    adls_account_name: Optional[str]
    adls_filesystem: Optional[str]
    adls_base_path: Optional[str]


class StorageAdapter(ABC):
    """Abstract adapter — implement for each storage backend."""

    @abstractmethod
    async def create_upload_session(
        self, file_name: str, file_size: int, dest_path: str
    ) -> dict:
        """Return adapter-specific session state for chunked upload."""

    @abstractmethod
    async def upload_chunk(
        self, session_state: dict, chunk_data: bytes,
        range_start: int, range_end: int, total_size: int
    ) -> None:
        """Append a single chunk to the in-progress upload."""

    @abstractmethod
    async def finalize_upload(self, session_state: dict) -> str:
        """Complete the upload. Returns the final destination URL."""

    @abstractmethod
    async def upload_small_file(
        self, file_stream: AsyncIterator[bytes],
        dest_path: str, file_name: str
    ) -> str:
        """Single-shot upload for small files (≤ 4 MB). Returns URL."""

    @abstractmethod
    async def create_folder(self, dest_path: str, folder_name: str) -> str:
        """Create a child folder. Returns full folder path."""


class SharePointAdapter(StorageAdapter):
    """
    Microsoft Graph API adapter for SharePoint Online document libraries.
    Uses resumable upload sessions for files > 4 MB.
    """

    GRAPH_BASE = "https://graph.microsoft.com/v1.0"
    SMALL_FILE_THRESHOLD = 4 * 1024 * 1024  # 4 MB

    def __init__(self, site_id: str, drive_id: str):
        self.site_id = site_id
        self.drive_id = drive_id

    async def _get_graph_token(self) -> str:
        """Acquire app-only token for Microsoft Graph API."""
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"https://login.microsoftonline.com/{settings.AZURE_TENANT_ID}/oauth2/v2.0/token",
                data={
                    "grant_type": "client_credentials",
                    "client_id": settings.GRAPH_CLIENT_ID,
                    "client_secret": settings.GRAPH_CLIENT_SECRET,
                    "scope": "https://graph.microsoft.com/.default",
                }
            ) as resp:
                data = await resp.json()
                return data["access_token"]

    async def create_upload_session(
        self, file_name: str, file_size: int, dest_path: str
    ) -> dict:
        token = await self._get_graph_token()
        url = (
            f"{self.GRAPH_BASE}/sites/{self.site_id}/drives/{self.drive_id}"
            f"/items/root:/{dest_path}/{file_name}:/createUploadSession"
        )
        async with aiohttp.ClientSession() as session:
            async with session.post(
                url,
                headers={"Authorization": f"Bearer {token}"},
                json={"item": {"@microsoft.graph.conflictBehavior": "rename"}}
            ) as resp:
                data = await resp.json()
                return {"upload_url": data["uploadUrl"], "type": "sharepoint"}

    async def upload_chunk(
        self, session_state: dict, chunk_data: bytes,
        range_start: int, range_end: int, total_size: int
    ) -> None:
        upload_url = session_state["upload_url"]
        async with aiohttp.ClientSession() as session:
            async with session.put(
                upload_url,
                headers={
                    "Content-Range": f"bytes {range_start}-{range_end}/{total_size}",
                    "Content-Length": str(len(chunk_data)),
                },
                data=chunk_data,
            ) as resp:
                if resp.status not in (200, 201, 202):
                    raise RuntimeError(
                        f"SharePoint chunk upload failed: {resp.status} {await resp.text()}"
                    )

    async def finalize_upload(self, session_state: dict) -> str:
        # SharePoint auto-finalizes on last chunk — session_state carries the item URL
        return session_state.get("item_url", "")

    async def upload_small_file(
        self, file_stream: AsyncIterator[bytes],
        dest_path: str, file_name: str
    ) -> str:
        token = await self._get_graph_token()
        data = b"".join([chunk async for chunk in file_stream])
        url = (
            f"{self.GRAPH_BASE}/sites/{self.site_id}/drives/{self.drive_id}"
            f"/items/root:/{dest_path}/{file_name}:/content"
        )
        async with aiohttp.ClientSession() as session:
            async with session.put(
                url,
                headers={
                    "Authorization": f"Bearer {token}",
                    "Content-Type": "application/octet-stream",
                },
                data=data,
            ) as resp:
                result = await resp.json()
                return result.get("webUrl", "")

    async def create_folder(self, dest_path: str, folder_name: str) -> str:
        token = await self._get_graph_token()
        url = (
            f"{self.GRAPH_BASE}/sites/{self.site_id}/drives/{self.drive_id}"
            f"/items/root:/{dest_path}:/children"
        )
        async with aiohttp.ClientSession() as session:
            async with session.post(
                url,
                headers={
                    "Authorization": f"Bearer {token}",
                    "Content-Type": "application/json",
                },
                json={
                    "name": folder_name,
                    "folder": {},
                    "@microsoft.graph.conflictBehavior": "fail",
                }
            ) as resp:
                result = await resp.json()
                return result.get("name", folder_name)


class OneLakeAdapter(StorageAdapter):
    """
    Azure Data Lake Gen2 adapter for Microsoft Fabric OneLake.
    Uses DataLakeFileClient append + flush pattern for chunked uploads.
    """

    def __init__(self, account_name: str, filesystem: str):
        self.account_name = account_name
        self.filesystem = filesystem
        self._client = DataLakeServiceClient(
            account_url=f"https://{account_name}.dfs.core.windows.net",
            credential=settings.ADLS_CREDENTIAL,  # DefaultAzureCredential or SAS
        )

    async def create_upload_session(
        self, file_name: str, file_size: int, dest_path: str
    ) -> dict:
        full_path = f"{dest_path}/{file_name}"
        fs = self._client.get_file_system_client(self.filesystem)
        file_client = fs.get_file_client(full_path)
        await file_client.create_file()
        return {"adls_path": full_path, "type": "onelake", "bytes_appended": 0}

    async def upload_chunk(
        self, session_state: dict, chunk_data: bytes,
        range_start: int, range_end: int, total_size: int
    ) -> None:
        full_path = session_state["adls_path"]
        offset = session_state["bytes_appended"]
        fs = self._client.get_file_system_client(self.filesystem)
        file_client = fs.get_file_client(full_path)
        await file_client.append_data(data=chunk_data, offset=offset)
        session_state["bytes_appended"] += len(chunk_data)

    async def finalize_upload(self, session_state: dict) -> str:
        full_path = session_state["adls_path"]
        total = session_state["bytes_appended"]
        fs = self._client.get_file_system_client(self.filesystem)
        file_client = fs.get_file_client(full_path)
        await file_client.flush_data(position=total)
        return (
            f"https://{self.account_name}.dfs.core.windows.net"
            f"/{self.filesystem}/{full_path}"
        )

    async def upload_small_file(
        self, file_stream: AsyncIterator[bytes],
        dest_path: str, file_name: str
    ) -> str:
        data = b"".join([chunk async for chunk in file_stream])
        full_path = f"{dest_path}/{file_name}"
        fs = self._client.get_file_system_client(self.filesystem)
        file_client = fs.get_file_client(full_path)
        await file_client.upload_data(data=data, overwrite=False)
        return (
            f"https://{self.account_name}.dfs.core.windows.net"
            f"/{self.filesystem}/{full_path}"
        )

    async def create_folder(self, dest_path: str, folder_name: str) -> str:
        full_path = f"{dest_path}/{folder_name}"
        fs = self._client.get_file_system_client(self.filesystem)
        dir_client = fs.get_directory_client(full_path)
        await dir_client.create_directory()
        return full_path


class FileUploadService:
    """
    Orchestrates file uploads across destinations.
    Resolves page_key → UploadDestination → correct StorageAdapter.
    """

    def __init__(self, tenant_id: str):
        self.tenant_id = tenant_id

    async def resolve_destination(self, page_key: str) -> tuple[UploadDestination, StorageAdapter]:
        """
        Looks up upload_page_config → upload_destinations for a page_key.
        Results are cached in Redis for 5 minutes.
        """
        cache_key = f"upload_config:{self.tenant_id}:{page_key}"
        # ... Redis lookup + DB fallback ...
        adapter = (
            SharePointAdapter(dest.sp_site_id, dest.sp_drive_id)
            if dest.destination_type == "sharepoint"
            else OneLakeAdapter(dest.adls_account_name, dest.adls_filesystem)
        )
        return dest, adapter

    def validate_files(
        self, files: list, dest: UploadDestination
    ) -> None:
        """Validate file count, extensions, per-file size, and total size."""
        total_bytes = 0
        for f in files:
            ext = f.filename.rsplit(".", 1)[-1].lower()
            if dest.allowed_extensions != ["*"] and ext not in dest.allowed_extensions:
                raise ValueError(f"File type '.{ext}' not allowed for this upload destination.")
            if f.size and f.size > dest.max_file_mb * 1024 * 1024:
                raise ValueError(
                    f"'{f.filename}' exceeds {dest.max_file_mb} MB per-file limit."
                )
            total_bytes += f.size or 0

        if total_bytes > dest.max_total_mb * 1024 * 1024:
            raise ValueError(
                f"Total upload size exceeds {dest.max_total_mb} MB limit for this page."
            )

    def resolve_path(
        self, dest: UploadDestination, page_config_sub_path: str,
        request_sub_path: str = ""
    ) -> str:
        """Build the full destination path from base + configured sub-path + request override."""
        base = (
            dest.sp_base_folder_path or dest.adls_base_path or ""
        ).rstrip("/")
        configured = page_config_sub_path.strip("/")
        override = request_sub_path.strip("/")
        parts = [p for p in [base, configured, override] if p]
        return "/".join(parts)
```

### 15.6 API Endpoints

#### GET /api/v1/files/upload-config

Returns the upload configuration for a given page key. Called by the frontend on page load to configure the file picker (allowed types, size limits, destination label, folder creation allowed).

```
GET /api/v1/files/upload-config?page_key=energy_data_ingestion
Authorization: Bearer <token>

Response 200:
{
  "page_key": "energy_data_ingestion",
  "destination_name": "Energy Data SharePoint Library",
  "destination_type": "sharepoint",
  "max_total_mb": 200,
  "max_file_mb": 100,
  "allowed_extensions": ["xlsx", "csv", "pdf", "zip"],
  "allow_folder_create": true,
  "base_path_label": "/EnergyData/Uploads"    ← display only, not actual path
}
```

#### POST /api/v1/files/upload

Direct multipart upload for files ≤ 5 MB per file (or small batches).

```
POST /api/v1/files/upload
Content-Type: multipart/form-data
Authorization: Bearer <token>
X-Page-Key: energy_data_ingestion

Form fields:
  files      (required)  — one or more files
  sub_path   (optional)  — override subfolder, e.g. "2026/February"
  new_folder (optional)  — create this folder and upload into it

Response 200:
{
  "uploaded": [
    {
      "file_name": "Budget_2026.xlsx",
      "file_size_bytes": 524288,
      "destination_url": "https://company.sharepoint.com/sites/.../Budget_2026.xlsx",
      "upload_id": "uuid"
    }
  ],
  "failed": [],
  "total_files": 1,
  "duration_ms": 1240
}
```

**Permission required:** `files:upload` on resource `page_key` value (e.g., `energy_data_ingestion`).

```python
# app/api/v1/endpoints/files.py
from fastapi import APIRouter, UploadFile, File, Form, Depends, Header
from typing import Optional
from app.services.file_upload_service import FileUploadService
from app.core.dependencies import get_current_user, require_permission

router = APIRouter(prefix="/api/v1/files", tags=["File Upload"])

CHUNK_STREAM_SIZE = 256 * 1024  # Read UploadFile in 256 KB increments


@router.post("/upload")
async def upload_files(
    files: list[UploadFile] = File(...),
    sub_path: Optional[str] = Form(default=""),
    new_folder: Optional[str] = Form(default=""),
    x_page_key: str = Header(..., alias="X-Page-Key"),
    current_user = Depends(get_current_user),
):
    await require_permission(current_user, resource=x_page_key, action="upload")

    svc = FileUploadService(tenant_id=current_user.tenant_id)
    dest, adapter = await svc.resolve_destination(x_page_key)
    svc.validate_files(files, dest)

    # Create new folder if requested
    if new_folder and dest.allow_folder_create:
        base_path = svc.resolve_path(dest, "", sub_path)
        new_folder = new_folder.strip().replace("/", "_")  # sanitize
        created_path = await adapter.create_folder(base_path, new_folder)
        dest_path = created_path
    else:
        dest_path = svc.resolve_path(dest, "", sub_path)

    results, errors = [], []
    import time
    start = time.monotonic()

    for f in files:
        try:
            url = await adapter.upload_small_file(
                file_stream=f,
                dest_path=dest_path,
                file_name=f.filename,
            )
            results.append({
                "file_name": f.filename,
                "file_size_bytes": f.size,
                "destination_url": url,
            })
        except Exception as exc:
            errors.append({"file_name": f.filename, "error": str(exc)})

    return {
        "uploaded": results,
        "failed": errors,
        "total_files": len(files),
        "duration_ms": int((time.monotonic() - start) * 1000),
    }
```

#### POST /api/v1/files/upload/init

Initiate a chunked upload session for a large file.

```
POST /api/v1/files/upload/init
Authorization: Bearer <token>
Content-Type: application/json

{
  "page_key": "per_report_uploads",
  "file_name": "Q1_Budget_Detailed.xlsx",
  "file_size_bytes": 104857600,
  "total_chunks": 13,
  "sub_path": "2026/Q1",
  "mime_type": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
}

Response 200:
{
  "upload_session_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "chunk_size_bytes": 8388608,
  "expires_at": "2026-02-27T12:30:00Z",
  "total_chunks": 13
}
```

#### PUT /api/v1/files/upload/{session_id}/chunk

Upload one chunk. The `Content-Range` header drives offset tracking server-side.

```
PUT /api/v1/files/upload/{session_id}/chunk
Authorization: Bearer <token>
Content-Range: bytes 0-8388607/104857600
Content-Type: application/octet-stream

<raw binary — 8 MB>

Response 200:
{
  "upload_session_id": "...",
  "chunks_received": 1,
  "bytes_received": 8388608,
  "percent_complete": 8,
  "next_expected_byte": 8388608
}

Response 416 (Conflict — chunk out of sequence):
{
  "detail": "Expected bytes starting at 8388608, received 0",
  "next_expected_byte": 8388608
}
```

```python
@router.put("/upload/{session_id}/chunk")
async def upload_chunk(
    session_id: str,
    request: Request,
    content_range: str = Header(...),
    current_user = Depends(get_current_user),
):
    # Parse Content-Range: bytes start-end/total
    import re
    match = re.match(r"bytes (\d+)-(\d+)/(\d+)", content_range)
    if not match:
        raise HTTPException(400, "Invalid Content-Range header")

    range_start = int(match.group(1))
    range_end   = int(match.group(2))
    total_size  = int(match.group(3))

    # Load session from DB
    session = await get_upload_session(session_id, current_user.tenant_id)
    if not session:
        raise HTTPException(404, "Upload session not found or expired")

    # Enforce sequential ordering
    if range_start != session.bytes_received:
        raise HTTPException(416, {
            "detail": f"Expected bytes starting at {session.bytes_received}",
            "next_expected_byte": session.bytes_received,
        })

    # Stream raw body → adapter
    chunk_data = await request.body()
    _, adapter = await FileUploadService(current_user.tenant_id).resolve_destination(
        session.page_key
    )
    await adapter.upload_chunk(
        session_state=session.adapter_state,
        chunk_data=chunk_data,
        range_start=range_start,
        range_end=range_end,
        total_size=total_size,
    )

    # Update session counters
    await update_upload_session(session_id, bytes_added=len(chunk_data))
    new_bytes = session.bytes_received + len(chunk_data)

    return {
        "upload_session_id": session_id,
        "chunks_received": session.chunks_received + 1,
        "bytes_received": new_bytes,
        "percent_complete": int(new_bytes / total_size * 100),
        "next_expected_byte": new_bytes,
    }
```

#### POST /api/v1/files/upload/{session_id}/complete

Finalize the upload after all chunks are delivered. The backend flushes the file to storage and records the audit entry.

```
POST /api/v1/files/upload/{session_id}/complete
Authorization: Bearer <token>

Response 200:
{
  "upload_id": "uuid",
  "file_name": "Q1_Budget_Detailed.xlsx",
  "destination_url": "https://company.sharepoint.com/sites/...xlsx",
  "file_size_bytes": 104857600,
  "duration_ms": 4820
}

Response 409 (Conflict — not all chunks received):
{
  "detail": "Upload incomplete: 11 of 13 chunks received (91.9%)",
  "bytes_received": 96468992,
  "bytes_expected": 104857600
}
```

#### POST /api/v1/files/folders

Create a child folder within a page's configured upload destination.

```
POST /api/v1/files/folders
Authorization: Bearer <token>
Content-Type: application/json

{
  "page_key": "energy_data_ingestion",
  "parent_sub_path": "2026",
  "folder_name": "February"
}

Response 201:
{
  "folder_name": "February",
  "full_path": "/EnergyData/Uploads/2026/February",
  "destination_type": "sharepoint"
}

Response 403:
{
  "detail": "Folder creation is not enabled for this upload destination."
}
```

#### GET /api/v1/files/browse

Browse the folder tree at a page's configured upload destination.

```
GET /api/v1/files/browse?page_key=energy_data_ingestion&sub_path=2026
Authorization: Bearer <token>

Response 200:
{
  "path": "/EnergyData/Uploads/2026",
  "items": [
    {"name": "January",  "type": "folder", "modified": "2026-01-31T18:20:00Z"},
    {"name": "February", "type": "folder", "modified": "2026-02-15T09:10:00Z"}
  ]
}
```

#### GET /api/v1/files/history

Returns the upload history for the current user (or all users for admins).

```
GET /api/v1/files/history?page_key=energy_data_ingestion&limit=20&offset=0
Authorization: Bearer <token>

Response 200:
{
  "total": 47,
  "items": [
    {
      "upload_id": "uuid",
      "file_name": "Q1_Budget.xlsx",
      "file_size_bytes": 2097152,
      "destination_url": "https://...",
      "status": "completed",
      "upload_completed_at": "2026-02-20T14:35:22Z",
      "duration_ms": 980
    }
  ]
}
```

### 15.7 File Validation & Security

```python
# app/services/file_upload_service.py — validation layer

DANGEROUS_EXTENSIONS = {
    "exe", "bat", "cmd", "com", "sh", "ps1", "vbs", "js",
    "jar", "msi", "dll", "scr", "reg", "lnk", "pif"
}

MIME_TYPE_MAP = {
    "xlsx": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    "csv":  "text/csv",
    "pdf":  "application/pdf",
    "docx": "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
    "zip":  "application/zip",
    "png":  "image/png",
    "jpg":  "image/jpeg",
}

def validate_file_safety(filename: str, content_type: str) -> None:
    """
    Security validation applied to every uploaded file.
    1. Block dangerous extensions unconditionally
    2. Verify declared Content-Type matches file extension
    3. Sanitize filename (no path traversal)
    """
    import os
    # Strip path components (prevent directory traversal)
    safe_name = os.path.basename(filename)
    if safe_name != filename:
        raise ValueError("Invalid filename: path traversal detected")

    ext = safe_name.rsplit(".", 1)[-1].lower() if "." in safe_name else ""

    if ext in DANGEROUS_EXTENSIONS:
        raise ValueError(f"File type '.{ext}' is not permitted for upload.")

    # Verify MIME type matches extension
    expected_mime = MIME_TYPE_MAP.get(ext)
    if expected_mime and content_type and content_type != expected_mime:
        raise ValueError(
            f"Content-Type mismatch: declared '{content_type}' "
            f"but extension '.{ext}' expects '{expected_mime}'"
        )
```

### 15.8 Frontend Integration Pattern

The frontend uses a **common `useFileUpload` hook** across all pages. Each page only provides its `pageKey` — all destination logic stays on the server.

```typescript
// hooks/useFileUpload.ts
import { useState, useCallback } from 'react';
import { apiClient } from '../config/api';

const CHUNK_SIZE = 8 * 1024 * 1024; // 8 MB

interface UploadProgress {
  fileName: string;
  percent: number;
  status: 'pending' | 'uploading' | 'done' | 'error';
  destinationUrl?: string;
  error?: string;
}

export function useFileUpload(pageKey: string) {
  const [progress, setProgress] = useState<Record<string, UploadProgress>>({});

  // Fetch config on component mount
  const fetchConfig = useCallback(async () => {
    const res = await apiClient.get('/files/upload-config', {
      params: { page_key: pageKey }
    });
    return res.data;
  }, [pageKey]);

  const uploadFiles = useCallback(async (
    files: File[],
    options?: { subPath?: string; newFolder?: string }
  ) => {
    const initProgress = Object.fromEntries(
      files.map(f => [f.name, { fileName: f.name, percent: 0, status: 'pending' as const }])
    );
    setProgress(initProgress);

    await Promise.allSettled(
      files.map(f => uploadSingleFile(f, options))
    );
  }, [pageKey]);

  const uploadSingleFile = async (
    file: File,
    options?: { subPath?: string; newFolder?: string }
  ) => {
    const name = file.name;

    // Small file: direct multipart
    if (file.size <= CHUNK_SIZE) {
      try {
        setProgress(prev => ({ ...prev, [name]: { ...prev[name], status: 'uploading' }}));
        const form = new FormData();
        form.append('files', file);
        if (options?.subPath)   form.append('sub_path', options.subPath);
        if (options?.newFolder) form.append('new_folder', options.newFolder);

        const res = await apiClient.post('/files/upload', form, {
          headers: {
            'X-Page-Key': pageKey,
            'Content-Type': 'multipart/form-data',
          },
          onUploadProgress: (e) => {
            const pct = Math.round((e.loaded / (e.total ?? 1)) * 100);
            setProgress(prev => ({ ...prev, [name]: { ...prev[name], percent: pct }}));
          }
        });
        const uploaded = res.data.uploaded[0];
        setProgress(prev => ({
          ...prev,
          [name]: { ...prev[name], status: 'done', percent: 100,
                    destinationUrl: uploaded.destination_url }
        }));
      } catch (err: any) {
        setProgress(prev => ({
          ...prev, [name]: { ...prev[name], status: 'error', error: err.message }
        }));
      }
      return;
    }

    // Large file: chunked protocol
    const totalChunks = Math.ceil(file.size / CHUNK_SIZE);

    // Phase 1 — init
    const initRes = await apiClient.post('/files/upload/init', {
      page_key:        pageKey,
      file_name:       file.name,
      file_size_bytes: file.size,
      total_chunks:    totalChunks,
      sub_path:        options?.subPath ?? '',
      mime_type:       file.type,
    });
    const { upload_session_id } = initRes.data;

    // Phase 2 — send chunks
    for (let i = 0; i < totalChunks; i++) {
      const start = i * CHUNK_SIZE;
      const end   = Math.min(start + CHUNK_SIZE - 1, file.size - 1);
      const chunk = file.slice(start, end + 1);

      await apiClient.put(
        `/files/upload/${upload_session_id}/chunk`,
        chunk,
        {
          headers: {
            'Content-Range': `bytes ${start}-${end}/${file.size}`,
            'Content-Type': 'application/octet-stream',
          },
        }
      );

      const pct = Math.round(((i + 1) / totalChunks) * 95); // 95% on last chunk
      setProgress(prev => ({
        ...prev, [name]: { ...prev[name], status: 'uploading', percent: pct }
      }));
    }

    // Phase 3 — complete
    const completeRes = await apiClient.post(
      `/files/upload/${upload_session_id}/complete`
    );
    setProgress(prev => ({
      ...prev,
      [name]: {
        ...prev[name], status: 'done', percent: 100,
        destinationUrl: completeRes.data.destination_url
      }
    }));
  };

  const createFolder = useCallback(async (
    folderName: string, parentSubPath = ''
  ) => {
    const res = await apiClient.post('/files/folders', {
      page_key:        pageKey,
      parent_sub_path: parentSubPath,
      folder_name:     folderName,
    });
    return res.data;
  }, [pageKey]);

  return { uploadFiles, createFolder, fetchConfig, progress };
}
```

**Usage in any React page (each page only specifies its `pageKey`):**

```tsx
// pages/EnergyDataUpload.tsx
import { useFileUpload } from '../hooks/useFileUpload';
import { useEffect, useState } from 'react';

export function EnergyDataUploadPage() {
  const { uploadFiles, createFolder, fetchConfig, progress } =
    useFileUpload('energy_data_ingestion');         // ← only thing that changes per page

  const [config, setConfig] = useState<any>(null);

  useEffect(() => {
    fetchConfig().then(setConfig);
  }, [fetchConfig]);

  const handleDrop = async (acceptedFiles: File[]) => {
    await uploadFiles(acceptedFiles, { subPath: '2026/February' });
  };

  return (
    <div>
      <DropZone
        accept={config?.allowed_extensions}
        maxSize={config?.max_file_mb * 1024 * 1024}
        onDrop={handleDrop}
      />
      {Object.values(progress).map(p => (
        <ProgressBar key={p.fileName} label={p.fileName} value={p.percent}
                     status={p.status} url={p.destinationUrl} />
      ))}
    </div>
  );
}

// pages/PERReportUpload.tsx  ← different page, same hook, different pageKey
export function PERReportUploadPage() {
  const { uploadFiles, progress } = useFileUpload('per_report_uploads');
  // ... exactly the same UI code ...
}
```

### 15.9 Redis Cache Keys

| Key Pattern | TTL | Content |
|-------------|-----|---------|
| `upload_config:{tenant_id}:{page_key}` | 5 min | Resolved `UploadDestination` object |
| `upload_session:{session_id}` | 1 hr | `UploadSession` state (bytes, adapter state) |
| `upload_quota:{tenant_id}:{user_id}:{date}` | 24 hr | Daily bytes uploaded (rate limit) |

### 15.10 RBAC Permission Model

| Action | Permission Resource | Permission Action | Minimum Role |
|--------|--------------------|--------------------|-------------|
| Upload files | `{page_key}` | `upload` | Contributor |
| Create folders | `{page_key}` | `upload` | Contributor |
| Browse destination | `{page_key}` | `read` | Viewer |
| View own upload history | `files` | `read` | Viewer |
| View all upload history | `files` | `admin` | Manager |
| Manage upload destinations | `upload_config` | `write` | Admin |

The `page_key` is used as the RBAC resource, so each upload page can have independent access control — e.g., one team can upload to the PER reports library while another can only upload to the energy data folder.

---

---

## PART VI — FRONTEND

*This part covers the React frontend. Section 16 describes the provider and component architecture that drives permission-aware rendering and multi-tenant theming. Section 17 is a practical step-by-step integration guide showing exactly how the frontend calls every backend API, manages token lifecycles, embeds Power BI reports, handles WebSocket events, and drives chunked file uploads.*

---

## 16. Frontend Architecture (React)

### 15.1 Project Structure

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

### 15.2 Permission-Driven UI System

#### 15.2.1 PermissionGate Component

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

#### 15.2.2 Permission-Aware Navigation

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

#### 15.2.3 Protected Route System

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

### 15.3 Multi-Tenant Theming System

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

### 15.4 Tenant-Specific Component Customization

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

### 15.5 Permission Matrix Admin UI

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

## 17. Frontend Integration Guide

### 16.1 Application Bootstrap Sequence

```
App Start (user visits analytics.acme-corp.com)
    │
    ▼
[1] GET /api/v1/tenants/resolve?domain=analytics.acme-corp.com
    → Apply branding, font, colors immediately (before any login)
    → Show correct login method(s): [Login with Microsoft] or [Email/Password] or both
    │
    ▼
[2] User logs in (form or Entra SSO)
    → Receive: access_token (in memory), refresh_token (HttpOnly cookie)
    │
    ▼
[3] Parallel bootstrap calls:
    ├── GET /api/v1/users/me              → Store user profile in context
    ├── GET /api/v1/users/me/permissions  → Store in PermissionProvider
    └── GET /api/v1/navigation/menu       → Build sidebar navigation tree
    │
    ▼
[4] Open WebSocket connections (background):
    ├── WS /ws/v1/notifications?token=...
    └── (WS /ws/v1/chat lazily — only when user opens chat panel)
    │
    ▼
[5] Render application shell:
    ├── Sidebar: from menu data (already filtered by backend)
    ├── All UI elements gated by PermissionGate using permission matrix
    └── Navigate user to their default route
```

---

### 16.2 Token Lifecycle Management (Frontend)

```typescript
// src/services/authService.ts

class AuthService {
  private accessToken: string | null = null;   // In-memory only (not localStorage)
  private refreshTimer: ReturnType<typeof setTimeout> | null = null;

  setAccessToken(token: string, expiresIn: number) {
    this.accessToken = token;
    // Schedule refresh 60 seconds before expiry
    this.scheduleRefresh(expiresIn - 60);
  }

  private scheduleRefresh(seconds: number) {
    if (this.refreshTimer) clearTimeout(this.refreshTimer);
    this.refreshTimer = setTimeout(() => this.refresh(), seconds * 1000);
  }

  async refresh(): Promise<void> {
    try {
      // Refresh token is in HttpOnly cookie — sent automatically
      const res = await apiClient.post('/api/v1/auth/refresh');
      this.setAccessToken(res.data.data.access_token, res.data.data.expires_in);
    } catch {
      // Refresh failed → force re-login
      this.logout();
    }
  }

  getAuthHeader(): string {
    return `Bearer ${this.accessToken}`;
  }
}

// Axios interceptor: auto-attach token
apiClient.interceptors.request.use(config => {
  config.headers.Authorization = authService.getAuthHeader();
  return config;
});

// Axios interceptor: handle 401 globally
apiClient.interceptors.response.use(
  res => res,
  async err => {
    if (err.response?.status === 401) {
      await authService.refresh();
      return apiClient(err.config);  // Retry original request
    }
    return Promise.reject(err);
  }
);
```

---

### 16.3 Power BI Embedding (React Component)

```typescript
// src/components/PowerBIReportViewer.tsx
import { models, Report } from 'powerbi-client';
import { PowerBIEmbed } from 'powerbi-client-react';
import { useEffect, useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { powerbiService } from '../services/powerbiService';

interface Props {
  menuItemId: string;
}

export function PowerBIReportViewer({ menuItemId }: Props) {
  const [report, setReport] = useState<Report | null>(null);

  // Fetch embed token from backend
  const { data, isLoading, refetch } = useQuery({
    queryKey: ['pbi-embed-token', menuItemId],
    queryFn: () => powerbiService.getEmbedToken(menuItemId),
    staleTime: Infinity,  // We manage refresh ourselves
  });

  // Auto-refresh token 5 minutes before expiry
  useEffect(() => {
    if (!data) return;
    const expiresAt = new Date(data.expiration).getTime();
    const refreshAt = expiresAt - 5 * 60 * 1000;  // 5 min before
    const delay = Math.max(0, refreshAt - Date.now());

    const timer = setTimeout(async () => {
      await powerbiService.refreshToken(menuItemId);
      refetch();
    }, delay);

    return () => clearTimeout(timer);
  }, [data]);

  // Also listen for WebSocket token refresh events
  const { socket } = useNotificationSocket();
  useEffect(() => {
    if (!socket) return;
    socket.addEventListener('message', (event) => {
      const msg = JSON.parse(event.data);
      if (msg.type === 'report.token_refresh' && msg.report_id === data?.report_id) {
        refetch();
      }
    });
  }, [socket, data]);

  if (isLoading) return <ReportSkeleton />;

  return (
    <PowerBIEmbed
      embedConfig={{
        type:          data!.embed_type ?? 'report',
        id:            data!.report_id,
        embedUrl:      data!.embed_url,
        accessToken:   data!.embed_token,
        tokenType:     models.TokenType.Embed,
        pageName:      data!.page_name ?? undefined,
        filters:       data!.default_filters,
        settings: {
          filterPaneEnabled: data!.settings.filter_pane_enabled,
          navContentPaneEnabled: data!.settings.nav_pane_enabled,
          background: models.BackgroundType.Transparent,
        },
      }}
      getEmbeddedComponent={(embedded) => setReport(embedded as Report)}
      cssClassName="powerbi-report-frame"
    />
  );
}
```

---

### 16.4 WebSocket Notification Client

```typescript
// src/hooks/useNotificationSocket.ts
import { useEffect, useRef, useCallback } from 'react';
import { useAuth } from './useAuth';
import { useQueryClient } from '@tanstack/react-query';

export function useNotificationSocket() {
  const { accessToken } = useAuth();
  const wsRef = useRef<WebSocket | null>(null);
  const queryClient = useQueryClient();

  const connect = useCallback(() => {
    if (!accessToken) return;
    const wsUrl = `${WS_BASE_URL}/ws/v1/notifications?token=${accessToken}`;
    const ws = new WebSocket(wsUrl);

    ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      handleNotification(msg);
    };

    ws.onclose = () => {
      // Reconnect after 3 seconds (exponential backoff in production)
      setTimeout(connect, 3000);
    };

    // Heartbeat: send ping every 30 seconds
    const heartbeat = setInterval(() => {
      if (ws.readyState === WebSocket.OPEN) ws.send('ping');
    }, 30_000);

    ws.onclose = () => {
      clearInterval(heartbeat);
      setTimeout(connect, 3000);
    };

    wsRef.current = ws;
  }, [accessToken]);

  function handleNotification(msg: any) {
    switch (msg.type) {
      case 'permission.changed':
        // Re-fetch permission matrix and menu
        queryClient.invalidateQueries({ queryKey: ['permissions'] });
        queryClient.invalidateQueries({ queryKey: ['navigation-menu'] });
        showToast('Your permissions have been updated.', 'info');
        break;

      case 'notification.info':
      case 'notification.warning':
      case 'notification.error':
        queryClient.invalidateQueries({ queryKey: ['notifications-count'] });
        showToast(msg.message, msg.severity);
        break;

      case 'report.ready':
      case 'report.token_refresh':
        queryClient.invalidateQueries({ queryKey: ['pbi-embed-token'] });
        break;

      case 'system.maintenance':
        showMaintenanceBanner(msg.message, msg.metadata?.scheduled_at);
        break;
    }
  }

  useEffect(() => {
    connect();
    return () => wsRef.current?.close();
  }, [connect]);

  return { socket: wsRef.current };
}
```

---

### 16.5 Menu Rendering & Dynamic Navigation

```typescript
// src/components/layout/Sidebar.tsx
import { useQuery } from '@tanstack/react-query';
import { navigationService } from '../../services/navigationService';
import { NavItem } from './NavItem';

export function Sidebar() {
  const { data: menu, isLoading } = useQuery({
    queryKey:  ['navigation-menu'],
    queryFn:   navigationService.getMenu,
    staleTime: 5 * 60 * 1000,  // 5 minutes; refreshed by permission.changed WS event
  });

  if (isLoading) return <SidebarSkeleton />;

  return (
    <nav className="sidebar">
      {menu?.data.map((item) => (
        <NavItem key={item.id} item={item} />
      ))}
    </nav>
  );
}

// NavItem handles all item types
function NavItem({ item }: { item: MenuItem }) {
  // No permission check here — backend already filtered the list
  switch (item.item_type) {
    case 'page':
      return <Link to={item.route}>{item.label}</Link>;

    case 'powerbi':
      return <Link to={`/reports/${item.id}`}>{item.label}</Link>;
      // Route renders: <PowerBIReportViewer menuItemId={item.id} />

    case 'powerbi_group':
    case 'group_header':
      return (
        <div className="nav-group">
          <span className="nav-group-label">{item.label}</span>
          {item.sub_items.map((child) => (
            <NavItem key={child.id} item={child} />
          ))}
        </div>
      );

    case 'divider':
      return <hr className="nav-divider" />;

    case 'external_link':
      return (
        <a href={item.route} target={item.is_new_tab ? '_blank' : '_self'}>
          {item.label}
        </a>
      );
  }
}
```

---

### 16.6 Content Security Policy Update for Power BI

Power BI embedding requires adding Power BI domains to the CSP:

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://cdn.powerbi.com;
  style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
  font-src 'self' https://fonts.gstatic.com;
  img-src 'self' data: https://cdn.vesper.com https://*.blob.core.windows.net
          https://*.powerbi.com;
  connect-src 'self'
              https://*.vesper-api.com
              https://login.microsoftonline.com
              https://api.powerbi.com
              wss://*.vesper-api.com;
  frame-src 'self' https://app.powerbi.com https://analysis.windows.net;
  frame-ancestors 'none';
  base-uri 'self';
```

> **Note**: `wss://` must be explicitly listed in `connect-src` for WebSocket connections. `frame-src` must include `app.powerbi.com` for the embedded `<iframe>` to load.

---

### 16.7 Redis Cache Keys — Complete Reference

| Purpose | Key Pattern | TTL |
|---------|-------------|-----|
| Session validation | `session:{user_id}` | 15 min |
| Token revocation | `revoked_token:{jti}` | Until token expiry |
| Permission matrix | `perm_matrix:{tenant_id}:{user_id}` | 5 min |
| Per-permission check | `perm:{tenant_id}:{user_id}:{resource}:{action}` | 5 min |
| Tenant config | `tenant_config:{domain}` | 10 min |
| Rate limiting | `rate:{identifier}:{endpoint_class}` | Window duration |
| Power BI access token | `pbi_access_token:{tenant_id}` | Token expiry - 5 min |
| Power BI embed token | `pbi_embed:{tenant_id}:{report_id}:{user_id}` | Token TTL - 5 min |
| Menu tree | `menu:{tenant_id}` | 10 min |
| Chat session history | `chat_history:{session_id}` | 24 hours |
| AI tool results | `ai_tool:{tenant_id}:{tool}:{query_hash}` | 5 min |

---

### 16.8 WebSocket Message Type Registry

| Type | Direction | Description |
|------|-----------|-------------|
| `connection.established` | Server → Client | Connection confirmed |
| `ping` | Client → Server | Keepalive |
| `pong` | Server → Client | Keepalive response |
| `notification.info` | Server → Client | Informational notification |
| `notification.warning` | Server → Client | Warning notification |
| `notification.error` | Server → Client | Error notification |
| `permission.changed` | Server → Client | User's permissions were updated |
| `report.ready` | Server → Client | Power BI report embed token ready |
| `report.token_refresh` | Server → Client | Token refreshed proactively |
| `system.maintenance` | Server → Client | Platform maintenance notice |
| `chat.session.started` | Server → Client | Chat session confirmed |
| `chat.message` | Client → Server | User sends a message |
| `chat.chunk` | Server → Client | Streaming LLM response chunk |
| `chat.tool_call` | Server → Client | Agent is invoking a tool |
| `chat.done` | Server → Client | Response generation complete |
| `chat.error` | Server → Client | Agent encountered an error |
| `chat.stop` | Client → Server | User requests stop generation |

---

**Document End**

*This document serves as the architectural blueprint for the Vesper Analytics multi-tenant platform. All design decisions should be reviewed and approved before implementation begins. Implementation should proceed in phases, starting with the core tenant and user management system, followed by the RBAC engine, Entra integration, WebSocket infrastructure, AI Chat Agent, Power BI integration, and finally the frontend permission system.*

---

## PART VII — REFERENCE & OPERATIONS

*This part serves as the reference layer for the rest of the document. The complete API catalogue (§18) is organised by the order a frontend client calls the APIs — from cold start to full operation. Infrastructure (§19) covers the Azure topology and CI/CD pipeline. The appendix (§20) provides entity relationship diagrams and a key design decisions summary. Use these sections as a reference rather than reading sequentially.*

---

## 18. Complete API Reference — End-to-End

This section catalogs every API endpoint in the order a frontend application would invoke them — from cold start to full operation.

### 17.1 Phase 1: Tenant Resolution (Cold Start)

Called by the React app before any other API, to discover which tenant the user belongs to.

```
GET /api/v1/tenants/resolve?domain={domain}

Auth: None (public)
```

**Request**:
```
GET /api/v1/tenants/resolve?domain=analytics.acme-corp.com
```

**Response** `200 OK`:
```json
{
  "status": "success",
  "data": {
    "tenant_id":      "uuid",
    "tenant_slug":    "acme-corp",
    "display_name":   "Acme Corporation",
    "auth_methods":   ["entra", "form"],
    "branding": {
      "logo_url":          "https://cdn.vesper.com/tenants/acme/logo.svg",
      "favicon_url":       "https://cdn.vesper.com/tenants/acme/favicon.ico",
      "primary_color":     "#1A73E8",
      "secondary_color":   "#F4F6F9",
      "font_family":       "Inter, sans-serif",
      "login_background_url": "https://cdn.vesper.com/tenants/acme/bg.jpg",
      "custom_css":        ""
    },
    "features": {
      "analytics_module": true,
      "ai_chat_enabled":  true,
      "powerbi_enabled":  true
    },
    "entra_config": {
      "client_id":    "azure-app-client-id",
      "authority_url": "https://login.microsoftonline.com/entra-tenant-id",
      "scopes":        ["openid", "profile", "email"]
    }
  }
}
```

**Frontend action**: Store `tenant_id`, apply branding, decide which login options to show.

---

### 17.2 Phase 2: Authentication

#### 17.2.1 Form-Based Login

```
POST /api/v1/auth/login

Auth: None (public)
Rate limit: 5 requests/min per IP
```

**Request**:
```json
{
  "email":      "alice@acme-corp.com",
  "password":   "...",
  "tenant_id":  "uuid"
}
```

**Response** `200 OK`:
```json
{
  "status": "success",
  "data": {
    "access_token":  "eyJ...",
    "token_type":    "bearer",
    "expires_in":    900,
    "mfa_required":  false
  }
}
```
> Refresh token is set as `HttpOnly; SameSite=Strict; Secure` cookie (`vesper_refresh`).

**Response when MFA required** `200 OK`:
```json
{
  "status": "success",
  "data": {
    "access_token": null,
    "mfa_required": true,
    "mfa_token":    "partial-session-token",
    "mfa_methods":  ["totp"]
  }
}
```

#### 17.2.2 MFA Verification

```
POST /api/v1/auth/mfa/verify

Auth: mfa_token (from login response)
```

**Request**:
```json
{
  "mfa_token": "partial-session-token",
  "code":      "123456"
}
```

**Response** `200 OK`: Same as successful login (access_token + refresh cookie).

#### 17.2.3 Microsoft Entra SSO

```
GET /api/v1/auth/entra/authorize?tenant_id={uuid}

Auth: None (redirects to Microsoft)
```

Backend generates the PKCE challenge and returns the Microsoft authorization URL. Frontend redirects the user there.

```
GET /api/v1/auth/entra/callback?code={code}&state={state}

Auth: None (callback from Microsoft)
```

Backend exchanges code for tokens, provisions user if first login, issues Vesper JWT.

#### 17.2.4 Token Refresh

```
POST /api/v1/auth/refresh

Auth: HttpOnly refresh token cookie (automatic)
```

**Response** `200 OK`:
```json
{
  "status": "success",
  "data": {
    "access_token": "eyJ...",
    "expires_in":   900
  }
}
```

#### 17.2.5 Logout

```
POST /api/v1/auth/logout

Auth: Bearer (access token)
```

Revokes the current session. Clears refresh token cookie.

#### 17.2.6 JWKS Endpoint (for token verification)

```
GET /api/v1/auth/.well-known/jwks.json

Auth: None (public)
```

Returns public key set for JWT signature verification.

---

### 17.3 Phase 3: Post-Login Bootstrap

Called immediately after successful login. These three calls can be made in parallel.

#### 17.3.1 Get Current User Profile

```
GET /api/v1/users/me

Auth: Bearer
```

**Response**:
```json
{
  "status": "success",
  "data": {
    "id":           "user-uuid",
    "email":        "alice@acme-corp.com",
    "display_name": "Alice Johnson",
    "first_name":   "Alice",
    "last_name":    "Johnson",
    "avatar_url":   "https://...",
    "user_type":    "standard",
    "auth_provider":"entra",
    "mfa_enabled":  true,
    "roles":        ["analyst"],
    "groups":       ["finance", "reporting"],
    "last_login_at":"2026-02-26T09:00:00Z"
  }
}
```

#### 17.3.2 Get Permission Matrix

```
GET /api/v1/users/me/permissions

Auth: Bearer
```

Returns the complete permission matrix. Cached in Redis (TTL 5 min).

**Response**:
```json
{
  "status": "success",
  "data": {
    "dashboard":                       {"view": true},
    "dashboard.revenue_chart":         {"view": true},
    "dashboard.export_button":         {"view": false},
    "reports":                         {"view": true, "export": true},
    "reports.sales":                   {"view": true, "export": true},
    "reports.customers":               {"view": true, "export": false},
    "admin.users":                     {"view": false},
    "ai.chat":                         {"use": true},
    "powerbi.sales_performance":       {"view": true},
    "powerbi.customer_insights":       {"view": false}
  }
}
```

**Frontend action**: Store in `PermissionProvider` context. Gate all UI elements using this data. On `permission.changed` WebSocket event, re-fetch this endpoint.

#### 17.3.3 Get Navigation Menu

```
GET /api/v1/navigation/menu

Auth: Bearer
```

Returns the permission-filtered menu tree (see Section 17.4 for full schema).

---

### 17.4 Phase 4: Menu-Driven Navigation

#### 17.4.1 Sub-Menu Lazy Loading

```
GET /api/v1/navigation/menu/{item_id}/sub-menu

Auth: Bearer
```

For deep hierarchies — load sub-items on demand when user expands a menu section.

#### 17.4.2 Menu Item Detail

```
GET /api/v1/navigation/menu/{item_id}

Auth: Bearer
```

Get metadata for a specific menu item (type, route, Power BI config reference).

---

### 17.5 Phase 5: Power BI Report Loading

When user navigates to a Power BI menu item (item_type = 'powerbi'):

#### Step 1 — Get Report Config + Embed Token

```
GET /api/v1/powerbi/reports/{menu_item_id}/embed-token

Auth: Bearer
Permission: [menu item's resource_key]:view
```

Single API call that returns everything the frontend needs to render the report.

**Response** (see Section 17.5 for full schema).

#### Step 2 — Report Renders in Frontend

Frontend uses the `embedUrl` + `embedToken` to initialize the Power BI JavaScript SDK. No credentials are ever passed to the frontend — only the short-lived embed token.

#### Step 3 — Token Auto-Refresh

```
POST /api/v1/powerbi/reports/{menu_item_id}/refresh-token

Auth: Bearer
```

Called by frontend 5 minutes before token expiry. Returns fresh embed token. If a valid cached token exists in Redis it is returned without a new Power BI API call.

---

### 17.6 Phase 6: Real-Time Connections

After the page loads, establish WebSocket connections:

```
# Notifications
WS  /ws/v1/notifications?token={access_token}

# Chat Agent (on-demand, when user opens chat)
WS  /ws/v1/chat?token={access_token}&session_id={optional}
```

---

### 17.7 User Management APIs

| Method | Endpoint | Description | Auth | Permission |
|--------|----------|-------------|------|------------|
| `GET` | `/api/v1/users` | List users (paginated, filtered) | Bearer | `admin.users:view` |
| `POST` | `/api/v1/users` | Create user | Bearer | `admin.users:create` |
| `GET` | `/api/v1/users/{id}` | Get user detail | Bearer | `admin.users:view` |
| `PUT` | `/api/v1/users/{id}` | Update user profile | Bearer | `admin.users:edit` |
| `PATCH` | `/api/v1/users/{id}/status` | Activate/deactivate user | Bearer | `admin.users:edit` |
| `DELETE` | `/api/v1/users/{id}` | Deactivate user | Bearer | `admin.users:delete` |
| `GET` | `/api/v1/users/{id}/roles` | Get user's roles | Bearer | `admin.users:view` |
| `PUT` | `/api/v1/users/{id}/roles` | Set user roles | Bearer | `admin.users:edit` |
| `GET` | `/api/v1/users/{id}/groups` | Get user's groups | Bearer | `admin.users:view` |
| `PUT` | `/api/v1/users/{id}/groups` | Set user groups | Bearer | `admin.users:edit` |
| `GET` | `/api/v1/users/{id}/permissions` | Get effective permissions | Bearer | `admin.users:view` |
| `GET` | `/api/v1/users/me` | Current user profile | Bearer | — |
| `PUT` | `/api/v1/users/me` | Update own profile | Bearer | — |
| `POST` | `/api/v1/users/me/change-password` | Change own password | Bearer | — |
| `GET` | `/api/v1/users/me/permissions` | Own permission matrix | Bearer | — |
| `GET` | `/api/v1/users/me/sessions` | Active sessions | Bearer | — |
| `DELETE` | `/api/v1/users/me/sessions/{id}` | Revoke a session | Bearer | — |

---

### 17.8 Group Management APIs

| Method | Endpoint | Description | Auth | Permission |
|--------|----------|-------------|------|------------|
| `GET` | `/api/v1/groups` | List groups | Bearer | `admin.groups:view` |
| `POST` | `/api/v1/groups` | Create group | Bearer | `admin.groups:create` |
| `GET` | `/api/v1/groups/{id}` | Get group detail | Bearer | `admin.groups:view` |
| `PUT` | `/api/v1/groups/{id}` | Update group | Bearer | `admin.groups:edit` |
| `DELETE` | `/api/v1/groups/{id}` | Delete group | Bearer | `admin.groups:delete` |
| `GET` | `/api/v1/groups/{id}/members` | List group members | Bearer | `admin.groups:view` |
| `POST` | `/api/v1/groups/{id}/members` | Add members to group | Bearer | `admin.groups:edit` |
| `DELETE` | `/api/v1/groups/{id}/members/{user_id}` | Remove member | Bearer | `admin.groups:edit` |
| `GET` | `/api/v1/groups/{id}/roles` | Get group's roles | Bearer | `admin.groups:view` |
| `PUT` | `/api/v1/groups/{id}/roles` | Set group roles | Bearer | `admin.groups:edit` |

---

### 17.9 Role & Permission APIs

| Method | Endpoint | Description | Auth | Permission |
|--------|----------|-------------|------|------------|
| `GET` | `/api/v1/roles` | List roles | Bearer | `admin.roles:view` |
| `POST` | `/api/v1/roles` | Create role | Bearer | `admin.roles:create` |
| `GET` | `/api/v1/roles/{id}` | Get role detail | Bearer | `admin.roles:view` |
| `PUT` | `/api/v1/roles/{id}` | Update role | Bearer | `admin.roles:edit` |
| `DELETE` | `/api/v1/roles/{id}` | Delete role (non-system) | Bearer | `admin.roles:delete` |
| `GET` | `/api/v1/roles/{id}/permissions` | Get role's permission matrix | Bearer | `admin.roles:view` |
| `PUT` | `/api/v1/roles/{id}/permissions` | Bulk update permissions | Bearer | `admin.roles:edit` |
| `GET` | `/api/v1/resources` | List all resources | Bearer | `admin.roles:view` |
| `POST` | `/api/v1/resources` | Register new resource | Bearer | `super_admin` |
| `GET` | `/api/v1/permissions/matrix` | Full permission matrix grid | Bearer | `admin.roles:view` |

---

### 17.10 Navigation & Menu Admin APIs

| Method | Endpoint | Description | Auth | Permission |
|--------|----------|-------------|------|------------|
| `GET` | `/api/v1/navigation/menu` | Get user's filtered menu | Bearer | — (filtered by perms) |
| `GET` | `/api/v1/navigation/menu/{id}/sub-menu` | Lazy-load sub-items | Bearer | — (filtered) |
| `GET` | `/api/v1/admin/navigation/menu` | Admin: full menu tree (all items) | Bearer | `admin.menu:view` |
| `POST` | `/api/v1/admin/navigation/menu` | Admin: create menu item | Bearer | `admin.menu:create` |
| `PUT` | `/api/v1/admin/navigation/menu/{id}` | Admin: update menu item | Bearer | `admin.menu:edit` |
| `DELETE` | `/api/v1/admin/navigation/menu/{id}` | Admin: delete menu item | Bearer | `admin.menu:delete` |
| `POST` | `/api/v1/admin/navigation/menu/reorder` | Admin: reorder items | Bearer | `admin.menu:edit` |
| `GET` | `/api/v1/admin/navigation/menu/{id}/assign-report` | Get report assignment | Bearer | `admin.menu:view` |
| `PUT` | `/api/v1/admin/navigation/menu/{id}/assign-report` | Assign Power BI report | Bearer | `admin.menu:edit` |

---

### 17.11 AI Chat APIs

| Method | Endpoint | Description | Auth | Permission |
|--------|----------|-------------|------|------------|
| `WS` | `/ws/v1/chat` | Bidirectional chat WebSocket | Bearer (query param) | `ai.chat:use` |
| `GET` | `/api/v1/chat/sessions` | List chat sessions | Bearer | `ai.chat:use` |
| `POST` | `/api/v1/chat/sessions` | Create session | Bearer | `ai.chat:use` |
| `GET` | `/api/v1/chat/sessions/{id}` | Get session | Bearer | `ai.chat:use` |
| `DELETE` | `/api/v1/chat/sessions/{id}` | Delete session | Bearer | `ai.chat:use` |
| `GET` | `/api/v1/chat/sessions/{id}/messages` | Get history (paginated) | Bearer | `ai.chat:use` |
| `GET` | `/api/v1/chat/tools` | List enabled agent tools | Bearer | `ai.chat:use` |
| `PUT` | `/api/v1/admin/chat/tools/{name}` | Enable/disable tool | Bearer | `admin.ai:configure` |

---

### 17.12 Notification APIs

| Method | Endpoint | Description | Auth |
|--------|----------|-------------|------|
| `WS` | `/ws/v1/notifications` | Notification push channel | Bearer (query param) |
| `GET` | `/api/v1/notifications` | List notifications | Bearer |
| `GET` | `/api/v1/notifications/unread-count` | Unread count for badge | Bearer |
| `PATCH` | `/api/v1/notifications/{id}/read` | Mark as read | Bearer |
| `POST` | `/api/v1/notifications/read-all` | Mark all as read | Bearer |
| `DELETE` | `/api/v1/notifications/{id}` | Delete notification | Bearer |

---

### 17.13 Tenant Management APIs (Super Admin)

| Method | Endpoint | Description | Auth | Role |
|--------|----------|-------------|------|------|
| `GET` | `/api/v1/tenants` | List all tenants | Bearer | `super_admin` |
| `POST` | `/api/v1/tenants` | Create tenant | Bearer | `super_admin` |
| `GET` | `/api/v1/tenants/{id}` | Tenant detail | Bearer | `super_admin` |
| `PUT` | `/api/v1/tenants/{id}` | Update tenant | Bearer | `super_admin` |
| `PATCH` | `/api/v1/tenants/{id}/status` | Suspend/activate | Bearer | `super_admin` |
| `POST` | `/api/v1/tenants/{id}/domains` | Add custom domain | Bearer | `super_admin` |
| `DELETE` | `/api/v1/tenants/{id}/domains/{domain_id}` | Remove domain | Bearer | `super_admin` |
| `GET` | `/api/v1/tenants/{id}/config` | Get config values | Bearer | `super_admin` |
| `PUT` | `/api/v1/tenants/{id}/config` | Update config values | Bearer | `super_admin` |
| `GET` | `/api/v1/tenants/{id}/theme` | Get theme | Bearer | `tenant_admin` |
| `PUT` | `/api/v1/tenants/{id}/theme` | Update theme/branding | Bearer | `tenant_admin` |

---

### 17.14 Entra Integration APIs

| Method | Endpoint | Description | Auth | Permission |
|--------|----------|-------------|------|------------|
| `GET` | `/api/v1/auth/entra/authorize` | Start Entra OIDC flow | Public | — |
| `GET` | `/api/v1/auth/entra/callback` | Entra OIDC callback | Public | — |
| `GET` | `/api/v1/entra/config` | Get Entra OIDC config | Bearer | `admin.entra:view` |
| `PUT` | `/api/v1/entra/config` | Update Entra config | Bearer | `admin.entra:edit` |
| `GET` | `/api/v1/entra/groups` | List Entra group mappings | Bearer | `admin.entra:view` |
| `POST` | `/api/v1/entra/groups/map` | Create group mapping | Bearer | `admin.entra:edit` |
| `DELETE` | `/api/v1/entra/groups/map/{id}` | Remove group mapping | Bearer | `admin.entra:edit` |
| `POST` | `/api/v1/entra/sync` | Trigger manual group sync | Bearer | `admin.entra:sync` |
| `GET` | `/api/v1/entra/sync/status` | Get last sync status | Bearer | `admin.entra:view` |

---

### 17.15 Audit & System APIs

| Method | Endpoint | Description | Auth | Permission |
|--------|----------|-------------|------|------------|
| `GET` | `/api/v1/audit/logs` | Query audit log | Bearer | `admin.audit:view` |
| `GET` | `/api/v1/audit/login-history` | Login history | Bearer | `admin.audit:view` |
| `GET` | `/api/v1/health` | Health check | Public | — |
| `GET` | `/api/v1/health/ready` | Readiness check (DB + Redis) | Public | — |
| `GET` | `/api/v1/versions` | API version discovery | Public | — |
| `GET` | `/api/v1/auth/.well-known/jwks.json` | Public key set | Public | — |

---

## 19. Azure Deployment & Infrastructure

### 18.1 Infrastructure Architecture

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

### 18.2 Auto-Scaling Configuration

| Service | Min | Max | Scale Trigger | Cooldown |
|---------|-----|-----|--------------|----------|
| API Gateway | 2 | 20 | CPU > 70% OR concurrent requests > 100 | 60s |
| Auth Service | 2 | 10 | CPU > 60% OR request latency P95 > 500ms | 60s |
| Core API | 3 | 30 | CPU > 70% OR concurrent requests > 200 | 90s |
| User Mgmt | 2 | 15 | CPU > 70% OR queue depth > 50 | 60s |
| Workers | 1 | 5 | Queue depth > 100 messages | 120s |

### 18.3 CI/CD Pipeline

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

### 18.4 Environment Strategy

| Environment | Purpose | Azure SQL | Redis | Infra |
|------------|---------|-----------|-------|-------|
| **Local** | Developer machine | Docker SQL Server | Docker Redis | docker-compose |
| **Dev** | Integration testing | Azure SQL (Basic) | Redis (Basic) | Container Apps (min scale) |
| **Staging** | Pre-production validation | Azure SQL (Standard) | Redis (Standard) | Container Apps (prod-like) |
| **Production** | Live traffic | Azure SQL (Business Critical) | Redis (Premium) | Container Apps (auto-scale) |

---

## 20. Appendix: Entity Relationship Diagrams

### 19.1 Complete Permission Resolution - Visual Flow

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

### 19.2 Entra ID Integration Flow

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

### 19.3 Key Design Decisions Summary

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

---

*This document is the authoritative architectural blueprint for the Vesper Analytics multi-tenant platform. All design decisions must be reviewed and approved before implementation begins. Implementation should proceed through the six phases defined in Section 1.4, starting with the database schema and authentication service, followed by the RBAC engine, backend middleware, infrastructure, platform features, and finally the frontend integration.*
