# Data binding

## Data Binding Domain Knowledge
Data binding connects data sources to UI components for display or interaction.

### Concepts
Schema path: array of keys/indices locating a bindable node in the project JSON schema.
Data binding options tree: hierarchical tree of all available data for a given schema path.

### Binding Kinds
OPTION: bind to an existing path in the data binding options tree.
  pathInHierarchicalMenu must be the EXACT complete path. Never guess or fabricate.
  If the tree root is a named context (e.g., "List Context"), include it as the first element.
CONST_VALUE: set a literal constant (e.g., "Submit", 0, false).
FORMULA: compute a value with an operator (text / math / time / array / enum / geography / json
  groups); discover the available operators before building one.
CONDITIONAL: choose the value by branch — define the branches, then build each branch's predicate.
DISPLAY_NAME: rename a component's displayed title.

### Priority
1. Prefer OPTION (live data) over CONST_VALUE.
2. No suitable option? Use CONST_VALUE with a reasonable inferred constant.
3. Component purpose unclear? Skip binding.
4. List component with an ancestor list: prefer binding from the ancestor list's item data
   to preserve relational linkage — not directly from a table.

### Iterative Refinement
Per-op errors are returned keyed by index. Successful bindings have already landed.
On rejection, fix ONLY the failing entries by index and re-call.

### Data Sources Available in Zion
Database tables (with filters/sorting/limits), page data sources, page variables, global
variables, page parameters, logged-in user, list/component item data, Actionflow
inputs/outputs, formula results.

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
| Read options at a path | `GET_DATA_BINDING_OPTIONS` | `schemaPath` |
| Expected type at a path | `GET_DATA_BINDING_TYPE` | `schemaPath` |
| List formula operators | `GET_FORMULA_OPERATORS` | `schemaPath` |
| Bind to a live option | `CREATE_OPTION_BINDING` | `pathInHierarchicalMenu`, `schemaPath` |
| Bind a constant | `CREATE_CONST_BINDING` | `constantValue`, `schemaPath` |
| Bind a formula | `CREATE_FORMULA_BINDING` | `schemaPath` |
| Bind a conditional | `CREATE_CONDITIONAL_BINDING` | `schemaPath` |

## schemaPath — what it is & how to get one

A binding attaches a value to one bindable slot in the project schema, located by a `schemaPath`:
a `DiffPathComponents` array — `{"key":"..."}` for an object step, `{"index":N}` for an array
step (see *Arguments* for the exact shape). **Never hand-build one.** Every schemaPath is read back
from a discovery call for the slot you want:
- Action-flow node value / condition slots: `GET_ACTION_FLOW_CONTEXT_INFO`, plus the
  `schemaDiff.pathComponents` echoed by the node-creating call.
- Request `where` / sort: `GET_REQUEST_FILTER_CONTEXT` returns `whereSchemaPath`,
  `sortConfigsSchemaPath`, and `conditionSchemaPath`.
- Component properties: discover with the component tools, then bind at the returned path.

These context ops don't bootstrap from nothing — each *takes* a schemaPath and returns finer ones
(or the options / columns at it). Your **entry** path is always echoed by the op that created the
slot: `ADD_ACTION_FLOW_NODE`, `ADD_REQUEST_FILTER_CONDITION`, and the `CREATE_*_BINDING` ops each
return a `schemaDiff.pathComponents` for what they just made. Build top-down — create the node /
condition, read back its echoed path, then drill in — so you never need a path you don't already hold.

## Binding kinds

Pick the `CREATE_*` op for the slot — all keyed by `schemaPath`:
- **OPTION** (preferred — live data): `CREATE_OPTION_BINDING`. First call `GET_DATA_BINDING_OPTIONS`
  for that schemaPath and pass back its exact `pathInHierarchicalMenu` (include the root context
  name, e.g. "List Context", when present). Inside a list nested in another list, bind from the
  ancestor item to preserve the relation.
- **CONST_VALUE**: `CREATE_CONST_BINDING` — a literal (string / number / boolean) when no option fits.
- **FORMULA**: `CREATE_FORMULA_BINDING` — computed values; discover operators with
  `GET_FORMULA_OPERATORS` (groups GENERIC / TEXT / MATH / TIME / ARRAY / ENUM / GEOGRAPHY / JSON).
- **CONDITIONAL**: `CREATE_CONDITIONAL_BINDING`, then manage branches and build each predicate with
  the condition-expression ops.
Check the slot's expected type first with `GET_DATA_BINDING_TYPE`. Per-op errors come back keyed by
index — fix only the failing entries and retry.

## Request filters (where + sort)

Query / mutation requests and action-flow query / mutation nodes carry conditional filters (a
named-filter list, each a `where` boolean expression plus sort configs). **Always start from
`GET_REQUEST_FILTER_CONTEXT`**: it returns the selectable columns (each with a ready
`columnPathComponents` + `allowedOperators`) and the `whereSchemaPath` / `sortConfigsSchemaPath`
to address. Never hand-build paths.

### Filter & sort operations

| Intent | `name` | Required `args` |
|---|---|---|
| Selectable columns + sub-paths | `GET_REQUEST_FILTER_CONTEXT` | `schemaPath` |
| Add a where condition | `ADD_REQUEST_FILTER_CONDITION` | `columnPathComponents`, `schemaPath` |
| Set a condition operator | `UPDATE_REQUEST_FILTER_CONDITION_OPERATOR` | `operator`, `schemaPath` |
| Add a sort | `ADD_REQUEST_SORT_CONFIG` | `field`, `schemaPath` |

