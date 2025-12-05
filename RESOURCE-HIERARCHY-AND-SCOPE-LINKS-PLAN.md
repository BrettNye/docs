# Resource Hierarchy Cascade & Scope Links Implementation Plan

This document covers three related changes:
1. **Rename `scopeId` → `ownerScopeId`** on `BedrockResource`
2. **Add `BedrockResourceScopeLink`** - replaces/enhances `BedrockResourceScope`
3. **Add `cascade` to `BedrockResourceHierarchy`** - permission inheritance through resource trees

---

## Part 1: Rename `scopeId` → `ownerScopeId`

### Rationale

The current `scopeId` on `BedrockResource` is ambiguous—it could mean:
- The scope that **owns** the resource
- A scope the resource **appears in**

Renaming to `ownerScopeId` makes the ownership relationship explicit, while `BedrockResourceScopeLink` handles additional scope associations.

### Type Definition (Already Done)

```typescript
export interface BedrockResource {
    id: string;
    resourceTypeId: string;
    ownerScopeId: string;  // RENAMED from scopeId
    externalResourceId: string;
    displayName?: string;
    createdAt: string;
    createdBy: string;
}
```

### Files to Update

| Layer | File | Change |
|-------|------|--------|
| **DB Schema** | `resource.schema.ts` | Rename column `scope_id` → `owner_scope_id` |
| **Zod Contract** | `resource.schema.ts` | Rename field `scopeId` → `ownerScopeId` |
| **Repository** | `resource.repository.ts` | Update field mapping |
| **Store** | `postgres-store.ts` | Update queries |
| **Store** | `in-memory-store.ts` | Update field references |
| **Engine** | `engine.ts` | Update `resolveResourceContext` |
| **Tests** | Various | Update test fixtures |

### Database Schema Change

```typescript
// resource.schema.ts
export const resources = pgTable('resources', {
    id: text('id').primaryKey(),
    resourceTypeId: text('resource_type_id').notNull().references(() => resourceTypes.id),
    ownerScopeId: text('owner_scope_id').notNull().references(() => scopes.id),  // RENAMED
    externalResourceId: text('external_resource_id').notNull(),
    displayName: text('display_name'),
    createdAt: text('created_at').notNull(),
    createdBy: text('created_by').notNull(),
}, (t) => [
    uniqueIndex('uq_resources_external').on(t.resourceTypeId, t.externalResourceId),
    index('idx_resources_owner_scope').on(t.ownerScopeId),  // RENAMED
    index('idx_resources_type_scope').on(t.ownerScopeId, t.resourceTypeId),  // RENAMED
]);
```

### Zod Contract Change

```typescript
// resource.schema.ts (contracts)
export const createResourceSchema = z.object({
    id: idField.optional(),
    resourceTypeId: resourceTypeIdField,
    ownerScopeId: scopeIdField,  // RENAMED
    externalResourceId: externalIdField,
    displayName: displayNameField.optional(),
});
```

### Migration SQL

```sql
-- Rename column
ALTER TABLE resources RENAME COLUMN scope_id TO owner_scope_id;

-- Rename indexes
ALTER INDEX idx_resources_scope RENAME TO idx_resources_owner_scope;
ALTER INDEX idx_resources_type_scope RENAME TO idx_resources_type_owner_scope;
```

---

## Part 2: BedrockResourceScopeLink

### Rationale

The existing `BedrockResourceScope` is a simple join table. The new `BedrockResourceScopeLink` adds:
- **`linkType`** - Describes how the resource appears in the scope (share, alias, mirror)
- **`metadata`** - Arbitrary data for UI/business logic

This replaces `BedrockResourceScope` with a more expressive model.

### Type Definition (Already Done)

```typescript
export interface BedrockResourceScopeLink {
    id: string;
    resourceId: string;
    scopeId: string;           // Extra scope this resource appears in
    linkType?: "share" | "alias" | "mirror";
    metadata?: Record<string, any>;
}
```

### Link Types

| Type | Description | Use Case |
|------|-------------|----------|
| `share` | Resource is shared into this scope | Cross-team collaboration |
| `alias` | Resource appears under a different identity | Rebranding, localization |
| `mirror` | Read-only copy synced from owner scope | Reporting, dashboards |

### Database Schema

```typescript
// resource.schema.ts
import { pgEnum, jsonb } from 'drizzle-orm/pg-core';

export const linkTypeEnum = pgEnum('link_type', ['share', 'alias', 'mirror']);

// BedrockResourceScopeLink (replaces resourceScopes)
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
    index('idx_resource_scope_links_type').on(t.linkType),
]);
```

### Zod Contract

