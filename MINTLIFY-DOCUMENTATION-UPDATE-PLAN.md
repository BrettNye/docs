# Mintlify Documentation Update Plan

This plan outlines all changes needed to update the Bedrock documentation to reflect the latest features and improve the overall user experience.

---

## 📋 Executive Summary

The documentation needs updates in three areas:

1. **New Features** - Resource Collections, Resource Policies, Resource Scope Links, Conditional Permissions
2. **Homepage Redesign** - Better onboarding, clearer value proposition, architecture diagram
3. **Navigation Restructure** - Improved flow, new sections, better discoverability

---

## 🆕 Phase 1: New Feature Documentation

### 1.1 New Concept Pages (Create)

| File | Description | Priority | Status |
|------|-------------|----------|--------|
| `concepts/conditional-permissions.mdx` | JSON Logic conditions on role-permissions and overrides | High | ✅ Done |
| `resources/resource-scope-links.mdx` | Replace `multi-scope-resources.mdx` with scope links (share/alias/mirror) | High | ✅ Done |
| `resources/resource-collections.mdx` | Dynamic resource grouping with match definitions | High | ✅ Done |
| `resources/resource-policies.mdx` | Fine-grained allow/deny policies on resources/collections | High | ✅ Done |

### 1.2 Update Existing Concept Pages

| File | Changes Needed | Status |
|------|----------------|--------|
| `resources/resource-hierarchies.mdx` | Add cascade modes (`inherit`/`none`), ancestor traversal | ✅ Done |
| `concepts/evaluation.mdx` | Update evaluation order: Policies → Hierarchy → Roles | ✅ Done |
| `concepts/permissions.mdx` | Add section on conditional permissions with JSON Logic | ⏳ (links to new page) |
| `concepts/index.mdx` | Add links to new concepts | ✅ Done |

### 1.3 New API Reference Pages (Create) ✅ Done

| Group | Files Created | Status |
|-------|---------------|--------|
| **Resource Scope Links** | `get-resource-scope-links.mdx`, `get-resource-scope-link.mdx`, `create-resource-scope-link.mdx`, `create-resource-scope-links-batch.mdx`, `update-resource-scope-link.mdx`, `delete-resource-scope-link.mdx` | ✅ |
| **Resource Collections** | `get-resource-collections.mdx`, `get-resource-collection.mdx`, `create-resource-collection.mdx`, `create-resource-collections-batch.mdx`, `update-resource-collection.mdx`, `delete-resource-collection.mdx` | ✅ |
| **Resource Policies** | `get-resource-policies.mdx`, `get-resource-policy.mdx`, `create-resource-policy.mdx`, `create-resource-policies-batch.mdx`, `update-resource-policy.mdx`, `delete-resource-policy.mdx` | ✅ |
| **Resource Hierarchy** | `get-resource-parent.mdx`, `get-resource-children.mdx`, `get-resource-ancestors.mdx`, `update-resource-hierarchy-edge.mdx` | ✅ |

### 1.4 Update docs.json Navigation ✅ Done

Added new groups to Guides tab:
- Split "Resources & Tags" into separate "Resources" and "Tags" groups
- Added `concepts/conditional-permissions` to Core Concepts
- Added `resources/resource-scope-links`, `resources/resource-collections`, `resources/resource-policies` to Resources

API reference tab updated with:
- Resource Scope Links (6 endpoints)
- Resource Collections (6 endpoints)
- Resource Policies (6 endpoints)
- Resource Hierarchy endpoints added to Resources group (4 endpoints)

---

## 🏠 Phase 2: Homepage Redesign ✅ Done

### 2.1 New Homepage Structure (`index.mdx`) ✅ Implemented

```
1. Hero Section
   - Strong tagline
   - Value proposition
   - Primary CTA → Quickstart

2. Why Bedrock? (Feature Cards)
   - Hierarchical Scopes
   - Resource Policies
   - Conditional Permissions
   - AI Agent Governance
   - Multi-tenant Isolation
   - Tag-based Access

3. Architecture Diagram
   - Visual: Subject → Scope → Role → Permission → Evaluation
   - Link to concepts

4. Core Concepts Grid
   - Quick reference table with links

5. Quickstart Preview
   - 5-step code snippet
   - Link to full quickstart

6. Use Cases
   - SaaS Platform
   - Multi-tenant App
   - Document Management
   - AI Agent Governance
```

