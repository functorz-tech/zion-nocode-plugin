# Permissions (RBAC + ABAC)

## Permission System Domain Knowledge
Zion uses RBAC (Role-Based Access Control) combined with ABAC (Attribute-Based Access Control) to secure data and actions. Permission changes require "Sync Backend" to take effect.

### Core Concepts
Role: a named collection of users. A user can have multiple roles; their effective permissions are the union of all roles' permissions. Data Permission: controls what a role can read/write in the database (table, column, row). Action Permission: controls which APIs, Actionflows, AI functions, and payment operations a role can invoke.

### Built-in Roles (always present, cannot be deleted)
- Logged-in User: automatically assigned to every authenticated user.
- Anonymous User: applies to unauthenticated (not logged in) users. Custom roles are created in Settings → Permission Management. Role names are immutable once deployed to the backend — advise users to name carefully. Plan limits: Free = 0 custom roles, Basic = 1, Pro = 10.

### Data Permissions (three levels, configured coarse → fine)
1. Table-level: CRUD access + aggregate queries (count, sum, avg) for the entire table.
2. Column-level: CRUD access per field — field-level security.
3. Row-level: conditional filter rules evaluated at query time — e.g., "user can only update orders they created" (ABAC). Configured as "Advanced Filtering" on the table op.

### Action Permissions
Covers: APIs, Actionflows, AI functions, payment operations. All except payments support "Advanced Filtering" for fine-grained ABAC rules (e.g., "Actionflow input params must not be empty").

### Role Assignment
- Manual: assign in the Permission Management UI.
- Automatic: use the Permissions node inside an Actionflow (e.g., grant VIP role after purchase). Actionflow-based role grants/revocations take effect immediately — no backend sync required.

### Troubleshooting Permission Errors
403 errors surface in runtime logs and client responses: ```json { "errorCode": 403, "extensions": { "classification": "TABLE_ACCESS" }, "message": "User 1 has no permission for SELECT on order" } ``` "User 1" means the user whose numeric ID ends in 1 (internal ID = 1000000000000001). Diagnostic steps:
1. Confirm the user has been assigned the correct role.
2. Verify the role's table/column/row permission configuration in Settings.
3. Confirm "Sync Backend" was run after the last permission config change.

### Security Flaws & Vulnerabilities in PermissionConfig / RoleConfig (Critical Analysis)
When analyzing permission structures, watch out for the following critical security flaws in how `com.functorz.ztype.typesystem.schema.PermissionConfig` and `RoleConfig` are defined in the project:

1. **The "allowAll" Bypass Flaw (`tpaPermission`, `actionflowPermission`, `zAiPermission`):**
   - The `allowAll` boolean field acts as an all-or-nothing bypass. If `allowAll == true`, it completely overrides any specific resource limits (like `allowedApiIds` or `allowedActionflowIds`). If a developer accidentally enables this on custom roles (or the built-in Anonymous User), it instantly exposes all APIs and backend Actionflows to exploitation, violating the Principle of Least Privilege.
2. **Union-Based Privilege Escalation:**
   - A user's effective permissions are the union of all assigned roles. If a user is assigned a restrictive role and a permissive role, the permissive rules will fully override the restrictions. This makes privilege escalation extremely easy if multi-role assignment is not carefully audited.
3. **Hardcoded Anonymous Role UUID (`ANONYMOUS_ROLE_UUID`):**
   - The anonymous role UUID is hardcoded as a global constant (`"f65a575e-c93a-4b1e-980e-96c3bcf81a70"`). Any runtime role checking that relies solely on matching this UUID (rather than robust session authentication) presents a spoofing risk if session headers can be tampered with.
4. **Fail-Open Column-Level Permissions & Dynamic Filtering:**
   - If a query action is permitted but columns (`columns: List<String>`) or row filters (`filter: BoolExp`) are missing or improperly configured, the system might default to exposing sensitive new fields when schema models are modified or updated.
