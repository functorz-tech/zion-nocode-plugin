# Project overview & relation graph

## Project Overview Domain Knowledge
Overview offers two complementary bird's-eye queries, both deliberately compact (names + ids only) so a single call orients you without flooding context.

- GET_PROJECT_OVERVIEW — the complete inventory of every top-level entity in the app
(tables, pages, modals, action flows, triggers, webhooks, APIs, agents, roles, types, global
variables, colour themes) with the ids/names the domain tools take. Call this **first** on an
unfamiliar project, and treat its answer as the census: the per-domain `list_*` tools return
subsets of it, so calling one after this buys nothing.
- GET_ENTITY_RELATION_GRAPH — the relation graph around one entity: who reads / writes / calls / governs it, expanded a bounded number of hops. Use it before changing or deleting an entity to see what depends on it.

Both are read-only; neither mutates the schema.

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
| Whole-project inventory | `GET_PROJECT_OVERVIEW` | — |
| Relations around one entity | `GET_ENTITY_RELATION_GRAPH` | `entityId`, `entityType` |

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `GET_PROJECT_OVERVIEW`

The whole project in one call: every table, page, modal, action flow, database and
scheduled trigger, webhook, API, AI agent, role, type, global variable and
colour-theme entry, each with the identifier its own domain tool takes. This is the
complete census, not a teaser for the per-domain list tools. After calling it do NOT
call GET_ALL_TABLE_DISPLAY_NAMES, GET_ALL_ROOTS_INFO, GET_ALL_ROLES_INFO,
GET_ALL_ACTION_FLOWS_INFO, GET_ALL_DB_TRIGGERS_INFO,
GET_ALL_SCHEDULED_TRIGGERS_INFO, GET_ALL_ZAI_CONFIGS_INFO,
GET_ALL_CALLBACKS_INFO, GET_ALL_SECRET_CONFIGS,
GET_COLOR_THEMES, or any other per-domain `list_*` tool: each returns
a subset of what you already hold, and re-listing cannot refresh it — this inventory
is rebuilt from live schema on every call. Call it again only after you have created
or deleted an entity. Narrow it with `entityTypes` when you already know which
domain you need.
- `entityTypes`: `array<enum(PAGE|TABLE|ACTION_FLOW|API|ZAI|TRIGGER|ROLE|TYPE|GLOBAL_VARIABLE|COLOR_THEME|… 13 total)>` — Optional entity-type filter. When set, only the matching inventory sections are returned: API — apiWorkspaces or thirdPartyApis; TRIGGER — dbTriggers + scheduledTriggers; TYPE — enums + objectTypes; other values map to their single section. Omit (or pass an empty list) to include every section.

### `GET_ENTITY_RELATION_GRAPH`

The relation graph around one entity — who reads / writes / calls / governs it, a bounded number of hops out (names + ids only). Use it to gauge blast radius before editing or deleting an entity.
- `depth`: `integer` — How many reference hops to expand from the entity (1-3, default 1). Each hop adds every entity that directly uses the previous layer.
- `entityId` *(required)*: `string` — The entity's identifier: TABLE — table displayName; API — apiId (workspace API) or tpaConfigId (legacy third-party API); ACTION_FLOW — actionFlowId; ZAI — configId; ROLE — role uuid or name; TRIGGER — triggerId; COMPONENT — componentId; GLOBAL_VARIABLE — variable key.
- `entityType` *(required)*: `enum(TABLE|API|ACTION_FLOW|ZAI|ROLE|TRIGGER|COMPONENT|GLOBAL_VARIABLE)` — The kind of entity at the center of the graph.
- `relationTypes`: `array<enum(CALL|CREATE|READ|UPDATE|DELETE|REFERENCE|TRIGGER|ASSIGN|GOVERN|TYPE|… 13 total)>` — Optional relation-type filter. When set, only edges carrying at least one listed relation are kept, and nodes no longer connected to the entity through the kept edges are dropped. Omit (or pass an empty list) to include all relation types.

Start here on an unfamiliar project. One `GET_PROJECT_OVERVIEW` call inventories every table, action flow, trigger, API, agent, role and type — cheaper and more complete than probing each domain with its own `GET_ALL_*` call, and it tells you which capability files are worth reading. Narrow it with `entityTypes` once you know what you are after.

`GET_ENTITY_RELATION_GRAPH` answers "what breaks if I change this": pass an `entityId` from the overview and it returns what reads, writes, calls, or references that entity. Run it before deleting or restructuring a table, flow, or API — the editor will not warn you about a dependent it is about to orphan.
