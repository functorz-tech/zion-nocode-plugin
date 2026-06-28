# Action flows (server-side workflows)

## Actionflow Domain Knowledge
Actionflows are server-side workflows for business logic. They run entirely on the server;
the frontend triggers them via "Request – Actionflow" component actions.

### Execution Modes
Synchronous: caller blocks for the result; ACID (any node error rolls back all DB writes).
Asynchronous: the caller gets a task handle instead of blocking and retrieves the result
  later — the flow still computes and returns its declared output; only the failing node's
  DB writes roll back; required for AI agents and video generation.

Both modes share a per-flow total timeout (`ACTION_FLOW_TIMEOUT_MILLISECONDS`): 15 s on
free-tier servers, 10 min on paid tiers. The budget is for the entire flow, not per node.

### Triggers
Manual (UI action), scheduled (cron), database-change event, webhook.

### Node Types
Database: query/insert/update/delete on a table.
Call API: invoke a configured API; bind response to output params.
Run AI: execute a ZAI agent (async only).
Run Actionflow: call another flow (sync cannot call async).
Set Variable: assign values to declared flow-level variables.
Current User: get current user's ID / Access Token.
Permissions: grant or revoke roles for a user.
Run Code: execute custom JavaScript.
Files: fetch/convert external images/videos/files to Zion native types.
Condition: branch logic evaluated left to right.
Loop: iterate over a list; inner nodes access item data.
Send SMS - Twilio: send SMS messages.
Gemini Veo 3.1: generate 8-second video clips (async only).

### Choosing How to Express Logic: nodes → formulas → code
Three tiers, in order of preference:
1. **Visual nodes** for orchestration — query / branch / iterate / write / call (Database,
   Condition, Loop, Set Variable, Call API, Run Actionflow). They take part in the flow's
   transaction, surface individually in logs and the edit-time error collector, and stay
   user-editable.
2. **Formula bindings** for value-level computation. A node value — a Set Variable value, a
   mutation column, a condition predicate — can be a FORMULA binding built with the bindings
   plugin, and formulas cover far more than people expect: text and regex (extract / match /
   replace, substring, split, concat), math, date/time arithmetic and formatting, array
   aggregation / mapping / filtering, JSON value access, and type casts. Prefer a formula over
   code whenever you are computing a single value.
3. **Run Code** only for genuinely procedural logic that neither a node nor a formula operator
   expresses — e.g. cryptographic hashing / signature building, multi-step algorithms with
   intermediate state, or assembling / parsing a complex nested payload. Keep it small and fed
   by node inputs; never collapse a whole flow into one Run Code node.

### Variables
Declared at flow level, accessible by all nodes, assigned with "Set Variable" node.

### Inputs & Outputs
Inspect a flow first with `list_flows` / `get_flow_detail`. Declare typed
**input params** with `add_input_params`; set each param's type to a value copied
verbatim from `get_selectable_types` (never hand-build the type string).
A flow returns a **set of named output fields**: add them with `add_output_fields`
(one entry per field, each with its own type) and remove them with
`delete_output_fields`.

### Building & Editing a Flow
Create flows with `create_flows` (each seeds an empty FLOW_START → FLOW_END), then
read node and structure ids from `get_flow_detail` before editing. Add a node after
another with `add_node`, reorder with `move_node`, remove with
`delete_nodes`, and branch a Condition node with `add_branch`. Edit a
node's name or type-scalar config with `update_node`, but set its data bindings
(values, conditions, data sources, mutation fields) with the bindings plugin at the node's
schema path — never inline. Declare flow variables with `add_global_variables` and
assign them in a Set Variable node with `add_variable_assignments`. A Run Code node's
`args.<name>` slots are managed with `add_code_input` (rename/delete variants); fill
the code body with `generate_code`.
An update or delete Database node is seeded with an always-true filter that matches **every**
row — narrow it with the bindings plugin's where conditions at the node's schema path before
syncing, or the write hits the whole table.

### Error Handling
Synchronous: all DB changes roll back on error.
Asynchronous: only the failing node's DB changes roll back.

### Versioning
Each save creates a new version. "Sync Backend" required after editing for changes to
take effect in production.

### JavaScript Sandbox (Run Code node)
Custom Code Blocks run in a synchronous GraalJS server sandbox. No async/await,
no require(), no browser APIs. All platform interactions go through the global
`context` object:
  context.getArg(key)                              — retrieve flow input
  context.setReturn(key, val)                      — pass output to downstream nodes
  context.runGql(query, variables, options)        — execute DB operations
  context.callThirdPartyApi(apiId, params)         — invoke a configured REST API
  context.callActionFlow(flowId, ver, args)        — run a sub-flow synchronously
  context.createActionFlowTask(flowId, ver, args)  — trigger a sub-flow async
  context.getSsoUserInfo()                         — get authenticated user profile
  context.uploadMedia(url, headers)                — stream remote image to asset server
  context.log(message)                             — emit to Log Service

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
| Flow detail (ids) | `GET_ACTION_FLOW_DETAIL` | `actionFlowId` |
| Selectable I/O types | `GET_ACTION_FLOW_SELECTABLE_TYPES` | — |
| Add flows | `ADD_ACTION_FLOWS` | `items` |
| Add input params | `ADD_ACTION_FLOW_INPUT_PARAMS` | `actionFlowId`, `items` |
| Add a node | `ADD_ACTION_FLOW_NODE` | `actionFlowId`, `afterNodeId`, `node` |
| Add output fields | `ADD_ACTION_FLOW_OUTPUT_FIELDS` | `actionFlowId`, `items` |

