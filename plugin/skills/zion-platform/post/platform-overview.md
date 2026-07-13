# Platform overview

Zion is a powerful no-code development platform that enables developers to build and deploy web, mobile, and WeChat Mini Program (微信小程序) applications. It integrates a visual drag-and-drop design canvas, an enterprise-grade relational database engine (PostgreSQL), and an event-driven backend logic execution pipeline.

This overview details the structural design, data engines, execution pipelines, and safety requirements of the platform.

---

## 1. Architectural Philosophy (The Three Pillars)

Zion decouples application systems into three isolated yet bound layers:

1. **Design**: The presentation layer, built strictly on nested HTML components obeying the CSS Box Model and CSS Flexbox layouts.
2. **Data**: The persistence layer, powered by a built-in, highly-scalable relational database engine (PostgreSQL). All UI contents are dynamically populated by binding to database rows or workspace API schemas.
3. **Action**: The behavior layer, using an event-driven "Trigger -> Execution" model that connects frontend components to local client tasks or backend operations.

---

## 2. The Mental Model & Engineering Workflow

Developing with Zion requires a rigorous **data-first** engineering approach. LLMs must outline solutions in this exact order:

1. **Data Modeling**: Design the Entity-Relationship (ER) diagram, database tables, constraints, and relationships first.
2. **UI Layout**: Construct nested structural containers (Views) and map static display/input components onto them.
3. **Logic & Binding**: Link dynamic variables/fields to inputs/displays, then map actions or backend workflows to events.
4. **Verification**: Run diagnostic reviews using edit-time error panels and runtime log tracing before deploying.

---

## 3. Database Architecture & Data Binding

Zion runs on an enterprise-grade relational database powered by PostgreSQL.

### 3.1 Relationship Modeling
All table relations map directly to standard SQL constraints:
* **One-to-One (1:1)**: e.g., `account` <-> `user_profile`. Direct foreign key binding.
* **One-to-Many (1:N)**: e.g., `account` -> `post` (One author, multiple posts).
* **Many-to-Many (N:M)**: Must be configured using a **junction/intermediate table** with two distinct 1:N relations. e.g., `account` -> `enrollment` <- `course` represents a Many-to-Many relation between students and courses.

### 3.2 State and Variable Scopes
Zion distinguishes scopes by lifecycle and visibility:

**Application scope** (lifetime = app session):
* **Global Variable**: shared across all pages. Theme, feature flags, cached user state.
* **Current User**: the logged-in user object — id, profile fields, SSO claims.

**Page scope** (lifetime = current page):
* **Page Parameter**: passed from the caller. Path params (`/product/:id`) and query params (`?q=abc`) are both Page Parameters.
* **Page Variable**: local UI/form state — active tab, search keywords, loading flag.
* **Page Data**: result of the page's Page Data Source (a remote query the page binds at load).
* **Component State**: the value of stateful input components (Input, Select, Switch). Two-way binding writes flow back into Component State automatically.

**List-iteration scope** (lifetime = single rendered row):
* **Current Item**: the row object inside a List component. Child components inside the list bind to `Current Item.field` instead of re-querying the table.

**Action-Flow scope** (lifetime = single flow execution):
* **Action Input**: parameters declared on the flow's Start node.
* **Flow Variable**: locals declared by a "Set Variable" node; visible to subsequent nodes in the same flow.
* **Flow Output**: a single typed output value returned by the flow — visible to the caller.
* **Loop Item**: the current iteration value inside a Loop node.

### 3.3 Data Interaction Types
Zion splits operations into two non-overlapping categories to prevent infinite render loops:
* **Query (Read-Only)**: Idempotent operations that fetch data. Used directly as dynamic data sources for Lists, Select Views, and Pages.
* **Mutation (State-Changing)**: Operations that insert, update, or delete records. These **must** be explicitly triggered as actions (e.g., On Click events) and can never be used as static data sources.

### 3.4 Data Binding
Data binding connects data sources to component properties. A property can be set to:
* **OPTION**: a live value from the hierarchical data binding options tree — sources include database table fields, page variables, global variables, logged-in user, list item context ("Current Item"), and actionflow inputs/outputs. The exact `pathInHierarchicalMenu` must match the options tree; never fabricate a path.
* **CONST_VALUE**: a literal constant (string, number, boolean).
* **DISPLAY_NAME**: rename the component's displayed label.

