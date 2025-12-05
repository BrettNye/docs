# Resource Collections & Resource Policies Implementation Plan

This document outlines the implementation plan for two new Bedrock features:
1. **Resource Collections** - Dynamic groupings of resources based on match criteria
2. **Resource Policies** - Direct allow/deny rules attached to resources or collections

---

## Part 1: Resource Collections

### Overview

Resource Collections are **dynamic, query-based groupings** of resources. Instead of manually assigning resources to groups, collections define match criteria that automatically include matching resources.

**Use Cases:**
- "All documents tagged as 'confidential'"
- "All resources where `status == 'active'`"
- "All invoices created in the last 30 days"
- "All resources owned by department 'finance'"

### Type Definition (Already Done)

```typescript
export interface BedrockResourceCollection {
    id: string;
    name: string;
    resourceType: string;  // Scoped to a specific resource type
    match: ResourceMatchDefinition;
}

export interface ResourceMatchDefinition {
    fields?: Record<string, PrimitiveValue | FieldMatchRule>;
    tags?: Record<string, string | string[]>;
    patterns?: Record<string, string>;  // Glob/regex patterns
    time?: Record<string, TimeMatchRule>;
    any?: ResourceMatchDefinition[];   // OR
    all?: ResourceMatchDefinition[];   // AND
    none?: ResourceMatchDefinition[];  // NOT
    condition?: JsonLogicExpr;         // Escape hatch for complex logic
}
```

### Implementation Steps

#### 1.1 Database Schema

**File:** `libs/bedrock/bedrock-core-storage/src/lib/postgres/schemas/resource.schema.ts`

```typescript
// BedrockResourceCollection
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

#### 1.2 Zod Contract Schema

**File:** `libs/bedrock/bedrock-contracts/src/lib/bedrock/resource-collection.schema.ts` (NEW)

```typescript
import { z } from 'zod';
import { idField, scopeIdField, nameField } from './standard-fields.schema';

const primitiveValueSchema = z.union([z.string(), z.number(), z.boolean(), z.null()]);

const fieldMatchRuleSchema = z.union([
    primitiveValueSchema,
    z.object({ equals: primitiveValueSchema.optional() }),
    z.object({ in: z.array(primitiveValueSchema).optional() }),
    z.object({ notIn: z.array(primitiveValueSchema).optional() }),
    z.object({ gt: primitiveValueSchema.optional() }),
    z.object({ gte: primitiveValueSchema.optional() }),
    z.object({ lt: primitiveValueSchema.optional() }),
    z.object({ lte: primitiveValueSchema.optional() }),
    z.object({ contains: primitiveValueSchema.optional() }),
    z.object({ exists: z.boolean().optional() }),
]);

const timeMatchRuleSchema = z.object({
    eq: z.union([z.string(), z.date()]).optional(),
    gt: z.union([z.string(), z.date()]).optional(),
    gte: z.union([z.string(), z.date()]).optional(),
    lt: z.union([z.string(), z.date()]).optional(),
    lte: z.union([z.string(), z.date()]).optional(),
    relative: z.string().optional(),
});

const resourceMatchDefinitionSchema: z.ZodType<unknown> = z.lazy(() =>
    z.object({
        fields: z.record(z.string(), z.union([primitiveValueSchema, fieldMatchRuleSchema])).optional(),
        tags: z.record(z.string(), z.union([z.string(), z.array(z.string())])).optional(),
        patterns: z.record(z.string(), z.string()).optional(),
        time: z.record(z.string(), timeMatchRuleSchema).optional(),
        any: z.array(resourceMatchDefinitionSchema).optional(),
        all: z.array(resourceMatchDefinitionSchema).optional(),
        none: z.array(resourceMatchDefinitionSchema).optional(),
        condition: z.record(z.string(), z.unknown()).optional(),
    })
);

export const createResourceCollectionSchema = z.object({
    id: idField.optional(),
    name: nameField,
    scopeId: scopeIdField,
    resourceTypeId: idField,
    match: resourceMatchDefinitionSchema,
});

export const updateResourceCollectionSchema = z.object({
    name: nameField.optional(),
    match: resourceMatchDefinitionSchema.optional(),
});

