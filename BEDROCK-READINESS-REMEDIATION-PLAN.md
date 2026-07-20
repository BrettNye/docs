# Bedrock Readiness Remediation Plan

**Author:** deep-audit (Claude) · **Date:** 2026-06-05 · **Goal:** make `libs/bedrock` + `apps/bedrock` ready for **embedded (in-process library)** and **sidecar HTTP service** consumption.

This plan covers the **structural** findings from the 2026-06-05 deep audit — the ones that need test-first engineering, a design decision, or a cross-cutting rollout. Mechanical findings (package deps, contract drift, doc corrections, sidecar infra hardening) are being fixed directly outside this plan.

Severity legend: 🔴 blocker · 🟠 high · 🟡 medium. Each item notes which consumption mode it gates (E = embedded, S = sidecar).

---

## Wave 0 — Engine test harness (do first; everything below depends on it)

### W0.1 — Author `engine.spec.ts` + `collection-matcher.spec.ts` 🔴 (E, S)
**Finding #6.** The authorization engine (`bedrock-core/src/lib/engine.ts`, ~1100 LoC) and `collection-matcher.ts` have **zero tests**, yet they decide every allow/deny. 32 distinct evaluation paths are unverified. A single wildcard/priority/condition bug = silent wrong-allow.

**Why first:** Waves 1–2 change engine semantics. Changing an untested authorization engine is unacceptable. Establish the safety net before touching evaluation logic.

**Approach (TDD):**
- Use `InMemoryBedrockStore` as the fixture (already exists, easy to seed).
- Cover, at minimum, the ranked top-10 risk paths: wildcard action, wildcard resourceType, JSON Logic condition pass/fail/error, pattern exact/wildcard, resource-policy allow/deny, delegation both-pass + each-leg-fail, hierarchy cascade=inherit vs none, priority ordering, collection matching (all 8 matcher modes).
- Add the **missing/edge** denial paths: subject-not-found, no-membership, no-role-assignment, no-role-permission, no-candidate-permission.
- Each test asserts the **decision AND the reason** (drives W1.5 / decision-reason work).

**Files:** new `bedrock-core/src/lib/engine.spec.ts`, `collection-matcher.spec.ts`. **Effort:** 2–3 days. **Deps:** none.

---

## Wave 1 — Concept-correctness blockers (engine; strictly test-first on W0)

### W1.1 — Apply scope overrides in the engine 🔴 (E, S)
**Finding #1.** Scope role / permission / role-permission overrides exist end-to-end (schema, store methods, repositories, API controllers) but `evaluate()` **never loads or applies them**. Overrides created via the API silently do nothing — a correctness hole and a latent security surprise (an operator "disables a role in prod" and it still works).

**Approach:** In `evaluateForSubject`, after loading memberships/role-assignments/role-permissions, load `getScopeRoleOverrides` / `getScopePermissionOverrides` / `getScopeRolePermissionOverrides` for the evaluation scope and filter/disable candidates accordingly. Define precedence vs conditions explicitly. Write failing tests first (override disables an otherwise-granted permission).

**Effort:** 1–2 days. **Deps:** W0.

### W1.2 — Fix delegation to evaluate resource policies for the principal 🔴 (E, S)
**Finding #2.** With `onBehalfOf`, resource policies are evaluated against the **actor only**; on policy short-circuit the principal is never checked. An explicit deny on the principal is ignored — breaks the documented both-must-pass rule.

**Approach:** When `onBehalfOf` is present, run the resource-policy stage for both actor and principal and AND the results; a deny on either leg denies. Add tests: principal-denied-by-policy + actor-allowed ⇒ deny.

**Effort:** 0.5–1 day. **Deps:** W0.

### W1.3 — Decide & implement deny-precedence at equal priority 🟠 (E, S)
**Finding #8.** Resource-policy conflicts at equal priority resolve by first-match after a stable sort (direct policies before collection policies) — effectively non-deterministic from the author's POV. Standard authz expectation is **deny-wins**.

