# Enums & custom types

## Type System Domain Knowledge
The type system owns named, reusable types: enums and custom objects. Data model changes require "Sync Backend" to take effect online.

### Enums
An enum has a single id and no separate "name". When you create an enum, the PascalCase name you give IS its id. When referencing an enum as a column type in the 'database' plugin, pass the exact canonical identifier `u:e:<enumId>` and include the `u:e:` prefix (for example, `u:e:OrderStatus`). For an EXISTING enum, replace <enumId> with the exact id from type.list_enums (which may be a short id like "mja44si4").
- Enum id: PascalCase, ≤63 chars, English regardless of product (e.g. "OrderStatus"). Must be unique and must not collide with a table name or another enum.
- Option value: FULL_CAPS_SNAKE_CASE, ≤63 chars, English regardless of product (e.g. "PENDING"); an option's value is also its id.
- displayName (for the enum and each option): user-visible; use a natural Chinese phrase (e.g. enum displayName "订单状态", option displayName "待处理"). The enum name and option values stay English as above. Non-blank, ≤200 chars.
- An enum may have an optional description (options may not).

Operations:
- create_enum: create an enum with its initial options inline ({name, optional displayName}).
- create_enum_option / modify_enum_option / delete_enum_option: add / rename / remove ONE option of an existing enum. modify_enum does NOT take options.
- modify_enum: change only the enum's own id(name) / displayName / description.
- delete_enum: delete an enum. Always call type.list_enums first to see current ids and options before modifying.

Editing in place is destructive: renaming or deleting an option or enum changes/removes its id, breaking any column that references it — such a change is rejected and rolled back on apply. Prefer ADDING options. To remove or rename an in-use enum or option, first delete or repoint the columns that use it, and tell the user the change is destructive.

If a column should use a new enum, create the enum here first (in the same turn, before the column), so the column can resolve its type by the enum's id (its name).

### Custom Objects
Custom object types are named, reusable structured types (a set of typed fields), referenced elsewhere by the identifier `u:o:<typeId>` (action-flow inputs/outputs, other types' fields, and other type slots).

Operations:
- `list_objects`: list object types with their fields. Always call it before editing — edits are rejected until the target type has been read.
- `create_object_types`: create object types. Each item takes a caller-supplied unique short id (lowercase alphanumeric, e.g. "mja44si4"), a displayName, and its initial fields.
- `update_object_types`: update a type's own displayName / description / private flag only (omitted fields are unchanged).
- `add_object_type_fields` / `delete_object_type_fields`: add or remove fields of an existing type.

Field types are TypeIdentifier strings: `s:p:<primitive>` (string, bigint, decimal, boolean, timestamptz, timetz, date, jsonb, image, video, file, geo_point), or `u:o:<typeId>` / `u:e:<enumId>` to nest an object or enum type; a field's arrayLevel makes it a list (1) or list of lists (2).

There is no field-level update: renaming or retyping a field means delete + re-add, which breaks anything bound to the old field — check usages and warn the user first. Deleting a type that is still referenced breaks those references the same way.

> Available only on **post-type-system-refactor** projects; the daemon hard-gates these tools on pre-refactor projects. Check `npx -y zion-mcp@2.3.0 schema status` → `typeSystem`.

## How to drive it (CLI only)

All commands are `npx -y zion-mcp@2.3.0 <verb>`. A long-lived daemon holds the in-memory CRDT schema session
between calls. **Edits do NOT go live until `project sync-backend`.**

```bash
npx -y zion-mcp@2.3.0 whoami                                    # check auth; if needed: npx -y zion-mcp@2.3.0 login
# create a NEW project (auto-pins it; its pre/post type-system state follows the account rollout):
npx -y zion-mcp@2.3.0 project create --projectName "My App"
# …or pin an EXISTING one (find its exId with npx -y zion-mcp@2.3.0 projects search):
npx -y zion-mcp@2.3.0 project set-current --projectExId <exId>
npx -y zion-mcp@2.3.0 schema load                               # warm the schema session
```

Operations run through one verb:

```bash
npx -y zion-mcp@2.3.0 schema tool-call --toolCalls '[{"name":"<TOOL_NAME>","args":{ ... }}]'
```
Each call is applied immediately — any resulting CRDT patch is uploaded. Batch several calls in one array; use `schema undo` to revert the last change.
A batch is all-or-nothing: when any call in the array fails, the whole batch's changes are discarded even though the other calls returned success — only the failing call's error is reported, so after a batch error re-read (`GET_*`) before assuming anything persisted.

## Operation reference (`schema tool-call` names)

| Intent | `name` | Required `args` |
|---|---|---|
| List enums | `GET_ALL_ENUM_DEFINITIONS` | — |
| Add enums | `ADD_ENUM_DEFINITIONS` | `enums` |
| Update enums | `UPDATE_ENUM_DEFINITIONS` | `enums` |
| Delete enums | `DELETE_ENUM_DEFINITIONS` | `enumIds` |
| List object types | `GET_ALL_OBJECT_DEFINITIONS` | — |
| Add object types | `ADD_OBJECT_TYPE_DEFINITIONS` | `types` |
| Update object types | `UPDATE_OBJECT_TYPE_DEFINITIONS` | `types` |
| Delete object types | `DELETE_OBJECT_TYPE_DEFINITIONS` | `typeIds` |
| Add fields to an object type | `ADD_TYPE_DEFINITION_FIELDS` | `fields`, `typeId` |
| Delete fields from an object type | `DELETE_TYPE_DEFINITION_FIELDS` | `fieldNames`, `typeId` |

## Worked example: an OrderStatus enum

```bash
npx -y zion-mcp@2.3.0 schema tool-call --toolCalls '[
  {"name":"ADD_ENUM_DEFINITIONS","args":{"enums":[
    {"name":"OrderStatus","displayName":"OrderStatus","options":[
      {"value":"PENDING","displayName":"PENDING"},
      {"value":"PAID","displayName":"PAID"}
    ]}
  ]}}
]'
```
Create the enum **before** any column that references it (by its PascalCase id). Editing in-use options is destructive — prefer ADDING.

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `ADD_ENUM_DEFINITIONS`
- `enums` *(required)*: `array<{displayName: string, groupId?: string, id: string, options: array<object>}>`

### `UPDATE_ENUM_DEFINITIONS`
- `enums` *(required)*: `map<string, {displayName?: string, options?: array<object>}>` — Map of enum ID to the fields to update

### `ADD_OBJECT_TYPE_DEFINITIONS`
- `types` *(required)*: `array<{description?: string, displayName?: string, groupId?: string, id: string, private?: boolean, properties: array<object>}>`

### `UPDATE_OBJECT_TYPE_DEFINITIONS`
- `types` *(required)*: `map<string, {description?: string, displayName?: string, private?: boolean}>` — Map of object type id to the fields to update (null fields are unchanged).

### `ADD_TYPE_DEFINITION_FIELDS`
- `fields` *(required)*: `array<{name: string, required?: boolean, type: string}>`
- `typeId` *(required)*: `string`

Then ship:

```bash
npx -y zion-mcp@2.3.0 schema validate && npx -y zion-mcp@2.3.0 project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
