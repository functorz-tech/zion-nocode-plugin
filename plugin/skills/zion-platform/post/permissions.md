# Permissions (RBAC + ABAC)

## Permission System Domain Knowledge
Zion secures data and actions with RBAC (roles) plus ABAC (per-row conditions). Every tool in this plugin edits the project schema only — the change reaches the deployed backend when "Sync Backend" runs, which is the last step of any permission task.

### Model
- **Role**: a named collection of users. Two are predefined and cannot be renamed or deleted: **Logged-in User** (held by every authenticated user) and **Anonymous User** (unauthenticated requests). Custom roles can then be created, renamed, and deleted, up to the project's role limit.
- **Data Permission**: what a role may read or write, at table, column and row level.
- **Action Permission**: which workspace HTTP APIs, Actionflows, AI agents and payment operations a role may invoke.
- Each action block is allow-all or an explicit allow-list. On top of those grants, per-target conditional checks are available for Actionflows and AI agents; payments take allow-all or a payment-type/billing-method map.
- **Roles only ever add.** A user's effective permissions are the union of every role they hold; a restrictive role never subtracts what a permissive one granted. To take access away, narrow or unassign the permissive role — never "add a limited role".

### Activate first
Always call `GET_ALL_ROLES_INFO` before editing roles. When it reports that role permission is not activated:
- In BUILD or AUTO_ACCEPT mode, you MUST immediately call `ACTIVATE_ROLE_PERMISSION`, which seeds the predefined roles. BUILD uses the normal user-confirmation gate; AUTO_ACCEPT applies it immediately.
- In RESEARCH mode, do not attempt a mutation; tell the user to look into permissions in 设置 → 权限管理.

### Grant
Data permissions are configured coarse → fine:
1. **Table**: which operations (select / insert / update / delete / count / aggregate) the role may run.
2. **Column**: which fields each operation covers.
3. **Row**: a per-operation condition — a filter over existing rows for select, update and delete, a check on the incoming row for insert. Each one starts always-true, so a fresh grant is open until narrowed.

Read before writing: `GET_ALL_ROLES_INFO`, then `GET_ROLE_DETAIL` for one role's grants, the schema path of each operation's row condition, and its `hasCustomCondition` flag. Edit with `ADD_ROLES`, `UPDATE_ROLE`, `DELETE_ROLES` and the per-block setters `UPDATE_ROLE_TABLE_PERMISSION`, `UPDATE_ROLE_ACTION_FLOW_PERMISSION`, `UPDATE_ROLE_ZAI_PERMISSION`, `UPDATE_ROLE_PAYMENT_PERMISSION`, `UPDATE_ROLE_API_PERMISSION` for workspace HTTP APIs. Account insert/delete is unavailable, and `id` is pinned into every table SELECT grant.

`allowAll` on `apiPermission`, `actionflowPermission` or `zAiPermission` also covers resources added later — prefer explicit allow-lists unless broad access is intentional.

**Role, or a field on Account?** Use a role for static functional boundaries: who may run a flow, read a table, see a column. Add a field on the Account table for per-user state: VIP status, nickname, bio.

**Never create one role per group.** For departments, stores, tenants or any other segmentation, use the relation-first pattern: create a `1:n` relation from the context table (e.g. `department`) to `Account`, and a second one from that same table to the business table (e.g. `sales_record`). Both foreign-key columns are generated and non-editable — never declare them as raw integer fields. Then define one generic role (`Sales_Staff`) and compare those two columns in its row conditions.

Simple single-table CRUD may run directly from the frontend when table, column, and row permissions fully express the authorization policy — do not wrap it in an Actionflow. Reach for a Backend Actionflow only when the rule cannot be expressed that way at all: it needs server-held secrets, cross-table atomicity, or multi-step state transitions.

### Row conditions (ABAC)
Narrowing those conditions is a two-plugin job: this plugin has no tool that edits one — they live in the **bindings** plugin, so you must `load_plugin(["bindings"])` to finish. The check `SET_ROLE_PERMISSION_CHECK` seeds for an Actionflow or AI agent works the same way.

Each operation has its own condition node and its own row branch to compare against, so reusing the select recipe on another op fails:
- `select` → path ends `/select/filter` → the branch holding the row being read
- `insert` → path ends `/insert/check`, not `filter` → the branch holding the row being inserted
- `update` → path ends `/update/filter` → the branch holding the row as stored, or the one holding the incoming values
- `delete` → path ends `/delete/filter` → the branch holding the row being deleted
- `count` / `aggregate` → no row branch exists, only the logged-in user and the current time, so they cannot be row-scoped. Whenever you narrow `select`, send `count` and `aggregate` `{enabled: false}` too, unless a whole-table count is meant to be public: a row-scoped `select` beside an open `count` still tells any user how many rows everyone else has.

