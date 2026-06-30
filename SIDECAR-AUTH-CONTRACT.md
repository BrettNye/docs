# Bedrock Sidecar Auth Contract

**Version:** 1.0 (W3.2) · **Date:** 2026-06-30 · **Audience:** consumer service authors, SDK maintainers.

This document is the authoritative auth contract for callers of the Bedrock **sidecar** (the `api-management` HTTP service). It covers which credentials exist, what surfaces each credential can reach, how end-user identity is forwarded across the service boundary, and how errors map to HTTP status codes.

---

## Credentials

Bedrock recognizes three credential kinds. Each maps to a disjoint set of API surfaces — choose the one that matches your use case.

### Service api-key (`kind=service`)

Used by **backend services** that call Bedrock on behalf of a user or autonomously.

- **Header:** `x-api-key: <key>`
- **Reaches:** the authorization decision path (`POST /evaluate`, `GET /effective-permissions`, gated by `@ServiceAuth`) and service-level operations.
- **Does NOT reach:** core control-plane endpoints (`@PlatformOnly`).
- **Cannot** substitute a user identity — the caller names the subject explicitly in the request body.

### User JWT, `Authorization: Bearer` (`kind=user`) — NEW

Used by **consumer services forwarding an authenticated end-user's identity** to the Bedrock management surfaces. This is the JWT pass-through model.

- **Headers:** `Authorization: Bearer <kinde-access-token>` and `X-Tenant-Id: <tenantId>`
- **Reaches:** tenant-scoped management APIs (`@TenantOwned`, `@Authenticated`).
- **Does NOT reach:** `/evaluate` or `/effective-permissions` (service api-key only), or core control-plane (`@PlatformOnly`).
- **Cannot** assert alternative subjects (`onBehalfOf` is disallowed for `kind=user`).

### Platform api-key (`kind=platform`)

Used by Bedrock administrators and internal tooling.

- **Headers:** `x-api-key: <key>` (with `kind=platform` on the key record)
- **Reaches:** everything, including `@PlatformOnly` endpoints.

---

## Credential → Surface Matrix

| Credential | `kind` | `@ServiceAuth` (`/evaluate`, `/effective-permissions`) | `@TenantOwned` / `@Authenticated` (management) | `@PlatformOnly` (control-plane) |
|---|---|---|---|---|
| Service api-key | `service` | Yes | Yes | No |
| User JWT (Bearer) | `user` | **No** | Yes | No |
| Platform api-key | `platform` | Yes | Yes | Yes |

**Key design intent:** the authorization decision path (`/evaluate`) is **service-key only**. A consumer service asks "can user U do action X?" using its *own* service key and names U as `actor.subjectId` in the request body. JWT pass-through is for end-users acting on their tenant's **management** surfaces, not for performing permission checks.

---

## JWT Pass-Through Sequence

The JWT pass-through flow lets a consumer service authenticate an end-user at the Bedrock sidecar without sharing browser sessions or issuing Bedrock-specific credentials to end-users.

```
Consumer service                    Bedrock sidecar                    Kinde JWKS
      │                                     │                               │
      │  1. Obtain user's Kinde             │                               │
      │     access token (your auth flow)   │                               │
      │                                     │                               │
      │  2. Forward to Bedrock:             │                               │
      │     Authorization: Bearer <token>   │                               │
      │     X-Tenant-Id: <tenantId>         │                               │
      │────────────────────────────────────>│                               │
      │                                     │  3. verifyToken(token):       │
      │                                     │     validate sig + iss +      │
      │                                     │     aud + exp (RS256 only)    │
      │                                     │────────────────────────────-->│
      │                                     │<──────────────────────────────│
      │                                     │                               │
      │                                     │  4. sub (kinde_id) →          │
      │                                     │     UsersService.findByKindeId│
      │                                     │     → Bedrock subject + userId│
      │                                     │                               │
      │                                     │  5. TenantsService             │
      │                                     │     .getTenantsForUser(userId)│
      │                                     │     validate X-Tenant-Id ∈    │
      │                                     │     user's memberships        │
      │                                     │                               │
      │                                     │  6. Authorize per guards      │
      │                                     │     (principal kind=user)     │
      │<────────────────────────────────────│                               │
```

Steps in plain terms:

1. **Consumer obtains the token.** Your existing Kinde auth flow issues the end-user an access token — forward it as-is.
2. **Consumer forwards the token.** Set `Authorization: Bearer <token>` and `X-Tenant-Id: <tenantId>` on the request to the sidecar.
3. **Sidecar validates the token** against Kinde's JWKS: checks the RS256 signature, `iss` (your Kinde domain), `aud` (must include `KINDE_AUDIENCE`), and `exp`. Any failure → **401**.
4. **Sidecar maps `sub` to a Bedrock identity.** The `sub` claim (Kinde user ID) is looked up in `UsersService`. If no Bedrock user exists for that `sub` → **401** (no auto-provisioning).
5. **Sidecar validates tenant membership.** The `X-Tenant-Id` header is checked against the user's tenant memberships (see [Tenant Selection](#tenant-selection)). Membership failure → **403**.
6. **Request authorized** per the standard guard chain with `kind=user` principal.

