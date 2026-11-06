# Webhooks (incoming HTTP triggers)

## Webhook Domain Knowledge
A webhook is a public HTTP endpoint that fires an action flow every time it is called; the
URL ends with the webhook's generated uniqueId. Only custom webhooks are authorable here —
payment-provider callbacks (Alipay / WeChat / Stripe / …) come back with `managed: true`:
their per-event action flows are bound only through the payments plugin's
SET_PAYMENT_CALLBACK_ACTION, and every other edit and delete is rejected. Deleting a
bound action flow unbinds that event kind instead of deleting the callback. Changes take
effect online only after "Sync Backend".

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

### Query parameters and headers
Values the caller sends in the URL query or in HTTP headers (a token, a signature, a tenant
id) are not part of the body. Declare each one with UPDATE_CALLBACK_TRIGGER's
`requestContextParameters` — a QUERY or HEADER source plus the exact name — and it becomes a
string binding source under `callback-request-context` > QUERY / HEADER for the flow's input
args. The field replaces the whole list, so send the entries already reported by
GET_CALLBACK_DETAIL along with the new ones.

### The response
A webhook answers its caller, and GET_CALLBACK_DETAIL reports that half as `responseBody`.
SET_CALLBACK_RESPONSE_FORMAT picks the content type and REPLACES the body with an empty one of
that format, so set the format first:
- `TEXT_PLAIN` / `TEXT_HTML` answer with a single string — bind it with the bindings plugin's
CREATE_*_BINDING tools at the schemaPath the result echoes.
- `APPLICATION_JSON` answers with a structure — SET_CALLBACK_RESPONSE_BODY_TYPE pointing at an
object type id (from GET_ALL_OBJECT_DEFINITIONS), then bind its value at the schemaPath the result
echoes.
   - CLEAR_CALLBACK_RESPONSE_BODY empties the response body again.

### Editing
UPDATE_CALLBACK_TRIGGER changes the display name, the enabled flag and the declared query
parameters / headers. DELETE_CALLBACK_TRIGGERS
removes custom webhooks by uniqueId. Both fail on managed payment-provider callbacks.

SET_CALLBACK_TYPE is a different thing entirely: it switches the endpoint from a
custom webhook (`DEFAULT`) to a payment provider's notification receiver, which RESETS both
bodies to that provider's fixed shape and loses whatever was configured. Only reach for it
when the user wants this endpoint to receive a provider's callbacks; the provider has to be
enabled for the project first — by a legacy payment activation or a payment config.

### Values that cannot travel as themselves
A media field, or an object field carried as a JSON string, needs a conversion before it can
go over HTTP, and a media field without one leaves the webhook incomplete.
GET_CALLBACK_CODEC_OPTIONS lists exactly which slots of the request and response bodies need one
and which conversions each allows; feed a slot's `typeIdentifier` / `field` back to
SET_CALLBACK_CODECS verbatim, and DELETE_CALLBACK_CODECS to remove one. A slot is a field of a type,
not a path — the same object type reused twice shares one conversion.

## How to drive it (CLI only)

All commands are `npx -y zion-mcp@2.8.3 <verb>`. A long-lived daemon holds the in-memory CRDT schema session
between calls. **Edits do NOT go live until `project sync-backend`.**

```bash
npx -y zion-mcp@2.8.3 whoami                                    # check auth; if needed: npx -y zion-mcp@2.8.3 login
# create a NEW project (auto-pins it and its client; BACKEND_ONLY only if the front end is hosted elsewhere):
# clients: WEB | WECHAT_MINI_PROGRAM | MOBILE | BACKEND_ONLY
npx -y zion-mcp@2.8.3 project create --projectName "My App" --appType WEB
# …or pin an EXISTING one (find its exId with npx -y zion-mcp@2.8.3 projects search):
npx -y zion-mcp@2.8.3 project set-current --projectExId <exId>
npx -y zion-mcp@2.8.3 schema load                               # warm the schema session
```

`project set-current` also makes the project's only client the current one, which is what pages and
components are written to. A project with several clients is refused with its client list (a project created before multi-client has a web client **and** a mini program), so pin the one to work on: `project set-current --projectExId <exId> --appExId <appExId>`.
Pass `--appExId null` to address the project's backend document alone (data model, action flows,
APIs, permissions, secrets).

Operations run through one verb:

```bash
npx -y zion-mcp@2.8.3 schema tool-call --toolCalls '[{"name":"<TOOL_NAME>","args":{ ... }}]'
```
Each call is applied immediately — any resulting CRDT patch is uploaded. Batch several calls in one array; use `schema undo` to revert the last change.
A batch is all-or-nothing: when any call in the array fails, the whole batch's changes are discarded even though the other calls returned success — only the failing call's error is reported, so after a batch error re-read (`GET_*`) before assuming anything persisted.

