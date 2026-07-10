# BaaS runtime — third-party API invocation

## Invoking imported third-party APIs from the runtime GraphQL API
Each imported HTTP API config is { id, name, operation: 'query'|'mutation' (roughly GET vs POST),
inputs: {key: TypeDefinition}, outputs: {key: TypeDefinition} }, where a TypeDefinition is a scalar
('string'|'boolean'|'number'|'integer') or a nested object of them. The operation fixes the root
GraphQL field: query → query operation_${id}, mutation → mutation operation_${id}. Get the id,
inputs, and outputs from the schema; provide every input unless the user drops it.

Shape:
  mutation ($…: …) { operation_<id>(fz_body:{…}, arg1:$…, …) {
      responseCode
      field_200_json { <sub-field selections> } } }

field_200_json is a fixed field on every TPA-derived operation — the parsed body for any 2xx
response. ALWAYS check responseCode: on 4xx/5xx it is the error code and field_200_json is empty.

## Testing against the deployed backend (CLI)

This is a **runtime** spoke — it describes calling a DEPLOYED Zion app's SINGLE auto-generated
GraphQL API, which exposes ALL backend interactions (database, action flows, third-party APIs, AI
agents), not editing the editor schema. Endpoints (`{projectExId}` = the project's external id):
- HTTP (queries + mutations): https://zion-app.functorz.com/zero/{projectExId}/api/graphql-v2
- WebSocket (subscriptions):  wss://zion-app.functorz.com/zero/{projectExId}/api/graphql-subscription

Exercise runtime queries/mutations straight from this CLI — already authenticated with the admin token:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" runtime graphql --args '{"query":"query { <root_op> { ... } }","variables":{}}'
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" runtime query   --args '{"tableName":"post","where":{"id":{"_eq":1}},"limit":20,"fields":["id","title"]}'
```
`runtime graphql` sends **raw** GraphQL (use the operator-first `where` grammar in `baas-database.md`); `runtime query/insert/update/delete` are typed helpers that take the **simplified** `where` (see `schema-table.md`). Subscriptions (async action-flow results, AI streaming) run from your generated frontend over the WebSocket endpoint (legacy `subscriptions-transport-ws` framing — see `baas-database.md`) — this CLI does not open runtime subscriptions.

Import & configure the external API in the design-time `third-party-api.md`, then invoke it via the protocol above.