5. **No Security Policy Auditing in RoleConfigValidator:**
   - The validation engine (`RoleConfigValidator.kt`) only verifies structural integrity (e.g., whether table IDs or API IDs exist). It completely lacks security linter/auditing logic, meaning it will silently accept dangerous configurations (e.g., Anonymous User having delete permissions or `allowAll = true` on actionflows) without issuing any warnings or errors.

### High-Level Business Logic & Core Architecture
When interacting with user projects, remember the core three-pillar business architecture of Zion:

1. **Design Layer (UI & Visual Layout):**
   - UI components (containers, inputs, text elements, and navigation bars) on the canvas are structured responsive using a Flexbox-based layout model and position types (Relative, Absolute, Fixed). Breakpoints control responsive design.
2. **Data Layer (PostgreSQL & Application State):**
   - Powered by a relational database schema. Tables use `snake_case` naming conventions for columns and fields (e.g., `user_id`, `created_at`).
   - Dynamic query results are bound directly to UI layouts via Hierarchical Data Bindings and visual formulas.
   - Client-side states are driven by variables: Page Variables (scoped to current page lifecycle) and Client/Global Variables (scoped application-wide).
3. **Action Layer (Logic Workflows & Actions):**
   - Functional logic is executed using Actionflows (sequential workflow trees containing Branch/Condition nodes, Loop/ForEach nodes, API nodes, DB action nodes, Delay nodes, and Return nodes).
   - High-security logic, mathematical billing, and state mutations should always run in Backend Actionflows rather than frontend event handlers to prevent client manipulation (Zero-Trust Client model).
4. **Publish & Deploy Operations:**
   - "Sync Backend" compiles and builds the schema models, relations, backend APIs, Actionflows, and compiles/bundles optimized preview/production client applications.

### Architectural Guidance: Roles vs. Account Fields (RBAC vs. ABAC)
When users ask whether to create a new Permission Role or add a custom Field on the Account table, guide them with these engineering principles:

1. Use a Permission System Role when defining static functional boundaries:
   - Controls overall capability (e.g., "Only AdminRole can run DeleteAccountActionFlow").
   - Restricts entire database tables (e.g., "Only FinanceRole can SELECT from Transactions table").
   - Implements column-level security (e.g., "Only HRRole can see the salary column").

2. Add a Field on the Account table when representing user-specific states:
   - Holds unique attributes (e.g., VIP status, user status, nickname, bio).

3. Use a Hybrid Model (Dynamic Filtering) to prevent "Role Explosion" for groups:
   - NEVER create separate roles for every department/store (e.g., do NOT make Sales_Dept_A, Sales_Dept_B).
   - Instead, use the **Relation-First Pattern**:
     - Create a `1:n` relation from your context table (e.g., `department`) to the `Account` table. This auto-generates a non-editable `department_id` FK column on the `Account` table.
     - Create a `1:n` relation from your context table to the data table (e.g., `sales_record`). This auto-generates `department_id` FK column on the data table.
   - Define a single generic role (`Sales_Staff`) and configure a Row-Level Security filter using the auto-generated FK columns:    Table.department_id == Account.department_id
   - Never manually declare foreign key columns (like `department_id` as a raw integer/bigint field), as the system forbids manual foreign key columns and requires relations.

### Scope Note
Permission configuration (creating roles, setting table/column/row permissions) is done in the editor Settings panel, not via agent tools. The agent can analyze permission errors in logs and advise on configuration — it cannot apply permission changes directly.

## Consultant mode (no direct edits)

The CLI cannot change permission configuration — it is editor-only (Settings → Permission Management). Use this capability to design the role + relation model, then give concrete editor steps. For data isolation, model the one-to-many relations the row-level filters compare with `schema-table.md`; for 403 debugging, see `runtime-logs.md`.

Context helpers (read-only):

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project metadata
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" logs search --customQueryCondition '<es-condition>'
```