---

## Required JWT Claims

The Kinde access token forwarded to the sidecar must carry the following claims:

| Claim | Description |
|---|---|
| `sub` | Kinde user ID — mapped to a Bedrock subject via `UsersService`. |
| `iss` | Must equal your Kinde domain (e.g. `https://your-app.kinde.com`). |
| `aud` | Array of strings; must **include** the value of `KINDE_AUDIENCE`. |
| `exp` | Unix expiry timestamp; expired tokens are rejected. |

The sidecar pins the algorithm to RS256. Tokens with `alg: none` or HMAC algorithms are rejected.

---

## Tenant Selection

Tenant context for a JWT request is resolved as follows:

1. **`X-Tenant-Id` header present:** the value is validated against the user's tenant memberships. If the user is not a member of that tenant → **403**. If valid, that tenant is used for the request.
2. **`X-Tenant-Id` header omitted:** the sidecar falls back to the user's `defaultTenantId` (the same field used by the browser session path). If `defaultTenantId` is unset or the user no longer has a membership in it → **400**.

This design lets a user act in any tenant they belong to by supplying `X-Tenant-Id` explicitly, while omitting the header is safe and consistent for single-tenant users.

---

## Error Contract

| Condition | HTTP Status |
|---|---|
| No token in `Authorization: Bearer` header (and `KINDE_JWT_COOKIE` not set / cookie absent) | Falls through to the session path; **401** if session also absent |
| Invalid token (bad signature, wrong `iss`, wrong `aud`, unsupported algorithm) | **401** |
| Expired token (`exp` in the past) | **401** |
| Valid JWT but no Bedrock user found for `sub` (kinde_id) | **401** |
| `X-Tenant-Id` present but not in the user's tenant memberships | **403** |
| `X-Tenant-Id` omitted and `defaultTenantId` is unset or not an active membership | **400** |

**Convention:** the sidecar is fail-closed. A present-but-invalid token always produces an error — it does not silently fall through to the browser session path. Only a genuinely absent token falls through.

---

## Configuration

| Environment variable | Required | Description |
|---|---|---|
| `KINDE_AUDIENCE` | **Yes** | The API audience registered in your Kinde application (matched against the `aud` claim). |
| `KINDE_JWT_COOKIE` | No | If set, the sidecar also reads a raw Kinde JWT from the browser cookie with this name, in addition to the `Authorization: Bearer` header. Unset by default. |

> `KINDE_AUDIENCE` is required. The sidecar will reject all JWT requests if this variable is not set because the `aud` check cannot be performed.

---

## Request Examples

### Service api-key — authorization decision

```bash
curl -X POST 'https://your-bedrock-sidecar/evaluate' \
  -H 'x-api-key: sk_svc_your_service_key' \
  -H 'Content-Type: application/json' \
  -d '{
    "actor": { "subjectId": "subject_user_abc", "subjectType": "user" },
    "scopeId": "scope_project_crm_prod",
    "action": "read",
    "resource": { "resourceType": "customer", "resourceId": "cust_123" }
  }'
```

### User JWT — tenant-scoped management (read memberships)

```bash
curl -X GET 'https://your-bedrock-sidecar/management/memberships' \
  -H 'Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...' \
  -H 'X-Tenant-Id: tenant_acme'
```

### Platform api-key — control-plane operation

```bash
curl -X POST 'https://your-bedrock-sidecar/management/tenants' \
  -H 'x-api-key: pk_platform_your_platform_key' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "New Tenant",
    "slug": "new-tenant",
    "kind": "standard",
    "planKey": "pro",
    "maxUsers": 25
  }'
```

---

## What This Contract Does NOT Cover

- **OIDC token-exchange** (future) — the consumer forwards the end-user's token directly; no exchange for a Bedrock-issued token is performed in this version.
- **Auto-provisioning** — if the Kinde `sub` has no corresponding Bedrock user, the sidecar returns 401. Users must be provisioned via the management API before they can authenticate via JWT pass-through.
- **Kinde org-claim → tenant mapping** — tenant context always comes from `X-Tenant-Id` (or `defaultTenantId` fallback), never from Kinde organization claims.
- **`onBehalfOf` delegation** — `kind=user` principals cannot assert other subjects. Service-key callers can use `onBehalfOf` normally.

---

*This document is the input contract for W3.3 (SDK + API versioning). Changes to guard behavior or claim validation must be reflected here first.*
