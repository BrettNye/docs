# Condition Migration Plan

Move `logic` from `BedrockPermission` to edge-based `condition` on `BedrockRolePermission` and `BedrockScopeRolePermissionOverride`.

---

## Overview

**Before:** Conditions lived on the permission itself—evaluated the same way for all roles.

**After:** Conditions live on the **role↔permission edge**—the same permission can have different conditions depending on which role grants it.

**Example:**
- Permission: `document:read:*`
- Role "Editor" grants it with condition: `{ "==": [{ "var": "resource.status" }, "draft"] }`
- Role "Admin" grants it with **no condition** (full access)

---

## Updated Type Definitions (Already Done)

```typescript
// core.types.ts
export type JsonLogicExpr = Record<string, unknown>;

export interface BedrockRolePermission {
    id: string;
    roleId: string;
    permissionId: string;
    condition?: JsonLogicExpr;  // NEW
}

export interface BedrockScopeRolePermissionOverride {
    id: string;
    childScopeId: string;
    roleId: string;
    permissionId: string;
    state: EdgeStateEnum;
    condition?: JsonLogicExpr;  // NEW
}

// BedrockPermission - logic field REMOVED
```

---

## Implementation Steps

### 1. Database Schemas (`core.schema.ts`)

**File:** `libs/bedrock/bedrock-core-storage/src/lib/postgres/schemas/core.schema.ts`

| Table | Change |
|-------|--------|
| `rolePermissions` | Add `condition: jsonb('condition').$type<Record<string, unknown>>()` |
| `scopeRolePermissionOverrides` | Add `condition: jsonb('condition').$type<Record<string, unknown>>()` |
| `permissions` | Remove `logic` column (Phase 2) |

```typescript
// rolePermissions - ADD condition
export const rolePermissions = pgTable('role_permissions', {
    id: text('id').primaryKey(),
    roleId: text('role_id').notNull().references(() => roles.id),
    permissionId: text('permission_id').notNull().references(() => permissions.id),
    condition: jsonb('condition').$type<Record<string, unknown>>(),  // NEW
}, (t) => [
    uniqueIndex('uq_role_permissions').on(t.roleId, t.permissionId),
]);

// scopeRolePermissionOverrides - ADD condition
export const scopeRolePermissionOverrides = pgTable('scope_role_permission_overrides', {
    id: text('id').primaryKey(),
    childScopeId: text('child_scope_id').notNull().references(() => scopes.id),
    roleId: text('role_id').notNull().references(() => roles.id),
    permissionId: text('permission_id').notNull().references(() => permissions.id),
    state: edgeStateEnum('state').notNull(),
    condition: jsonb('condition').$type<Record<string, unknown>>(),  // NEW
}, (t) => [
    uniqueIndex('uq_scope_role_permission_overrides').on(t.childScopeId, t.roleId, t.permissionId),
]);
```

---

### 2. Zod Contract Schemas

**Files:**
- `libs/bedrock/bedrock-contracts/src/lib/bedrock/role-permission.schema.ts`
- `libs/bedrock/bedrock-contracts/src/lib/bedrock/overrides.schema.ts`

#### role-permission.schema.ts

```typescript
import { z } from 'zod';
import { idField, roleIdField, permissionIdField, logicField } from './standard-fields.schema';

export const createRolePermissionSchema = z.object({
    id: idField.optional(),
    roleId: roleIdField,
    permissionId: permissionIdField,
    condition: logicField.optional(),  // NEW
});

export type CreateRolePermissionDto = z.infer<typeof createRolePermissionSchema>;
```

#### overrides.schema.ts

```typescript
// createScopeRolePermissionOverrideSchema - ADD condition
export const createScopeRolePermissionOverrideSchema = z.object({
    id: idField.optional(),
    childScopeId: scopeIdField,
    roleId: roleIdField,
    permissionId: permissionIdField,
    state: edgeStateEnum,
    condition: logicField.optional(),  // NEW
});

// updateScopeRolePermissionOverrideSchema - ADD condition
export const updateScopeRolePermissionOverrideSchema = z.object({
    state: edgeStateEnum.optional(),
    condition: logicField.optional(),  // NEW
});
```

---

### 3. Repositories

**Files:**
- `apps/bedrock/api-management/src/core/repositories/services/role-permission.repository.ts`
- `apps/bedrock/api-management/src/core/repositories/services/scope-role-permission-override.repository.ts`

#### role-permission.repository.ts

```typescript
async create(dto: CreateRolePermissionDto): Promise<BedrockRolePermission> {
    const [created] = await this.db
        .insert(rolePermissions)
        .values({
            id: generateId('role_perm'),
            roleId: dto.roleId,
            permissionId: dto.permissionId,
            condition: dto.condition,  // NEW
        })
        .returning();

    return this.map(created);
}

private map(row: typeof rolePermissions.$inferSelect): BedrockRolePermission {
    return {
        id: row.id,
        roleId: row.roleId,
        permissionId: row.permissionId,
        condition: row.condition ?? undefined,  // NEW
    };
}
```

#### scope-role-permission-override.repository.ts