A condition's comparison `value` is itself a DataBinding — `ADD_REQUEST_FILTER_CONDITION` seeds
an empty one of the column's type; fill it afterwards with a `CREATE_*_BINDING` on the condition's
`value`. Every request keeps a mandatory "Default" filter that cannot be deleted or renamed.

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `GET_DATA_BINDING_OPTIONS`
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>`

### `CREATE_OPTION_BINDING`
- `operation`: `enum(REPLACE|CONCAT)` — Specifies how to apply the data: 'REPLACE' overwrites the current value, 'CONCAT' appends to it. Default is 'REPLACE'.
- `pathInHierarchicalMenu` *(required)*: `array<string>` — The full hierarchical path to the specific data option (VALUE) to be bound. This path corresponds to the '[Data Binding Options Tree]' in the context.
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — The specific schema path in the project where the data binding is occurring.

### `CREATE_CONST_BINDING`
- `constantValue` *(required)*: `boolean | string | number` — The constant value to set (e.g. 'Create Post', 0, true).
- `operation`: `enum(REPLACE|CONCAT)` — Specifies how to apply the data: 'REPLACE' overwrites the current value, 'CONCAT' appends to it. Default is 'REPLACE'.
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — The specific schema path in the project where the data binding is occurring.

### `CREATE_FORMULA_BINDING`
- `config`: `object · kind: enum(GEO_DISTANCE|GEO_POINT_GET_VALUE|DECIMAL|NUMBER_FORMATTING|DURATION_FORMATTING|DURATION|TIME_GET_PART|TIME_OPERATION|GET_CURRENT_TIME|DATE_TIME_FORMATTING|RELATIVE_TIME) · per-kind: {language: enum(EN|ZH)} | {clearTrailingZeros: boolean, roundingMode: enum(HALF_EVEN|HALF_UP|HALF_DOWN|UP|DOWN|CEILING|FLOOR)} | {dateTimeUnit: enum(YEAR|MONTH|DAY|HOUR|MINUTE|SECOND|MILLISECOND|WEEKDAY|WEEK)} | {timeUnit: enum(DAY|HOUR|MINUTE|SECOND|MILLISECONDS)} | {unit: enum(METER|KILOMETER|MILE)} | {getValueType: enum(LATITUDE|LONGITUDE)} | {timeType: enum(DATE|TIME|TIMESTAMP)} | {decimalPlaces?: integer, format: enum(THOUSANDS_SEPARATOR|PERCENT)} | {hideSuffix?: boolean, language: enum(EN|ZH)} | {direction: enum(LATER|BEFORE)}` — Optional non-databinding scalar config for the created formula (e.g. distance unit, rounding mode, time unit). The config kind must match the chosen operator, otherwise the call fails. Use GET_FORMULA_CONFIG_OPTIONS on an existing formula to discover the exact shape and the allowed values.
- `operation`: `enum(REPLACE|CONCAT)`
- `operator`: `enum(TO_STRING|TO_INTEGER|TO_DECIMAL|TO_DATE_TIME|ADD|SUBTRACT|MULTIPLY|DIVIDE|MODULO|MIN|… 80 total)`
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>`

### `CREATE_CONDITIONAL_BINDING`
- `initialBranchNames`: `array<string>`
- `operation`: `enum(REPLACE|CONCAT)`
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>`

### `GET_REQUEST_FILTER_CONTEXT`
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>`

### `ADD_REQUEST_FILTER_CONDITION`
- `columnPathComponents` *(required)*: `array<{componentMRef?: string, displayName?: string, isArrayElementMapping?: boolean, isArrayType: boolean, itemType?: string, name: string, tpaResultSource?: enum(SUCCESS|PERMANENT_FAIL|TEMPORARY_FAIL), type?: string}>` — The column to filter on, as a `pathComponents` list copied verbatim from a GET_REQUEST_FILTER_CONTEXT `selectableColumns` entry — never hand-built.
- `operator`: `string` — Comparison operator (e.g. `_eq`, `_neq`, `_gt`, `_lt`, `_gte`, `_lte`, `_is_null`, `_is_not_null`, `_in`, `_nin`, `_like`, `_ilike`). Defaults to `_eq`. Pick one from the column's `allowedOperators` in GET_REQUEST_FILTER_CONTEXT.
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — Schema path of the target conditional filter's `where` (the `whereSchemaPath` from GET_REQUEST_FILTER_CONTEXT), or a nested AND/OR sub-expression within it to append into.
- `value`: `object` — Optional comparison value binding; an empty value of the column's type is seeded when omitted, to be filled afterwards with a CREATE_*_BINDING tool on this condition's `value`.

### `UPDATE_REQUEST_FILTER_CONDITION_OPERATOR`
- `operator` *(required)*: `string` — New comparison operator (see ADD_REQUEST_FILTER_CONDITION for the allowed values).
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — Schema path of the column condition whose operator should change.

### `ADD_REQUEST_SORT_CONFIG`
- `field` *(required)*: `array<{componentMRef?: string, displayName?: string, isArrayElementMapping?: boolean, isArrayType: boolean, itemType?: string, name: string, tpaResultSource?: enum(SUCCESS|PERMANENT_FAIL|TEMPORARY_FAIL), type?: string}>` — The column to sort by, as a `pathComponents` list copied from a GET_REQUEST_FILTER_CONTEXT `selectableColumns` entry.
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — Schema path of the target conditional filter's `filter` (the `sortConfigsSchemaPath` from GET_REQUEST_FILTER_CONTEXT).
- `sort`: `enum(DESCENDING|ASCENDING)` — Sort direction; defaults to ascending.

Then ship:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema validate && "${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