Narrowing one condition:
1. `GET_ROLE_DETAIL` (or the result of `SET_ROLE_PERMISSION_CHECK`) → that operation's condition schema path.
2. `INSERT_CONDITION_BOOL_EXP` at that path → returns a `conditionSchemaPath`. Copy it verbatim; never hand-build one.
3. `GET_EXPRESSION_CONDITION_OPERATORS` at that path, then `UPDATE_EXPRESSION_CONDITION_OPERATOR` with an operator copied verbatim. The `target` and `value` operand slots do not exist until the operator is set — binding first fails with "Cannot find type definition under path …".
4. Fill both operands at `<conditionSchemaPath>/target` and `<conditionSchemaPath>/value`, addressed by `pathInHierarchicalMenu`. Read that path — branch label included — from `BROWSE_DATA_BINDING_OPTIONS` at the operand's own schema path and copy it verbatim; never hand-build one. Relations traverse, so a child table can compare against its parent's owner — that path is four segments deep: context root, row branch, relation, column. Every segment is a label from the tree, including the root.
5. Delete the always-true placeholder the grant started with: `DELETE_CONDITION_BOOL_EXP` at `<path>/_and/[0]`. Harmless at runtime, but the editor renders it as an empty condition row, so the user sees a half-configured rule.
6. Shape multi-clause predicates with `NEST_CONDITION_BOOL_EXP`, `TOGGLE_CONDITION_BOOL_EXP_AND_OR`, `TOGGLE_CONDITION_BOOL_EXP_NOT`.
7. Sync Backend.

Example — "a staff member only ever touches their own department's sales records" — after granting `Sales_Staff` select, insert, update and delete on `sales_record`, configure four conditions, each comparing that operation's own row branch `department_id` against the logged-in user's `department_id`. Configuring only select leaves the writes unrestricted.

A batch of tool calls is all-or-nothing: one bad call discards every change in that batch. Send exploratory reads on their own — especially the first `BROWSE_DATA_BINDING_OPTIONS` for an operation whose context branch you have not seen — then batch the writes.

### Assign the role
A custom role does nothing until an account holds it. Two ways:
- Manual: assign in the editor, under 设置 → 权限管理.
- Automatic: the Permissions node inside an Actionflow (e.g. grant VIP after purchase). These grants and revocations take effect immediately — no backend sync required.

### Definition of done
A permission task is finished only when all of these hold. Check them before reporting:
1. Every operation you enabled shows `hasCustomCondition: true` in `GET_ROLE_DETAIL`, or is deliberately open.
2. `GET_ROLE_DETAIL` on **Anonymous User** shows no unintended grant on the tables you touched. A newly created table lands in that role automatically: observed on a deployed project, an unauthenticated request could read every row, and after only SELECT was revoked it could still INSERT.
3. Any custom role you created is actually assigned to someone.
4. "Sync Backend" has run.
If you stop short of any of these, say so explicitly and name which grants are still open. Never report the permission work as complete.

### Troubleshooting 403
403s surface in runtime logs and client responses:
```json
{
  "errorCode": 403,
  "extensions": { "classification": "TABLE_ACCESS" },
  "message": "User 1 has no permission for SELECT on order"
}
```
"User 1" is the user whose internal id ends in 1 (1000000000000001). Then:
1. Confirm the user holds the role you expect.
2. Read that role with `GET_ROLE_DETAIL`: is the operation enabled, is the column inside its grant, and does its row condition match the row the user expected?
3. Confirm "Sync Backend" ran after the last permission change.

> Role permission (RBAC) must already be **activated** on the project — activation itself is editor-only (Settings → Permission Management) and every op below fails until then. Design the role + relation model first: for data isolation, model the one-to-many relations the row-level filters compare with `schema-table.md`; for 403 debugging, see `runtime-logs.md`.

## How to drive it (CLI only)

All commands are `npx -y zion-mcp@2.3.0 <verb>`. A long-lived daemon holds the in-memory CRDT schema session
between calls. **Edits do NOT go live until `project sync-backend`.**

```bash
npx -y zion-mcp@2.3.0 whoami                                    # check auth; if needed: npx -y zion-mcp@2.3.0 login
# create a NEW project (auto-pins it; its pre/post type-system state follows the account rollout):
npx -y zion-mcp@2.3.0 project create --projectName "My App"
# …or pin an EXISTING one (find its exId with npx -y zion-mcp@2.3.0 projects search):
npx -y zion-mcp@2.3.0 project set-current --projectExId <exId>
npx -y zion-mcp@2.3.0 schema load                               # warm the schema session
```

Operations run through one verb:

```bash
npx -y zion-mcp@2.3.0 schema tool-call --toolCalls '[{"name":"<TOOL_NAME>","args":{ ... }}]'
```
Each call is applied immediately — any resulting CRDT patch is uploaded. Batch several calls in one array; use `schema undo` to revert the last change.
A batch is all-or-nothing: when any call in the array fails, the whole batch's changes are discarded even though the other calls returned success — only the failing call's error is reported, so after a batch error re-read (`GET_*`) before assuming anything persisted.