**Decision needed (you):** deny-wins at equal priority (recommended) vs explicit priority-only. Then implement two-pass (scan denies first) or document the tie-break. Tests for equal-priority allow+deny.

**Effort:** 0.5 day + decision. **Deps:** W0.

### W1.4 — Resolve resource scope links: evaluate or document as metadata 🟠 (E, S)
**Finding #18.** `share`/`alias`/`mirror` resource-scope links have full CRUD + storage but are **never consulted during evaluation**. Either they're an unfinished feature or metadata-only.

**Decision needed (you):** (a) wire links into evaluation (a resource is reachable from linked scopes per link type), or (b) declare metadata-only and document it. Don't ship an authz concept that looks load-bearing but isn't.

**Effort:** 0.5 day (document) to 2 days (evaluate). **Deps:** W0, and the decision.

### W1.5 — Condition error visibility + shared evaluator 🟠 (E, S)
**Findings #7, #20.** JSON Logic errors are swallowed in 4 duplicated try/catch sites — fail-closed but invisible, so a malformed condition silently denies with no signal.

**Approach:** Extract one `evaluateCondition(condition, ctx)` helper returning `{ ok, result, error }`. Add an optional injectable logger / diagnostics sink on `BedrockEngine` (no hard dependency — keep core portable for embedding). Surface condition-error vs condition-false distinctly in the decision (feeds W4.3). Replace all 4 call sites.

**Effort:** 1 day. **Deps:** W0.

---

## Wave 2 — Authorization & tenant isolation (sidecar-gating; also good hygiene for embedded)

### W2.1 — Authz on the core controllers 🔴 (S)
**Finding #4.** All 20 core controllers (roles, permissions, scopes, resources, tags, subjects, memberships, overrides) have **no authn and no authz**. Any caller can `DELETE /roles/:id` or mint backdoor permissions in any tenant's scope. The management layer already proves the pattern (`@Authorized()` + `EvalGuard`, 10/11 controllers) — core is 0/20.

**Approach:** Apply `@Authenticated()` + `@Authorized()` to every core endpoint, with a scopeId resolver per resource type (resolve the owning scope from the entity, then evaluate `action`/`resourceType` against the caller). This is dogfooding — bedrock authorizing bedrock. Roll out controller-by-controller; add an integration test per controller asserting cross-tenant access is denied.

**Effort:** 2–3 days. **Deps:** the EvalGuard pattern (exists). Independent of Wave 1, but W1.1 (overrides) should land first so self-authz respects overrides.

### W2.2 — Lock down `/evaluate` and `/effective-permissions` 🔴 (S)
**Finding #3.** Both are completely unauthenticated — anonymous permission probing, model enumeration, free `onBehalfOf`.

**Approach:** Require `@Authenticated()`; apply per-caller rate limiting; gate `onBehalfOf` (see W2.3). Decide whether `/evaluate` is service-key-only or any authenticated subject. `/effective-permissions` must check the caller may query that subject.

**Effort:** 0.5 day + policy decision. **Deps:** token-exchange decision (W3.2).

### W2.3 — Delegation authorization model 🟠 (S)
**Finding #16.** Any actor may claim `onBehalfOf` any principal; there's no delegation grant. Combined with W2.2 this is impersonation.

**Approach:** Introduce a delegation primitive — either a `subject:delegate` permission checked against the principal's scope, or an explicit delegation-grant table (actor→principal, optional expiry). Engine checks it before evaluating the principal leg.

**Effort:** 1–2 days. **Deps:** W1.2.

### W2.4 — Authz on api-keys & invites controllers 🟠 (S)
**Finding #15.** Authenticated-but-unauthorized: a tenant's user can list/revoke/delete another tenant's API keys and invites.

**Approach:** Add `@Authorized()` with tenant-scoped resolver. **Effort:** 0.5 day. **Deps:** W2.1 pattern.