Lists render one row per record from their data source. Children inside a list bind to the list's "Current Item" context (a scoped reference to that row's fields) rather than querying a new table — this preserves the relational link.

---

## 4. Visual Layout & Positioning Engine

Zion's canvas operates as a visual CSS rendering engine.

### 4.1 Positioning Methods
* **Relative (Default)**: Components follow the natural page layout flow, stacking rows/columns according to parent Flexbox rules. Moving a sibling pushes surrounding elements.
* **Absolute**: Removed from the natural document flow. Positioned relative to the boundaries of its **immediate parent** container, ignoring other siblings.
* **Fixed**: Locked to the viewport (browser window). Stays static even when scrolling.

### 4.2 Views & Conditional Views
* **Container (View)**: Wraps child elements, serving as a Flexbox box (Horizontal/Vertical arrangement).
* **Conditional View**: Act as a logical switch statement. Contains multiple distinct child canvases (Cases). At runtime, only the single canvas corresponding to the true evaluation condition is rendered in the DOM, keeping the page clean and secure.

---

## 5. Logic Engines: Frontend vs. Backend

### 5.1 Frontend Actions
Lightweight operations executing directly in the client browser/app. Triggered by UI events.

**Direct database mutations**: The frontend can insert/update/delete rows directly, without routing through an Actionflow, **when both conditions hold**:
1. No ACID requirement (a partial failure across operations is acceptable), and
2. No trust concern (the user has no incentive to forge or tamper with the request — e.g., liking your own post, marking your own task done, incrementing a personal counter). When either condition fails (cross-table consistency needed, or a malicious client could exploit the operation), the mutation must go through a Backend Actionflow instead — see §6.

**Other typical frontend actions**:
* Navigation: push page, redirect, back, new tab, external link.
* Feedback: show/hide Toast, Modal, loading overlay.
* Local state: set Page Variable or Global Variable.
* Data refresh: Refresh (reload a remote data source), List Control (scroll-to / load-more).
* File operations: upload (opens picker), download, clipboard copy, QR scan.
* AI agent invocation: "Run AI" triggers a ZAI agent — returns streamed text or structured JSON; multi-turn via sessionId.
* Trigger Actionflow: "Request – Actionflow" calls a backend flow with input params and receives its outputs; use this for CRUDs that require ACID or trust.
* Auth: Login, Logout, SSO Login.
* Payment: WeChat Pay / Alipay.

### 5.2 Backend Actionflows
Server-side visual execution graphs used to perform robust operations. They are triggered by:
* Client requests (explicit frontend action calls).
* Cron schedules (scheduled background workflows).
* Webhooks (third-party external integrations).
* Database Triggers (automatically runs on `INSERT`/`UPDATE`/`DELETE` of target tables).

**Node types available inside an Actionflow:** Database (query/insert/update/delete on a table), Call workspace API, Run AI (async only), Run Actionflow (sub-flow; sync cannot call async), Set Variable, Permissions (grant/revoke roles), Run Code (JavaScript), Condition (branching), Loop (iterate a list). Current-user and file/media operations are preset integration nodes, not built-in node types. The editor additionally offers a dynamic, server-managed catalog of **preset integration nodes** (the `TEMPLATE_CODE` node type — published templates such as SMS sending or video/AI generation) whose set changes over time, so confirm the current options in the editor or docs rather than assuming a fixed provider.

**Execution modes**: Synchronous flows are ACID — any node failure rolls back all DB changes and the caller blocks waiting for the result. Asynchronous flows are fire-and- forget; only the failing node's DB changes roll back; required for AI agents and video generation. Both modes share a **per-flow total timeout** governed by `ACTION_FLOW_TIMEOUT_MILLISECONDS` — 15 s on free-tier servers, 10 min (600 s) on paid tiers. The budget is for the whole flow, not per node.

---

## 6. Architectural Security Principles

LLMs must enforce the **Zero Trust Client** model when designing Zion architectures:

