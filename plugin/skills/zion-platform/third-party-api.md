# Third-party API configs (TPA)

## Third-Party API (TPA) Domain Knowledge
A Third-Party API config is a saved external REST endpoint the project can call — from an action flow's "Call API" node or from Run Code via `context.callThirdPartyApi(apiId, params)`.

### Anatomy of a config
- **Endpoint**: a `url` (absolute, with scheme and host — the runtime has no base to resolve a relative path against) plus an HTTP `method` (GET/POST/PUT/DELETE) and an `operation` (`query` = read-only, `mutation` = state-changing). Optionally a request content type and a CA certificate (PEM, base64) for mutual TLS.
- **Request parameters**: typed inputs sent with the call. Each has a `position`: `QUERY` (query string), `PATH` (URL path segment), `HEADER`, or `BODY`. A parameter whose type is `OBJECT` or `ARRAY` holds nested **children**.
- **Response**: three branches keyed by outcome — `SUCCESS`, `PERMANENT_FAIL`, `TEMPORARY_FAIL`. Each branch carries a typed **response-data** tree describing the JSON the endpoint returns so downstream nodes can bind to its fields. `OBJECT`/`ARRAY` response data hold nested children.
- **Paging** (optional): pagination config for list endpoints.

### Data types
TPA fields use the TPA type set, independent of the project's type-system setting: TEXT, INTEGER, BIGINT, FLOAT8, DECIMAL, BOOLEAN, IMAGE, OBJECT, ARRAY. For an ARRAY give its `itemType`; for an OBJECT or ARRAY add the field shape with the `add_parameter_children` / `add_response_data_children` tools.

### Workflow
List configs with `list_configs`, then `get_config_detail` to read a config's parameter and response `uniqueId`s — you pass these ids to the child tools. Create endpoints with `create_configs`; add request inputs with `add_parameters` (and `add_parameter_children` for object/array fields). Describe each response branch with `set_response_data`, then `add_response_data_children` for nested fields. Configure list pagination with `set_paging`. Editing a config creates a new version; "Sync Backend" is required for changes to take effect in production.

### Importing from an OpenAPI / Swagger spec
To onboard an existing API, prefer `import_from_openapi` over hand-building configs: pass the spec as a `url` (fetched server-side; internal/private addresses are blocked) or paste it as `specText` (JSON or YAML). It creates each operation as a full TPA endpoint — parameters, body, and success/fail response shape — in one call, capped at 25 endpoints (use `pathsFilter` to scope larger specs); a failed import is rolled back. If the spec declares no absolute server URL the imported configs get relative urls — fix each with `update_config` using the base URL from the documentation. When you only have a human-readable docs page, `fetch_doc` retrieves it (SSRF-guarded) and lets you explore the documentation iteratively: follow the returned same-site `pageLinks`, keep reading long pages with `offset`, and watch `specLinks` for a machine-readable spec to import — all within a 15-fetch session budget. If no spec exists, author the configs with the granular tools above from what you read.

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
| List TPA configs | `GET_ALL_TPA_CONFIG_INFOS` | — |
| Config detail (ids) | `GET_TPA_CONFIG_DETAIL` | `tpaConfigId` |
| Add configs | `ADD_TPA_CONFIGS` | `items` |
| Update a config | `UPDATE_TPA_CONFIG` | `tpaConfigId` |
| Delete configs | `DELETE_TPA_CONFIGS` | `tpaConfigIds` |
| Add request parameters | `ADD_TPA_CONFIG_PARAMETERS` | `items`, `tpaConfigId` |
| Update a request parameter | `UPDATE_TPA_CONFIG_PARAMETER` | `parameterUniqueId`, `tpaConfigId` |
| Delete request parameters | `DELETE_TPA_CONFIG_PARAMETERS` | `parameterUniqueIds`, `tpaConfigId` |
| Add nested parameter children | `ADD_TPA_PARAMETER_CHILDREN` | `items`, `parentUniqueId`, `tpaConfigId` |
| Update a parameter child | `UPDATE_TPA_PARAMETER_CHILD` | `dataUniqueId`, `tpaConfigId` |
| Delete parameter children | `DELETE_TPA_PARAMETER_CHILDREN` | `dataUniqueIds`, `tpaConfigId` |
| Add response data | `ADD_TPA_RESPONSE_DATA` | `responseBranch`, `responseData`, `tpaConfigId` |
| Update response data | `UPDATE_TPA_RESPONSE_DATA` | `responseBranch`, `tpaConfigId` |
| Delete response data | `DELETE_TPA_RESPONSE_DATA` | `responseBranch`, `tpaConfigId` |
| Add nested response children | `ADD_TPA_RESPONSE_DATA_CHILDREN` | `items`, `parentUniqueId`, `responseBranch`, `tpaConfigId` |
| Update a response child | `UPDATE_TPA_RESPONSE_DATA_CHILD` | `dataUniqueId`, `responseBranch`, `tpaConfigId` |
| Delete response children | `DELETE_TPA_RESPONSE_DATA_CHILDREN` | `dataUniqueIds`, `responseBranch`, `tpaConfigId` |
| Set paging | `SET_TPA_CONFIG_PAGING` | `tpaConfigId` |

