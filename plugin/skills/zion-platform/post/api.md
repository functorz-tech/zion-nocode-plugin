# API integrations (external HTTP data sources)

## API Integration Domain Knowledge
An API integration is a saved external HTTP endpoint the project can use as a data source. APIs are organized into **workspaces**: a workspace groups related endpoints and holds shared **constants** (e.g. a base URL or API key) the endpoints reference, so credentials live in one place. The action-flow "Call API" node invokes these workspace APIs.

### Anatomy
- **Workspace**: a named group with a description and a set of `constants`.
- **API**: an HTTP `method` + URL under a workspace, with request **parameters** (path / query / header / body), typed **response configs** (the result shape downstream binds from), and declared **input variables** (the values a caller supplies).

### Workflow
List with `GET_ALL_API_WORKSPACES` / `GET_ALL_APIS_INFO`, then `GET_API_DETAIL` to read an API's `apiId`, `workspaceId`, and parameter / response `uniqueId`s before editing — never fabricate them. Build top-down: `ADD_API_WORKSPACES` → `ADD_API_WORKSPACE_CONSTANTS` (put API keys / base URLs here, never inline) → `ADD_APIS` (each under a `workspaceId`) → `ADD_API_PARAMETERS`, `ADD_API_RESPONSE_CONFIGS`, `ADD_API_INPUT_VARIABLES`. Editing creates a new version; "Sync Backend" is required for changes to take effect in production.

> Available only on **post-type-system-refactor** projects; the daemon hard-gates every op below on pre-refactor projects, where the API-integration workspace feature does not exist. On a pre-refactor project integrate external HTTP endpoints as TPA configs (`third-party-api.md`) instead. Check `npx -y zion-mcp@2.7.2 schema load` → `typeSystem` first.

## How to drive it (CLI only)

All commands are `npx -y zion-mcp@2.7.2 <verb>`. A long-lived daemon holds the in-memory CRDT schema session
between calls. **Edits do NOT go live until `project sync-backend`.**

```bash
npx -y zion-mcp@2.7.2 whoami                                    # check auth; if needed: npx -y zion-mcp@2.7.2 login
# create a NEW project (auto-pins it; its pre/post type-system state follows the account rollout):
npx -y zion-mcp@2.7.2 project create --projectName "My App"
# …or pin an EXISTING one (find its exId with npx -y zion-mcp@2.7.2 projects search):
npx -y zion-mcp@2.7.2 project set-current --projectExId <exId>
npx -y zion-mcp@2.7.2 schema load                               # warm the schema session
```

Operations run through one verb:

```bash
npx -y zion-mcp@2.7.2 schema tool-call --toolCalls '[{"name":"<TOOL_NAME>","args":{ ... }}]'
```
Each call is applied immediately — any resulting CRDT patch is uploaded. Batch several calls in one array; use `schema undo` to revert the last change.
A batch is all-or-nothing: when any call in the array fails, the whole batch's changes are discarded even though the other calls returned success — only the failing call's error is reported, so after a batch error re-read (`GET_*`) before assuming anything persisted.

## Operation reference (`schema tool-call` names)

| Intent | `name` | Required `args` |
|---|---|---|
| List workspaces | `GET_ALL_API_WORKSPACES` | — |
| List API endpoints | `GET_ALL_APIS_INFO` | — |
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
| Move API endpoints between workspaces | `MOVE_APIS_TO_WORKSPACE` | `apiIds`, `targetWorkspaceId` |
| Body slots that accept a value conversion | `GET_API_CODEC_OPTIONS` | `apiId` |
| Set a body slot's value conversion | `SET_API_CODECS` | `apiId`, `items`, `target` |
| Remove a body slot's value conversion | `DELETE_API_CODECS` | `apiId`, `items`, `target` |
| Add request parameters | `ADD_API_PARAMETERS` | `apiId`, `items` |
| Update request parameters | `UPDATE_API_PARAMETERS` | `apiId`, `items` |
| Delete request parameters | `DELETE_API_PARAMETERS` | `apiId`, `items` |
| Add response configs | `ADD_API_RESPONSE_CONFIGS` | `apiId`, `items` |
| Update response configs | `UPDATE_API_RESPONSE_CONFIGS` | `apiId`, `items` |
| Delete response configs | `DELETE_API_RESPONSE_CONFIGS` | `apiId`, `uniqueIds` |
| Add input variables | `ADD_API_INPUT_VARIABLES` | `apiId`, `items` |
| Update input variables | `UPDATE_API_INPUT_VARIABLES` | `apiId`, `items` |
| Delete input variables | `DELETE_API_INPUT_VARIABLES` | `apiId`, `variableNames` |
| Duplicate an API with its private types | `DUPLICATE_API` | `sourceApiId` |
| Regroup APIs without copying them | `MOVE_APIS_TO_WORKSPACE` | `apiIds`, `targetWorkspaceId` |
| Slots that can carry a value conversion | `GET_API_CODEC_OPTIONS` | `apiId` |
| Set a body slot's wire conversion | `SET_API_CODECS` | `apiId`, `items`, `target` |
| Remove a body slot's wire conversion | `DELETE_API_CODECS` | `apiId`, `items`, `target` |

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
- `items` *(required)*: `array<{displayName: string, inputVariables?: array<object>, method: enum(GET|POST|PUT|DELETE|PATCH|OPTIONS|HEAD), paginationEnabled?: boolean, responseConfigs?: array<object>, url: string, useAsData?: boolean, workspaceId: string}>`

### `UPDATE_API`

Update an endpoint's scalar config: name, method, URL, content type, or whether it is usable as data. Read the apiId from GET_ALL_APIS_INFO.
- `apiId` *(required)*: `string`
- `displayName`: `string`
- `method`: `enum(GET|POST|PUT|DELETE|PATCH|OPTIONS|HEAD)` — New HTTP method. Switching from a body-less method (GET/DELETE/HEAD/OPTIONS) to one that carries a body defaults the API to a JSON body and seeds an empty object type for it; change or drop that with SET_API_CONTENT_TYPE.
- `paginationEnabled`: `boolean`
- `url`: `string` — New base URL, as a literal — scheme + host (+ port) only. To point the URL at a workspace constant or another value instead of a literal, bind it with the CREATE_*_BINDING tools at the `urlSchemaPath` GET_API_DETAIL reports.
- `useAsData`: `boolean`

### `ADD_API_PARAMETERS`

Add request parameters to an API. Each carries a position (path / query / header / body) and a type.
- `apiId` *(required)*: `string`
- `items` *(required)*: `array<object · location: PATH → {value: string} | JSON_BODY → {type: string} | QUERY|HEADER|PATH|FORM_BODY → {displayName?: string, name: string, type: string}>` — Parameters to add. The item shape is chosen by `location`: QUERY/HEADER/FORM_BODY (and variable PATH segments) carry name + type, a constant PATH segment carries `value`, and JSON_BODY carries just the body type.

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
npx -y zion-mcp@2.7.2 schema validate && npx -y zion-mcp@2.7.2 project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
