# BaaS runtime — action-flow invocation

## Invoking actionflows from the runtime GraphQL API
An actionflow is a DAG of nodes (DB ops, sub-flows, conditions, loops) with input/output nodes
defining its args and return. Get the actionflow id, its input arg names, and (optionally) versionId
from the project schema first. $args keys match the input node's argument names.

Sync actionflow — single DB transaction, rolls back ALL writes on error; has a runtime limit
(ACTION_FLOW_TIMEOUT_MILLISECONDS: 15s free / 10min paid). Invoke with a plain mutation; the result
returns inline:
  mutation ($args: Json!) { fz_invoke_action_flow(actionFlowId:"…", versionId:N, args:$args) }

Async actionflow — each node in its own transaction (no global rollback); required for long tasks,
AI-agent nodes, and video generation. Two steps:
1. mutation ($args: Json!) { fz_create_action_flow_task(actionFlowId:"…", versionId:N, args:$args) }
   → returns a taskId (Long).
2. subscription ($taskId: Long!) { fz_listen_action_flow_result(taskId:$taskId){ output status } }
   Several messages may arrive; status transitions CREATED → PROCESSING → COMPLETED|FAILED. The
   final output lives in the COMPLETED message's output field.

## Testing against the deployed backend (CLI)

This is a **runtime** spoke — it describes calling a DEPLOYED Zion app's SINGLE auto-generated
GraphQL API, which exposes ALL backend interactions (database, action flows, third-party APIs, AI
agents), not editing the editor schema. Endpoints (`{projectExId}` = the project's external id):
- HTTP (queries + mutations): https://zion-app.functorz.com/zero/{projectExId}/api/graphql-v2
- WebSocket (subscriptions):  wss://zion-app.functorz.com/zero/{projectExId}/api/graphql-subscription

Exercise runtime queries/mutations straight from this CLI — already authenticated with the admin token:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" support graphql --args '{"query":"query { <root_op> { ... } }","variables":{}}'
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" support query   --args '{"tableName":"post","where":{"id":{"_eq":1}},"limit":20,"fields":["id","title"]}'
```
`support graphql` sends **raw** GraphQL (use the operator-first `where` grammar in `baas-database.md`); `support query/insert/update/delete` are typed helpers that take the **simplified** `where` (see `schema-table.md`). Subscriptions (async action-flow results, AI streaming) run from your generated frontend over the WebSocket endpoint — this CLI does not open runtime subscriptions.

Design the flow in the design-time `actionflow.md`, then invoke the deployed version via the protocol above.