```typescript
async create(dto: CreateScopeRolePermissionOverrideDto): Promise<BedrockScopeRolePermissionOverride> {
    const [created] = await this.db
        .insert(scopeRolePermissionOverrides)
        .values({
            id: generateId('override'),
            childScopeId: dto.childScopeId,
            roleId: dto.roleId,
            permissionId: dto.permissionId,
            state: dto.state as EdgeStateEnum,
            condition: dto.condition,  // NEW
        })
        .returning();

    return this.map(created);
}

async update(id: string, dto: UpdateScopeRolePermissionOverrideDto): Promise<BedrockScopeRolePermissionOverride> {
    const updateData: Partial<typeof scopeRolePermissionOverrides.$inferInsert> = {};
    if (dto.state !== undefined) updateData.state = dto.state as EdgeStateEnum;
    if (dto.condition !== undefined) updateData.condition = dto.condition;  // NEW

    const [updated] = await this.db
        .update(scopeRolePermissionOverrides)
        .set(updateData)
        .where(eq(scopeRolePermissionOverrides.id, id))
        .returning();

    return this.map(updated);
}

private map(row: typeof scopeRolePermissionOverrides.$inferSelect): BedrockScopeRolePermissionOverride {
    return {
        id: row.id,
        childScopeId: row.childScopeId,
        roleId: row.roleId,
        permissionId: row.permissionId,
        state: row.state as BedrockScopeRolePermissionOverride['state'],
        condition: row.condition ?? undefined,  // NEW
    };
}
```

---

### 4. Engine Evaluation (`engine.ts`)

**File:** `libs/bedrock/bedrock-core/src/lib/engine.ts`

**Current behavior:** Evaluates `permission.logic` after matching action/resourceType/pattern.

**New behavior:**
1. Load `rolePermissions` with their `condition` fields
2. For each matching permission, check if the granting `rolePermission.condition` passes
3. A permission is granted if **any** role-permission edge with a passing condition exists

#### Key Changes

```typescript
// In evaluateForSubject():

// Load role-permission links WITH conditions
const rolePermissions = await this.store.getRolePermissionsForRoles(roleIds);

// For each permission, check if ANY granting role-permission has a passing condition
const matchingPermissions = permissions.filter((p) => {
    if (p.action !== action) return false;
    if (p.resourceType !== resourceTypeKey) return false;
    if (!this.matchesPattern(p.resourcePattern, resourcePattern)) return false;

    // Find role-permissions that grant this permission
    const grantingRolePerms = rolePermissions.filter(rp => rp.permissionId === p.id);
    
    // Check if ANY granting role-permission passes its condition
    return grantingRolePerms.some(rp => {
        if (!rp.condition) return true;  // No condition = always passes
        try {
            return !!jsonLogic.apply(rp.condition, logicContextBase);
        } catch {
            return false;  // Fail closed
        }
    });
});
```

#### Store Interface Update

**File:** `libs/bedrock/bedrock-core/src/lib/interfaces/store.types.ts`

Ensure `getRolePermissionsForRoles` returns the full `BedrockRolePermission` including `condition`.

---

### 5. Remove `logic` from Permissions

**Direct removal (no backward compatibility):**

- `core.schema.ts` - Remove `logic` from `permissions` table
- `permission.schema.ts` - Remove `logic` from create/update schemas
- `permission.service.ts` - Remove logic handling
- `permission.repository.ts` - Remove logic from map function

---

### 6. Documentation Updates

| Document | Updates Needed |
|----------|----------------|
| `docs/bedrock-api.md` | Update Role Permissions section to document `condition` field |
| `documentation/Chatgpt-overview-suggestion.md` | Update "Conditional Permissions" section to reflect edge-based conditions |

#### bedrock-api.md Updates

Add to Role Permissions section:

```markdown
### Role Permission Conditions

Role permissions can include a `condition` field containing a JSON Logic expression.
The condition is evaluated at runtime against the request context.

**Request:**
```json
POST /role-permissions
{
    "roleId": "role_abc123",
    "permissionId": "perm_xyz789",
    "condition": {
        "==": [{ "var": "resource.ownerId" }, { "var": "subject.id" }]
    }
}
```

**Context available in conditions:**
- `subject` - The subject being evaluated
- `resource` - Resource metadata (if resource lookup enabled)
- `tags` - Resource tags as key-value pairs
- `tagList` - Array of tag objects
- `actorSubjectId` - The actor's subject ID
- `principalSubjectId` - The principal's subject ID (if delegated)
```

---

## Migration Checklist

- [ ] Update `rolePermissions` table schema
- [ ] Update `scopeRolePermissionOverrides` table schema
- [ ] Update `createRolePermissionSchema` Zod schema
- [ ] Update `createScopeRolePermissionOverrideSchema` Zod schema
- [ ] Update `updateScopeRolePermissionOverrideSchema` Zod schema
- [ ] Update `RolePermissionRepository`
- [ ] Update `ScopeRolePermissionOverrideRepository`
- [ ] Update `BedrockEngine.evaluateForSubject()`
- [ ] Update `BedrockStore` interface if needed
- [ ] Update `postgres-store.ts` implementation
- [ ] Update `in-memory-store.ts` implementation
- [ ] Run database migration
- [ ] Update `bedrock-api.md`
- [ ] Update `Chatgpt-overview-suggestion.md`
- [ ] Remove `logic` from permissions table
- [ ] Remove `logic` from permission Zod contracts
- [ ] Remove `logic` from permission repository/service

---

## Testing Scenarios

1. **No condition** - Role-permission without condition grants access
2. **Passing condition** - Role-permission with condition that evaluates to true
3. **Failing condition** - Role-permission with condition that evaluates to false
4. **Multiple roles** - Same permission granted by multiple roles with different conditions
5. **Override with condition** - ScopeRolePermissionOverride adds condition at child scope
6. **Malformed condition** - Invalid JSON Logic fails closed (denies access)
