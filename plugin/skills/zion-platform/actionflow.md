# Action flows (server-side workflows)

## Actionflow Domain Knowledge
Actionflows are server-side workflows for business logic. They run entirely on the server; the frontend triggers them via "Request – Actionflow" component actions.

### Execution Modes
Synchronous: caller blocks for the result; ACID (any node error rolls back all DB writes). Asynchronous: the caller gets a task handle instead of blocking and retrieves the result later — the flow still computes and returns its declared output; only the failing node's DB writes roll back; required for AI agents and video generation.

Both modes share a per-flow total timeout (`ACTION_FLOW_TIMEOUT_MILLISECONDS`): 15 s on free-tier servers, 10 min on paid tiers. The budget is for the entire flow, not per node.

### Triggers
What fires a flow. Four kinds: Manual (a UI action — no config), scheduled (cron), database-change event, and webhook. Scheduled and database-change triggers are tool-managed: inspect with `GET_ALL_SCHEDULED_TRIGGER_INFOS` / `GET_ALL_DB_TRIGGER_INFOS`, create with `ADD_SCHEDULED_TRIGGERS` (a Quartz cron — set a future `endInstant` or it never fires) and `ADD_DB_TRIGGERS` (a table + INSERT/UPDATE/DELETE), each pointing at a flow id from `GET_ALL_ACTION_FLOW_INFOS` (update/delete variants too). Manual triggers need no setup; webhook triggers are editor-only (no tool) — say so if asked to add one.

### Node Types
These are the fixed built-in node types — the only values accepted for a node's `type`:
Database: query/insert/update/delete on a table. Call API: invoke a configured API; bind response to output params. Run AI: execute a ZAI agent (async only). Run Actionflow: call another flow (sync cannot call async). Set Variable: assign values to declared flow-level variables. Permissions: grant or revoke roles for a user. Run Code: execute custom JavaScript. Condition: branch logic evaluated left to right. Loop: iterate over a list; inner nodes access item data.

Everything else is a **preset integration node** (the `TEMPLATE_CODE` node type) from a server-managed catalog that varies by deployment — including getting the current user's ID / WeChat OpenID / Access Token, file/media conversion to Zion native types, SMS, and video/AI generation. There is NO built-in `CURRENT_USER` / `FILES` / `SMS` node type; do not pass those to `ADD_ACTION_FLOW_NODE`. The current catalog (each template's `templateCodeId` plus its input/output types) is pre-fetched for you in the "Preset Integration Node Catalog" section below — look up the template there and insert it with `ADD_ACTION_FLOW_NODE` using the `TEMPLATE_CODE` node type and its `templateCodeId`. Call `list_node_templates` only to refresh that list.

### Choosing How to Express Logic: nodes → formulas → code
Three tiers, in order of preference:
1. **Visual nodes** for orchestration — query / branch / iterate / write / call (Database, Condition, Loop, Set Variable, Call API, Run Actionflow). They take part in the flow's transaction, surface individually in logs and the edit-time error collector, and stay user-editable.
2. **Formula bindings** for value-level computation. A node value — a Set Variable value, a mutation column, a condition predicate — can be a FORMULA binding built with the bindings plugin, and formulas cover far more than people expect: text and regex (extract / match / replace, substring, split, concat), math, date/time arithmetic and formatting, array aggregation / mapping / filtering, JSON value access, and type casts. Prefer a formula over code whenever you are computing a single value.
3. **Run Code** only for genuinely procedural logic that neither a node nor a formula operator expresses — e.g. cryptographic hashing / signature building, multi-step algorithms with intermediate state, or assembling / parsing a complex nested payload. Keep it small and fed by node inputs; never collapse a whole flow into one Run Code node.

### Variables
Declared at flow level, accessible by all nodes, assigned with "Set Variable" node.

