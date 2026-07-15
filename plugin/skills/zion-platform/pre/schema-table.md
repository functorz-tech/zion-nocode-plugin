# Database schema (tables, fields, relations, constraints)

## Database Domain Knowledge
Zion uses PostgreSQL. Data model changes require "Sync Backend" to take effect online.

### Capabilities & Limitations
You can: create and delete tables, fields, relations, and unique constraints; rename a table; rename a column or change its displayName / required / unique / default value.

You CANNOT:
- Change an existing column's TYPE. To change a type, delete the column and create a new one with the desired type.
- Update an existing relation or constraint. To change one, delete it and recreate it.
- Create formula / computed fields. If the user asks for one, explain that formulas must be configured manually in the editor; do not create them.

### Column Types
A column's 'type' is an exact legacy ColumnType name (case-sensitive):
{{legacyColumnTypes}}.

Put one of those values directly in a column's 'type' field (e.g. "DECIMAL"). Do not use
refactored primitive/enum identifiers in this project.

### Naming
systemName: English snake_case. Tables are nouns or noun phrases, singular not plural ("order", not "orders"), concise (e.g. "user_profile"); fields are snake_case (e.g. "first_name", "is_active"). displayName: user-visible; use the natural Chinese name as the displayName and derive the English snake_case systemName from it (e.g. displayName "用户名" → systemName "user_name").

### System Built-ins & Product Context
Every table has non-deletable built-in fields: id (BIGINT), created_at (TIMESTAMPTZ), updated_at (TIMESTAMPTZ). Do NOT include these when creating a table. Any table, field, or relation where 'editable' is false is system built-in and cannot be modified or deleted. System tables and timezone configurations for Zion:
* UTC Offset: +08:00
* Protected Account Table: '帐户(账户)' (can add/delete user-defined fields, but cannot delete table itself)
* Protected Payment Tables: '支付', '周期性支付', '退款' (cannot be modified or deleted)
* Protected AI/Session Tables: '会话', '消息', '工具使用记录', '消息内容' (cannot be modified or deleted) The system built-in AI tables are strictly for system AI functions. For user chat systems, always create custom user-defined tables (e.g. 'user_chat', 'chat_message').

### Required Fields & Default Values
When creating a new field with 'required = true', a default value must be set simultaneously, except for types that do not support default values. Default value formatting:
- Numbers/Booleans: Use literal values (e.g., 10, true).
- Dates/Times: Strictly use ISO 8601 strings (e.g., TIMESTAMPTZ: '2025-12-09T16:02:03.000Z', DATE: '2025-12-09', TIMETZ: '16:02:03+08:00').
- TEXT: Use plain strings.
- JSONB: Use stringified JSON objects.
- Unsupported: IMAGE, VIDEO, FILE, and GEO_POINT do not support default values.

### Relations
Types: one_to_one, one_to_many. Defined on the source table.
To make one table reference another, create a RELATION — never add a manual foreign-key column (e.g. a "*_id" field) or a column whose type is another table. The FK column and the virtual reference fields are generated automatically. Add the relation on the SOURCE table only; it is reflected on the target automatically.
A relation is configured by five fields: source table, target table, source-side field display name, target-side field display name, and relation type. Relation record fields: relationType, sourceTableDisplayName, targetTableDisplayName, fieldDisplayNameInSourceTable, fieldDisplayNameInTargetTable, editable.
Creating a relation auto-generates:
- A non-editable FK field in the target table named fieldDisplayNameInTargetTable + "_id" (e.g. "user_id", "活动_id"). Stores the source row's id. Deleted when the relation is deleted.
- Virtual reference fields in both tables (NOT real columns), one per side: fieldDisplayNameInSourceTable lives on the source; fieldDisplayNameInTargetTable lives on the target.
Examples:
- 1:n user (source) → post (target), source field "posts", target field "user"
⇒ user has virtual list "posts"; post has virtual reference "user" and FK column "user_id".
- 1:n post (source) → comment (target), source field "comments", target field "post"
⇒ post has virtual list "comments"; comment has virtual reference "post" and FK column "post_id".
- 1:1 user (source) → profile (target), source field "profile", target field "user"
⇒ user has virtual reference "profile"; profile has virtual reference "user" and FK column "user_id".
Relations only between editable (user-created) tables.
Many-to-many: use an intermediate join table + two one-to-many relations.
To unique-constrain a relation field, use the FK field name ("user_id"), not the virtual name.

### Geographic Location
Use GEO_POINT for coordinates. Never split into separate latitude/longitude fields. A GEO_POINT field auto-generates a companion DECIMAL hack field named "fz_distance_from_<systemName>", where <systemName> is the geo_point's systemName (may differ from its displayName). At request time it returns the distance from the stored geo_point to the user-supplied location in the request. Treat it as a distance-calculation hack — future migration: this will be replaced by formula fields.