### 2.2 New Hero Section Copy

```markdown
# Bedrock Authorization Platform

**Multi-tenant, hierarchical authorization for modern applications.**

Model scopes, roles, permissions, resources, tags, and policies—then evaluate 
access with full inheritance, overrides, and conditional logic.

<CardGroup cols={2}>
  <Card title="Quickstart" icon="rocket" href="/quickstart">
    Get started in 5 minutes
  </Card>
  <Card title="Core Concepts" icon="book" href="/concepts">
    Understand the fundamentals
  </Card>
</CardGroup>
```

### 2.3 Why Bedrock Section

```markdown
## Why Bedrock?

Bedrock solves authorization problems traditional RBAC cannot:

<CardGroup cols={3}>
  <Card title="Hierarchical Scopes" icon="sitemap">
    Org → Workspace → Project → Environment with inheritance
  </Card>
  <Card title="Resource Policies" icon="shield-check">
    Fine-grained allow/deny on specific resources or collections
  </Card>
  <Card title="Conditional Permissions" icon="code">
    JSON Logic expressions for dynamic, context-aware access
  </Card>
  <Card title="AI Agent Governance" icon="robot">
    Same authorization model for users, services, and AI agents
  </Card>
  <Card title="Multi-tenant Isolation" icon="building">
    Complete tenant separation with scope hierarchies
  </Card>
  <Card title="Tag-based Access" icon="tags">
    Dynamic permissions based on resource and subject tags
  </Card>
</CardGroup>
```

### 2.4 Core Concepts Table

| Concept | Description |
|---------|-------------|
| **Scope** | Hierarchical node (org, workspace, project) |
| **ScopeType** | Defines scope behavior (DEFINE vs MERGE) |
| **Subject** | Actor: user, service, agent, api_key |
| **Role** | Bundle of permissions |
| **Permission** | Action + resource type + pattern |
| **Override** | Modify role/permission at child scope |
| **Resource** | Protected object with type and owner |
| **Collection** | Dynamic resource group via match rules |
| **Policy** | Allow/deny rule on resource or collection |
| **Tag** | Metadata for conditional access |

---

## 🧭 Phase 3: Navigation Restructure ✅ Done

### 3.1 Sidebar Structure ✅ Implemented

**New pages created:**
- `architecture.mdx` - Architecture Overview in Getting Started
- `concepts/overrides.mdx` - Scope Overrides in Core Concepts

**Removed:**
- `resources/multi-scope-resources.mdx` (replaced by `resource-scope-links.mdx`)

### 3.2 Final Sidebar Structure

```
📚 Guides Tab
├── Getting Started
│   ├── Introduction (index.mdx) ← REDESIGN
│   ├── Quickstart ← UPDATE with new features
│   └── Architecture Overview ← NEW
│
├── Core Concepts
│   ├── Overview (concepts/index.mdx)
│   ├── Scopes
│   ├── Scope Types
│   ├── Subjects
│   ├── Roles
│   ├── Permissions
│   ├── Conditional Permissions ← NEW
│   ├── Overrides ← NEW (extract from scope-overrides guide)
│   └── Evaluation ← UPDATE
│
├── Resources
│   ├── Resources
│   ├── Resource Types
│   ├── Resource Hierarchies ← UPDATE (cascade modes)
│   ├── Resource Scope Links ← NEW (replaces multi-scope)
│   ├── Resource Collections ← NEW
│   └── Resource Policies ← NEW
│
├── Tags
│   ├── Tags
│   ├── Tag Groups
│   ├── Tag Bindings
│   └── Tag-based Access
│
├── Delegation
│   ├── Overview
│   ├── On Behalf Of
│   └── Agent Delegation
│
├── Use Cases
│   ├── User Governance
│   ├── Agent Governance
│   ├── Multi-tenant
│   └── Scope Overrides
│
└── Modeling Guides
    ├── SaaS Platform
    ├── Organizations
    ├── Row-level Access
    └── Conditional Permissions

📡 API Reference Tab
├── Overview
├── Subjects
├── Memberships
├── Roles
├── Role Assignments
├── Permissions
├── Role Permissions
├── Scopes
├── Scope Types
├── Scope Hierarchy
├── Scope Type Hierarchy
├── Scope Overrides
├── Resources
├── Resource Types
├── Resource Hierarchy ← UPDATE (add endpoints)
├── Resource Scope Links ← NEW
├── Resource Collections ← NEW
├── Resource Policies ← NEW
├── Tag Groups
├── Tags
├── Tag Bindings
├── Tag Assignments
├── Tenants
├── Workspaces
├── Projects
├── Environments
├── Users
└── API Keys
```

