# Payments (WeChat Pay / Alipay)

## Payment System Domain Knowledge
Zion's payment system is built around a developer-owned **order table** plus a set
of platform-managed payment records and **auto-generated Actionflows** that react to gateway
webhooks. On Zion the gateway is WeChat Pay / Alipay (one-time + subscriptions). Enabling and configuring
payments is done in the editor (Settings → Payment), not via agent tools — the agent designs
the order table and guides the user through editing the auto-created flows.

### Prerequisites (done in the editor, in order)
1. Upgrade the project to the Pro plan or higher (payments are gated by plan).
2. Create the **order table** (see below) with its required fields.
3. Enable the payment module in Settings → Payment and bind it to the order table.
4. Configure the WeChat Pay / Alipay keys (publishable + secret), brand name, logo, slogan.
5. Confirm the auto-generated webhook(s) are wired (see "Webhook rules").

### The Order Table (you design this — follow database conventions)
The order table is a normal business table you create; the payment engine references one of
its rows by its primary-key id (the `orderId` everywhere below). At minimum it needs:
- an **amount/price** column — type `DECIMAL`, the authoritative price computed server-side.
- a **status** column — model it as a `TEXT` column holding the lifecycle value (e.g.
  `pending`, `paid`, `cancelled`, `refunded`); your fulfillment flow updates it. Enums are
  not available without the refactored type system, so use TEXT here.
- a **buyer link** — do NOT add a raw `user_id`/`account_id` column. Create a 1:n RELATION
  from `account` to the order table so the buyer FK is generated automatically (the same
  manual-foreign-key rule the database plugin enforces). Add product/quantity links as
  relations too.
Naming follows the usual rules: `snake_case` system names, no manual `id`/`created_at`
columns (the runner mints the PK).

**Irreversibility warning:** once the order table is linked to the payment module it CANNOT
be unlinked, replaced, or deleted. Always confirm the table design with the user and warn
them of this before they bind it. Choose the table deliberately.

### Platform-Managed Tables (do NOT create or model these)
The platform owns the payment bookkeeping tables — never recreate them as user tables and
never write to them directly:
- `fz_payment` / `fz_payment_record` — one row per payment attempt, carrying `orderId`,
  `accountId`, `type`, `status`, `currency`, `grandTotalValue`, `transactionId`.
- `recurring_payment` — subscription agreements and their billing cycle.
- `refund` — refund transactions.
The internal payment `status` is a fixed lifecycle enum: `PENDING`, `CANCELLED`,
`SUCCESSFUL`, `FAILED`, `PARTIALLY_REFUNDED`, `REFUNDING`, `REFUNDED`. Your order table's own
status column is separate from this — your fulfillment flow translates one into the other.

### Auto-Created Actionflows (WeChat Pay / Alipay)
Enabling WeChat Pay / Alipay payments auto-generates four backend Actionflows. You don't create them; you
edit their bodies to run your business logic. Do not delete or rename them.
1. **StripePayment** — runs after a successful one-time payment.
2. **StripeRefund** — runs after a successful refund.
3. **StripeRecurringPaymentManagement** — runs when a subscription starts or is cancelled.
4. **StripeRecurringPaymentDeduction** — runs when WeChat Pay / Alipay auto-charges a renewal.

### Editing StripePayment (the one you almost always modify)
The flow receives, as inputs:
- `orderId` — the order-table row to look up and update.
- `paymentStatus` — `SUCCESSFUL` or `FAILED`.
- `alreadyProcessed` — boolean; true when this webhook has already been handled.
**Idempotency contract (critical):** WeChat Pay / Alipay can deliver the same webhook more than once. Only
perform fulfillment / mark the order paid or failed when `paymentStatus` is the expected value
AND `alreadyProcessed == false`. Guard the fulfillment branch with a Condition node on
`alreadyProcessed`, otherwise duplicate webhooks double-fulfill (double-ship, double-credit).
Typical body: branch on `paymentStatus`; on SUCCESSFUL + not-already-processed → update the
order row's status to your "paid" enum value and run fulfillment (grant a role, decrement
stock, etc.); on FAILED → mark the order failed/cancelled.

### Creating an Order (the secure-transaction pattern)
Never let the client decide the price. Order creation must run server-side:
1. Frontend triggers a backend Actionflow passing only identifiers (e.g. `product_id`,
   quantity) — never the price.
2. The flow queries the product/price server-side, computes the authoritative amount, inserts
   a new order row with status `pending`, and returns the generated `orderId` and verified
   amount.
3. The frontend maps the *returned* `orderId` and amount into the WeChat Pay / Alipay payment action.
This is the Zero-Trust pattern: the price the customer is charged is the one the server
computed and stored on the order, not anything the browser sent.

### Amounts & Currency
WeChat Pay / Alipay charges in the **smallest currency unit**. For two-decimal currencies (USD, EUR, GBP,
CNY, ...) that means cents — $10.00 → `1000`. For zero-decimal currencies (JPY, KRW, CLP,
BIF, ...) the amount is the whole-unit value with no ×100. Compute the smallest-unit amount on
the server from the order's `DECIMAL` price; mismatching the unit silently over/under-charges.

### Webhook Rules
- The system auto-generates ONE webhook for successful payments. Do NOT modify it, and do NOT
  add extra WeChat Pay / Alipay events to it in the WeChat Pay / Alipay dashboard — that misfires the payment-success
  logic.
- For other events (failed charges, refunds, subscription changes), create SEPARATE
  webhook trigger + endpoint pairs in Zion and register them in WeChat Pay / Alipay.

### Scope Note
The agent cannot enable the payment module, set WeChat Pay / Alipay keys, or edit Actionflow bodies via
tools — those are editor actions. The agent CAN design the order table (via the database
plugin), and should walk the user step by step through enabling payments, wiring the secure
order-creation flow, and adding the `alreadyProcessed` idempotency guard in StripePayment.

## Consultant mode (no direct edits)

Payment enablement and WeChat Pay / Alipay configuration are editor-only (Settings → Payment). Use this capability to design the order table with `schema-table.md` and guide the user through the auto-generated WeChat Pay / Alipay Actionflows and webhook wiring. Enforce the secure transaction pattern — never compute price on the client; fulfill only in the idempotent webhook Actionflow.

Context helpers (read-only):

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project metadata
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" logs search --customQueryCondition '<es-condition>'
```
