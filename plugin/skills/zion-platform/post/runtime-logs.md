# Runtime logs & debugging

## Log Domain Knowledge
Zion provides a runtime log service showing server-side execution traces.

### Log Types
GATEWAY, ACTION_FLOW, ACTION_FLOW_NODE, ACTION_FLOW_CONTEXT_LOG, DEPLOYMENT, TPA, TRIGGER, SQL_GENERATION, GQL, ZAI.

### Searching
Every entry carries a traceId, and it is the only filter matched against a real indexed column — pass it as the traceId argument to get the whole request in causal order. types and levels are also real filters. Everything in customQueryCondition is a substring scan over the log body, so treat its matches as approximate and never as proof a field equals a value.

### Interpreting Logs
ACTION_FLOW / ACTION_FLOW_NODE errors: misconfigured nodes (wrong field names, type mismatches). GATEWAY errors: API or authentication issues. SQL_GENERATION: data model or query configuration problems. GQL: GraphQL schema or permission issues. Except for GATEWAY, an entry's payload sits under content.message — read nodeName, nodeType, status and errorMessage there first, then input/output only when you need them. Translate technical errors into plain language. Distinguish configuration errors from platform bugs. Never expose sensitive fields (token, account_id) in responses.

### Time Window
Logs fetched in a ±5-minute window around requestCreatedAt (defaults to now).

## How to drive it (CLI only)

Read-only — no session load or save needed.

```bash
npx -y zion-mcp@2.6.2 project set-current --projectExId <exId>
npx -y zion-mcp@2.6.2 logs search --customQueryCondition 'traceId: "abc-123"' --types ACTION_FLOW ACTION_FLOW_NODE --levels ERROR
```
- `--customQueryCondition` is Elasticsearch syntax (e.g. `traceId: "..."`, `message.request.operationName: "MyOp"`).
- `--types`: GATEWAY, ACTION_FLOW, ACTION_FLOW_NODE, ACTION_FLOW_CONTEXT_LOG, DEPLOYMENT, TPA, TRIGGER, SQL_GENERATION, GQL, ZAI. `--levels`: INFO, WARNING, ERROR.
- Window is ±5 minutes around `--requestCreatedAt` (ISO-8601; defaults to now).

Translate technical errors into plain language; never surface sensitive fields (token, account_id) to the user.
