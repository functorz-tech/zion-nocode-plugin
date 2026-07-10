# AI agents (ZAI)

## AI Agent (ZAI) Domain Knowledge
A ZAI config is an LLM-backed agent the app runs via a "Run AI" action / action-flow node (asynchronous only). An agent has a name + description, a model, sampling settings (temperature, maxRound, max output tokens), a system + user **prompt**, and its **output**.

### Reading
`GET_ALL_ZAI_CONFIG_INFOS` summarizes every agent; `GET_ZAI_CONFIG_DETAIL` returns one agent's full config — its input args (with their map keys), its prompt components each with the **schema path** of its text binding, and its output config.

### Creating & editing
`ADD_ZAI_CONFIGS` seeds an agent with default empty system + user prompts and plain-text output; adding the first agent also provisions the AI conversation tables/relations/permissions. Edit scalar config (name, description, temperature, maxRound) with `UPDATE_ZAI_CONFIG`. The **model** is also set via `UPDATE_ZAI_CONFIG` — pass `customModelIdentifier` ({id, namespace}) with an id from the platform's supported-model descriptor (`supportedCustomModelDescriptor.chatModelDescriptors`, which lists each model's exact identifier and features such as vision / file support). Never fabricate an id; if you cannot obtain one, leave the model unset for the user to pick in the editor.

### Prompts are bindings, not a ZAI tool
A prompt's text is an ordinary data binding. Read its `valueSchemaPath` from `GET_ZAI_CONFIG_DETAIL` and edit it with the bindings plugin (the CREATE_*_BINDING tools) at that path — there is no ZAI prompt-edit tool.

### System AI conversation tables
Creating the first agent provisions four protected system tables that store agent runs. They are platform-managed (read-only — never add fields to or edit them; for a user-facing chat feature build your own tables instead). Their individual roles, in a 1:n chain:
- `conversation` — a chat thread with an agent, owned by an account (account → conversations).
- `message` — one turn within a conversation (conversation → messages); also linked to its account.
- `message_content` — a single content block of a message: text or media, image / video / file (message → contents).
- `tool_usage_record` — a record of one tool call made while producing a message, including any error (message → tool_calls).

### Typed input args & output

**Input args**: each input arg's `type` must be one of these scalar types — this is the complete
set (copy the matching `typeIdentifier` from `GET_ZAI_CONFIG_SELECTABLE_TYPES`; never hand-build it):
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
When the user wants a typed agent output, either (a) ask them to configure the structured output
in the editor with these field types and confirm before continuing, or (b) — fully via tools —
keep **plain-text** output and pin the exact JSON shape in the **user prompt** (e.g. "Return ONLY
JSON matching {...}"), then have the caller parse the returned text (stripping any ```json fences).
Option (b) is the only no-editor path in the legacy type system.

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

**Choosing a model (no editor needed):** set it with `UPDATE_ZAI_CONFIG`'s `customModelIdentifier` ({id, namespace}). Discover valid ids + features (vision / file support) from the backend descriptor:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" platform graphql --query '{ supportedCustomModelDescriptor { chatModelDescriptors } }'
```
Copy an `id` (with its `namespace`) back verbatim — never fabricate one — then verify with `GET_ZAI_CONFIG_DETAIL`.

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `ADD_ZAI_CONFIGS`

Create one or more AI agents. Each is seeded with default empty system + user prompts, no input args, and plain-text output; edit prompt text afterwards with the bindings plugin at the schema paths from GET_ZAI_CONFIG_DETAIL.
- `items` *(required)*: `array<{customModelIdentifier?: object, name?: string}>` — AI agents to create. Each is seeded with the default system + user prompt components (empty text bindings, edit them via the data-binding tools at the schema paths from GET_ZAI_CONFIG_DETAIL), an empty input-arg set and a plain-text output config. Adding the first agent also provisions the AI conversation tables/relations/permissions if absent.

### `UPDATE_ZAI_CONFIG`

Update an agent's scalar config: name, description, temperature, maxRound, or model.
- `configId` *(required)*: `string` — The id of the AI agent to update.
- `customModelIdentifier`: `{id: string, namespace?: string}` — Model to use: the full model identifier ({ id, namespace }) as returned by the model-listing interface — pass it verbatim, never hand-build it. The deprecated `model` field is never written; selecting a model goes through this identifier.
- `description`: `string` — New description.
- `ignoreNullValuesFromContext`: `boolean` — Whether null values from the selected context are ignored.
- `imageInputQuality`: `enum(LOW|HIGH)` — Image input quality: LOW or HIGH.
- `maxRound`: `integer` — Maximum number of agent rounds.
- `name`: `string` — New display name.
- `temperature`: `number` — Sampling temperature.

### `ADD_ZAI_CONFIG_INPUT_ARGS`

Add typed input arguments to an agent. Each arg's type is a typeIdentifier copied from GET_ZAI_CONFIG_SELECTABLE_TYPES. Works in both type-system modes: the refactored system allows arrays (arrayLevel), tables, and custom types; the legacy system accepts basic scalars only (no arrayLevel/arrays, tables, or objects).
- `configId` *(required)*: `string`
- `items` *(required)*: `array<{arrayLevel?: integer, displayName: string, type?: string}>`

### `UPDATE_ZAI_CONFIG_OUTPUT`

Configure the agent's output: plain streamed text (isStructured=false) or a structured typed object (isStructured=true with an outputType copied from GET_ZAI_CONFIG_SELECTABLE_TYPES).
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