### Inputs & Outputs
Inspect a flow first with `GET_ALL_ACTION_FLOW_INFOS` / `GET_ACTION_FLOW_DETAIL`. Declare typed **input params** with `ADD_ACTION_FLOW_INPUT_PARAMS`; set each param's type to a value copied verbatim from `GET_ACTION_FLOW_SELECTABLE_TYPES` (never hand-build the type string).
A flow returns a **set of named output fields**: add them with `ADD_ACTION_FLOW_OUTPUT_FIELDS`
(one entry per field, each with its own type) and remove them with
`DELETE_ACTION_FLOW_OUTPUT_FIELDS`.

### Building & Editing a Flow
Create flows with `ADD_ACTION_FLOWS` (each seeds an empty FLOW_START → FLOW_END), then read node and structure ids from `GET_ACTION_FLOW_DETAIL` before editing. Add a node after another with `ADD_ACTION_FLOW_NODE`, reorder with `MOVE_ACTION_FLOW_NODE`, remove with `DELETE_ACTION_FLOW_NODES`, and branch a Condition node with `ADD_ACTION_FLOW_BRANCH_ITEM`. Edit a node's name or type-scalar config with `UPDATE_ACTION_FLOW_NODE`, but set its data bindings (values, conditions, data sources, mutation fields) with the bindings plugin at the node's schema path — never inline. Declare flow variables with `ADD_ACTION_FLOW_GLOBAL_VARIABLES` and assign them in a Set Variable node with `ADD_GLOBAL_VARIABLES_NODE_TARGETS`. A Run Code node's `args.<name>` input slots are managed with `ADD_CUSTOM_CODE_NODE_INPUT` (rename/delete variants); its result type is declared with `SET_CUSTOM_CODE_NODE_OUTPUT_TYPE` (or several named results with `ADD_CUSTOM_CODE_NODE_OUTPUT_VALUE`) so downstream nodes can bind to it; fill the code body with `generate_code`. An update or delete Database node is seeded with an always-true filter that matches **every** row — narrow it with the bindings plugin's request filters before syncing, or the write hits the whole table.

### Error Handling
Synchronous: all DB changes roll back on error. Asynchronous: only the failing node's DB changes roll back.

### Versioning
Each save creates a new version. "Sync Backend" required after editing for changes to take effect in production.

### JavaScript Sandbox (Run Code node)
Custom Code Blocks run in a synchronous GraalJS server sandbox. No async/await, no require(), no browser APIs. All platform interactions go through the global `context` object: context.getArg(key)                              — retrieve flow input context.setReturn(key, val)                      — pass output to downstream nodes context.runGql(query, variables, options)        — execute DB operations context.callThirdPartyApi(apiId, params)         — invoke a configured REST API context.callActionFlow(flowId, ver, args)        — run a sub-flow synchronously context.createActionFlowTask(flowId, ver, args)  — trigger a sub-flow async context.getSsoUserInfo()                         — get authenticated user profile context.uploadMedia(url, headers)                — stream remote image to asset server context.log(message)                             — emit to Log Service

## How to drive it (CLI only)

