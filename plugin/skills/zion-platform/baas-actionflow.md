# BaaS runtime — action-flow invocation

## Invoking actionflows from the runtime GraphQL API
An actionflow is a DAG of nodes (DB ops, sub-flows, conditions, loops) with input/output nodes
defining its args and return. Each actionflow has its own root mutation fz_af_<flowId> (every "-"
in the id becomes "_"): its arguments are the flow's input args, its return type is the flow's
output. Introspect the live schema before assuming exact fields — the argument names and types,
and the sub-fields to select when the output is an object.

Sync actionflow — single DB transaction, rolls back ALL writes on error; has a runtime limit
(ACTION_FLOW_TIMEOUT_MILLISECONDS: 15s free / 10min paid). Invoke with a plain mutation; the result
returns inline:
  mutation ($arg1: …, …) { fz_af_<flowId>(arg1:$arg1, …) { <sub-fields, when the output is an object> } }

Async actionflow — each node in its own transaction (no global rollback); required for long tasks,
AI-agent nodes, and video generation. Two steps:
1. mutation ($arg1: …, …) { fz_af_<flowId>(arg1:$arg1, …) }
   → returns a taskId (bigint).
2. Read the result either by the subscription of the same name — subscription ($taskId: bigint!) {
   fz_af_<flowId>(taskId:$taskId){ status output } } — or, without a WebSocket, by polling
   query ($taskId: bigint!) { fz_af_<flowId>_result(taskId:$taskId){ status output } }. Status
   transitions CREATED → PROCESSING → COMPLETED|FAILED; the final output lives in the COMPLETED
   payload's output field (select its sub-fields when it is an object).

fz_af_<flowId> runs the DEPLOYED version of the flow — changes made since the last backend sync are
not exercised by it. Always pass arguments as GraphQL variables, never assembled inline in the query
string. Permission failures surface as errorCode 403 entries in the GraphQL errors array
(classification ACTION_FLOW) — anonymous users commonly lack invoke permission.

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

Design the flow in the design-time `actionflow.md`, then invoke the deployed version via the protocol above.
