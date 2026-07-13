# Payments (WeChat Pay / Alipay)

## Payment System Domain Knowledge
Zion's payment system is built around a developer-owned **order table** plus a set of platform-managed payment records and **auto-generated Actionflows** that react to gateway callbacks. On Zion the gateways are WeChat Pay (mini-program and H5/JSAPI) and Alipay (PC web; one-time plus recurring "agreement" subscriptions). Enabling and configuring payments is done in the editor (Settings → Payment), not via agent tools — the agent designs the order table and guides the user through editing the auto-created flows.

### Prerequisites (done in the editor, in order)
1. Upgrade the project to the Pro plan or higher (payments are gated by plan; an expired plan breaks live payments and blocks publishing).
2. Create the **order table** (see below) with its required fields.
3. Enable the payment module in Settings → Payment and bind it to the order table.
4. Configure the gateway credentials — WeChat Pay (merchant ID, APIv2 key, the linked Mini Program AppID, plus the apiclient_cert.p12 for refunds) and/or Alipay (app ID and an RSA2 key pair) — plus brand name, logo, slogan.
5. Confirm the auto-generated payment/refund callback flows are wired.

### The Order Table (you design this — follow database conventions)
The order table is a normal business table you create; the payment engine references one of its rows by its primary-key id (the `orderId` everywhere below). At minimum it needs:
- an **amount/price** column — type `DECIMAL`, the authoritative price computed server-side.
- a **status** column — model it as a `TEXT` column holding the lifecycle value (e.g.
  `pending`, `paid`, `cancelled`, `refunded`); your fulfillment flow updates it. Custom
  enums are unavailable in this project, so use TEXT here.
- a **buyer link** — do NOT add a raw `user_id`/`account_id` column. Create a 1:n RELATION from `account` to the order table so the buyer FK is generated automatically (the same manual-foreign-key rule the database plugin enforces). Add product/quantity links as relations too. Naming follows the usual rules: `snake_case` system names, no manual `id`/`created_at` columns (the runner mints the PK).

**Irreversibility warning:** once the order table is linked to the payment module it CANNOT be unlinked, replaced, or deleted. Always confirm the table design with the user and warn them of this before they bind it. Choose the table deliberately.

### Platform-Managed Tables (do NOT create or model these)
Enabling payments auto-creates three system tables — never recreate them as user tables and never write to them directly:
- the **payment** table (`fz_payment` / `fz_payment_record`, 1:N from the order table) — one row per payment attempt, carrying `orderId`, `accountId`, `type`, `status`, `currency`, `grandTotalValue`, `transactionId`.
- the **recurring_payment** table — Alipay agreement subscriptions and their billing cycle.
- the **refund** table (1:N from payment) — refund transactions. The internal payment `status` is a fixed lifecycle enum: `PENDING`, `CANCELLED`, `SUCCESSFUL`, `FAILED`, `PARTIALLY_REFUNDED`, `REFUNDING`, `REFUNDED`. Your order table's own status column is separate from this — your fulfillment flow translates one into the other.

### Auto-Created Callback Actionflows (WeChat Pay / Alipay)
Enabling payments auto-generates the backend callback Actionflows the editor wires to each payment action. They have no fixed English name (they are the "preset" flows for the channel you enabled); don't delete or rename them. You edit their bodies to run business logic:
- the **payment** flow — triggered automatically after a charge resolves.
- the **refund** flow — triggered automatically after a refund resolves.
- the Alipay **recurring-deduction** flow — a daily scheduled loop that charges due agreements.

### Editing the payment flow (the one you almost always modify)
The flow has a built-in "parse payment result" code block that outputs four values:
- `orderId` — the order-table row you passed to the payment action; use it to find and update the order.
- `paymentFound` — whether a matching payment record was found.
- `paymentStatus` — `SUCCESSFUL` or `FAILED`.
- `alreadyProcessed` — boolean; `false` on the first trigger, `true` on a re-delivery. **Idempotency contract (critical):** the same payment can trigger the flow more than once. Only run fulfillment / mark the order paid when `paymentFound` is true, `paymentStatus` is the expected value, AND `alreadyProcessed == false`. Guard the fulfillment branch with Condition nodes on these, otherwise duplicate callbacks double-fulfill (double-ship, double-credit). For safety, also add a branch that re-checks the actual paid amount against the order's amount before fulfilling. The refund flow works the same way: its parser returns `orderId`, `refundFound`, `paymentStatus` (`REFUNDED` or `FAILED`), and `alreadyProcessed`.

### Creating an Order (the secure-transaction pattern)
Never let the client decide the price. Order creation must run server-side:
1. Frontend triggers a backend Actionflow passing only identifiers (e.g. `product_id`, quantity) — never the price.
2. The flow queries the product/price server-side, computes the authoritative amount, inserts a new order row with status `pending`, and returns the generated `orderId` and verified amount.
3. The frontend maps the *returned* `orderId` and amount into the WeChat Pay / Alipay payment action. This is the Zero-Trust pattern: the price the customer is charged is the one the server computed and stored on the order, not anything the browser sent.

### Amounts & Currency
The WeChat Pay and Alipay payment actions take the amount in **yuan (元) as a `DECIMAL` with at most two decimal places, and it must be greater than 0.01** — pass the order's server-computed price directly; do NOT convert to cents/分.

### Recurring Payments (Alipay only)
Alipay supports subscriptions via a signed agreement. The start action takes `orderId`, a price (Alipay caps recurring amounts at ¥100), a product name, a deduction interval (≥ 7 days), and an Alipay scene code (a user can subscribe to the same scene only once); the cancel action takes the recurring-payment row id. Deductions run in the auto-generated daily loop flow: for each due agreement you must create a NEW order inside the loop and bind its id to the deduction node's `orderId` input. Note: a full Alipay refund does not re-trigger the payment flow.

### Scope Note
The agent cannot enable the payment module, set gateway credentials, or edit Actionflow bodies via tools — those are editor actions. The agent CAN design the order table (via the database plugin), and should walk the user step by step through enabling payments, wiring the secure order-creation flow, and adding the `alreadyProcessed` idempotency guard in the payment flow.

## Consultant mode (no direct edits)

Payment enablement and WeChat Pay / Alipay configuration are editor-only (Settings → Payment). Use this capability to design the order table with `schema-table.md` and guide the user through the auto-generated WeChat Pay / Alipay Actionflows and webhook wiring. Enforce the secure transaction pattern — never compute price on the client; fulfill only in the idempotent webhook Actionflow.

Context helpers (read-only):

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project metadata
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" logs search --customQueryCondition '<es-condition>'
```
