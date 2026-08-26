# Database schema (tables, fields, relations, constraints)

## Database Domain Knowledge
Zion uses PostgreSQL. Data model changes require "Sync Backend" to take effect online.

### Capabilities & Limitations
You can: create and delete tables, fields, relations, and unique constraints; change a table's displayName / description; change a field's displayName / required / default value.
Tables and fields are addressed by their displayName in every tool on this plugin.
(Enums are created and edited via the 'type' plugin.)
You CANNOT:
- Rename a table or a field. A systemName is fixed when the entity is created — only its displayName can change afterwards. If a systemName is genuinely wrong, delete and recreate it; recreating a field discards the data already stored in it. Choose systemNames carefully up front.
- Change an existing column's TYPE, or its uniqueness. To change either, delete the column and create a new one; uniqueness cannot be added later because existing rows may already hold duplicates.
- Update an existing relation or constraint. To change one, delete it and recreate it.
- Create formula / computed fields. If the user asks for one, explain that formulas must be configured manually in the editor; do not create them.

### Every table edit rewrites role permissions
Creating or deleting a table, and adding, retyping or deleting a field, rewrites the role permissions keyed to that table. Nobody asks for it, and a new table lands in every role including Anonymous User, so the grants you end up with are not the ones you chose. Each of those results names the roles it changed. Before changing one of them, read that role with GET_ROLE_DETAIL: a permission write is rejected until the role has been read in this session, and the role you would reach for is one your own table edit just moved.

### Column Types
A new field's 'basicTypeNameOrTypeId' is a bare primitive type NAME: string, decimal, bigint, boolean, timestamptz, timetz, date, jsonb, image, video, file, geo_point.
For an enum, pass the bare enum id returned by type.list_enums — or, for an enum you
created earlier this turn, the id you supplied when creating it.

Pass the name or id on its own (for example "decimal", or "mja44si4" for an enum). Do NOT
pass a wrapped type identifier such as "s:p:decimal" or "u:e:<enumId>" — those are the
stored form, and the tool builds them for you. The separate 'required' field controls
nullability; never wrap a type in a null union.

Read results report a field's existing type using legacy uppercase names (TEXT, BIGINT).
That is the read vocabulary only — when creating a field, still pass the primitive name
(e.g. a column shown as TEXT is created with "string").

### Formula Fields
Formula (computed) fields — a column whose value is derived from other fields — are a post-refactor concept but are NOT yet authorable through these tools. If the user asks for a derived value, either compute it at write time (an action flow or mutation binding) or tell them to add the formula field in the editor; never fake one with a plain column.

### Naming
systemName: English snake_case. Tables are nouns or noun phrases, singular not plural ("order", not "orders"), concise (e.g. "user_profile"); fields are snake_case (e.g. "first_name", "is_active"). No name may contain a space — not tables, fields, relations, or constraints, and not even the displayName. A systemName is permanent once created; only the displayName can be changed later. displayName: user-visible; use the natural Chinese name as the displayName and derive the English snake_case systemName from it (e.g. displayName "用户名" → systemName "user_name").

### System Built-ins & Product Context
Every table has non-deletable built-in fields: id (s:p:bigint), created_at (s:p:timestamptz), updated_at (s:p:timestamptz). Do NOT include these when creating a table. Any table, field, or relation where 'editable' is false is system built-in and cannot be modified or deleted. System tables and timezone configurations for Zion:
* UTC Offset: +08:00
* Protected Account Table: '帐户(账户)' (can add/delete user-defined fields, but cannot delete table itself)
* Protected Payment Tables: '支付', '周期性支付', '退款' (cannot be modified or deleted)
* Protected AI/Session Tables: '会话', '消息', '工具使用记录', '消息内容' (cannot be modified or deleted) The system built-in AI tables are strictly for system AI functions. For user chat systems, always create custom user-defined tables (e.g. 'user_chat', 'chat_message').

