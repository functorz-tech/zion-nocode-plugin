# BaaS runtime — data queries

## Querying Zion data (auto-generated GraphQL over PostgreSQL)
The schema is generated from the data model — introspect the live schema before assuming exact
fields/enums. Each table [table] generates:
- Queries: [table] (list; where/order_by/distinct_on/limit/offset), [table]_by_pk (by id),
  [table]_aggregate (count/sum/avg/min/max), [table]_group_by. A geo_point column adds
  fz_[table]_by_[column] (proximity search); Relay pagination adds a Connection_[table] query.
- Mutations: insert_[table] / insert_[table]_one (nested inserts + on_conflict{constraint,
  update_columns}); update_[table] / update_[table]_by_pk (_set replaces, _inc increments numeric;
  where required for bulk — Hasura _append/_delete_key are NOT supported); delete_[table] /
  delete_[table]_by_pk (where required for bulk); export_[table].
- Subscriptions: [table], [table]_by_pk, [table]_aggregate mirror the queries as live queries.

Column type → GraphQL: text→String, integer→Int, bigint/bigserial→bigint, float8→Float8,
decimal→Decimal, boolean→Boolean, jsonb→jsonb; timestamptz/timetz/date/interval as-is;
geo_point→geography. System columns id/created_at/updated_at are read-only — never in
insert/_set/_inc.

Media columns are stored as ids, never URLs/paths: single = [col]_id (image→FZ_Image, file→FZ_File,
video→FZ_Video); lists (image_list/video_list/file_list) = [col]_ids arrays.

Relationships (defined by foreign keys) become nested fields: 1:1 / N:1 → a single object (e.g.
meta: post_meta), 1:N → an array + a [rel]_aggregate object. Filter to-one relations directly;
to-many use EXISTS semantics.

## Filtering — the strict operator-first where grammar ([table]_bool_exp)
This is the RAW GraphQL where shape for hand-written frontend queries and supportservice_graphql. It
is NOT the simplified where the CLI `support query` helper accepts (that one is {column:{_op:value}})
— see schema-table.md for the helper; use this grammar for real GraphQL.

Logical: _and:[bool_exp], _or:[bool_exp], _not:bool_exp.
Relation filters: navigate by the relationship field name to a nested [related]_bool_exp. To-one
(1:1/N:1) filters the parent by the single related row (author:{name:{_eq:…}}); to-many (1:N/N:M)
uses EXISTS — the parent matches if ANY child does (post_tags:{tag:{name:{_eq:…}}}).

A comparison predicate starts with the OPERATOR, then an operand-wrapper whose type matches the
FINAL evaluated type (not necessarily the column type):
  { "_eq": { "bigint_operand": { "left_operand": {…}, "right_operand": {…} } } }
Operands are {"literal":v} | {"column":"field"} | {"FUNCTION_NAME":{…args}}. Operand-wrapper types:
bigint_operand, text_operand, boolean_operand, timestamptz_operand, decimal_operand, … (the final
type). Operators:
- comparison: _eq _neq _gt _lt _gte _lte
- array: _in _nin
- nullity (unary): _is_null _is_not_null
- text pattern: _like _nlike _ilike _nilike _similar _nsimilar
- jsonb: _contains _contained_in _has_key _has_keys_any _has_keys_all
- boolean: _is_true _is_false
- collection: _is_empty _is_not_empty

Example (month of created_at equals 12):
  { "_eq": { "bigint_operand": {
      "left_operand": { "extract_timestamptz": { "time": {"column":"created_at"}, "unit":"MONTH" } },
      "right_operand": { "literal": "12" } } } }

## Aggregations, window functions & order_by
[table]_aggregate returns { nodes: [table!]! (the raw rows matching where/limit/offset/order_by),
aggregate: {…} }. aggregate fields:
- count(columns:[Enum], distinct:Boolean): no columns → COUNT(*); distinct:true with 1+ columns →
  unique values/combinations; distinct:false with multiple columns → counts only the FIRST column.
- sum, avg: numeric columns only. max, min: comparable columns (numeric/time/text).

Window functions (ROW_NUMBER, RANK, DENSE_RANK, NTH_VALUE) are available in the relevant formula /
window-frame inputs.

order_by supports: direct columns ({title:asc}); related 1:1 records ({author:{name:desc}});
aggregates of N:1 ({post_tags_aggregate:{count:desc}}); and text vector-similarity sort when the
TEXT_COLUMN_VECTOR_SORT extension is applied.

## App-user authentication (for the frontend you generate)
The deployed app's end-users obtain a JWT by logging in / registering against the same GraphQL API;
unauthenticated requests fall back to the anonymous role. Distinct from the developer `login` that
authenticates THIS CLI to Zion.

  mutation AuthenticateWithUsername($username:String!, $password:String!, $register:Boolean!) {
    authenticateWithUsername(username:$username, password:$password, register:$register) {
      account { id permissionRoles } jwt { token }
    }
  }

The returned jwt.token is the Bearer token for that user's subsequent requests. authenticateWith-
Username returns FZ_Account (a subset of account: email, id, permissionRoles, phoneNumber,
profileImageUrl, roles, username). A server-side admin token, when you need one, is copied from the
editor's Connect Backend modal — never hardcode it in client code.

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

Runtime counterpart to the design-time `schema-table.md` (define tables there; query the deployed data here). The formula functions usable inside `where` / `order_by` operands are catalogued in `baas-formulas.md`.