---

## Wave 3 — Deployment & migrations (sidecar-gating)

### W3.1 — Author the missing migrations 🔴 (S, and E-on-Postgres)
**Finding #5.** Migrations don't match the schema: `0002_phase1_condition_migration.sql` is empty; no `scope_id → owner_scope_id` rename; `resource_scope_links`, `resource_collections`, `resource_policies` tables exist in code but in **no** migration; snapshots stale. A fresh deploy or an existing DB both break. (Repositories themselves are fully synced — the drift is migrations-only.)

**Approach:** Regenerate via `drizzle-kit generate` from current schema, then **hand-review** the diff (especially the column rename — must be `ALTER ... RENAME`, not drop/add, to preserve data) before committing. Add the rename as an explicit migration so existing deployments survive. Regenerate snapshots. Do **not** run destructive SQL against any live DB without a backup/dry-run. Verify against a scratch Postgres.

**Effort:** 1 day. **Deps:** none, but coordinate with publish (don't tag a release with broken migrations).

### W3.2 — Token-exchange / sidecar auth contract decision 🟠 (S)
**Finding #25 (S5).** Consumer↔sidecar auth is undecided (Kinde JWT pass-through vs service API key vs OIDC exchange). Blocks W2.2 policy and the SDK.

**Decision needed (you):** recommended = JWT pass-through (sidecar validates against Kinde JWKS, preserves per-user audit). Document the contract. **Effort:** 0.5 day decision + 0.5–1 day impl.

### W3.3 — Consumer SDK + API versioning 🟠 (S)
**Findings #25 (S4, S7).** No `@quarry-systems/node-bedrock-sdk`; routes unversioned. Sidecar consumers hand-roll HTTP clients against an unversioned surface.

**Approach:** Add `app.enableVersioning()` + `/v1` prefix; scaffold a thin typed SDK (`evaluate`, `effectivePermissions`, management CRUD) reusing `bedrock-contracts` types. **Effort:** 1–2 days. **Deps:** W3.2, route surface stable.

---

## Wave 4 — Hardening & parity (quality; do before GA, not blocking alpha)

### W4.1 — Store parity: InMemory vs Postgres 🟠 (E)
**Finding #9.** `getSubjectsByIds`/`getResourcesByIds` differ on partial-missing-id (null vs partial results) and empty-array handling. Embedded consumers test on InMemory, run Postgres in prod — divergence ships as a prod-only bug.

**Approach:** Pick one contract (recommend: return found subset, never null for partial), document it on the `BedrockStore` interface, align both stores, add a **shared parity test suite** run against both implementations. **Effort:** 1 day. **Deps:** W0 infra.

### W4.2 — JSON Logic write-time validation 🟠 (S)
**Finding #19.** `logicField = z.record(z.unknown())` — conditions are never validated on write (API or policy-loader). Broken rules only surface as silent denies at eval time.

**Approach:** Validate parseability / known operators on create/update of role-permission links, resource policies, and in `bedrock-policy-loader`. **Effort:** 0.5–1 day.

### W4.3 — Structured decision reasons 🟡 (E, S)
**Finding #21.** `BedrockDecision` exposes only `allowed` + a human string. Audit/debug requires string parsing. Add `denialReason` enum + per-leg (actor/principal) sub-decisions. **Effort:** 0.5 day. **Deps:** W1.5.

### W4.4 — Deny-by-condition on role-permission links 🟡 (E, S)
**Finding #24.** Role-permission conditions can only grant, not deny — asymmetric with resource policies. Decide whether to add a deny mode. **Effort:** 0.5–1 day + decision.

### W4.5 — RuleForge embedded integration completion 🟠 (E)
**Finding #17.** `RULEFORGE-BEDROCK-INTEGRATION-STATUS.md` claims Phase 1 complete, but `BedrockModule` is **not imported** into the RuleForge `app.module.ts` and the bootstrap script never runs. (Different app; tracked here for the embedded story.) Correct the status doc, then actually wire it. **Effort:** per RuleForge plan.

---

## Decisions needed from you (gating)
1. **W1.3** deny-wins at equal priority? (recommend yes)
2. **W1.4** resource scope links — evaluate, or metadata-only?
3. **W2.2** `/evaluate` — service-key-only or any authenticated subject?
4. **W3.2** sidecar auth — JWT pass-through (recommend), service key, or OIDC exchange?
5. **W4.4** add deny-by-condition to role-permission links?

## Suggested sequencing
W0 → (W1 + W3.1 in parallel) → W2 → W3.2/3.3 → W4. Embedded readiness is reachable after **W0, W1, W3.1, W4.1** (engine correct + tested + migrations + store parity). Sidecar readiness additionally requires **W2 + W3.2/3.3** and the mechanical infra fixes (helmet, trust proxy, health path, graceful shutdown, Swagger gating).

---

## Appendix — Test-suite validation state & known conditions

_Last updated 2026-07-13. Vault system-of-record: the `bedrock` wiki pickup synthesis. This appendix is the version-controlled mirror._

### How to run the suites (nx is broken in the working tree)

Sibling git worktrees create duplicate nx project names, so **never `nx test`** — run the runners directly:

- Engine libs (bedrock-contracts / bedrock-core / bedrock-core-storage) = **Vitest**: `cd libs/bedrock/<lib> && npx vitest run --config vite.config.mts [<spec>]`
- `apps/bedrock/api-management` = **Jest**: `npx jest --config apps/bedrock/api-management/jest.config.ts <spec>`

### Known pre-existing failures (NOT regressions)

- **`app.controller.spec` / `app.service.spec`** — fail DI (missing `BEDROCK_CORE_ENGINE` provider); this blocks a full booted HTTP e2e. Controller authz is instead verified via guard-metadata reflection (e.g. `core-platform-only.integration.spec.ts`) and versioning via a throwaway-controller mechanism test.

### Environment condition (not a test failure)

- **`pnpm install` currently fails** in this tree: the in-flight publish-prep WIP rewrites internal package deps to hard `"0.0.1"` versions that aren't published, so pnpm 404s fetching `@quarry-systems/*`. Tests and `tsc` still run because `@quarry-systems/*` imports resolve via `tsconfig.base.json` path mappings. `helmet` is declared + committed but absent from `node_modules` for the same reason.

### Resolved conditions (formerly undocumented)

- The `helmet` import in `apps/bedrock/api-management/src/main.ts` is **committed** (W3.2 hardening), no longer an "uncommitted import error."
- `engine.adversarial.spec.ts` delegation *"both legs allowed via different roles"* was a **stale test**, not an engine bug — see the lesson below. **Fixed**, and the whole class of fail-closed-masked delegation tests hardened.

### Process lesson — fail-closed / precondition gates and test coverage

When a commit adds a **fail-closed or precondition gate** to an evaluation path (e.g. W2.3's `enforce delegation grants fail-closed` — no `actor→principal` delegation grant ⇒ deny *before* evaluating either leg), it can:

1. **Ship a sibling spec red** — the change updated its primary spec (`engine.spec.ts`) with the new precondition (seed a grant) but missed a duplicate scenario in `engine.adversarial.spec.ts`, which then fail-closed to the opposite verdict.
2. **Silently mask coverage** — every *deny*-expecting test that omits the new precondition now denies *at the gate* rather than via its stated mechanism (RBAC leg, principal deny-policy, no-match-leak), so it keeps passing while no longer testing what it claims.

**Rule:** when adding such a gate, `grep` **all** spec files (not just the primary suite) for the old-model scenario, update every one to satisfy the new precondition, and add an **explicit backstop test for the gate itself** (precondition absent ⇒ deny even when the legs would pass) so the new behavior is covered on purpose, not incidentally.
