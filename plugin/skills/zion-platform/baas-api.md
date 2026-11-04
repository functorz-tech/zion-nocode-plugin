# BaaS runtime — workspace API invocation

## Invoking workspace APIs from the runtime GraphQL API
Each workspace API has its own root field fz_api_<apiId> — a mutation, and also a query when the
API is usable as a data source — whose arguments are the API's input variables and whose return
type is the API's response object. Introspect the live schema before assuming exact fields — the
argument names and types, and the response's sub-fields, including its responseCode and the
per-branch bodies. ALWAYS check responseCode before reading a body.

## Testing against the deployed backend (CLI)

This is a **runtime** spoke — it describes calling a DEPLOYED Zion app's SINGLE auto-generated
GraphQL API, which exposes ALL backend interactions (database, action flows, third-party APIs, AI
agents), not editing the editor schema. Endpoints (`{projectExId}` = the project's external id):
- HTTP (queries + mutations): https://zion-app.functorz.com/zero/{projectExId}/api/graphql-v2
- WebSocket (subscriptions):  wss://zion-app.functorz.com/zero/{projectExId}/api/graphql-subscription

Exercise runtime queries/mutations straight from this CLI — already authenticated with the admin token:

```bash
npx -y zion-mcp@2.8.2 runtime graphql --args '{"query":"query { <root_op> { ... } }","variables":{}}'
npx -y zion-mcp@2.8.2 runtime query   --args '{"tableName":"post","where":{"id":{"_eq":1}},"limit":20,"fields":["id","title"]}'
```
`runtime graphql` sends **raw** GraphQL (use the operator-first `where` grammar in `baas-database.md`); `runtime query/insert/update/delete` are typed helpers that take the **simplified** `where` (see `schema-table.md`). Subscriptions (async action-flow results, AI streaming) run from your generated frontend over the WebSocket endpoint (legacy `subscriptions-transport-ws` framing — see `baas-database.md`) — this CLI does not open runtime subscriptions.

Build the endpoint in the design-time `api.md`, then invoke it via the protocol above.
