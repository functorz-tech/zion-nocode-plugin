# AI agents (ZAI)

## AI Agent (ZAI) Domain Knowledge
A ZAI config is an LLM-backed agent the app runs via a "Run AI" action / action-flow node (asynchronous only). An agent has a name + description, a model, sampling settings (temperature, maxRound, max output tokens), a system + user **prompt**, and its **output**.

### Reading
`GET_ZAI_MODEL_OPTIONS` returns the models currently selectable in this project. `GET_ALL_ZAI_CONFIG_INFOS` summarizes every agent; `GET_ZAI_CONFIG_DETAIL` returns one agent's full config — its input args (with their map keys), its prompt components each with the **schema path** of its text binding, its output config and structured-output fields, its database/API contexts (with the query schema paths for the request-filter tools), its knowledge base, and its callable tools.

### Creating & editing
`ADD_ZAI_CONFIGS` seeds an agent with default empty system + user prompts and plain-text output; adding the first agent also provisions the AI conversation tables/relations/permissions. Edit scalar config (name, description, temperature, maxRound) with `UPDATE_ZAI_CONFIG`. The **model** is required when creating an agent and can be changed via `UPDATE_ZAI_CONFIG`. Call `GET_ZAI_MODEL_OPTIONS` first. If the user did not name a model, choose the single selectable option whose `defaultModel` is true, matching the editor's default; otherwise choose a selectable model whose capabilities fit the request. Copy its exact `customModelIdentifier` ({id, namespace}). Never fabricate or omit the identifier.

### Prompts are bindings, not a ZAI tool
A prompt's text is an ordinary data binding. Read its `valueSchemaPath` from `GET_ZAI_CONFIG_DETAIL` and edit it with the bindings plugin (the CREATE_*_BINDING tools) at that path — there is no ZAI prompt-edit tool.

### System AI conversation tables
Creating the first agent provisions four protected system tables that store agent runs. They are platform-managed (read-only — never add fields to or edit them; for a user-facing chat feature build your own tables instead). Their individual roles, in a 1:n chain:
- `conversation` — a chat thread with an agent, owned by an account (account → conversations).
- `message` — one turn within a conversation (conversation → messages); also linked to its account.
- `message_content` — a single content block of a message: text or media, image / video / file (message → contents).
- `tool_usage_record` — a record of one tool call made while producing a message, including any error (message → tool_calls).

### Giving the agent context (what it can read)
Beyond its prompt, an agent reads context — all shown by `GET_ZAI_CONFIG_DETAIL`:
- **Database tables** — `ADD_ZAI_CONFIG_DB_CONTEXTS` (`UPDATE_ZAI_CONFIG_DB_CONTEXT` / `DELETE_ZAI_CONFIG_DB_CONTEXTS`). Each is a per-table query with column/relation selection; narrow which rows it sees by editing the query's `filters` with the request-filter tools at the `querySchemaPath` from `GET_ZAI_CONFIG_DETAIL`.
- **Knowledge base** — existing knowledge-base files and retrieval settings are shown by `GET_ZAI_CONFIG_DETAIL`, but Copilot cannot mutate them. Ask the user to manage them in the editor.
- **External APIs** — an external-API context; the tool is generation-specific (see the input-args section below).

