# Bedrock Documentation Implementation Plan

## Current State Analysis

### What Exists ✅
| Section | Status | Notes |
|---------|--------|-------|
| **Introduction** | ✅ Complete | Good overview, agent-first messaging |
| **Quickstart** | ✅ Complete | Batch operations, clear examples |
| **User Governance Guide** | ✅ Complete | 7.4KB |
| **Agent Governance Guide** | ✅ Complete | 12.7KB - comprehensive |
| **Multi-tenant Guide** | ✅ Complete | 8.9KB |
| **Scope Overrides Guide** | ✅ Complete | 8.9KB - covers all 3 override types |
| **API Reference** | ✅ Complete | 102 files covering all endpoints |
| **AI Tools** | ✅ Complete | Claude Code, Cursor, Windsurf |

### What's Missing ❌
Based on the updated ChatGPT outline and codebase analysis:

| Section | Priority | Effort | Notes |
|---------|----------|--------|-------|
| **Core Concepts** | 🔴 High | Medium | Scopes, roles, permissions, evaluation engine |
| **Platform Structure** | 🔴 High | Medium | Scope types, scope type hierarchy |
| **Management Models** | 🔴 High | Medium | Tenants, Workspaces, Projects, Environments, Users, API Keys |
| **Resources & Resource Types** | 🔴 High | Medium | Resource hierarchies, multi-scope resources |
| **Tags & Tag Groups** | 🔴 High | Medium | Tag group bindings, taggable models |
| **API Keys** | 🔴 High | Low | `BedrockApiKey` exists in management-core |
| **Delegation (On Behalf Of)** | 🟡 Medium | Low | AI agent delegation patterns |
| **Conditional Permissions** | 🟡 Medium | Medium | JSON Logic, metadata-based conditions |
| **SDK Documentation** | 🟡 Medium | High | TypeScript SDK usage |
| **Modeling Guides** | 🟡 Medium | High | Industry-specific patterns |
| **Testing & Debugging** | 🟢 Low | Low | Permission evaluator, debugging denies |
| **Security** | 🟢 Low | Low | Tenant isolation, best practices |

### Management Models (from `bedrock-management-core`)

These are Bedrock Cloud's built-in management entities that map to scopes:

| Model | Interface | Key Properties |
|-------|-----------|----------------|
| **Tenant** | `BedrockTenant` | `kind`, `status`, `planKey`, `maxUsers`, `maxWorkspaces`, billing fields |
| **Workspace** | `BedrockWorkspace` | `tenantId`, `scopeId`, `name` |
| **Project** | `BedrockProject` | `tenantId`, `workspaceId`, `scopeId`, `name` |
| **Environment** | `BedrockEnvironment` | `tenantId`, `projectId`, `scopeId`, `name` |
| **User** | `BedrockUser` | `subjectId`, `kindeId`, `email`, `status`, `defaultTenantId` |
| **Tenant Member** | `TenantMember` | `tenantId`, `userId`, `subjectId`, `isOwner`, `status` |
| **API Key** | `BedrockApiKey` | `subjectId`, `tenantId`, `environmentId`, `prefix`, `keyHash`, `status` |

---

## Implementation Plan

### Phase 1: Core Concepts (Priority: High)
Create foundational documentation that explains Bedrock's mental model.

#### 1.1 Create `/concepts/` section ✅ COMPLETE
- [x] `concepts/index.mdx` - Core concepts overview
- [x] `concepts/scopes.mdx` - Scopes & scope hierarchy
- [x] `concepts/scope-types.mdx` - Scope types & scope type hierarchy
- [x] `concepts/subjects.mdx` - Subject types (user, agent, service, system)
- [x] `concepts/roles.mdx` - Roles & role assignments
- [x] `concepts/permissions.mdx` - Permissions, actions, resource patterns
- [x] `concepts/evaluation.mdx` - How permission evaluation works

**Status:** ✅ Complete

---

### Phase 1b: Management Models (Priority: High)
Document Bedrock Cloud's built-in management layer.