1. **Untrusted Client Rule**: The client browser is entirely compromiseable. Never calculate prices, validate stock counts, or verify user authorization clearance on the frontend.
2. **Secure Transaction Pattern**:
   * Frontend calls a Backend Actionflow, passing only the `product_id`.
   * The Actionflow queries the product table *on the server* to fetch the immutable price.
   * The Actionflow inserts a secure `Pending` order in the database and returns the    generated `order_id` and the verified price.
   * Frontend maps the *returned* `order_id` and verified price to the payment action.
3. **Webhook Verification**: Business state mutations must **only** occur inside the backend Webhook actionflow triggered directly and securely by the Payment Gateway, never from a frontend callback.

### 6.4 Data Isolation & Access Boundaries (Relation-First Hybrid Pattern)
Whenever a system requires segmented data access (e.g., multi-tenant, department, region, client, or creator separation):
1. **Avoid Role Explosion:** Do NOT create separate roles for every individual organizational entity or group (e.g., do not make separate roles for every department or store branch).
2. **The Hybrid ABAC Pattern (Relation-First):**
   - Instead of manually creating a `group_id` or `tenant_id` column (which is strictly forbidden by database conventions), **create a one-to-many (1:n) RELATION** from your context table (e.g., `department` or `tenant`) to the `Account` table. This auto-generates a non-editable foreign key column `department_id` or `tenant_id` on the `Account` table.
   - Similarly, **create a one-to-many (1:n) RELATION** from the same context table to your business/data table (e.g., `sales_record`), which auto-generates the corresponding FK column on the business table.
   - Define a single generic role representing the user's functional capability (e.g., `Staff`).
   - Configure Row-Level Security (RLS) filters inside that role's table permissions to dynamically isolate rows at query time by comparing the auto-generated FK columns:    Table.group_id == Account.group_id

---

## 7. Environment Lifecycle Model

* **Mirror**: Hot-reloading, local sandbox. Instantly reflects UI edits in a local window. No build compile or backend synchronization is required during rapid frontend iteration.
* **Preview**: Compiles and builds the current source into a staging release, providing a unique staging URL. Necessary for deep verification testing of custom components and payment simulations.
* **Publish**: Deploys the code to the production environment for live users. If database tables, APIs, or Actionflows were modified, they must be synchronized using **Sync Backend** to apply schema migrations to production servers.

---

## 8. Diagnostics and Troubleshooting Framework

1. **Check Edit-time Warnings (Error Collector)**:
   * Look for unmapped input parameters, invalid expressions, or variable mismatch warnings    listed under Zion's navigation bar error console.
2. **Retrieve execution details (Log Service)**:
   * Query the runtime logs using the exact `traceId` attached to the failed request.
   * Trace backend code block execution steps, comparing timestamp drift and evaluating    input parameters.
3. **Database Constraints Validation**:
   * If database actions throw `unique_constraint_violation` during data imports or    integrations, check if the conflict resolution rule is set to `Skip` or `Update`    on the unique fields.

---

## 9. Platform-Specific Features

### Deployment Targets
Each project can publish to three independent client app targets, all sharing the same backend:
* **Web**: Standard browser application with a unique URL.
* **Mobile**: Native iOS/Android app.
* **WeChat Mini Program (微信小程序)**: Mini program running inside the WeChat app. Requires a registered WeChat Mini Program account (AppID) linked to the project.

### WeChat Mini Program Lifecycle
The mini program publish flow is distinct from web/mobile:
1. **Mirror**: Hot-reload preview inside the editor — same as web.
2. **Preview**: Builds a trial version QR code scannable in WeChat DevTools. No audit needed.
3. **Submit for Audit**: Submits the build to WeChat platform for review (1–7 business days).
4. **Release**: After audit approval, publish the version. Backend schema changes still require "Sync Backend" separately; mini program code and backend are released independently. Mini program releases are fully independent from the web/mobile release cycle.

### Payment Gateways
* **WeChat Pay**: Native WeChat Pay v3 API. Requires merchant ID and API credentials.
* **Alipay**: Alipay payment gateway.

### Authentication Providers
* WeChat (OAuth via WeChat Open Platform)
* QQ
* DingTalk (钉钉)
* Feishu / Lark (飞书)
* Email/password

### Timezone & Locale
Default timezone: **UTC+08:00** (China Standard Time). Default UI language: Simplified Chinese (zh-CN).
