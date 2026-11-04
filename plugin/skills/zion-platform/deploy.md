# Sync backend (make edits live)

## Backend Sync Domain Knowledge
A project's running backend serves the last SYNCED schema, not the one being edited. The data model, action flows, APIs, permissions, secrets, AI agents and triggers all live there, so until a sync lands, edits to any of them do not exist for a preview, a published app, or anything queried against the runtime — the editor shows the new schema while the backend still answers with the old one.

project sync-backend saves the project, deploys that backend, waits for the pipeline, and reports what happened. It takes no arguments.

### What a sync costs
A sync redeploys the whole backend and takes anywhere from tens of seconds to a few minutes, and one sync after a batch of backend edits covers all of them. Purely visual work (components, styles, and bindings that reference no server entity) needs none at all. Anything tested against the runtime, other than a debug run of a draft flow, sees only what was last synced, so a table, flow or API is not working online until a sync lands.

### Reading the result
- SYNCED — the saved schema is live on the runtime.
- FAILED — the deployment pipeline rejected the schema. Something in the project is invalid; find it (the component plugin's project-error list and the logs plugin's DEPLOYMENT entries are the two places it shows up), fix it, and sync again. Repeating the sync unchanged fails the same way.
- IN_PROGRESS — the pipeline was still running when the wait expired. Nothing is wrong; call project sync-backend again to keep waiting for the same pipeline.
- NOT_STARTED — nothing was deployed. The usual cause is the user dismissing the save confirmation the editor raises when the project changed elsewhere; tell them rather than retrying in a loop.

### One backend, every client app
A project has ONE backend and any number of client apps on top of it. A sync redeploys that shared backend, so the data model, flows, APIs and permissions you changed take effect for every one of them at once — including apps you are not editing and cannot see from this session; the confirmation names the apps it lands for.

### What it is not
Syncing is not publishing. The user's live web or mini-program app is released separately through the editor's Publish flow, which is theirs to run — a sync makes the backend match the editor, it does not push a new version of their app to end users.

## How to drive it (CLI only)

```bash
npx -y zion-mcp@2.8.2 schema validate && npx -y zion-mcp@2.8.2 project sync-backend
```

Every schema edit lands in the CRDT session immediately, but the running backend keeps serving the previously synced schema until `project sync-backend`. So a data model, action flow, API, permission, secret, or AI agent change is invisible to anything that talks to the live app — including `runtime query`, `runtime graphql`, and a published site — until you sync.

Validate first: `schema validate` surfaces the same problems the editor's error center would, and syncing a schema with structural errors is how a project ends up broken in production rather than broken in the editor. `project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending, so a "nothing to sync" error means your edits were never applied — re-read before assuming they were.

Purely visual work (component layout, styling) does not need a sync to be inspected in the editor, but does need one before it is published.
