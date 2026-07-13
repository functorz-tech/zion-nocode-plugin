# Permissions (RBAC + ABAC)

## Permission System Domain Knowledge
Zion uses RBAC (Role-Based Access Control) combined with ABAC (Attribute-Based Access Control) to secure data and actions. Permission configuration changes require "Sync Backend" to take effect.

### Core Concepts
Role: a named collection of users. A user can have multiple roles; their effective permissions are the union of all roles' permissions. Data Permission: controls what a role can read or write in the database at table, column, and row level. Action Permission: controls which APIs, Actionflows, AI agents, and payment operations a role can invoke.

### Activation and Predefined Roles
Always call `GET_ALL_ROLE_INFOS` before editing roles. When it reports that role permission is not activated:
- In BUILD or AUTO_ACCEPT mode, you MUST immediately call   `ACTIVATE_ROLE_PERMISSION`. BUILD uses the normal user-confirmation gate; AUTO_ACCEPT   applies it immediately.
- In RESEARCH mode, do not attempt a mutation; tell the user to look into permissions in   Settings → Permission.

`ACTIVATE_ROLE_PERMISSION` seeds the predefined roles in the project schema. Like every permission configuration change, it requires "Sync Backend" to affect the deployed backend.

After activation, these predefined roles exist and cannot be renamed or deleted:
- Logged-in User: automatically assigned to every authenticated user.
- Anonymous User: applies to unauthenticated users. Custom roles can then be created, renamed, and deleted with the permission tools, subject to the project's role limit.

### Data Permissions (three levels, configured coarse → fine)
1. Table-level: CRUD access + aggregate queries (count, sum, avg) for the entire table.
2. Column-level: CRUD access per field — field-level security.
3. Row-level: conditional filter rules evaluated at query time — e.g., "user can only update orders they created" (ABAC). Configured as "Advanced Filtering" on the table op.

### Action Permissions
Covers: Third-Party API configurations, Actionflows, AI agents, and payment operations. Resource grants use either allow-all or an explicit allow-list. On top of those grants, per-target conditional checks are available for Actionflows and AI agents. Payment permissions use either allow-all or an explicit payment-type/billing-method map.

### Role Assignment
- Manual: assign in the Permission Management UI.
- Automatic: use the Permissions node inside an Actionflow (e.g., grant VIP role after purchase). Actionflow-based role grants/revocations take effect immediately — no backend sync required.

### Troubleshooting Permission Errors
403 errors surface in runtime logs and client responses: ```json { "errorCode": 403, "extensions": { "classification": "TABLE_ACCESS" }, "message": "User 1 has no permission for SELECT on order" } ``` "User 1" means the user whose numeric ID ends in 1 (internal ID = 1000000000000001). Diagnostic steps:
1. Confirm the user has been assigned the correct role.
2. Verify the role's table/column/row permission configuration in Settings.
3. Confirm "Sync Backend" was run after the last permission config change.

### Least-Privilege Review Rules
- `allowAll` on `tpaPermission`, `actionflowPermission`, or `zAiPermission` allows   every resource in that block, including resources added later. Prefer explicit allow-lists   unless broad access is intentional.
- Effective permissions are the union of all assigned roles. A restrictive role does not   subtract grants supplied by a more permissive role.
- Newly granted table operations start with an always-true row condition. Narrow the   returned condition path before syncing whenever access should be row-scoped.
- A newly created custom role starts with the ztype-defined defaults: minimal table grants,   empty Actionflow and API allow-lists, all AI agents allowed, and default payment access.   Inspect and narrow those defaults for the project's policy.

### High-Level Business Logic & Core Architecture
When interacting with user projects, remember the core architecture of Zion:

1. **Design Layer (UI & Visual Layout):**
   - UI components on the canvas use a Flexbox-based layout model and Relative, Absolute,      or Fixed position types. Breakpoints control responsive design.
2. **Data Layer (PostgreSQL & Application State):**
   - Powered by a relational database schema. Tables use `snake_case` names for columns and      fields (e.g., `user_id`, `created_at`).
   - Dynamic query results bind directly to UI layouts through hierarchical data bindings      and visual formulas.
   - Page Variables hold page-lifecycle state; Client/Global Variables hold      application-wide state.