### Required Fields & Default Values
When creating a new field with 'required = true', a default value must be set simultaneously, except for types that do not support default values. Default value formatting:
- s:p:bigint, s:p:decimal, and s:p:boolean: Use literal values (e.g., 10, true).
- s:p:timestamptz, s:p:date, and s:p:timetz: Strictly use ISO 8601 strings (e.g., '2025-12-09T16:02:03.000Z', '2025-12-09', '16:02:03+08:00').
- u:e:<enumId>: Use the option's id, i.e. its FULL_CAPS_SNAKE_CASE value (e.g., 'PENDING').
- s:p:string: Use plain strings.
- s:p:jsonb: Use stringified JSON objects.
- Unsupported: s:p:image, s:p:video, s:p:file, and s:p:geo_point do not support default values.

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

### Enums (managed by the 'type' plugin)
Enum types are created and edited via the 'type' plugin, not here. To use an enum as a column type, use "u:e:<enumId>" with the actual id from type.list_enums. Create the enum first via the 'type' plugin (same turn, before the column) if it does not yet exist — you supply the id there, so you can reference it immediately.

### Geographic Location
Use s:p:geo_point for coordinates. Never split into separate latitude/longitude fields. A s:p:geo_point field auto-generates a companion s:p:decimal hack field named "fz_distance_from_<systemName>", where <systemName> is the geo_point's systemName (may differ from its displayName). At request time it returns the distance from the stored geo_point to the user-supplied location in the request. Treat it as a distance-calculation hack — future migration: this will be replaced by formula fields.

### Constraints
Only unique constraints supported. Constraint name: lowercase English snake_case, no uppercase. Reference fields by their displayName — for a relation's foreign key use the generated FK field ("user_id"). Uniqueness is fixed when a field is created and cannot be turned on afterwards (existing rows could already hold duplicates), so to make an existing field unique, add a constraint over it. Use a constraint to span multiple fields (composite unique): list the fields' displayNames. Unique constraints are also the only DB-enforced invariant writers can lean on for atomic insert-if-absent / toggle semantics (insert with on_conflict): whenever the design has "at most one row per X" semantics (a join/like/save table, an idempotency key), create the unique constraint up front — read-check-then-insert cannot be made race-safe without it.

A relation's generated FK carries two names, and which one a call wants depends on the call. The schema tools here take displayNames, so a relation named 转译行 gives a FK whose displayName is `转译行_id` — that is what ADD_CONSTRAINTS matches on. The runtime API takes system names, so the same column is `translation_row_id` in a `runtime.query` filter or a `runtime.insert` object. Neither side accepts the other's name, and passing the wrong one reports the field as missing rather than as misnamed. Read both off the field record instead of transliterating one into the other.

## How to drive it (CLI only)

All commands are `npx -y zion-mcp@2.7.0 <verb>`. A long-lived daemon holds the in-memory CRDT schema session
between calls. **Edits do NOT go live until `project sync-backend`.**

```bash
npx -y zion-mcp@2.7.0 whoami                                    # check auth; if needed: npx -y zion-mcp@2.7.0 login
# create a NEW project (auto-pins it; its pre/post type-system state follows the account rollout):
npx -y zion-mcp@2.7.0 project create --projectName "My App"
# …or pin an EXISTING one (find its exId with npx -y zion-mcp@2.7.0 projects search):
npx -y zion-mcp@2.7.0 project set-current --projectExId <exId>
npx -y zion-mcp@2.7.0 schema load                               # warm the schema session
```

Operations run through one verb:

```bash
npx -y zion-mcp@2.7.0 schema tool-call --toolCalls '[{"name":"<TOOL_NAME>","args":{ ... }}]'
```
Each call is applied immediately — any resulting CRDT patch is uploaded. Batch several calls in one array; use `schema undo` to revert the last change.
A batch is all-or-nothing: when any call in the array fails, the whole batch's changes are discarded even though the other calls returned success — only the failing call's error is reported, so after a batch error re-read (`GET_*`) before assuming anything persisted.

## Operation reference (`schema tool-call` names)