#### 1b.1 Create `/platform/` section
- [ ] `platform/index.mdx` - Platform structure overview
- [ ] `platform/tenants.mdx` - Tenant model (`BedrockTenant`, `TenantKindEnum`, `TenantStatusEnum`)
- [ ] `platform/workspaces.mdx` - Workspace model (`BedrockWorkspace`)
- [ ] `platform/projects.mdx` - Project model (`BedrockProject`)
- [ ] `platform/environments.mdx` - Environment model (`BedrockEnvironment`)
- [ ] `platform/users.mdx` - User model (`BedrockUser`, Kinde integration)
- [ ] `platform/tenant-members.mdx` - Tenant membership (`TenantMember`, ownership)
- [ ] `platform/api-keys.mdx` - API key model (`BedrockApiKey`, `ApiKeyStatusEnum`)

#### Key relationships to document:
```
Tenant (scopeId) ──┬── Workspace (scopeId) ── Project (scopeId) ── Environment (scopeId)
                   │
                   ├── TenantMember ── User (subjectId)
                   │
                   └── ApiKey (subjectId, tenantId?, environmentId?)
```

**Estimated effort:** 4-5 hours

---

### Phase 2: Resources & Tags (Priority: High) ✅ COMPLETE
Document the resource and tagging systems.

#### 2.1 Create `/resources/` section ✅
- [x] `resources/index.mdx` - Resources overview
- [x] `resources/resource-types.mdx` - Defining resource types
- [x] `resources/resource-hierarchies.mdx` - Parent-child resources (`BedrockResourceHierarchy`)
- [x] `resources/multi-scope-resources.mdx` - Sharing resources across scopes (`BedrockResourceScope`)

#### 2.2 Create `/tags/` section ✅
- [x] `tags/index.mdx` - Tags overview
- [x] `tags/tag-groups.mdx` - Tag groups, constraints, locking
- [x] `tags/tag-bindings.mdx` - Tag group bindings (`BedrockTagGroupBinding`)
- [x] `tags/tag-based-access.mdx` - Using tags for access control

**Status:** ✅ Complete

---

### Phase 3: Delegation & Advanced Patterns (Priority: Medium) ✅ COMPLETE
Document delegation and advanced authorization patterns.

#### 3.1 Create `/delegation/` section ✅
- [x] `delegation/index.mdx` - Delegation overview
- [x] `delegation/on-behalf-of.mdx` - Actor vs Principal, `onBehalfOf` usage
- [x] `delegation/agent-delegation.mdx` - AI agents acting for users

#### 3.2 Enhance existing guides ✅
- [x] Create `guides/conditional-permissions.mdx` - JSON Logic, metadata conditions

**Status:** ✅ Complete

---

### Phase 4: SDK Documentation (Priority: Medium)
Document SDK usage for developers.

#### 4.1 Create `/sdks/` section
- [ ] `sdks/index.mdx` - SDK overview
- [ ] `sdks/typescript.mdx` - TypeScript/JavaScript SDK
- [ ] `sdks/nestjs.mdx` - NestJS integration
- [ ] `sdks/angular.mdx` - Angular guards and directives

**Estimated effort:** 4-6 hours (depends on SDK maturity)

---

### Phase 5: Modeling Guides (Priority: Medium) ✅ COMPLETE
Real-world patterns that differentiate Bedrock docs.

#### 5.1 Expand `/guides/` section ✅
- [x] `guides/modeling-saas.mdx` - Multi-tenant SaaS patterns
- [x] `guides/modeling-organizations.mdx` - Org → Team → Project
- [x] `guides/row-level-access.mdx` - Resource-level permissions
- [ ] `guides/industry-construction.mdx` - Construction SaaS example (deferred)
- [ ] `guides/industry-healthcare.mdx` - HIPAA-style permissions (deferred)

**Status:** ✅ Core guides complete (industry examples deferred)

---

### Phase 6: Testing & Security (Priority: Low)
Operational documentation.

#### 6.1 Create `/testing/` section
- [ ] `testing/index.mdx` - Testing overview
- [ ] `testing/debugging-denies.mdx` - Why was access denied?
- [ ] `testing/simulating-conditions.mdx` - Testing conditional permissions

