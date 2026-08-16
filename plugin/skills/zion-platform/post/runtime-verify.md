# Runtime verification — run it, then check it

## Verifying your own work
These tools run against the project's REAL backend. There is no staging copy, so prefer the primitives that undo themselves and reach for the ones that do not only when the check genuinely requires them.

Draft versus deployed, which is the distinction that decides whether a green result means anything:
- `runtime.debug_flow` and `runtime.run_code` execute the DRAFT — what you just edited — and revert their database writes by default. This is the verification loop.
- `runtime.invoke_flow` executes the DEPLOYED version with permissions enforced and reverts nothing. A green invoke on a project with pending changes proves nothing about your edits; the result says so when that is the case.

What "done" means here:
- an action flow you built or changed is not done until `runtime.debug_flow` has run it and you have read the terminal node's output;
- a permission change is not done until you have run the same read twice, once as `admin` and once as the role, and compared them — `admin` bypasses row and column permissions, so a single admin read tells you nothing about what a user sees;
- if a check cannot be run, say which one and why. Never report work as verified because it looked right.

When a run fails, its nodes carry a `traceId`: pass it to `logs.search` as `traceId: "…"` to get the server-side detail.

### Two filter grammars — do not mix them
The operator-first grammar documented above (`{"_eq": {"bigint_operand": {…}}}`) is the RAW shape, for documents you hand-write and send with `runtime.graphql`. The typed tools — `runtime.query`, `runtime.aggregate`, `runtime.update`, `runtime.delete` — take the SIMPLIFIED shape instead: `{"column": {"_op": value}}`, e.g. `{"status": {"_eq": "PAID"}}`. Feeding the raw grammar to a typed tool, or the simplified one to a hand-written document, fails validation at the runtime rather than in the tool.