export type CreateResourceCollectionDto = z.infer<typeof createResourceCollectionSchema>;
export type UpdateResourceCollectionDto = z.infer<typeof updateResourceCollectionSchema>;
```

#### 1.3 Store Interface

**File:** `libs/bedrock/bedrock-core/src/lib/interfaces/store.types.ts`

```typescript
// Add to BedrockStore interface
getResourceCollectionById(id: string): Promise<BedrockResourceCollection | null>;
getResourceCollectionsForScope(scopeId: string): Promise<BedrockResourceCollection[]>;
getResourceCollectionsForResourceType(resourceTypeId: string): Promise<BedrockResourceCollection[]>;
```

#### 1.4 Collection Matcher Engine

**File:** `libs/bedrock/bedrock-core/src/lib/collection-matcher.ts` (NEW)

```typescript
import * as jsonLogic from 'json-logic-js';
import { ResourceMatchDefinition, BedrockResource, BedrockTag } from './interfaces';

export class CollectionMatcher {
    /**
     * Check if a resource matches a collection's match definition
     */
    matches(
        resource: BedrockResource,
        resourceData: Record<string, unknown>,
        tags: BedrockTag[],
        match: ResourceMatchDefinition
    ): boolean {
        // Field matching
        if (match.fields) {
            for (const [field, rule] of Object.entries(match.fields)) {
                if (!this.matchField(resourceData[field], rule)) return false;
            }
        }

        // Tag matching
        if (match.tags) {
            for (const [tagKey, expected] of Object.entries(match.tags)) {
                const tag = tags.find(t => t.identifier === tagKey);
                if (!tag) return false;
                if (Array.isArray(expected)) {
                    if (!expected.includes(tag.label)) return false;
                } else {
                    if (tag.label !== expected) return false;
                }
            }
        }

        // Time matching
        if (match.time) {
            for (const [field, rule] of Object.entries(match.time)) {
                if (!this.matchTime(resourceData[field], rule)) return false;
            }
        }

        // Pattern matching
        if (match.patterns) {
            for (const [field, pattern] of Object.entries(match.patterns)) {
                if (!this.matchPattern(resourceData[field], pattern)) return false;
            }
        }

        // Combinators
        if (match.all && match.all.length > 0) {
            if (!match.all.every(m => this.matches(resource, resourceData, tags, m))) return false;
        }
        if (match.any && match.any.length > 0) {
            if (!match.any.some(m => this.matches(resource, resourceData, tags, m))) return false;
        }
        if (match.none && match.none.length > 0) {
            if (match.none.some(m => this.matches(resource, resourceData, tags, m))) return false;
        }

        // JSON Logic escape hatch
        if (match.condition) {
            const context = { resource, data: resourceData, tags };
            try {
                if (!jsonLogic.apply(match.condition, context)) return false;
            } catch {
                return false;
            }
        }

        return true;
    }

    private matchField(value: unknown, rule: unknown): boolean {
        // Implementation for field matching rules
        // ...
    }

    private matchTime(value: unknown, rule: TimeMatchRule): boolean {
        // Implementation for time matching with relative support
        // ...
    }

    private matchPattern(value: unknown, pattern: string): boolean {
        // Glob or regex pattern matching
        // ...
    }
}
```

#### 1.5 Repository & Service

**Files:**
- `apps/bedrock/api-management/src/core/repositories/services/resource-collection.repository.ts`
- `apps/bedrock/api-management/src/core/resources/services/resource-collection.service.ts`

#### 1.6 Controller

**File:** `apps/bedrock/api-management/src/core/resources/controllers/resource-collection.controller.ts`

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/resource-collections?scopeId=` | Get collections for scope |
| GET | `/resource-collections/:id` | Get collection by ID |
| GET | `/resource-collections/:id/resources` | Get resources matching collection |
| POST | `/resource-collections` | Create collection |
| PATCH | `/resource-collections/:id` | Update collection |
| DELETE | `/resource-collections/:id` | Delete collection |
| POST | `/resource-collections/:id/test` | Test if a resource matches |

---

## Part 2: Resource Policies

### Overview

Resource Policies are **direct allow/deny rules** attached to specific resources or collections. They bypass the role-based permission system for resource-specific access control.

**Use Cases:**
- "Deny all access to this resource except the owner"
- "Allow read access to all resources in the 'public-docs' collection"
- "Deny delete on resources tagged 'archived'"
- "Allow access only during business hours"

### Type Definition (Already Done)

```typescript
type ResourcePolicyTarget =
  | { kind: "resource"; resourceId: string }
  | { kind: "collection"; collectionId: string };

export interface BedrockResourcePolicy {
    id: string;
    target: ResourcePolicyTarget;
    actions: string[];              // e.g., ["read", "update"] or ["*"]
    effect: "allow" | "deny";
    subjectCondition?: JsonLogicExpr;   // Who this applies to
    contextCondition?: JsonLogicExpr;   // When this applies
}
```

### Implementation Steps

#### 2.1 Database Schema

**File:** `libs/bedrock/bedrock-core-storage/src/lib/postgres/schemas/resource.schema.ts`

