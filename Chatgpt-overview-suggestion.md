Below is a **complete, production-grade documentation outline tailored for Mintlify**, built specifically for **Bedrock** and consistent with your architecture (scopes, permissions, tags, custom scope types, project/env model, API keys, resource permissions, tenant management, etc.).

This structure fits perfectly into Mintlify's left-nav + page-based organization and naturally separates **public customer-facing docs** from **internal platform docs** you should not publish.

I'll format it exactly how Mintlify expects (sections, nested pages, suggested slugs, and recommended ordering).

---

> **⚠️ Review Notes (Based on Codebase Analysis)**
>
> This outline is ~95% aligned with the actual Bedrock implementation. Recent additions:
> - ✅ Three-tier override system (`BedrockScopeRoleOverride`, `BedrockScopePermissionOverride`, `BedrockScopeRolePermissionOverride`)
> - ✅ Resource hierarchies (`BedrockResourceHierarchy`, `BedrockResourceTypeHierarchy`) with cascade modes
> - ✅ Resource Scope Links (`BedrockResourceScopeLink`) - replaces old `BedrockResourceScope`
> - ✅ Resource Collections (`BedrockResourceCollection`) - dynamic resource grouping via match rules
> - ✅ Resource Policies (`BedrockResourcePolicy`) - fine-grained allow/deny policies on resources/collections
> - ✅ Tag group bindings (`BedrockTagGroupBinding`)
> - ✅ Scope type hierarchy (`BedrockScopeTypeHierarchyEdge`)
> - ✅ Conditional permissions via JSON Logic on role-permissions and scope overrides
> - API Keys section may be speculative (no types found in codebase)
> - Delegation/"On Behalf Of" should be promoted to first-class section
>
> See inline `[UPDATED]` markers for corrections.

---

# ✅ **Mintlify Documentation Outline for Bedrock**

---

# 🧱 **1. Overview**

### **/overview/**

* **What is Bedrock?**

  * Authorization + identity context engine
  * Scopes, roles, permissions, tags, resources
  * Designed for multi-tenant, multi-project systems
* **Core Concepts**

  * Scopes & hierarchy
  * Role-based access
  * Permission granularity
  * Resource-driven permissions
  * Tag-based conditions
  * Evaluation engine overview
* **Quick Start**

  * Install SDK
  * Initialize client
  * Perform permission check
  * Basic modeling example

---

# 🧱 **2. Platform Structure**

### **/platform/**

* **Scope Types (Tenant → Workspace → Project → Environment)**

  * Explanation of Bedrock Cloud's built-in types
  * `permissionMode`: DEFINE vs MERGE
* **Customer-Defined Scope Types**

  * Creating custom scope types (Organization, Department, Division, Team)
  * When to create custom types
* **Scope Type Hierarchy** `[UPDATED]`

  * Defining which scope types can contain which (`BedrockScopeTypeHierarchyEdge`)
  * Configuring the scope type schema
* **Scope Hierarchy**

  * How scopes create the permission tree
  * Inheritance rules
  * Overriding roles/permissions

---

# 👤 **3. Subjects & Membership**

### **/subjects/**

* **Subjects**

  * User, Service, Agent, System
  * Subject metadata
* **Membership**

  * Assigning subjects to scopes
  * Nested membership rules
  * Inheriting roles from parent scopes
* **External IDs**

  * Mapping your app’s user IDs
  * Multiple external systems

---

# 🔐 **4. Roles**

### **/roles/**

* **What is a Role?**

  * Conceptual overview
  * Role lifecycle
* **Defining Roles**

  * What scope they belong to
  * How roles inherit through scopes
* **Custom Roles**

  * Creating custom application-specific roles
  * When customers should create roles
* **Role Overrides**

  * Overriding roles at a deeper scope
  * Use cases
* **Role Templates (Optional)**

  * Recommended starting roles per industry

---

# 🛂 **5. Permissions**

### **/permissions/**

* **Permission Model**

  * Key
  * Scope
  * Logic/conditions
  * Resource connection
* **Actions**

  * CRUD-style semantics
  * Recommended patterns (`job.read`, `job.update`, `doc.approve`, etc.)
* **Permission Inheritance**

  * From project → environment → custom scopes
  * DEFINE vs MERGE
* **Conditional Permissions** `[UPDATED]`

  * JSON Logic expressions for dynamic evaluation
  * Conditions on role-permissions (`BedrockRolePermission.condition`)
  * Conditions on scope overrides (`BedrockScopeRolePermissionOverride.condition`)
  * Using subject metadata in conditions
  * Using resource metadata in conditions
  * Using tags in conditions
  * Context variables available during evaluation
* **Permission Overrides** `[UPDATED - Three-Tier Override System]`

  * `BedrockScopeRoleOverride` — Toggle a role on/off at a child scope
  * `BedrockScopePermissionOverride` — Toggle a permission on/off at a child scope
  * `BedrockScopeRolePermissionOverride` — Toggle a specific role↔permission edge with `EdgeStateEnum`
  * When to use each override type
