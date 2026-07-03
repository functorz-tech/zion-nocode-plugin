# API integrations (external HTTP data sources)

## API Integration Domain Knowledge
An API integration is a saved external HTTP endpoint the project can use as a data source. APIs are organized into **workspaces**: a workspace groups related endpoints and holds shared **constants** (e.g. a base URL or API key) the endpoints reference, so credentials live in one place. (Distinct from a Third-Party API config — see the `tpa` plugin — which is what the action-flow "Call API" node invokes.)

### Anatomy
- **Workspace**: a named group with a description and a set of `constants`.
- **API**: an HTTP `method` + URL under a workspace, with request **parameters** (path / query / header / body), typed **response configs** (the result shape downstream binds from), and declared **input variables** (the values a caller supplies).

### Workflow
List with `GET_ALL_API_WORKSPACES` / `GET_ALL_API_INFOS`, then `GET_API_DETAIL` to read an API's `apiId`, `workspaceId`, and parameter / response `uniqueId`s before editing — never fabricate them. Build top-down: `ADD_API_WORKSPACES` → `ADD_API_WORKSPACE_CONSTANTS` (put API keys / base URLs here, never inline) → `ADD_APIS` (each under a `workspaceId`) → `ADD_API_PARAMETERS`, `ADD_API_RESPONSE_CONFIGS`, `ADD_API_INPUT_VARIABLES`. Editing creates a new version; "Sync Backend" is required for changes to take effect in production.

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
| List workspaces | `GET_ALL_API_WORKSPACES` | — |
| List API endpoints | `GET_ALL_API_INFOS` | — |
| API detail (ids, params, responses) | `GET_API_DETAIL` | `apiId` |
| Add workspaces | `ADD_API_WORKSPACES` | `items` |
| Update a workspace | `UPDATE_API_WORKSPACE` | `workspaceId` |
| Delete workspaces | `DELETE_API_WORKSPACES` | `workspaceIds` |
| Add workspace constants | `ADD_API_WORKSPACE_CONSTANTS` | `items`, `workspaceId` |
| Update workspace constants | `UPDATE_API_WORKSPACE_CONSTANTS` | `items`, `workspaceId` |
| Delete workspace constants | `DELETE_API_WORKSPACE_CONSTANTS` | `constantNames`, `workspaceId` |
| Add API endpoints | `ADD_APIS` | `items` |
| Update an API endpoint | `UPDATE_API` | `apiId` |
| Delete API endpoints | `DELETE_APIS` | `apiIds` |
| Add request parameters | `ADD_API_PARAMETERS` | `apiId`, `items` |
| Update request parameters | `UPDATE_API_PARAMETERS` | `apiId`, `items` |
| Delete request parameters | `DELETE_API_PARAMETERS` | `apiId`, `items` |
| Add response configs | `ADD_API_RESPONSE_CONFIGS` | `apiId`, `items` |
| Update response configs | `UPDATE_API_RESPONSE_CONFIGS` | `apiId`, `items` |
| Delete response configs | `DELETE_API_RESPONSE_CONFIGS` | `apiId`, `uniqueIds` |
| Add input variables | `ADD_API_INPUT_VARIABLES` | `apiId`, `items` |
| Update input variables | `UPDATE_API_INPUT_VARIABLES` | `apiId`, `items` |
| Delete input variables | `DELETE_API_INPUT_VARIABLES` | `apiId`, `variableNames` |

Build top-down: `ADD_API_WORKSPACES` → `ADD_API_WORKSPACE_CONSTANTS` (API keys / base URLs) → `ADD_APIS` (each under a `workspaceId`) → `ADD_API_PARAMETERS` + `ADD_API_RESPONSE_CONFIGS` + `ADD_API_INPUT_VARIABLES`. Read `apiId` / `workspaceId` and parameter / response unique ids back from `GET_ALL_API_WORKSPACES` / `GET_API_DETAIL` before editing or deleting — never fabricate them. Bind a constant or input variable into a parameter value with `data-binding.md`.

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `ADD_API_WORKSPACES`

Create one or more API workspaces (name + description). Add their constants and APIs afterwards.
- `items` *(required)*: `array<{description?: string, displayName: string}>`

### `ADD_API_WORKSPACE_CONSTANTS`

Add shared constants (base URLs, API keys, tokens) to a workspace; its APIs reference them instead of inlining secrets.
- `items` *(required)*: `array<{name: string, type: string}>`
- `workspaceId` *(required)*: `string`

### `ADD_APIS`

Create one or more API endpoints (name, HTTP method, URL) under a workspaceId. Each is seeded with empty parameters / responses; add those afterwards.
- `items` *(required)*: `array<{displayName: string, inputVariables?: array<object>, method: enum(GET|POST|PUT|DELETE|PATCH|OPTIONS|HEAD), pagination?: object, responseConfigs?: array<object>, url: string, useAsData?: boolean, workspaceId: string}>`

### `UPDATE_API`

Update an endpoint's scalar config: name, method, URL, content type, or whether it is usable as data. Read the apiId from GET_ALL_API_INFOS.
- `apiId` *(required)*: `string`
- `displayName`: `string`
- `method`: `enum(GET|POST|PUT|DELETE|PATCH|OPTIONS|HEAD)`
- `pagination`: `{pageIndexPath?: array<string>, pageIndexStartValue?: integer, pageSizePath?: array<string>}`
- `url`: `string`
- `useAsData`: `boolean`

### `ADD_API_PARAMETERS`

Add request parameters to an API. Each carries a position (path / query / header / body) and a type.
- `apiId` *(required)*: `string`
- `items` *(required)*: `array<object>`

### `ADD_API_RESPONSE_CONFIGS`

Add typed response configs to an API — the shape of the JSON it returns, so downstream can bind to its fields.
- `apiId` *(required)*: `string`
- `items` *(required)*: `array<{isCustom?: boolean, name: string, responseType?: string, statusCode: array<string>}>`

### `ADD_API_INPUT_VARIABLES`

Declare input variables on an API — the values a caller supplies, bindable into the URL / parameters.
- `apiId` *(required)*: `string`
- `items` *(required)*: `array<{displayName: string, type: string}>`

Then ship:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema validate && "${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
