# Project secrets (API keys & tokens)

## Secret Domain Knowledge
A secret is a named reference to a sensitive value (an API key, token, …) that action flows
and workspace API configs consume by handle rather than inlining the plaintext. Changes take effect
online only after "Sync Backend".

Always call GET_ALL_SECRET_CONFIGS before editing — it returns each secret's id, displayName,
description, and `hasValue` (whether an actual value handle is attached yet).

### Setting the plaintext value
You never handle the plaintext yourself. Call secret save-value with either `secretId` (an
existing secret from GET_ALL_SECRET_CONFIGS) or `displayName` (to create a new one): the editor
prompts the user to type the value, stores it, and attaches the resulting handle, after which
the secret reports `hasValue: true`. The value is entered in the editor and never passes
through the chat, so never ask the user to paste a secret value to you.
- ADD_SECRET_CONFIGS / UPDATE_SECRET_CONFIG take only the opaque `secretKey` handle (never plaintext);
use them to register/rename/retire entries or edit metadata when you are not setting a value.

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

## Setting a value (CLI)

You never handle the plaintext. `secret save-value` opens a native masked dialog on the user's
desktop, stores what they type, and attaches the returned handle:

```bash
npx -y zion-mcp@2.7.4 secret save-value --displayName "OpenAI API key"   # create a secret and set its value
npx -y zion-mcp@2.7.4 secret save-value --secretId <id>                  # set / replace an existing secret's value
```
Pass exactly one of `--displayName` or `--secretId` (ids come from `GET_ALL_SECRET_CONFIGS`). It
needs a desktop session, so it fails on headless / SSH / CI — there is no fallback in which you take
the value yourself: never ask the user to paste a secret into the chat.

## Operation reference (`schema tool-call` names)

| Intent | `name` | Required `args` |
|---|---|---|
| List secrets (with `hasValue`) | `GET_ALL_SECRET_CONFIGS` | — |
| Register secrets | `ADD_SECRET_CONFIGS` | `items` |
| Update a secret | `UPDATE_SECRET_CONFIG` | `secretId` |
| Delete secrets | `DELETE_SECRET_CONFIGS` | `secretIds` |

These ops carry metadata and the opaque `secretKey` handle only — none of them takes a plaintext
value. Register an entry with `ADD_SECRET_CONFIGS` when the value comes later (it reports
`hasValue: false` until one is set), or go straight to `secret save-value`, which creates the entry
and sets the value in one step. Deleting a secret does not clean up the flows / API configs that
reference it — check usages first.

Consume a secret through the **Secret** option in the binding selector at server-side action-flow
binding sites (e.g. a Call API or TPA node's auth header) — never as a literal binding or a parameter
`defaultValue`. Secret options are offered only once at least one secret is registered, and never at
page / component binding sites.

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `ADD_SECRET_CONFIGS`

Register secret entries in the schema. The plaintext value is NOT passed here — supply a pre-uploaded `secretKey` handle or omit it and use save_secret_value to set a value later. Echoes each new secret's id.
- `items` *(required)*: `array<{description?: string, displayName: string, secretKey?: string}>`

### `UPDATE_SECRET_CONFIG`

Update a secret's metadata (displayName / description) and/or its value handle (`secretKey`), incrementally. You cannot set a plaintext value here — use save_secret_value to set one.
- `description`: `string`
- `displayName`: `string` — New display name. Must stay unique across the project's secrets.
- `secretId` *(required)*: `string` — The id of the secret to update (from GET_ALL_SECRET_CONFIGS).
- `secretKey`: `string` — New opaque value handle (NOT the plaintext) after re-uploading the value; omit to keep the current value.

### `DELETE_SECRET_CONFIGS`

Delete secrets by id. References to a deleted secret elsewhere (flows, API configs) are not cleaned up automatically — check usages first.
- `secretIds` *(required)*: `array<string>`

Then ship:

```bash
npx -y zion-mcp@2.7.4 schema validate && npx -y zion-mcp@2.7.4 project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