* **Common Permission Patterns (Deep Dive)**

  * Row-level access
  * Dynamic conditions
  * Feature-level permissions

---

# 📦 **6. Resources & Resource Permissions**

### **/resources/**

* **Resource Types**

  * What is a resource?
  * Resource type definitions
* **Resource Type Hierarchies** `[UPDATED]`

  * Parent-child resource type relationships (`BedrockResourceTypeHierarchy`)
  * Inheritance patterns for resource types
* **Resource Instances**

  * Modeling documents, jobs, products, employees, etc.
* **Resource Hierarchies** `[UPDATED]`

  * Parent-child resource relationships (`BedrockResourceHierarchy`)
  * Cascade modes: `inherit` (propagate permissions) or `none` (no inheritance)
  * Hierarchical permission propagation (e.g., folder → document)
  * Relationship types for semantic meaning
  * Ancestor traversal for permission evaluation
* **Resource Ownership**

  * How owner-based permissions work
* **Resource Scope Links** `[UPDATED]`

  * Associating resources with multiple scopes (`BedrockResourceScopeLink`)
  * Link types: `share`, `alias`, `mirror`
  * Metadata on links for custom attributes
  * Sharing and classification use cases
* **Resource Collections** `[NEW]`

  * Dynamic resource grouping (`BedrockResourceCollection`)
  * Match rules: fields, tags, patterns, time-based, JSON Logic conditions
  * Composable matching with `any`, `all`, `none` operators
  * Use cases: "all finance documents", "resources created this week"
* **Resource Policies** `[NEW]`

  * Fine-grained access control (`BedrockResourcePolicy`)
  * Target: specific resource or collection
  * Effects: `allow` or `deny`
  * Subject conditions: JSON Logic for actor matching
  * Context conditions: JSON Logic for request context
  * Priority-based evaluation (higher priority wins)
  * Evaluated before role-based permissions
* **Resource Tags**

  * Department-level access
  * Category-level access
* **Permission Examples**

  * `job.update` for job `123`
  * `document.read` for finance-only documents
  * `employee.manage` by division

---

# 🏷️ **7. Tags & Tag Groups**

### **/tags/**

* **What Tags Are**

  * Taggable models (`TaggableModelTypeEnum`)
  * Auto-generated vs manual
* **Tag Groups**

  * Grouping similar tags
  * Example: Labor Classes, Departments
  * `maxAppliedPerTarget` constraints
  * `TagGroupOriginEnum` — origin tracking
  * Locked tag groups (`isLocked`)
* **Tag Group Bindings** `[UPDATED]`

  * Binding tag groups to specific models (`BedrockTagGroupBinding`)
  * `TagAssociationModelTypeEnum` — which models can use which tag groups
* **Tag-Based Access**

  * Grant access based on subject tags
  * Grant access based on resource tags
* **UI Modeling Guide**

  * How to show tag-based rules in your UI

---

# 🧩 **8. API Keys & Machine Access** `[⚠️ VERIFY - No types found in codebase]`

### **/api-keys/**

> **Note:** This section may describe planned functionality. Verify implementation status before documenting.

* **API Key Types**

  * Access levels
  * Role-limited API keys
* **Key Permissions**

  * Scope constraints
  * Resource constraints
* **Rotating Keys**
* **Best Practices**

  * Short-lived keys
  * Tenant-level vs environment-level keys

---

# 🧩 **9. Management API (Public)**

### **/api/**

(Each page includes: request/response, validation, errors, examples)

* **Auth & Request Signing**
* **Subjects API**

  * Create subject
  * Update subject
  * Attach membership
* **Roles API**

  * Create role
  * Assign to scope
  * Override at child scope
* **Permissions API**

  * Create permission
  * Attach logic
* **Scopes API**

  * Create scope
  * Hierarchy operations
* **Resources API**

  * Register resource
  * Tag resource
* **Tags API**

  * Create tags
  * Assign tags to subjects/resources
* **API Keys**

  * Create/delete/regenerate
* **Evaluation API**

  * `can(user).do("action").on(resource)`
  * Bulk evaluations
  * Condition debugging

---

# 🧰 **10. SDKs**

### **/sdks/**

* **JavaScript/TypeScript**

  * Installation
  * Usage
  * Examples
* **Node**

  * Permission check wrapper
  * Express/NestJS middleware
* **Angular**

  * Route guards
  * Component visibility guards
* **Backend Integration Guides**

  * NestJS
  * Fastify
  * Express
* **Frontend Integration Guides**

  * Angular
  * React (later)

---

# 🌳 **11. Modeling Guides (Your strongest section)**

### **/guides/**

This is where your documentation becomes truly valuable.

* **Modeling Multi-Tenant SaaS**

  * Tenants → workspaces → projects → env
* **Modeling Organizations Inside Projects**

  * Org group
  * Organization
  * Team
* **Modeling Divisions & Departments**

  * Cross-environment consistency
* **How to Build a Role System in Your App**

  * Good defaults
  * Overrides
* **Resource Permissions Deep Dive**

  * How to secure documents, jobs, products
* **Tag-Based Access Patterns**

  * Finance-only documents
  * Construction crew labor classes
