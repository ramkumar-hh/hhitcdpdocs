# Analytics Backend — Frontend Integration Guide

**Audience:** Frontend developers
**Purpose:** Step-by-step guide to integrating with the Vesper DataHub API from a frontend application
**Base URL:** `https://<your-api-host>/api/v1`
**Interactive Docs:** `https://<your-api-host>/docs` (non-production only)

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Step 0 — Configure Your HTTP Client](#2-step-0--configure-your-http-client)
3. [Step 1 — Bootstrap: Resolve Tenant](#3-step-1--bootstrap-resolve-tenant)
4. [Step 2 — Authenticate (Login)](#4-step-2--authenticate-login)
5. [Step 3 — Store & Use the Access Token](#5-step-3--store--use-the-access-token)
6. [Step 4 — Fetch Current User & Permissions](#6-step-4--fetch-current-user--permissions)
7. [Step 5 — Load the Navigation Menu](#7-step-5--load-the-navigation-menu)
8. [Step 6 — Token Refresh Strategy](#8-step-6--token-refresh-strategy)
9. [Step 7 — Logout](#9-step-7--logout)
10. [Step 8 — Microsoft Entra ID (SSO) Login](#10-step-8--microsoft-entra-id-sso-login)
11. [Step 9 — User Management APIs](#11-step-9--user-management-apis)
12. [Step 10 — Role Management APIs](#12-step-10--role-management-apis)
13. [Step 11 — Group Management APIs](#13-step-11--group-management-apis)
14. [Step 12 — Permission Check APIs](#14-step-12--permission-check-apis)
15. [Step 13 — Password Reset Flow](#15-step-13--password-reset-flow)
16. [Error Handling Reference](#16-error-handling-reference)
17. [Permission-Gated UI Patterns](#17-permission-gated-ui-patterns)
18. [Full App Startup Sequence Diagram](#18-full-app-startup-sequence-diagram)
19. [Common Pitfalls](#19-common-pitfalls)

---

## 1. Core Concepts

Before writing any code, understand these three rules — every other section depends on them.

### Rule 1 — Every request needs `X-Tenant-ID`

The platform is multi-tenant. Every API call (except health check) must carry a tenant identifier:

```
X-Tenant-ID: <tenant-uuid>
```

You obtain the tenant UUID during the bootstrap step (Step 1). If this header is missing, the API returns `400 TENANT_REQUIRED`.

### Rule 2 — All responses share one envelope shape

**Success:**
```json
{
  "status": "success",
  "data": { ... },
  "meta": null,
  "errors": null
}
```

**Paginated success:**
```json
{
  "status": "success",
  "data": [ ... ],
  "meta": { "page": 1, "per_page": 20, "total": 150, "total_pages": 8 },
  "errors": null
}
```

**Error:**
```json
{
  "status": "error",
  "data": null,
  "errors": [{ "code": "INVALID_CREDENTIALS", "message": "Invalid credentials" }],
  "meta": null
}
```

Your HTTP client should always read `.data` for success and `.errors[0].code` for error handling.

### Rule 3 — Access token is a Bearer token, refresh token is an HttpOnly cookie

- The **access token** (15-minute JWT) is returned in the response body — store it in memory (never `localStorage`).
- The **refresh token** (7-day opaque token) is delivered as an `HttpOnly Set-Cookie` header — the browser manages it automatically. You never read or write it in JavaScript.

---

## 2. Step 0 — Configure Your HTTP Client

Set up a shared Axios (or Fetch) instance once. All other sections use this client.

```ts
// src/api/client.ts
import axios from 'axios';

const BASE_URL = import.meta.env.VITE_API_URL; // e.g. https://api.vesper.io/api/v1

export const apiClient = axios.create({
  baseURL: BASE_URL,
  withCredentials: true,     // Required — sends the HttpOnly refresh token cookie
  headers: { 'Content-Type': 'application/json' },
});

// Attach access token + tenant ID to every request
apiClient.interceptors.request.use((config) => {
  const token = sessionStorage.getItem('access_token');
  const tenantId = sessionStorage.getItem('tenant_id');

  if (token) config.headers['Authorization'] = `Bearer ${token}`;
  if (tenantId) config.headers['X-Tenant-ID'] = tenantId;

  return config;
});
```

> `withCredentials: true` is mandatory — without it, the browser will not send the HttpOnly refresh token cookie on token refresh calls.

---

## 3. Step 1 — Bootstrap: Resolve Tenant

Before showing the login screen, resolve the current tenant so you have the `tenant_id` and know which auth methods to offer.

**Endpoint:** `GET /api/v1/tenants/resolve?slug=<slug>`
_(This endpoint is public — no `X-Tenant-ID` or token needed.)_

```ts
// src/api/auth.ts
export async function resolveTenant(slug: string) {
  const res = await apiClient.get(`/tenants/resolve?slug=${slug}`);
  return res.data.data; // { tenant_id, slug, display_name, auth_methods, ... }
}
```

**Response:**
```json
{
  "status": "success",
  "data": {
    "tenant_id": "00000000-0000-0000-0000-000000000002",
    "slug": "acme-corp",
    "display_name": "Acme Corporation",
    "auth_methods": "form,entra",
    "primary_domain": "acme.vesper.io"
  }
}
```

**What to do with the response:**
```ts
const tenant = await resolveTenant('acme-corp');
sessionStorage.setItem('tenant_id', tenant.tenant_id);

// Decide which login UI to show:
const methods = tenant.auth_methods.split(',');
// "form"  → show email/password form
// "entra" → show "Sign in with Microsoft" button
```

---

## 4. Step 2 — Authenticate (Login)

### Option A — Email + Password (Form Login)

**Endpoint:** `POST /api/v1/auth/login`
**Headers:** `X-Tenant-ID: <tenant_id>` (set by your interceptor after Step 1)

```ts
export async function login(email: string, password: string, tenantSlug: string) {
  const res = await apiClient.post('/auth/login', {
    email,
    password,
    tenant_slug: tenantSlug,
  });
  return res.data.data; // TokenResponse
}
```

**Request body:**
```json
{
  "email": "user@acme.com",
  "password": "MyPassword@1",
  "tenant_slug": "acme-corp",
  "mfa_code": null
}
```

**Success response (200):**
```json
{
  "status": "success",
  "data": {
    "access_token": "eyJhbGci...",
    "token_type": "bearer",
    "expires_in": 900,
    "user_id": "d86ae2a8-c50c-4ad8-8bbe-665a18b74259",
    "user_type": "tenant_admin",
    "tenant_id": "00000000-0000-0000-0000-000000000002"
  }
}
```

**Store the access token:**
```ts
const { access_token, expires_in, user_id, user_type } = await login(email, password, slug);

// Store in memory — never localStorage
sessionStorage.setItem('access_token', access_token);
sessionStorage.setItem('user_id', user_id);
sessionStorage.setItem('user_type', user_type);

// Schedule a token refresh before it expires
scheduleTokenRefresh(expires_in); // see Step 6
```

**Possible error codes:**

| HTTP | Code | Meaning |
|------|------|---------|
| 401 | `INVALID_CREDENTIALS` | Wrong email or password |
| 423 | `ACCOUNT_LOCKED` | Too many failed attempts (locked for 15 min) |
| 401 | `INVALID_TENANT` | Tenant slug not found |

### Option B — MFA (when required)

If the tenant has MFA enforced, the login endpoint returns a `temp_token` instead of `access_token`. You then call:

**Endpoint:** `POST /api/v1/auth/mfa/verify`

```ts
export async function verifyMfa(mfaCode: string, tempToken: string) {
  const res = await apiClient.post('/auth/mfa/verify', {
    mfa_code: mfaCode,
    temp_token: tempToken,
  });
  return res.data.data; // Full TokenResponse
}
```

---

## 5. Step 3 — Store & Use the Access Token

**Never put the access token in `localStorage`** — it is readable by any script on the page. Use in-memory storage (`sessionStorage` or a module-level variable):

```ts
// src/store/auth.ts  (simple in-memory store, works with Redux/Pinia/Zustand too)
let _accessToken: string | null = null;

export const tokenStore = {
  set(token: string) { _accessToken = token; },
  get() { return _accessToken; },
  clear() { _accessToken = null; },
};
```

The Axios interceptor from Step 0 reads this store and attaches `Authorization: Bearer <token>` to every request automatically.

---

## 6. Step 4 — Fetch Current User & Permissions

Immediately after login, fetch the current user profile and their permission matrix. This single call tells you:
- Who the user is (name, email, avatar)
- What they are allowed to do (used to hide/show UI elements)

**Endpoint:** `GET /api/v1/auth/me`
**Auth required:** Yes (Bearer token)

```ts
export async function getMe() {
  const res = await apiClient.get('/auth/me');
  return res.data.data; // CurrentUserResponse
}
```

**Response:**
```json
{
  "status": "success",
  "data": {
    "id": "d86ae2a8-c50c-4ad8-8bbe-665a18b74259",
    "email": "jane@acme.com",
    "first_name": "Jane",
    "last_name": "Smith",
    "display_name": "Jane Smith",
    "avatar_url": null,
    "status": "active",
    "auth_method": "local",
    "is_tenant_admin": true,
    "is_super_admin": false,
    "mfa_enabled": false,
    "permissions": {
      "users":  { "view": true, "create": true, "edit": true, "delete": false },
      "roles":  { "view": true, "create": true, "edit": true, "delete": false },
      "groups": { "view": true, "create": false, "edit": false, "delete": false }
    }
  }
}
```

**Store the permission matrix** globally so your UI components can check it:
```ts
const me = await getMe();
permissionStore.set(me.permissions);
userStore.set(me);
```

---

## 7. Step 5 — Load the Navigation Menu

The navigation endpoint returns only the menu items the current user is allowed to see (RBAC-filtered server-side).

**Endpoint:** `GET /api/v1/navigation/menu`
**Auth required:** Yes

```ts
export async function getNavigation() {
  const res = await apiClient.get('/navigation/menu');
  return res.data.data.items; // MenuItemWithChildren[]
}
```

**Response:**
```json
{
  "status": "success",
  "data": {
    "items": [
      {
        "id": "...",
        "label": "Dashboard",
        "icon": "dashboard",
        "route": "/dashboard",
        "sort_order": 1,
        "children": []
      },
      {
        "id": "...",
        "label": "Admin",
        "icon": "settings",
        "route": null,
        "sort_order": 2,
        "children": [
          { "label": "Users", "route": "/admin/users", ... },
          { "label": "Roles", "route": "/admin/roles", ... }
        ]
      }
    ]
  }
}
```

Render this tree directly as your sidebar/navbar — no client-side filtering needed.

---

## 8. Step 6 — Token Refresh Strategy

Access tokens expire in **15 minutes**. The refresh token lives in an HttpOnly cookie for **7 days**. Your app must transparently refresh tokens before they expire.

### Approach A — Proactive refresh (recommended)

Schedule a refresh 60 seconds before expiry:

```ts
let refreshTimer: ReturnType<typeof setTimeout>;

export function scheduleTokenRefresh(expiresInSeconds: number) {
  clearTimeout(refreshTimer);
  const refreshInMs = (expiresInSeconds - 60) * 1000; // 60s before expiry
  refreshTimer = setTimeout(doRefresh, refreshInMs);
}

async function doRefresh() {
  try {
    const res = await apiClient.post('/auth/refresh');
    // Cookie is rotated automatically by the browser
    const { access_token, expires_in } = res.data.data;
    tokenStore.set(access_token);
    scheduleTokenRefresh(expires_in);
  } catch {
    // Refresh token expired — redirect to login
    handleSessionExpired();
  }
}
```

### Approach B — Reactive refresh (interceptor)

Intercept 401 responses and retry after refreshing:

```ts
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    const original = error.config;
    if (error.response?.status === 401 && !original._retried) {
      original._retried = true;
      try {
        const res = await apiClient.post('/auth/refresh');
        const { access_token } = res.data.data;
        tokenStore.set(access_token);
        original.headers['Authorization'] = `Bearer ${access_token}`;
        return apiClient(original);
      } catch {
        handleSessionExpired();
        return Promise.reject(error);
      }
    }
    return Promise.reject(error);
  }
);
```

**Endpoint:** `POST /api/v1/auth/refresh`
No body required — the HttpOnly cookie is sent automatically by the browser (`withCredentials: true`).

---

## 9. Step 7 — Logout

Call logout to invalidate the server-side session and clear the refresh token cookie:

**Endpoint:** `POST /api/v1/auth/logout`
**Auth required:** Yes

```ts
export async function logout() {
  await apiClient.post('/auth/logout');
  // Clear all local state
  tokenStore.clear();
  sessionStorage.clear();
  clearTimeout(refreshTimer);
  window.location.href = '/login';
}
```

The server:
1. Adds the access token to a Redis blocklist
2. Deletes the `refresh_token` cookie via `Set-Cookie: refresh_token=; Max-Age=0`

---

## 10. Step 8 — Microsoft Entra ID (SSO) Login

Use this flow when `auth_methods` includes `"entra"`.

### Step 8a — Get the authorization URL

**Endpoint:** `GET /api/v1/auth/entra/authorize?tenant_slug=<slug>`

```ts
export async function getEntraAuthUrl(tenantSlug: string) {
  const res = await apiClient.get(`/auth/entra/authorize?tenant_slug=${tenantSlug}`);
  return res.data.data; // { authorization_url, state }
}
```

```ts
// In your "Sign in with Microsoft" button handler:
const { authorization_url } = await getEntraAuthUrl('acme-corp');
window.location.href = authorization_url; // Redirect user to Microsoft
```

### Step 8b — Handle the callback

After Microsoft redirects back to your app at `/auth/callback?code=...&state=...`:

```ts
// Your callback page reads query params and calls the backend:
const params = new URLSearchParams(window.location.search);
const code  = params.get('code');
const state = params.get('state');

const res = await apiClient.get(
  `/auth/entra/callback?code=${code}&state=${state}`
);
const { access_token, expires_in } = res.data.data;
tokenStore.set(access_token);
scheduleTokenRefresh(expires_in);
// Continue to getMe() → getNavigation() as with form login
```

---

## 11. Step 9 — User Management APIs

All user management endpoints require a **tenant admin** token (`user_type === "tenant_admin"`) or super admin.

### List users (paginated)

**`GET /api/v1/users?page=1&per_page=20&status=active`**

```ts
export async function listUsers(page = 1, perPage = 20, status?: string) {
  const params = new URLSearchParams({ page: String(page), per_page: String(perPage) });
  if (status) params.append('status', status);
  const res = await apiClient.get(`/users?${params}`);
  return res.data; // { data: UserListItem[], meta: PaginatedMeta }
}
```

**Response `meta`:**
```json
{ "page": 1, "per_page": 20, "total": 87, "total_pages": 5 }
```

### Create a user

**`POST /api/v1/users`**

```ts
export async function createUser(payload: {
  email: string;
  first_name: string;
  last_name: string;
  password?: string;
  auth_method?: 'local' | 'entra';
  is_tenant_admin?: boolean;
}) {
  const res = await apiClient.post('/users', payload);
  return res.data.data; // UserResponse
}
```

### Get a user

**`GET /api/v1/users/{user_id}`**

```ts
export async function getUser(userId: string) {
  const res = await apiClient.get(`/users/${userId}`);
  return res.data.data;
}
```

### Update a user

**`PATCH /api/v1/users/{user_id}`**
Send only the fields you want to change (all fields optional):

```ts
export async function updateUser(userId: string, patch: Partial<{
  first_name: string;
  last_name: string;
  display_name: string;
  status: string;          // 'active' | 'inactive' | 'suspended'
  is_tenant_admin: boolean;
  mfa_enabled: boolean;
}>) {
  const res = await apiClient.patch(`/users/${userId}`, patch);
  return res.data.data;
}
```

### Deactivate / delete a user

```ts
// Soft deactivate (recommended)
await updateUser(userId, { status: 'inactive' });

// Hard delete
await apiClient.delete(`/users/${userId}`);
```

### Assign roles to a user

**`POST /api/v1/users/{user_id}/roles`**

```ts
export async function assignRolesToUser(userId: string, roleIds: string[]) {
  await apiClient.post(`/users/${userId}/roles`, { role_ids: roleIds });
}
```

---

## 12. Step 10 — Role Management APIs

Roles require **tenant admin** access.

### List roles

**`GET /api/v1/roles?page=1&per_page=20`**

```ts
export async function listRoles(page = 1, perPage = 20) {
  const res = await apiClient.get(`/roles?page=${page}&per_page=${perPage}`);
  return res.data; // { data: RoleResponse[], meta: PaginatedMeta }
}
```

### Create a role

**`POST /api/v1/roles`**

```ts
export async function createRole(name: string, description?: string) {
  const res = await apiClient.post('/roles', { name, description });
  return res.data.data; // RoleResponse
}
```

**Request body:**
```json
{ "name": "Report Viewer", "description": "Can view but not export reports", "priority": 10 }
```

### Get role with its permissions

**`GET /api/v1/roles/{role_id}`**

```ts
export async function getRole(roleId: string) {
  const res = await apiClient.get(`/roles/${roleId}`);
  return res.data.data; // RoleWithPermissions
}
```

**Response:**
```json
{
  "id": "...",
  "name": "Report Viewer",
  "is_system_role": false,
  "permissions": [
    { "resource_key": "reports", "action_key": "view", "grant_type": "allow" }
  ]
}
```

### Grant a permission to a role

**`POST /api/v1/roles/{role_id}/permissions`**

```ts
export async function grantPermission(
  roleId: string,
  resourceKey: string,
  actionKey: string,
  grantType: 'allow' | 'deny' = 'allow'
) {
  await apiClient.post(`/roles/${roleId}/permissions`, {
    resource_key: resourceKey,
    action_key: actionKey,
    grant_type: grantType,
  });
}
```

### Revoke a permission

**`DELETE /api/v1/roles/{role_id}/permissions/{permission_id}`**

```ts
export async function revokePermission(roleId: string, permissionId: string) {
  await apiClient.delete(`/roles/${roleId}/permissions/${permissionId}`);
}
```

---

## 13. Step 11 — Group Management APIs

Groups allow assigning roles to a collection of users. Requires **tenant admin**.

### List groups

**`GET /api/v1/groups?page=1&per_page=20`**

```ts
export async function listGroups() {
  const res = await apiClient.get('/groups');
  return res.data;
}
```

### Create a group

**`POST /api/v1/groups`**

```ts
export async function createGroup(name: string, description?: string) {
  const res = await apiClient.post('/groups', { name, description });
  return res.data.data;
}
```

### Add a user to a group

**`POST /api/v1/groups/{group_id}/members`**

```ts
export async function addGroupMember(groupId: string, userId: string) {
  await apiClient.post(`/groups/${groupId}/members`, { user_id: userId });
}
```

### Remove a user from a group

**`DELETE /api/v1/groups/{group_id}/members/{user_id}`**

```ts
export async function removeGroupMember(groupId: string, userId: string) {
  await apiClient.delete(`/groups/${groupId}/members/${userId}`);
}
```

### Assign a role to a group

**`POST /api/v1/groups/{group_id}/roles`**

```ts
export async function assignRoleToGroup(groupId: string, roleId: string) {
  await apiClient.post(`/groups/${groupId}/roles`, { role_id: roleId });
}
```

---

## 14. Step 12 — Permission Check APIs

### Check a single permission on-demand

Use this when you need a fresh server-side check (e.g. before showing a destructive action):

**`POST /api/v1/permissions/check`**

```ts
export async function checkPermission(resource: string, action: string) {
  const res = await apiClient.post('/permissions/check', { resource, action });
  return res.data.data.allowed; // boolean
}

// Example: before showing "Delete User" button
const canDelete = await checkPermission('users', 'delete');
```

### Get available resources and actions

Useful for building a permission-assignment UI:

```ts
// All resources (protected areas of the app)
const resources = await apiClient.get('/permissions/resources').then(r => r.data.data);

// All actions (view, create, edit, delete, export, …)
const actions = await apiClient.get('/permissions/actions').then(r => r.data.data);
```

---

## 15. Step 13 — Password Reset Flow

### Request a reset link

**`POST /api/v1/auth/password/reset`**
No auth required. Always returns 200 (avoids account enumeration).

```ts
export async function requestPasswordReset(email: string) {
  await apiClient.post('/auth/password/reset', { email });
  // Always show "If an account exists, we've sent a link" — regardless of response
}
```

### Confirm the reset (from the email link)

The email link takes the user to your frontend with a `?token=<token>` query param.

**`POST /api/v1/auth/password/reset/confirm`**

```ts
export async function confirmPasswordReset(token: string, newPassword: string) {
  await apiClient.post('/auth/password/reset/confirm', {
    token,
    new_password: newPassword,
  });
}
```

### Change password (authenticated user)

**`POST /api/v1/auth/password/change`**

```ts
export async function changePassword(currentPassword: string, newPassword: string) {
  await apiClient.post('/auth/password/change', {
    current_password: currentPassword,
    new_password: newPassword,
  });
}
```

---

## 16. Error Handling Reference

Handle errors by reading `error.response.data.errors[0].code`:

```ts
// Global Axios error handler (put in your interceptor)
apiClient.interceptors.response.use(
  (res) => res,
  (error) => {
    const code    = error.response?.data?.errors?.[0]?.code;
    const message = error.response?.data?.errors?.[0]?.message;
    const status  = error.response?.status;

    switch (code) {
      case 'INVALID_CREDENTIALS':  showToast('Invalid email or password');    break;
      case 'ACCOUNT_LOCKED':       showToast('Account locked. Try again in 15 minutes.'); break;
      case 'PERMISSION_DENIED':    showToast('You do not have permission.');   break;
      case 'TENANT_REQUIRED':      redirectToTenantResolver();                break;
      default:
        if (status === 401) redirectToLogin();
        else showToast(message ?? 'An unexpected error occurred.');
    }
    return Promise.reject(error);
  }
);
```

**HTTP status → meaning:**

| Status | Typical code | What it means for UI |
|--------|-------------|----------------------|
| 400 | `TENANT_REQUIRED` | Restart bootstrap — `X-Tenant-ID` missing |
| 401 | `INVALID_CREDENTIALS` | Show login error |
| 401 | `TOKEN_EXPIRED` | Trigger token refresh; if that fails, redirect to login |
| 403 | `PERMISSION_DENIED` | Hide the feature or show "Access Denied" |
| 404 | `USER_NOT_FOUND` | Show not-found state |
| 409 | `USER_ALREADY_EXISTS` | Show duplicate-email error on user creation form |
| 422 | (Pydantic) | Validation failure — read `errors[0].message` for field-level detail |
| 423 | `ACCOUNT_LOCKED` | Show lockout timer |
| 500 | `INTERNAL_ERROR` | Show generic error; log to Sentry |

---

## 17. Permission-Gated UI Patterns

The `permissions` object from `GET /auth/me` has this shape:

```json
{
  "users":   { "view": true, "create": true, "edit": true, "delete": false },
  "roles":   { "view": true, "create": false, "edit": false, "delete": false },
  "reports": { "view": true, "export": false }
}
```

### React example

```tsx
// src/hooks/usePermission.ts
import { usePermissionStore } from './permissionStore';

export function usePermission(resource: string, action: string): boolean {
  const matrix = usePermissionStore((s) => s.matrix);
  return matrix?.[resource]?.[action] ?? false;
}

// Usage in a component:
function UserListPage() {
  const canCreate = usePermission('users', 'create');
  const canDelete = usePermission('users', 'delete');

  return (
    <>
      {canCreate && <Button onClick={openCreateModal}>+ Invite User</Button>}
      <UserTable
        onDelete={canDelete ? handleDelete : undefined}
      />
    </>
  );
}
```

### Vue example

```ts
// composables/usePermission.ts
import { useAuthStore } from '@/stores/auth';

export function usePermission(resource: string, action: string) {
  const auth = useAuthStore();
  return computed(() => auth.permissions?.[resource]?.[action] ?? false);
}
```

### `is_tenant_admin` vs permission matrix

- `is_tenant_admin: true` → admin UI sections (Users, Roles, Groups, Audit tabs) should be visible
- Permission matrix → fine-grained per-action control within those sections
- Regular users (`is_tenant_admin: false`) should never see admin sections even if the route is typed in

---

## 18. Full App Startup Sequence Diagram

```
Browser loads app
       │
       ▼
1. GET /tenants/resolve?slug=<slug>
   → store tenant_id, auth_methods
       │
       ▼
2. Show login screen based on auth_methods
   ┌─────────────────┐   ┌──────────────────────────────┐
   │  form login     │   │  Entra SSO                   │
   │  POST /auth/    │   │  GET /auth/entra/authorize   │
   │  login          │   │  → redirect to Microsoft     │
   └────────┬────────┘   │  Microsoft redirects back    │
            │            │  GET /auth/entra/callback    │
            └─────────────────────────┘
                         │
                         ▼
              3. Store access_token in memory
                 Schedule token refresh timer
                         │
                         ▼
              4. GET /auth/me
                 → store user profile + permissions
                         │
                         ▼
              5. GET /navigation/menu
                 → render sidebar/navbar
                         │
                         ▼
              6. Navigate to default route
                 (App is ready)

--- Every 14 minutes (or on 401) ---

              POST /auth/refresh
              → new access_token (cookie rotates automatically)
              → reschedule refresh timer
```

---

## 19. Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Missing `X-Tenant-ID` header | `400 TENANT_REQUIRED` on every call | Set it in your Axios interceptor after Step 1 |
| Missing `withCredentials: true` | Token refresh always fails (401) | Add to Axios instance config; check CORS `allow_credentials` |
| Storing access token in `localStorage` | XSS can steal tokens | Use in-memory store (`sessionStorage` or module variable) |
| Not scheduling token refresh | Users get logged out after 15 min | Schedule `POST /auth/refresh` 60 seconds before `expires_in` |
| Reading `.data` without checking `.status` | Showing stale data on error | Always check `response.status === "success"` first |
| Sending wrong tenant field name | `422` on tenant create | Use `display_name`, not `name`; use `subscription_tier`, not `plan` |
| Permission check on every render | Performance issues | Cache `GET /auth/me` permissions; only call `POST /permissions/check` for sensitive one-off actions |
| Not handling MFA in login flow | Users with MFA can't log in | Check if response contains `temp_token` — if so, prompt for TOTP and call `/auth/mfa/verify` |
