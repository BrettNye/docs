# Bedrock Master Implementation Plan

This document consolidates all pending feature implementations into a single phased execution plan.

---

## Implementation Guidelines

All implementations MUST follow these principles:

### Code Quality Principles

| Principle | Description |
|-----------|-------------|
| **DRY (Don't Repeat Yourself)** | Extract shared logic into reusable utilities, base classes, or shared modules. Never duplicate code across files. |
| **Single Responsibility** | Each module, class, or function should have one clear purpose. Split complex logic into focused, composable units. |
| **Code Reuse** | Leverage existing utilities, helpers, and patterns. Check `libs/` for shared code before creating new implementations. |
| **Performance** | Write efficient queries, avoid N+1 problems, use batch operations where possible, and consider caching strategies. |
| **Effectiveness** | Code should be clear, maintainable, and solve the actual problem without over-engineering. |

### Consistency Requirements

| Area | Requirement |
|------|-------------|
| **Models/Interfaces** | Follow patterns in `libs/bedrock/bedrock-core/src/lib/interfaces/` |
| **Database Schemas** | Follow patterns in `libs/bedrock/bedrock-core-storage/src/lib/postgres/schemas/` |
| **Zod Contracts** | Follow patterns in `libs/bedrock/bedrock-contracts/src/lib/bedrock/` |
| **Repositories** | Follow patterns in `apps/bedrock/api-management/src/core/repositories/services/` |
| **Services** | Follow patterns in `apps/bedrock/api-management/src/core/services/` |
| **Controllers** | Follow patterns in `apps/bedrock/api-management/src/core/controllers/` |
| **Store Methods** | Follow patterns in `libs/bedrock/bedrock-core-storage/src/lib/postgres/postgres-store.ts` |

### Checklist for Each Feature

Before marking a feature complete, verify:

- [ ] Follows existing naming conventions (camelCase for fields, PascalCase for types)
- [ ] Uses existing shared utilities (e.g., `generateId()`, standard field schemas)
- [ ] Implements proper TypeScript types with no `any` usage
- [ ] Includes proper null/undefined handling
- [ ] Updates ALL affected layers (schema → contracts → store → repository → service → controller)
- [ ] Mapper functions follow existing patterns
- [ ] Error handling is consistent with existing code
- [ ] No duplicate code - extracted to shared modules if needed

---

## Features Overview

| Feature | Description | Priority |
|---------|-------------|----------|
| **Condition Migration** | Move `logic` from Permission to RolePermission edge | High |
| **scopeId → ownerScopeId** | Rename resource ownership field | High |
| **Resource Scope Links** | Replace ResourceScope with enhanced link model | Medium |
| **Resource Hierarchy Cascade** | Permission inheritance through resource trees | Medium |
| **Resource Collections** | Dynamic query-based resource groupings | Medium |
| **Resource Policies** | Direct allow/deny rules on resources/collections | Medium |

---

## Phase 1: Foundation Changes (Breaking Changes)

These changes affect core models and should be done first.

### 1.1 Condition Migration

**Goal:** Move `logic` from `BedrockPermission` to `BedrockRolePermission.condition` and `BedrockScopeRolePermissionOverride.condition`

#### Database Schema

| File | Change |
|------|--------|
| `core.schema.ts` | Add `condition: jsonb()` to `rolePermissions` table |
| `core.schema.ts` | Add `condition: jsonb()` to `scopeRolePermissionOverrides` table |
| `core.schema.ts` | Remove `logic` from `permissions` table |

```typescript
// rolePermissions - ADD
condition: jsonb('condition').$type<Record<string, unknown>>(),

// scopeRolePermissionOverrides - ADD
condition: jsonb('condition').$type<Record<string, unknown>>(),
```

#### Zod Contracts

| File | Change |
|------|--------|
| `role-permission.schema.ts` | Add `condition: logicField.optional()` |
| `overrides.schema.ts` | Add `condition: logicField.optional()` to create/update schemas |

#### Repositories

| File | Change |
|------|--------|
| `role-permission.repository.ts` | Handle `condition` in create/createBatch/map |
| `scope-role-permission-override.repository.ts` | Handle `condition` in create/createBatch/update/map |

#### Engine

| File | Change |
|------|--------|
| `engine.ts` | Evaluate `rolePermission.condition` instead of `permission.logic` |

#### Tasks

- [x] Add `condition` column to `rolePermissions` table
- [x] Add `condition` column to `scopeRolePermissionOverrides` table
- [x] Update `createRolePermissionSchema`
- [x] Update `createScopeRolePermissionOverrideSchema`
- [x] Update `updateScopeRolePermissionOverrideSchema`
- [x] Update `RolePermissionRepository`
- [x] Update `ScopeRolePermissionOverrideRepository`
- [x] Update `BedrockEngine.evaluateForSubject()`
- [x] Update `postgres-store.ts`
- [x] Update `in-memory-store.ts`
- [x] Remove `logic` from `permissions` table
- [x] Remove `logic` from permission Zod contracts
- [ ] Run database migration

> **⚠️ IMPORTANT:** Repositories in `apps/bedrock/api-management/src/core/repositories/services/` must be updated to handle the new `condition` field.

---

### 1.2 Rename scopeId → ownerScopeId

**Goal:** Clarify that `BedrockResource.scopeId` represents ownership, not just association

#### Database Schema

```typescript
// resources table
ownerScopeId: text('owner_scope_id').notNull().references(() => scopes.id),  // RENAMED
```

#### Migration SQL

```sql
ALTER TABLE resources RENAME COLUMN scope_id TO owner_scope_id;
ALTER INDEX idx_resources_scope RENAME TO idx_resources_owner_scope;
```

#### Tasks

- [x] Rename column in `resource.schema.ts`
- [x] Update `createResourceSchema` in contracts
- [x] Update `resource.repository.ts`
- [x] Update `postgres-store.ts`
- [x] Update `in-memory-store.ts`
- [x] Update `engine.ts` resource context
- [ ] Create migration script
- [ ] Update tests

---

## Phase 2: Resource Scope Links

**Goal:** Replace `BedrockResourceScope` with `BedrockResourceScopeLink` adding `linkType` and `metadata`

### New Type

```typescript
export interface BedrockResourceScopeLink {
    id: string;
    resourceId: string;
    scopeId: string;
    linkType?: "share" | "alias" | "mirror";
    metadata?: Record<string, any>;
    createdAt: string;
    createdBy: string;
}
```

### Database Schema

```typescript
export const linkTypeEnum = pgEnum('link_type', ['share', 'alias', 'mirror']);

export const resourceScopeLinks = pgTable('resource_scope_links', {
    id: text('id').primaryKey(),
    resourceId: text('resource_id').notNull().references(() => resources.id),
    scopeId: text('scope_id').notNull().references(() => scopes.id),
    linkType: linkTypeEnum('link_type'),
    metadata: jsonb('metadata').$type<Record<string, unknown>>(),
    createdAt: text('created_at').notNull(),
    createdBy: text('created_by').notNull(),
}, (t) => [
    uniqueIndex('uq_resource_scope_links').on(t.resourceId, t.scopeId),
    index('idx_resource_scope_links_resource').on(t.resourceId),
    index('idx_resource_scope_links_scope').on(t.scopeId),
]);
```

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/resource-scope-links?resourceId=` | Get links for resource |
| GET | `/resource-scope-links?scopeId=` | Get links for scope |
| GET | `/resource-scope-links/:id` | Get link by ID |
| POST | `/resource-scope-links` | Create link |
| PATCH | `/resource-scope-links/:id` | Update link |
| DELETE | `/resource-scope-links/:id` | Delete link |

### Tasks

- [x] Create `linkTypeEnum` in schema
- [x] Create `resourceScopeLinks` table
- [x] Create `resource-scope-link.schema.ts` Zod contracts
- [x] Export from `bedrock-contracts` index
- [x] Add store interface methods
- [x] Implement in `postgres-store.ts`
- [x] Implement in `in-memory-store.ts`
- [x] Create `ResourceScopeLinkRepository`
- [x] Create `ResourceScopeLinkService`
- [x] Create `ResourceScopeLinkController`
- [x] Add to resources module
- [ ] Update documentation

> **⚠️ IMPORTANT:** When implementing new features, always check and update the corresponding repositories in `apps/bedrock/api-management/src/core/repositories/services/`. Repositories handle the actual database operations and must be kept in sync with schema changes.

---

## Phase 3: Resource Hierarchy Cascade

**Goal:** Add `cascade` field to control permission inheritance through resource hierarchies

### Updated Type

```typescript
export interface BedrockResourceHierarchy {
    parentResourceId: string;
    childResourceId: string;
    relationshipType?: string;
    cascade?: "inherit" | "none";  // NEW
    createdAt: string;
}
```

### Database Schema

```typescript
export const cascadeModeEnum = pgEnum('cascade_mode', ['inherit', 'none']);

// Add to resourceHierarchyEdges
cascade: cascadeModeEnum('cascade'),
```

### Engine Integration

```typescript
// In evaluate(), after resource policies, before role-based:
if (resourceContext.resource) {
    const hierarchyDecision = await this.evaluateWithHierarchy({
        resource: resourceContext.resource,
        action,
        subjectId: actor.subjectId,
        scopeId,
        context: resourceContext.context,
    });
    if (hierarchyDecision?.allowed) {
        return hierarchyDecision;
    }
}
```

### Store Interface Additions

```typescript
getParentResourceEdges(childResourceId: string): Promise<BedrockResourceHierarchy[]>;
getChildResourceEdges(parentResourceId: string): Promise<BedrockResourceHierarchy[]>;
getResourceAncestors(resourceId: string, cascadeOnly?: boolean): Promise<BedrockResource[]>;
```

### Tasks

- [x] Create `cascadeModeEnum` in schema
- [x] Add `cascade` column to `resourceHierarchyEdges`
- [x] Update `createResourceHierarchyEdgeSchema`
- [x] Create `updateResourceHierarchyEdgeSchema`
- [x] Add store methods for hierarchy traversal
- [x] Implement in `postgres-store.ts`
- [x] Implement in `in-memory-store.ts`
- [x] Update `ResourceHierarchyRepository`
- [x] Add PATCH endpoint for hierarchy edges
- [x] Update `BedrockEngine` with hierarchy evaluation
- [x] Add `inheritedFrom` to `BedrockDecision`
- [ ] Update documentation

> **⚠️ IMPORTANT:** Repositories in `apps/bedrock/api-management/src/core/repositories/services/` must be updated to handle the new `cascade` field.

---

## Phase 4: Resource Collections

**Goal:** Dynamic query-based groupings of resources

### Types

```typescript
export interface BedrockResourceCollection {
    id: string;
    name: string;
    scopeId: string;
    resourceTypeId: string;
    match: ResourceMatchDefinition;
    createdAt: string;
    createdBy: string;
}

export interface ResourceMatchDefinition {
    fields?: Record<string, PrimitiveValue | FieldMatchRule>;
    tags?: Record<string, string | string[]>;
    patterns?: Record<string, string>;
    time?: Record<string, TimeMatchRule>;
    any?: ResourceMatchDefinition[];
    all?: ResourceMatchDefinition[];
    none?: ResourceMatchDefinition[];
    condition?: JsonLogicExpr;
}
```

### Database Schema

```typescript
export const resourceCollections = pgTable('resource_collections', {
    id: text('id').primaryKey(),
    name: text('name').notNull(),
    scopeId: text('scope_id').notNull().references(() => scopes.id),
    resourceTypeId: text('resource_type_id').notNull().references(() => resourceTypes.id),
    match: jsonb('match').notNull().$type<ResourceMatchDefinition>(),
    createdAt: text('created_at').notNull(),
    createdBy: text('created_by').notNull(),
}, (t) => [
    index('idx_resource_collections_scope').on(t.scopeId),
    index('idx_resource_collections_type').on(t.resourceTypeId),
]);
```

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/resource-collections?scopeId=` | Get collections for scope |
| GET | `/resource-collections/:id` | Get collection by ID |
| GET | `/resource-collections/:id/resources` | Get matching resources |
| POST | `/resource-collections` | Create collection |
| PATCH | `/resource-collections/:id` | Update collection |
| DELETE | `/resource-collections/:id` | Delete collection |
| POST | `/resource-collections/:id/test` | Test if resource matches |

### Collection Matcher

Create `libs/bedrock/bedrock-core/src/lib/collection-matcher.ts`:

```typescript
export class CollectionMatcher {
    matches(
        resource: BedrockResource,
        resourceData: Record<string, unknown>,
        tags: BedrockTag[],
        match: ResourceMatchDefinition
    ): boolean {
        // Field matching, tag matching, time matching, patterns
        // Combinators: any, all, none
        // JSON Logic escape hatch
    }
}
```

### Tasks

- [x] Add `resourceCollections` table to schema
- [x] Create `resource-collection.schema.ts` Zod contracts
- [x] Export from `bedrock-contracts` index
- [x] Add store interface methods
- [x] Implement in `postgres-store.ts`
- [x] Implement in `in-memory-store.ts`
- [x] Create `CollectionMatcher` engine
- [x] Create `ResourceCollectionRepository`
- [x] Create `ResourceCollectionService`
- [x] Create `ResourceCollectionController`
- [x] Add to resources module
- [ ] Update documentation

---

## Phase 5: Resource Policies

**Goal:** Direct allow/deny rules on resources or collections

### Types

```typescript
type ResourcePolicyTarget =
    | { kind: "resource"; resourceId: string }
    | { kind: "collection"; collectionId: string };

export interface BedrockResourcePolicy {
    id: string;
    scopeId: string;
    target: ResourcePolicyTarget;
    actions: string[];
    effect: "allow" | "deny";
    subjectCondition?: JsonLogicExpr;
    contextCondition?: JsonLogicExpr;
    priority?: number;
    createdAt: string;
    createdBy: string;
}
```

### Database Schema

```typescript
export const policyTargetKindEnum = pgEnum('policy_target_kind', ['resource', 'collection']);
export const policyEffectEnum = pgEnum('policy_effect', ['allow', 'deny']);

export const resourcePolicies = pgTable('resource_policies', {
    id: text('id').primaryKey(),
    scopeId: text('scope_id').notNull().references(() => scopes.id),
    targetKind: policyTargetKindEnum('target_kind').notNull(),
    targetId: text('target_id').notNull(),
    actions: jsonb('actions').notNull().$type<string[]>(),
    effect: policyEffectEnum('effect').notNull(),
    subjectCondition: jsonb('subject_condition').$type<Record<string, unknown>>(),
    contextCondition: jsonb('context_condition').$type<Record<string, unknown>>(),
    priority: integer('priority').notNull().default(0),
    createdAt: text('created_at').notNull(),
    createdBy: text('created_by').notNull(),
}, (t) => [
    index('idx_resource_policies_scope').on(t.scopeId),
    index('idx_resource_policies_target').on(t.targetKind, t.targetId),
]);
```

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/resource-policies?scopeId=` | Get policies for scope |
| GET | `/resource-policies/:id` | Get policy by ID |
| GET | `/resource-policies/for-resource/:resourceId` | Get policies for resource |
| GET | `/resource-policies/for-collection/:collectionId` | Get policies for collection |
| POST | `/resource-policies` | Create policy |
| PATCH | `/resource-policies/:id` | Update policy |
| DELETE | `/resource-policies/:id` | Delete policy |

### Engine Integration

Update evaluation order:

```
1. Resource Policies (explicit allow/deny)
   ├── Direct resource policies (sorted by priority)
   └── Collection policies (sorted by priority)

2. Resource Hierarchy Inheritance (if cascade=inherit)
   └── Check parent resource permissions recursively

3. Role-Based Permissions
   ├── Role-permission conditions
   └── Scope overrides with conditions
```

### Tasks

- [x] Create enums in schema
- [x] Add `resourcePolicies` table
- [x] Create `resource-policy.schema.ts` Zod contracts
- [x] Export from `bedrock-contracts` index
- [x] Add store interface methods
- [x] Implement in `postgres-store.ts`
- [x] Implement in `in-memory-store.ts`
- [x] Create `ResourcePolicyRepository`
- [x] Create `ResourcePolicyService`
- [x] Create `ResourcePolicyController`
- [x] Add to resources module
- [x] Update `BedrockEngine.evaluate()` to check policies first
- [x] Add `evaluatedPolicy` to `BedrockDecision`
- [ ] Update documentation

---

## Phase 6: Documentation & Cleanup

### Remove Deprecated Code

- [x] Remove old `resourceScopes` table and all references
- [x] Remove `logic` field from all permission-related code (already removed)
- [x] Clean up any backward compatibility shims

### Documentation Updates

| Document | Updates | Status |
|----------|---------|--------|
| `Chatgpt-overview-suggestion.md` | Updated conditional permissions, added collections/policies sections | ✅ Done |
| `bedrock-core/README.md` | Added feature documentation, types, enums | ✅ Done |
| `docs/bedrock-api.md` | Added Resource Collections, Policies, Scope Links, Authorization Engine | ✅ Done |

### Testing

- [ ] Unit tests for `CollectionMatcher`
- [ ] Unit tests for policy evaluation
- [ ] Unit tests for hierarchy cascade
- [ ] Integration tests for new endpoints
- [ ] E2E tests for evaluation scenarios
- [ ] Migration tests

---

## Execution Order Summary

```
Phase 1.1: Condition Migration          [Week 1]
    └── DB + Contracts + Repos + Engine

Phase 1.2: scopeId → ownerScopeId       [Week 1]
    └── DB + Contracts + Repos + Stores

Phase 2: Resource Scope Links           [Week 2]
    └── DB + Contracts + Repos + Service + Controller

Phase 3: Resource Hierarchy Cascade     [Week 2]
    └── DB + Contracts + Repos + Engine

Phase 4: Resource Collections           [Week 3]
    └── DB + Contracts + Matcher + Repos + Service + Controller

Phase 5: Resource Policies              [Week 3-4]
    └── DB + Contracts + Repos + Service + Controller + Engine

Phase 6: Documentation & Testing        [Week 4]
    └── Docs + Tests
```

---

## Database Migration Order

```sql
-- Phase 1.1
ALTER TABLE role_permissions ADD COLUMN condition JSONB;
ALTER TABLE scope_role_permission_overrides ADD COLUMN condition JSONB;

-- Phase 1.2
ALTER TABLE resources RENAME COLUMN scope_id TO owner_scope_id;

-- Phase 2
CREATE TYPE link_type AS ENUM ('share', 'alias', 'mirror');
CREATE TABLE resource_scope_links (...);
-- Migrate data from resource_scopes
DROP TABLE resource_scopes;

-- Phase 3
CREATE TYPE cascade_mode AS ENUM ('inherit', 'none');
ALTER TABLE resource_hierarchy_edges ADD COLUMN cascade cascade_mode;

-- Phase 4
CREATE TABLE resource_collections (...);

-- Phase 5
CREATE TYPE policy_target_kind AS ENUM ('resource', 'collection');
CREATE TYPE policy_effect AS ENUM ('allow', 'deny');
CREATE TABLE resource_policies (...);

-- Phase 1.1 (immediate)
ALTER TABLE permissions DROP COLUMN logic;
```

---

## Risk Mitigation

| Risk | Mitigation |
|------|------------|
| Engine performance with hierarchy traversal | Add caching, limit recursion depth |
| Collection matching performance | Index resource fields, cache collection membership |
| Policy evaluation overhead | Evaluate policies lazily, cache policy lookups |