```typescript
// resource-scope-link.schema.ts (NEW)
import { z } from 'zod';
import { idField, scopeIdField } from './standard-fields.schema';

const linkTypeSchema = z.enum(['share', 'alias', 'mirror']);

export const createResourceScopeLinkSchema = z.object({
    id: idField.optional(),
    resourceId: idField,
    scopeId: scopeIdField,
    linkType: linkTypeSchema.optional(),
    metadata: z.record(z.string(), z.unknown()).optional(),
});

export const updateResourceScopeLinkSchema = z.object({
    linkType: linkTypeSchema.optional(),
    metadata: z.record(z.string(), z.unknown()).optional(),
});

export type CreateResourceScopeLinkDto = z.infer<typeof createResourceScopeLinkSchema>;
export type UpdateResourceScopeLinkDto = z.infer<typeof updateResourceScopeLinkSchema>;
```

### Store Interface

```typescript
// store.types.ts - Add to BedrockStore
getResourceScopeLinkById(id: string): Promise<BedrockResourceScopeLink | null>;
getResourceScopeLinksForResource(resourceId: string): Promise<BedrockResourceScopeLink[]>;
getResourceScopeLinksForScope(scopeId: string): Promise<BedrockResourceScopeLink[]>;
getResourceScopeLinksByType(linkType: string): Promise<BedrockResourceScopeLink[]>;
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
| DELETE | `/resource-scope-links/:resourceId/:scopeId` | Delete by resource+scope |

### Replace BedrockResourceScope

```sql
-- Drop old table
DROP TABLE resource_scopes;

-- Create new table
CREATE TABLE resource_scope_links (
    id TEXT PRIMARY KEY,
    resource_id TEXT NOT NULL REFERENCES resources(id),
    scope_id TEXT NOT NULL REFERENCES scopes(id),
    link_type link_type,
    metadata JSONB,
    created_at TEXT NOT NULL,
    created_by TEXT NOT NULL
);
```

---

## Part 3: Resource Hierarchy Cascade

### Rationale

When resources form hierarchies (folder → document, project → task), permissions should optionally cascade from parent to child. The `cascade` field controls this behavior.

### Type Definition (Already Done)

```typescript
export interface BedrockResourceHierarchy {
    parentResourceId: string;
    childResourceId: string;
    relationshipType?: string;
    cascade?: "inherit" | "none";  // NEW
    createdAt: string;
}
```

### Cascade Modes

| Mode | Description | Behavior |
|------|-------------|----------|
| `inherit` | Child inherits parent's permissions | Access to parent grants access to child |
| `none` | No inheritance | Child requires explicit permissions |
| (null/undefined) | Default behavior | Treat as `inherit` for backward compat |

### Database Schema

```typescript
// resource.schema.ts
import { pgEnum } from 'drizzle-orm/pg-core';

export const cascadeModeEnum = pgEnum('cascade_mode', ['inherit', 'none']);

export const resourceHierarchyEdges = pgTable('resource_hierarchy_edges', {
    parentResourceId: text('parent_resource_id').notNull().references(() => resources.id),
    childResourceId: text('child_resource_id').notNull().references(() => resources.id),
    relationshipType: text('relationship_type'),
    cascade: cascadeModeEnum('cascade'),  // NEW
    createdAt: text('created_at').notNull(),
}, (t) => [
    { pk: { columns: [t.parentResourceId, t.childResourceId] } },
    index('idx_resource_hierarchy_cascade').on(t.cascade),  // NEW
]);
```

### Zod Contract

```typescript
// resource.schema.ts (contracts)
const cascadeModeSchema = z.enum(['inherit', 'none']);

export const createResourceHierarchyEdgeSchema = z.object({
    parentResourceId: resourceIdField,
    childResourceId: resourceIdField,
    relationshipType: z.string().optional(),
    cascade: cascadeModeSchema.optional(),  // NEW
});

