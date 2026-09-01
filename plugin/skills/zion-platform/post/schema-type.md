# Enums & custom types

## Type System Domain Knowledge
The type system owns named, reusable types: enums and custom objects. Data model changes require "Sync Backend" to take effect online.

### Enums
An enum and each of its options carry a permanent id. The platform assigns it — never send one — and every create returns the minted ids in its result. An id is never editable afterwards, which is what keeps every column and binding that references it valid. Reference an enum as a column type in the 'database' plugin by the exact canonical identifier `u:e:<enumId>` — include the `u:e:` prefix (for example, `u:e:OrderStatus`).
- An option's id is the value a column's default and any binding stores, so it is the id you carry to the other plugins, not the label.
- An option's `name` is its user-visible label — NOT an identifier, and the only part of an option you can edit. Use a natural Chinese phrase (e.g. enum displayName "订单状态", option name "待处理"). The ids stay English as above. Non-blank, ≤200 chars.
- An enum also has a displayName (same rule) and an optional description; options have no description.

Operations:
- `GET_ALL_ENUM_DEFINITIONS`: list enums with their options. Always call it before editing — edits are rejected until the target enum has been read.
- `ADD_ENUM_DEFINITIONS`: create enums with their initial options inline.
- `UPDATE_ENUM_DEFINITIONS`: change an enum's own displayName / description. Omitted fields are unchanged. Its `options` argument REPLACES the whole option list, so use the per-option tools below for single-option edits.
- `ADD_ENUM_OPTIONS` / `UPDATE_ENUM_OPTIONS` / `DELETE_ENUM_OPTIONS`: append options, rename their labels, or remove them.

Adding an option is always safe. Deleting an enum or an option is destructive: anything referencing it breaks, and nothing repoints it for you — check usages, delete or repoint the columns first, and tell the user. There is no way to change an id, so a "rename" of an id means delete + recreate with the same breakage; if the user only wants different wording, edit the label (`name` / displayName) instead and the id can stay.

If a column should use a new enum, create the enum here first, in the same turn and before the column: take the minted id out of the create result and the column can reference `u:e:<thatId>` straight away.

