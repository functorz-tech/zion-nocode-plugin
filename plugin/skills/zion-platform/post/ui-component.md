# UI components & layout

## Component Domain Knowledge
The UI is a tree of components organized into pages.

### Identity & Tree Rules
Every component has an immutable id and a displayName. Reads return both; write tools verify the {id, displayName} pair and reject a mismatch — re-read, then retry. A container's children order IS the visual order. A component can never be moved into its own subtree.

### Categories
Layout (container — can hold other components via `children`):
- PAGE: root of each page; created without a parent and registered as a new page.
- LAYOUT_VIEW: generic Flexbox container; the most common container.
- MODAL: popup/dialog container; created without a parent, opened/closed via events.

Special (complex — children live in built-in slots seeded at creation and mutated only via the slot tools, never via ADD_COMPONENT / MOVE_COMPONENTS):
- LIST: repeating list; its "cell" slot renders once per data row (optional header/footer).
- TAB_VIEW: tabbed interface; one child view per tab in tabList.
- SELECT_VIEW: built-in normal/selected state views.
- CONDITIONAL_VIEW: one built-in view per condition branch.
- MAP: optional "marker" slot taking a MAP_MARKER.

Leaf/Business (cannot have children): BUTTON, TEXT, IMAGE, VIDEO, LOTTIE, RICH_TEXT, TEXT_INPUT, NUMBER_INPUT, VIDEO_PICKER, FILE_PICKER, SWITCH, MAP_MARKER, CALENDAR, HORIZONTAL_LINE, SHEET, HTML, RICH_TEXT_EDITOR, MIX_IMAGE_PICKER, DATE_TIME_PICKER, DATA_SELECTOR, PROGRESS_BAR. MAP_MARKER can only be placed inside MAP. `GET_SELECTABLE_COMPONENT_TYPES` returns the full creatable catalog with each type's slot fields.

### Hierarchy Rules
Every component except PAGE and MODAL must have a Layout Component as its parent. Prefer relative positioning; use absolute/fixed only when truly necessary.

### Layout
Zion uses Flexbox. Layout containers configure: direction (row/column), justify-content (start, space-between, space-evenly, space-around), gap, wrap, overflow (scroll/visible/hidden).

### Responsive Breakpoints
Phone: 0–767 px (375 px panel) | Tablet: 768–1279 px (820 px panel) | Desktop: 1280–1920 px (1440 px panel). Desktop is the primary breakpoint and the one every read and write defaults to. A narrower breakpoint stores only its differences from desktop, so a field it does not override renders the desktop value at that width — an app built without touching phone is not unstyled there, it is desktop-sized there, which is how a 600 px card ends up running off a 375 px screen. Nothing fails: the schema is valid, the error centre stays clean, and the only symptom is what the user sees on a phone. So a page is not finished at desktop. Read what phone actually resolves to by passing `breakpoint` to `GET_COMPONENT_INFO` and `GET_CONTAINER_CHILDREN_INFO`; the read names which fields that breakpoint overrides and which it inherits. Fix one by resending that field with the `breakpoint` param to `UPDATE_COMPONENT_STYLE` — usually flexDirection 'column' on rows, and widths in % or fill rather than px — and have the user confirm the phone breakpoint in the editor. Resending a value unchanged at a breakpoint pins it there and stops it following desktop, so override only what has to differ. The closing check at the end of a turn names any component that cannot fit the narrow panel and has nothing set there; that is a defect report, not a note.