### Constraints
Only unique constraints supported. Constraint name: lowercase English snake_case, no uppercase. Reference columns by their system name (snake_case), not their display name — for a relation's foreign key use the FK column name ("user_id"). A single-column unique constraint can also be set via a field's 'unique' flag; use a constraint to span multiple columns (composite unique): list the columns' system names. Unique constraints are also the only DB-enforced invariant writers can lean on for atomic insert-if-absent / toggle semantics (insert with on_conflict): whenever the design has "at most one row per X" semantics (a join/like/save table, an idempotency key), create the unique constraint up front — read-check-then-insert cannot be made race-safe without it.

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
| Project overview | `GET_PROJECT_OVERVIEW` | — |
| Entity relation graph | `GET_ENTITY_RELATION_GRAPH` | `entityId`, `entityType` |
| List table names | `GET_ALL_TABLE_DISPLAY_NAMES` | — |
| Inspect tables | `GET_TABLE_INFOS` | `tableDisplayNames` |
| Create tables | `ADD_TABLES` | `items` |
| Delete tables | `DELETE_TABLES` | `tableDisplayNames` |
| Add fields/relations | `ADD_FIELDS_AND_RELATIONS` | `fields`, `relations`, `tableDisplayName` |
| Delete fields/relations | `DELETE_FIELDS_AND_RELATIONS` | `fieldDisplayNames`, `relationFieldDisplayNamesInSourceTable`, `tableDisplayName` |
| Add unique constraints | `ADD_CONSTRAINTS` | `constraints` |
| Delete unique constraints | `DELETE_CONSTRAINTS` | `constraints` |
| Extend a built-in table (add column) | `ADD_TABLE_EXTENSION` | `columnDisplayName`, `tableDisplayName` |
| Remove a built-in-table extension column | `DELETE_TABLE_EXTENSION` | `columnDisplayName`, `tableDisplayName` |
| List embedding models (vector columns) | `GET_AVAILABLE_EMBEDDING_MODELS` | — |

## Worked example: create a `post` table

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema tool-call --toolCalls '[
  {"name":"ADD_TABLES","args":{"items":[
    {"tableDisplayName":"post","tableApiName":"post","relations":[],"fields":[
      {"apiName":"title","displayName":"title","basicTypeNameOrTypeId":"TEXT","required":true,"defaultValue":""},
      {"apiName":"view_count","displayName":"view_count","basicTypeNameOrTypeId":"BIGINT","required":true,"defaultValue":0}
    ]}
  ]}}
]'
```

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `ADD_TABLES`
- `items` *(required)*: `array<{fields: array<object>, relations: array<object>, tableApiName: string, tableDisplayName: string}>`

### `ADD_FIELDS_AND_RELATIONS`
- `fields` *(required)*: `array<{apiName: string, basicTypeNameOrTypeId: string, defaultValue?: boolean | string | number, displayName: string, required: boolean}>`
- `relations` *(required)*: `array<{fieldApiNameInSourceTable: string, fieldApiNameInTargetTable: string, fieldDisplayNameInSourceTable: string, fieldDisplayNameInTargetTable: string, relationType: string, sourceTableDisplayName: string, targetTableDisplayName: string}>`
- `tableDisplayName` *(required)*: `string`

### `DELETE_FIELDS_AND_RELATIONS`
- `fieldDisplayNames` *(required)*: `array<string>` — Display names of the fields to delete
- `relationFieldDisplayNamesInSourceTable` *(required)*: `array<string>` — fieldDisplayNameInSourceTable of the relations to delete
- `tableDisplayName` *(required)*: `string` — Source table display name

### `ADD_CONSTRAINTS`
- `constraints` *(required)*: `array<{constraintName: string, constraintType: string, fieldDisplayNames: array<string>, tableDisplayName: string}>`

### `DELETE_CONSTRAINTS`
- `constraints` *(required)*: `array<{constraintName: string, tableDisplayName: string}>`

### `ADD_TABLE_EXTENSION`
- `columnDisplayName` *(required)*: `string` — Display name of the TEXT column to build the embedding index on.
- `customEmbeddingId`: `string` — Embedding model id (from GET_AVAILABLE_EMBEDDING_MODELS); the platform default when omitted.
- `tableDisplayName` *(required)*: `string`

### `DELETE_TABLE_EXTENSION`
- `columnDisplayName` *(required)*: `string`
- `tableDisplayName` *(required)*: `string`

Then ship:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema validate && "${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.

## Notes & guardrails

- **Column type** goes in `basicTypeNameOrTypeId` using the same UPPERCASE `ColumnType` names from *Column Types* above (`TEXT`, `BIGINT`, `DECIMAL`, …).
- **Destructive ops** (`DELETE_TABLES`, `DELETE_FIELDS_AND_RELATIONS`, `DELETE_CONSTRAINTS`) lose data; list what will be deleted and warn the user.
- **Type changes** aren't editable: delete + recreate the column.
- If results look stale, run `"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema reload`.

## Reading & writing deployed rows (runtime backend)

These verbs hit the **deployed** database, not the editor model, and take a single `--args` JSON blob (no per-field flags). `tableName` must be a real deployed table (`account`, your synced user tables, …); an unknown name fails server-side with `Unknown type '<name>_bool_exp'`.

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" runtime query  --args '{"tableName":"post","where":{"id":{"_eq":1}},"limit":20,"fields":["id","title"]}'
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" runtime insert --args '{"tableName":"post","objects":[{"title":"hi"}],"fields":["id"]}'
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" runtime update --args '{"tableName":"post","where":{"id":{"_eq":1}},"set":{"title":"bye"}}'
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" runtime delete --args '{"tableName":"post","where":{"id":{"_eq":1}}}'
```
- `insert` must supply every NOT-NULL column; object keys are the column **systemName**.
- `update` / `delete` require `where` unless you pass `allowUpdateAll` / `allowDeleteAll=true`.
- `affected_rows` is authoritative; `returning` can be empty when row-level read permission hides the row.
