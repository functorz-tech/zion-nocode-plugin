# POST capability router

Use this router only after `schema load` returned `post_type_system_refactor`.
Read only the capability files needed for the task. Never read from the sibling variant directory.
A capability absent from this directory is unavailable for this project variant.

## Capabilities

Each capability is a sibling file in this skill folder. When a task calls for one, read that file for its domain rules and `zion-mcp` CLI recipes. Work data-first: build the data model before the UI that displays it and the action flows that mutate it.

- **Database schema (tables, fields, relations, constraints)** (`schema-table.md`) — Create, modify, or delete database tables, columns, relations, and unique constraints. Use when designing or changing the data model (adding a table, field, relation, or constraint), or when you need column-type rules, naming conventions, or system-table constraints.
- **Enums & custom types** (`schema-type.md`) — Create, edit, and inspect enum types and their options, and inspect custom object types. Use before referencing an enum as a database column type, or when adding, renaming, or removing enum options.
- **Data binding** (`data-binding.md`) — Connect a UI component to the data it displays: populate a list or text from a database query, a page or global variable, or the logged-in user; set a literal constant; or rename a component's title. Use when an existing component needs live data or a value bound to it, rather than to create or lay out the component itself.
- **UI components & layout** (`ui-component.md`) — Inspect the UI component tree and learn the component system. Read tools list pages and return a component's info, its tree context (ancestors/siblings), and a container's children. Also supplies component domain knowledge — types, slots for LIST/TAB_VIEW/SELECT_VIEW, Flexbox layout, responsive breakpoints — to guide edits, which the user makes in the editor.
- **Action flows (server-side workflows)** (`actionflow.md`) — Inspect and edit server-side action flows: the workflows that run business logic on a button press, a schedule (cron), a database-change event, or a webhook. Use when adding or changing backend logic such as querying or writing the database, calling an API, running custom JavaScript, branching or looping, or choosing synchronous (transactional) vs. asynchronous execution.
- **API integrations (external HTTP data sources)** (`api.md`) — Inspect and build the project's API integration library — workspaces of external HTTP endpoints (method, URL, typed request parameters + response shapes) plus shared workspace constants for secrets / base URLs — usable as a project data source. The action-flow "Call API" nodes invoke these workspace APIs.
- **AI agents (ZAI)** (`ai-agent.md`) — Inspect and edit AI agents (ZAI): LLM-backed agents the app runs via a "Run AI" action / action-flow node (async). Use to create or configure an agent's model, temperature, rounds, prompts, typed input arguments, and plain-text or structured output.
- **Runtime logs & debugging** (`runtime-logs.md`) — Fetch and interpret server-side runtime logs. Use when debugging a runtime error, when the user shares a traceId, or when they ask about server logs (covers log types like ACTION_FLOW, GATEWAY, SQL_GENERATION and Elasticsearch query syntax).
- **Permissions (RBAC + ABAC)** (`permissions.md`) — Design RBAC + ABAC access control and guide its setup. Use when the project involves multiple user groups, data isolation, restricted actions, 403 errors, or table/column/row permissions and relation-first tenant isolation.
- **Payments (WeChat Pay / Alipay)** (`payments.md`) — Design and wire up payments and guide their setup. Use when the project involves checkout, orders, subscriptions, or refunds (covers order-table design, the auto-created payment Actionflows, the webhook idempotency contract, and secure order creation).
- **Documentation search** (`docs.md`) — Load when you need general platform guidance, references, or specific step-by-step documentation to query the official Zion developer manual.
- **BaaS runtime — data queries** (`baas-database.md`) — Query & mutate a DEPLOYED project's data over its auto-generated GraphQL API — root operations, the operator-first where grammar, aggregations, and app-user auth. Runtime counterpart to schema-table.
- **BaaS runtime — formula functions** (`baas-formulas.md`) — The runtime GraphQL formula-function catalogue (100+ operand functions) and their enum values, used inside where / order_by operands. Distinct from data-binding UI operators.
- **BaaS runtime — action-flow invocation** (`baas-actionflow.md`) — Invoke server-side action flows from the runtime GraphQL API — sync mutation vs async task + result subscription. Runtime counterpart to actionflow.
- **BaaS runtime — AI agent invocation** (`baas-ai-agent.md`) — Invoke ZAI AI agents from the runtime GraphQL API — create a conversation, stream / structured results, continue / stop. Runtime counterpart to ai-agent.
- **BaaS runtime — binary asset upload** (`baas-assets.md`) — Upload binary assets (image / video / file) to a deployed project via the mandatory presigned-URL + MD5 two-step, then reference them by id.
- **BaaS runtime — payments** (`baas-payments.md`) — Charge end-users of a deployed project via Alipay / WeChat Pay — server-side order creation, the gateway mutations, and async webhook + idempotent fulfillment. Runtime counterpart to payments.

For broad architecture or cross-domain planning, read `platform-overview.md`.
