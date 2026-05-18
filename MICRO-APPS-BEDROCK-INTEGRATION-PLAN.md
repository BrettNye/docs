# micro-apps ↔ Bedrock Integration Plan (Sidecar)

**Target consumer**: `My_Projects/micro-apps` (Nx monorepo: NestJS 11 API + Angular 20 portals + Drizzle/Postgres + Kinde + BullMQ/Redis)
**Integration mode**: Sidecar — bedrock runs as its own Render service (`bedrock-api` + `bedrock-db` + `bedrock-redis` per `render.yaml`); micro-apps' NestJS BFF calls it over HTTPS.
**Companion docs**:
- Bedrock-side readiness: [`apps/bedrock/api-management/src/management/ALPHA-RELEASE-CRITICAL.md`](../apps/bedrock/api-management/src/management/ALPHA-RELEASE-CRITICAL.md)
- Sidecar-specific deltas: [`apps/bedrock/api-management/src/management/SIDECAR-READINESS-DELTA.md`](../apps/bedrock/api-management/src/management/SIDECAR-READINESS-DELTA.md)
- In-process integration template: [`./RULEFORGE-BEDROCK-INTEGRATION-PLAN.md`](./RULEFORGE-BEDROCK-INTEGRATION-PLAN.md) — most phase structure is borrowed from here

---

## Why this plan is shorter than RULEFORGE-BEDROCK-INTEGRATION-PLAN.md

RuleFORGE chose **library mode** — bedrock runs in-process, same database, shared Drizzle. Most of its plan is about plumbing bedrock's modules into RuleFORGE's DI graph and backfilling resource IDs into existing tables.

micro-apps chose **sidecar mode**. That means:

- No shared database — bedrock owns `bedrock-db` on its own Postgres instance.
- No shared modules — micro-apps imports a thin HTTP client, not `BEDROCK_ENGINE` or `BEDROCK_STORE`.
- No backfill problem — bedrock manages its own scopes/subjects/resources via its REST API; micro-apps stores foreign-key-style references.
- New problems instead — network failures, token exchange, caching, SDK shape, SLA.

The 7 phases below map roughly to the RuleFORGE plan's 7 phases, but Phases 2 and 3 collapse, and a new Phase 0 (sidecar bootstrap) and Phase 4a (caching + failure modes) appear.

---

## Preconditions (bedrock-side, not micro-apps' problem to fix)

Before micro-apps can integrate, these must be done in `quarry-systems`:

1. **Wave 1 of ALPHA-RELEASE-CRITICAL.md** — `@Eval()` decorator, api-key guard fix, platform admin, tenant isolation.
2. **Wave 1 of SIDECAR-READINESS-DELTA.md** — healthcheck path fix (S1), helmet + trust proxy (S2), graceful shutdown (S3), admin UI gating (S9).
3. **Token-exchange decision** — pick an option from SIDECAR-READINESS-DELTA S5. This plan assumes **Option B (Kinde JWT pass-through)**.
4. **SDK exists** — `@quarry-systems/node-bedrock-sdk` published or path-aliased, per SIDECAR-READINESS-DELTA S4.

If any of these aren't done when micro-apps wants to ship, micro-apps either waits or implements the integration against a non-production-grade backend.

---

## Phase 0 — Sidecar bootstrap (½ day)

A one-time setup per environment.

### 0.1 Deploy bedrock to Render

Bedrock's `render.yaml` already declares the sidecar stack. Deploy as a separate Render Blueprint. Confirm:
- `bedrock-api` health check passes (post-S1 fix from SIDECAR-READINESS-DELTA)
- `bedrock-db` migrations applied (run `db:migrate` once after first deploy)
- Kinde M2M env vars populated: `KINDE_M2M_CLIENT_ID`, `KINDE_M2M_CLIENT_SECRET`, `KINDE_M2M_ISSUER`
- CORS origin pattern (`bedrock.quarry-systems.com` per `main.ts:20`) covers micro-apps' production hostnames; add micro-apps origins to `CORS_ORIGINS` env

### 0.2 Bootstrap the micro-apps tenant in bedrock

