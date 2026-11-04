# Payments (WeChat Pay / Alipay)

## Payment System Domain Knowledge
Zion's payment system is built around a developer-owned **order table** plus a set of platform-managed payment records and the Actionflows that react to gateway callbacks. On Zion the legacy gateways are WeChat Pay (mini-program and H5/JSAPI) and Alipay (PC web; one-time plus recurring "agreement" subscriptions); payment-config projects also offer Stripe. Airwallex is available on both models where it is enabled for the project. There are two payment models, and GET_PAYMENT_CONFIGS's `legacyPaymentModel` says which one the project uses:
- **Legacy**: activate the module with ACTIVATE_PAYMENT, which generates the gateway Actionflows and webhooks. The gateway credentials, the brand assets and the Pro-plan upgrade are the user's own, in the editor (Action → Payment).
- **Payment configs**: add a config with ADD_PAYMENT_CONFIGS; the first add initializes the payment tables. Credentials (project secrets), products, clients and callback flows are edited through the payment tools; only brand assets and the plan upgrade stay in the editor.

### Prerequisites (in order)
1. The user upgrades the project to the Pro plan or higher (payments are gated by plan; an expired plan breaks live payments and blocks publishing).
2. Create the **order table** (see below) with its required fields.
3. Read `overview.project` (enabled / active types / the order table the payment module is bound to) and GET_PAYMENT_CONFIGS, then follow the matching branch:
   - **Legacy** (`legacyPaymentModel: true`): call ACTIVATE_PAYMENT with the order table's display name. It appends the type's generated flows/callbacks/jobs and grants the logged-in user role permission; a first activation inserts the payment tables. The order table is fixed by the first activation and ignored afterwards. It names what it created; say so in your answer.
   - **Payment configs**: call ADD_PAYMENT_CONFIGS with one or more items. On the first add also pass the order table's display name — it inserts the payment tables; later adds omit it. Each item's provider must be one the product offers (see above). Every item mints a payment config and its empty PAYMENT_CALLBACK and grants every role the provider's default billing methods; several configs may share a provider.
4. Configure credentials.
   - Legacy: the gateway credentials are the user's own, in the editor (Action → Payment) — WeChat Pay (merchant ID, APIv2 key, linked Mini Program AppID, apiclient_cert.p12 for refunds), Alipay (app ID and RSA2 key pair) and/or Airwallex (client ID, API key, webhook secret).
   - Payment configs: create or reuse a project secret (ADD_SECRET_CONFIGS, GET_ALL_SECRET_CONFIGS) and bind it with UPDATE_PAYMENT_CONFIGS — text fields inline, secret fields by secret id; the same call sets supportedProducts and supportedClients. Brand assets stay editor-only.
5. Wire the fulfillment flows.
   - Legacy: the auto-generated payment/refund callback flows are already wired.
   - Payment configs: the callback starts empty. Create or edit the fulfillment Actionflow, then bind it per event kind with SET_PAYMENT_CALLBACK_ACTION and fill the returned `inputArgsSchemaPath` with the binding tools.

### The Order Table (you design this — follow database conventions)
The order table is a normal business table you create; the payment engine references one of its rows by its primary-key id (the `orderId` everywhere below). At minimum it needs:
- an **amount/price** column — type `s:p:decimal`, the authoritative price computed server-side.
- a **status** column — create a custom enum whose options mirror the payment lifecycle
  (e.g. `pending`, `paid`, `cancelled`, `refunded`), then use the exact
  `u:e:<enumId>` returned by the type plugin as the column type; your fulfillment flow
  updates it.
- a **buyer link** — do NOT add a raw `user_id`/`account_id` column. Create a 1:n RELATION from `account` to the order table so the buyer FK is generated automatically (the same manual-foreign-key rule the database plugin enforces). Add product/quantity links as relations too. Naming follows the usual rules: `snake_case` system names, no manual `id`/`created_at` columns (the runner mints the PK).

**Irreversibility warning:** once the order table is linked to the payment module it CANNOT be unlinked, replaced, or deleted. Always confirm the table design with the user and warn them of this before they bind it. Choose the table deliberately.

