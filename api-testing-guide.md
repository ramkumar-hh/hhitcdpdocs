# Vesper Analytics API — Testing Guide

**Audience:** Backend developers, frontend developers, QA engineers
**Purpose:** Understand how to call and test every API endpoint individually, following the platform standards.

---

## Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [Tools Setup](#2-tools-setup)
3. [Mandatory Request Headers](#3-mandatory-request-headers)
4. [Standard Response Envelope](#4-standard-response-envelope)
5. [Standard Error Envelope](#5-standard-error-envelope)
6. [Step 1 — Health Check (Smoke Test)](#6-step-1--health-check-smoke-test)
7. [Step 2 — Authentication Flow](#7-step-2--authentication-flow)
8. [Step 3 — Get Your Permissions](#8-step-3--get-your-permissions)
9. [Step 4 — Navigation Menu](#9-step-4--navigation-menu)
10. [Step 5 — User Management APIs](#10-step-5--user-management-apis)
11. [Step 6 — Role Management APIs](#11-step-6--role-management-apis)
12. [Step 7 — Group Management APIs](#12-step-7--group-management-apis)
13. [Step 8 — Permission APIs](#13-step-8--permission-apis)
14. [Step 9 — Audit Log APIs](#14-step-9--audit-log-apis)
15. [Step 10 — Tenant APIs (Super-Admin)](#15-step-10--tenant-apis-super-admin)
16. [Token Lifecycle & Refresh Strategy](#16-token-lifecycle--refresh-strategy)
17. [Rate Limiting](#17-rate-limiting)
18. [RBAC — Permission Matrix Quick Reference](#18-rbac--permission-matrix-quick-reference)
19. [Common Mistakes & How to Fix Them](#19-common-mistakes--how-to-fix-them)
20. [Frontend Developer Checklist](#20-frontend-developer-checklist)
21. [Entra ID — Azure Prerequisites & App Registration](#21-entra-id--azure-prerequisites--app-registration)
22. [Entra ID — Platform-Level Environment Configuration](#22-entra-id--platform-level-environment-configuration)
23. [Entra ID — Per-Tenant Connection Setup (API)](#23-entra-id--per-tenant-connection-setup-api)
24. [Entra ID — Group Mappings](#24-entra-id--group-mappings)
25. [Entra ID — Login Flow & Frontend Integration](#25-entra-id--login-flow--frontend-integration)
26. [Entra ID — Sync Operations](#26-entra-id--sync-operations)
27. [Entra ID — Testing Strategy](#27-entra-id--testing-strategy)
28. [Entra ID — Troubleshooting](#28-entra-id--troubleshooting)

---

## 1. Overview & Architecture

Every request passes through a **6-layer middleware stack** before reaching your endpoint handler. Understanding this is critical to diagnosing failures.

```
Request In
    │
    ▼
[1] CorrelationIdMiddleware   → assigns X-Correlation-ID (tracing)
    │
    ▼
[2] CORSMiddleware            → handles preflight, validates Origin
    │
    ▼
[3] RateLimiterMiddleware     → enforces per-IP or per-user rate limits
    │
    ▼
[4] TenantContextMiddleware   → resolves tenant from X-Tenant-ID header
    │
    ▼
[5] AuditMiddleware           → logs all write operations (POST/PUT/PATCH/DELETE)
    │
    ▼
[6] ErrorHandlerMiddleware    → converts domain exceptions → structured JSON
    │
    ▼
  JWT Auth Dependency         → validates Bearer token
    │
    ▼
  Permission Dependency       → checks RBAC for the specific resource + action
    │
    ▼
  Route Handler               → your business logic
```

**Key takeaway for debugging:**
- `401 Unauthorized` → JWT problem (expired, missing, wrong audience)
- `403 Forbidden` → Permission denied (your role doesn't have the required resource:action)
- `400 Bad Request` with `error_code: TENANT_NOT_FOUND` → Missing or wrong `X-Tenant-ID` header
- `429 Too Many Requests` → Rate limit hit
- `502/503` without JSON body → Infrastructure/gateway issue, not app-level

---

## 2. Tools Setup

### Option A — Swagger UI (Recommended for exploration)

When running locally with `APP_ENV=development`:

```
http://localhost:8000/docs
```

Swagger UI is **disabled in production** (`APP_ENV=production`). Use curl or Postman for production testing.

### Option B — curl

All examples in this guide use `curl`. Set these shell variables once per session:

```bash
# Base URL — change for each environment
export BASE_URL="http://localhost:8000"

# Tenant slug — get this from your DBA or platform admin
export TENANT_SLUG="acme"

# After login, you'll store the token here
export TOKEN=""
```

### Option C — Postman / Insomnia

1. Create an **Environment** with variables: `base_url`, `tenant_id`, `access_token`
2. Add a **Pre-request Script** on the Collection level to auto-refresh tokens when expired
3. Import the OpenAPI spec from `GET /openapi.json` (dev only)

---

## 3. Mandatory Request Headers

Every single API call (except `/health` endpoints) requires these headers:

| Header | Required | Value | Notes |
|--------|----------|-------|-------|
| `X-Tenant-ID` | **Yes** | Tenant UUID | Get from `/api/v1/tenants/resolve?slug=<slug>` |
| `Authorization` | Yes (for protected routes) | `Bearer <access_token>` | JWT from login |
| `Content-Type` | Yes (for POST/PUT/PATCH) | `application/json` | Always send JSON body |
| `X-Correlation-ID` | Optional | UUID v4 | If you send it, it is echoed back; useful for tracing |

**Frontend developers:** Store `X-Tenant-ID` once at app bootstrap and attach it to every Axios/fetch request via an interceptor. Never hardcode it per-call.

```javascript
// Axios interceptor example
axios.defaults.headers.common['X-Tenant-ID'] = tenantId;
axios.defaults.headers.common['Authorization'] = `Bearer ${accessToken}`;
```

---

## 4. Standard Response Envelope

Every successful response is wrapped in this envelope:

```json
{
  "success": true,
  "data": { ... },          // actual payload (object or array)
  "message": "optional",
  "correlation_id": "uuid"  // echoed from X-Correlation-ID header
}
```

For paginated list responses:

```json
{
  "success": true,
  "data": {
    "items": [ ... ],
    "total": 100,
    "page": 1,
    "page_size": 20,
    "pages": 5
  },
  "correlation_id": "uuid"
}
```

**Frontend rule:** Always read from `response.data.data`, not `response.data` directly.

---

## 5. Standard Error Envelope

All errors return:

```json
{
  "success": false,
  "error": {
    "code": "PERMISSION_DENIED",
    "message": "You do not have permission to perform this action",
    "details": {}           // optional, may include field-level validation errors
  },
  "correlation_id": "uuid"
}
```

Common `error.code` values:

| Code | HTTP Status | Meaning |
|------|-------------|---------|
| `TENANT_NOT_FOUND` | 400 | X-Tenant-ID header missing or invalid |
| `AUTHENTICATION_FAILED` | 401 | Wrong credentials |
| `TOKEN_EXPIRED` | 401 | Access token expired — refresh it |
| `TOKEN_INVALID` | 401 | Malformed or tampered JWT |
| `PERMISSION_DENIED` | 403 | Insufficient RBAC permissions |
| `NOT_FOUND` | 404 | Resource does not exist |
| `CONFLICT` | 409 | Duplicate email, slug, etc. |
| `VALIDATION_ERROR` | 422 | Request body failed schema validation |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `INTERNAL_SERVER_ERROR` | 500 | Unexpected server-side failure |

---

## 6. Step 1 — Health Check (Smoke Test)

Always start here. These endpoints have **no auth and no tenant header required**.

```bash
# Full health check (checks DB + Redis connectivity)
curl -s "$BASE_URL/api/v1/health" | python3 -m json.tool

# Readiness probe (K8s: is the app ready to serve?)
curl -s "$BASE_URL/api/v1/health/ready"

# Liveness probe (K8s: is the process alive?)
curl -s "$BASE_URL/api/v1/health/live"
```

Expected response for healthy instance:

```json
{
  "success": true,
  "data": {
    "status": "healthy",
    "database": "connected",
    "redis": "connected",
    "version": "1.0.0"
  }
}
```

If `database` or `redis` shows `disconnected`, stop here — all other tests will fail.

---

## 7. Step 2 — Authentication Flow

### 2a. Resolve Tenant ID from Slug

Before logging in, you need the tenant's UUID. The slug is human-readable (e.g., `acme`).

```bash
curl -s "$BASE_URL/api/v1/tenants/resolve?slug=$TENANT_SLUG" | python3 -m json.tool
```

Expected response:

```json
{
  "success": true,
  "data": {
    "id": "a1b2c3d4-...",
    "name": "ACME Corp",
    "slug": "acme",
    "status": "active"
  }
}
```

Save the tenant ID:

```bash
export TENANT_ID="a1b2c3d4-..."
```

### 2b. Login (Local Authentication)

```bash
curl -s -c cookies.txt -X POST "$BASE_URL/api/v1/auth/login" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -d '{
    "email": "admin@acme.com",
    "password": "YourPassword123!",
    "tenant_slug": "acme"
  }' | python3 -m json.tool
```

**Important:** The `-c cookies.txt` flag saves the HttpOnly refresh token cookie. You need this for token refresh.

Expected response:

```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJSUzI1NiJ9...",
    "token_type": "bearer",
    "expires_in": 900,
    "user": {
      "id": "...",
      "email": "admin@acme.com",
      "display_name": "Admin User",
      "is_tenant_admin": true
    }
  }
}
```

Save the access token:

```bash
export TOKEN=$(curl -s -c cookies.txt -X POST "$BASE_URL/api/v1/auth/login" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -d '{"email":"admin@acme.com","password":"YourPassword123!","tenant_slug":"acme"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['access_token'])")

echo "Token saved: ${TOKEN:0:40}..."
```

### 2c. MFA Verification (if enabled)

If the login response contains `"mfa_required": true` instead of `access_token`:

```bash
# You'll receive a partial token for MFA challenge
export MFA_TOKEN="<partial_token_from_login_response>"

curl -s -X POST "$BASE_URL/api/v1/auth/mfa/verify" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $MFA_TOKEN" \
  -d '{"code": "123456"}' | python3 -m json.tool
```

The response will now contain the full `access_token`.

### 2d. Get Current User Profile

```bash
curl -s "$BASE_URL/api/v1/auth/me" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

Expected response includes user profile + full permission matrix.

### 2e. Logout

```bash
curl -s -b cookies.txt -X POST "$BASE_URL/api/v1/auth/logout" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

The `-b cookies.txt` flag sends the refresh cookie so the server can revoke it.

---

## 8. Step 3 — Get Your Permissions

Always check what your current user can do before testing protected endpoints.

```bash
# Full permission matrix
curl -s "$BASE_URL/api/v1/permissions/matrix" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

# Check a specific permission
curl -s -X POST "$BASE_URL/api/v1/permissions/check" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"resource": "users", "action": "create"}' | python3 -m json.tool
```

The matrix response shows every `resource:action` pair and whether it is allowed or denied. If you get `403` on a protected endpoint, come back here to verify the permission exists for your role.

---

## 9. Step 4 — Navigation Menu

Returns the RBAC-filtered navigation tree for the current user. Frontend should call this once after login to build the sidebar.

```bash
curl -s "$BASE_URL/api/v1/navigation/menu" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

Expected response:

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "...",
        "label": "Analytics",
        "item_type": "group",
        "icon": "chart-bar",
        "children": [
          {
            "id": "...",
            "label": "Sales Dashboard",
            "item_type": "powerbi",
            "route": "/reports/sales"
          }
        ]
      }
    ]
  }
}
```

Items with `item_type: "powerbi"` require a separate embed token call (see Power BI section in main README).

---

## 10. Step 5 — User Management APIs

Required permission: `users:read` (list/get), `users:create`, `users:update`, `users:delete`

### List Users

```bash
curl -s "$BASE_URL/api/v1/users?page=1&page_size=20" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

Query params:
- `page` (default: 1)
- `page_size` (default: 20, max: 100)
- `search` — search by email or name
- `status` — `active`, `inactive`, `pending`

### Get a Specific User

```bash
export USER_ID="<user-uuid>"

curl -s "$BASE_URL/api/v1/users/$USER_ID" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

### Get Own Profile

```bash
curl -s "$BASE_URL/api/v1/users/me" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

### Create a User

```bash
curl -s -X POST "$BASE_URL/api/v1/users" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "email": "newuser@acme.com",
    "password": "SecurePass123!",
    "first_name": "Jane",
    "last_name": "Doe",
    "status": "active"
  }' | python3 -m json.tool
```

### Update a User

```bash
curl -s -X PUT "$BASE_URL/api/v1/users/$USER_ID" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "first_name": "Jane",
    "last_name": "Smith",
    "status": "active"
  }' | python3 -m json.tool
```

Only include the fields you want to change. Omitted fields are not touched.

### Assign Roles to a User

```bash
curl -s -X POST "$BASE_URL/api/v1/users/$USER_ID/roles" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "role_ids": ["role-uuid-1", "role-uuid-2"]
  }' | python3 -m json.tool
```

### Add User to Groups

```bash
curl -s -X POST "$BASE_URL/api/v1/users/$USER_ID/groups" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "group_ids": ["group-uuid-1"]
  }' | python3 -m json.tool
```

### Delete a User

```bash
curl -s -X DELETE "$BASE_URL/api/v1/users/$USER_ID" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

---

## 11. Step 6 — Role Management APIs

Required permission: `roles:read`, `roles:create`, `roles:update`, `roles:delete`

### List Roles

```bash
curl -s "$BASE_URL/api/v1/roles?page=1&page_size=20" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

### Create a Role

```bash
curl -s -X POST "$BASE_URL/api/v1/roles" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "Report Viewer",
    "description": "Can view reports and dashboards",
    "status": "active"
  }' | python3 -m json.tool
```

### Get Role with Permissions

```bash
export ROLE_ID="<role-uuid>"

curl -s "$BASE_URL/api/v1/roles/$ROLE_ID" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

### Grant a Permission to a Role

First get the permission ID from the permissions list, then:

```bash
export PERMISSION_ID="<permission-uuid>"

curl -s -X POST "$BASE_URL/api/v1/roles/$ROLE_ID/permissions" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "permission_id": "'$PERMISSION_ID'"
  }' | python3 -m json.tool
```

### Revoke a Permission from a Role

```bash
curl -s -X DELETE "$BASE_URL/api/v1/roles/$ROLE_ID/permissions/$PERMISSION_ID" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

---

## 12. Step 7 — Group Management APIs

Required permission: `groups:read`, `groups:create`, `groups:update`, `groups:delete`

### List Groups

```bash
curl -s "$BASE_URL/api/v1/groups" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

### Create a Group

```bash
curl -s -X POST "$BASE_URL/api/v1/groups" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "Sales Team",
    "description": "Sales department users"
  }' | python3 -m json.tool
```

### Add Member to Group

```bash
export GROUP_ID="<group-uuid>"

curl -s -X POST "$BASE_URL/api/v1/groups/$GROUP_ID/members" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "user_id": "'$USER_ID'"
  }' | python3 -m json.tool
```

---

## 13. Step 8 — Permission APIs

These are catalogue APIs — they list what permissions exist in the system, not what a user has.

### List All Resources

```bash
curl -s "$BASE_URL/api/v1/permissions/resources" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

### List All Actions

```bash
curl -s "$BASE_URL/api/v1/permissions/actions" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

### List Permission Combinations

```bash
# All permissions
curl -s "$BASE_URL/api/v1/permissions" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

# Filter by resource
curl -s "$BASE_URL/api/v1/permissions?resource_id=<uuid>" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

---

## 14. Step 9 — Audit Log APIs

Required permission: `audit:read` (typically tenant-admin only)

### List Audit Logs

```bash
curl -s "$BASE_URL/api/v1/audit/logs?page=1&page_size=50" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

Useful query params:
- `user_id` — filter by who performed the action
- `resource_type` — e.g., `users`, `roles`
- `action` — e.g., `create`, `update`, `delete`
- `from_date`, `to_date` — ISO 8601 timestamps

### Get Single Audit Log Entry

```bash
export LOG_ID="<log-uuid>"

curl -s "$BASE_URL/api/v1/audit/logs/$LOG_ID" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

The `details` field contains `before` and `after` JSON snapshots of the changed resource.

---

## 15. Step 10 — Tenant APIs (Super-Admin)

These endpoints require the caller to be a **super-admin** (`is_super_admin: true`). They are not available to regular tenant admins.

### List All Tenants

```bash
curl -s "$BASE_URL/api/v1/tenants" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $SUPER_ADMIN_TOKEN" | python3 -m json.tool
```

### Create a New Tenant

```bash
curl -s -X POST "$BASE_URL/api/v1/tenants" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $SUPER_ADMIN_TOKEN" \
  -d '{
    "name": "New Corp",
    "slug": "newcorp",
    "plan": "enterprise",
    "status": "active"
  }' | python3 -m json.tool
```

Slug must be unique, lowercase, alphanumeric with hyphens allowed.

### Resolve Tenant by Slug (Public — no auth needed)

```bash
curl -s "$BASE_URL/api/v1/tenants/resolve?slug=newcorp" | python3 -m json.tool
```

---

## 16. Token Lifecycle & Refresh Strategy

```
Login
  │
  ▼
Access Token (15 min, JWT)  +  Refresh Token (7 days, HttpOnly cookie)
  │
  │ Use access token for all API calls
  │
  ▼ (when 401 TOKEN_EXPIRED received)
POST /api/v1/auth/refresh  ← sends refresh cookie automatically
  │
  ▼
New Access Token + New Refresh Cookie (rotation)
  │
  ▼
Continue with new token
  │
  ▼ (7 days elapsed or explicit logout)
Session ends
```

### Refresh Token Call

```bash
curl -s -b cookies.txt -c cookies.txt -X POST "$BASE_URL/api/v1/auth/refresh" \
  -H "X-Tenant-ID: $TENANT_ID" | python3 -m json.tool
```

The `-b cookies.txt` sends the saved cookie, `-c cookies.txt` saves the new rotated cookie.

### Frontend Implementation Pattern

```javascript
// Intercept 401 responses and refresh
axios.interceptors.response.use(
  response => response,
  async error => {
    const original = error.config;
    if (
      error.response?.status === 401 &&
      error.response?.data?.error?.code === 'TOKEN_EXPIRED' &&
      !original._retry
    ) {
      original._retry = true;
      const res = await axios.post('/api/v1/auth/refresh');
      const newToken = res.data.data.access_token;
      // Update stored token
      setAccessToken(newToken);
      original.headers['Authorization'] = `Bearer ${newToken}`;
      return axios(original);
    }
    return Promise.reject(error);
  }
);
```

**Rules:**
- Never store the access token in `localStorage` — use memory (React state / Zustand / Redux)
- The refresh token is HttpOnly cookie — the browser sends it automatically, your JS cannot read it
- On `TOKEN_INVALID` (not `TOKEN_EXPIRED`), do NOT refresh — redirect to login immediately
- On app startup, call `/api/v1/auth/refresh` to silently restore session before showing the UI

---

## 17. Rate Limiting

The API enforces rate limits per IP address (or per user when authenticated).

| Endpoint Group | Limit | Window |
|----------------|-------|--------|
| `/auth/login` | 10 requests | 60 seconds |
| All other endpoints | Configurable (default 100) | 60 seconds |

When exceeded, you receive:

```json
HTTP 429 Too Many Requests

{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again later.",
    "details": {
      "retry_after": 45
    }
  }
}
```

Check the `retry_after` seconds value and wait before retrying.

---

## 18. RBAC — Permission Matrix Quick Reference

The system enforces fine-grained Resource + Action permissions.

| Resource | Actions Available |
|----------|------------------|
| `users` | `create`, `read`, `update`, `delete` |
| `roles` | `create`, `read`, `update`, `delete` |
| `groups` | `create`, `read`, `update`, `delete` |
| `tenants` | `create`, `read`, `update`, `delete` |
| `reports` | `read`, `execute`, `export` |
| `dashboards` | `read`, `execute` |
| `audit` | `read` |
| `navigation` | `read` |

### How Resolution Works

1. User has direct role assignments → collect all permissions from those roles
2. User is in groups → groups have role assignments → collect all permissions
3. Merge all permissions: **explicit deny always wins** over allow
4. Result is cached in Redis for 5 minutes per user/tenant

### If You Get a 403

```bash
# Check your permission matrix
curl -s "$BASE_URL/api/v1/permissions/matrix" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

# Check a specific permission
curl -s -X POST "$BASE_URL/api/v1/permissions/check" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"resource": "users", "action": "create"}' | python3 -m json.tool
```

If the permission is missing, a tenant admin must assign the appropriate role to your user.

---

## 19. Common Mistakes & How to Fix Them

### "TENANT_NOT_FOUND" on every request

**Cause:** Missing or wrong `X-Tenant-ID` header
**Fix:** Call `GET /api/v1/tenants/resolve?slug=<slug>` to get the correct UUID, then include it as `X-Tenant-ID: <uuid>` in every request.

### "TOKEN_EXPIRED" immediately after login

**Cause:** Your system clock is out of sync with the server
**Fix:** Sync your machine clock with NTP. The JWT has a 15-minute window.

### "TOKEN_INVALID" even with a fresh token

**Cause:** Sending the token in the wrong format
**Fix:** The header must be exactly `Authorization: Bearer <token>` — note the capital B and single space. Do not include quotes around the token.

### 403 on a route that should work

**Cause 1:** Your role doesn't have the permission → assign the correct role via tenant admin
**Cause 2:** Permission cache is stale → Redis caches permissions for 5 minutes; wait or ask backend to invalidate
**Cause 3:** You're calling a super-admin route with a regular admin token

### 422 Validation Error

The server returns field-level details:

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "details": {
      "fields": [
        {"field": "email", "message": "value is not a valid email address"},
        {"field": "password", "message": "must be at least 8 characters"}
      ]
    }
  }
}
```

Check `error.details.fields` to identify exactly which field failed.

### Refresh cookie not being sent

**Cause:** Browser same-site/secure cookie restrictions
**Fix:**
- In development (HTTP), ensure the login request was made from the same origin
- In production, the cookie requires HTTPS and `Secure` flag
- Never try to manually attach the refresh token — it is HttpOnly and must be sent by the browser automatically

### CORS preflight failing

**Cause:** Your frontend's `Origin` header is not in the allowed origins list
**Fix:** The backend `CORS_ORIGINS` env var must include your frontend's exact origin (including port). Contact DevOps to add it.

---

## 20. Frontend Developer Checklist

Use this checklist when integrating with the backend:

### Bootstrap Sequence

- [ ] On app load: call `POST /api/v1/auth/refresh` to silently restore session
- [ ] On success: store access token in memory, call `GET /api/v1/auth/me` for user profile
- [ ] On failure (401): redirect to login page
- [ ] Call `GET /api/v1/navigation/menu` to build the sidebar
- [ ] Store `X-Tenant-ID` and attach to all requests via interceptor

### Axios/fetch Setup

- [ ] Set `withCredentials: true` on your HTTP client (required for cookies)
- [ ] Set default header `X-Tenant-ID` from tenant context
- [ ] Set default header `Authorization: Bearer <token>` after login
- [ ] Add response interceptor to handle `TOKEN_EXPIRED` → auto-refresh → retry

### Token Management

- [ ] Store access token in memory only (React state, Zustand, etc.)
- [ ] Never store access token in localStorage or sessionStorage
- [ ] Never manually read or write the refresh cookie
- [ ] On `TOKEN_INVALID`: clear state, redirect to login (do NOT retry)
- [ ] On `TOKEN_EXPIRED`: refresh token, then retry original request once

### Error Handling

- [ ] Always read `response.data.error.code` for programmatic error handling
- [ ] Show `response.data.error.message` to the user (it is human-readable)
- [ ] On `403`: show "You don't have permission" message — do NOT retry
- [ ] On `429`: show "Too many requests, try again in X seconds" using `retry_after`
- [ ] On `5xx`: show generic error, log `correlation_id` for backend debugging

### Response Reading

- [ ] Always read payload from `response.data.data` (not `response.data`)
- [ ] For paginated lists: read `response.data.data.items`, use `total`/`pages` for pagination UI
- [ ] Log or display `correlation_id` in error UIs so DevOps can trace the request

### Power BI Reports

- [ ] Check `item_type === 'powerbi'` in navigation menu items
- [ ] Call `GET /api/v1/powerbi/reports/{menu_item_id}/embed-token` before rendering
- [ ] Set up token refresh timer using `expires_in` from the embed token response
- [ ] Listen for `report.token_refresh` push events on the WebSocket notification channel

### WebSocket (Notifications)

- [ ] Connect to `wss://<host>/ws/v1/notifications?token=<access_token>`
- [ ] Send `{"type":"ping"}` every 30 seconds to keep the connection alive
- [ ] Handle reconnection on disconnect using exponential backoff
- [ ] Pass a fresh access token on reconnect (the token in the URL must be valid)

---

## 21. Entra ID — Azure Prerequisites & App Registration

Before the backend can handle Entra logins for any tenant, two Azure AD registrations must exist: one owned by the **platform** (Vesper), and one owned by **each customer tenant**.

### Architecture Overview

```
Customer's Azure AD Tenant
    │
    │  (OIDC Authorization Code Flow)
    ▼
Platform App Registration  ←──  Single registration in Vesper's Azure AD
    │                            client_id + client_secret (in Key Vault)
    ▼
Vesper Backend
    │
    ▼
Issues platform JWT (RS256) to the browser
```

There are **two distinct layers** of Azure AD configuration:

| Layer | Who owns it | Stored where | Purpose |
|-------|-------------|--------------|---------|
| **Platform-level** | Vesper DevOps | `.env` + Key Vault | The app registration used to perform OIDC against any customer tenant |
| **Per-tenant** | Customer admin (set via API) | `entra_connections` DB table + Key Vault | Customer's own Azure AD tenant ID and the OAuth credentials Vesper uses to call their tenant |

### Step-by-Step: Platform App Registration (One Time — DevOps Only)

1. Open [Azure Portal](https://portal.azure.com) → **Azure Active Directory** → **App registrations** → **New registration**
2. Name: `Vesper Analytics Platform`
3. Supported account types: **Accounts in any organizational directory (Multi-tenant)**
4. Redirect URI: Web — `https://api.vesper-analytics.com/api/v1/auth/entra/callback`
   - For local dev add: `http://localhost:8000/api/v1/auth/entra/callback`
5. Click **Register**
6. Note the values:
   - **Application (client) ID** → `ENTRA_PLATFORM_CLIENT_ID`
   - **Directory (tenant) ID** → `ENTRA_PLATFORM_TENANT_ID`
7. Go to **Certificates & secrets** → **New client secret** → copy the value
8. Store the secret in **Azure Key Vault** under a key name like `vesper-entra-client-secret`
   - Set `ENTRA_PLATFORM_CLIENT_SECRET_REF=vesper-entra-client-secret` in `.env`
   - **Never put the raw secret in `.env` or source control**
9. Go to **API permissions** → Add:
   - Microsoft Graph → Delegated: `openid`, `profile`, `email`
   - Microsoft Graph → Application: `GroupMember.Read.All`, `User.Read.All` (for group sync)
10. Click **Grant admin consent**

### Step-by-Step: Per-Customer App Registration (One Per Customer Tenant)

Each customer must perform this in **their own** Azure AD tenant:

1. Azure Portal → App registrations → New registration
2. Name: `Vesper Analytics` (or any name they choose)
3. Supported account types: **Single tenant** (accounts in this directory only)
4. Redirect URI: Web — `https://api.vesper-analytics.com/api/v1/auth/entra/callback`
5. Register → note **Application (client) ID** and **Directory (tenant) ID**
6. Create a client secret under Certificates & secrets
7. API permissions → Microsoft Graph → Delegated: `openid profile email`; Application: `GroupMember.Read.All` (if group sync is needed) → Grant admin consent
8. Provide Vesper with:
   - Their **Directory (tenant) ID** → `entra_tenant_id` in connection API call
   - Their **Application (client) ID** → `client_id`
   - Their **client secret** → `client_secret` (Vesper stores this in Key Vault, not the DB)

---

## 22. Entra ID — Platform-Level Environment Configuration

These variables configure the **platform's own** Azure AD identity. Set them once in your deployment environment (`.env` locally, Key Vault / App Service config in Azure).

```bash
# The platform's own Azure AD tenant (Vesper's Azure subscription)
ENTRA_PLATFORM_TENANT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# The client ID of the platform App Registration
ENTRA_PLATFORM_CLIENT_ID=yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy

# Name of the Key Vault secret that holds the platform client secret
# Never put the raw secret here
ENTRA_PLATFORM_CLIENT_SECRET_REF=vesper-entra-client-secret

# Base URL used to construct the callback redirect URI
# Must exactly match the Redirect URI registered in Azure
ENTRA_CALLBACK_BASE_URL=https://api.vesper-analytics.com

# Key Vault URL (used by Azure SDK to retrieve secrets)
KEY_VAULT_URL=https://your-keyvault.vault.azure.net/

# Managed Identity client ID (used by the app to authenticate to Key Vault)
AZURE_CLIENT_ID=zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz
```

### Verifying Platform Config is Correct

```bash
# Smoke test: if health returns healthy, Key Vault access is working
curl -s "$BASE_URL/api/v1/health" | python3 -m json.tool

# Try to generate an authorize URL for a known tenant slug
# If ENTRA_PLATFORM_CLIENT_ID or KEY_VAULT_URL are wrong, this will return 500
curl -s "$BASE_URL/api/v1/auth/entra/authorize?tenant_slug=$TENANT_SLUG" \
  | python3 -m json.tool
```

Expected success response from the authorize endpoint:

```json
{
  "success": true,
  "data": {
    "authorization_url": "https://login.microsoftonline.com/<customer_tenant_id>/oauth2/v2.0/authorize?...",
    "state": "a1b2c3d4e5f6..."
  }
}
```

---

## 23. Entra ID — Per-Tenant Connection Setup (API)

These endpoints are called by a **tenant admin** to configure Entra SSO for their tenant. No super-admin token is needed — just a tenant admin JWT.

All endpoints require:
- `X-Tenant-ID: <tenant_uuid>`
- `Authorization: Bearer <tenant_admin_token>`

### Get Current Entra Connection

```bash
curl -s "$BASE_URL/api/v1/entra/connection" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

Expected response when configured:

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "tenant_id": "uuid",
    "entra_tenant_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "client_id": "****yyyyyyyy",
    "scopes": "openid profile email",
    "auto_provision": true,
    "auto_sync_groups": true,
    "sync_interval_min": 30,
    "last_sync_at": null,
    "status": "active"
  }
}
```

Note: `client_id` is **masked** in the response (only last characters shown). The raw value is never returned after creation.

If no connection exists yet, you receive a `400` with `error_code: ENTRA_CONFIG_ERROR`.

### Create a New Entra Connection

Call this once per tenant when onboarding Entra SSO. Use the customer's app registration credentials.

```bash
curl -s -X POST "$BASE_URL/api/v1/entra/connection" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "entra_tenant_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "client_id": "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy",
    "client_secret": "the-actual-secret-from-customer",
    "scopes": "openid profile email",
    "whitelisted_groups": [],
    "auto_provision": true,
    "auto_sync_groups": true,
    "sync_interval_min": 30
  }' | python3 -m json.tool
```

**Security note:** The `client_secret` is forwarded to Azure Key Vault immediately and is never persisted in the database. The `client_secret_ref` key name stored in the DB refers to the Key Vault entry.

Returns `201 Created` on success. Returns `409 CONFLICT` if a connection already exists for this tenant.

### Update an Existing Connection

Send only the fields you want to change. All fields are optional.

```bash
curl -s -X PUT "$BASE_URL/api/v1/entra/connection" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "auto_provision": false,
    "sync_interval_min": 60
  }' | python3 -m json.tool
```

To rotate the client secret (e.g., after expiry):

```bash
curl -s -X PUT "$BASE_URL/api/v1/entra/connection" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "client_secret": "new-rotated-secret"
  }' | python3 -m json.tool
```

This triggers a Key Vault update — the old secret is replaced.

### Delete an Entra Connection

Disables Entra SSO for the tenant. Existing users who logged in via Entra keep their accounts but can no longer use SSO.

```bash
curl -s -X DELETE "$BASE_URL/api/v1/entra/connection" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

Expected response:

```json
{
  "success": true,
  "data": {
    "message": "Entra connection deleted"
  }
}
```

---

## 24. Entra ID — Group Mappings

Group mappings define how Azure AD groups in the customer tenant map to internal Vesper groups. When a user logs in via Entra or when a sync runs, their Entra group memberships are translated to Vesper group memberships automatically.

### List Group Mappings

```bash
curl -s "$BASE_URL/api/v1/entra/group-mappings" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

Expected response:

```json
{
  "success": true,
  "data": [
    {
      "id": "uuid",
      "tenant_id": "uuid",
      "entra_group_id": "aaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee",
      "entra_group_name": "Sales Team - Entra",
      "internal_group_id": "uuid-of-vesper-group",
      "sync_direction": "entra_to_internal",
      "auto_remove": true,
      "is_whitelisted": false,
      "last_synced_at": null
    }
  ]
}
```

### Create a Group Mapping

You need two pieces of information before calling this:
1. The **Entra group object ID** — find it in Azure AD Portal → Groups → select group → copy the Object ID
2. The **internal Vesper group UUID** — call `GET /api/v1/groups` to list groups and get the ID

```bash
# First get your internal Vesper group IDs
curl -s "$BASE_URL/api/v1/groups" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool

export INTERNAL_GROUP_ID="<vesper-group-uuid>"
export ENTRA_GROUP_ID="<azure-ad-group-object-id>"

# Create the mapping
curl -s -X POST "$BASE_URL/api/v1/entra/group-mappings" \
  -H "Content-Type: application/json" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "entra_group_id": "'$ENTRA_GROUP_ID'",
    "entra_group_name": "Sales Team - Entra",
    "internal_group_id": "'$INTERNAL_GROUP_ID'",
    "sync_direction": "entra_to_internal",
    "auto_remove": true,
    "is_whitelisted": false
  }' | python3 -m json.tool
```

**Field reference:**

| Field | Description |
|-------|-------------|
| `entra_group_id` | Object ID of the group in Azure AD (UUID format) |
| `entra_group_name` | Display label (for human readability only) |
| `internal_group_id` | Vesper group to sync into |
| `sync_direction` | Always `"entra_to_internal"` (other directions not yet implemented) |
| `auto_remove` | If `true`, users removed from the Entra group are also removed from the Vesper group on next sync |
| `is_whitelisted` | If `true`, only users in whitelisted groups can log in via Entra (access control gate) |

### Delete a Group Mapping

```bash
export MAPPING_ID="<mapping-uuid>"

curl -s -X DELETE "$BASE_URL/api/v1/entra/group-mappings/$MAPPING_ID" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

---

## 25. Entra ID — Login Flow & Frontend Integration

### Complete Flow Diagram

```
Browser / Frontend
    │
    │ 1. GET /api/v1/auth/entra/authorize?tenant_slug=acme
    ▼
Backend
    │  • Looks up tenant's entra_tenant_id from DB
    │  • Generates state (UUID) and stores it in Redis (5-min TTL)
    │  • Builds Microsoft authorization URL with state, client_id, scopes, redirect_uri
    ▼
    Returns { authorization_url, state }
    │
    │ 2. Frontend redirects browser to authorization_url
    ▼
Microsoft Login Page  (user enters Entra credentials)
    │
    │ 3. Microsoft redirects to:
    │    GET /api/v1/auth/entra/callback?code=<auth_code>&state=<state>
    ▼
Backend
    │  • Validates state matches value stored in Redis (CSRF check)
    │  • Exchanges code for ID token + access token via MSAL
    │  • Extracts user claims: oid (object ID), email, name
    │  • Looks up or auto-provisions user in DB (if auto_provision=true)
    │  • Creates/updates UserIdentity record linking Entra OID to local user
    │  • Applies group mappings (adds/removes from Vesper groups)
    │  • Issues platform JWT (RS256, 15 min) + refresh token
    │  • Sets HttpOnly refresh cookie
    ▼
    Returns { access_token, token_type, expires_in, user }
    │
    │ 4. Frontend stores access_token in memory, user is logged in
    ▼
All subsequent requests use platform JWT exactly like local login
```

### Step 1 — Get the Authorization URL

The frontend calls this when the user clicks "Sign in with Microsoft":

```bash
curl -s "$BASE_URL/api/v1/auth/entra/authorize?tenant_slug=$TENANT_SLUG" \
  | python3 -m json.tool
```

This endpoint does **not** require `X-Tenant-ID` or `Authorization` headers — it is public.

Expected response:

```json
{
  "success": true,
  "data": {
    "authorization_url": "https://login.microsoftonline.com/CUSTOMER_TENANT_ID/oauth2/v2.0/authorize?client_id=...&response_type=code&redirect_uri=...&scope=openid+profile+email&state=abc123",
    "state": "abc123"
  }
}
```

**Frontend:** Redirect the browser (not an XHR/fetch call) to `authorization_url`. Do not make a fetch request — the user must navigate to Microsoft's login page.

```javascript
const res = await fetch(`/api/v1/auth/entra/authorize?tenant_slug=${tenantSlug}`);
const { data } = await res.json();
// Full-page redirect to Microsoft login
window.location.href = data.authorization_url;
```

### Step 2 — Handle the Callback

Microsoft redirects the browser back to:

```
GET /api/v1/auth/entra/callback?code=<authorization_code>&state=<state>
```

This URL is called **by the browser**, not by your frontend code — it is a server-side redirect. The backend handles everything: token exchange, state validation, user provisioning. It then responds with the access token (and sets the refresh cookie).

The frontend only needs to handle the final page load at the callback URL. If the app is a SPA, configure the backend to redirect to a frontend route after the callback, carrying the token in the response body or a short-lived URL parameter.

### Step 3 — After Callback: Standard Token Handling

Once the backend returns the access token from the callback, treat it exactly like a local login response:

```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGci...",
    "token_type": "bearer",
    "expires_in": 900,
    "user": {
      "id": "uuid",
      "email": "jane@customer.com",
      "display_name": "Jane Doe",
      "is_tenant_admin": false
    }
  }
}
```

Store `access_token` in memory. The refresh cookie is automatically set by the server.

### Frontend Code Example

```javascript
// Login page: "Sign in with Microsoft" button click
async function initiateEntraLogin(tenantSlug) {
  const res = await fetch(`/api/v1/auth/entra/authorize?tenant_slug=${tenantSlug}`);
  const json = await res.json();
  if (json.success) {
    // Store state for optional frontend-side verification
    sessionStorage.setItem('entra_state', json.data.state);
    // Full-page redirect — cannot use fetch/XHR here
    window.location.href = json.data.authorization_url;
  }
}

// Callback page (if your SPA handles /auth/entra/callback route):
// The backend has already processed the code exchange.
// Read the access token from the API response body that was set in the redirect.
// Typically: backend sets access_token as a query param in a redirect to the SPA.
// Implementation depends on your redirect strategy (see backend config).
```

---

## 26. Entra ID — Sync Operations

Group sync keeps Vesper group memberships in sync with Azure AD group memberships.

### Trigger a Manual Sync

```bash
curl -s -X POST "$BASE_URL/api/v1/entra/sync" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

Expected response (sync is queued asynchronously):

```json
{
  "success": true,
  "data": {
    "status": "queued",
    "message": "Sync has been queued"
  }
}
```

The sync runs in the background. Poll the connection endpoint to check `last_sync_at`:

```bash
# Poll until last_sync_at is updated
curl -s "$BASE_URL/api/v1/entra/connection" \
  -H "X-Tenant-ID: $TENANT_ID" \
  -H "Authorization: Bearer $TOKEN" \
  | python3 -c "import sys,json; d=json.load(sys.stdin)['data']; print('Last sync:', d['last_sync_at'])"
```

### What the Sync Does

1. Fetches all group memberships for each mapped Entra group via Microsoft Graph API
2. For each user in the Entra group, finds the matching local user by Entra `oid` (object ID) or email
3. Adds the user to the corresponding Vesper group if not already a member
4. If `auto_remove: true` on the mapping, removes users from the Vesper group who are no longer in the Entra group
5. Updates `last_sync_at` on the connection record
6. Writes a `sync_audit_log` row with counts: `users_created`, `users_updated`, `users_disabled`, `groups_synced`

### Sync Audit Log

Check sync history in the database via the `sync_audit_log` table. Fields:

| Field | Description |
|-------|-------------|
| `sync_type` | `user_sync`, `group_sync`, or `full_sync` |
| `status` | `started`, `success`, `partial`, `failed` |
| `users_created` | New users auto-provisioned |
| `users_updated` | Existing users updated |
| `users_disabled` | Users deactivated (removed from all whitelisted groups) |
| `groups_synced` | Number of group mappings processed |
| `error_message` | Error details if `status` is `failed` or `partial` |
| `started_at`, `completed_at` | Sync duration |

---

## 27. Entra ID — Testing Strategy

### Test Layers

Testing Entra requires mocking at different levels. Here is the recommended approach for each layer.

#### Layer 1 — Unit Tests (Mock MSAL & Key Vault)

Test the service logic without real Azure calls.

```python
# tests/test_entra_service.py

import pytest
from unittest.mock import AsyncMock, patch, MagicMock
from app.services.entra_service import EntraService

@pytest.mark.asyncio
async def test_create_connection_success(db, fake_redis, tenant):
    service = EntraService(db=db, cache=fake_redis)

    with patch("app.services.entra_service.KeyVaultClient") as mock_kv:
        mock_kv.return_value.set_secret = AsyncMock()

        connection = await service.create_connection(
            tenant_id=tenant.id,
            data=MagicMock(
                entra_tenant_id="aaa-bbb-ccc",
                client_id="yyy-zzz",
                client_secret="test-secret",
                scopes="openid profile email",
                whitelisted_groups=[],
                auto_provision=True,
                auto_sync_groups=False,
                sync_interval_min=30,
            )
        )

        assert connection.entra_tenant_id == "aaa-bbb-ccc"
        mock_kv.return_value.set_secret.assert_called_once()

@pytest.mark.asyncio
async def test_create_connection_duplicate_raises(db, fake_redis, tenant):
    service = EntraService(db=db, cache=fake_redis)
    # Create once
    await service.create_connection(tenant.id, ...)
    # Second call should raise ValidationError
    with pytest.raises(ValidationError):
        await service.create_connection(tenant.id, ...)
```

#### Layer 2 — API Integration Tests (Mock AuthService)

Test the route layer using the test client, mocking the MSAL exchange.

```python
# tests/test_entra_api.py

import pytest
from unittest.mock import AsyncMock, patch

@pytest.mark.asyncio
async def test_get_authorize_url(client, tenant):
    with patch("app.api.v1.auth.AuthService.get_entra_authorize_url",
               new_callable=AsyncMock) as mock_auth:
        mock_auth.return_value = {
            "authorization_url": "https://login.microsoftonline.com/...",
            "state": "abc123"
        }

        resp = await client.get(
            f"/api/v1/auth/entra/authorize?tenant_slug={tenant.slug}"
        )

        assert resp.status_code == 200
        data = resp.json()["data"]
        assert "authorization_url" in data
        assert "state" in data
        assert data["authorization_url"].startswith("https://login.microsoftonline.com")


@pytest.mark.asyncio
async def test_entra_callback_success(client, tenant):
    with patch("app.api.v1.auth.AuthService.entra_callback",
               new_callable=AsyncMock) as mock_cb:
        mock_cb.return_value = MagicMock(
            access_token="eyJhbGci...",
            token_type="bearer",
            expires_in=900,
            user_id="uuid",
            refresh_token="opaque-token"
        )

        resp = await client.get(
            "/api/v1/auth/entra/callback",
            params={"code": "auth-code-123", "state": "abc123"}
        )

        assert resp.status_code == 200
        assert "access_token" in resp.json()["data"]
        # Verify refresh cookie was set
        assert "refresh_token" in resp.cookies


@pytest.mark.asyncio
async def test_entra_callback_invalid_state(client, tenant):
    with patch("app.api.v1.auth.AuthService.entra_callback",
               side_effect=AuthenticationError("Invalid state")):

        resp = await client.get(
            "/api/v1/auth/entra/callback",
            params={"code": "auth-code", "state": "tampered-state"}
        )

        assert resp.status_code == 401
        assert resp.json()["error"]["code"] == "AUTHENTICATION_FAILED"


@pytest.mark.asyncio
async def test_create_entra_connection(client, tenant, admin_token):
    resp = await client.post(
        "/api/v1/entra/connection",
        headers={
            "X-Tenant-ID": str(tenant.id),
            "Authorization": f"Bearer {admin_token}",
        },
        json={
            "entra_tenant_id": "aaa-bbb-ccc-ddd",
            "client_id": "yyy-yyy-yyy-yyy",
            "client_secret": "test-secret",
            "scopes": "openid profile email",
            "whitelisted_groups": [],
            "auto_provision": True,
            "auto_sync_groups": True,
            "sync_interval_min": 30,
        },
    )
    assert resp.status_code == 201
    data = resp.json()["data"]
    assert data["entra_tenant_id"] == "aaa-bbb-ccc-ddd"
    # client_id should be masked
    assert "yyy-yyy" not in data["client_id"]


@pytest.mark.asyncio
async def test_group_mapping_lifecycle(client, tenant, admin_token, test_group):
    # Create mapping
    resp = await client.post(
        "/api/v1/entra/group-mappings",
        headers={"X-Tenant-ID": str(tenant.id), "Authorization": f"Bearer {admin_token}"},
        json={
            "entra_group_id": "entra-group-object-id",
            "entra_group_name": "Sales",
            "internal_group_id": str(test_group.id),
            "sync_direction": "entra_to_internal",
            "auto_remove": True,
            "is_whitelisted": False,
        },
    )
    assert resp.status_code == 201
    mapping_id = resp.json()["data"]["id"]

    # List
    resp = await client.get(
        "/api/v1/entra/group-mappings",
        headers={"X-Tenant-ID": str(tenant.id), "Authorization": f"Bearer {admin_token}"},
    )
    assert any(m["id"] == mapping_id for m in resp.json()["data"])

    # Delete
    resp = await client.delete(
        f"/api/v1/entra/group-mappings/{mapping_id}",
        headers={"X-Tenant-ID": str(tenant.id), "Authorization": f"Bearer {admin_token}"},
    )
    assert resp.status_code == 200
```

#### Layer 3 — End-to-End Manual Testing

For validating the full OIDC round-trip in a dev/staging environment. You cannot do this with curl alone because the flow requires a real browser redirect to Microsoft.

**Option A — Use a real browser (simplest)**

1. Open `http://localhost:8000/docs`
2. Call `GET /auth/entra/authorize?tenant_slug=<your_slug>` directly from Swagger
3. Copy `authorization_url` from the response
4. Open it in a browser tab
5. Sign in with a Microsoft account that belongs to the customer's Azure AD tenant
6. Microsoft redirects to your callback URL — the backend processes it and returns tokens
7. Copy the `access_token` from the response and use it in subsequent Swagger calls

**Option B — Mock Microsoft's token endpoint (dev only)**

Use a tool like [MSALTestApplication](https://github.com/AzureAD/microsoft-authentication-library-for-python) or intercept the MSAL call in your service with a stub that returns a hardcoded ID token.

Minimal fixture for a fake ID token:

```python
# tests/fixtures/entra_mock.py

FAKE_ID_TOKEN_CLAIMS = {
    "oid": "entra-user-object-id-12345",
    "email": "jane@customer.com",
    "name": "Jane Doe",
    "tid": "customer-azure-tenant-id",
    "iss": "https://login.microsoftonline.com/customer-azure-tenant-id/v2.0",
}

# Use in tests:
with patch("app.services.entra_service.msal.ConfidentialClientApplication") as mock_app:
    mock_app.return_value.acquire_token_by_authorization_code.return_value = {
        "id_token_claims": FAKE_ID_TOKEN_CLAIMS,
        "access_token": "entra-access-token",
    }
```

### Test Cases Checklist

#### Authentication Flow
- [ ] `GET /auth/entra/authorize` returns `authorization_url` and `state`
- [ ] `state` is a non-empty string (CSRF token)
- [ ] `GET /auth/entra/callback` with valid `code` + `state` returns `access_token`
- [ ] Callback sets `refresh_token` HttpOnly cookie
- [ ] Callback with tampered `state` returns `401 AUTHENTICATION_FAILED`
- [ ] Callback with expired `state` (Redis TTL elapsed) returns `401`
- [ ] Callback with invalid `code` (MSAL exchange fails) returns `401`
- [ ] First-time login auto-provisions user when `auto_provision=true`
- [ ] First-time login is rejected when `auto_provision=false`
- [ ] Second login for same Entra OID updates `last_used_at` on `UserIdentity`

#### Connection Management
- [ ] `POST /entra/connection` creates connection, returns `201`
- [ ] `POST /entra/connection` a second time returns `409 CONFLICT`
- [ ] `GET /entra/connection` returns connection with masked `client_id`
- [ ] `PUT /entra/connection` updates only the provided fields
- [ ] `PUT /entra/connection` with new `client_secret` triggers Key Vault update
- [ ] `DELETE /entra/connection` removes the connection
- [ ] All connection endpoints return `403` for non-admin users

#### Group Mappings
- [ ] `POST /entra/group-mappings` creates mapping, returns `201`
- [ ] `GET /entra/group-mappings` lists all mappings for tenant (tenant-scoped)
- [ ] `DELETE /entra/group-mappings/{id}` removes mapping
- [ ] Attempting to delete another tenant's mapping returns `400 VALIDATION_ERROR`

#### Sync
- [ ] `POST /entra/sync` returns `200` with `status: queued`
- [ ] Non-admin users receive `403` on `POST /entra/sync`

---

## 28. Entra ID — Troubleshooting

### "ENTRA_CONFIG_ERROR" on connection endpoints

**Cause:** No Entra connection has been created for this tenant yet.
**Fix:** Call `POST /api/v1/entra/connection` with the customer's Azure AD credentials first.

### Authorization URL returns 500

**Cause:** Platform-level Entra config is incomplete (missing `ENTRA_PLATFORM_CLIENT_ID`, `ENTRA_PLATFORM_TENANT_ID`, or Key Vault unreachable).
**Fix:**
1. Verify all `ENTRA_PLATFORM_*` env vars are set
2. Verify `KEY_VAULT_URL` is reachable from the backend container
3. Verify the Managed Identity (`AZURE_CLIENT_ID`) has `Key Vault Secrets User` role on the Key Vault

```bash
# Check env vars are loaded (dev only — never do this in prod)
curl -s "$BASE_URL/api/v1/health" | python3 -m json.tool
```

### "AUTHENTICATION_FAILED" on callback with correct credentials

**Cause 1:** State has expired — Redis TTL (5 minutes) elapsed between `authorize` and `callback`.
**Fix:** The authorization flow must complete within 5 minutes. If users are slow, increase the Redis TTL for state keys in the auth service config.

**Cause 2:** `state` mismatch — frontend modified or re-encoded the state parameter.
**Fix:** Never modify the `state` value from the `authorize` response before it reaches the `callback` URL.

**Cause 3:** Redirect URI mismatch — the `ENTRA_CALLBACK_BASE_URL` doesn't match what's registered in Azure.
**Fix:** Go to Azure AD → App registrations → Authentication → Redirect URIs. The value must exactly match `ENTRA_CALLBACK_BASE_URL + /api/v1/auth/entra/callback`. Any trailing slash or HTTP vs HTTPS difference causes failure.

### User logs in with Entra but gets "User not found" or 403

**Cause:** `auto_provision` is `false` and the user has never logged in locally.
**Fix:** Either set `auto_provision: true` on the connection (via `PUT /entra/connection`) or create the user manually via `POST /api/v1/users` using the same email address as their Entra account.

### Group memberships not applying after Entra login

**Cause 1:** No group mappings have been configured.
**Fix:** Create mappings via `POST /api/v1/entra/group-mappings`.

**Cause 2:** The Entra group object ID in the mapping is wrong.
**Fix:** Go to Azure AD → Groups → select the group → copy the **Object ID** (not the display name). Recreate the mapping with the correct value.

**Cause 3:** The App Registration in Azure does not have `GroupMember.Read.All` permission granted.
**Fix:** In Azure AD → App registrations → API permissions → add `GroupMember.Read.All` (Application) and grant admin consent.

### "CONFLICT" when creating a connection

**Cause:** A connection already exists for this tenant.
**Fix:** Use `PUT /api/v1/entra/connection` to update the existing connection instead of POST.

### client_secret rotation: connection still uses old secret

**Cause:** Key Vault secret was updated but the cached MSAL `ConfidentialClientApplication` instance uses an in-memory copy.
**Fix:** After rotating the secret via `PUT /entra/connection`, restart the backend pods to flush the MSAL token cache. In a future release, dynamic credential reloading will be implemented.

---

*Last updated: March 2026 | Vesper Analytics Platform*