* **Condition-Based Access**

  * Time-based
  * Metadata-based
  * User attribute-based
* **Industry Examples**

  * Construction SaaS (Crew, Jobs, Approvals)
  * HR SaaS (Employees, Departments)
  * Marketplace apps
  * Healthcare apps (HIPAA-style permissions)
  * Education SaaS
* **Complex Scenarios**

  * Multi-level approvals

---

# 🤝 **11b. Delegation & Acting On Behalf** `[UPDATED - Promoted to first-class section]`

### **/delegation/**

Given the rise of AI agents, this deserves dedicated coverage.

* **Overview**

  * What is delegation in Bedrock?
  * Actor vs Principal (`onBehalfOf`)
* **Delegation Scenarios**

  * AI agents acting for users
  * Service accounts acting for users
  * Automated workflows
* **Evaluation Behavior**

  * How `BedrockEvaluateInput.onBehalfOf` works
  * Validation between actor and principal
  * `usedDelegation` and `delegationId` in decisions
* **Security Considerations**

  * Limiting delegation scope
  * Audit trails for delegated actions

---

# 🔎 **12. Tutorials**

### **/tutorials/**

* **RBAC in 10 Minutes**
* **Build an app using Bedrock from scratch**
* **Implement environment overrides**
* **Use tags for flexible access**
* **Resource-level permissions**
* **Custom authorization in NestJS using Bedrock**
* **Add Bedrock authorization to Angular routes**

---

# 🧭 **13. Concepts Explained (Visual)**

### **/concepts/**

* Scope hierarchy diagrams
* Permission inheritance flow
* Tag evaluation logic
* Resource evaluation graph
* API key access flow

---

# 🧪 **14. Testing & Debugging**

### **/testing/**

* **Using the permission evaluator**
* **Logging & debugging Deny results**
* **Testing overrides**
* **Simulating conditions**
* **Testing tag-based permissions**

---

# 🔒 **15. Security**

### **/security/**

* Tenant isolation
* Scope isolation
* API key security
* Permission logic sandboxing
* JSON logic security best practices
* Recommended audits

---

# ⚙️ **16. Self Hosting (Optional)**

### **/self-host/**

* Container deployment
* Environment variables
* Store integration
* Database setup
* Migrating from In-memory → Drizzle → Postgres

---

# 🧭 **17. Bedrock Cloud (Optional Section)**

### **/cloud/**

This is where you describe the SaaS version (if you choose to offer it).

---

# 🧩 **18. Changelog**

### **/changelog/**

Mintlify automatically supports this.

---

# 🟢 **19. Examples Library**

### **/examples/**

* JSON policy examples
* Example scope hierarchies
* Example permission models

---

# 🎨 **20. UI Guide (For Bedrock UI App)**

### **/ui/**

* Scopes UI
* Permissions UI
* Role builder
* Tag manager
* Resource registration
* API key management

---

# Want me to generate the actual page content next?

I can produce:

✅ Complete written pages
✅ Diagrams for scope/permission flow
✅ Example API docs in Mintlify format
✅ A sidebar.json structure you can paste directly

Just tell me where you want to start.

---

# 📋 **Summary of Changes Made**

The following updates were made based on analysis of the actual Bedrock codebase:

| Section | Change |
|---------|--------|
| **Platform Structure** | Added "Scope Type Hierarchy" covering `BedrockScopeTypeHierarchyEdge` |
| **Permissions** | Expanded "Permission Overrides" to document three-tier override system |
| **Permissions** | Updated "Conditional Permissions" with JSON Logic on role-permissions and overrides |
| **Resources** | Added "Resource Type Hierarchies" with inheritance patterns |
| **Resources** | Added "Resource Hierarchies" with cascade modes (`inherit`/`none`) |
| **Resources** | Replaced "Multi-Scope Resources" with "Resource Scope Links" (link types + metadata) |
| **Resources** | Added "Resource Collections" for dynamic resource grouping via match rules |
| **Resources** | Added "Resource Policies" for fine-grained allow/deny on resources/collections |
| **Tags** | Added "Tag Group Bindings" and expanded tag group properties |
| **API Keys** | Added warning about unverified implementation status |
| **New Section 11b** | Promoted "Delegation & Acting On Behalf" to first-class section |

### Key Interfaces Referenced:

- `BedrockScopeTypeHierarchyEdge`
- `BedrockScopeRoleOverride`
- `BedrockScopePermissionOverride`
- `BedrockScopeRolePermissionOverride`
- `BedrockResourceHierarchy` (with `CascadeModeEnum`)
- `BedrockResourceTypeHierarchy`
- `BedrockResourceScopeLink` (replaces `BedrockResourceScope`)
- `BedrockResourceCollection` (with `ResourceMatchDefinition`)
- `BedrockResourcePolicy` (with `PolicyEffectEnum`, `PolicyTargetKindEnum`)
- `BedrockTagGroupBinding`
- `BedrockEvaluateInput.onBehalfOf`
- `BedrockDecision.evaluatedPolicy` and `decidedByPolicy`
