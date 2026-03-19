# Internal Integration Guide — Commercial NPV Tool

**Document Version:** 1.0
**Date:** 2026-03-02
**Classification:** Internal — Confidential
**Author:** Platform Engineering Team
**Audience:** Platform Team, Commercial Tool Team, DevOps, CTO

---

## How to Read This Document

This document defines the strategy and implementation blueprint for integrating the **Commercial NPV Tool** (built by a separate team, hosted in GitHub) into the **Vesper DataHub Platform** (hosted in Azure DevOps). The result is:

- Commercial NPV Tool users log in through Vesper's existing auth system
- Access to the Commercial NPV page is controlled by Vesper RBAC (permission-gated menu item)
- The Commercial NPV Tool's frontend and backend are **never directly accessible** from the public internet
- Both platforms are deployed in Azure with separate but coordinated CI/CD pipelines

**Audience guide:**

| Role | Read These Sections |
|------|-------------------|
| **CTO / Architect** | §1, §2, §3 |
| **Platform Backend Engineer** | §3, §4, §5 |
| **Commercial Team (Frontend + Backend)** | §2, §4, §6 |
| **DevOps** | §5, §7 |
| **Security Reviewer** | §3, §4.4 |

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current State Analysis](#2-current-state-analysis)
3. [Integration Architecture](#3-integration-architecture)
4. [Security Model — Preventing Direct Access](#4-security-model--preventing-direct-access)
5. [Azure Deployment — Commercial NPV Tool](#5-azure-deployment--commercial-npv-tool)
6. [Changes Required by Each Team](#6-changes-required-by-each-team)
7. [CI/CD Pipeline — GitHub Repo to Azure](#7-cicd-pipeline--github-repo-to-azure)
8. [Team Collaboration Protocol](#8-team-collaboration-protocol)
9. [Rollout Checklist](#9-rollout-checklist)

---

## 1. Executive Summary

The **Commercial NPV Tool** is a self-contained React + FastAPI application built by a separate commercial team and hosted in a GitHub repository. It currently has **no authentication or access control**.

The **Vesper DataHub Platform** already has a production-grade auth and RBAC system (JWT RS256, role-based permissions, dynamic menu system, multi-tenant isolation).

### Goal

Integrate the Commercial NPV Tool into Vesper DataHub so that:

| Requirement | How It Is Met |
|------------|--------------|
| Users must be authenticated via Vesper to access the Commercial NPV Tool | Vesper JWT is required; the Commercial Tool backend only accepts requests from the Vesper API Gateway |
| Access is controlled per user/role | New RBAC resource `commercial_npv` added; Vesper menu item is gated by this permission |
| The Commercial NPV frontend is never directly accessible | Deployed with **internal-only ingress** (no public URL); served only through Vesper's frontend as an embedded page |
| The Commercial NPV backend is never directly accessible | Deployed with **internal ingress only** inside the Azure Container Apps VNET; no public endpoint |
| Commercial team continues working independently in GitHub | Their repo remains in GitHub; CI/CD deploys into the shared Azure Container Apps environment |
| No auth code changes in the Commercial Tool | Vesper API Gateway acts as an authenticated reverse proxy; the Commercial backend trusts the internal network |

### Integration Pattern: Authenticated Reverse Proxy + Embedded Frontend

```
Vesper React App (authenticated user)
        │
        │  User clicks "Commercial NPV" in the nav menu
        │  (menu item visible only if user has permission: commercial_npv:view)
        │
        ▼
Vesper Frontend loads /commercial/npv route
        │
        │  React lazy-loads the Commercial NPV micro-frontend bundle
        │  OR renders an iframe pointing to the embedded URL
        │
        ▼
Vesper API Gateway  (/api/v1/commercial/npv/**)
        │
        │  1. Validates Vesper JWT
        │  2. Checks RBAC: commercial_npv:view permission
        │  3. Injects X-Vesper-User-Id, X-Vesper-Tenant-Id headers
        │  4. Forwards to Commercial NPV Backend (internal VNET only)
        │
        ▼
Commercial NPV Backend (Container App — internal ingress only)
        │
        │  Trusts X-Vesper-* headers (only reachable from VNET)
        │  Returns data to Vesper Gateway
        │
        ▼
Response flows back through Gateway → Frontend → User
```

---

## 2. Current State Analysis

### 2.1 Vesper DataHub Platform (Azure DevOps)

| Component | Status | Azure Service |
|-----------|--------|--------------|
| React 18 Frontend | Production | Azure Static Web App |
| FastAPI Backend | Production | Azure Container Apps |
| Auth (JWT RS256 + Entra OIDC) | Production | Container Apps |
| RBAC + Dynamic Menu | Production | Azure SQL + Redis |
| CI/CD | Azure Pipelines (Azure DevOps) | Azure DevOps |

### 2.2 Commercial NPV Tool (GitHub)

| Component | Status | Notes |
|-----------|--------|-------|
| React Frontend | Built, no auth | Simple tool, no login, no token handling |
| FastAPI Backend | Built, no auth | CRUD + calculation APIs, no middleware for auth |
| CI/CD | None / manual | GitHub repo, no pipeline to Azure |
| Secrets / Config | .env files | No Azure Key Vault integration |
| Containerisation | Unknown — **must confirm with commercial team** | May need Dockerfile added |

### 2.3 Gap Analysis

```
                    VESPER HAS                    COMMERCIAL TOOL LACKS
                ─────────────────               ──────────────────────────
                ✅ JWT Auth                      ❌ Any authentication
                ✅ RBAC (per resource)           ❌ Access control
                ✅ Multi-tenant isolation         ❌ Tenant awareness
                ✅ Azure deployment               ❌ Azure deployment pipeline
                ✅ Key Vault for secrets          ❌ Secure secret storage
                ✅ Container Registry             ❌ Published Docker image
                ✅ WAF / Front Door               ❌ Network security
                ✅ App Insights monitoring        ❌ Observability
```

### 2.4 Chosen Integration Approach: Reverse Proxy (no auth code changes to Commercial Tool)

We deliberately avoid modifying the Commercial Tool's authentication logic. This keeps the commercial team's repo independent and avoids coupling their codebase to Vesper's JWT implementation. Instead:

- **Auth is enforced entirely by the Vesper API Gateway** before any request reaches the Commercial backend.
- The Commercial backend is **not reachable from the public internet** — only from inside the VNET.
- The Commercial frontend bundle is **served through the Vesper deployment** or behind an internal URL that requires a valid Vesper session to render.

---

## 3. Integration Architecture

### 3.1 Full Architecture Diagram

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        AZURE INFRASTRUCTURE                                │
│                                                                            │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Azure Front Door (Premium) — same AFD instance as Vesper           │  │
│  │  • Routes *.vesper.com and tenant domains → Static Web App          │  │
│  │  • Routes /api/** → Container Apps API Gateway                      │  │
│  │  • WAF blocks all direct access to internal services                │  │
│  └────────────────┬─────────────────────────────────────────────────────┘  │
│                   │                                                         │
│        ┌──────────▼──────────┐     ┌──────────────────────────────────┐    │
│        │ Azure Static Web App │     │  Container Apps Environment      │    │
│        │ (Vesper React SPA)   │     │  (VNET-integrated)               │    │
│        │                      │     │                                  │    │
│        │ • Serves Vesper UI   │     │  ┌─────────────────────────────┐ │    │
│        │ • Includes Commercial│     │  │ Vesper API Gateway          │ │    │
│        │   NPV module (lazy   │     │  │ (public ingress)            │ │    │
│        │   loaded or iframe)  │     │  │                             │ │    │
│        └──────────────────────┘     │  │ /api/v1/commercial/npv/**   │ │    │
│                                     │  │   → validates JWT           │ │    │
│                                     │  │   → checks RBAC             │ │    │
│                                     │  │   → proxies to internal     │ │    │
│                                     │  └──────────────┬──────────────┘ │    │
│                                     │                 │                 │    │
│                                     │  ┌──────────────▼──────────────┐ │    │
│                                     │  │ Commercial NPV Backend      │ │    │
│                                     │  │ (INTERNAL ingress only)     │ │    │
│                                     │  │ No public endpoint          │ │    │
│                                     │  │ Reachable only from VNET    │ │    │
│                                     │  └─────────────────────────────┘ │    │
│                                     │                                  │    │
│                                     │  ┌─────────────────────────────┐ │    │
│                                     │  │ (All existing Vesper        │ │    │
│                                     │  │  services unchanged)        │ │    │
│                                     │  └─────────────────────────────┘ │    │
│                                     └──────────────────────────────────┘    │
│                                                                            │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Shared Data Layer (Private Endpoints in VNET)                       │  │
│  │  Azure SQL  │  Azure Redis  │  Azure Service Bus  │  Key Vault       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Request Flow — User Accesses the Commercial NPV Page

```
1. User logs into Vesper DataHub (existing login flow)
   → Vesper issues JWT + refresh token (HttpOnly cookie)

2. GET /api/v1/navigation/menu
   → Vesper returns menu tree
   → If user has commercial_npv:view permission, menu item "Commercial NPV" appears
   → If not, the menu item is absent (no 403 — just not shown)

3. User clicks "Commercial NPV" in the Vesper sidebar

4. Vesper React Router navigates to /commercial/npv
   → React lazy-loads CommercialNPVPage component
   → Component makes API calls to /api/v1/commercial/npv/** (same origin)

5. Vesper API Gateway receives /api/v1/commercial/npv/**
   → MW-1 CorrelationId assigned
   → MW-3 RateLimit applied
   → MW-4 TenantContext extracted from JWT
   → MW-5 JWTAuth validates signature
   → MW-7 Permission check: commercial_npv:view
   → If authorized: strip auth headers, inject X-Vesper-* headers,
     forward to http://commercial-npv-backend (internal DNS)

6. Commercial NPV Backend (internal only)
   → Receives request with X-Vesper-User-Id, X-Vesper-Tenant-Id
   → Processes and returns data

7. Vesper Gateway returns response → React component → User
```

### 3.3 Vesper Menu System Integration

Following the pattern defined in Section 14 of the main README, a new menu item is inserted into `menu_items`:

```sql
-- Add parent menu item (if commercial tools need a section)
INSERT INTO menu_items (
    tenant_id, parent_id, label, item_type,
    resource_key, icon, sort_order, is_active
) VALUES (
    @tenant_id, NULL, 'Commercial Tools', 'section',
    'commercial_section', 'Briefcase', 90, 1
);

-- Add Commercial NPV Tool page item
INSERT INTO menu_items (
    tenant_id, parent_id, label, item_type,
    route_path, resource_key, icon, sort_order, is_active
) VALUES (
    @tenant_id,
    (SELECT id FROM menu_items WHERE resource_key = 'commercial_section'
     AND tenant_id = @tenant_id),
    'Commercial NPV',
    'embedded_app',                          -- new item_type
    '/commercial/npv',
    'commercial_npv',                        -- RBAC resource key
    'Calculator',
    10,
    1
);
```

New RBAC resource to add to the permission matrix:

```
Resource Key: commercial_npv
Actions: view, export
Description: Access to the Commercial NPV calculation tool
```

---

## 4. Security Model — Preventing Direct Access

### 4.1 Threat Model

| Threat | Mitigation |
|--------|-----------|
| User tries to access Commercial NPV frontend URL directly in browser | Frontend is NOT served at a public URL — only embedded in Vesper's React app |
| User tries to call Commercial NPV backend API directly (bypassing Vesper gateway) | Backend has **internal ingress only** in Container Apps — zero public endpoint; only reachable from VNET |
| Authenticated Vesper user without `commercial_npv:view` accesses the API | Vesper API Gateway RBAC middleware rejects with HTTP 403 before request reaches Commercial backend |
| Commercial Team exposes an API accidentally | WAF rule on Front Door blocks all requests to `commercial-npv-backend.*` domains |
| Token replay attack | Commercial backend is on internal VNET — no external token to replay; Vesper JWT has 15 min TTL |
| SSRF through the proxy endpoint | Proxy route is hardcoded in Vesper gateway config — only `/api/v1/commercial/npv/**` is proxied; no user-controlled target URLs |

### 4.2 Commercial Backend — Internal Ingress Configuration

When deploying the Commercial NPV Backend as a Container App, set ingress to **internal**:

```json
// Container App ingress config (bicep / ARM)
"ingress": {
    "external": false,          // ← CRITICAL: false = internal VNET only
    "targetPort": 8000,
    "transport": "http",
    "allowInsecure": false
}
```

This means:
- The Container App gets a VNET-internal DNS name: `commercial-npv-backend.internal.<env>.azurecontainerapps.io`
- There is **no public IP** or public hostname
- Azure Front Door **cannot** route to it even if misconfigured

### 4.3 Vesper API Gateway — Proxy Route Configuration

Add to the Vesper FastAPI gateway a dedicated proxy router:

```python
# app/routers/commercial_proxy.py

import httpx
from fastapi import APIRouter, Request, Response, Depends
from app.middleware.auth import get_current_user
from app.middleware.permissions import require_permission
from app.config import settings

router = APIRouter(prefix="/api/v1/commercial/npv", tags=["commercial-npv"])

COMMERCIAL_BACKEND_URL = settings.COMMERCIAL_NPV_INTERNAL_URL
# e.g. "http://commercial-npv-backend.internal.<env>.azurecontainerapps.io"

@router.api_route(
    "/{path:path}",
    methods=["GET", "POST", "PUT", "DELETE", "PATCH"],
)
async def proxy_commercial_npv(
    path: str,
    request: Request,
    current_user=Depends(get_current_user),
    _=Depends(require_permission("commercial_npv", "view")),
):
    """
    Authenticated reverse proxy to the Commercial NPV backend.
    JWT is validated and RBAC is checked BEFORE proxying.
    The downstream service receives no auth token — it trusts the VNET.
    """
    target_url = f"{COMMERCIAL_BACKEND_URL}/{path}"

    # Build forwarded headers — strip auth, inject user context
    forward_headers = {
        k: v for k, v in request.headers.items()
        if k.lower() not in ("authorization", "cookie", "host")
    }
    forward_headers["X-Vesper-User-Id"] = str(current_user.user_id)
    forward_headers["X-Vesper-Tenant-Id"] = str(current_user.tenant_id)
    forward_headers["X-Vesper-User-Email"] = current_user.email
    forward_headers["X-Vesper-Roles"] = ",".join(current_user.roles)
    forward_headers["X-Forwarded-For"] = request.client.host

    body = await request.body()

    async with httpx.AsyncClient(timeout=30.0) as client:
        response = await client.request(
            method=request.method,
            url=target_url,
            headers=forward_headers,
            params=request.query_params,
            content=body,
        )

    return Response(
        content=response.content,
        status_code=response.status_code,
        headers=dict(response.headers),
        media_type=response.headers.get("content-type"),
    )
```

Add to environment variables (via Key Vault reference):
```
COMMERCIAL_NPV_INTERNAL_URL=http://commercial-npv-backend.internal.<env>.azurecontainerapps.io
```

### 4.4 Commercial Frontend — Two Deployment Options

Choose **Option A** (recommended) or **Option B** based on commercial team capacity.

#### Option A — React Micro-Frontend Bundle (Recommended)

The Commercial NPV Tool's React app is **built as a library bundle** and hosted as a static asset on the Vesper Static Web App. No separate frontend deployment needed.

```
Build output:  commercial-npv-ui.js  (Vite library mode)
Hosted at:     https://app.vesper.com/assets/commercial-npv-ui.js
Loaded by:     Vesper React app lazily when user navigates to /commercial/npv
```

This means:
- The Commercial NPV UI has **no standalone URL**
- There is nothing to "directly access" — it's just a JS bundle that only works when rendered inside the Vesper authenticated shell
- All API calls from the bundle use relative paths (`/api/v1/commercial/npv/...`) — they automatically go through the Vesper gateway

Commercial team Vite config change:

```js
// vite.config.ts (Commercial NPV tool)
export default defineConfig({
  build: {
    lib: {
      entry: 'src/main.tsx',
      name: 'CommercialNPV',
      fileName: 'commercial-npv-ui',
      formats: ['es'],
    },
    rollupOptions: {
      external: ['react', 'react-dom'],   // use Vesper's React instance
      output: {
        globals: { react: 'React', 'react-dom': 'ReactDOM' },
      },
    },
  },
})
```

Vesper React integration:

```tsx
// src/pages/CommercialNPVPage.tsx  (in Vesper frontend)
import React, { Suspense, lazy } from 'react';
import { usePermission } from '../hooks/usePermission';

// Loaded dynamically from CDN asset — only when user navigates here
const CommercialNPVApp = lazy(() =>
  import(/* webpackChunkName: "commercial-npv" */ '/assets/commercial-npv-ui.js')
    .then(m => ({ default: m.CommercialNPV }))
);

export function CommercialNPVPage() {
  const canView = usePermission('commercial_npv', 'view');

  if (!canView) return <Navigate to="/403" />;  // belt and suspenders

  return (
    <Suspense fallback={<LoadingSpinner />}>
      <CommercialNPVApp
        apiBaseUrl="/api/v1/commercial/npv"   // proxied through Vesper gateway
      />
    </Suspense>
  );
}
```

#### Option B — Iframe with Session Token

If building a library bundle is too disruptive for the Commercial team, the Vesper frontend can embed the Commercial NPV frontend in an **iframe** with a short-lived session token. The Commercial NPV frontend (deployed as a separate Static Web App) validates this token before rendering.

```
Flow:
1. Vesper backend issues a short-lived signed token (5 min TTL) for the user
   GET /api/v1/commercial/npv/session-token → { token, embed_url }

2. Vesper React renders:
   <iframe src="{embed_url}?session_token={token}" sandbox="allow-scripts allow-same-origin" />

3. Commercial NPV frontend (on first load):
   → Reads session_token from URL params
   → Calls POST /api/v1/commercial/npv/validate-session { token }
     (goes through Vesper gateway, which validates it against Redis cache)
   → On success, renders normally
   → On failure/expiry, shows "Session expired — please return to Vesper"

4. Commercial NPV frontend has NO login page.
   If session_token is absent → renders blank page with no useful content.
```

**Tradeoffs:**

| | Option A (Bundle) | Option B (Iframe + Token) |
|--|-------------------|--------------------------|
| Commercial team effort | Medium (Vite lib mode config) | Low (small token validation hook) |
| Security | Higher (no public URL at all) | Good (token + short TTL) |
| UX | Seamless (same DOM) | Slight iframe isolation |
| Deployment complexity | Lower (single static web app) | Higher (two static web apps) |
| Recommended for | Default | When Option A is not feasible |

---

## 5. Azure Deployment — Commercial NPV Tool

### 5.1 Resource Group Strategy

The Commercial NPV Tool is deployed **into the existing Vesper resource group**:

```
Resource Group: rg-vesper-production
├── (all existing Vesper resources)
├── ca-commercial-npv-backend     ← NEW: Container App (internal ingress)
└── (if Option B: swa-commercial-npv-frontend)  ← NEW: Static Web App
```

Using the same resource group gives:
- Shared VNET — Commercial backend can communicate with Vesper services internally
- Shared Key Vault — Commercial team's secrets go in the same vault
- Shared Container Registry — Commercial Docker images use the same ACR
- Shared monitoring (App Insights, Log Analytics)

### 5.2 Commercial NPV Backend Deployment Spec

```
Azure Resource:     Azure Container App
Name:               ca-commercial-npv-backend
Resource Group:     rg-vesper-production
Container Apps Env: cae-vesper-production  (shared with Vesper)
Ingress:            Internal (VNET only — no public endpoint)
Target Port:        8000
Min Replicas:       1
Max Replicas:       5
Scale Trigger:      CPU > 70%
CPU:                0.5 vCPU
Memory:             1.0 Gi
Container Registry: acrvesperproduction.azurecr.io
Image:              acrvesperproduction.azurecr.io/commercial-npv-backend:latest
Environment Vars:   (from Key Vault references)
  - DATABASE_URL    → kv-vesper/commercial-npv-db-url
  - (other app vars as needed)
```

### 5.3 Commercial NPV Frontend Deployment Spec (Option A — Bundle)

No separate Azure resource required. The build artifact (`commercial-npv-ui.js`) is copied into the Vesper Static Web App's `/public/assets/` directory as part of the Vesper CI/CD pipeline.

```
Pipeline step (in Vesper's Azure Pipeline or GitHub Actions):
1. Trigger: after Commercial NPV frontend build completes (artifact published)
2. Download artifact: commercial-npv-ui.js
3. Copy to: vesper-frontend/public/assets/commercial-npv-ui.js
4. Deploy Vesper Static Web App (includes the bundle)
```

### 5.4 Commercial NPV Frontend Deployment Spec (Option B — Separate Static Web App)

```
Azure Resource:     Azure Static Web App
Name:               swa-commercial-npv
Resource Group:     rg-vesper-production
SKU:                Standard (required for private endpoints)
Build Preset:       React
App Location:       /                   (repo root)
Output Location:    dist
API Location:       (none — backend is separate)
```

Static Web App configuration (`staticwebapp.config.json`) to block direct access:

```json
{
  "globalHeaders": {
    "X-Frame-Options": "SAMEORIGIN",
    "Content-Security-Policy": "frame-ancestors 'self' https://*.vesper.com",
    "X-Content-Type-Options": "nosniff"
  },
  "navigationFallback": {
    "rewrite": "/index.html"
  },
  "responseOverrides": {
    "401": {
      "rewrite": "/index.html",
      "statusCode": 200
    }
  }
}
```

> **Note on direct access prevention for Option B:** The `frame-ancestors` CSP header ensures the Commercial NPV frontend can only be embedded in iframes served from `*.vesper.com`. Visiting the URL directly still renders a page, but it shows a blank screen without a valid session token (the app checks for `?session_token=` on mount and shows nothing without it).

### 5.5 Network Topology

```
┌─────────────────────────────────────────────────────────────┐
│  VNET: vnet-vesper-production (10.0.0.0/16)                 │
│                                                             │
│  Subnet: snet-containerApps (10.0.1.0/24)                  │
│  ┌────────────────────────────────────────────────────┐     │
│  │  Container Apps Environment (cae-vesper-production)│     │
│  │                                                    │     │
│  │  ca-vesper-api-gateway     (public ingress)        │     │
│  │  ca-vesper-auth-service    (internal)              │     │
│  │  ca-vesper-core-api        (internal)              │     │
│  │  ca-commercial-npv-backend (internal) ← NEW        │     │
│  │                                                    │     │
│  │  Internal DNS:                                     │     │
│  │  commercial-npv-backend.internal.*.azurecontainerapps.io│
│  └────────────────────────────────────────────────────┘     │
│                                                             │
│  Subnet: snet-data (10.0.2.0/24)                           │
│  Azure SQL (private endpoint)                               │
│  Azure Redis (private endpoint)                             │
│  Azure Key Vault (private endpoint)                         │
└─────────────────────────────────────────────────────────────┘
```

### 5.6 Shared Key Vault — Commercial NPV Secrets

Commercial NPV secrets are added to the **existing Vesper Key Vault** (`kv-vesper-production`) under a dedicated prefix:

```
kv-vesper-production
├── (existing Vesper secrets)
├── commercial-npv-db-url          ← Commercial NPV database connection string
├── commercial-npv-secret-key      ← App secret key (if any)
└── (any other Commercial NPV env vars)
```

Commercial team provides secret values to the platform team; platform team adds them to Key Vault. Commercial team is granted **Key Vault Secrets Reader** role on secrets prefixed `commercial-npv-*`.

---

## 6. Changes Required by Each Team

### 6.1 Changes Required from the Commercial Team

The following is the formal request list to send to the Commercial NPV team. Each item includes the reason and priority.

---

**REQUEST TO COMMERCIAL NPV TEAM**

**From:** Vesper Platform Engineering
**To:** Commercial NPV Tool Team
**Subject:** Integration Requirements for Vesper Platform Onboarding

---

#### CR-01 — Add Dockerfile and docker-compose.yml (Priority: Critical)

The Commercial NPV backend must be containerised to deploy on Azure Container Apps.

Requirements:
- `Dockerfile` for the FastAPI backend using Python 3.12 slim base image
- `docker-compose.yml` for local development (backend + local DB)
- Health check endpoint: `GET /health` returning `{"status": "ok"}`
- Non-root user in the container
- No hardcoded secrets — all config via environment variables

Expected output: `Dockerfile` and `docker-compose.yml` committed to main branch.

---

#### CR-02 — Externalise All Configuration (Priority: Critical)

No `.env` files or hardcoded values in the codebase. All configuration must come from environment variables.

Required environment variables to expose:
```
DATABASE_URL          Connection string for the database
APP_SECRET_KEY        Application secret (if used)
ALLOWED_HOSTS         Comma-separated list of allowed hostnames
CORS_ORIGINS          Allowed CORS origins (will be Vesper's frontend URL)
APP_ENV               dev | staging | production
```

---

#### CR-03 — CORS Configuration (Priority: High)

The Commercial NPV backend must allow CORS from Vesper's frontend domain only:

```python
# In FastAPI main.py
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=os.getenv("CORS_ORIGINS", "").split(","),
    allow_credentials=False,    # No direct cookies — auth handled by proxy
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["X-Vesper-User-Id", "X-Vesper-Tenant-Id",
                   "X-Vesper-User-Email", "X-Vesper-Roles",
                   "Content-Type"],
)
```

Note: CORS headers allow the Vesper gateway to call the backend. In practice, since the backend is internal-only, CORS is a secondary concern — but it must be configured for staging environments.

---

#### CR-04 — Read Vesper Context Headers (Optional but Recommended, Priority: Medium)

The Vesper API Gateway injects the following headers on every proxied request:

```
X-Vesper-User-Id      UUID of the authenticated user
X-Vesper-Tenant-Id    UUID of the user's tenant
X-Vesper-User-Email   Email address of the user
X-Vesper-Roles        Comma-separated list of role names
```

The Commercial backend can use these headers for:
- Audit logging (who ran which calculation)
- Scoping data to the correct tenant
- Any role-based logic within the tool itself

If the Commercial backend currently ignores user identity entirely, this can be added later as an enhancement.

---

#### CR-05 — GitHub Actions Pipeline for Azure Deployment (Priority: High)

A CI/CD pipeline must be added to the Commercial NPV GitHub repository. The platform team will provide:
- Azure Container Registry credentials (as GitHub Actions secrets)
- Azure Container App deployment credentials (OIDC federation — no long-lived secrets)
- Target Container App name and resource group

The pipeline template to use is provided in [§7 of this document](#7-cicd-pipeline--github-repo-to-azure).

---

#### CR-06 — Frontend: API Base URL via Props (Priority: High, Option A only)

If integrating via Option A (micro-frontend bundle), the root App component must accept an `apiBaseUrl` prop:

```tsx
// Commercial NPV: src/App.tsx
interface AppProps {
  apiBaseUrl?: string;   // defaults to '' (relative) for standalone dev
}

export function App({ apiBaseUrl = '' }: AppProps) {
  // Use apiBaseUrl as prefix for all API calls
  // e.g. fetch(`${apiBaseUrl}/calculations`)
}
```

This allows the Vesper frontend to set `apiBaseUrl="/api/v1/commercial/npv"` when embedding the app.

---

#### CR-07 — Frontend: Remove Login/Register Pages (Priority: Medium, Option A only)

Since authentication is handled entirely by Vesper, the Commercial NPV React app should not have a login page. If the current app has login/register screens, they should be:
- Removed from the bundle build
- OR conditionally excluded via a `BUILD_MODE=embedded` environment variable

---

### 6.2 Changes Required from the Vesper Platform Team

| Task | Owner | Notes |
|------|-------|-------|
| Add `commercial_npv` resource to RBAC permission matrix | Platform Backend | SQL migration + Redis cache invalidation |
| Add `commercial_npv:view` and `commercial_npv:export` permissions | Platform Backend | |
| Create `menu_items` entry for "Commercial NPV" | Platform Backend | Per-tenant; controllable by tenant admin |
| Add proxy router to Vesper API Gateway | Platform Backend | See §4.3 |
| Add `COMMERCIAL_NPV_INTERNAL_URL` to Key Vault + Container App env | Platform DevOps | |
| Deploy Commercial NPV Container App (internal ingress) | Platform DevOps | See §5.2 |
| Add Commercial NPV secrets to Key Vault | Platform DevOps | After receiving values from Commercial team |
| Integrate Commercial NPV bundle into Vesper React (Option A) | Platform Frontend | See §4.4 |
| OR set up iframe embed flow (Option B) | Platform Frontend | See §4.4 |
| Add Vesper users / roles the `commercial_npv:view` permission | Platform Admin | Controlled by tenant admin in the UI |

---

## 7. CI/CD Pipeline — GitHub Repo to Azure

The Commercial NPV Tool lives in GitHub. It must deploy to Azure (the same Container Apps environment as Vesper). This is achieved via **GitHub Actions** with **Azure OIDC federated credentials** (no long-lived secrets).

### 7.1 Architecture

```
Commercial NPV GitHub Repo
        │
        ├── Push to main branch
        │
        ▼
GitHub Actions Workflow
        │
        ├── Build & Test
        ├── Docker build
        │
        ▼
Azure Container Registry (shared with Vesper)
acrvesperproduction.azurecr.io/commercial-npv-backend:<sha>
        │
        ▼
Azure Container Apps
ca-commercial-npv-backend (internal ingress)
```

### 7.2 GitHub Actions Workflow — Backend

```yaml
# .github/workflows/deploy.yml  (in Commercial NPV GitHub repo)

name: Deploy Commercial NPV Backend

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  id-token: write    # Required for OIDC auth to Azure
  contents: read

env:
  REGISTRY: acrvesperproduction.azurecr.io
  IMAGE_NAME: commercial-npv-backend
  CONTAINER_APP_NAME: ca-commercial-npv-backend
  RESOURCE_GROUP: rg-vesper-production

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest --tb=short

  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    steps:
      - uses: actions/checkout@v4

      - name: Azure Login (OIDC — no secrets stored)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Log into Azure Container Registry
        run: az acr login --name acrvesperproduction

      - name: Build and push Docker image
        run: |
          IMAGE_TAG=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          docker build -t $IMAGE_TAG .
          docker push $IMAGE_TAG
          # Also tag as latest
          docker tag $IMAGE_TAG ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

      - name: Deploy to Container Apps
        run: |
          az containerapp update \
            --name ${{ env.CONTAINER_APP_NAME }} \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: Health check
        run: |
          # Internal endpoint — check via az containerapp exec or a Vesper API call
          sleep 20
          az containerapp show \
            --name ${{ env.CONTAINER_APP_NAME }} \
            --resource-group ${{ env.RESOURCE_GROUP }} \
            --query "properties.runningStatus" -o tsv
```

### 7.3 Azure OIDC Setup for GitHub Actions

To enable password-less Azure login from GitHub Actions:

```bash
# 1. Create an App Registration for Commercial NPV CI/CD
az ad app create --display-name "sp-commercial-npv-github-actions"

# 2. Create a Service Principal
az ad sp create --id <app-id-from-above>

# 3. Add federated credential for GitHub OIDC
az ad app federated-credential create \
  --id <app-id> \
  --parameters '{
    "name": "commercial-npv-main-branch",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:<github-org>/commercial-npv-tool:ref:refs/heads/main",
    "description": "Commercial NPV main branch deploy",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# 4. Assign roles
az role assignment create \
  --assignee <sp-object-id> \
  --role "AcrPush" \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-vesper-production/providers/Microsoft.ContainerRegistry/registries/acrvesperproduction

az role assignment create \
  --assignee <sp-object-id> \
  --role "Contributor" \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-vesper-production/providers/Microsoft.App/containerApps/ca-commercial-npv-backend
```

GitHub Secrets to add in the Commercial NPV repository:
```
AZURE_CLIENT_ID      = <app-id from step 1>
AZURE_TENANT_ID      = <your Azure tenant ID>
AZURE_SUBSCRIPTION_ID = <your Azure subscription ID>
```

> **Security note:** These credentials only permit deploying to `ca-commercial-npv-backend` and pushing to ACR. They cannot access Vesper's database, Key Vault secrets, or any other resources.

### 7.4 Frontend Bundle Pipeline (Option A)

```yaml
# .github/workflows/build-frontend.yml  (in Commercial NPV GitHub repo)

name: Build Commercial NPV Frontend Bundle

on:
  push:
    branches: [main]
    paths:
      - 'frontend/**'

jobs:
  build-bundle:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install and build
        working-directory: ./frontend
        run: |
          npm ci
          npm run build:lib    # produces dist/commercial-npv-ui.js

      - name: Publish artifact
        uses: actions/upload-artifact@v4
        with:
          name: commercial-npv-frontend-bundle
          path: frontend/dist/commercial-npv-ui.js
          retention-days: 30
```

The Vesper Azure DevOps pipeline then downloads this artifact and includes it in the Vesper Static Web App deployment.

---

## 8. Team Collaboration Protocol

### 8.1 Communication Channels

| Topic | Channel | Owner |
|-------|---------|-------|
| API contract changes (new endpoints, breaking changes) | Shared Confluence page or PR in docs repo | Commercial team raises, Platform team approves |
| Infrastructure requests (secrets, scaling, new env vars) | Platform Engineering Jira board | Commercial team submits ticket |
| Bug affecting the integration (proxy, RBAC) | Platform Engineering Jira board | Whichever team identifies it |
| Bug within the NPV tool itself | Commercial team GitHub Issues | Commercial team |
| New feature requiring RBAC changes | RFC document (see §8.2) | Platform team review required |
| Emergency production fix | Slack #vesper-platform-ops + Jira P1 ticket | Immediate escalation to Platform lead |

### 8.2 Change Request Process

Any change from the Commercial NPV team that affects the integration boundary requires a **Change Request (CR)** document:

```
CR Template (Markdown)
──────────────────────
Title:
Date:
Author:
Priority: [Critical | High | Medium | Low]

## What is changing?
(Describe the change in the Commercial NPV Tool)

## Does this affect the Vesper API Gateway proxy route?
[ ] Yes — new API paths added/removed
[ ] No

## Does this require new RBAC permissions?
[ ] Yes — specify resource keys and actions
[ ] No

## Does this require new secrets in Key Vault?
[ ] Yes — specify secret names (not values)
[ ] No

## Does this require schema changes to Vesper's DB?
[ ] Yes — specify tables/columns
[ ] No

## Deployment dependencies
(Does this backend deploy before or after Vesper deploys? Are they independent?)

## Rollback plan
```

CRs are submitted as a PR to the `hhitcdpdocs` repository (this docs repo) and reviewed by the Platform Engineering lead before implementation.

### 8.3 API Contract Versioning

The Commercial NPV backend exposes its API through the Vesper gateway at `/api/v1/commercial/npv/**`. All paths are passed through as-is. Therefore:

- The Commercial NPV backend's API paths are public API paths (from the user's perspective)
- Breaking changes to the Commercial NPV API must follow Vesper's API versioning standard (Section 9 of the main README)
- A new major version of the Commercial NPV API = `/api/v2/commercial/npv/**` with a corresponding update to the Vesper proxy route
- Both versions must be maintained in parallel for a minimum of **2 sprint cycles** (typically 4 weeks)

### 8.4 Environment Parity

| Environment | Vesper Components | Commercial NPV Components |
|------------|-----------------|--------------------------|
| Local dev | docker-compose (Vesper stack) | docker-compose (Commercial NPV, standalone) |
| Dev | Fully deployed in Azure | `ca-commercial-npv-backend-dev` (internal) |
| Staging | Fully deployed | `ca-commercial-npv-backend-staging` (internal) |
| Production | Fully deployed | `ca-commercial-npv-backend-prod` (internal) |

The Commercial team runs their app **standalone** (no Vesper dependency) in local dev. The integration is tested only from Dev environment onwards.

### 8.5 Secrets Handover Process

```
Commercial Team → Platform Team Secrets Handover
─────────────────────────────────────────────────
1. Commercial team identifies required secrets (names only, no values)
2. Commercial team creates a Jira ticket: "Secret handover for Commercial NPV [env]"
3. Platform team schedules a secure handover call (screenshare)
4. During call: Commercial team pastes secret values directly into
   Azure Key Vault via the Azure Portal (platform team grants temporary access)
5. Platform team verifies secrets are stored and removes Commercial team access
6. Platform team adds Key Vault references to the Container App environment

NEVER send secrets via Slack, email, or Jira comments.
```

### 8.6 Repository Access Control

| Repo | Where Hosted | Platform Team Access | Commercial Team Access |
|------|-------------|---------------------|----------------------|
| Vesper Backend | Azure DevOps | Full | Read-only (for API reference) |
| Vesper Frontend | Azure DevOps | Full | Read-only |
| Commercial NPV Tool | GitHub | Read-only | Full |
| This docs repo (hhitcdpdocs) | Azure DevOps (or GitHub) | Full | PR submit only |

---

## 9. Rollout Checklist

Use this checklist to track the integration rollout. Each item has a designated owner.

### Phase 1 — Foundation (Platform Team)

- [ ] Add `commercial_npv` resource to permission matrix SQL migration
- [ ] Add `commercial_npv:view` and `commercial_npv:export` permissions to seed data
- [ ] Add proxy router `/api/v1/commercial/npv/**` to Vesper API Gateway (unauthenticated initially, behind a feature flag)
- [ ] Add `COMMERCIAL_NPV_INTERNAL_URL` placeholder to Key Vault
- [ ] Create `ca-commercial-npv-backend` Container App in Azure (internal ingress, placeholder image)
- [ ] Create Azure OIDC credentials for Commercial NPV GitHub Actions
- [ ] Share GitHub Actions secrets with Commercial team

### Phase 2 — Commercial Team Deliverables

- [ ] CR-01: Dockerfile and docker-compose.yml committed
- [ ] CR-02: All config externalised to environment variables
- [ ] CR-03: CORS configured for Vesper origins
- [ ] CR-05: GitHub Actions pipeline added and first image pushed to ACR
- [ ] CR-06/07: Frontend adapted for embedding (Option A or B)
- [ ] Commercial team confirms `GET /health` returns 200

### Phase 3 — Integration & Testing (Dev Environment)

- [ ] Commercial NPV backend deployed to `ca-commercial-npv-backend-dev`
- [ ] Update `COMMERCIAL_NPV_INTERNAL_URL` in dev Key Vault with actual internal URL
- [ ] Enable proxy feature flag in Vesper API Gateway (dev)
- [ ] Add `menu_items` entry for "Commercial NPV" in dev tenant
- [ ] Assign `commercial_npv:view` to test user role in dev
- [ ] End-to-end test: log in as test user → menu shows "Commercial NPV" → tool loads → API calls succeed
- [ ] Confirm direct access to Commercial backend is blocked (no public endpoint)
- [ ] Confirm unauthenticated user gets 401 from Vesper gateway

### Phase 4 — Staging Validation

- [ ] Repeat Phase 2/3 in staging environment
- [ ] Security review: penetration test of the proxy endpoint
- [ ] Load test: Commercial NPV backend under Vesper gateway throughput

### Phase 5 — Production Go-Live

- [ ] Production deployment approved by CTO
- [ ] Add `menu_items` entry in production tenants (start with internal users only)
- [ ] Monitor App Insights for errors and latency after go-live
- [ ] Commercial team on-call for first 48 hours post-launch

---

## Appendix A — Data Flow Diagram

```
User Browser
    │
    │  HTTPS (public internet)
    │
    ▼
Azure Front Door WAF
    │  Terminates SSL
    │  Applies OWASP WAF rules
    │
    ├──────────────────────────────────────────────────────────┐
    │ Static content                                           │ API requests
    ▼                                                          ▼
Azure Static Web App                              Container App: vesper-api-gateway
(Vesper React SPA)                                (public ingress — port 443)
    │                                                          │
    │ Lazy load CommercialNPV bundle                          │ /api/v1/commercial/npv/**
    │ (from /assets/commercial-npv-ui.js)                     │
    │                                                          │ JWT validated ✓
    │ All API calls → /api/v1/**                              │ RBAC checked ✓
    │ (same-origin, goes to AFD → Gateway)                    │ Headers injected
    │                                                          │
    ▼                                                          │ Internal VNET HTTP
                                               ┌──────────────▼──────────────────┐
                                               │ Container App:                  │
                                               │ ca-commercial-npv-backend       │
                                               │ (INTERNAL ingress — no public   │
                                               │  endpoint)                      │
                                               │                                 │
                                               │ Reads X-Vesper-* headers        │
                                               │ Returns JSON response           │
                                               └─────────────────────────────────┘
```

## Appendix B — Key Vault Secret Naming Convention

All Commercial NPV secrets in Key Vault follow this naming pattern:

```
commercial-npv-{environment}-{secret-name}

Examples:
  commercial-npv-prod-db-url
  commercial-npv-prod-secret-key
  commercial-npv-staging-db-url
```

## Appendix C — Contacts

| Role | Responsibility | Contact Method |
|------|---------------|----------------|
| Platform Engineering Lead | Integration architecture decisions, CR approval | Jira + Slack |
| Platform DevOps | Azure infrastructure, Key Vault, ACR, Container Apps | Jira ticket |
| Commercial NPV Team Lead | API contract changes, Commercial tool issues | GitHub Issues or Jira |
| Security Reviewer | Penetration testing, threat model review | Scheduled review via Jira |

---

*This document governs the integration between the Vesper DataHub Platform and the Commercial NPV Tool. All integration changes must follow the Change Request process in §8.2. For questions, raise a Jira ticket against the Platform Engineering board.*
