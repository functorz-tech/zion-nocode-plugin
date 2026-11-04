# BaaS runtime — AI agent invocation

## Invoking Zion AI agents (ZAI) from the runtime GraphQL API
Agents support multi-modal in/out, prompt templating, context fetching, tool use, and structured
output. Get the agent id (zaiConfigId) from the schema. Agents are ASYNC-only and delivery depends
on config: plain text may stream or not; structured output never streams. Each agent has its own
root mutation fz_zai_<zaiConfigId> (every "-" in the id becomes "_"); introspect the live schema
before assuming exact fields — the argument names and types, and the result's sub-fields.

1. Start — mutation returns just the conversationId:
   mutation ($arg1: …, …) { fz_zai_<zaiConfigId>(arg1:$arg1, …) }
   Arguments are the agent's input args. Binary inputs (IMAGE/VIDEO/FILE, or arrays of them) take
   asset id(s) under a "_id" / "_ids" suffixed argument (e.g. the_video → the_video_id: 1030…).
2. Subscribe on the conversationId with the subscription of the same name:
   subscription ($conversationId: Long!) {
     fz_zai_<zaiConfigId>(conversationId:$conversationId){
       conversationId status reasoningContent images{ id } data output } }
   Status transitions IN_PROGRESS → (STREAMING…) → COMPLETED. Streaming plain text arrives across
   multiple STREAMING messages; the COMPLETED message's data holds the consolidated output.
   reasoningContent streams the same way (partial, then whole at COMPLETED). Non-streaming text: one
   IN_PROGRESS then COMPLETED, no STREAMING. Image-output models fill images[] with FZ_Image ids in
   the COMPLETED message (see asset upload for the id system). output is the agent's result once
   COMPLETED: a String for plain text, or the declared type for a structured-output agent (select
   its sub-fields when that type is an object). Structured-output agents never stream: a single
   COMPLETED carrying it.

Continue a COMPLETED conversation — the same subscription keeps emitting (IN_PROGRESS → … →
COMPLETED again): mutation ($conversationId: Long!, $text: String) {
  fz_zai_send_ai_message(conversationId:$conversationId, text:$text) }
Stop an IN_PROGRESS/STREAMING run (returns true): fz_zai_stop_responding(conversationId:…) — calling
it on a COMPLETED conversation yields a 400 in the GraphQL errors field.

Without a WebSocket, poll the equivalent query instead of subscribing: query ($conversationId: Long!)
{ fz_zai_conversation_result(conversationId:$conversationId){ conversationId status data
reasoningContent } }. It reports the same state the subscription would, so a caller that cannot open
a socket still sees IN_PROGRESS → COMPLETED|FAILED and the final data. Streaming deltas are the one
thing polling cannot reconstruct — for a partial-token view you need the subscription.

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

Design the agent in the design-time `ai-agent.md`, then invoke the deployed version via the protocol above.