All commands are `"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" <verb>`. A long-lived daemon holds the in-memory CRDT schema session
between calls. **Edits do NOT go live until `project sync-backend`.**

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" whoami                                    # check auth; if needed: "${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" login
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project set-current --projectExId <exId>  # pin the project ("${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" projects search to find it)
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema load                               # warm the schema session
```

Operations run through one verb:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema tool-call --toolCalls '[{"name":"<TOOL_NAME>","args":{ ... }}]'
```
Each call is applied immediately — any resulting CRDT patch is uploaded. Batch several calls in one array; use `schema undo` to revert the last change.

## Operation reference (`schema tool-call` names)

| Intent | `name` | Required `args` |
|---|---|---|
| List flows | `GET_ALL_ACTION_FLOW_INFOS` | — |
| Flow detail (nodes, ids) | `GET_ACTION_FLOW_DETAIL` | `actionFlowId` |
| Selectable I/O types | `GET_ACTION_FLOW_SELECTABLE_TYPES` | — |
| Data/vars in scope at a node path | `GET_ACTION_FLOW_CONTEXT_INFO` | `schemaPath` |
| Create flows | `ADD_ACTION_FLOWS` | `items` |
| Update a flow (name/async/timeout) | `UPDATE_ACTION_FLOW` | `actionFlowId` |
| Delete flows | `DELETE_ACTION_FLOWS` | `actionFlowIds` |
| Add input params | `ADD_ACTION_FLOW_INPUT_PARAMS` | `actionFlowId`, `items` |
| Update input params | `UPDATE_ACTION_FLOW_INPUT_PARAMS` | `actionFlowId`, `items` |
| Delete input params | `DELETE_ACTION_FLOW_INPUT_PARAMS` | `actionFlowId`, `names` |
| Add a node | `ADD_ACTION_FLOW_NODE` | `actionFlowId`, `afterNodeId`, `node` |
| Update a node config | `UPDATE_ACTION_FLOW_NODE` | `actionFlowId`, `nodeId` |
| Delete nodes | `DELETE_ACTION_FLOW_NODES` | `actionFlowId`, `nodeIds` |
| Reorder a node | `MOVE_ACTION_FLOW_NODE` | `actionFlowId`, `afterNodeId`, `nodeId` |
| Add a branch (Condition node) | `ADD_ACTION_FLOW_BRANCH_ITEM` | `actionFlowId`, `branchSeparationId` |
| Declare flow variables | `ADD_ACTION_FLOW_GLOBAL_VARIABLES` | `actionFlowId`, `items` |
| Update flow variables | `UPDATE_ACTION_FLOW_GLOBAL_VARIABLES` | `actionFlowId`, `items` |
| Delete flow variables | `DELETE_ACTION_FLOW_GLOBAL_VARIABLES` | `actionFlowId`, `variableKeys` |
| Assign Set-Variable node targets | `ADD_GLOBAL_VARIABLES_NODE_TARGETS` | `actionFlowId`, `nodeId`, `variableKeys` |
| Remove Set-Variable node targets | `DELETE_GLOBAL_VARIABLES_NODE_TARGETS` | `actionFlowId`, `nodeId`, `variableKeys` |
| Add a Run Code input | `ADD_CUSTOM_CODE_NODE_INPUT` | `actionFlowId`, `name`, `nodeId` |
| Rename a Run Code input | `RENAME_CUSTOM_CODE_NODE_INPUT` | `actionFlowId`, `newName`, `nodeId`, `oldName` |
| Delete a Run Code input | `DELETE_CUSTOM_CODE_NODE_INPUT` | `actionFlowId`, `name`, `nodeId` |
| Set a Run Code output type | `SET_CUSTOM_CODE_NODE_OUTPUT_TYPE` | `actionFlowId`, `nodeId`, `type` |
| Clear a Run Code output type | `CLEAR_CUSTOM_CODE_NODE_OUTPUT_TYPE` | `actionFlowId`, `nodeId` |
| Add a Run Code output value | `ADD_CUSTOM_CODE_NODE_OUTPUT_VALUE` | `actionFlowId`, `name`, `nodeId`, `type` |
| Update a Run Code output value | `UPDATE_CUSTOM_CODE_NODE_OUTPUT_VALUE` | `actionFlowId`, `name`, `nodeId` |
| Delete a Run Code output value | `DELETE_CUSTOM_CODE_NODE_OUTPUT_VALUE` | `actionFlowId`, `name`, `nodeId` |
| Rename a node/slot (by path) | `SET_DISPLAY_NAME` | `displayName`, `schemaPath` |
| Add output fields | `ADD_ACTION_FLOW_OUTPUT_FIELDS` | `actionFlowId`, `items` |
| Delete output fields | `DELETE_ACTION_FLOW_OUTPUT_FIELDS` | `actionFlowId`, `names` |

### Trigger operations

| Intent | `name` | Required `args` |
|---|---|---|
| List DB triggers | `GET_ALL_DB_TRIGGER_INFOS` | — |
| DB trigger detail | `GET_DB_TRIGGER_DETAIL` | `triggerId` |
| Add DB triggers | `ADD_DB_TRIGGERS` | `items` |
| Update a DB trigger | `UPDATE_DB_TRIGGER` | `triggerId` |
| Delete DB triggers | `DELETE_DB_TRIGGERS` | `triggerIds` |
| List scheduled triggers | `GET_ALL_SCHEDULED_TRIGGER_INFOS` | — |
| Scheduled trigger detail | `GET_SCHEDULED_TRIGGER_DETAIL` | `triggerId` |
| Add scheduled triggers | `ADD_SCHEDULED_TRIGGERS` | `items` |
| Update a scheduled trigger | `UPDATE_SCHEDULED_TRIGGER` | `triggerId` |
| Delete scheduled triggers | `DELETE_SCHEDULED_TRIGGERS` | `triggerIds` |

Triggers fire an action flow without a manual call: a **DB trigger** fires on a table
INSERT/UPDATE/DELETE (`dbOperationType`), a **scheduled trigger** fires on a Quartz `cron`
expression. Both seed the target flow's input params as empty bindings — fill them via
`data-binding.md` at the schema paths from `GET_DB_TRIGGER_DETAIL` / `GET_SCHEDULED_TRIGGER_DETAIL`.
A DB trigger's firing condition (which rows it applies to) is edited with the request-filter ops at
its condition schema path; changing `dbOperationType` or the watched table resets it to always-true.

## Node configuration

`ADD_ACTION_FLOW_NODE`'s `node` and `UPDATE_ACTION_FLOW_NODE`'s `config` are **discriminated by a
`type` field**; each `type` carries a different body, and the add and update bodies differ (e.g.
`CUSTOM_CODE` gains `outputType` on update, `THIRD_PARTY_API` is editable only on update, and
`UPDATE_GLOBAL_VARIABLES` / `FOR_EACH_START` / `WHILE_START` / `BREAK` are add-only). The exact
per-`type` body for each is the discriminated union under `node` / `config` in *Arguments* below;
`type` must match the target node's actual type. AI nodes require an async flow (`isAsync=true`).

Block nodes (branch / for-each / while) auto-create their end + initial contents; deleting a
block-start deletes the whole block. A DB node seeds its editable columns as empty bindings — fill
each value at the node's `schemaPath` per `data-binding.md`, and narrow its rows with the request-filter
ops there. Those ops edit a request's `filters` (the live where/sort model) and work the same on a query
node and an update / delete node. An insert / update node must bind **at least one** column: a mutation node with no bound
field fails `schema validate` with "Mutation updates at least one field". Input-param and
output-field **types** are copied
verbatim from `GET_ACTION_FLOW_SELECTABLE_TYPES` — never hand-built.

**An update / delete record node ships with an always-true default filter, so it matches every row until you
narrow it.** Add `where` conditions with the request-filter ops (`GET_REQUEST_FILTER_CONTEXT` →
`ADD_REQUEST_FILTER_CONDITION`) on the node's filters before syncing, or the write hits the whole table.

> Output on **pre-refactor** projects uses `ADD_ACTION_FLOW_OUTPUT_FIELDS` / `DELETE_ACTION_FLOW_OUTPUT_FIELDS`. On post-refactor projects use `SET_ACTION_FLOW_OUTPUT` instead.

AI / video nodes must be async (`isAsync=true`). Discover node/ids via `GET_ACTION_FLOW_DETAIL`; fill node value bindings with `data-binding.md`.

**Preset integration nodes (dynamic catalog):** beyond the built-in node types above, the editor exposes a server-managed set of published `TEMPLATE_CODE` templates (SMS, file/media helpers, video/AI generation, …) that varies by deployment — never assume a specific provider exists. Discover the current set with `"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" actionflow list-node-templates` (returns each template's `templateCodeId` plus its input/output param types), then insert one via `ADD_ACTION_FLOW_NODE` with the `TEMPLATE_CODE` node type and that `templateCodeId`, and bind its inputs at the node's `schemaPath` per `data-binding.md`.

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `GET_ACTION_FLOW_DETAIL`

Get the full structure of one action flow: its input params, output, declared variables, and node tree. Use a flow id from GET_ALL_ACTION_FLOW_INFOS.
- `actionFlowId` *(required)*: `string` — The unique id of the action flow to inspect.

### `GET_ACTION_FLOW_CONTEXT_INFO`

Return the data and variables in scope at a node's schema path — what a binding at that path may reference (upstream node outputs, flow inputs, variables).
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>`

### `ADD_ACTION_FLOWS`

Create one or more action flows. Each is seeded empty (FLOW_START connected straight to FLOW_END); add nodes afterwards with ADD_ACTION_FLOW_NODE.
- `items` *(required)*: `array<{displayName: string, isAsync?: boolean, timeout?: integer}>` — Action flows to create. Each is seeded with an empty body (a FLOW_START connected directly to a FLOW_END); add nodes afterwards with ADD_ACTION_FLOW_NODE.

### `UPDATE_ACTION_FLOW`

Update an action flow's display name, async/sync execution mode, or timeout.
- `actionFlowId` *(required)*: `string` — The unique id of the action flow to update.
- `displayName`: `string`
- `isAsync`: `boolean` — Whether the flow runs asynchronously (fire-and-forget).
- `timeout`: `integer` — Execution timeout in seconds.
- `useEventLoop`: `boolean`

### `DELETE_ACTION_FLOWS`

Delete action flows by id.
- `actionFlowIds` *(required)*: `array<string>` — The unique ids of the action flows to delete.

### `ADD_ACTION_FLOW_INPUT_PARAMS`

Declare one or more typed input params on an action flow. Each param's type must be a value copied from GET_ACTION_FLOW_SELECTABLE_TYPES.
- `actionFlowId` *(required)*: `string`
- `items` *(required)*: `array<{arrayLevel?: integer, name: string, type?: string}>`

### `UPDATE_ACTION_FLOW_INPUT_PARAMS`

Update existing action-flow input params (rename, retype, or change required).
- `actionFlowId` *(required)*: `string`
- `items` *(required)*: `array<{arrayLevel?: integer, name: string, newName?: string, type?: string}>`

### `DELETE_ACTION_FLOW_INPUT_PARAMS`

Remove input params from an action flow.
- `actionFlowId` *(required)*: `string`
- `names` *(required)*: `array<string>` — Names (keys) of the input parameters to delete.

### `ADD_ACTION_FLOW_NODE`

Insert a node immediately after an existing node (afterNodeId). Read node ids from GET_ACTION_FLOW_DETAIL first. Set the node's data bindings afterwards with the bindings plugin at the node's schema path.
- `actionFlowId` *(required)*: `string`
- `afterNodeId` *(required)*: `string` — Insert the new node immediately after this node (its uniqueId).
- `displayName`: `string` — Optional display name; defaults to the localized node-type name.
- `node` *(required)*: `object · type: AI_CREATE_CONVERSATION|AI_SEND_MESSAGE|AI_DELETE_CONVERSATION|AI_STOP_RESPONSE → {configId?: string, taskId?: string} | BRANCH_SEPARATION → {branchNames?: array<string>, conditionType?: enum(MUTUAL_EXCLUSION|MUTUAL_TOLERANCE)} | BREAK → {} | ACTION_FLOW → {targetActionFlowId?: string} | CUSTOM_CODE → {code?: string} | FOR_EACH_START → {} | INSERT_RECORD|UPDATE_RECORD|DELETE_RECORD → {tableDisplayName?: string} | QUERY_RECORD → {limit?: integer, tableDisplayName?: string} | ADD_ROLE_TO_ACCOUNT|REMOVE_ROLE_FROM_ACCOUNT → {roleUuid?: string} | TEMPLATE_CODE → {templateCodeId: string} | THIRD_PARTY_API → {thirdPartyApiId?: string} | UPDATE_GLOBAL_VARIABLES → {} | WHILE_START → {}`

### `UPDATE_ACTION_FLOW_NODE`

Edit a node's display name or type-specific scalar config (e.g. queried table, row limit, target flow). Data-binding config — values, conditions, data sources, mutation fields — is edited with the bindings plugin, not here.
- `actionFlowId` *(required)*: `string`
- `config`: `object · type: AI_CREATE_CONVERSATION|AI_SEND_MESSAGE|AI_DELETE_CONVERSATION|AI_STOP_RESPONSE → {configId?: string, taskId?: string} | BRANCH_SEPARATION → {conditionType?: enum(MUTUAL_EXCLUSION|MUTUAL_TOLERANCE)} | ACTION_FLOW → {targetActionFlowId?: string} | CUSTOM_CODE → {code?: string} | INSERT_RECORD|UPDATE_RECORD|DELETE_RECORD → {tableDisplayName?: string} | QUERY_RECORD → {limit?: integer, tableDisplayName?: string} | ADD_ROLE_TO_ACCOUNT|REMOVE_ROLE_FROM_ACCOUNT → {roleUuid?: string} | TEMPLATE_CODE → {templateCodeId?: string} | THIRD_PARTY_API → {operation?: string, thirdPartyApiId?: string}` — Node-type-specific scalar config to update. Its `type` must match the node's actual type; fields left null are unchanged. Only non-data-binding scalars are editable here — data-binding config (mutation set values, conditions, dataSource, input args, target account, AI message) is edited with the data-binding tools at the node's schema path, and a query/mutation's where/sort (now held in `filters`) is not editable here yet.
- `displayName`: `string` — New display name; applies to any node type.
- `nodeId` *(required)*: `string` — The uniqueId of the node to update.

### `DELETE_ACTION_FLOW_NODES`

Delete nodes from a flow by id.
- `actionFlowId` *(required)*: `string`
- `nodeIds` *(required)*: `array<string>` — uniqueIds of nodes to delete. A leaf node is removed on its own; passing a block start node (BRANCH_SEPARATION / FOR_EACH_START / WHILE_START) removes the entire block. The flow start/end and block boundary/branch-item nodes cannot be deleted directly.

### `MOVE_ACTION_FLOW_NODE`

Move a leaf node to immediately after another node.
- `actionFlowId` *(required)*: `string`
- `afterNodeId` *(required)*: `string` — Move the node to immediately after this node (its uniqueId).
- `nodeId` *(required)*: `string` — The uniqueId of the leaf node to move.

### `ADD_ACTION_FLOW_BRANCH_ITEM`

Add a branch to a Condition node's branch separation (branchSeparationId from GET_ACTION_FLOW_DETAIL).
- `actionFlowId` *(required)*: `string`
- `branchSeparationId` *(required)*: `string` — The uniqueId of the BRANCH_SEPARATION to add a branch to.
- `name`: `string` — Optional display name for the new branch.

### `ADD_ACTION_FLOW_GLOBAL_VARIABLES`

Declare flow-level variables (accessible by all nodes, assigned via a Set Variable node). Each type is copied from GET_ACTION_FLOW_SELECTABLE_TYPES.
- `actionFlowId` *(required)*: `string`
- `items` *(required)*: `array<{arrayLevel?: integer, displayName: string, type?: string}>`

### `UPDATE_ACTION_FLOW_GLOBAL_VARIABLES`

Rename or retype existing flow-level variables. Read variable keys from GET_ACTION_FLOW_DETAIL.
- `actionFlowId` *(required)*: `string`
- `items` *(required)*: `array<{arrayLevel?: integer, displayName?: string, type?: string, variableKey: string}>`

### `DELETE_ACTION_FLOW_GLOBAL_VARIABLES`

Remove flow-level variables by key.
- `actionFlowId` *(required)*: `string`
- `variableKeys` *(required)*: `array<string>` — The map keys (ids) of the global variables to delete.

### `ADD_GLOBAL_VARIABLES_NODE_TARGETS`

Add assignment targets (flow-variable keys) to a Set Variable node; bind each value afterwards with the bindings plugin.
- `actionFlowId` *(required)*: `string`
- `nodeId` *(required)*: `string` — uniqueId of the UPDATE_GLOBAL_VARIABLES node.
- `variableKeys` *(required)*: `array<string>` — Keys of the action-flow global variables this node should assign.

### `ADD_CUSTOM_CODE_NODE_INPUT`

Add a named input slot (referenced as args.<name>) to a Run Code node; bind its value with the bindings plugin. Generate the code body with generate_code.
- `actionFlowId` *(required)*: `string`
- `name` *(required)*: `string` — Name (key) of the new input; must be unique within the node.
- `nodeId` *(required)*: `string` — uniqueId of the CUSTOM_CODE node.

### `RENAME_CUSTOM_CODE_NODE_INPUT`

Rename a Run Code node input slot, keeping its current value binding.
- `actionFlowId` *(required)*: `string`
- `newName` *(required)*: `string` — New input name; must be unique within the node.
- `nodeId` *(required)*: `string` — uniqueId of the CUSTOM_CODE node.
- `oldName` *(required)*: `string` — Current input name.

### `ADD_CUSTOM_CODE_NODE_OUTPUT_VALUE`

Add a named output value to a Run Code node (for a node that returns several named results rather than one). Each value's type is copied from GET_ACTION_FLOW_SELECTABLE_TYPES.
- `actionFlowId` *(required)*: `string`
- `name` *(required)*: `string` — Name (key) of the new output; must be unique within the node.
- `nodeId` *(required)*: `string` — uniqueId of the CUSTOM_CODE node.
- `type` *(required)*: `enum(BIGSERIAL|BIGINT|INTEGER|FLOAT8|DECIMAL|TIMESTAMPTZ|TIMETZ|DATE|INTERVAL|TEXT|… 19 total)` — The output's type. Legacy custom-code outputs support only primitive column types: TEXT, BIGINT, DECIMAL, BOOLEAN, DATE, TIMETZ, TIMESTAMPTZ, IMAGE, VIDEO, FILE, GEO_POINT, JSONB (no arrays / tables / custom types).

### `SET_CUSTOM_CODE_NODE_OUTPUT_TYPE`

Set a Run Code node's single output type so downstream nodes can bind to its result (the value the code passes to context.setReturn). type is a value copied verbatim from GET_ACTION_FLOW_SELECTABLE_TYPES; arrayLevel wraps it in a list (1) or list-of-lists (2), omit for a scalar.
- `actionFlowId` *(required)*: `string`
- `arrayLevel`: `integer` — Array nesting level for [type] (1 = list, 2 = list of lists); omit/0 for a scalar.
- `nodeId` *(required)*: `string` — uniqueId of the CUSTOM_CODE node.
- `type` *(required)*: `string` — Output type: a TypeIdentifier selected from GET_ACTION_FLOW_SELECTABLE_TYPES (pass its `typeIdentifier` verbatim — never hand-build it).

### `SET_DISPLAY_NAME`
- `displayName` *(required)*: `string` — The suggested display name for the target.
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — The specific schema path in the project where the display name should be set.

### `GET_DB_TRIGGER_DETAIL`

Get one database-change trigger's full configuration by id.
- `triggerId` *(required)*: `string` — The uniqueId of the database trigger to inspect.

### `ADD_DB_TRIGGERS`

Create database-change triggers. Each fires a flow (actionFlowId from GET_ALL_ACTION_FLOW_INFOS) when a row in a table (tableDisplayName from the database plugin's list_tables) is inserted / updated / deleted (dbOperationType, defaults to INSERT).
- `items` *(required)*: `array<{actionFlowId: string, dbOperationType?: enum(INSERT|UPDATE|INSERT_OR_UPDATE|DELETE), displayName?: string, enabled?: boolean, tableDisplayName: string}>` — Database triggers to create. Each fires its action flow on the chosen table operation, with the flow's input args seeded as empty bindings (fill them via the data-binding tools at the schema paths from GET_DB_TRIGGER_DETAIL) and an always-true firing condition (edit it via the condition tools at the condition schema path from GET_DB_TRIGGER_DETAIL).

### `UPDATE_DB_TRIGGER`

Update a database-change trigger by id (table, operation, target flow, or enabled).
- `dbOperationType`: `enum(INSERT|UPDATE|INSERT_OR_UPDATE|DELETE)` — Which database operation fires the trigger: INSERT, UPDATE, DELETE or INSERT_OR_UPDATE. Changing it resets the firing condition to always-true.
- `displayName`: `string` — New display name.
- `enabled`: `boolean` — Whether the trigger is enabled.
- `tableDisplayName`: `string` — Display name of the table the trigger watches. Changing the table resets the firing condition to always-true (it references the table's columns).
- `triggerId` *(required)*: `string` — The uniqueId of the database trigger to update.

### `DELETE_DB_TRIGGERS`

Delete database-change triggers by id.
- `triggerIds` *(required)*: `array<string>` — The uniqueIds of the database triggers to delete.

### `GET_SCHEDULED_TRIGGER_DETAIL`

Get one scheduled trigger's full configuration by id.
- `triggerId` *(required)*: `string` — The uniqueId of the scheduled trigger to inspect.

### `ADD_SCHEDULED_TRIGGERS`

Create scheduled (cron) triggers. Each fires a flow (actionFlowId from GET_ALL_ACTION_FLOW_INFOS) on a Quartz cron schedule. IMPORTANT: endInstant defaults to the start, so the schedule never fires unless you set endInstant to a future epoch-millisecond timestamp.
- `items` *(required)*: `array<{actionFlowId: string, cron?: string, enabled?: boolean, endInstant?: integer, name?: string, startInstant?: integer}>` — Scheduled triggers to create. Each fires its action flow on a cron schedule, with the flow's input args seeded as empty bindings (fill them via the data-binding tools at the schema paths from GET_SCHEDULED_TRIGGER_DETAIL).

### `UPDATE_SCHEDULED_TRIGGER`

Update a scheduled trigger by id (cron, active window, target flow, or enabled).
- `cron`: `string` — New Quartz cron expression (6 fields: second minute hour day-of-month month day-of-week).
- `enabled`: `boolean` — Whether the trigger is enabled.
- `endInstant`: `integer` — New epoch-millisecond end timestamp.
- `name`: `string` — New display name.
- `startInstant`: `integer` — New epoch-millisecond start timestamp.
- `triggerId` *(required)*: `string` — The uniqueId of the scheduled trigger to update.

### `DELETE_SCHEDULED_TRIGGERS`

Delete scheduled triggers by id.
- `triggerIds` *(required)*: `array<string>` — The uniqueIds of the scheduled triggers to delete.

### `ADD_ACTION_FLOW_OUTPUT_FIELDS`

Add named output fields to a flow (legacy output model), one entry per field with its own type.
- `actionFlowId` *(required)*: `string`
- `items` *(required)*: `array<{name: string, type?: enum(BIGSERIAL|BIGINT|INTEGER|FLOAT8|DECIMAL|TIMESTAMPTZ|TIMETZ|DATE|INTERVAL|TEXT|… 19 total)}>`

### `DELETE_ACTION_FLOW_OUTPUT_FIELDS`

Remove named output fields from a flow (legacy output model).
- `actionFlowId` *(required)*: `string`
- `names` *(required)*: `array<string>` — Keys of the output fields to delete.

Then ship:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema validate && "${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