## Node configuration

`ADD_ACTION_FLOW_NODE`'s `node` and `UPDATE_ACTION_FLOW_NODE`'s `config` are **discriminated by a
`type` field**; each `type` carries a different body, and the add and update bodies differ (e.g.
`CUSTOM_CODE` gains `outputType` on update, `THIRD_PARTY_API` is editable only on update, and
`UPDATE_GLOBAL_VARIABLES` / `FOR_EACH_START` / `WHILE_START` / `BREAK` are add-only). The exact
per-`type` body for each is the discriminated union under `node` / `config` in *Arguments* below;
`type` must match the target node's actual type. AI nodes require an async flow (`isAsync=true`).

Block nodes (branch / for-each / while) auto-create their end + initial contents; deleting a
block-start deletes the whole block. A DB node seeds its editable columns as empty bindings — fill
each value at the node's `schemaPath` per `data-binding.md`, and add query `where` / gating
conditions with the request-filter ops there. An insert / update node must bind **at least one** column: a mutation node with no bound
field fails `schema validate` with "Mutation updates at least one field". Input-param and
output-field **types** are copied
verbatim from `GET_ACTION_FLOW_SELECTABLE_TYPES` — never hand-built.

> Output on **pre-refactor** projects uses `ADD_ACTION_FLOW_OUTPUT_FIELDS` / `DELETE_ACTION_FLOW_OUTPUT_FIELDS`. On post-refactor projects use `SET_ACTION_FLOW_OUTPUT` instead.

AI / video nodes must be async (`isAsync=true`). Discover node/ids via `GET_ACTION_FLOW_DETAIL`; fill node value bindings with `data-binding.md`.

**Preset integration nodes (dynamic catalog):** beyond the built-in node types above, the editor exposes a server-managed set of published `TEMPLATE_CODE` templates (SMS, file/media helpers, video/AI generation, …) that varies by deployment — never assume a specific provider exists. Discover the current set with `"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" actionflow list-node-templates` (returns each template's `templateCodeId` plus its input/output param types), then insert one via `ADD_ACTION_FLOW_NODE` with the `TEMPLATE_CODE` node type and that `templateCodeId`, and bind its inputs at the node's `schemaPath` per `data-binding.md`.

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `ADD_ACTION_FLOWS`
- `items` *(required)*: `array<{displayName: string, isAsync?: boolean, timeout?: integer}>` — Action flows to create. Each is seeded with an empty body (a FLOW_START connected directly to a FLOW_END); add nodes afterwards with ADD_ACTION_FLOW_NODE.

### `ADD_ACTION_FLOW_INPUT_PARAMS`
- `actionFlowId` *(required)*: `string`
- `items` *(required)*: `array<{arrayLevel?: integer, name: string, type?: string}>`

### `ADD_ACTION_FLOW_NODE`
- `actionFlowId` *(required)*: `string`
- `afterNodeId` *(required)*: `string` — Insert the new node immediately after this node (its uniqueId).
- `displayName`: `string` — Optional display name; defaults to the localized node-type name.
- `node` *(required)*: `object · type: AI_CREATE_CONVERSATION|AI_SEND_MESSAGE|AI_DELETE_CONVERSATION|AI_STOP_RESPONSE → {configId?: string, taskId?: string} | BRANCH_SEPARATION → {branchNames?: array<string>, conditionType?: enum(MUTUAL_EXCLUSION|MUTUAL_TOLERANCE)} | BREAK → {} | ACTION_FLOW → {targetActionFlowId?: string} | CUSTOM_CODE → {code?: string} | FOR_EACH_START → {} | INSERT_RECORD|UPDATE_RECORD|DELETE_RECORD → {tableDisplayName?: string} | QUERY_RECORD → {limit?: integer, tableDisplayName?: string} | ADD_ROLE_TO_ACCOUNT|REMOVE_ROLE_FROM_ACCOUNT → {roleUuid?: string} | TEMPLATE_CODE → {templateCodeId: string} | THIRD_PARTY_API → {} | UPDATE_GLOBAL_VARIABLES → {} | WHILE_START → {}`

### `ADD_ACTION_FLOW_OUTPUT_FIELDS`
- `actionFlowId` *(required)*: `string`
- `items` *(required)*: `array<{name: string, type?: enum(BIGSERIAL|BIGINT|INTEGER|FLOAT8|DECIMAL|TIMESTAMPTZ|TIMETZ|DATE|INTERVAL|TEXT|… 19 total)}>`

Then ship:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema validate && "${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
