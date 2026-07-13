# Data binding

## Data Binding Domain Knowledge
Data binding connects data sources to UI components for display or interaction.

### Concepts
Schema path: array of keys/indices locating a bindable node in the project JSON schema. Data binding options tree: hierarchical tree of all available data for a given schema path. Never hand-build a schema path for an action-flow node: copy it from the actionflow plugin's add_node result or get_flow_detail's nodeSchemaPaths (form: server/actionFlows/{i}/allNodes/{j}), then append key segments for the binding site inside the node (e.g. inputArgsDataBinding/<argName>, mutation/object/<columnName>).
Type metadata in read results uses exact typeIdentifier values such as `s:p:string` and `u:e:<enumId>`; copy those values verbatim.

### Binding Kinds
OPTION: bind to an existing path in the data binding options tree. pathInHierarchicalMenu must be the EXACT complete path: copy every label verbatim from the options tree attached to binding tool responses (or from a failure's available-options list) — never invent, translate, or shorten a label, and include every intermediate menu level. If the tree root is a named context (e.g., "List Context"), include it as the first element. CONST_VALUE: set a literal constant (e.g., "Submit", 0, false). FORMULA: compute a value with an operator (text / math / time / array / enum / geography / json groups); discover the available operators before building one. CONDITIONAL: choose the value by branch — define the branches, then build each branch's predicate. DISPLAY_NAME: rename a component's displayed title.

### Priority
1. Prefer OPTION (live data) over CONST_VALUE.
2. No suitable option? Use CONST_VALUE with a reasonable inferred constant.
3. Component purpose unclear? Skip binding.
4. List component with an ancestor list: prefer binding from the ancestor list's item data to preserve relational linkage — not directly from a table.
5. Bind ONLY what the task requires: in an update/insert node's object leave every other column untouched — never fill it with a Null option or a placeholder constant (Null on a required column breaks the flow at runtime).
6. Never seed a typed site with a throwaway constant "for now" and upgrade later — bind the real value directly; a constant re-types nothing but still wastes a round.

### Iterative Refinement
Per-op errors are returned keyed by index. Successful bindings have already landed. On rejection, fix ONLY the failing entries by index and re-call.

### Data Sources Available in Zion
Database tables (with filters/sorting/limits), page data sources, page variables, global variables, page parameters, logged-in user, list/component item data, Actionflow inputs/outputs, formula results.

### Request Filters (where / sort)
A database-query data source — a list/component request, or an action-flow query / update / delete node — carries **request filters**. FIRST call `GET_REQUEST_FILTER_CONTEXT` with the request's (or owning node's) schema path: it returns the table's selectable columns (with the `pathComponents` and `allowedOperators` to use) and the filter entries' `whereSchemaPath` / `sortConfigsSchemaPath`. Copy those paths and column `pathComponents` verbatim — never hand-build them. Queries and component requests carry a LIST of conditional filters (where + sort groups, each gated by its own condition, always including a system Default filter). An action-flow update/delete node instead carries ONE bare where (`supportsConditionalFilters: false` in the context result) — edit it via the single entry's `whereSchemaPath` with the where-condition tools only; the filter-group and sort tools do not apply, and insert nodes have no request filter at all.
To change a numeric column RELATIVE to its current value (stock, counters, balances), set the column's write mode with `SET_UPDATE_COLUMN_ARITHMETIC` and bind only the DELTA as the column's value — the database applies it atomically. Never query the current value, compute in a formula, and set it back: that read-modify-write loses concurrent updates. Idiom — decrease field_a by x: mode 'decrement' on field_a, value x, and a where guard `field_a > x` (or `>= x` when draining to zero is allowed) so the update changes nothing when the stored value is insufficient. That rule generalizes to every mutation this plugin configures — frontend component mutation events and action-flow nodes alike: when a write's correctness depends on current row state, put the check in the mutation's own where filter and let follow-ups branch on its affected rows, instead of querying first and writing on the stale read (concurrent users race it). Insert-if-absent semantics additionally need a unique constraint — the only DB-enforced constraint kind today (see the database plugin).
- Filter groups: `ADD_REQUEST_CONDITIONAL_FILTER` / `UPDATE_REQUEST_CONDITIONAL_FILTER` / `DELETE_REQUEST_CONDITIONAL_FILTER` / `REORDER_REQUEST_CONDITIONAL_FILTERS`. The Default filter cannot be renamed or deleted.
- Where conditions: `ADD_REQUEST_FILTER_CONDITION` (column + operator), then fill its comparison value with a binding (OPTION / CONST_VALUE) at the condition's `value` path; shape the expression with `NEST_REQUEST_FILTER_CONDITION`, `TOGGLE_REQUEST_FILTER_CONDITION_AND_OR`, `TOGGLE_REQUEST_FILTER_CONDITION_NOT`, `UPDATE_REQUEST_FILTER_CONDITION_OPERATOR`, `DELETE_REQUEST_FILTER_CONDITION`.
- Sort: `ADD_REQUEST_SORT_CONFIG` / `UPDATE_REQUEST_SORT_CONFIG` / `DELETE_REQUEST_SORT_CONFIG` / `REORDER_REQUEST_SORT_CONFIGS`.