```typescript
// Policy target kind enum
export const policyTargetKindEnum = pgEnum('policy_target_kind', ['resource', 'collection']);
export const policyEffectEnum = pgEnum('policy_effect', ['allow', 'deny']);

// BedrockResourcePolicy
export const resourcePolicies = pgTable('resource_policies', {
    id: text('id').primaryKey(),
    scopeId: text('scope_id').notNull().references(() => scopes.id),
    targetKind: policyTargetKindEnum('target_kind').notNull(),
    targetId: text('target_id').notNull(),  // resourceId or collectionId
    actions: jsonb('actions').notNull().$type<string[]>(),
    effect: policyEffectEnum('effect').notNull(),
    subjectCondition: jsonb('subject_condition').$type<Record<string, unknown>>(),
    contextCondition: jsonb('context_condition').$type<Record<string, unknown>>(),
    priority: integer('priority').notNull().default(0),  // Higher = evaluated first
    createdAt: text('created_at').notNull(),
    createdBy: text('created_by').notNull(),
}, (t) => [
    index('idx_resource_policies_scope').on(t.scopeId),
    index('idx_resource_policies_target').on(t.targetKind, t.targetId),
]);
```

#### 2.2 Zod Contract Schema

**File:** `libs/bedrock/bedrock-contracts/src/lib/bedrock/resource-policy.schema.ts` (NEW)

```typescript
import { z } from 'zod';
import { idField, scopeIdField, logicField } from './standard-fields.schema';

const policyTargetSchema = z.discriminatedUnion('kind', [
    z.object({ kind: z.literal('resource'), resourceId: idField }),
    z.object({ kind: z.literal('collection'), collectionId: idField }),
]);

const policyEffectSchema = z.enum(['allow', 'deny']);

export const createResourcePolicySchema = z.object({
    id: idField.optional(),
    scopeId: scopeIdField,
    target: policyTargetSchema,
    actions: z.array(z.string().min(1)),
    effect: policyEffectSchema,
    subjectCondition: logicField.optional(),
    contextCondition: logicField.optional(),
    priority: z.number().int().optional(),
});

export const updateResourcePolicySchema = z.object({
    actions: z.array(z.string().min(1)).optional(),
    effect: policyEffectSchema.optional(),
    subjectCondition: logicField.optional(),
    contextCondition: logicField.optional(),
    priority: z.number().int().optional(),
});

export type CreateResourcePolicyDto = z.infer<typeof createResourcePolicySchema>;
export type UpdateResourcePolicyDto = z.infer<typeof updateResourcePolicySchema>;
```

#### 2.3 Store Interface

**File:** `libs/bedrock/bedrock-core/src/lib/interfaces/store.types.ts`

```typescript
// Add to BedrockStore interface
getResourcePolicyById(id: string): Promise<BedrockResourcePolicy | null>;
getResourcePoliciesForScope(scopeId: string): Promise<BedrockResourcePolicy[]>;
getResourcePoliciesForResource(resourceId: string): Promise<BedrockResourcePolicy[]>;
getResourcePoliciesForCollection(collectionId: string): Promise<BedrockResourcePolicy[]>;
```

#### 2.4 Engine Integration

**File:** `libs/bedrock/bedrock-core/src/lib/engine.ts`

Update `evaluate()` to check resource policies **before** role-based permissions:

```typescript
async evaluate(input: BedrockEvaluateInput): Promise<BedrockDecision> {
    const resourceContext = await this.resolveResourceContext(input);

    // 1) Check resource policies FIRST (explicit allow/deny)
    if (resourceContext.resource) {
        const policyDecision = await this.evaluateResourcePolicies({
            resource: resourceContext.resource,
            action: input.action,
            subject: await this.store.getSubjectById(input.actor.subjectId),
            context: resourceContext.context,
        });

        if (policyDecision) {
            // Policy made an explicit decision
            return policyDecision;
        }
    }

    // 2) Fall through to role-based evaluation
    // ... existing logic ...
}

private async evaluateResourcePolicies(params: {
    resource: BedrockResource;
    action: string;
    subject: BedrockSubject;
    context: Record<string, unknown>;
}): Promise<BedrockDecision | null> {
    const { resource, action, subject, context } = params;

    // Get policies for this specific resource
    const directPolicies = await this.store.getResourcePoliciesForResource(resource.id);

    // Get policies for collections this resource belongs to
    const collections = await this.getCollectionsContainingResource(resource);
    const collectionPolicies = await Promise.all(
        collections.map(c => this.store.getResourcePoliciesForCollection(c.id))
    );

    // Combine and sort by priority (higher first)
    const allPolicies = [...directPolicies, ...collectionPolicies.flat()]
        .sort((a, b) => (b.priority ?? 0) - (a.priority ?? 0));

    for (const policy of allPolicies) {
        // Check if action matches
        if (!policy.actions.includes('*') && !policy.actions.includes(action)) {
            continue;
        }

        // Check subject condition
        if (policy.subjectCondition) {
            const subjectContext = { subject, ...context };
            if (!jsonLogic.apply(policy.subjectCondition, subjectContext)) {
                continue;
            }
        }

        // Check context condition
        if (policy.contextCondition) {
            if (!jsonLogic.apply(policy.contextCondition, context)) {
                continue;
            }
        }

        // Policy matches - return decision based on effect
        return {
            allowed: policy.effect === 'allow',
            matches: [],
            explanation: `Resource policy ${policy.id} ${policy.effect}s ${action}`,
            evaluatedPolicy: policy,
        };
    }

    // No policy matched - fall through to role-based
    return null;
}
```