3. **Action Layer (Logic Workflows & Actions):**
   - Actionflows execute server-side workflow trees containing conditions, loops, API and      database operations, delays, and returns.
   - Security-sensitive logic, billing calculations, and state mutations should run in      Backend Actionflows rather than frontend event handlers.
4. **Publish & Deploy Operations:**
   - "Sync Backend" applies schema models, relations, backend APIs, Actionflows, and      permission configuration to the deployed backend.

### Architectural Guidance: Roles vs. Account Fields (RBAC vs. ABAC)
When users ask whether to create a Permission Role or add a field on the Account table, guide them with these principles:

1. Use a Permission System Role when defining static functional boundaries:
   - Controls overall capability (e.g., "Only AdminRole can run DeleteAccountActionFlow").
   - Restricts database tables (e.g., "Only FinanceRole can SELECT Transactions").
   - Implements column-level security (e.g., "Only HRRole can see the salary column").

2. Add a Field on the Account table when representing user-specific states:
   - Holds unique attributes (e.g., VIP status, user status, nickname, bio).

3. Use a Hybrid Model (Dynamic Filtering) to prevent "Role Explosion" for groups:
   - NEVER create separate roles for every department or store.
   - Instead, use the **Relation-First Pattern**:
     - Create a `1:n` relation from the context table (e.g., `department`) to `Account`;        this generates the non-editable `department_id` foreign-key column.
     - Create a `1:n` relation from the same context table to the business table (e.g.,        `sales_record`); this generates the matching foreign-key column.
   - Define one generic role (`Sales_Staff`) and configure a Row-Level Security filter      using the generated foreign-key columns:    Table.department_id == Account.department_id
   - Never manually declare foreign-key columns as raw integer fields; create relations.

### Editing roles with tools
Roles ARE editable via tools after initial activation. Read first: `GET_ALL_ROLE_INFOS` lists every role; `GET_ROLE_DETAIL` returns one role's grants and the schema paths of its row conditions.
- Role CRUD: `ADD_ROLES`, `UPDATE_ROLE`, `DELETE_ROLES`. Custom roles can be   renamed; predefined roles cannot be renamed or deleted.
- Per-block grants: `UPDATE_ROLE_TABLE_PERMISSION`, `UPDATE_ROLE_ACTION_FLOW_PERMISSION`,   `UPDATE_ROLE_ZAI_PERMISSION`, `UPDATE_ROLE_PAYMENT_PERMISSION`, and `UPDATE_ROLE_TPA_PERMISSION` for Third-Party API configurations. Account   insert/delete is unavailable, and `id` is pinned into table SELECT grants.
- Row conditions (ABAC): a newly granted table op gets an always-true row condition;   `SET_ROLE_PERMISSION_CHECK` seeds an always-true check for one Actionflow or AI agent.   Narrow conditions with the condition tools at the schema paths returned by   `GET_ROLE_DETAIL` or `SET_ROLE_PERMISSION_CHECK`.
Permission configuration changes require "Sync Backend" to take effect.

> Role permission (RBAC) must already be **activated** on the project — activation itself is editor-only (Settings → Permission Management) and every op below fails until then. Design the role + relation model first: for data isolation, model the one-to-many relations the row-level filters compare with `schema-table.md`; for 403 debugging, see `runtime-logs.md`.

## How to drive it (CLI only)