## Operation reference (`schema tool-call` names)

| Intent | `name` | Required `args` |
|---|---|---|
| List roles | `GET_ALL_ROLES_INFO` | — |
| Role detail (grants + condition paths) | `GET_ROLE_DETAIL` | `roleId` |
| Create roles (minimal grants) | `ADD_ROLES` | `items` |
| Rename a role | `UPDATE_ROLE` | `roleId` |
| Delete roles | `DELETE_ROLES` | `roleIds` |
| Grant/revoke table ops (per-column) | `UPDATE_ROLE_TABLE_PERMISSION` | `roleId`, `tableDisplayName` |
| Allow-list action flows | `UPDATE_ROLE_ACTION_FLOW_PERMISSION` | `roleId` |
| Allow-list AI agents | `UPDATE_ROLE_ZAI_PERMISSION` | `roleId` |
| Allow payment actions | `UPDATE_ROLE_PAYMENT_PERMISSION` | `roleId` |
| Allow-list API workspaces | `UPDATE_ROLE_API_PERMISSION` | `roleId` |
| Seed a conditional check (flow/agent) | `SET_ROLE_PERMISSION_CHECK` | `category`, `roleId`, `targetId` |

Newly granted table operations get an always-true row condition and `SET_ROLE_PERMISSION_CHECK`
seeds an always-true check — both return schema paths; narrow them with the condition tools at those
paths (`data-binding.md`). `GET_ROLE_DETAIL` echoes every existing condition's schema path.

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `ADD_ROLES`

Create custom roles with the editor's minimal-grant defaults (extend them with the set_*_permission tools). If GET_ALL_ROLES_INFO reports "not activated", call ACTIVATE_ROLE_PERMISSION first.
- `items` *(required)*: `array<{description?: string, name: string}>` — Custom roles to create. A new role starts with no table operations granted (except the account table's basic profile read/update), empty action-flow and API allow-lists, all AI agents allowed and the default payment permission — grant more via the UPDATE_ROLE_*_PERMISSION tools. Requires role permission (RBAC) to already be activated: when GET_ALL_ROLES_INFO reports that it is not activated, call ACTIVATE_ROLE_PERMISSION first.

### `UPDATE_ROLE_TABLE_PERMISSION`

Grant or revoke one role's operations on one table, with per-operation column sets. `tableDisplayName` is the display name from the database plugin's read tools, not the snake_case table name. `count` and `aggregate` cannot be row-scoped, so disable them whenever select is.
- `aggregate`: `{columns?: array<string>, enabled?: boolean}` — Aggregation permission; only numeric columns are grantable, others are dropped from the column set.
- `count`: `{columns?: array<string>, enabled?: boolean}` — Row-count permission (row-level; takes no columns).
- `delete`: `{columns?: array<string>, enabled?: boolean}` — Row-delete permission (row-level; takes no columns). Not available on the account table.
- `insert`: `{columns?: array<string>, enabled?: boolean}` — Row-insert permission. Not available on the account table. Auto-generated columns (serial ids, created_at/updated_at, formula columns) are not grantable and are dropped from the column set.
- `roleId` *(required)*: `string` — The uuid of the role whose table permission to change.
- `select`: `{columns?: array<string>, enabled?: boolean}` — Row-read permission. Newly granted operations get an always-true row condition; narrow it via the condition tools at the schema paths from GET_ROLE_DETAIL.
- `tableDisplayName` *(required)*: `string` — Display name of the table (from GET_ALL_TABLE_DISPLAY_NAMES).
- `update`: `{columns?: array<string>, enabled?: boolean}` — Row-update permission. Auto-generated columns (serial ids, created_at/updated_at, formula columns) are not grantable and are dropped from the column set.

### `SET_ROLE_PERMISSION_CHECK`

Seed a role's conditional check on one action flow / AI agent, on top of the allow-list: writes an always-true condition and returns its schema path, so it gates nothing until narrowed. Calling it again for the same target resets the check back to always-true.
- `category` *(required)*: `enum(ACTION_FLOW|ZAI)` — Which permission block the check gates: ACTION_FLOW or ZAI.
- `roleId` *(required)*: `string` — The uuid of the role the check belongs to.
- `targetId` *(required)*: `string` — The action-flow id / AI-agent config id the check applies to. The check gates calls to that target for this role on top of the allow-list. This tool seeds the target's check as an always-true condition and returns its checkSchemaPath — narrow the condition via the condition tools (INSERT_CONDITION_BOOL_EXP etc.) at that path. Calling it again for the same target resets the check back to always-true.

Then ship:

```bash
npx -y zion-mcp@2.3.0 schema validate && npx -y zion-mcp@2.3.0 project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