| Intent | `name` | Required `args` |
|---|---|---|
| List table names | `GET_ALL_TABLE_DISPLAY_NAMES` | — |
| Inspect tables | `GET_TABLES_INFO` | `tableDisplayNames` |
| List selectable field types | `GET_TABLE_FIELD_SELECTABLE_TYPES` | — |
| Create tables | `ADD_TABLES` | `items` |
| Rename tables / edit descriptions | `UPDATE_TABLES` | `items` |
| Delete tables | `DELETE_TABLES` | `tableDisplayNames` |
| Reorder tables in the editor list | `REORDER_TABLES` | `reorderedTableDisplayNames` |
| Add fields/relations | `ADD_FIELDS_AND_RELATIONS` | `tableDisplayName` |
| Rename/retype fields, rename relations | `UPDATE_FIELDS_AND_RELATIONS` | `tableDisplayName` |
| Delete fields/relations | `DELETE_FIELDS_AND_RELATIONS` | `tableDisplayName` |
| Reorder a table's fields | `REORDER_TABLE_FIELDS` | `reorderedFieldDisplayNames`, `tableDisplayName` |
| Add unique constraints | `ADD_CONSTRAINTS` | `constraints` |
| Delete unique constraints | `DELETE_CONSTRAINTS` | `constraints` |
| Reorder a table's unique constraints | `REORDER_CONSTRAINTS` | `reorderedConstraintNames`, `tableDisplayName` |
| List embedding models | `GET_AVAILABLE_EMBEDDING_MODELS` | — |
| Enable vector search on a TEXT field | `ADD_TABLE_EXTENSION` | `fieldDisplayName`, `tableDisplayName` |
| Retune vector search (e.g. change model) | `UPDATE_TABLE_EXTENSION` | `customEmbeddingId`, `fieldDisplayName`, `tableDisplayName` |
| Disable vector search | `DELETE_TABLE_EXTENSION` | `fieldDisplayName`, `tableDisplayName` |

## Worked example: create a `post` table

Read the field-type picker first, then copy its `typeIdentifier` values verbatim:

```bash
npx -y zion-mcp@2.7.0 schema tool-call --toolCalls '[{"name":"GET_TABLE_FIELD_SELECTABLE_TYPES","args":{}}]'
npx -y zion-mcp@2.7.0 schema tool-call --toolCalls '[
  {"name":"ADD_TABLES","args":{"items":[
    {"tableDisplayName":"post","tableApiName":"post","relations":[],"fields":[
      {"apiName":"title","displayName":"title","typeIdentifier":"s:p:string","required":true,"defaultValue":""},
      {"apiName":"view_count","displayName":"view_count","typeIdentifier":"s:p:bigint","required":true,"defaultValue":0}
    ]}
  ]}}
]'
```

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `ADD_TABLES`

Create tables, each with a displayName and its initial fields.
- `items` *(required)*: `array<{fields: array<object>, relations?: array<object>, tableApiName: string, tableDisplayName: string}>`

### `ADD_FIELDS_AND_RELATIONS`

Add fields and relations to one table, addressed by displayName. Each field's `typeIdentifier` comes verbatim from GET_TABLE_FIELD_SELECTABLE_TYPES — never assemble one by hand. Relations go in the `relations` argument, declared on the SOURCE table; the target side and its foreign-key field are generated automatically, so never add a manual foreign-key field instead. Send whichever of the two lists the change needs; omit the other.
- `fields`: `array<{apiName: string, computed?: boolean, defaultValue?: boolean | string | number | object, displayName: string, required: boolean, typeIdentifier?: string}>`
- `relations`: `array<{fieldApiNameInSourceTable: string, fieldApiNameInTargetTable: string, fieldDisplayNameInSourceTable: string, fieldDisplayNameInTargetTable: string, relationType: string, sourceTableDisplayName: string, targetTableDisplayName: string}>`
- `tableDisplayName` *(required)*: `string`

### `DELETE_FIELDS_AND_RELATIONS`

Delete fields and relations from one table by displayName — fields in `fieldDisplayNames`, relations in `relationFieldDisplayNamesInSourceTable`. Send whichever list the change needs and omit the other. Data stored in a deleted field is lost, and a deleted relation takes the generated foreign-key field on the target table with it.
- `fieldDisplayNames`: `array<string>` — Display names of the fields to delete
- `relationFieldDisplayNamesInSourceTable`: `array<string>` — fieldDisplayNameInSourceTable of the relations to delete
- `tableDisplayName` *(required)*: `string` — Source table display name