#### 6.2 Create `/security/` section
- [ ] `security/index.mdx` - Security overview
- [ ] `security/tenant-isolation.mdx` - Scope isolation guarantees
- [ ] `security/best-practices.mdx` - API key security, JSON Logic safety

**Estimated effort:** 3-4 hours

---

## Updated Navigation Structure

```json
{
  "navigation": {
    "tabs": [
      {
        "tab": "Guides",
        "groups": [
          {
            "group": "Getting Started",
            "pages": ["index", "quickstart"]
          },
          {
            "group": "Core Concepts",
            "pages": [
              "concepts/index",
              "concepts/scopes",
              "concepts/scope-types",
              "concepts/subjects",
              "concepts/roles",
              "concepts/permissions",
              "concepts/evaluation"
            ]
          },
          {
            "group": "Platform (Bedrock Cloud)",
            "pages": [
              "platform/index",
              "platform/tenants",
              "platform/workspaces",
              "platform/projects",
              "platform/environments",
              "platform/users",
              "platform/tenant-members",
              "platform/api-keys"
            ]
          },
          {
            "group": "Resources & Tags",
            "pages": [
              "resources/index",
              "resources/resource-types",
              "resources/resource-hierarchies",
              "resources/multi-scope-resources",
              "tags/index",
              "tags/tag-groups",
              "tags/tag-bindings",
              "tags/tag-based-access"
            ]
          },
          {
            "group": "Use Cases",
            "pages": [
              "guides/user-governance",
              "guides/agent-governance",
              "guides/multi-tenant",
              "guides/scope-overrides"
            ]
          },
          {
            "group": "Delegation",
            "pages": [
              "delegation/index",
              "delegation/on-behalf-of",
              "delegation/agent-delegation"
            ]
          },
          {
            "group": "Modeling Guides",
            "pages": [
              "guides/modeling-saas",
              "guides/modeling-organizations",
              "guides/row-level-access",
              "guides/conditional-permissions"
            ]
          },
          {
            "group": "SDKs",
            "pages": [
              "sdks/index",
              "sdks/typescript",
              "sdks/nestjs",
              "sdks/angular"
            ]
          },
          {
            "group": "Testing & Debugging",
            "pages": [
              "testing/index",
              "testing/debugging-denies"
            ]
          },
          {
            "group": "Security",
            "pages": [
              "security/index",
              "security/tenant-isolation",
              "security/best-practices"
            ]
          }
        ]
      },
      {
        "tab": "API Reference",
        "groups": [
          // ... existing API reference structure
        ]
      }
    ]
  }
}
```

---

## Recommended Execution Order

| Phase | Section | Est. Time | Dependencies | Status |
|-------|---------|-----------|--------------|--------|
| 1 | Core Concepts | 4-6 hrs | None | ✅ Complete |
| 1b | Management Models (Platform) | 4-5 hrs | Phase 1 | Pending |
| 2 | Resources & Tags | 4-6 hrs | Phase 1 | ✅ Complete |
| 3 | Delegation | 3-4 hrs | Phase 1 | ✅ Complete |
| 4 | SDKs | 4-6 hrs | Phase 1 | Pending |
| 5 | Modeling Guides | 6-8 hrs | Phases 1-3 | ✅ Complete |
| 6 | Testing & Security | 3-4 hrs | Phase 1 | Pending |

**Total estimated effort:** 28-39 hours

---

## Files to Create (Summary)

### New Directories
- `/concepts/` (7 files)
- `/platform/` (8 files) — Management models
- `/resources/` (4 files)
- `/tags/` (4 files)
- `/delegation/` (3 files)
- `/sdks/` (4 files)
- `/testing/` (2 files)
- `/security/` (3 files)

### New Files in Existing Directories
- `/guides/` (5 new files)

**Total new files:** ~40 files

---

## Notes

1. **API Keys** - ✅ Found in `bedrock-management-core` (`BedrockApiKey`), included in Phase 1b
2. **Changelog** - Mintlify handles this automatically, no action needed
3. **UI Guide** - Deferred until Bedrock UI app is ready for documentation
4. **Examples Library** - Can be added incrementally to existing guides

---

## Next Steps

1. Review this plan and adjust priorities
2. Decide which phase to start with
3. I can generate the actual content for any section on approval