### Platform-Managed Tables (do NOT create or model these)
Enabling payments auto-creates three system tables — never recreate them as user tables and never write to them directly:
- the **payment** table (`fz_payment` / `fz_payment_record`, 1:N from the order table) — one row per payment attempt, carrying `orderId`, `accountId`, `type`, `status`, `currency`, `grandTotalValue`, `transactionId`.
- the **recurring_payment** table — Alipay agreement subscriptions and their billing cycle.
- the **refund** table (1:N from payment) — refund transactions. The internal payment `status` is a fixed lifecycle enum: `PENDING`, `CANCELLED`, `SUCCESSFUL`, `FAILED`, `PARTIALLY_REFUNDED`, `REFUNDING`, `REFUNDED`. Your order table's own status column is separate from this — your fulfillment flow translates one into the other.

### Auto-Created Callback Actionflows (WeChat Pay / Alipay, legacy projects only)
On a legacy project, enabling payments auto-generates the backend callback Actionflows the editor wires to each payment action. They have no fixed English name (they are the "preset" flows for the channel you enabled); don't delete or rename them. You edit their bodies to run business logic:
- the **payment** flow — triggered automatically after a charge resolves.
- the **refund** flow — triggered automatically after a refund resolves.
- the Alipay **recurring-deduction** flow — a daily scheduled loop that charges due agreements.

Payment-config projects generate no flows: each config's callback starts empty and is bound with SET_PAYMENT_CALLBACK_ACTION per event kind.

### Editing the payment flow (legacy projects only — the one you almost always modify)
The flow has a built-in "parse payment result" code block that outputs four values:
- `orderId` — the order-table row you passed to the payment action; use it to find and update the order.
- `paymentFound` — whether a matching payment record was found.
- `paymentStatus` — `SUCCESSFUL` or `FAILED`.
- `alreadyProcessed` — boolean; `false` on the first trigger, `true` on a re-delivery. **Idempotency contract (critical):** the same payment can trigger the flow more than once. Only run fulfillment / mark the order paid when `paymentFound` is true, `paymentStatus` is the expected value, AND `alreadyProcessed == false`. Guard the fulfillment branch with Condition nodes on these, otherwise duplicate callbacks double-fulfill (double-ship, double-credit). For safety, also add a branch that re-checks the actual paid amount against the order's amount before fulfilling. The refund flow works the same way: its parser returns `orderId`, `refundFound`, `paymentStatus` (`REFUNDED` or `FAILED`), and `alreadyProcessed`.

### Creating an Order (the secure-transaction pattern)
Never let the client decide the price. Order creation must run server-side:
1. Frontend triggers a backend Actionflow passing only identifiers (e.g. `product_id`, quantity) — never the price.
2. The flow queries the product/price server-side, computes the authoritative amount, inserts a new order row with status `pending`, and returns the generated `orderId` and verified amount.
3. The frontend maps the *returned* `orderId` and amount into the WeChat Pay / Alipay / Airwallex payment action. This is the Zero-Trust pattern: the price the customer is charged is the one the server computed and stored on the order, not anything the browser sent.

### Amounts & Currency
The WeChat Pay and Alipay payment actions take the amount in **yuan (元) as a `s:p:decimal` with at most two decimal places, and it must be greater than 0.01** — pass the order's server-computed price directly; do NOT convert to cents/分.

### Airwallex (only where enabled for the project)
Airwallex takes both one-time payments and subscriptions. On a legacy project it is activated like any other type and generates two backend Actionflows; don't delete or rename them. Payment-config projects add an Airwallex config with ADD_PAYMENT_CONFIGS; no flows are generated, so create the fulfillment flow yourself and bind it per event kind with SET_PAYMENT_CALLBACK_ACTION.
1. **Airwallex Payment** — receives every Airwallex webhook as `fz_payment_callback_input` and branches on the event type: payment status changed (`payment_intent.succeeded` / `payment_intent.cancelled`), refund (`refund.accepted` / `refund.failed`), subscription status changes, and `invoice.finalized`. In the payment branch the built-in "Sync payment status" block outputs `orderId`, `paymentStatus` (`SUCCESSFUL` or `CANCELLED`) and `alreadyProcessed`; in the refund branch "Sync refund status" outputs `orderId`, `status` (`REFUNDED` or `FAILED`) and `alreadyProcessed`. Put fulfillment after those blocks under the same idempotency contract: act only on the expected status AND `alreadyProcessed == false`.
2. **Airwallex Subscription Renewal** — called by Airwallex Payment with the invoice id to create the order and payment for a subscription renewal.
Airwallex amounts are decimals in the currency's major unit (10.00, not 1000): pass the order's server-computed decimal price directly.

