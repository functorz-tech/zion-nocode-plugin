# Runtime logs & debugging

## Log Domain Knowledge
Zion provides a runtime log service showing server-side execution traces.

### Log Types
GATEWAY, ACTION_FLOW, ACTION_FLOW_NODE, ACTION_FLOW_CONTEXT_LOG, DEPLOYMENT,
TPA, TRIGGER, SQL_GENERATION, GQL, ZAI.
Search logs using customQueryCondition (Elasticsearch syntax), e.g.:
'traceId: "abc-123"' or 'message.request.operationName: "MyOp"'.

### Interpreting Logs
ACTION_FLOW / ACTION_FLOW_NODE errors: misconfigured nodes (wrong field names, type mismatches).
GATEWAY errors: API or authentication issues.
SQL_GENERATION: data model or query configuration problems.
GQL: GraphQL schema or permission issues.
Translate technical errors into plain language. Distinguish configuration errors from platform bugs.
Never expose sensitive fields (token, account_id) in responses.

### Time Window
Logs fetched in a ±5-minute window around requestCreatedAt (defaults to now).

## How to drive it (CLI only)

Read-only — no session load or save needed.

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project set-current --projectExId <exId>
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" logs search --customQueryCondition 'traceId: "abc-123"' --types ACTION_FLOW ACTION_FLOW_NODE --levels ERROR
```
- `--customQueryCondition` is Elasticsearch syntax (e.g. `traceId: "..."`, `message.request.operationName: "MyOp"`).
- `--types`: GATEWAY, ACTION_FLOW, ACTION_FLOW_NODE, ACTION_FLOW_CONTEXT_LOG, DEPLOYMENT, TPA, TRIGGER, SQL_GENERATION, GQL, ZAI. `--levels`: INFO, WARNING, ERROR.
- Window is ±5 minutes around `--requestCreatedAt` (ISO-8601; defaults to now).

Translate technical errors into plain language; never surface sensitive fields (token, account_id) to the user.