---

## 📝 Phase 4: Content Updates ✅ Done

### 4.1 Quickstart Updates ✅ Implemented

Added new sections:
- **Step 5: Add Resources and Policies** - resource types, resources, tags, collections, policies
- **Step 6: Evaluate Permissions** - testing the authorization setup with real examples
- Updated overview to mention new features
- Updated "What You Built" diagram
- Updated Next Steps with new concept pages

### 4.2 Evaluation Page Updates ✅ (Done in Phase 1)

Update `concepts/evaluation.mdx` with:

```markdown
## Evaluation Order

The engine evaluates permissions in this order:

1. **Resource Policies** (highest priority)
   - Direct policies on the resource
   - Policies on matching collections
   - Sorted by priority (higher first)
   - Subject and context conditions via JSON Logic

2. **Resource Hierarchy Inheritance**
   - If resource has parents with `cascade: inherit`
   - Check ancestor permissions recursively

3. **Role-Based Permissions**
   - Memberships in scope hierarchy
   - Role assignments and permissions
   - Scope overrides
   - Conditional permissions via JSON Logic
```

### 4.3 New Architecture Page

Create `concepts/architecture.mdx`:

```markdown
# Architecture Overview

## Evaluation Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    BedrockEngine.evaluate()                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  1. RESOURCE POLICIES (highest priority)                    │
│     ├── Get policies targeting resource                     │
│     ├── Get collections matching resource → their policies  │
│     ├── Filter by action, sort by priority                  │
│     └── Evaluate conditions → return if match               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  2. RESOURCE HIERARCHY                                      │
│     ├── Get parent resources with cascade: inherit          │
│     └── Check ancestor permissions recursively              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  3. ROLE-BASED PERMISSIONS                                  │
│     ├── Get memberships in scope hierarchy                  │
│     ├── Get role assignments → permissions                  │
│     ├── Apply scope overrides                               │
│     └── Evaluate conditional permissions                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ BedrockDecision │
                    │  allowed: bool  │
                    │  matches: [...]  │
                    │  policy?: ...   │
                    └─────────────────┘
```
```

---

## ✅ Phase 5: Checklist

### New Files to Create

- [ ] `concepts/architecture.mdx`
- [ ] `concepts/conditional-permissions.mdx`
- [ ] `resources/resource-scope-links.mdx`
- [ ] `resources/resource-collections.mdx`
- [ ] `resources/resource-policies.mdx`
- [ ] `api-reference/resource-scope-links/*.mdx` (6 files)
- [ ] `api-reference/resource-collections/*.mdx` (6 files)
- [ ] `api-reference/resource-policies/*.mdx` (6 files)
- [ ] `api-reference/resource-hierarchy/get-ancestors.mdx`
- [ ] `api-reference/resource-hierarchy/update-edge.mdx`

### Files to Update

- [ ] `index.mdx` - Complete redesign
- [ ] `quickstart.mdx` - Add new features
- [ ] `concepts/index.mdx` - Add new concept links
- [ ] `concepts/evaluation.mdx` - New evaluation order
- [ ] `concepts/permissions.mdx` - Conditional permissions
- [ ] `resources/resource-hierarchies.mdx` - Cascade modes
- [ ] `docs.json` - New navigation structure

### Files to Remove/Rename

- [ ] `resources/multi-scope-resources.mdx` → Replace with `resource-scope-links.mdx`

---

## 📅 Execution Order

1. **Week 1: Foundation**
   - Create new concept pages
   - Update existing concept pages
   - Update docs.json navigation

2. **Week 2: API Reference**
   - Create all new API reference pages
   - Update resource hierarchy endpoints

3. **Week 3: Homepage & Polish**
   - Redesign homepage
   - Create architecture page
   - Update quickstart
   - Final review

---

## 🎯 Success Metrics

- [ ] All new features documented
- [ ] Homepage clearly explains Bedrock value
- [ ] New users can find "start here" path
- [ ] Evaluation order clearly documented
- [ ] All API endpoints have reference pages
- [ ] Navigation is logical and complete