### Recurring Payments (Alipay, legacy projects only)
Alipay supports subscriptions via a signed agreement. The start action takes `orderId`, a price (Alipay caps recurring amounts at ¥100), a product name, a deduction interval (≥ 7 days), and an Alipay scene code (a user can subscribe to the same scene only once); the cancel action takes the recurring-payment row id. Deductions run in the auto-generated daily loop flow: for each due agreement you must create a NEW order inside the loop and bind its id to the deduction node's `orderId` input. Note: a full Alipay refund does not re-trigger the payment flow. On a payment-config project the Alipay callback offers only `PAYMENT_COMPLETED` and `REFUND_COMPLETED`, and no daily deduction loop is generated — recurring deductions are not managed by these tools there.

### Scope Note
GET_PAYMENT_CONFIGS decides the model. Legacy: ACTIVATE_PAYMENT turns the module on one type at a time and the generated flows are edited as Actionflows; credentials, brand assets and the plan upgrade are the user's own in the editor. Payment configs: ADD_PAYMENT_CONFIGS (the first add initializes), UPDATE_PAYMENT_CONFIGS sets credentials/products/clients, SET_PAYMENT_CALLBACK_ACTION binds the fulfillment flows, GET_PAYMENT_CONFIG_DETAIL reads one and DELETE_PAYMENT_CONFIGS removes one; deleting a bound fulfillment flow only unbinds that event kind — the callback stays until its config is deleted. Brand assets and the plan upgrade remain editor-only. Either way, design the order table first (via the database plugin), then wire the secure order-creation flow. On a legacy project add the `alreadyProcessed` idempotency guard in the generated fulfillment flow; on a payment-config project the callback carries the payment record (and `previousStatus` for subscription changes) — branch on its status and fulfill only on the transition you expect.

## How to drive it (CLI only)

All commands are `npx -y zion-mcp@2.8.2 <verb>`. A long-lived daemon holds the in-memory CRDT schema session
between calls. **Edits do NOT go live until `project sync-backend`.**

```bash
npx -y zion-mcp@2.8.2 whoami                                    # check auth; if needed: npx -y zion-mcp@2.8.2 login
# create a NEW project (auto-pins it):
npx -y zion-mcp@2.8.2 project create --projectName "My App"
# …or pin an EXISTING one (find its exId with npx -y zion-mcp@2.8.2 projects search):
npx -y zion-mcp@2.8.2 project set-current --projectExId <exId>
npx -y zion-mcp@2.8.2 schema load                               # warm the schema session
```

Operations run through one verb:

```bash
npx -y zion-mcp@2.8.2 schema tool-call --toolCalls '[{"name":"<TOOL_NAME>","args":{ ... }}]'
```
Each call is applied immediately — any resulting CRDT patch is uploaded. Batch several calls in one array; use `schema undo` to revert the last change.
A batch is all-or-nothing: when any call in the array fails, the whole batch's changes are discarded even though the other calls returned success — only the failing call's error is reported, so after a batch error re-read (`GET_*`) before assuming anything persisted.

## Operation reference (`schema tool-call` names)

| Intent | `name` | Required `args` |
|---|---|---|
| Read the payment model and existing configs | `GET_PAYMENT_CONFIGS` | — |
| Turn on a legacy-payment project, naming the order table | `ACTIVATE_PAYMENT` | `paymentType` |
| Read one config: products, clients, callbacks, credentials | `GET_PAYMENT_CONFIG_DETAIL` | `configId` |
| Add configs; the first add initializes the payment tables | `ADD_PAYMENT_CONFIGS` | `items` |
| Change a config name/products/clients/credentials | `UPDATE_PAYMENT_CONFIGS` | `items` |
| Delete configs with their callbacks and role grants | `DELETE_PAYMENT_CONFIGS` | `configIds` |
| Bind or unbind a flow for one payment event kind | `SET_PAYMENT_CALLBACK_ACTION` | `callbackId`, `eventKind` |

Read `GET_PAYMENT_CONFIGS` first: `legacyPaymentModel` decides which payment tools apply.