### `ADD_CONSTRAINTS`

Add unique constraints to a table, spanning one or more of its fields. Use this for composite uniqueness.
- `constraints` *(required)*: `array<{constraintName: string, constraintType: string, fieldDisplayNames: array<string>, tableDisplayName: string}>`

### `DELETE_CONSTRAINTS`

Remove unique constraints from a table by constraint name.
- `constraints` *(required)*: `array<{constraintName: string, tableDisplayName: string}>`

### `ADD_TABLE_EXTENSION`

Enable vector-embedding search on one TEXT field, using a model from GET_AVAILABLE_EMBEDDING_MODELS. Adds hidden embedding and token-count columns the backend maintains; use it when the app needs semantic search over that text rather than exact or prefix matching.
- `customEmbeddingId`: `string` — Embedding model id (from GET_AVAILABLE_EMBEDDING_MODELS); the platform default when omitted.
- `fieldDisplayName` *(required)*: `string` — Display name of the TEXT field to build the embedding index on.
- `tableDisplayName` *(required)*: `string`

### `UPDATE_TABLE_EXTENSION`

Change an existing vector-search extension, e.g. to a different embedding model. Re-embedding is done by the backend.
- `customEmbeddingId` *(required)*: `string` — Embedding model id (from GET_AVAILABLE_EMBEDDING_MODELS) to switch the index to; it is the only reconfigurable part of an extension.
- `fieldDisplayName` *(required)*: `string` — Display name of the field whose embedding index is reconfigured.
- `tableDisplayName` *(required)*: `string`

### `DELETE_TABLE_EXTENSION`

Remove a table's vector-search extension. The generated embedding columns and their stored vectors go with it.
- `fieldDisplayName` *(required)*: `string`
- `tableDisplayName` *(required)*: `string`

Then ship:

```bash
npx -y zion-mcp@2.7.0 schema validate && npx -y zion-mcp@2.7.0 project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.

## Notes & guardrails

- **Column type** goes in `typeIdentifier`, copied verbatim from `GET_TABLE_FIELD_SELECTABLE_TYPES` (on this project they look like `s:p:string`) — never assemble or guess one.
- The picker lists **primitives and enums only**, and returns the *concrete* type: `required` decides nullability, so never pass a `|null` union. A type it did not offer is rejected.
- **Destructive ops** (`DELETE_TABLES`, `DELETE_FIELDS_AND_RELATIONS`, `DELETE_CONSTRAINTS`) lose data; list what will be deleted and warn the user.
- **Type changes** aren't editable: delete + recreate the column.
- If results look stale, run `npx -y zion-mcp@2.7.0 schema reload`.
- **Enums / custom types** are out of scope here — see `schema-type.md`.

## Reading & writing deployed rows (runtime backend)

These verbs hit the **deployed** database, not the editor model, and take a single `--args` JSON blob (no per-field flags). `tableName` must be a real deployed table (`account`, your synced user tables, …); an unknown name fails server-side with `Unknown type '<name>_bool_exp'`.

```bash
npx -y zion-mcp@2.7.0 runtime query  --args '{"tableName":"post","where":{"id":{"_eq":1}},"limit":20,"fields":["id","title"]}'
npx -y zion-mcp@2.7.0 runtime insert --args '{"tableName":"post","objects":[{"title":"hi"}],"fields":["id"]}'
npx -y zion-mcp@2.7.0 runtime update --args '{"tableName":"post","where":{"id":{"_eq":1}},"set":{"title":"bye"}}'
npx -y zion-mcp@2.7.0 runtime delete --args '{"tableName":"post","where":{"id":{"_eq":1}}}'
```
- `insert` must supply every NOT-NULL column; object keys are the column **systemName**.
- `update` / `delete` require `where` unless you pass `allowUpdateAll` / `allowDeleteAll=true`.
- `affected_rows` is authoritative; `returning` can be empty when row-level read permission hides the row.