Adapt `apps/ruleforge/ruleforge-api/scripts/bootstrap-bedrock.ts` into `apps/api/scripts/bootstrap-bedrock.ts`:
- Create resource types per portal: `micro-apps:peptide:peptide`, `micro-apps:peptide:document`, `micro-apps:peptide:subscription`, etc.
- Create default roles per portal (Admin / Editor / Viewer / Subscriber)
- Define permission strings: `peptide:read`, `peptide:write`, `subscription:create`, etc.
- Create one tenant per micro-apps customer; one workspace per portal; one environment for prod
- Record returned `tenantId` / `workspaceId` / `environmentId` in micro-apps' `shared.tenants` table

### 0.3 Generate an admin API key

Use bedrock's `/api-keys` endpoint to mint one **service-level** API key scoped to the platform tenant. Store in micro-apps as `BEDROCK_SERVICE_API_KEY` (Render `sync: false`). This key authenticates the BFF when minting per-user tokens.

---

## Phase 1 — Wire the SDK into micro-apps (1-2 hours)

### 1.1 Install + create a Nest module

Per micro-apps' CLAUDE.md lib-layering rules, bedrock is **infra**, so it lives in `libs/api/bedrock-client/` (or `libs/api/bedrock/`).

```bash
pnpm exec nx g @nx/nest:library --name=api-bedrock-client \
  --directory=libs/api/bedrock-client \
  --importPath=@micro-apps/api/bedrock-client \
  --linter=eslint --unitTestRunner=jest --strict --no-interactive
```

Inside the lib:

```typescript
// libs/api/bedrock-client/src/lib/bedrock-client.module.ts
import { Module, DynamicModule, Global } from '@nestjs/common';
import { BedrockClient } from '@quarry-systems/node-bedrock-sdk';

@Global()
@Module({})
export class BedrockClientModule {
  static forRoot(): DynamicModule {
    return {
      module: BedrockClientModule,
      providers: [
        {
          provide: BedrockClient,
          useFactory: () => new BedrockClient({
            baseUrl: process.env['BEDROCK_API_URL']!,
            apiKey: process.env['BEDROCK_SERVICE_API_KEY']!,
            timeout: 5000,
            retries: 2,
          }),
        },
      ],
      exports: [BedrockClient],
    };
  }
}
```

Add to `apps/api/src/app/app.module.ts`:

```typescript
imports: [
  // ...
  BedrockClientModule.forRoot(),
]
```

### 1.2 Add env vars to micro-apps' `.env.example` and `render.yaml`

```bash
BEDROCK_API_URL=https://bedrock-api.onrender.com
BEDROCK_SERVICE_API_KEY=<from Phase 0.3>
```

---

## Phase 2 — Schema: store bedrock references (1-2 hours)

micro-apps does NOT replicate bedrock's tables. It stores **references** (`bedrock_resource_id`, `bedrock_scope_id`) on its own domain rows.

### 2.1 Extend `shared.tenants` with bedrock IDs

```typescript
// libs/api/db/src/lib/schemas/shared/tenants.ts
export const tenants = sharedSchema.table('tenants', {
  id: uuid('id').primaryKey().defaultRandom(),
  // ... existing fields
  bedrockTenantId: text('bedrock_tenant_id').notNull().unique(),
  bedrockEnvironmentScopeId: text('bedrock_environment_scope_id').notNull(),
});
```

### 2.2 Extend per-portal domain tables with `bedrock_resource_id`

Example for peptide-portal:

```typescript
// libs/api/db/src/lib/schemas/peptide_portal/peptides.ts
export const peptides = peptidePortalSchema.table('peptides', {
  id: uuid('id').primaryKey().defaultRandom(),
  // ... existing fields
  bedrockResourceId: text('bedrock_resource_id').notNull().unique(),
});
```

Same pattern for documents, subscriptions, etc.

### 2.3 Migration strategy

- New rows: create bedrock resource via SDK before insert; store returned ID.
- Existing rows: one-off backfill script (per portal) iterating existing rows and creating resources. Wrap in a transaction; if any single resource creation fails, roll back and retry.

---