- **Legacy** (`legacyPaymentModel: true`): `ACTIVATE_PAYMENT` per type. Pass `orderTableDisplayName` on the first activation only — an existing table you designed with `schema-table.md`. It provisions the payment tables and the auto-generated WeChat Pay / Alipay Actionflows and webhooks. The gateway credentials, brand assets and plan upgrade stay editor-only (Action → Payment), so hand those to the user.
- **Payment configs**: `ADD_PAYMENT_CONFIGS` — the first add also initializes the payment tables, so it takes the order table's display name; every item carries its provider — from WeChat Pay, Alipay and Stripe, plus Airwallex where the feature is enabled — products, clients and optional credentials, and mints a config plus an empty PAYMENT_CALLBACK. Configure credentials with `UPDATE_PAYMENT_CONFIGS`: text fields inline, secret fields as ids from `secrets.md`; the same call replaces products/clients. `GET_PAYMENT_CONFIG_DETAIL` reads one config — read it before `UPDATE_PAYMENT_CONFIGS` or `DELETE_PAYMENT_CONFIGS`, and read the callback with `GET_CALLBACK_DETAIL` before `SET_PAYMENT_CALLBACK_ACTION` (the lock engine requires those reads). Then bind the callback's per-event fulfillment flows with `SET_PAYMENT_CALLBACK_ACTION` (omit `actionFlowId` to unbind) and fill the returned `inputArgsSchemaPath` through `data-binding.md`. Deleting a bound fulfillment flow only unbinds that event kind — the callback stays until its config is deleted. `DELETE_PAYMENT_CONFIGS` removes configs with their callbacks and every role's grant.

A frontend payment action (`component.md`) carries a `configId` and a `paymentProduct`, and both come from `GET_PAYMENT_CONFIGS`: the id of a config, and one of that config's `usableProductsOnThisClient`. On a legacy project the action carries neither.

Enforce the secure transaction pattern — never compute price on the client; fulfill only in the idempotent fulfillment flow. Provider callbacks arrive `managed: true`: their per-event flows are bound only through `SET_PAYMENT_CALLBACK_ACTION`, and every other edit or delete is rejected.

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `ACTIVATE_PAYMENT`

Activate one payment type on a legacy-payment project. Read GET_PAYMENT_CONFIGS first: its `legacyPaymentModel` must be true — on a payment-config project this call fails with PAYMENT_MODEL_MISMATCH and ADD_PAYMENT_CONFIGS is the right tool. `orderTableDisplayName` is required on the first activation and ignored afterwards. This appends the type's generated action flows, callbacks and scheduled jobs and grants the logged-in user role permission to pay with it; a first activation also inserts the payment tables and relations. The result names what it created. In AUTO_ACCEPT nobody is asked first, so tell the user what appeared in their project. Provider credentials, brand assets and the plan upgrade are the user's own, in the editor's payment settings.
- `orderTableDisplayName`: `string` — Display name of the existing table to use as the order table. Required on first activation (when no payment tables exist yet); ignored afterwards — the order table is then resolved from the existing order→payment relation.
- `paymentType` *(required)*: `enum(STRIPE|AIRWALLEX|ALIPAY|WECHAT)` — Payment type to activate. Which types are selectable depends on the product/client (ZION mini-program: WECHAT only; ZION otherwise: WECHAT/ALIPAY/AIRWALLEX; MOMEN: STRIPE/AIRWALLEX).

### `GET_PAYMENT_CONFIG_DETAIL`

One payment config by id: its provider, products, clients, callback ids and credentials (secret fields appear as secret ids, never values). Read it before UPDATE_PAYMENT_CONFIGS or DELETE_PAYMENT_CONFIGS; its callbackIds are what SET_PAYMENT_CALLBACK_ACTION addresses.
- `configId` *(required)*: `string` — Id of the payment config (from GET_PAYMENT_CONFIGS).

### `ADD_PAYMENT_CONFIGS`

