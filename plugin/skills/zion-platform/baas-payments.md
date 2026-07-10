# BaaS runtime — payments

## Invoking payments from the runtime GraphQL API
Prerequisites are design-time (payments.md): payment module enabled and bound to the order table,
gateway credentials configured. Create the order SERVER-SIDE first — an actionflow takes product
ids, computes the authoritative amount, inserts the order row, and returns its id; never trust a
client-computed price. Fulfillment happens ONLY in the idempotent payment callback flow — never on
the return/result page. The gateway's webhook is asynchronous: after payment, wait ~2s before the
first order-status check, then poll briefly.

Alipay (PC web):
  mutation ($outTradeNo:String!, $totalAmount:Float!, $subject:String!, $productCode:String!,
            $returnUrl:String!) {
    alipayTrade(bizContent:{ out_trade_no:$outTradeNo, total_amount:$totalAmount,
      subject:$subject, product_code:$productCode }, returnUrl:$returnUrl) }
out_trade_no is the order id; product_code is FAST_INSTANT_TRADE_PAY for PC web; total_amount is
yuan as Float (> 0.01, ≤ 2 decimals — the backend stringifies it for Alipay); returnUrl MUST be the
site origin (window.location.origin), not a deep path. Returns [formHtml, tradeNo]: render formHtml
AS-IS (extract its <form> + <script> and document.write them) — NEVER rebuild the form; re-escaping
biz_content breaks the RSA2 signature. tradeNo is informational only (the real trade no arrives via
the webhook). alipayAppTrade(bizContent) is the in-app variant (no returnUrl). After payment Alipay
redirects to returnUrl?out_trade_no=…; check status via your order-status flow or the query
getPaymentRecordsByOutTradeNo(outTradeNo:…).

WeChat Pay (mini-program / JSAPI — requires the account's WeChat openId):
  createWechatPayment(type, orderId, description, amount) → SignResult (the client-side sign
  fields); signedPaymentInfo(type, paymentId) re-signs an existing payment record.
Recurring payments (Alipay agreements) and refunds are modeled in payments.md — drive them from
there.

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

Enable the module, design the order table, and wire the idempotent callback flows via the design-time `payments.md`; create orders server-side with an actionflow (`baas-actionflow.md`) — never compute price on the client.