## Operation reference (`schema tool-call` names)

| Intent | `name` | Required `args` |
|---|---|---|
| List webhooks | `GET_ALL_CALLBACKS_INFO` | — |
| Webhook detail (+ input binding paths) | `GET_CALLBACK_DETAIL` | `callbackId` or `schemaPath` |
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
npx -y zion-mcp@2.8.3 schema tool-call --toolCalls '[
  {"name":"ADD_CALLBACK_TRIGGERS","args":{"items":[
    {"actionFlowId":"<id from GET_ALL_ACTION_FLOWS_INFO>","name":"Order paid"}
  ]}}
]'
npx -y zion-mcp@2.8.3 schema tool-call --toolCalls '[{"name":"GET_CALLBACK_DETAIL","args":{"callbackId":"<echoed id>"}}]'
```
Read the bound flow, the request-body shape and each input arg's binding `schemaPath` back from
`GET_CALLBACK_DETAIL` — the args start as empty bindings, and you fill them from the request-body
fields with the `CREATE_*_BINDING` ops in `data-binding.md`. The bound action flow is fixed at
creation: to re-point a webhook, delete it and create a new one. Payment-provider callbacks come back
with `managed: true`: their per-event action flows are bound only through
`SET_PAYMENT_CALLBACK_ACTION` (see `payments.md`), and every other edit or delete is rejected.

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `ADD_CALLBACK_TRIGGERS`

Create custom webhook endpoints, each firing an action flow (by actionFlowId from GET_ALL_ACTION_FLOWS_INFO) when its URL is called; enabled immediately. Describe the request body next with SET_CALLBACK_REQUEST_TYPE. Echoes each new webhook's id.
- `items` *(required)*: `array<{actionFlowId: string, name?: string}>` — Webhooks to create; each webhook's public URL ends with its generated uniqueId.
  - `items[].actionFlowId` — The id of the action flow this webhook fires (from GET_ALL_ACTION_FLOWS_INFO).
  - `items[].name` — Display name of the webhook; defaults to 'Webhook-<id>' when omitted.

### `UPDATE_CALLBACK_TRIGGER`

Update a custom webhook's display name, enabled flag and/or the query parameters and headers it exposes to the flow (omitted fields unchanged; the parameter list is replaced whole). The bound action flow cannot be changed — delete and re-create to re-point it. Fails on managed payment-provider callbacks.
- `callbackId` *(required)*: `string` — The uniqueId of the webhook to update.
- `enabled`: `boolean` — Whether the webhook is enabled (an unset flag counts as enabled).
- `name`: `string` — New display name.
- `requestContextParameters`: `array<{name: string, source: enum(QUERY|HEADER)}>` — Replaces the whole list of URL query parameters and HTTP headers the webhook reads from its incoming request. Each one becomes a string binding source for the action flow's input args, under `callback-request-context` > QUERY or HEADER. Pass the full list, including entries to keep; [] removes them all, omitting the field leaves them unchanged.
  - `requestContextParameters[].name` — The parameter or header name exactly as the caller sends it, e.g. `token` or `X-Signature`. Unique per source; header names compare case-insensitively.
  - `requestContextParameters[].source` — QUERY for a URL query parameter, HEADER for an HTTP request header.

### `GET_CALLBACK_DETAIL`

Inspect one webhook by uniqueId: its bound action flow, request and response body shapes, declared query parameters / headers, and the flow input args plus their binding schemaPaths (fill these with the bindings tools).
- `callbackId`: `string` — The uniqueId of the webhook to inspect (from GET_ALL_CALLBACKS_INFO).
- `schemaPath`: `array<{index: integer} | {key: string}>` — The schema path of anything inside the webhook, as an alternative to `callbackId`; exactly one of the two is required. Bindings tools already hold a path, so passing it here avoids a lookup.

### `SET_CALLBACK_REQUEST_TYPE`

Set a webhook's request body to an existing object type definition (from GET_ALL_OBJECT_DEFINITIONS); its fields describe the incoming payload and become binding sources for the flow's input args.
- `callbackId` *(required)*: `string` — The uniqueId of the webhook whose request body type to set.
- `objectTypeId` *(required)*: `string` — The id of an object type definition (from GET_ALL_OBJECT_DEFINITIONS, or one just created with ADD_OBJECT_TYPE_DEFINITIONS) describing the incoming request payload; its fields become available as binding sources for the action flow's input args.

Then ship:

```bash
npx -y zion-mcp@2.8.3 schema validate && npx -y zion-mcp@2.8.3 project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