### Composing text from fragments and variables
For any `s:p:string` value that mixes literal fragments with variables — AI prompt templates, titles, labels, messages — do NOT build a stringConcat formula. Append the pieces IN ORDER at the same schema path with operation CONCAT: `CREATE_CONST_BINDING` (operation CONCAT) for each literal fragment, `CREATE_OPTION_BINDING` (operation CONCAT) for each variable. This produces the same inline template the editor's text fields produce and stays editable there. Reserve formulas for real computation (math, time, regex, arrays) — never for simple text templating.

### Formula & Conditional Bindings
A FORMULA or CONDITIONAL value is built in several steps at the value's schema path, not in a single call.
- Formula: call `GET_FORMULA_OPERATORS` at the path, then `CREATE_FORMULA_BINDING` with an operator copied verbatim (the wire token, e.g. `-` not SUBTRACT; operation REPLACE). If the operator needs scalar config (e.g. distance unit, rounding mode), read its shape with `GET_FORMULA_CONFIG_OPTIONS` and pass it or apply it later with `SET_FORMULA_CONFIG`. The create result returns `operandSchemaPaths` — fill each operand with an OPTION / CONST_VALUE binding at its returned path, copied verbatim; never derive an operand path yourself.
- Conditional: `CREATE_CONDITIONAL_BINDING` (seed branches with initialBranchNames; operation REPLACE), add more with `INSERT_CONDITIONAL_DATA`. For each branch, build its predicate with `INSERT_CONDITION_BOOL_EXP`, choose the comparison via `GET_EXPRESSION_CONDITION_OPERATORS` + `UPDATE_EXPRESSION_CONDITION_OPERATOR`, fill both operands with OPTION / CONST_VALUE bindings, and shape the expression with `NEST_CONDITION_BOOL_EXP`, `TOGGLE_CONDITION_BOOL_EXP_AND_OR`, `TOGGLE_CONDITION_BOOL_EXP_NOT`, `DELETE_CONDITION_BOOL_EXP`; then bind the branch's resulting value. The first branch whose predicate is true wins — order them with `REORDER_CONDITIONAL_DATA`.

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
| Read options at a path | `GET_DATA_BINDING_OPTIONS` | `schemaPath` |
| Expected type at a path | `GET_DATA_BINDING_TYPE` | `schemaPath` |
| List formula operators | `GET_FORMULA_OPERATORS` | `schemaPath` |
| Formula config options at a path | `GET_FORMULA_CONFIG_OPTIONS` | `schemaPath` |
| Bind to a live option | `CREATE_OPTION_BINDING` | `pathInHierarchicalMenu`, `schemaPath` |
| Bind a constant | `CREATE_CONST_BINDING` | `constantValue`, `schemaPath` |
| Bind a formula | `CREATE_FORMULA_BINDING` | `schemaPath` |
| Set a formula config | `SET_FORMULA_CONFIG` | `config`, `schemaPath` |
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