#### 2.5 Repository & Service

**Files:**
- `apps/bedrock/api-management/src/core/repositories/services/resource-policy.repository.ts`
- `apps/bedrock/api-management/src/core/resources/services/resource-policy.service.ts`

#### 2.6 Controller

**File:** `apps/bedrock/api-management/src/core/resources/controllers/resource-policy.controller.ts`

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/resource-policies?scopeId=` | Get policies for scope |
| GET | `/resource-policies/:id` | Get policy by ID |
| GET | `/resource-policies/for-resource/:resourceId` | Get policies for resource |
| GET | `/resource-policies/for-collection/:collectionId` | Get policies for collection |
| POST | `/resource-policies` | Create policy |
| PATCH | `/resource-policies/:id` | Update policy |
| DELETE | `/resource-policies/:id` | Delete policy |

---

## Evaluation Order

The updated evaluation order in `BedrockEngine.evaluate()`:

```
1. Resource Policies (explicit allow/deny)
   ├── Direct resource policies (sorted by priority)
   └── Collection policies (sorted by priority)
   
2. Role-Based Permissions (if no policy matched)
   ├── Role-permission conditions
   └── Scope overrides with conditions
```

**Deny takes precedence:** If any matching policy has `effect: "deny"`, access is denied regardless of other policies.

---

## Implementation Checklist

### Resource Collections
- [ ] Add `resourceCollections` table to `resource.schema.ts`
- [ ] Create `resource-collection.schema.ts` Zod contracts
- [ ] Export from `bedrock-contracts` index
- [ ] Add store interface methods
- [ ] Implement in `postgres-store.ts`
- [ ] Implement in `in-memory-store.ts`
- [ ] Implement `CollectionMatcher` engine
- [ ] Create repository
- [ ] Create service
- [ ] Create controller
- [ ] Add to module
- [ ] Update documentation

### Resource Policies
- [ ] Add `resourcePolicies` table to `resource.schema.ts`
- [ ] Add enums for target kind and effect
- [ ] Create `resource-policy.schema.ts` Zod contracts
- [ ] Export from `bedrock-contracts` index
- [ ] Add store interface methods
- [ ] Implement in `postgres-store.ts`
- [ ] Implement in `in-memory-store.ts`
- [ ] Update `BedrockEngine.evaluate()` to check policies first
- [ ] Add `evaluatedPolicy` to `BedrockDecision` type
- [ ] Create repository
- [ ] Create service
- [ ] Create controller
- [ ] Add to module
- [ ] Update documentation

---

## Example Use Cases

### Collection: "Confidential Documents"

```json
{
    "name": "Confidential Documents",
    "resourceType": "document",
    "match": {
        "tags": { "classification": "confidential" }
    }
}
```

### Policy: "Only owners can delete"

```json
{
    "target": { "kind": "collection", "collectionId": "col_confidential" },
    "actions": ["delete"],
    "effect": "deny",
    "subjectCondition": {
        "!=": [{ "var": "subject.id" }, { "var": "resource.ownerId" }]
    }
}
```

### Policy: "Business hours only"

```json
{
    "target": { "kind": "resource", "resourceId": "res_sensitive" },
    "actions": ["*"],
    "effect": "deny",
    "contextCondition": {
        "or": [
            { "<": [{ "var": "time.hour" }, 9] },
            { ">": [{ "var": "time.hour" }, 17] }
        ]
    }
}
```

### Collection: "Active invoices from last 30 days"

```json
{
    "name": "Recent Active Invoices",
    "resourceType": "invoice",
    "match": {
        "all": [
            { "fields": { "status": "active" } },
            { "time": { "createdAt": { "gte": { "relative": "now_minus_30d" } } } }
        ]
    }
}
```
