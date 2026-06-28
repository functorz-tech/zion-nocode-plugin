# AI agents (ZAI)

## AI Agent (ZAI) Domain Knowledge
A ZAI config is an LLM-backed agent the app runs via a "Run AI" action / action-flow node
(asynchronous only). An agent has a name + description, a model, sampling settings
(temperature, maxRound, max output tokens), a system + user **prompt**, and its **output**.

### Reading
`list_configs` summarizes every agent; `get_config_detail` returns one agent's full config — its
input args (with their map keys), its prompt components each with the **schema path** of its
text binding, and its output config.

### Creating & editing
`create_configs` seeds an agent with default empty system + user prompts and plain-text output;
adding the first agent also provisions the AI conversation tables/relations/permissions. Edit
scalar config (name, description, temperature, maxRound, model) with `update_config`. Leave the
model unset to configure it later in the editor — there is no model-listing tool, so never
hand-build a model identifier.

### Prompts are bindings, not a ZAI tool
A prompt's text is an ordinary data binding. Read its `valueSchemaPath` from `get_config_detail`
and edit it with the bindings plugin (the CREATE_*_BINDING tools) at that path — there is no ZAI
prompt-edit tool.

### Typed input args & output

**Input args**: each input arg's `type` must be one of these scalar types — this is the complete
set (copy the matching `typeIdentifier` from `get_selectable_types`; never hand-build it):
- text (string)
- integer
- decimal
- boolean
- date
- time
- date-time (timestamp)
- image
- video
- file
- geographic point
- JSON
An input arg can only be one of the scalar types above — do not pass `arrayLevel` (arrays) or
pick a table / object / enum / union type; those are rejected.

**Output**: an agent returns plain streamed text. A **structured (typed) output** is configured
in the editor (there is no tool for it here): it is an object whose fields the user defines, where
each field's type is one of:
- text (string)
- number
- decimal
- integer
- boolean
- date-time (timestamp)
- time
- date
- object (a nested object with its own fields)
- array (a list of any of the above except a nested array)
When the user wants a typed agent output, ask them to configure the structured output in the
editor with these field types, and wait for their confirmation before continuing.

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
| List agents | `GET_ALL_ZAI_CONFIG_INFOS` | — |
| Agent detail (ids/paths) | `GET_ZAI_CONFIG_DETAIL` | `configId` |
| Selectable I/O types | `GET_ZAI_CONFIG_SELECTABLE_TYPES` | — |
| Create agents | `ADD_ZAI_CONFIGS` | `items` |
| Update an agent | `UPDATE_ZAI_CONFIG` | `configId` |
| Delete agents | `DELETE_ZAI_CONFIGS` | `configIds` |
| Add input args | `ADD_ZAI_CONFIG_INPUT_ARGS` | `configId`, `items` |
| Update input args | `UPDATE_ZAI_CONFIG_INPUT_ARGS` | `configId`, `items` |
| Delete input args | `DELETE_ZAI_CONFIG_INPUT_ARGS` | `argKeys`, `configId` |
| Set output config | `UPDATE_ZAI_CONFIG_OUTPUT` | `configId` |

`ADD_ZAI_CONFIGS` seeds each agent with empty system + user prompt components and a plain-text
output; the **first** agent also provisions the AI-conversation tables / relations / permissions.
Read ids and the prompt/output `schemaPath`s back from `GET_ZAI_CONFIG_DETAIL`, then fill the prompts
and input-arg bindings with `data-binding.md` at those paths. An action flow invokes the agent via a
Run AI node (`actionflow.md`) that references the config by id.

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `ADD_ZAI_CONFIGS`
- `items` *(required)*: `array<{customModelIdentifier?: object, name?: string}>` — AI agents to create. Each is seeded with the default system + user prompt components (empty text bindings, edit them via the data-binding tools at the schema paths from GET_ZAI_CONFIG_DETAIL), an empty input-arg set and a plain-text output config. Adding the first agent also provisions the AI conversation tables/relations/permissions if absent.

### `UPDATE_ZAI_CONFIG`
- `configId` *(required)*: `string` — The id of the AI agent to update.
- `customModelIdentifier`: `{id: string, namespace?: string}` — Model to use: the full model identifier ({ id, namespace }) as returned by the model-listing interface — pass it verbatim, never hand-build it. The deprecated `model` field is never written; selecting a model goes through this identifier.
- `description`: `string` — New description.
- `ignoreNullValuesFromContext`: `boolean` — Whether null values from the selected context are ignored.
- `imageInputQuality`: `enum(LOW|HIGH)` — Image input quality: LOW or HIGH.
- `maxRound`: `integer` — Maximum number of agent rounds.
- `name`: `string` — New display name.
- `temperature`: `number` — Sampling temperature.

### `ADD_ZAI_CONFIG_INPUT_ARGS`
- `configId` *(required)*: `string`
- `items` *(required)*: `array<{arrayLevel?: integer, displayName: string, type?: string}>`

### `UPDATE_ZAI_CONFIG_OUTPUT`
- `configId` *(required)*: `string` — The id of the AI agent whose output config to update.
- `isStreaming`: `boolean` — Whether the plain-text output streams. Only meaningful when not structured.
- `isStructured`: `boolean` — Whether the agent emits a structured (typed) output (true) or plain text (false). Switching to structured seeds `outputType` to string when none is given; switching to plain text drops the output type.
- `maxTokenSize`: `integer` — Max output token size.
- `outputDescriptionConfig`: `{description?: string, fieldDescriptionByType?: object}` — Field-description config for the output: a `description` plus `fieldDescriptionByType` (per-type field descriptions). Replaces the whole description config when provided.
- `outputType`: `string` — The structured output's type (refactored type system). Copy a `typeIdentifier` returned by GET_ZAI_CONFIG_SELECTABLE_TYPES verbatim — never hand-build the string. Only meaningful when the output is structured.

Then ship:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema validate && "${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