### Giving the agent tools (what it can call)
`ADD_ZAI_CONFIG_TOOLS` lets the agent call action flows, external APIs, or other agents (`UPDATE_ZAI_CONFIG_TOOL` / `DELETE_ZAI_CONFIG_TOOLS`). The tool for tuning a callable tool's per-input descriptions is generation-specific (see the input-args section below).
### Passing a list into a list-typed input arg
To declare a list input, choose a base type from `GET_ZAI_CONFIG_SELECTABLE_TYPES` and set `arrayLevel: 1` (or `arrayLevel: 2` for a list of lists). Fill it where the agent is invoked. In the calling action flow's Run-AI node, bind it with an **array-mapping** formula at the input's schema path. The agent then runs once over the whole list.
### Typed input args, output & context (refactored type system enabled)
Declare typed inputs with `ADD_ZAI_CONFIG_INPUT_ARGS` (and `UPDATE_ZAI_CONFIG_INPUT_ARGS` / `DELETE_ZAI_CONFIG_INPUT_ARGS`). Each
arg's `type` is a `typeIdentifier` copied verbatim from `GET_ZAI_CONFIG_SELECTABLE_TYPES` — primitives, tables,
and custom object/enum types are all selectable; `arrayLevel` wraps it as a list (1) or list-of-lists
(2), for any element type (a list of image is the common multi-image intake — see "Passing a list into
an input arg" above). Configure output with `UPDATE_ZAI_CONFIG_OUTPUT`: `isStructured=false` for plain streamed text, or
`isStructured=true` with an `outputType` (a `typeIdentifier` from `GET_ZAI_CONFIG_SELECTABLE_TYPES`) for a typed
object — for a bespoke object shape, define the object type first with the type-system tools, then
select its `typeIdentifier`.
External-API context here is workspace HTTP APIs: `ADD_ZAI_CONFIG_API_CONTEXTS` (`UPDATE_ZAI_CONFIG_API_CONTEXT` /
`DELETE_ZAI_CONFIG_API_CONTEXTS`). Tune a callable tool's per-input descriptions with
`UPDATE_ZAI_CONFIG_TOOL_DESCRIPTION_CONFIGS`.

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
| List agents | `GET_ALL_ZAI_CONFIG_INFOS` | — |
| Agent detail (ids/paths) | `GET_ZAI_CONFIG_DETAIL` | `configId` |
| Selectable I/O types | `GET_ZAI_CONFIG_SELECTABLE_TYPES` | `slot` |
| Create agents | `ADD_ZAI_CONFIGS` | `items` |
| Update an agent | `UPDATE_ZAI_CONFIG` | `configId` |
| Delete agents | `DELETE_ZAI_CONFIGS` | `configIds` |
| Add input args | `ADD_ZAI_CONFIG_INPUT_ARGS` | `configId`, `items` |
| Update input args | `UPDATE_ZAI_CONFIG_INPUT_ARGS` | `configId`, `items` |
| Delete input args | `DELETE_ZAI_CONFIG_INPUT_ARGS` | `argKeys`, `configId` |
| Update knowledge-base config | `UPDATE_ZAI_CONFIG_KNOWLEDGE_BASE` | — |
| Set output config | `UPDATE_ZAI_CONFIG_OUTPUT` | `configId` |

`ADD_ZAI_CONFIGS` seeds each agent with empty system + user prompt components and a plain-text
output; the **first** agent also provisions the AI-conversation tables / relations / permissions.
Read ids and the prompt/output `schemaPath`s back from `GET_ZAI_CONFIG_DETAIL`, then fill the prompts
and input-arg bindings with `data-binding.md` at those paths. An action flow invokes the agent via a
Run AI node (`actionflow.md`) that references the config by id.

**Choosing a model (no editor needed):** set it with `UPDATE_ZAI_CONFIG`'s `customModelIdentifier` ({id, namespace}). Discover valid ids + features (vision / file support) from the backend descriptor:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" platform graphql --query '{ supportedCustomModelDescriptor { chatModelDescriptors } }'
```
Copy an `id` (with its `namespace`) back verbatim — never fabricate one — then verify with `GET_ZAI_CONFIG_DETAIL`.

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `ADD_ZAI_CONFIGS`

Create one or more AI agents. Each is seeded with default empty system + user prompts, no input args, and plain-text output; edit prompt text afterwards with the bindings plugin at the schema paths from GET_ZAI_CONFIG_DETAIL.
- `items` *(required)*: `array<{customModelIdentifier: object, name?: string}>` — AI agents to create. Each is seeded with the default system + user prompt components (empty text bindings, edit them via the CREATE_*_BINDING tools at the schema paths from GET_ZAI_CONFIG_DETAIL), an empty input-arg set and a plain-text output config. Adding the first agent also provisions the AI conversation tables/relations/permissions if absent.

### `UPDATE_ZAI_CONFIG`

Update an agent's scalar config: name, description, temperature, maxRound, or model.
- `configId` *(required)*: `string` — The id of the AI agent to update.
- `customModelIdentifier`: `{id: string, namespace?: string}` — Model to use: the full model identifier ({ id, namespace }) returned by GET_ZAI_MODEL_OPTIONS — pass it verbatim, never hand-build it. The deprecated `model` field is never written; selecting a model goes through this identifier.
- `description`: `string` — New description.
- `ignoreNullValuesFromContext`: `boolean` — Whether null values from the selected context are ignored.
- `imageInputQuality`: `enum(LOW|HIGH)` — Image input quality: LOW or HIGH.
- `maxRound`: `integer` — Maximum number of agent rounds.
- `name`: `string` — New display name.
- `temperature`: `number` — Sampling temperature.

### `ADD_ZAI_CONFIG_INPUT_ARGS`

Add typed input arguments. Copy each base typeIdentifier from GET_ZAI_CONFIG_SELECTABLE_TYPES and use arrayLevel when a list or nested list is required.
- `configId` *(required)*: `string`
- `items` *(required)*: `array<{arrayLevel?: integer, displayName: string, type?: string}>`

### `UPDATE_ZAI_CONFIG_OUTPUT`

Configure the agent's output: plain streamed text (isStructured=false) or a structured typed object (isStructured=true with an outputType copied from GET_ZAI_CONFIG_SELECTABLE_TYPES).
- `arrayLevel`: `integer` — Array nesting level for [outputType]; 1 = list, 2 = list of lists. Ignored when [outputType] is null.
- `configId` *(required)*: `string` — The id of the AI agent whose output config to update.
- `isStreaming`: `boolean` — Whether the plain-text output streams. Only meaningful when not structured.
- `isStructured`: `boolean` — Whether the agent emits a structured (typed) output (true) or plain text (false). Switching to structured seeds `outputType` to string when none is given; switching to plain text drops the output type.
- `maxTokenSize`: `integer` — Max output token size.
- `outputDescriptionConfig`: `{description?: string, fieldDescriptionByType?: map<string, object>}` — Field-description config for the output: a `description` plus `fieldDescriptionByType` (per-type field descriptions). Replaces the whole description config when provided.
- `outputType`: `string` — The structured output's type (refactored type system). Copy a `typeIdentifier` returned by GET_ZAI_CONFIG_SELECTABLE_TYPES verbatim — never hand-build the string. Only meaningful when the output is structured.

Then ship:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema validate && "${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
