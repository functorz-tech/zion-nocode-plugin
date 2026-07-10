# BaaS runtime — AI agent invocation

## Invoking Zion AI agents (ZAI) from the runtime GraphQL API
Agents support multi-modal in/out, prompt templating, context fetching, tool use, and structured
output. Get the agent id (zaiConfigId) and input names from the schema. Agents are ASYNC-only and
delivery depends on config: plain text may stream or not; structured output never streams and always
carries a JSONSchema.

1. Start — mutation returns just the conversationId:
   mutation ($inputArgs: Map_String_ObjectScalar!, $zaiConfigId: String!) {
     fz_zai_create_conversation(inputArgs:$inputArgs, zaiConfigId:$zaiConfigId) }
   inputArgs keys are the schema's input keys. Binary inputs (IMAGE/VIDEO/FILE, or arrays of them)
   use a "_id" suffix on the key and take asset id(s) as values (e.g. the_video → the_video_id:
   1030…; images → images_id: [1020…, …]).
2. Subscribe on the conversationId:
   subscription ($conversationId: Long!) {
     fz_zai_listen_conversation_result(conversationId:$conversationId){
       conversationId status reasoningContent images{ id } data } }
   Status transitions IN_PROGRESS → (STREAMING…) → COMPLETED. Streaming plain text arrives across
   multiple STREAMING messages; the COMPLETED message's data holds the consolidated output.
   reasoningContent streams the same way (partial, then whole at COMPLETED). Non-streaming text: one
   IN_PROGRESS then COMPLETED, no STREAMING. Image-output models fill images[] with FZ_Image ids in
   the COMPLETED message (see asset upload for the id system). Structured-output agents never stream:
   a single COMPLETED whose data is JSON matching the agent's JSONSchema.

Continue a COMPLETED conversation — the same subscription keeps emitting (IN_PROGRESS → … →
COMPLETED again): mutation ($conversationId: Long!, $text: String) {
  fz_zai_send_ai_message(conversationId:$conversationId, text:$text) }
Stop an IN_PROGRESS/STREAMING run (returns true): fz_zai_stop_responding(conversationId:…) — calling
it on a COMPLETED conversation yields a 400 in the GraphQL errors field.

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

Design the agent in the design-time `ai-agent.md`, then invoke the deployed version via the protocol above.