### Conditional bindings & predicate expressions

| Intent | `name` | Required `args` |
|---|---|---|
| Add a conditional branch | `INSERT_CONDITIONAL_DATA` | `schemaPath` |
| Update a conditional branch | `UPDATE_CONDITIONAL_DATA` | `conditionData`, `id`, `schemaPath` |
| Delete a conditional branch | `DELETE_CONDITIONAL_DATA` | `id`, `schemaPath` |
| Reorder conditional branches | `REORDER_CONDITIONAL_DATA` | `reorderedIds`, `schemaPath` |
| Condition operators at a path | `GET_EXPRESSION_CONDITION_OPERATORS` | `schemaPath` |
| Add a predicate (bool expr) | `INSERT_CONDITION_BOOL_EXP` | `schemaPath` |
| Set a predicate operator | `UPDATE_EXPRESSION_CONDITION_OPERATOR` | `operator`, `schemaPath` |
| Delete a predicate | `DELETE_CONDITION_BOOL_EXP` | `schemaPath` |
| Nest a predicate (group) | `NEST_CONDITION_BOOL_EXP` | `schemaPath` |
| Toggle AND/OR on a group | `TOGGLE_CONDITION_BOOL_EXP_AND_OR` | `schemaPath` |
| Toggle NOT on a predicate | `TOGGLE_CONDITION_BOOL_EXP_NOT` | `schemaPath` |

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
| Add a named filter | `ADD_REQUEST_CONDITIONAL_FILTER` | `schemaPath` |
| Rename a named filter | `UPDATE_REQUEST_CONDITIONAL_FILTER` | `id`, `name`, `schemaPath` |
| Delete a named filter | `DELETE_REQUEST_CONDITIONAL_FILTER` | `id`, `schemaPath` |
| Reorder named filters | `REORDER_REQUEST_CONDITIONAL_FILTERS` | `reorderedIds`, `schemaPath` |
| Add a where condition | `ADD_REQUEST_FILTER_CONDITION` | `columnPathComponents`, `schemaPath` |
| Set a condition operator | `UPDATE_REQUEST_FILTER_CONDITION_OPERATOR` | `operator`, `schemaPath` |
| Delete a where condition | `DELETE_REQUEST_FILTER_CONDITION` | `schemaPath` |
| Nest a where condition (group) | `NEST_REQUEST_FILTER_CONDITION` | `schemaPath` |
| Toggle AND/OR on a group | `TOGGLE_REQUEST_FILTER_CONDITION_AND_OR` | `schemaPath` |
| Toggle NOT on a condition | `TOGGLE_REQUEST_FILTER_CONDITION_NOT` | `schemaPath` |
| Add a sort | `ADD_REQUEST_SORT_CONFIG` | `field`, `schemaPath` |
| Update a sort | `UPDATE_REQUEST_SORT_CONFIG` | `index`, `schemaPath` |
| Delete a sort | `DELETE_REQUEST_SORT_CONFIG` | `index`, `schemaPath` |
| Reorder sorts | `REORDER_REQUEST_SORT_CONFIGS` | `reorderedIndexes`, `schemaPath` |

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `GET_DATA_BINDING_OPTIONS`
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — Schema path addressing the target element, taken from a read tool's output (e.g. a conditionSchemaPath / checkSchemaPath from GET_ROLE_DETAIL, node and binding paths from the entity detail tools); never hand-built.

### `GET_DATA_BINDING_TYPE`
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — Schema path addressing the target element, taken from a read tool's output (e.g. a conditionSchemaPath / checkSchemaPath from GET_ROLE_DETAIL, node and binding paths from the entity detail tools); never hand-built.