Add payment configs on a payment-config project. The first add also initializes the payment module: pass the existing order table's display name (later adds omit it) and it inserts the payment tables and relations. Every item names its provider — one this product offers, which fixes its credential shape — plus products and clients; credentials may be supplied here or later via UPDATE_PAYMENT_CONFIGS. Each item mints a config and its empty PAYMENT_CALLBACK and grants every role the provider's default billing methods; bind the callback flows with SET_PAYMENT_CALLBACK_ACTION. Several configs for the same provider are allowed.
- `items` *(required)*: `array<object · provider: AIRWALLEX → {credentials?: {apiKeySecretId?: string, clientId?: string, env?: enum(DEMO|PROD), legalEntityId?: string, linkedPaymentAccountId?: string, webhookSecretSecretId?: string}, name: string, supportedClients?: {allowAll?: boolean, clientIds?: array<string>}, supportedProducts?: array<enum(DROP_IN_ELEMENT|HOSTED_PAYMENT_PAGE)>} | ALIPAY → {credentials?: {alipayPublicKey?: string, appId?: string, merchantPrivateKeySecretId?: string}, name: string, supportedClients?: {allowAll?: boolean, clientIds?: array<string>}, supportedProducts?: array<enum(ALIPAY_PC_WEB|ALIPAY_WAP|ALIPAY_APP)>} | STRIPE → {credentials?: {endpointSecretSecretId?: string, publishableKey?: string, secretKeySecretId?: string}, name: string, supportedClients?: {allowAll?: boolean, clientIds?: array<string>}, supportedProducts?: array<enum(CHECKOUT|ELEMENT|PAYMENT_SHEET)>} | WECHAT → {credentials?: {appId?: string, mchId?: string, mchKeySecretId?: string, wechatPaymentBase64CertSecretId?: string}, name: string, supportedClients?: {allowAll?: boolean, clientIds?: array<string>}, supportedProducts?: array<enum(WECHAT_MINI_PROGRAM|WECHAT_JSAPI|WECHAT_NATIVE|WECHAT_H5|WECHAT_APP)>}>` — Configs to add; at least one. Each item's provider selects its shape.
  - `items[].credentials` — Credentials for this provider. Omit a field to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.apiKeySecretId` *(when `provider=AIRWALLEX`)* — Id of a project secret to bind as the API key. Omit to leave it unchanged (unset on create); an empty string clears the binding.
  - `items[].credentials.clientId` *(when `provider=AIRWALLEX`)* — Airwallex client id (plain text). Omit to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.env` *(when `provider=AIRWALLEX`)* — Airwallex environment. Omit to leave it unchanged (unset on create); it cannot be cleared.
  - `items[].credentials.legalEntityId` *(when `provider=AIRWALLEX`)* — Airwallex legal entity id (plain text). Omit to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.linkedPaymentAccountId` *(when `provider=AIRWALLEX`)* — Airwallex linked payment account id (plain text). Omit to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.webhookSecretSecretId` *(when `provider=AIRWALLEX`)* — Id of a project secret to bind as the webhook secret. Omit to leave it unchanged (unset on create); an empty string clears the binding.
  - `items[].name` — Display name, unique among the project's payment configs.
  - `items[].provider` *(when `provider=AIRWALLEX`)* — Required. Must be AIRWALLEX for this item shape.
  - `items[].provider` *(when `provider=ALIPAY`)* — Required. Must be ALIPAY for this item shape.
  - `items[].provider` *(when `provider=STRIPE`)* — Required. Must be STRIPE for this item shape.
  - `items[].provider` *(when `provider=WECHAT`)* — Required. Must be WECHAT for this item shape.
  - `items[].supportedClients` — Clients allowed to use this config. Omit for every client.
  - `items[].supportedClients.allowAll` — true means every current and future client; false restricts the config to clientIds.
  - `items[].supportedClients.clientIds` — Client app exIds the config is usable on. Opaque ids of the project's clients; passed through without validation.
  - `items[].supportedProducts` — Products this config enables; every entry must be one the provider supports. Omit for the provider's defaults.
  - `items[].credentials.alipayPublicKey` *(when `provider=ALIPAY`)* — Alipay public key (plain text). Omit to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.appId` *(when `provider=ALIPAY`)* — Alipay app id (plain text). Omit to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.appId` *(when `provider=WECHAT`)* — WeChat app id (plain text). Omit to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.merchantPrivateKeySecretId` *(when `provider=ALIPAY`)* — Id of a project secret to bind as the merchant private key. Omit to leave it unchanged (unset on create); an empty string clears the binding.
  - `items[].credentials.endpointSecretSecretId` *(when `provider=STRIPE`)* — Id of a project secret to bind as the webhook endpoint secret. Omit to leave it unchanged (unset on create); an empty string clears the binding.
  - `items[].credentials.publishableKey` *(when `provider=STRIPE`)* — Stripe publishable key (plain text, not a secret). Omit to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.secretKeySecretId` *(when `provider=STRIPE`)* — Id of a project secret (GET_ALL_SECRET_CONFIGS) to bind as the Stripe secret key. Omit to leave it unchanged (unset on create); an empty string clears the binding. Never the secret's value.
  - `items[].credentials.mchId` *(when `provider=WECHAT`)* — WeChat merchant id (plain text). Omit to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.mchKeySecretId` *(when `provider=WECHAT`)* — Id of a project secret to bind as the merchant key. Omit to leave it unchanged (unset on create); an empty string clears the binding.
  - `items[].credentials.wechatPaymentBase64CertSecretId` *(when `provider=WECHAT`)* — Id of a project secret to bind as the base64 payment certificate. Omit to leave it unchanged (unset on create); an empty string clears the binding.
- `orderTableDisplayName`: `string` — Display name of the existing table to use as the order table. Required on the first add (when the payment tables do not exist yet); ignored afterwards.

### `UPDATE_PAYMENT_CONFIGS`

Incrementally update payment configs by id; omitted fields stay unchanged. Read each config with GET_PAYMENT_CONFIG_DETAIL first — the lock engine requires it. Each item repeats the config's provider as its discriminator. Secret credential fields take a secret id from the secret tools (an empty string clears the binding); text fields and supportedProducts/supportedClients are written inline, products and clients replaced wholesale. The provider itself cannot be changed.
- `items` *(required)*: `array<object · provider: AIRWALLEX → {configId: string, credentials?: {apiKeySecretId?: string, clientId?: string, env?: enum(DEMO|PROD), legalEntityId?: string, linkedPaymentAccountId?: string, webhookSecretSecretId?: string}, name?: string, supportedClients?: {allowAll?: boolean, clientIds?: array<string>}, supportedProducts?: array<enum(DROP_IN_ELEMENT|HOSTED_PAYMENT_PAGE)>} | ALIPAY → {configId: string, credentials?: {alipayPublicKey?: string, appId?: string, merchantPrivateKeySecretId?: string}, name?: string, supportedClients?: {allowAll?: boolean, clientIds?: array<string>}, supportedProducts?: array<enum(ALIPAY_PC_WEB|ALIPAY_WAP|ALIPAY_APP)>} | STRIPE → {configId: string, credentials?: {endpointSecretSecretId?: string, publishableKey?: string, secretKeySecretId?: string}, name?: string, supportedClients?: {allowAll?: boolean, clientIds?: array<string>}, supportedProducts?: array<enum(CHECKOUT|ELEMENT|PAYMENT_SHEET)>} | WECHAT → {configId: string, credentials?: {appId?: string, mchId?: string, mchKeySecretId?: string, wechatPaymentBase64CertSecretId?: string}, name?: string, supportedClients?: {allowAll?: boolean, clientIds?: array<string>}, supportedProducts?: array<enum(WECHAT_MINI_PROGRAM|WECHAT_JSAPI|WECHAT_NATIVE|WECHAT_H5|WECHAT_APP)>}>` — Configs to update; at least one. Each item repeats its config's provider.
  - `items[].configId` — Id of the config to update (from GET_PAYMENT_CONFIGS).
  - `items[].credentials` — Credentials for this provider. Omit a field to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.apiKeySecretId` *(when `provider=AIRWALLEX`)* — Id of a project secret to bind as the API key. Omit to leave it unchanged (unset on create); an empty string clears the binding.
  - `items[].credentials.clientId` *(when `provider=AIRWALLEX`)* — Airwallex client id (plain text). Omit to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.env` *(when `provider=AIRWALLEX`)* — Airwallex environment. Omit to leave it unchanged (unset on create); it cannot be cleared.
  - `items[].credentials.legalEntityId` *(when `provider=AIRWALLEX`)* — Airwallex legal entity id (plain text). Omit to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.linkedPaymentAccountId` *(when `provider=AIRWALLEX`)* — Airwallex linked payment account id (plain text). Omit to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.webhookSecretSecretId` *(when `provider=AIRWALLEX`)* — Id of a project secret to bind as the webhook secret. Omit to leave it unchanged (unset on create); an empty string clears the binding.
  - `items[].name` — New display name, unique among the project's payment configs. Omit to keep.
  - `items[].provider` *(when `provider=AIRWALLEX`)* — Required. Must be AIRWALLEX, the provider of the config this item updates.
  - `items[].provider` *(when `provider=ALIPAY`)* — Required. Must be ALIPAY, the provider of the config this item updates.
  - `items[].provider` *(when `provider=STRIPE`)* — Required. Must be STRIPE, the provider of the config this item updates.
  - `items[].provider` *(when `provider=WECHAT`)* — Required. Must be WECHAT, the provider of the config this item updates.
  - `items[].supportedClients` — Replaces the supported clients wholesale. Omit to keep.
  - `items[].supportedClients.allowAll` — true means every current and future client; false restricts the config to clientIds.
  - `items[].supportedClients.clientIds` — Client app exIds the config is usable on. Opaque ids of the project's clients; passed through without validation.
  - `items[].supportedProducts` — Replaces the enabled products wholesale. Omit to keep.
  - `items[].credentials.alipayPublicKey` *(when `provider=ALIPAY`)* — Alipay public key (plain text). Omit to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.appId` *(when `provider=ALIPAY`)* — Alipay app id (plain text). Omit to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.appId` *(when `provider=WECHAT`)* — WeChat app id (plain text). Omit to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.merchantPrivateKeySecretId` *(when `provider=ALIPAY`)* — Id of a project secret to bind as the merchant private key. Omit to leave it unchanged (unset on create); an empty string clears the binding.
  - `items[].credentials.endpointSecretSecretId` *(when `provider=STRIPE`)* — Id of a project secret to bind as the webhook endpoint secret. Omit to leave it unchanged (unset on create); an empty string clears the binding.
  - `items[].credentials.publishableKey` *(when `provider=STRIPE`)* — Stripe publishable key (plain text, not a secret). Omit to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.secretKeySecretId` *(when `provider=STRIPE`)* — Id of a project secret (GET_ALL_SECRET_CONFIGS) to bind as the Stripe secret key. Omit to leave it unchanged (unset on create); an empty string clears the binding. Never the secret's value.
  - `items[].credentials.mchId` *(when `provider=WECHAT`)* — WeChat merchant id (plain text). Omit to leave it unchanged (unset on create); an empty string clears.
  - `items[].credentials.mchKeySecretId` *(when `provider=WECHAT`)* — Id of a project secret to bind as the merchant key. Omit to leave it unchanged (unset on create); an empty string clears the binding.
  - `items[].credentials.wechatPaymentBase64CertSecretId` *(when `provider=WECHAT`)* — Id of a project secret to bind as the base64 payment certificate. Omit to leave it unchanged (unset on create); an empty string clears the binding.

### `DELETE_PAYMENT_CONFIGS`

Delete payment configs together with their payment callbacks, revoking each config from every role's payment permission. Read each config with GET_PAYMENT_CONFIG_DETAIL first — the lock engine requires it. Secrets, payment tables and attached payment actions are left untouched.
- `configIds` *(required)*: `array<string>` — Ids of the configs to delete (from GET_PAYMENT_CONFIGS).

### `SET_PAYMENT_CALLBACK_ACTION`

Bind or unbind the Actionflow that runs for one payment event kind on a payment callback. Take the callbackId from GET_PAYMENT_CONFIG_DETAIL, read the callback with GET_CALLBACK_DETAIL first — the lock engine requires it — and take the flow id from the actionflow tools. Omit `actionFlowId` to unbind that event kind. Input args are seeded empty — bind them at the returned `inputArgsSchemaPath` with the binding tools.
- `actionFlowId`: `string` — Id of the action flow to fire for this event kind (from GET_ALL_ACTION_FLOWS_INFO). Omit to unbind this event kind.
- `callbackId` *(required)*: `string` — Id of the payment callback (from GET_PAYMENT_CONFIG_DETAIL's callbackIds). Read the callback with GET_CALLBACK_DETAIL before this call — the lock engine requires that read.
- `eventKind` *(required)*: `enum(PAYMENT_COMPLETED|REFUND_COMPLETED|SUBSCRIPTION_STATUS_CHANGED|SUBSCRIPTION_INVOICE_CREATED|VIRTUAL_PAYMENT_COMPLETED|VIRTUAL_REFUND_COMPLETED)` — Payment event kind to bind or unbind; a callback holds at most one action per kind.

Then ship:

```bash
npx -y zion-mcp@2.8.2 schema validate && npx -y zion-mcp@2.8.2 project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