### Custom Objects
Custom object types are named, reusable structured types (a set of typed fields), referenced elsewhere by the identifier `u:o:<typeId>` (action-flow inputs/outputs, other types' fields, and other type slots).

Operations:
- `GET_ALL_OBJECT_DEFINITIONS`: list object types with their fields. Always call it before editing — edits are rejected until the target type has been read.
- `ADD_OBJECT_TYPE_DEFINITIONS`: create object types, each with a displayName and its initial fields. Ids are assigned by the platform and returned in the result — do not send one.
- `UPDATE_OBJECT_TYPE_DEFINITIONS`: update a type's own displayName / description / private flag only (omitted fields are unchanged).
- `ADD_TYPE_DEFINITION_FIELDS` / `UPDATE_TYPE_DEFINITION_FIELDS` / `DELETE_TYPE_DEFINITION_FIELDS`: add fields, rename or retype existing ones, or remove them.
- `REORDER_TYPE_DEFINITION_FIELDS`: set the order the type's fields are rendered and emitted in — pass every field name, in the order wanted.
- `COPY_PRIVATE_OBJECT_TYPE_AS_PUBLIC` / `COPY_PUBLIC_OBJECT_TYPE_AS_PRIVATE`: copy a type between the shared list and a single feature (see below).

Field types are TypeIdentifier strings: `s:p:<primitive>` (string, bigint, decimal, boolean, timestamptz, timetz, date, jsonb, image, video, file, geo_point), or `u:o:<typeId>` / `u:e:<enumId>` to nest an object or enum type; a field's arrayLevel makes it a list (1) or list of lists (2).

Rename and retype through `UPDATE_TYPE_DEFINITION_FIELDS` rather than delete + re-add: a rename carries the field's references along with it, where a re-add breaks every one of them. Retyping away from an image / video / file or JSON type still drops the references, because nothing that pointed at the old shape can read the new one — check usages and warn the user first. Deleting a field, or a type that is still referenced, breaks those references the same way.

### Shared vs feature-owned object types
A public type is an entry in the project's shared object type list. A private one belongs to a single feature — a webhook request body, an API JSON body, an action flow's output — and is not listed. Do NOT flip `private` with `UPDATE_OBJECT_TYPE_DEFINITIONS` to share or unshare a shape: that MOVES the type the feature is already using, so the feature's contract silently becomes an editor-listed type everything else can bind to (or stops being one). Copy instead:
- `COPY_PRIVATE_OBJECT_TYPE_AS_PUBLIC`: put a copy of a private type into the shared list, leaving the feature on the original. Read `GET_OBJECT_TYPE_DEFINITION_GROUPS` first — the copy joins that configuration.
- `COPY_PUBLIC_OBJECT_TYPE_AS_PRIVATE`: take a feature-owned copy of a shared type, so that one feature can diverge without changing the shape for everyone else.

Either way the copy is a NEW type with a new id, and nothing points at it yet: the feature keeps using the old type until you set that slot's type to the `typeIdentifier` the result echoes.

> Available only on **post-type-system-refactor** projects; the daemon hard-gates these tools on pre-refactor projects. Check `npx -y zion-mcp@2.7.4 schema status` → `typeSystem`.

## How to drive it (CLI only)

All commands are `npx -y zion-mcp@2.7.4 <verb>`. A long-lived daemon holds the in-memory CRDT schema session
between calls. **Edits do NOT go live until `project sync-backend`.**

```bash
npx -y zion-mcp@2.7.4 whoami                                    # check auth; if needed: npx -y zion-mcp@2.7.4 login
# create a NEW project (auto-pins it; its pre/post type-system state follows the account rollout):
npx -y zion-mcp@2.7.4 project create --projectName "My App"
# …or pin an EXISTING one (find its exId with npx -y zion-mcp@2.7.4 projects search):
npx -y zion-mcp@2.7.4 project set-current --projectExId <exId>
npx -y zion-mcp@2.7.4 schema load                               # warm the schema session
```

Operations run through one verb:

```bash
npx -y zion-mcp@2.7.4 schema tool-call --toolCalls '[{"name":"<TOOL_NAME>","args":{ ... }}]'
```
Each call is applied immediately — any resulting CRDT patch is uploaded. Batch several calls in one array; use `schema undo` to revert the last change.
A batch is all-or-nothing: when any call in the array fails, the whole batch's changes are discarded even though the other calls returned success — only the failing call's error is reported, so after a batch error re-read (`GET_*`) before assuming anything persisted.

## Operation reference (`schema tool-call` names)

| Intent | `name` | Required `args` |
|---|---|---|
| List enums | `GET_ALL_ENUM_DEFINITIONS` | — |
| Read enum group config (before adding an enum) | `GET_ENUM_DEFINITION_GROUPS` | — |
| Add enums | `ADD_ENUM_DEFINITIONS` | `enums` |
| Update enums | `UPDATE_ENUM_DEFINITIONS` | `enums` |
| Delete enums | `DELETE_ENUM_DEFINITIONS` | `enumIds` |
| Create enum groups | `ADD_ENUM_DEFINITION_GROUPS` | `groups` |
| Rename enum groups | `UPDATE_ENUM_DEFINITION_GROUPS` | `groups` |
| Delete enum groups | `DELETE_ENUM_DEFINITION_GROUPS` | `groupIds` |
| File enums into a group | `MOVE_ENUM_DEFINITIONS_TO_GROUP` | `items` |
| List object types | `GET_ALL_OBJECT_DEFINITIONS` | — |
| Rename or retype an object type's fields | `UPDATE_TYPE_DEFINITION_FIELDS` | `fields`, `typeId` |
| Reorder an object type's fields | `REORDER_TYPE_DEFINITION_FIELDS` | `orderedFieldNames`, `typeId` |
| Publish a private object type as reusable | `COPY_PRIVATE_OBJECT_TYPE_AS_PUBLIC` | `displayName`, `typeId` |
| Take a private copy of a public object type | `COPY_PUBLIC_OBJECT_TYPE_AS_PRIVATE` | `typeId` |
| Read object-type group config (before adding a type) | `GET_OBJECT_TYPE_DEFINITION_GROUPS` | — |
| Add object types | `ADD_OBJECT_TYPE_DEFINITIONS` | `types` |
| Update object types | `UPDATE_OBJECT_TYPE_DEFINITIONS` | `types` |
| Delete object types | `DELETE_OBJECT_TYPE_DEFINITIONS` | `typeIds` |
| Create object-type groups | `ADD_OBJECT_TYPE_DEFINITION_GROUPS` | `groups` |
| Rename object-type groups | `UPDATE_OBJECT_TYPE_DEFINITION_GROUPS` | `groups` |
| Delete object-type groups | `DELETE_OBJECT_TYPE_DEFINITION_GROUPS` | `groupIds` |
| File object types into a group | `MOVE_OBJECT_TYPE_DEFINITIONS_TO_GROUP` | `items` |
| Rename/retype an object type's fields in place | `UPDATE_TYPE_DEFINITION_FIELDS` | `fields`, `typeId` |
| Reorder an object type's fields | `REORDER_TYPE_DEFINITION_FIELDS` | `orderedFieldNames`, `typeId` |
| Publish a private object type as a reusable one | `COPY_PRIVATE_OBJECT_TYPE_AS_PUBLIC` | `displayName`, `typeId` |
| Take a private copy of a public object type | `COPY_PUBLIC_OBJECT_TYPE_AS_PRIVATE` | `typeId` |
| Add fields to an object type | `ADD_TYPE_DEFINITION_FIELDS` | `fields`, `typeId` |
| Delete fields from an object type | `DELETE_TYPE_DEFINITION_FIELDS` | `fieldNames`, `typeId` |

## Worked example: an OrderStatus enum

```bash
npx -y zion-mcp@2.7.4 schema tool-call --toolCalls '[
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

Create enum types, each with a displayName and its initial options. Ids are assigned by the platform and returned in the result — do not send one. Read list_enum_groups first — creating an enum also writes the enum group configuration.
- `enums` *(required)*: `array<{description?: string, displayName: string, groupId?: string, options: array<{name: string}>}>`

### `UPDATE_ENUM_DEFINITIONS`

Update enums' own displayName / description; omitted fields are unchanged. The `options` argument replaces the entire option list — prefer ADD_ENUM_OPTIONS / UPDATE_ENUM_OPTIONS / DELETE_ENUM_OPTIONS for single-option edits.
- `enums` *(required)*: `map<string, {description?: string, displayName?: string, options?: array<{id: string, name: string}>}>` — Map of enum ID to the fields to update

### `ADD_OBJECT_TYPE_DEFINITIONS`

Create custom object types, each with a displayName and its initial fields. Ids are assigned by the platform and returned in the result — do not send one. Read list_object_type_groups first — creating an object type also writes the object-type group configuration.
- `types` *(required)*: `array<{description?: string, displayName?: string, groupId?: string, private?: boolean, properties: array<{name: string, required?: boolean, type: string}>}>`

### `UPDATE_OBJECT_TYPE_DEFINITIONS`

Update object types' own metadata (displayName / description / private); omitted fields are unchanged. Fields are edited with ADD_TYPE_DEFINITION_FIELDS / UPDATE_TYPE_DEFINITION_FIELDS / DELETE_TYPE_DEFINITION_FIELDS instead. `private` MOVES the type between the shared list and its owning feature — to share or unshare a shape without moving it, use COPY_PRIVATE_OBJECT_TYPE_AS_PUBLIC / COPY_PUBLIC_OBJECT_TYPE_AS_PRIVATE.
- `types` *(required)*: `map<string, {description?: string, displayName?: string, private?: boolean}>` — Map of object type id to the fields to update (null fields are unchanged).

### `ADD_TYPE_DEFINITION_FIELDS`

Add fields to an existing object type.
- `fields` *(required)*: `array<{name: string, required?: boolean, type: string}>`
- `typeId` *(required)*: `string`

Then ship:

```bash
npx -y zion-mcp@2.7.4 schema validate && npx -y zion-mcp@2.7.4 project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