export const updateResourceHierarchyEdgeSchema = z.object({
    relationshipType: z.string().optional(),
    cascade: cascadeModeSchema.optional(),  // NEW
});
```

### Engine Integration

Update `BedrockEngine` to check parent permissions when evaluating child resources:

```typescript
// engine.ts
private async evaluateWithHierarchy(params: {
    resource: BedrockResource;
    action: string;
    subjectId: string;
    scopeId: string;
    context: Record<string, unknown>;
}): Promise<BedrockDecision | null> {
    const { resource, action, subjectId, scopeId, context } = params;

    // Get parent hierarchy edges with cascade=inherit
    const parentEdges = await this.store.getParentResourceEdges(resource.id);
    const inheritingEdges = parentEdges.filter(e => e.cascade !== 'none');

    if (inheritingEdges.length === 0) {
        return null;  // No inheritance, use direct evaluation
    }

    // Check if any parent grants access
    for (const edge of inheritingEdges) {
        const parentResource = await this.store.getResourceById(edge.parentResourceId);
        if (!parentResource) continue;

        const parentDecision = await this.evaluateForResource({
            resource: parentResource,
            action,
            subjectId,
            scopeId,
            context: {
                ...context,
                inheritedFrom: {
                    resourceId: parentResource.id,
                    relationshipType: edge.relationshipType,
                },
            },
        });

        if (parentDecision.allowed) {
            return {
                ...parentDecision,
                explanation: `Access inherited from parent resource ${parentResource.id}`,
                inheritedFrom: parentResource.id,
            };
        }
    }

    return null;  // No parent granted access
}
```

### Store Interface

```typescript
// store.types.ts - Add to BedrockStore
getParentResourceEdges(childResourceId: string): Promise<BedrockResourceHierarchy[]>;
getChildResourceEdges(parentResourceId: string): Promise<BedrockResourceHierarchy[]>;
getResourceAncestors(resourceId: string, cascadeOnly?: boolean): Promise<BedrockResource[]>;
getResourceDescendants(resourceId: string, cascadeOnly?: boolean): Promise<BedrockResource[]>;
```

### API Endpoints Update

| Method | Endpoint | Description |
|--------|----------|-------------|
| PATCH | `/resource-hierarchy/:parentId/:childId` | Update edge (cascade, relationshipType) |

---

## Evaluation Order with Hierarchy

Updated evaluation flow in `BedrockEngine.evaluate()`:

```
1. Resource Policies (explicit allow/deny)
   ├── Direct resource policies
   └── Collection policies

2. Resource Hierarchy Inheritance (if cascade=inherit)
   └── Check parent resource permissions recursively

3. Role-Based Permissions
   ├── Role-permission conditions
   └── Scope overrides with conditions
```

---

## Migration Checklist

### Part 1: scopeId → ownerScopeId
- [ ] Update `resources` table schema (rename column)
- [ ] Update `createResourceSchema` Zod contract
- [ ] Update `resource.repository.ts` mapping
- [ ] Update `postgres-store.ts` queries
- [ ] Update `in-memory-store.ts` field references
- [ ] Update `engine.ts` resource context
- [ ] Create database migration script
- [ ] Update tests

### Part 2: BedrockResourceScopeLink
- [ ] Create `linkTypeEnum` in schema
- [ ] Create `resourceScopeLinks` table
- [ ] Create `resource-scope-link.schema.ts` Zod contracts
- [ ] Add store interface methods
- [ ] Create repository
- [ ] Create service
- [ ] Create controller
- [ ] Remove old `resourceScopes` table and all references
- [ ] Update documentation

### Part 3: Resource Hierarchy Cascade
- [ ] Create `cascadeModeEnum` in schema
- [ ] Add `cascade` column to `resourceHierarchyEdges`
- [ ] Update `createResourceHierarchyEdgeSchema`
- [ ] Create `updateResourceHierarchyEdgeSchema`
- [ ] Add store methods for ancestor/descendant queries
- [ ] Update engine with hierarchy evaluation
- [ ] Add `inheritedFrom` to `BedrockDecision`
- [ ] Update documentation

---

## Example Scenarios

### Scenario 1: Shared Document

```typescript
// Document owned by Engineering, shared to Marketing
const doc: BedrockResource = {
    id: "res_doc123",
    ownerScopeId: "scope_engineering",
    // ...
};

const link: BedrockResourceScopeLink = {
    id: "link_1",
    resourceId: "res_doc123",
    scopeId: "scope_marketing",
    linkType: "share",
    metadata: { sharedBy: "user_alice", sharedAt: "2024-01-15" }
};
```

### Scenario 2: Folder → Document Inheritance

```typescript
// Folder with documents that inherit permissions
const folderEdge: BedrockResourceHierarchy = {
    parentResourceId: "res_folder_finance",
    childResourceId: "res_doc_budget",
    relationshipType: "contains",
    cascade: "inherit",  // Document inherits folder permissions
    createdAt: "2024-01-01"
};

// User with "read" on folder automatically gets "read" on document
```

### Scenario 3: Breaking Inheritance

```typescript
// Sensitive document that doesn't inherit from parent folder
const sensitiveEdge: BedrockResourceHierarchy = {
    parentResourceId: "res_folder_hr",
    childResourceId: "res_doc_salaries",
    relationshipType: "contains",
    cascade: "none",  // Requires explicit permission
    createdAt: "2024-01-01"
};
```