All commands are `"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" <verb>`. A long-lived daemon holds the in-memory CRDT schema session
between calls. **Edits do NOT go live until `project sync-backend`.**

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" whoami                                    # check auth; if needed: "${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" login
# create a NEW project (auto-pins it; its pre/post type-system state follows the account rollout):
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project create --projectName "My App"
# …or pin an EXISTING one (find its exId with "${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" projects search):
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project set-current --projectExId <exId>
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema load                               # warm the schema session
```

Operations run through one verb:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema tool-call --toolCalls '[{"name":"<TOOL_NAME>","args":{ ... }}]'
```
Each call is applied immediately — any resulting CRDT patch is uploaded. Batch several calls in one array; use `schema undo` to revert the last change.
A batch is all-or-nothing: when any call in the array fails, the whole batch's changes are discarded even though the other calls returned success — only the failing call's error is reported, so after a batch error re-read (`GET_*`) before assuming anything persisted.

## Operation reference (`schema tool-call` names)

| Intent | `name` | Required `args` |
|---|---|---|
| List roles | `GET_ALL_ROLE_INFOS` | — |
| Role detail (grants + condition paths) | `GET_ROLE_DETAIL` | `roleId` |
| Create roles (minimal grants) | `ADD_ROLES` | `items` |
| Rename a role | `UPDATE_ROLE` | `roleId` |
| Delete roles | `DELETE_ROLES` | `roleIds` |
| Grant/revoke table ops (per-column) | `UPDATE_ROLE_TABLE_PERMISSION` | `roleId`, `tableDisplayName` |
| Allow-list action flows | `UPDATE_ROLE_ACTION_FLOW_PERMISSION` | `roleId` |
| Allow-list AI agents | `UPDATE_ROLE_ZAI_PERMISSION` | `roleId` |
| Allow payment actions | `UPDATE_ROLE_PAYMENT_PERMISSION` | `roleId` |
| Allow-list TPA configs | `UPDATE_ROLE_TPA_PERMISSION` | `roleId` |
| Seed a conditional check (flow/agent) | `SET_ROLE_PERMISSION_CHECK` | `category`, `roleId`, `targetId` |

Newly granted table operations get an always-true row condition and `SET_ROLE_PERMISSION_CHECK`
seeds an always-true check — both return schema paths; narrow them with the condition tools at those
paths (`data-binding.md`). `GET_ROLE_DETAIL` echoes every existing condition's schema path.

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `ADD_ROLES`

Create custom roles with the editor's minimal-grant defaults (extend them with the set_*_permission tools). If GET_ALL_ROLE_INFOS reports "not activated", call ACTIVATE_ROLE_PERMISSION first.
- `items` *(required)*: `array<{description?: string, name: string}>` — Custom roles to create. A new role starts with no table operations granted (except the account table's basic profile read/update), empty action-flow and API allow-lists, all AI agents allowed and the default payment permission — grant more via the UPDATE_ROLE_*_PERMISSION tools. Requires role permission (RBAC) to already be activated: when GET_ALL_ROLE_INFOS reports that it is not activated, call ACTIVATE_ROLE_PERMISSION first.

### `UPDATE_ROLE_TABLE_PERMISSION`

Grant or revoke one role's operations on one table, with per-operation column sets. Newly granted operations get an always-true row condition — narrow it via the condition tools at the schema paths from GET_ROLE_DETAIL.
- `aggregate`: `{columns?: array<string>, enabled?: boolean}` — Aggregation permission; only numeric columns can be granted.
- `count`: `{columns?: array<string>, enabled?: boolean}` — Row-count permission (row-level; takes no columns).
- `delete`: `{columns?: array<string>, enabled?: boolean}` — Row-delete permission (row-level; takes no columns). Not available on the account table.
- `insert`: `{columns?: array<string>, enabled?: boolean}` — Row-insert permission. Not available on the account table. Auto-generated columns (serial ids, created_at/updated_at, formula columns) cannot be granted.
- `roleId` *(required)*: `string` — The uuid of the role whose table permission to change.
- `select`: `{columns?: array<string>, enabled?: boolean}` — Row-read permission. Newly granted operations get an always-true row condition; narrow it via the condition tools at the schema paths from GET_ROLE_DETAIL.
- `tableDisplayName` *(required)*: `string` — Display name of the table (from GET_ALL_TABLE_DISPLAY_NAMES).
- `update`: `{columns?: array<string>, enabled?: boolean}` — Row-update permission. Auto-generated columns (serial ids, created_at/updated_at, formula columns) cannot be granted.

### `SET_ROLE_PERMISSION_CHECK`

Seed (or reset) a role's conditional check on one action flow / AI agent: writes an always-true condition and returns its schema path — then narrow it via the condition tools at that path. The check gates calls to the target on top of the allow-list.
- `category` *(required)*: `enum(ACTION_FLOW|ZAI)` — Which permission block the check gates: ACTION_FLOW or ZAI.
- `roleId` *(required)*: `string` — The uuid of the role the check belongs to.
- `targetId` *(required)*: `string` — The action-flow id / AI-agent config id the check applies to. The check gates calls to that target for this role on top of the allow-list. This tool seeds the target's check as an always-true condition and returns its checkSchemaPath — narrow the condition via the condition tools (INSERT_CONDITION_BOOL_EXP etc.) at that path. Calling it again for the same target resets the check back to always-true.

Then ship:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema validate && "${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
