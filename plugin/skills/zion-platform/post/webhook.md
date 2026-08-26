# Webhooks (incoming HTTP triggers)

## Webhook Domain Knowledge
A webhook is a public HTTP endpoint that fires an action flow every time it is called; the
URL ends with the webhook's generated uniqueId. Only custom webhooks are authorable here —
payment-provider callbacks (Alipay / WeChat / Stripe / …) come back with `managed: true` and
cannot be edited or deleted. Changes take effect online only after "Sync Backend".

Always call GET_ALL_CALLBACKS_INFO (and GET_CALLBACK_DETAIL for one) before editing.

### Creating a webhook
1. Pick the action flow the webhook fires: an id from GET_ALL_ACTION_FLOWS_INFO. The bound
flow cannot be changed later — to re-point a webhook, delete it and create a new one.
2. ADD_CALLBACK_TRIGGERS with that actionFlowId (optional display name); the webhook is enabled
immediately.
3. Describe the incoming request body so the payload can feed the flow's inputs:
- SET_CALLBACK_REQUEST_TYPE pointing at an object type id (from GET_ALL_OBJECT_DEFINITIONS, or one
you just made with ADD_OBJECT_TYPE_DEFINITIONS). Its fields become binding sources for
the flow's input args.
   - CLEAR_CALLBACK_REQUEST_BODY empties it again.
4. The bound flow's input args start as empty bindings — fill each with the bindings plugin's
CREATE_*_BINDING tools at the schemaPaths echoed by GET_CALLBACK_DETAIL (bind them to the
request-body fields).

### Editing
UPDATE_CALLBACK_TRIGGER changes only the display name and/or the enabled flag. DELETE_CALLBACK_TRIGGERS
removes custom webhooks by uniqueId. Both fail on managed payment-provider callbacks.

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
| List webhooks | `GET_ALL_CALLBACKS_INFO` | — |
| Webhook detail (+ input binding paths) | `GET_CALLBACK_DETAIL` | `callbackId` |
| Create webhooks | `ADD_CALLBACK_TRIGGERS` | `items` |
| Rename / enable / disable | `UPDATE_CALLBACK_TRIGGER` | `callbackId` |
| Delete webhooks | `DELETE_CALLBACK_TRIGGERS` | `callbackIds` |
| Set the request body to an object type | `SET_CALLBACK_REQUEST_TYPE` | `callbackId`, `objectTypeId` |
| Clear the request body | `CLEAR_CALLBACK_REQUEST_BODY` | `callbackId` |
| Set the response content type | `SET_CALLBACK_RESPONSE_FORMAT` | `callbackId`, `format` |
| Set the response body to an object type | `SET_CALLBACK_RESPONSE_BODY_TYPE` | `callbackId`, `objectTypeId` |
| Clear the response body | `CLEAR_CALLBACK_RESPONSE_BODY` | `callbackId` |
| List body codec slots | `GET_CALLBACK_CODEC_OPTIONS` | `callbackId` |
| Encode a body field (media→url, json→string) | `SET_CALLBACK_CODECS` | `callbackId`, `items`, `target` |
| Remove a body codec | `DELETE_CALLBACK_CODECS` | `callbackId`, `items`, `target` |
| Switch the webhook protocol (e.g. a payment provider) | `SET_CALLBACK_TYPE` | `callbackId`, `callbackType` |

## Worked example: an incoming order webhook

```bash
npx -y zion-mcp@2.7.0 schema tool-call --toolCalls '[
  {"name":"ADD_CALLBACK_TRIGGERS","args":{"items":[
    {"actionFlowId":"<id from GET_ALL_ACTION_FLOWS_INFO>","name":"Order paid"}
  ]}}
]'
npx -y zion-mcp@2.7.0 schema tool-call --toolCalls '[{"name":"GET_CALLBACK_DETAIL","args":{"callbackId":"<echoed id>"}}]'
```
Read the bound flow, the request-body shape and each input arg's binding `schemaPath` back from
`GET_CALLBACK_DETAIL` — the args start as empty bindings, and you fill them from the request-body
fields with the `CREATE_*_BINDING` ops in `data-binding.md`. The bound action flow is fixed at
creation: to re-point a webhook, delete it and create a new one. Payment-provider callbacks come back
with `managed: true` and reject every edit and delete.

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `ADD_CALLBACK_TRIGGERS`

Create custom webhook endpoints, each firing an action flow (by actionFlowId from GET_ALL_ACTION_FLOWS_INFO) when its URL is called; enabled immediately. Describe the request body next with the request-body tool this project's type system provides (the domain prompt names it). Echoes each new webhook's id.
- `items` *(required)*: `array<{actionFlowId: string, name?: string}>` — Webhooks to create; each webhook's public URL ends with its generated uniqueId.

### `UPDATE_CALLBACK_TRIGGER`

Update a custom webhook's display name and/or enabled flag (omitted fields unchanged). The bound action flow cannot be changed — delete and re-create to re-point it. Fails on managed payment-provider callbacks.
- `callbackId` *(required)*: `string` — The uniqueId of the webhook to update.
- `enabled`: `boolean` — Whether the webhook is enabled (an unset flag counts as enabled).
- `name`: `string` — New display name.

### `GET_CALLBACK_DETAIL`

Inspect one webhook by uniqueId: its bound action flow, request body shape, and the flow input args plus their binding schemaPaths (fill these with the bindings tools).
- `callbackId` *(required)*: `string` — The uniqueId of the webhook to inspect (from GET_ALL_CALLBACKS_INFO).

### `SET_CALLBACK_REQUEST_TYPE`

Set a webhook's request body to an existing object type definition (from GET_ALL_OBJECT_DEFINITIONS); its fields describe the incoming payload and become binding sources for the flow's input args. Post-type-refactor projects only.
- `callbackId` *(required)*: `string` — The uniqueId of the webhook whose request body type to set.
- `objectTypeId` *(required)*: `string` — The id of an object type definition (from GET_ALL_OBJECT_DEFINITIONS, or one just created with ADD_OBJECT_TYPE_DEFINITIONS) describing the incoming request payload; its fields become available as binding sources for the action flow's input args.

Then ship:

```bash
npx -y zion-mcp@2.7.0 schema validate && npx -y zion-mcp@2.7.0 project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