## Phase 3 — Token exchange (½ day, assuming Option B from SIDECAR-READINESS-DELTA S5)

micro-apps' BFF holds the user's Kinde session cookie. When calling bedrock, the BFF forwards the user's Kinde JWT in `Authorization: Bearer <jwt>`. Bedrock validates against the Kinde JWKS and maps `kinde_id` → bedrock subject.

### 3.1 Extract Kinde JWT from session

In micro-apps' `libs/auth/backend`, expose a method that returns the current user's Kinde access token from the Redis-backed session.

### 3.2 Create a per-request bedrock client

The injected `BedrockClient` (Phase 1) authenticates with the SERVICE key. For per-user authz checks, you want the **user's** identity to show up in bedrock's audit log. Two patterns:

**Pattern A — clone the client per request:**
```typescript
const userClient = bedrockClient.withAuth({ bearer: kindeJwt });
const decision = await userClient.evaluate({ ... });
```

**Pattern B — pass the JWT explicitly per call:**
```typescript
const decision = await bedrockClient.evaluate({ ... }, { bearer: kindeJwt });
```

Pattern B is simpler if the SDK supports it; otherwise A.

### 3.3 Verify on bedrock side

Test: a request with a valid Kinde JWT but no matching bedrock subject should return a clear error (404 subject not found), not a 500. Bedrock should provision a subject lazily on first call OR reject explicitly — pick one.

---

## Phase 4 — Authorization in controllers (1 day)

Mirror the RuleFORGE pattern, but the engine call is HTTP instead of in-process.

### 4.1 BedrockAuthGuard

```typescript
// libs/api/bedrock-client/src/lib/bedrock-auth.guard.ts
@Injectable()
export class BedrockAuthGuard implements CanActivate {
  constructor(
    private bedrockClient: BedrockClient,
    private sessionService: SessionService,
  ) {}

  async canActivate(ctx: ExecutionContext): Promise<boolean> {
    const req = ctx.switchToHttp().getRequest();
    const session = await this.sessionService.fromRequest(req);
    if (!session) throw new UnauthorizedException();

    req.bedrockContext = {
      kindeJwt: session.kindeAccessToken,
      tenantId: session.bedrockTenantId,
      environmentScopeId: session.bedrockEnvironmentScopeId,
    };
    return true;
  }
}
```

### 4.2 An `@Authz()` decorator

Mirror bedrock's own `@Eval()` decorator (ALPHA-RELEASE-CRITICAL item #1). The guard:
1. Reads metadata from `@Authz({ action, resourceType, resourceId, scopeId })`
2. Resolves dynamic fields from the request
3. Calls `bedrockClient.evaluate(..., { bearer: req.bedrockContext.kindeJwt })`
4. Throws `ForbiddenException` if `!decision.allowed`

### 4.3 Apply to controllers

```typescript
@Controller('peptides')
@UseGuards(BedrockAuthGuard)
export class PeptidesController {
  @Get(':id')
  @Authz({ action: 'read', resourceType: 'peptide', resourceId: (req) => req.params.id })
  async get(@Param('id') id: string) { ... }
}
```

---

## Phase 4a — Caching and failure modes (½ day) — SIDECAR-ONLY

In-process consumers don't have this problem; sidecars do.

### 4a.1 Per-request cache

A given HTTP request might check 5+ permissions. Cache decisions in a request-scoped service so each unique `(subjectId, action, resourceId)` only hits bedrock once.

### 4a.2 Short-TTL Redis cache for hot decisions

For decisions that don't change often (e.g., "is this user a tenant admin?"), cache the boolean in micro-apps' Redis with a 30-60s TTL. Invalidate on bedrock's `role-assignment.changed` webhook (if/when bedrock implements webhooks per ALPHA-RELEASE-CRITICAL #15).

### 4a.3 Fail-closed defaults

If bedrock is unreachable, every authz check should fail closed (return 503, not allow the action). Wrap the SDK in a circuit breaker (e.g., `opossum`) that opens after N consecutive failures and returns 503 immediately during the cooldown.

### 4a.4 Observable