### `CREATE_OPTION_BINDING`
- `operation`: `enum(REPLACE|CONCAT)` — Specifies how to apply the data: 'REPLACE' overwrites the current value, 'CONCAT' appends to it. Default is 'REPLACE'.
- `pathInHierarchicalMenu` *(required)*: `array<string>` — The full hierarchical path to the specific data option (VALUE) to be bound. This path corresponds to the '[Data Binding Options Tree]' in the context.
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — The specific schema path in the project where the data binding is occurring.

### `CREATE_CONST_BINDING`
- `constantValue` *(required)*: `boolean | string | number` — The constant value to set (e.g. 'Create Post', 0, true).
- `operation`: `enum(REPLACE|CONCAT)` — Specifies how to apply the data: 'REPLACE' overwrites the current value, 'CONCAT' appends to it. Default is 'REPLACE'.
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — The specific schema path in the project where the data binding is occurring.

### `CREATE_FORMULA_BINDING`

Create a formula binding at a schema path that computes a value with an operator (from GET_FORMULA_OPERATORS, copied verbatim — e.g. '-' not SUBTRACT); operation is REPLACE (set the value) or CONCAT (append). Some operators need a scalar config (e.g. distance unit, rounding mode) — discover its shape with GET_FORMULA_CONFIG_OPTIONS and pass it, or set it later with SET_FORMULA_CONFIG. The result returns formulaSchemaPath and operandSchemaPaths — fill each operand afterwards with a const/option binding tool at its returned path, copied verbatim; never derive an operand path yourself.
- `config`: `object · kind: enum(GEO_DISTANCE|GEO_POINT_GET_VALUE|DECIMAL|NUMBER_FORMATTING|DURATION_FORMATTING|DURATION|TIME_GET_PART|TIME_OPERATION|GET_CURRENT_TIME|DATE_TIME_FORMATTING|RELATIVE_TIME) · per-kind: {language: enum(EN|ZH)} | {clearTrailingZeros: boolean, roundingMode: enum(HALF_EVEN|HALF_UP|HALF_DOWN|UP|DOWN|CEILING|FLOOR)} | {dateTimeUnit: enum(year|month|day|hour|minute|second|millisecond|weekday|week)} | {timeUnit: enum(day|hour|minute|second|millisecond)} | {unit: enum(METER|KILOMETER|MILE)} | {getValueType: enum(latitude|longitude)} | {timeType: enum(DATE|TIME|TIMESTAMP)} | {decimalPlaces?: integer, format: enum(THOUSANDS_SEPARATOR|PERCENT)} | {hideSuffix?: boolean, language: enum(EN|ZH)} | {direction: enum(later|before)}` — Optional non-databinding scalar config for the created formula (e.g. distance unit, rounding mode, time unit). The config kind must match the chosen operator, otherwise the call fails. Use GET_FORMULA_CONFIG_OPTIONS on an existing formula to discover the exact shape and the allowed values.
- `operation`: `enum(REPLACE|CONCAT)`
- `operator`: `enum(toText|toInteger|toDecimal|toDateTime|+|-|*|/|%|min|… 80 total)`
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>`

### `SET_FORMULA_CONFIG`

Set the non-databinding scalar config of an existing formula (e.g. rounding mode, time unit); the shape must match the operator — read it with GET_FORMULA_CONFIG_OPTIONS.
- `config` *(required)*: `object · kind: enum(GEO_DISTANCE|GEO_POINT_GET_VALUE|DECIMAL|NUMBER_FORMATTING|DURATION_FORMATTING|DURATION|TIME_GET_PART|TIME_OPERATION|GET_CURRENT_TIME|DATE_TIME_FORMATTING|RELATIVE_TIME) · per-kind: {language: enum(EN|ZH)} | {clearTrailingZeros: boolean, roundingMode: enum(HALF_EVEN|HALF_UP|HALF_DOWN|UP|DOWN|CEILING|FLOOR)} | {dateTimeUnit: enum(year|month|day|hour|minute|second|millisecond|weekday|week)} | {timeUnit: enum(day|hour|minute|second|millisecond)} | {unit: enum(METER|KILOMETER|MILE)} | {getValueType: enum(latitude|longitude)} | {timeType: enum(DATE|TIME|TIMESTAMP)} | {decimalPlaces?: integer, format: enum(THOUSANDS_SEPARATOR|PERCENT)} | {hideSuffix?: boolean, language: enum(EN|ZH)} | {direction: enum(later|before)}` — The non-databinding scalar config to apply to the formula at the path. The config kind must match the target formula; call GET_FORMULA_CONFIG_OPTIONS first to discover the exact shape and the allowed values for each field.
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — The schema path of the formula binding whose scalar config should be set.

### `CREATE_CONDITIONAL_BINDING`

Create a conditional binding at a schema path: a value chosen by the first branch whose predicate is true. initialBranchNames seeds the branches; operation is REPLACE or CONCAT. Add more branches with INSERT_CONDITIONAL_DATA, build each branch's predicate with the condition tools below, and fill each branch value with a binding tool.
- `initialBranchNames`: `array<string>`
- `operation`: `enum(REPLACE|CONCAT)`
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>`