A TPA config is an external HTTP/GraphQL integration usable as a project data source. Add the
config (`ADD_TPA_CONFIGS`), then its request parameters and typed response data; nested object/array
fields are built with the `*_CHILDREN` / `*_CHILD` ops against a `parentUniqueId` read back from
`GET_TPA_CONFIG_DETAIL`. `responseBranch` selects SUCCESS / PERMANENT_FAIL / TEMPORARY_FAIL.

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `ADD_TPA_CONFIGS`
- `items` *(required)*: `array<{method: enum(GET|POST|PUT|DELETE), name: string, operation: enum(QUERY|MUTATION), url: string}>`

### `UPDATE_TPA_CONFIG`
- `caX509PemBase64`: `string`
- `description`: `string`
- `method`: `enum(GET|POST|PUT|DELETE)`
- `name`: `string`
- `operation`: `enum(QUERY|MUTATION)`
- `requestContentType`: `string`
- `tpaConfigId` *(required)*: `string`
- `url`: `string`

### `ADD_TPA_CONFIG_PARAMETERS`
- `items` *(required)*: `map<enum(BODY|HEADER|QUERY|PATH), {defaultValue?: string, itemType?: string, name: string, required: boolean, type: string}>`
- `tpaConfigId` *(required)*: `string`

### `UPDATE_TPA_CONFIG_PARAMETER`
- `defaultValue`: `string`
- `itemType`: `string`
- `name`: `string`
- `parameterUniqueId` *(required)*: `string`
- `required`: `boolean`
- `tpaConfigId` *(required)*: `string`
- `type`: `string`

### `ADD_TPA_RESPONSE_DATA`
- `responseBranch` *(required)*: `enum(SUCCESS|PERMANENT_FAIL|TEMPORARY_FAIL)`
- `responseData` *(required)*: `{defaultValue?: string, itemType?: string, name: string, required: boolean, type: string}`
- `tpaConfigId` *(required)*: `string`

### `UPDATE_TPA_RESPONSE_DATA`
- `defaultValue`: `string`
- `itemType`: `string`
- `name`: `string`
- `required`: `boolean`
- `responseBranch` *(required)*: `enum(SUCCESS|PERMANENT_FAIL|TEMPORARY_FAIL)`
- `tpaConfigId` *(required)*: `string`
- `type`: `string`

### `SET_TPA_CONFIG_PAGING`
- `pageIndexPath`: `array<string>`
- `pageIndexStartValue`: `integer`
- `pageSizePath`: `array<string>`
- `tpaConfigId` *(required)*: `string`

Then ship:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema validate && "${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
