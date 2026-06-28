# Database schema (tables, fields, relations, constraints)

## Database Domain Knowledge
Zion uses PostgreSQL. Data model changes require "Sync Backend" to take effect online.

### Capabilities & Limitations
You can: create and delete tables, fields, relations, and unique constraints; rename a
table; rename a column or change its displayName / required / unique / default value.

You CANNOT:
- Change an existing column's TYPE. To change a type, delete the column and create a new
  one with the desired type.
- Update an existing relation or constraint. To change one, delete it and recreate it.
- Create formula / computed fields. If the user asks for one, explain that formulas must
  be configured manually in the editor; do not create them.

### Column Types
A column's 'type' is the UPPERCASE type name, exactly as written (it is case-sensitive).
Basic: BIGINT, DECIMAL, TEXT (for strings/VARCHAR/CHAR), BOOLEAN, TIMESTAMPTZ, DATE,
TIMETZ, IMAGE, VIDEO, FILE, JSONB, GEO_POINT.

Put one of those values directly in a column's 'type' field (e.g. "DECIMAL"). Do NOT put a
lowercase name or a type-identifier ("s:p:...") in 'type'.

Type Identifier encoding (a SEPARATE form that appears only in type-identifier
inputs/outputs — never in a column's 'type' field):
- Basic types prefix with "s:p:" (e.g. "s:p:bigint", "s:p:string").
- Optional fields prepend "null|" (e.g. "null|s:p:bigint").

### Naming
systemName: English snake_case. Tables are nouns or noun phrases, singular not plural
("order", not "orders"), concise (e.g. "user_profile"); fields are snake_case
(e.g. "first_name", "is_active").
displayName: user-visible; prefer it IDENTICAL to the systemName (e.g. systemName "first_name" → displayName "first_name").

### System Built-ins & Product Context
Every table has non-deletable built-in fields: id (BIGINT), created_at (TIMESTAMPTZ),
updated_at (TIMESTAMPTZ). Do NOT include these when creating a table.
Any table, field, or relation where 'editable' is false is system built-in and cannot be modified or deleted.
System tables and timezone configurations for Zion:
* UTC Offset: +00:00
* Protected Account Table: 'account' (can add/delete user-defined fields, but cannot delete table itself)
* Protected Payment Tables: 'payment_record', 'recurring_payment', 'refund' (cannot be modified or deleted)
* Protected AI/Session Tables: 'conversation', 'message', 'tool_usage_record', 'message_content' (cannot be modified or deleted)
The system built-in AI tables are strictly for system AI functions. For user chat systems, always create custom user-defined tables (e.g. 'user_chat', 'chat_message').

### Required Fields & Default Values
When creating a new field with 'required = true', a default value must be set simultaneously, except for types that do not support default values.
Default value formatting:
- Numbers/Booleans: Use literal values (e.g., 10, true).
- Dates/Times: Strictly use ISO 8601 strings (e.g., TIMESTAMPTZ: '2025-12-09T16:02:03.000Z', DATE: '2025-12-09', TIMETZ: '16:02:03+00:00').

- String: Use plain strings.
- Jsonb: Use stringified JSON objects.
- Unsupported: IMAGE, VIDEO, FILE, and GEO_POINT do NOT support setting default values.

### Relations
Types: one_to_one, one_to_many. Defined on the source table.
To make one table reference another, create a RELATION — never add a manual foreign-key
column (e.g. a "*_id" field) or a column whose type is another table. The FK column and
the virtual reference fields are generated automatically. Add the relation on the SOURCE
table only; it is reflected on the target automatically.
A relation is configured by five fields: source table, target table, source-side field
display name, target-side field display name, and relation type.
Relation record fields: relationType, sourceTableDisplayName, targetTableDisplayName,
fieldDisplayNameInSourceTable, fieldDisplayNameInTargetTable, editable.
Creating a relation auto-generates:
- A non-editable FK field in the target table named fieldDisplayNameInTargetTable + "_id"
  (e.g. "user_id", "活动_id"). Stores the source row's id. Deleted when the relation is deleted.
- Virtual reference fields in both tables (NOT real columns), one per side:
  fieldDisplayNameInSourceTable lives on the source; fieldDisplayNameInTargetTable lives on the target.
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
Use GEO_POINT for coordinates. Never split into separate latitude/longitude fields.
A GEO_POINT field auto-generates a companion DECIMAL hack field named
"fz_distance_from_<systemName>", where <systemName> is the geo_point's systemName
(may differ from its displayName). At request time it returns the distance from the
stored geo_point to the user-supplied location in the request. Treat it as a
distance-calculation hack — future migration: this will be replaced by formula fields.

### Constraints
Only unique constraints supported. Names: lowercase English snake_case, no uppercase.
Combination constraints: list multiple field display names.

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
| List table names | `GET_ALL_TABLE_DISPLAY_NAMES` | — |
| Inspect tables | `GET_TABLE_INFOS` | `tableDisplayNames` |
| Create tables | `ADD_TABLES` | `items` |
| Delete tables | `DELETE_TABLES` | `tableDisplayNames` |
| Add fields/relations | `ADD_FIELDS_AND_RELATIONS` | `fields`, `relations`, `tableDisplayName` |
| Add unique constraints | `ADD_CONSTRAINTS` | `constraints` |

## Worked example: create a `post` table

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema tool-call --toolCalls '[
  {"name":"ADD_TABLES","args":{"items":[
    {"tableDisplayName":"post","tableSystemName":"post","relations":[],"fields":[
      {"systemName":"title","displayName":"title","basicTypeNameOrTypeId":"TEXT","required":true,"defaultValue":""},
      {"systemName":"view_count","displayName":"view_count","basicTypeNameOrTypeId":"BIGINT","required":true,"defaultValue":0}
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

### `ADD_CONSTRAINTS`
- `constraints` *(required)*: `array<{constraintName: string, constraintType: string, fieldDisplayNames: array<string>, tableDisplayName: string}>`

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

## Reading & writing deployed rows (supportservice)

These verbs hit the **deployed** database, not the editor model, and take a single `--args` JSON blob (no per-field flags). `tableName` must be a real deployed table (`account`, your synced user tables, …); an unknown name fails server-side with `Unknown type '<name>_bool_exp'`.

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" support query  --args '{"tableName":"post","where":{"id":{"_eq":1}},"limit":20,"fields":["id","title"]}'
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" support insert --args '{"tableName":"post","objects":[{"title":"hi"}],"fields":["id"]}'
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" support update --args '{"tableName":"post","where":{"id":{"_eq":1}},"set":{"title":"bye"}}'
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" support delete --args '{"tableName":"post","where":{"id":{"_eq":1}}}'
```
- `insert` must supply every NOT-NULL column; object keys are the column **systemName**.
- `update` / `delete` require `where` unless you pass `allowUpdateAll` / `allowDeleteAll=true`.
- `affected_rows` is authoritative; `returning` can be empty when row-level read permission hides the row.