### `INSERT_CONDITIONAL_DATA`

Add a branch (a predicate plus the value it yields) to a conditional binding, after an existing branch id.
- `afterId`: `string`
- `conditionData`: `{condition?: object, name?: string, value?: object}`
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>`

### `UPDATE_CONDITIONAL_DATA`

Update a branch of a conditional binding (e.g. rename it) by its schema path.
- `conditionData` *(required)*: `{condition?: object, name?: string, value?: object}`
- `id` *(required)*: `string`
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>`

### `INSERT_CONDITION_BOOL_EXP`

Add a comparison to a branch's predicate at the given where-expression schema path. Returns the new comparison's conditionSchemaPath — copy it verbatim into UPDATE_EXPRESSION_CONDITION_OPERATOR and extend it with /target and /value for the operand bindings; never hand-build condition paths.
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — Schema path addressing the target element, taken from a read tool's output (e.g. a conditionSchemaPath / checkSchemaPath from GET_ROLE_DETAIL, node and binding paths from the entity detail tools); never hand-built.

### `UPDATE_EXPRESSION_CONDITION_OPERATOR`

Set the comparison operator of a condition at the given schema path (values from GET_EXPRESSION_CONDITION_OPERATORS).
- `operator` *(required)*: `string` — Operator wire name (e.g. '_eq', '_gt', '_in') — pick one returned by GET_EXPRESSION_CONDITION_OPERATORS for this condition.
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — Schema path of the condition whose operator to set, from a read tool's output.

### `GET_REQUEST_FILTER_CONTEXT`

Inspect a request's filters before editing them. Call this first; never hand-build a filter or column path — copy the column paths and operators it returns verbatim. Pass the schema path of the query/mutation request, or of the action-flow query/mutation node that owns it (from GET_ACTION_FLOW_DETAIL). When the result says supportsConditionalFilters: false (an action-flow update/delete node has ONE bare where, not filter groups), only the where-condition tools apply — use the single entry's whereSchemaPath and skip the conditional-filter and sort tools. Insert nodes have no request filter.
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — Schema path addressing the target element, taken from a read tool's output (e.g. a conditionSchemaPath / checkSchemaPath from GET_ROLE_DETAIL, node and binding paths from the entity detail tools); never hand-built.

### `ADD_REQUEST_CONDITIONAL_FILTER`

Add a named conditional filter (a where + sort group gated by its own condition) to a request. The always-applied Default filter already exists; add more only for conditionally-applied filtering.
- `afterId`: `string` — Insert the new filter right after the conditional filter with this id; inserts at the front when omitted. Must reference a non-Default filter: the Default filter is always the last branch (its always-true condition would short-circuit any filter placed after it), so a new filter is always placed before the Default filter — passing the Default filter's id is rejected.
- `name`: `string` — Display name for the new conditional filter; auto-generated when omitted.
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — Schema path of the query/mutation request (or the action-flow query/mutation node that owns it). Get it from GET_REQUEST_FILTER_CONTEXT.