### Reading the Component Tree
- `GET_ALL_ROOTS_INFO` — every page's {id, displayName}; start here.
- `FIND_COMPONENTS` — when the user names a component ("the submit button") rather than giving an id, resolve the name to its id here instead of descending the tree container by container. One call, whole client, every match tagged with the page it sits on.
- `GET_COMPONENT_INFO` — one component's type and config, by its component id.
- `GET_CONTAINER_CHILDREN_INFO` — the direct children of a container (or a special component's slot), by the container's id.
- `GET_COMPONENT_CONTEXT_INFO` — a component together with its ancestor chain and siblings, so you can see where it sits in the page.
Ids are opaque: copy them verbatim from these read tools and never modify them. A page id (from `GET_ALL_ROOTS_INFO`) is not a component id, and breakpoint keys (phone/tablet/desktop) are tool parameters, never part of an id — passing something like `<pageId>-desktop` as a component target fails with "target component not found".

### Building & Editing the Component Tree
Construction workflow:
1. `GET_ALL_ROOTS_INFO` / `GET_COMPONENT_CONTEXT_INFO` to locate where to build, and read the target container (writes are rejected until their targets have been read).
2. `GET_SELECTABLE_COMPONENT_TYPES` to pick types; `GET_COMPONENT_TEMPLATE` for a valid starter definition — always start from a template, never hand-write component JSON.
3. Edit the template (displayName, parentComponentId, inline children / slot definitions) and submit via `ADD_COMPONENT`. Ids are minted server-side and echoed back — use them directly instead of re-reading. Set the visible copy via content fields ('title'/'text' for BUTTON and TEXT, 'placeholder' for TEXT_INPUT); styles and remaining properties are seeded with the editor's creation defaults so components render visibly; nested children may omit parentComponentId.
4. Verify with `GET_CONTAINER_CHILDREN_INFO`.
Container sizing: a LAYOUT_VIEW hugs its content, so a container created WITH its children inside is sized by them and needs no height of its own; an empty one keeps a small minimum height so it stays visible and droppable, and filling it makes that minimum irrelevant. Give a container an explicit height only when you want a fixed-size box or a scroll viewport — a fixed height its content outgrows clips that content invisibly, and a fixed width crops text mid-word. `UPDATE_COMPONENT_STYLE` merges, so a size you do not name keeps its old value.
Width/height `fill` means "grow along the flex main axis", so it only makes sense on the axis the parent lays out along: `width: fill` in a row, `height: fill` in a column. On the other axis it would unset the size and let the child shrink to its content, so it is applied as '100%' instead and the result says so. There is no `stretch` alignment to fall back on — in Zion a component declares its own size rather than having its container negotiate one.
A row of cards is sized one axis at a time. Along the row, give every child `fill`, or fr shares — `65fr` and `35fr` — when the split is deliberate; never percentages, because fr divides what is left after the gap and the padding while `65%` + `35%` claims the whole width and the gap then pushes it over, wrapping or clipping the last column. Across the row, give every child the SAME `minHeight` so they bottom out together: a real `height` would clip the card whose content grows past it, and with no `stretch` to fall back on a shared floor is as close to equal-height cards as this system gets — a card whose content clears the floor still grows alone, so pick a floor that clears the tallest copy you expect. Set `minWidth` on a child only where that column has a genuine floor: it, not `flexWrap`, is what makes a row break into 2+1 once the container is narrower than the floors plus the gaps. Size the children this way and the container needs nothing beyond `flexDirection`, `gap` and its own width — `justifyContent` and `alignItems` have no free space left to distribute, so reaching for them to fix a ragged row means the children are the wrong size.
A TEXT holds a paragraph only because it is allowed to break: `textWrap` false pins it to one line and cuts it with an ellipsis at its box edge, however wide the sentence is. Anything longer than a label — body copy, a description, a card blurb — wants `textWrap: true`, and `lineClamp` on top of that when it must stop after N lines. When a sentence reports as clipped horizontally, this is nearly always why: widening the box, or setting `overflowX`, does not make the text reflow.
A TEXT_INPUT collecting a password needs `password: true` via `UPDATE_COMPONENT_PROPERTIES`; a "Password" placeholder alone does not mask anything.
Restructure with `MOVE_COMPONENTS` (reparent and/or reorder; the index is the position in the final children list). Restyle with `UPDATE_COMPONENT_STYLE` — merge-based: only the fields you provide change, everything else keeps its current value; flex settings (gap/direction/alignment) apply to containers only and text settings to BUTTON/TEXT/TEXT_INPUT only; target phone/tablet with the breakpoint param and the hover state with variant. Change scalar behavior properties (enums/booleans/numbers, e.g. list direction, input type) or rename a component with `UPDATE_COMPONENT_PROPERTIES` — binding-typed properties stay with the bindings tools. Remove with `DELETE_COMPONENTS` — it deletes each id's entire subtree, and references elsewhere (bindings, event targets, navigation) are NOT cleaned up automatically, so check them afterwards.

### Name The Palette Before You Spend It
A colour written as a hex is pinned to the component you wrote it on. A colour written as a palette reference follows the palette. Both render identically today, so the difference only shows up later — when the same surface colour has to be repeated by hand on the fifth page, or when one recolour turns into an edit per component. Before styling a screen, call `GET_COLOR_THEMES`; a project ships with only Background / Primary / Accent, so define the rest — surface, ink, muted, border, danger, whatever this app actually reuses — with `ADD_COLOR_THEMES` and then pass their referenceValues to `UPDATE_COMPONENT_STYLE`. Naming them by role rather than by colour is what lets a later `UPDATE_COLOR_THEMES` restyle the whole app in one call. A literal hex is for a one-off that genuinely should not move with the theme.
Spacing, corners and type work the same way but through a scale rather than references: call `GET_STYLE_SCALE`, and if the project has none, set one with `SET_STYLE_SCALE` before the first screen — six or seven spacing steps, four or five radii, five or six named text roles (page title, section heading, body, label, caption), at most two font families. Define text as roles rather than sizes: a role carries its weight and line height, so asking for a heading gets you the whole heading and not heading size at body weight. Values you send afterwards are pulled onto the nearest step and the result says when that happened. This is not decoration: a build without a scale ends up with paddings of 12 and 14 and 18 and radii of 9 and 10 and 12, each defensible alone, and the result reads as unfinished even though every screen is individually reasonable. A value far from every step is left as sent, so a deliberate one-off — a pill radius, a hero size — still survives.

### When Nothing Is Said About The Look, Choose One Anyway
A request that names no visual direction — "a task tracker", "a booking page" — is not a request for no styling. It is a request for you to choose, and declining to choose is what produces the grey-box build the user then has to describe their way out of. Set the palette with `ADD_COLOR_THEMES` and the scale with `SET_STYLE_SCALE` before the first screen, and state in one line what you picked, so the user corrects a stated choice instead of discovering an accidental one.
Absent direction, build this: one neutral surface family — page background, a card surface lifted slightly off it, and a border a step darker — plus a single saturated accent spent only on the primary action and the active state. Ink at two weights, one for content and one muted for secondary text. One font family. Spacing in even steps with the page gutter two steps above the in-card gap, so a card reads as sitting on the page rather than pasted onto it. One radius across every surface. Restraint is what reads as finished; a second accent, a gradient or a third font almost never does, and each one costs a decision on every screen after it.
The moment the user does state a direction — a brand colour, "dark", "playful", a screenshot — it overrides all of the above. What fails is splitting the difference: a stated direction applied to the first two screens and the default to the rest reads worse than either one applied throughout, because the inconsistency looks like a bug rather than a choice.

### Three Things Can Be Conditional — Pick The Smallest One
When a screen has two modes, do NOT reach for a conditional container first. Ask what actually differs between the modes, and make only that conditional:
- Conditional DATA (`CREATE_CONDITIONAL_BINDING`) when only VALUES differ — a label, a title, a placeholder, a colour. One component, one conditional binding whose branches are keyed off the deciding value. This is the default answer.
- Conditional ACTION when only BEHAVIOR differs — one button that submits, signs in or signs up depending on the mode. One button, one handler that branches.
- Conditional VIEW (CONDITIONAL_VIEW) only when the STRUCTURE differs — different fields, different counts of children. Branches do NOT share children, so anything you put in a branch is duplicated in every other branch. If the two modes share a form, keep the shared form OUTSIDE the CONDITIONAL_VIEW and put only the differing part inside it.
A "switchable signup / login" is therefore conditional data plus a conditional action over ONE shared form — not two branches each holding a copy of the same fields. Wrapping shared UI in branches is the classic wrong answer: it looks right and silently doubles the tree.
TAB_VIEW is a different question entirely: use it only when the tab strip itself is the affordance the user should see and click. It renders tab chrome you cannot remove.
Both TAB_VIEW and CONDITIONAL_VIEW take `preserveStateOnSwitch`, and it is the same mechanism in both: true keeps every case alive, so a scrolled list, a half-filled input or a loaded query survives switching away and back; false tears the inactive case down and rebuilds it from scratch. Both are created with it ON. Turn it off only when stale state would be wrong (a form that must reset), and remember the cost: every case stays mounted.

### Who Owns A CONDITIONAL_VIEW's Active Branch
Branch conditions and SWITCH_VIEW_CASE write the SAME runtime slot, so exactly one of them owns a given container — never both. A tool call that mixes them is rejected.
- Conditions own it (the default). The mode is app state: a page variable, a query result, a login state. Narrow each branch's condition at the conditionSchemaPath `ADD_COMPONENT` / `ADD_CONDITIONAL_VIEW_BRANCHES` echoes, and change the mode with SET_VARIABLE_DATA. Only this way does the right branch show on first render, react to data, and stay readable by everything else — a title, a colour, a query filter can all condition on the same variable.
- SWITCH_VIEW_CASE owns it only when the selection is purely visual, nothing outside the container will ever need to know which branch is active, and the first branch is the correct starting view — a carousel or an accordion. It never runs on page load and stores the choice where nothing can read it.
Left alone, branch conditions are always-true and the FIRST branch always shows. That is the state right after creation, so narrow them or switch to SWITCH_VIEW_CASE deliberately.

### Slots of Special Components
Slot children are mutated only with the slot tools:
- TAB_VIEW tabs: `ADD_TAB_VIEW_TABS` creates each tab's content container (a LAYOUT_VIEW child) plus its tabList entry, echoing the content child's id and the tab title binding's schemaPath. Tabs are addressed by their content-child id everywhere: `DELETE_TAB_VIEW_TABS` (a TAB_VIEW keeps at least 2 tabs) and `REORDER_TAB_VIEW_TABS` (pass a full permutation).
- CONDITIONAL_VIEW branches: `ADD_CONDITIONAL_VIEW_BRANCHES` inserts always-true branches — narrow each one with the bindings plugin's condition tools at the echoed conditionSchemaPath. Branch order IS evaluation order (first matching condition wins): `REORDER_CONDITIONAL_VIEW_BRANCHES`. The default case cannot be deleted (`DELETE_CONDITIONAL_VIEW_BRANCHES` takes non-default branches only) and always stays last.
- Optional slots: `SET_COMPONENT_SLOT` fills an empty LIST 'header'/'footer' or MAP 'marker' slot with a freshly created default child; `CLEAR_COMPONENT_SLOT` empties it again, deleting the subtree. Required slots (LIST 'cell', SELECT_VIEW states) are fixed at creation — to change one, recreate the component.
- SHEET columns: a SHEET does not take its data through a binding. `SET_SHEET_DATA_SOURCE` builds its embedded query from a table — call it immediately after creating the SHEET, because one without a data source is invalid — then add the displayed columns with `ADD_SHEET_COLUMNS`, managing them by data-field name with `DELETE_SHEET_COLUMNS` / `REORDER_SHEET_COLUMNS`. Filters and sorts go on the echoed querySchemaPath with the request tools.

### Components That Display a Field of Their Data
DATA_SELECTOR, SELECT_VIEW and LIST name one field of their data source: `displayDataField` (the value the user sees) and `cellKeyDataField` (the row identity). Binding the dataSource is not enough — a DATA_SELECTOR or SELECT_VIEW whose data source resolves to a list of objects is invalid until its displayDataField is set. Set it with `UPDATE_COMPONENT_PROPERTIES`, passing the field's plain NAME as a string; the full field reference is resolved for you, so never hand-assemble one. The candidates appear as selectableDataFields in `GET_COMPONENT_INFO` once the dataSource is bound — so the order is always: create → bind dataSource → read the component → name the field.

### Page Data: Queries, Variables & Inputs
Pages have NO separate query list — a page query IS a read-only page variable whose dataSource is a database query:
- `ADD_COMPONENT_QUERY` attaches a table query to a PAGE or MODAL (limit 1 = single row, >1 = list). Components consume it through data bindings (bindings plugin) at the echoed dataSourceSchemaPath; narrow filters and sorting with the request-filter tools at that same path. The result also reports rolesWithoutSelect — grant SELECT with the permission plugin's table-permission tool; never assume it is granted automatically. Tune with `UPDATE_COMPONENT_QUERY`; remove queries and variables alike with `DELETE_COMPONENT_VARIABLES`.
- Page-local mutable state: `ADD_COMPONENT_VARIABLES`, with each typeIdentifier picked from `GET_COMPONENT_VARIABLE_SELECTABLE_TYPES` — never hand-assembled. A new variable starts with no value, so give it an initial one at the `defaultValueSchemaPath` the result echoes whenever anything reads it: conditional cases, branch conditions and SET_VARIABLE_DATA values all test the variable against specific values, and none of them match an empty variable. The page then loads with every dependent case unmatched — blank labels, the fallback branch showing — and a toggle whose new value is itself conditional on the old one cannot recover, because it has nothing to match either.
- Page parameters: `ADD_COMPONENT_INPUTS` / `UPDATE_COMPONENT_INPUT` / `DELETE_COMPONENT_INPUTS`. WEB pages take URL query/path params; WECHAT/MOBILE pages and MODALs take general inputs. Navigation and show-modal call sites pass one value per input.
- `SET_INITIAL_PAGE_ID` sets the app's home page for the current client.

### Data Bindings on Components
`GET_COMPONENT_BINDABLE_PROPERTIES` lists every place a component can be bound — always start there and copy the paths verbatim, never hand-assemble a binding path. It returns two sections: `properties`, the component's own slots, each with its ready-to-use schemaPath, expected type, current binding and scope; and `actionSlots`, the parameter slots of the actions already wired to its events, chained success/failure branches included. `ADD_COMPONENT_ACTIONS` echoes an action's slots when it creates it — bind them in the same turn when you can, and use `actionSlots` to find them again in any later turn. Fill the echoed paths with the bindings plugin's tools: `BROWSE_DATA_BINDING_OPTIONS` + `CREATE_OPTION_BINDING` for dataSource slots and other dynamic values, `CREATE_CONST_BINDING` for literals. A LIST or SELECT_VIEW consumes a page query by binding its dataSource slot to the query's page variable this way (`ADD_COMPONENT_QUERY` first). Inside a list cell (scope "list-cell:<listId>") the options tree additionally offers the current list item's fields under 'item-data' — bind cell children to those instead of re-querying the table.

### Editing Protocol
- Read-before-edit: editing a component (or inserting into a container) you have not read fails with "has not been read" — read it first; after a conflict, re-read and retry. One read of a container covers both adding children to it and editing its own configuration, so `GET_COMPONENT_INFO` and `GET_CONTAINER_CHILDREN_INFO` on the same component is one call wasted.
- Everything you call in a single turn applies as one atomic batch: if any call in it fails, none of them landed, so fix the offending call and resend the whole group. Calls in *different* turns are independent — a failure there leaves earlier turns' work in place.
- Prefer one turn over many. Independent reads should always go together, and a read may sit in the same turn as an edit that depends on it. The one thing you cannot do is use an id or schema path echoed by a call in the same turn — those are only visible once the turn returns, so anything consuming them belongs in the next turn.
- Wiring behavior is always the same three steps: `GET_COMPONENT_SELECTABLE_EVENTS` for the events this component exposes, `GET_SELECTABLE_COMPONENT_ACTIONS` for the actions that event accepts here, then `ADD_COMPONENT_ACTIONS` to attach one. Never guess a kind — the selectable list is already filtered to this client's platform and the project's configuration, so anything absent from it cannot be attached and must be built in the editor instead. Remove any handler with `DELETE_COMPONENT_EVENT_HANDLERS`.
- Every action seeds its parameters as empty bindings and echoes them in bindableSchemaPaths; fill them with the bindings plugin's tools. Anything a specific kind returns beyond that arrives under that handler's `details`.
- ACTION_FLOW calls an action flow (verified {actionFlowId, actionFlowDisplayName} pair, from the actionflow plugin's reads); the flow's declared inputs are seeded and reachable at details.inputArgsSchemaPath.
- NAVIGATION goes to a page (verified {pageId, pageDisplayName} pair), seeding one binding per input the target page declares at details.inputsSchemaPath; NAVIGATION_GO_BACK takes no target.
- SHOW_MODAL opens a MODAL the project defines (verified {modalId, modalDisplayName} pair), seeding one binding per input the modal declares at details.inputsSchemaPath. Chain anything that should run after it closes onto details.onModalClosedSchemaPath. Building a MODAL does not make it reachable — without this action nothing opens it. CLOSE_MODAL is separate and takes no target: it closes the topmost modal (or all of them).
- ALERT_DIALOG is the runtime's own confirm box — a title, a message and two buttons it draws itself, with no MODAL to build or lay out. Use it for "are you sure?"; use SHOW_MODAL when the dialog needs your own components in it. Chain what confirming does onto its ON_CONFIRM branch, or the buttons do nothing.
- MUTATION is database CRUD without an action flow: INSERT/UPDATE/DELETE against a table, with each column seeded at details.columnBindingSchemaPaths and the request reachable at details.requestSchemaPath (narrow UPDATE/DELETE conditions with the request-filter tools). rowScope CURRENT_ITEM — valid only inside a LIST cell — pre-seeds the current-row id condition; refreshOnSuccess re-runs the given targets' queries after it succeeds. rolesWithoutPermission in the result is report-only: grant the table permission with the permission plugin or the mutation is denied at runtime.
- REFRESH re-runs queries of its targets (a PAGE/MODAL with page queries, or a query-bearing LIST / DATA_SELECTOR / SELECT_VIEW). Prefer a MUTATION's refreshOnSuccess when the refresh should follow a write.
- SET_VARIABLE_DATA writes a page/modal variable created with `ADD_COMPONENT_VARIABLES`: send the owning component's verified {targetComponentId, targetDisplayName} pair plus variableName, and fill the seeded `value` slot. Read-only variables (page queries) cannot be written.
- HIDE_MODAL closes a MODAL by its verified {modalId, modalDisplayName} pair, the mirror of SHOW_MODAL. LOG records a titled line plus one fillable slot per name in argNames. AUDIO and LOTTIE play/pause/stop their media.
- USER_LOGIN signs a user in or out. For a sign-in pick identifierType (EMAIL / PHONE / USERNAME) and credentialType (PASSWORD / VERIFICATION_CODE): it builds the per-method login the editor's own picker builds, with two seeded slots — the identifier and the credential, named in bindableSchemaPaths — to fill from the form's inputs. logout takes no credential. Its createAccountOnLogin flag is just an alias for SIGN_UP, building the same register action, so reach for SIGN_UP directly.
- SIGN_UP is the sign-up button, and it is a separate action from USER_LOGIN. Pick identifierType: USERNAME fills username + password, while EMAIL and PHONE fill their address plus a verificationCode you send with SEND_VERIFICATION_CODE (purpose register). Those two also offer a password, unbound by default — the editor's own "use password" switch — and it decides what the account can sign in with later: leave it empty and the account has no password, so a USER_LOGIN with credentialType PASSWORD against it can never succeed for anyone who signed up here. Build the pair together: fill the password path the add echo returns, or sign in with VERIFICATION_CODE.
- SEND_VERIFICATION_CODE / CHECK_VERIFICATION_CODE send a code to an email or phone and verify what the user typed; fill their target (and verificationCode) slots from the form's inputs, and chain what happens next onto onSuccess / onFailure.
- GET_LOCATION, UPLOAD_FILE and GENERATE_QR_CODE produce a RESULT, and the only way to keep it is assignToVariable — a writable page variable created with `ADD_COMPONENT_VARIABLES` on the page the component sits on. Their success actions cannot see the result, so an action without assignToVariable runs and throws its result away. Create the variable first.
- SCHEDULED_JOB_CONTROL starts or pauses a page timer. Scheduled jobs are created in the editor's page Action panel and NO tool creates one, so if the page has none, say so and ask the user to add the job — do not try to build it.
- CONDITIONAL is how ONE event does different things in different cases. Pass `branches` in evaluation order; the first branch whose condition holds runs its actions and no later branch runs, so it is an if / else-if / else chain. Every branch starts always-true and empty: narrow each condition at the echoed conditionSchemaPath with the bindings plugin's condition tools, and put the actions in a branch with a further `ADD_COMPONENT_ACTIONS` whose attachTo carries that branch's conditionalBranchId. Leave the last branch always-true to make it the else. Without it, every action attached to the event fires on every click — onSuccess / onFailure branch on whether a call worked, never on a data condition. Do NOT reach for an action flow to get branching: a flow's BRANCH node runs on the server and cannot navigate, toast or open a modal, so frontend branching belongs here. This decides what HAPPENS; a CONDITIONAL_VIEW decides what is DISPLAYED.
- After substantial UI / binding / event work, run `schema validate` and resolve anything your changes introduced before declaring the work done.
- The editor's error center and canvas screenshots are not reachable from here, so you cannot confirm a result visually: make ALL edits first, validate, then ask the user to check the page in the editor and report what they see.
- There is no per-component conditional visibility — an ordinary component has no data-driven show/hide flag. To make content appear only when a condition holds, wrap it in a CONDITIONAL_VIEW container and gate the branch with the bindings plugin's condition tools (see CONDITIONAL_VIEW branches under "Slots of Special Components").
- Report only what you actually changed. Every claim about appearance — a heading looks larger, a selected state looks different, spacing was tightened — must correspond to an `UPDATE_COMPONENT_STYLE` (or creation-time style) call you made in this session. If a requested look was not achievable with the available tools, say which part you skipped and why; do not describe the component's built-in default styling as if you had applied it. A user who is told the work is done stops checking.

## How to drive it (CLI only)

All commands are `npx -y zion-mcp@2.7.2 <verb>`. A long-lived daemon holds the in-memory CRDT schema session
between calls. **Edits do NOT go live until `project sync-backend`.**

```bash
npx -y zion-mcp@2.7.2 whoami                                    # check auth; if needed: npx -y zion-mcp@2.7.2 login
# create a NEW project (auto-pins it; its pre/post type-system state follows the account rollout):
npx -y zion-mcp@2.7.2 project create --projectName "My App"
# …or pin an EXISTING one (find its exId with npx -y zion-mcp@2.7.2 projects search):
npx -y zion-mcp@2.7.2 project set-current --projectExId <exId>
npx -y zion-mcp@2.7.2 schema load                               # warm the schema session
```

Operations run through one verb:

```bash
npx -y zion-mcp@2.7.2 schema tool-call --toolCalls '[{"name":"<TOOL_NAME>","args":{ ... }}]'
```
Each call is applied immediately — any resulting CRDT patch is uploaded. Batch several calls in one array; use `schema undo` to revert the last change.
A batch is all-or-nothing: when any call in the array fails, the whole batch's changes are discarded even though the other calls returned success — only the failing call's error is reported, so after a batch error re-read (`GET_*`) before assuming anything persisted.

## Operation reference (`schema tool-call` names)

| Intent | `name` | Required `args` |
|---|---|---|
| List pages and modals | `GET_ALL_ROOTS_INFO` | — |
| Component info | `GET_COMPONENT_INFO` | `componentId` |
| Style keys a component type accepts | `GET_COMPONENT_TYPE_CAPABILITIES` | `componentTypes` |
| Data/vars in scope at a component | `GET_COMPONENT_CONTEXT_INFO` | `componentId` |
| Container children info | `GET_CONTAINER_CHILDREN_INFO` | `componentId` |
| Switch a TAB_VIEW between built-in and custom tabs | `SET_TAB_VIEW_TAB_MODE` | `componentId`, `displayName`, `mode` |
| Native tab bar: state, colours, slots | `GET_TAB_BAR_INFO` | — |
| Show or hide the native tab bar | `SET_TAB_BAR_ENABLED` | `enabled` |
| Point a tab-bar slot at a page | `UPDATE_TAB_BAR_ITEMS` | `items` |
| Colour the tab bar | `UPDATE_TAB_BAR_STYLE` | — |
| Duplicate components with their subtrees | `DUPLICATE_COMPONENTS` | `componentIds` |
| Append one event's action handlers to another | `DUPLICATE_COMPONENT_ACTIONS` | `componentId`, `displayName`, `eventType`, `sourceComponentId` |
| Whole theme: variables + scrollbar | `GET_THEME_INFO` | — |
| Add theme variables | `ADD_THEME_VARIABLES` | `items` |
| Rename or revalue theme variables | `UPDATE_THEME_VARIABLES` | `items` |
| Delete theme variables (presets rejected) | `DELETE_THEME_VARIABLES` | `items` |
| Scrollbar appearance (web only) | `SET_SCROLLBAR_STYLE` | — |
| Remove colour-palette entries | `DELETE_COLOR_THEMES` | `items` |
| Theme variables, all categories, plus scrollbar style | `GET_THEME_INFO` | — |
| Add theme variables | `ADD_THEME_VARIABLES` | `items` |
| Rename a theme variable or replace its value | `UPDATE_THEME_VARIABLES` | `items` |
| Remove theme variables | `DELETE_THEME_VARIABLES` | `items` |
| Scrollbar appearance, web only | `SET_SCROLLBAR_STYLE` | — |

Component writes go through the CLI like any other edit: `GET_COMPONENT_TEMPLATE` then `ADD_COMPONENT` to build, `UPDATE_COMPONENT_STYLE` to restyle, `MOVE_COMPONENTS` to restructure, `ADD_COMPONENT_ACTIONS` to wire events, `DELETE_COMPONENTS` to remove — see "Building & Editing the Component Tree" above. Two limits are real. A project still on the legacy component model rejects every component write with `TRACK_MISMATCH` and says so; when that happens, give the user numbered editor steps instead. And you cannot see what you built — the editor canvas and error center are not reachable from here, so a screen can be schema-valid and still look wrong. Verify structurally with `GET_CONTAINER_CHILDREN_INFO` and `GET_INCOMPLETE_STRUCTURES` (its `responsiveRisks` catches the desktop-width-on-phone case), and where appearance is the acceptance criterion, prefer handing the user steps over guessing.

Theme variables carry a value whose shape differs per category, and that shape is not documented per category — read `GET_THEME_INFO` first and copy the shape of an existing variable in the same category, the same way component styles are copied from `GET_COMPONENT_TYPE_CAPABILITIES`. Preset variables cannot be deleted: their ids are what bindings and the theme algorithm resolve against. A component still bound to a deleted variable falls back to no value rather than erroring, so it goes wrong silently — check usages before removing one.

The preset tab-bar icon library is not reachable from here either. A shown tab needs an icon for both its normal and its selected state or the project reports an error, and those exIds are not derivable — `GET_TAB_BAR_INFO` reads back what the slots already hold, so a re-point can reuse those, but a slot with no icon needs one brought in through the editor (`web-assets.md`) or picked there by the user.

Then ship:

```bash
npx -y zion-mcp@2.7.2 schema validate && npx -y zion-mcp@2.7.2 project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