Emit metrics: `bedrock_client_request_duration_seconds`, `bedrock_client_errors_total`, `bedrock_client_cache_hits_total`. Add a `bedrock_circuit_open` gauge.

---

## Phase 5 — Audit logging on the micro-apps side (1-2 hours)

Bedrock already audits authz decisions. micro-apps should additionally audit business-side mutations (so e.g. "user updated peptide X" is captured even if the action didn't traverse `/evaluate`).

Reuse the audit-interceptor pattern from RULEFORGE-BEDROCK-INTEGRATION-PLAN Phase 5, but the interceptor calls bedrock's audit-log endpoint (if exposed per ALPHA-RELEASE-CRITICAL #9 follow-up work) or writes to micro-apps' own audit table.

Decision needed: **single audit trail in bedrock**, or **per-consumer audit trails with bedrock as the authz subset**? Recommendation: per-consumer, since bedrock's audit log is decision-only and consumers have more context.

---

## Phase 6 — Migrate existing customers to bedrock-managed tenants (per-portal, varies)

Out of scope for this plan since micro-apps doesn't currently have customers in production. When that changes:

1. Run Phase 0.2 (bootstrap) for each existing customer
2. Run Phase 2 backfill (assign `bedrock_resource_id` to existing rows)
3. Cut over the BFF to require `BedrockAuthGuard`
4. Keep the legacy auth path behind a feature flag for one release; remove after a soak period

---

## Phase 7 — Testing (1-2 days)

### 7.1 Unit tests (micro-apps)
- BedrockClientModule provider factory
- BedrockAuthGuard with mocked SDK
- @Authz decorator metadata resolution

### 7.2 Integration tests (micro-apps + real bedrock)
- Per-portal smoke test: BFF endpoint → guard → SDK → real bedrock → real DB → assert allowed/denied
- Run against a dedicated `bedrock-staging` environment, not prod

### 7.3 Failure-mode tests (sidecar-specific)
- Bedrock returns 5xx — micro-apps fails closed
- Bedrock times out — circuit breaker opens
- Kinde JWT expired during call — micro-apps returns 401 with refresh hint
- Bedrock CORS rejects micro-apps origin (use the wrong `CORS_ORIGINS`) — clear error, not silent fail

### 7.4 Tenant isolation E2E (mirror ALPHA-RELEASE-CRITICAL #6)
- Two customers, identical resource names
- User in customer A can't read customer B's peptides via micro-apps API

---

## Timeline summary

| Phase | Effort | Depends on |
|---|---|---|
| Preconditions (bedrock-side) | — | (separate work tracked in ALPHA-RELEASE-CRITICAL + SIDECAR-READINESS-DELTA) |
| 0. Sidecar bootstrap | ½ day | bedrock deployed |
| 1. Wire SDK | 1-2 hours | SDK published |
| 2. Schema refs | 1-2 hours | Phase 0 |
| 3. Token exchange | ½ day | Phase 1, S5 decision |
| 4. Authz in controllers | 1 day | Phase 3 |
| 4a. Caching + failure modes | ½ day | Phase 4 |
| 5. Audit logging | 1-2 hours | Phase 4 |
| 7. Testing | 1-2 days | All |

**Total micro-apps-side effort**: ~3-5 days.
**Plus bedrock-side preconditions**: see ALPHA-RELEASE-CRITICAL (17-21 days blocking) + SIDECAR-READINESS-DELTA (~1 day same-day fixes + 1-2 weeks pre-launch).

---

## Open decisions

1. **Token-exchange option** (A/B/C from SIDECAR-READINESS-DELTA S5) — this plan assumes B.
2. **Audit trail**: single in bedrock, or split between bedrock (decisions) + per-consumer (mutations)? Recommendation: split.
3. **Resource-ID strategy**: store bedrock's UUID on every domain row (Phase 2 approach), or compute deterministically from (tenant, type, business-key)? Latter is simpler but couples micro-apps to bedrock's ID format.
4. **Per-portal tenants vs per-customer tenants**: who maps to a bedrock tenant — each portal, each customer, or each portal×customer combo? Default in this plan: each customer is a bedrock tenant; each portal is a workspace inside it.

These should be settled before Phase 0.