### `UPDATE_REQUEST_CONDITIONAL_FILTER`

Rename a conditional filter by id (from GET_REQUEST_FILTER_CONTEXT). The Default filter cannot be renamed.
- `id` *(required)*: `string` — Id of the conditional filter to rename; the Default filter cannot be renamed.
- `name` *(required)*: `string` — New display name for the conditional filter.
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>`

### `ADD_REQUEST_FILTER_CONDITION`

Add a column where-condition to a conditional filter. Target the filter's whereSchemaPath (or a nested AND/OR within it) from GET_REQUEST_FILTER_CONTEXT, pass the column's pathComponents copied verbatim from selectableColumns, and pick an operator from that column's allowedOperators (defaults to _eq). The comparison value is seeded empty — fill it afterwards with a binding tool at the condition's value path.
- `columnPathComponents` *(required)*: `array<{componentMRef?: string, displayName?: string, isArrayElementMapping?: boolean, isArrayType: boolean, itemType?: string, name: string, tpaResultSource?: enum(2xx|4xx|5xx), type?: string}>` — The column to filter on, as a `pathComponents` list copied verbatim from a GET_REQUEST_FILTER_CONTEXT `selectableColumns` entry — never hand-built.
- `operator`: `string` — Comparison operator (e.g. `_eq`, `_neq`, `_gt`, `_lt`, `_gte`, `_lte`, `_is_null`, `_is_not_null`, `_in`, `_nin`, `_like`, `_ilike`). Defaults to `_eq`. Pick one from the column's `allowedOperators` in GET_REQUEST_FILTER_CONTEXT.
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — Schema path of the target conditional filter's `where` (the `whereSchemaPath` from GET_REQUEST_FILTER_CONTEXT), or a nested AND/OR sub-expression within it to append into.
- `value`: `object` — Optional comparison value binding; an empty value of the column's type is seeded when omitted, to be filled afterwards with a CREATE_*_BINDING tool on this condition's `value`.

### `UPDATE_REQUEST_FILTER_CONDITION_OPERATOR`

Change the comparison operator of an existing column where-condition at the given schema path (same operator values as ADD_REQUEST_FILTER_CONDITION).
- `operator` *(required)*: `string` — New comparison operator (see ADD_REQUEST_FILTER_CONDITION for the allowed values).
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — Schema path of the column condition whose operator should change.

### `ADD_REQUEST_SORT_CONFIG`

Add a sort rule to a conditional filter. Target the filter's sortConfigsSchemaPath from GET_REQUEST_FILTER_CONTEXT and pass the column pathComponents copied from selectableColumns; direction defaults to ascending.
- `field` *(required)*: `array<{componentMRef?: string, displayName?: string, isArrayElementMapping?: boolean, isArrayType: boolean, itemType?: string, name: string, tpaResultSource?: enum(2xx|4xx|5xx), type?: string}>` — The column to sort by, as a `pathComponents` list copied from a GET_REQUEST_FILTER_CONTEXT `selectableColumns` entry.
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>` — Schema path of the target conditional filter's `filter` (the `sortConfigsSchemaPath` from GET_REQUEST_FILTER_CONTEXT).
- `sort`: `enum(descending|ascending)` — Sort direction; defaults to ascending.

### `UPDATE_REQUEST_SORT_CONFIG`

Change a sort rule's column and/or direction by its index within the filter's sortConfigs.
- `field`: `array<{componentMRef?: string, displayName?: string, isArrayElementMapping?: boolean, isArrayType: boolean, itemType?: string, name: string, tpaResultSource?: enum(2xx|4xx|5xx), type?: string}>` — New sort column path components; unchanged when omitted.
- `index` *(required)*: `integer` — Index of the sort config to update within the filter's `sortConfigs`.
- `schemaPath` *(required)*: `array<{index?: integer, key?: string}>`
- `sort`: `enum(descending|ascending)` — New sort direction; unchanged when omitted.

Then ship:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema validate && "${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