### Binary assets are ids, on the way in and on the way out
Images, videos and files live in object storage; a table column holds only the asset id, never a URL or a path. That changes every side of a runtime call:
- **Reading**: the column is an object in GraphQL, not a scalar, so selecting `cover_image` bare fails with "Subselection required for type 'FZ_Image'". Ask for `cover_image_id`, or select a subfield — `cover_image { id url }`. The typed tools do this for you when they default a field list; hand-written `runtime.graphql` selections have to get it right. The list-valued media types are deprecated; if a project still has one, it reads as a `[col]_ids` array.
- **Writing**: `runtime.insert` and `runtime.update` take the id column too — `{"cover_image_id": 1030…}`, never `{"cover_image": "https://…"}`. A URL in a media column is rejected, and a plausible-looking id you invented points at someone else's asset or at nothing.
- **Getting an id**: an asset does not exist until its bytes have been uploaded, which is a two-step flow — request a presigned URL (`imagePresignedUrl` / `videoPresignedUrl` / `filePresignedUrl`, keyed by the file's Base64 MD5), then HTTP PUT the bytes to it. You can do the first step through `runtime.graphql` but **not the second**: no runtime tool sends raw bytes. Three ways to get a storable id, in order of preference: `web.import_asset` with `mode: RUNTIME`, which does both steps server-side from a public URL and hands back the numeric id; `runtime.query` on `fz_images` / `fz_videos` / `fz_files`, which lists the ids the app already holds; or asking the user. Never fabricate one — a plausible number points at another project's asset or at nothing, and the row looks written until someone opens it.
- **Which id, though**: a data row stores the id the RUNNING APP holds, which is what those tables list and what `mode: RUNTIME` returns. Editor-side asset ids are a different space — an id taken from the editor and written into a row, or an app id bound to a component, resolves to nothing and reports no error either way.
- **Importing at run time is a different problem**: `web.import_asset` runs now, while you are building. An app that has to turn a URL into an asset while it runs needs the **Files** action flow node, which does the same import inside a flow. Build that node into the flow rather than importing the file yourself and writing the id in.
- `url` is a rendering detail: read it when you want to show or check an asset, and store the id everywhere else.

### What runtime.graphql can reach
`runtime.graphql` runs against the app's own API — the generated table operations, `fz_invoke_action_flow*`, `fz_action_flow_result`, the ZAI conversation operations, asset upload. The debug operations behind `runtime.debug_flow`, `runtime.run_code` and `zai.debug_chat` live on a separate admin surface those tools reach for you; asking for them through `runtime.graphql` reports the field as undefined. Subscriptions are unavailable from every tool here — poll the documented query counterpart instead.

### Triggers
There is no fire-a-trigger tool, for anyone — the editor cannot do it either. A database trigger fires when a matching row is really written **and the transaction commits**, so a rollback debug run cannot fire one: the notification the runtime listens for is rolled back with everything else. To exercise a database trigger, write a row with `runtime.insert` or `runtime.update` that matches the trigger's table, operation and condition, then read what it did with `runtime.query` and `logs.search`. Your write is recorded so you can report it; the rows the trigger's own flow wrote in response are not, so name the trigger as well when you hand the run over. `runtime.status` lists the project's triggers with the table and operation each one watches, and a write to a triggered table says so in its confirmation.

A schedule cannot be fired on demand at all. Verify the flow it points at with `runtime.debug_flow`, check the cron expression and end date as configuration, and tell the user the schedule itself is unverified.

### What you leave behind
Nothing you write here can be taken back. There is no undo tool and no cleanup tool: a row from `runtime.insert`, an account from `runtime.as_user`, an overwritten column, a deleted row, and whatever a trigger did in response are all simply in the user's app now. That is the reason to reach for a rollback debug run first and to write only the fixtures a check genuinely needs.

What you do owe the user is an account of it. Rows you inserted and accounts you created are recorded with their ids, and you are reminded of them as you work — finish a verification run by saying what you created and in which table, so they can delete it from the data table if they want to. "I added some test data" is not a handover; "I added order#41 and order#42, and account 9931 with role Customer" is.

The wire protocol for each of these calls — the query language, filter grammar, aggregations, action-flow and AI invocation, third-party calls and asset upload — is documented in the `baas-*` capabilities.

## Testing against the deployed backend (CLI)

This is a **runtime** spoke — it describes calling a DEPLOYED Zion app's SINGLE auto-generated
GraphQL API, which exposes ALL backend interactions (database, action flows, third-party APIs, AI
agents), not editing the editor schema. Endpoints (`{projectExId}` = the project's external id):
- HTTP (queries + mutations): https://zion-app.functorz.com/zero/{projectExId}/api/graphql-v2
- WebSocket (subscriptions):  wss://zion-app.functorz.com/zero/{projectExId}/api/graphql-subscription

Exercise runtime queries/mutations straight from this CLI — already authenticated with the admin token:

```bash
npx -y zion-mcp@2.6.0 runtime graphql --args '{"query":"query { <root_op> { ... } }","variables":{}}'
npx -y zion-mcp@2.6.0 runtime query   --args '{"tableName":"post","where":{"id":{"_eq":1}},"limit":20,"fields":["id","title"]}'
```
`runtime graphql` sends **raw** GraphQL (use the operator-first `where` grammar in `baas-database.md`); `runtime query/insert/update/delete` are typed helpers that take the **simplified** `where` (see `schema-table.md`). Subscriptions (async action-flow results, AI streaming) run from your generated frontend over the WebSocket endpoint (legacy `subscriptions-transport-ws` framing — see `baas-database.md`) — this CLI does not open runtime subscriptions.

## How to drive it (CLI)

```bash
npx -y zion-mcp@2.6.0 runtime query --tableName order --where '{"status":{"_eq":"PAID"}}' --fields '["id","status","total"]'
npx -y zion-mcp@2.6.0 runtime insert --tableName order --objects '[{"status":"PAID","total":100}]'
npx -y zion-mcp@2.6.0 runtime update --tableName order --where '{"id":{"_eq":42}}' --set '{"status":"SHIPPED"}'
npx -y zion-mcp@2.6.0 runtime delete --tableName order --where '{"id":{"_eq":42}}'
npx -y zion-mcp@2.6.0 runtime graphql --query 'query { order_aggregate { aggregate { count } } }'
npx -y zion-mcp@2.6.0 runtime run-code --jsCode 'const total = 2 + 2; total;'
```

`runtime run-code` executes the snippet in the app's Run Code sandbox and returns its value, with
database writes reverted unless `--updateDb` is passed — the cheapest way to see what a Run Code
node will actually return before wiring it into a flow. It is the one runtime verb that goes to the
admin surface rather than the app's own API.

These run against the **deployed** app through the admin channel, which bypasses row and column
permissions — a row you can read here is not necessarily a row your users can read.

### Yours alone
Both you and the in-editor agent can deploy the backend — `npx -y zion-mcp@2.6.0 project sync-backend` here,
`deploy.sync_backend` there — and either way the loop is the same: edit, deploy, then test what
you deployed, because a runtime call always hits the DEPLOYED app. These two have no in-editor
counterpart at all:

- `npx -y zion-mcp@2.6.0 schema validate` — type-check the loaded schema before deploying it.
- `npx -y zion-mcp@2.6.0 site deploy` — ship a built frontend directory, with `site status` / `site abort` for the run; a PROD target needs a human to approve it.

### Not available from this CLI
The verification contract above is written for the in-editor agent, which has tools this CLI does
not. Do not call these; do not report a check as done that needed one.

- `runtime.debug_flow` — the draft lives in the editor session, which this CLI has none of — deploy with `project sync-backend`, then invoke the flow through `runtime graphql` and read the outcome with `runtime query` and `logs search`.
- `runtime.invoke_flow` — call `fz_invoke_action_flow_default_by_latest_version` (or `fz_create_action_flow_task` plus `fz_action_flow_result`) through `runtime graphql` yourself.
- `runtime.as_user` — there is no identity switching here at all — every call goes as admin, so a permission check cannot be run from this CLI. Say so rather than reporting an admin read as evidence of what a user sees.
- `runtime.status` — nothing here reports whether the draft is ahead of the deployed app; run `project sync-backend` first if you need to be sure of what you are testing.
- `zai.debug_chat` — the draft agent is editor-only — invoke a DEPLOYED agent through `runtime graphql` with the ZAI conversation operations.
- `web.fetch_page` — fetch the page with your own tools.
- `web.import_asset` — request a presigned url through `runtime graphql` and PUT the bytes yourself, or ask the user to upload it in the editor.

There is no staging database: everything above hits the project's real backend. Prefer reads and aggregates for assertions, and tell the user before you write.
