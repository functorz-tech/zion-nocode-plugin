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
Phone: 0–767 px (375 px panel) | Tablet: 768–1279 px (820 px panel) | Desktop: 1280–1920 px (1440 px panel). Desktop is the primary breakpoint and the one every read and write defaults to. A narrower breakpoint stores only its differences from desktop, so a field it does not override renders the desktop value at that width — an app built without touching phone is not unstyled there, it is desktop-sized there, which is how a 600 px card ends up running off a 375 px screen. Nothing fails: the schema is valid, the error center stays clean, and the only symptom is what the user sees on a phone. Read what phone actually resolves to by passing `breakpoint` to `GET_COMPONENT_INFO` and `GET_CONTAINER_CHILDREN_INFO`; the read names which fields that breakpoint overrides and which it inherits. Fix one by resending that field with the `breakpoint` param to `UPDATE_COMPONENT_STYLE` — usually flexDirection 'column' on rows, and widths in % or fill rather than px; the phone breakpoint in the editor shows the result. Resending a value unchanged at a breakpoint pins it there and stops it following desktop, so override only what has to differ.

### Reading the Component Tree
Ids are opaque: copy them verbatim from these read tools and never modify them. A page id (from `GET_ALL_ROOTS_INFO`) is not a component id, and breakpoint keys (phone/tablet/desktop) are tool parameters, never part of an id — passing something like `<pageId>-desktop` as a component target fails with "target component not found".

### Building & Editing the Component Tree
Construction workflow: read the target container, pick types with `GET_SELECTABLE_COMPONENT_TYPES`, edit `GET_COMPONENT_TEMPLATE` output and submit it via `ADD_COMPONENT`, then use the ids it echoes instead of re-reading.
Container sizing: a LAYOUT_VIEW hugs its content, so a container created WITH its children inside is sized by them and needs no height of its own; an empty one keeps a small minimum height so it stays visible and droppable, and filling it makes that minimum irrelevant. Give a container an explicit height only when you want a fixed-size box or a scroll viewport — a fixed height its content outgrows clips that content invisibly, and a fixed width crops text mid-word. `UPDATE_COMPONENT_STYLE` merges, so a size you do not name keeps its old value.
Width/height `fill` means "grow along the flex main axis", so it only makes sense on the axis the parent lays out along: `width: fill` in a row, `height: fill` in a column. On the other axis it would unset the size and let the child shrink to its content, so it is applied as '100%' instead and the result says so. There is no `stretch` alignment to fall back on — in Zion a component declares its own size rather than having its container negotiate one.
A row of cards is sized one axis at a time. Along the row, give every child `fill`, or fr shares — `65fr` and `35fr` — when the split is deliberate; never percentages, because fr divides what is left after the gap and the padding while `65%` + `35%` claims the whole width and the gap then pushes it over, wrapping or clipping the last column. Across the row, give every child the SAME `minHeight` so they bottom out together: a real `height` would clip the card whose content grows past it, and with no `stretch` to fall back on a shared floor is as close to equal-height cards as this system gets — a card whose content clears the floor still grows alone, so pick a floor that clears the tallest copy you expect. Set `minWidth` on a child only where that column has a genuine floor: it, not `flexWrap`, is what makes a row break into 2+1 once the container is narrower than the floors plus the gaps. Size the children this way and the container needs nothing beyond `flexDirection`, `gap` and its own width — `justifyContent` and `alignItems` have no free space left to distribute, so reaching for them to fix a ragged row means the children are the wrong size.
When a sentence reports as clipped horizontally, it is nearly always `textWrap` false (see Typography below): widening the box, or setting `overflowX`, does not make the text reflow.
A TEXT_INPUT collecting a password needs `password: true` via `UPDATE_COMPONENT_PROPERTIES`; a "Password" placeholder alone does not mask anything.

### A Look Is Decided, Not Defaulted
A request that names no visual direction — "a task tracker", "a booking page" — is not a request for no styling: the platform's creation defaults are what produce the grey-box build the user then has to describe their way out of. The subject is where a non-generic direction comes from — what this app is, who opens it, the one job it does, and the palette, typeface and type roles that reading implies — and it is the only input no other project shares: a shift-handover tool for night-shift nurses and a portfolio for a ceramicist should not resolve to the same screen. Written into the theme (next section), that direction holds every screen built out of it together. A direction the user states — a brand colour, "dark", "playful", a screenshot — replaces your own everywhere. What fails is splitting the difference: a stated direction on the first two screens and your own default on the rest reads worse than either applied throughout, because inconsistency looks like a bug rather than a choice.

### The Theme Is The Design System
Every style value that has a theme category is sent as that variable's referenceValue, `$$<id>`, never as a literal: colours (backgroundColor, textColor, borderColor, gradient stops, shadow.color) from colour variables, padding/margin/gap from spacings, borderRadius from radii, fontSize, fontWeight and fontFamily from their own categories, and a whole shadow as `shadow: {variable: '$$<id>'}`. The id resolves against the field's category, so `$$md` under padding is the spacing md and under borderRadius the radius md. A literal renders the same today and is pinned to the component it was written on; a reference follows the theme. That difference is the whole point: "make it rounder", "a different font", "warmer" is then a few `UPDATE_THEME_VARIABLES` edits that every screen follows, instead of a sweep that touches every component and still misses some. Only lineHeight, letterSpacing, borderWidth, opacity and sizes have no category and stay literal.
That is the default, not a rule over the user:
- A value the user gives for one component ("make this button 13px", "this banner #ff6600") is written as that literal on that component — not rounded to the nearest variable, not turned into a new variable, and not applied by retuning a variable every screen shares. When it is unclear whether they mean this component or the whole app, ask.
- Change only what the request touches. Existing literals stay literal unless the user asks for the restyle; do not convert them on the side.
- Add a variable only for a value several components share, never for a one-off.
`GET_THEME_INFO` shows what the theme holds. A theme already carries colour roles (Primary, Secondary, Tertiary, Text_primary, Text_secondary, Background, Surface, Border, Success, Alert, Error), font sizes h1 through caption that already shrink per breakpoint, weights 100–900, radii sm–full, spacings xs–xl3 and shadows xs–xl2 — and no font family. Retune those values to the direction you decided rather than working around them, and add a variable only for a role the theme lacks. When the user later asks for a different look, change the variables, not the components.
On a web client a font size, radius or spacing variable holds one value per breakpoint (`{desktop, tablet, phone}`). When one has to differ on phone — an h1 at 28 on desktop and 24 on phone — set that on the variable, not as a breakpoint override of fontSize, padding, gap or borderRadius on the components that reference it: an override replaces the reference at that width, so the component stops following the theme there.

### The Palette Is Named Before It Is Spent
Give a new colour variable a palette level (the theme plugin explains which), not a hex, so it recolours with the rest. A working set is roughly eight roles, named by ROLE, not by colour — "surface", not "light-grey":
- Four neutrals, not two: the page ground, a surface lifted off it, a border a step darker again, and a hairline/divider weaker than the border. Two neutrals is the single most common cause of a flat screen — with only "background" and "border" available, every card, input, header and divider ends up drawn with the same one, and nothing separates from anything.
- Two inks: primary for content, muted for secondary text and labels. Do NOT make muted text by dropping `opacity` on a TEXT — opacity fades the whole component including anything it contains, and it composites against whatever is behind it, so the same "muted" reads differently on a card than on the page. Muted is a colour.
- One accent, and one status colour (danger) if the app writes data.
Spend the accent only on the primary action and the active/selected state. The moment it also appears on a heading, an icon and a border, nothing on the screen is being pointed at.

### A Text Role Is Three Values Sent Together
The theme has font sizes and font weights but no line height and no combined text style, so a role — page title, section heading, body, label, caption, rarely more than five — is a pairing you keep yourself: a size variable, a weight variable and a lineHeight, sent together every time. Asking for a heading means all three; a heading size at body weight is two roles blended. lineHeight is absolute px and does not follow the size: roughly 1.2× the size for headings, 1.5× for body copy, and without it anything above the default crops or overlaps. The display sizes shrink on phone while the lineHeight you sent at the main breakpoint does not, so give a heading role a phone lineHeight as well. For the same reason, changing a font size variable does not move the lineHeight of the text that uses it: in the same turn, re-send lineHeight (and the phone lineHeight) on every TEXT and BUTTON of that role, or the larger text crops and overlaps.

### Typography Carries Most Of The Impression
The type is the largest coloured area on almost every screen, so it decides the character of the page before any accent does.
- Typeface is a real choice: the default (DM Sans on web, pingFang SC on mobile) is the face every other project ships with. Add it as a FONT_FAMILIES variable and reference it as fontFamily on every TEXT, BUTTON and TEXT_INPUT — then switching the face later is one edit. `GET_THEME_INFO` lists the families this client offers under offeredFontFamilies, and only those have font files; any other name renders as the system fallback. One family is a complete answer; two only when clearly distinct in role (a display face and a text face), never two sans faces that differ slightly. If the copy is Chinese, the BODY face has to be a CJK one, not just the headline: a Latin family carries no Chinese glyphs, so every Chinese character falls back to whatever the device has. A display face carries a headline and nothing longer.
- Hierarchy comes from weight and space as much as from size. Three text roles that differ only in size read as one undifferentiated block; a heading at 600–700 against body at 400 separates at a glance even when the sizes are close.
- `letterSpacing` is in PIXELS, not em, and goes NEGATIVE on large display type — roughly -0.5px at a 28–32px page title, -1px or beyond as the size grows — because a face designed at body size is too loose when set large. Body text wants 0. A fractional value like -0.02 changes nothing visible, so sending one and then reporting the tracking as tightened is a false claim. It goes slightly POSITIVE only on small uppercase labels, which are themselves worth avoiding (see below).
- Set `maxWidth` on any TEXT holding a paragraph. Unbounded body text stretches to the container and a 1200px-wide line is unreadable regardless of how well it is styled; hold it near 60–70 characters, which at body size is roughly 600–680px. A capped column keeps hugging the left edge until its PARENT centres it — in a column parent (a PAGE's default) that is `alignItems: 'center'`. This is the most-skipped and most-visible typographic fix in a wide desktop layout.
- Body text needs `textWrap: true` — false pins it to one line and ellipsises it at the box edge however wide the sentence is. `lineClamp` on top when it must stop after N lines.

### Depth Is One Decision, Applied Once
The failure mode is shadow everywhere rather than shadow used well.
- A card lifts off the page with EITHER a hairline border OR a soft shadow, not both. Pick one for the whole app.
- Elevation is at most two levels — resting surface, and the raised thing (modal, dropdown, the one hero card) — so reference two of the theme's shadow variables and no more. A third level means the second is not doing its job.
- A real shadow is large, soft and nearly transparent, and offset downward only — light comes from above: `blur` well above `offsetY`, a colour at low alpha. If the theme's shadows do not read that way, retune those variables rather than drawing a shadow on one component.
- `inset: true` is for the pressed state of a button and for a well an input sits in. It is the only correct way to make something look recessed rather than raised.
- `backdropBlur` belongs on overlay chrome that sits over content — a sticky header, a modal scrim. On an ordinary card it costs paint and buys nothing, since there is nothing behind it to blur.
- `borderRadiusCorners` exists for the cases a single radius cannot express: a segmented control, a sheet rounded only at the top, a cell at the end of a row. Reach for it there and nowhere else.
- Radius is hierarchy. If the card, the input, the badge and the button all carry the same radius variable, the screen has told the user nothing about what is what.

### Space Is The Cheapest Thing That Reads As Finished
The common failure is far too little of it, and it is not a matter of taste — the numbers are knowable. On desktop, section padding starts at 64px vertical and goes up, not down; card padding at 20–24px; the page gutter at least two spacing steps above the in-card gap, so a card reads as sitting on the page rather than pasted onto it. When a screen feels cramped, the fix is almost always more space AROUND the groups, not less space inside them. Proximity is the actual signal: things that belong together sit closer to each other than to anything else, and if a label is equidistant from its own field and the next one, the layout has lost the reader regardless of how the label is styled.

### The Look That Reads As Generated
None of these is forbidden — each is right for some brief — but they are what gets reached for when nothing has been decided, so they appear regardless of subject. Where the user asked for one, give it to them. Where they left the axis free, spend the freedom elsewhere:
- Cream page (near #F4F1EA) with a high-contrast serif display and a terracotta/clay accent (near #D97757).
- Near-black page with one acid-green or vermilion accent.
- Every surface on the same radius under the same soft grey shadow, whatever its role.
- A tracked-out ALL-CAPS eyebrow label above every heading.
- Meta strings joined with middle dots ('A · B · C'); '→' appended to button text.
- One word in a heading coloured or bolded to add interest.
- Numbered markers (01 / 02 / 03) on content that is not actually a sequence.
- A gradient used as decoration on a surface that has no reason to be gradient. A gradient is legitimate as the subject of a hero, or to keep text legible over an image; as a wash behind ordinary cards it is the clearest single tell.

### Every Interactive Thing Answers The Cursor
`UPDATE_COMPONENT_STYLE` takes `variant: 'hover'`, and a build that never sends it is a build where nothing on screen responds to the mouse. That, more than any colour choice, is what makes a page feel like a mockup rather than an application. A hover state on buttons, clickable cards and list rows — a surface a step lighter or darker is enough — costs one extra call per component type, not per component, because the same few shapes repeat. A LAYOUT_VIEW with an ON_CLICK handler still shows an arrow cursor until it gets `cursor: 'pointer'`, so the user has no way to know it is a target. Where a SELECT_VIEW or a TAB_VIEW has a selected state, style it. The default selected state is not a design decision, and "the active tab looks like the inactive ones" is a bug the user will report in different words.

### Three Things Can Be Conditional — Pick The Smallest One
When a screen has two modes, do NOT reach for a conditional container first. Ask what actually differs between the modes, and make only that conditional:
- Conditional DATA (`CREATE_CONDITIONAL_BINDING`) when only VALUES differ — a label, a title, a placeholder, a colour. One component, one conditional binding whose branches are keyed off the deciding value. This is the default answer.
- Conditional ACTION when only BEHAVIOR differs — one button that submits, signs in or signs up depending on the mode. One button, one handler that branches.
- Conditional VIEW (CONDITIONAL_VIEW) only when the STRUCTURE differs — different fields, different counts of children. Branches do NOT share children, so anything you put in a branch is duplicated in every other branch. If the two modes share a form, keep the shared form OUTSIDE the CONDITIONAL_VIEW and put only the differing part inside it. Its branch conditions start always-true, so the first branch shows until you narrow them; make the mode a page variable the conditions read and change it with SET_VARIABLE_DATA.
A "switchable signup / login" is therefore conditional data plus a conditional action over ONE shared form — not two branches each holding a copy of the same fields. Wrapping shared UI in branches is the classic wrong answer: it looks right and silently doubles the tree.
TAB_VIEW is a different question entirely: use it only when the tab strip itself is the affordance the user should see and click. It renders tab chrome you cannot remove.
Both TAB_VIEW and CONDITIONAL_VIEW take `preserveStateOnSwitch`, and it is the same mechanism in both: true keeps every case alive, so a scrolled list, a half-filled input or a loaded query survives switching away and back; false tears the inactive case down and rebuilds it from scratch. Both are created with it ON. Turn it off only when stale state would be wrong (a form that must reset), and remember the cost: every case stays mounted.

### Components That Display a Field of Their Data
DATA_SELECTOR, SELECT_VIEW and LIST name one field of their data source: `displayDataField` (the value the user sees) and `cellKeyDataField` (the row identity). Binding the dataSource is not enough — a DATA_SELECTOR or SELECT_VIEW whose data source resolves to a list of objects is invalid until its displayDataField is set. Set it with `UPDATE_COMPONENT_PROPERTIES`, passing the field's plain NAME as a string; the full field reference is resolved for you, so never hand-assemble one. The candidates appear as selectableDataFields in `GET_COMPONENT_INFO` once the dataSource is bound — so the order is always: create → bind dataSource → read the component → name the field.

### Page Data: Queries, Variables & Inputs
Pages have NO separate query list — a page query IS a read-only page variable whose dataSource is a database query. It is for data more than one component reads, or a filter the page owns centrally; a single list showing one filtered table binds that table itself (see Data Bindings below) and needs nothing here:
- `ADD_COMPONENT_QUERY` attaches a table query to a PAGE or MODAL (limit 1 = single row, >1 = list). Components consume it through data bindings (bindings plugin) at the echoed dataSourceSchemaPath; narrow filters and sorting with the request-filter tools at that same path. The result also reports rolesWithoutSelect: those roles see no rows until they are granted SELECT with the permission plugin's table-permission tool. Tune with `UPDATE_COMPONENT_QUERY` — which also sets or clears the row limit of the query a LIST, SELECT_VIEW, DATA_SELECTOR or SHEET owns; remove queries and variables alike with `DELETE_COMPONENT_VARIABLES`.
- Page-local mutable state: `ADD_COMPONENT_VARIABLES`, with each typeIdentifier picked from `GET_COMPONENT_VARIABLE_SELECTABLE_TYPES` — never hand-assembled. A new variable starts with no value, so give it an initial one at the `defaultValueSchemaPath` the result echoes whenever anything reads it: conditional cases, branch conditions and SET_VARIABLE_DATA values all test the variable against specific values, and none of them match an empty variable. The page then loads with every dependent case unmatched — blank labels, the fallback branch showing — and a toggle whose new value is itself conditional on the old one cannot recover, because it has nothing to match either.
- Page parameters: `ADD_COMPONENT_INPUTS` / `UPDATE_COMPONENT_INPUT` / `DELETE_COMPONENT_INPUTS`. WEB pages take URL query/path params; WECHAT/MOBILE pages and MODALs take general inputs. Navigation and show-modal call sites pass one value per input.
- `SET_INITIAL_PAGE_ID` sets the app's home page for the current client.

### Data Bindings on Components
`GET_COMPONENT_BINDABLE_PROPERTIES` lists every place a component can be bound — always start there and copy the paths verbatim, never hand-assemble a binding path. It returns two sections: `properties`, the component's own slots, each with its ready-to-use schemaPath, expected type, current binding and scope; and `actionSlots`, the parameter slots of the actions already wired to its events, chained success/failure branches included. `ADD_COMPONENT_ACTIONS` echoes an action's slots when it creates it — bind them in the same turn when you can, and use `actionSlots` to find them again in any later turn. Fill the echoed paths with the bindings plugin's tools: `BROWSE_DATA_BINDING_OPTIONS` + `CREATE_OPTION_BINDING` for dataSource slots and other dynamic values, `CREATE_CONST_BINDING` for literals. A LIST, SELECT_VIEW or DATA_SELECTOR can take its dataSource either way, and the options tree offers both in one answer: the tables under 'Table', and any page query under 'Context > Current page > Data source'. **Prefer the table** — a component that owns its data source is filtered, sorted and retyped in one place, and `GET_REQUEST_FILTER_CONTEXT` narrows it at the component's own path. Reach for a page query (`ADD_COMPONENT_QUERY` first) only when the SAME result set is genuinely shared — two components reading one query, or a filter the page tunes centrally. One component, one filtered list is not a sharing case: adding a page variable there buys nothing and leaves a second thing to keep in step. Inside a list cell (scope "list-cell:<listId>") the options tree additionally offers the current list item's fields under 'item-data' — bind cell children to those instead of re-querying the table.

### Editing Protocol
- One read of a container covers both adding children to it and editing its own configuration, so `GET_COMPONENT_INFO` and `GET_CONTAINER_CHILDREN_INFO` on the same component is one call wasted. After a conflict, re-read and retry.
- Wiring behavior: `GET_COMPONENT_SELECTABLE_EVENTS`, then `GET_SELECTABLE_COMPONENT_ACTIONS` for what that event accepts, then `ADD_COMPONENT_ACTIONS`. Anything a specific kind returns beyond its bindableSchemaPaths arrives under that handler's `details`.
- Pick the kind from `GET_SELECTABLE_COMPONENT_ACTIONS` and take its parameter shape from `ADD_COMPONENT_ACTIONS`'s parameter schema (fetch it first if it was left out as "schema on request"), which is the only complete list of the kinds and their fields — the editor offers far more than the handful named below, including SHOW_TOAST, OPEN_EXTERNAL_LINK, SET_CLIPBOARD, DOWNLOAD_FILE, CALL_PHONE, the payment kinds and the animation kinds. Do not conclude an interaction is impossible because it is not discussed here; the selectable list and the schema decide that. What follows is only the judgment the schema cannot carry.
- ALERT_DIALOG is the runtime's own confirm box — a title, a message and two buttons it draws itself, with no MODAL to build or lay out. Use it for "are you sure?"; use SHOW_MODAL when the dialog needs your own components in it. Chain what confirming does onto ALERT_DIALOG's ON_CONFIRM branch, or the buttons do nothing.
- Building a MODAL does not make it reachable. Without a SHOW_MODAL action pointing at it, nothing opens it, and the modal you just laid out is dead UI the user cannot reach.
- An action that produces a RESULT — GET_LOCATION, UPLOAD_FILE, GENERATE_QR_CODE and the other result-bearing kinds — can only keep it through assignToVariable, a writable page variable created with `ADD_COMPONENT_VARIABLES` on the page the component sits on. Their success branches cannot see the result, so an action without assignToVariable runs and throws its answer away. Create the variable first.
- SIGN_UP and USER_LOGIN are separate actions, and the password slot on a SIGN_UP decides what the account can sign in with later: leave it unbound and the account has no password, so a USER_LOGIN with credentialType PASSWORD can never succeed for anyone who signed up there. Build the pair together, or sign in with VERIFICATION_CODE.
- A WeChat mini-program signs its user in by itself. The client is created with a silent WeChat login in its own app-did-load configuration, which the editor will not let anyone delete, so whoever opens the app is already signed in, and the account is created there too. Do NOT build a sign-in or sign-up screen there and do NOT wire a login action: USER_LOGIN offers no WeChat mode at all, the silent login belongs to that configuration alone, and SIGN_IN / SIGN_UP / SIGN_OUT are absent from the mini-program's selectable actions. When the app needs the user's phone number, use OBTAIN_PHONE_NUMBER.
- MUTATION writes the database without an action flow. rolesWithoutPermission in its result is report-only — grant the table permission with the permission plugin or the write is denied at runtime, which looks like a silent no-op to the user. rowScope CURRENT_ITEM is valid only inside a LIST cell; refreshOnSuccess is preferable to a separate REFRESH.
- CONDITIONAL is how ONE event does different things in different cases: branches in evaluation order, first match wins, each branch narrowed at the echoed conditionSchemaPath and filled by a further `ADD_COMPONENT_ACTIONS` carrying that branch's conditionalBranchId. Leave the last branch always-true as the else. Without it every action on the event fires on every click — ON_SUCCESS / ON_FAILURE branch on whether a call worked, never on a data condition. Do NOT reach for an action flow to get branching: a flow's BRANCH node runs on the server and cannot navigate, toast or open a modal, so frontend branching belongs here. This decides what HAPPENS; a CONDITIONAL_VIEW decides what is DISPLAYED.
- There is no per-component conditional visibility — an ordinary component has no data-driven show/hide flag. To make content appear only when a condition holds, wrap it in a CONDITIONAL_VIEW container and gate the branch with the bindings plugin's condition tools.

### Motion Is One Moment, Not A Coat Of Paint
Zion has animation as ACTIONS, not style fields, so they are attached with `ADD_COMPONENT_ACTIONS` like any other handler: ANIMATION_FADE_EFFECT and ANIMATION_SLIDE_EFFECT (APPEAR / DISAPPEAR, eight directions, a distance defaulting to 150px), ANIMATION_SCALE_EFFECT, ANIMATION_COMMON_EMPHASIS (BOUNCE / PULSE / BLINK), ANIMATION_FLIP_EMPHASIS, and ANIMATION_SCROLLING_INTERACTION. Each takes delay, and the emphasis kinds take repeat or loop.
The rules are hard ones, and breaking them shows up as a project error rather than a subtle defect: exactly ONE animation per event, and only on onClick, onHover or onScrollIntoPage — the runtime reads the last effect it finds on the event and ignores any attached elsewhere. ANIMATION_SCROLLING_INTERACTION is the exception and goes ONLY on onPageScroll, which carries nothing else at all, because the curve is driven by scroll distance. None of them take attachTo and none take bindings, so they cannot be chained onto a success branch.
Spend motion the way you spend the accent. Movement that answers what a person just did — a press, an open, a confirm — is worth it, because it shows what changed. Movement that merely announces that content exists is the generated-page tell: a fade-and-slide on every section as it scrolls into view reads as a template, and it reads that way precisely because it is the first thing reached for. One orchestrated moment on the screen that matters beats an effect on every card, and a screen with no animation at all is a perfectly good answer — far better than one where everything twitches. The emphasis kinds earn their place on something genuinely needing attention, never as decoration, and BLINK almost never earns it.

### Reading A Screenshot
Only the editor's canvas shows what the user will see, and canvas screenshots are not reachable from here: the user has to open the screen in the editor. `schema validate` knows only about invalid schema. What it shows is worth reading against: one clear focal point rather than everything at the same weight; related things closer together than unrelated ones; at least three levels of text that differ in weight, not only size; gutters where things would otherwise touch an edge, and lines under ~70 characters; the accent only on the primary action and the active state; something that responds to hover; colour, spacing, radius, type and shadow values that are theme references, so a restyle reaches them. Report only what you actually changed. Every claim about appearance — a heading looks larger, a selected state looks different, spacing was tightened — must correspond to an `UPDATE_COMPONENT_STYLE` (or creation-time style) call you made in this session. If a requested look was not achievable with the available tools, say which part you skipped and why; do not describe the component's built-in default styling as if you had applied it. A user who is told the work is done stops checking.

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
| List pages and modals | `GET_ALL_ROOTS_INFO` | — |
| Component info | `GET_COMPONENT_INFO` | `componentId` or `schemaPath` |
| Style keys a component type accepts | `GET_COMPONENT_TYPE_CAPABILITIES` | `componentTypes` |
| Data/vars in scope at a component | `GET_COMPONENT_CONTEXT_INFO` | `componentId` |
| Types a page variable or page input accepts | `GET_COMPONENT_VARIABLE_SELECTABLE_TYPES` | — |
| Container children info | `GET_CONTAINER_CHILDREN_INFO` | `componentId` |
| Find components by display name | `FIND_COMPONENTS` | `query` |
| Remove event handlers | `DELETE_COMPONENT_EVENT_HANDLERS` | `componentId`, `handlerIds`, `eventType` or `scheduledJobId` |
| Add TAB_VIEW tabs | `ADD_TAB_VIEW_TABS` | `componentId`, `items` |
| Delete TAB_VIEW tabs | `DELETE_TAB_VIEW_TABS` | `componentId`, `tabChildComponentIds` |
| Reorder TAB_VIEW tabs | `REORDER_TAB_VIEW_TABS` | `componentId`, `orderedTabChildComponentIds` |
| Switch a TAB_VIEW between built-in and custom tabs | `SET_TAB_VIEW_TAB_MODE` | `componentId`, `mode` |
| Add CONDITIONAL_VIEW branches | `ADD_CONDITIONAL_VIEW_BRANCHES` | `componentId`, `items` |
| Delete CONDITIONAL_VIEW branches | `DELETE_CONDITIONAL_VIEW_BRANCHES` | `branchChildComponentIds`, `componentId` |
| Reorder CONDITIONAL_VIEW branches | `REORDER_CONDITIONAL_VIEW_BRANCHES` | `componentId`, `orderedBranchChildComponentIds` |
| Fill an optional slot (LIST header/footer, MAP marker) | `SET_COMPONENT_SLOT` | `componentId`, `slotField` |
| Empty an optional slot | `CLEAR_COMPONENT_SLOT` | `componentId`, `slotField` |
| Point a SHEET at a table | `SET_SHEET_DATA_SOURCE` | `componentId`, `tableDisplayName` |
| Add SHEET columns | `ADD_SHEET_COLUMNS` | `componentId`, `items` |
| Remove SHEET columns | `DELETE_SHEET_COLUMNS` | `componentId`, `dataFieldNames` |
| Reorder SHEET columns | `REORDER_SHEET_COLUMNS` | `componentId`, `orderedDataFieldNames` |
| Create page timer jobs | `ADD_COMPONENT_SCHEDULED_JOBS` | `componentId`, `items` |
| Change a timer job | `UPDATE_COMPONENT_SCHEDULED_JOB` | `componentId`, `scheduledJobId` |
| Delete timer jobs | `DELETE_COMPONENT_SCHEDULED_JOBS` | `componentId`, `scheduledJobIds` |
| Native tab bar: state, colours, slots | `GET_TAB_BAR_INFO` | — |
| Show or hide the native tab bar | `SET_TAB_BAR_ENABLED` | `enabled` |
| Point a tab-bar slot at a page | `UPDATE_TAB_BAR_ITEMS` | `items` |
| Colour the tab bar | `UPDATE_TAB_BAR_STYLE` | — |
| Client variables: state, values, value paths | `GET_CLIENT_VARIABLES_INFO` | — |
| Types a client variable accepts | `GET_CLIENT_VARIABLE_SELECTABLE_TYPES` | — |
| Declare client variables | `ADD_CLIENT_VARIABLES` | `items` |
| Rename a client variable or change its type | `UPDATE_CLIENT_VARIABLES` | `items` |
| Remove client variables | `DELETE_CLIENT_VARIABLES` | `variableKeys` |
| Duplicate components with their subtrees | `DUPLICATE_COMPONENTS` | `componentIds` |
| Append one event's action handlers to another | `DUPLICATE_COMPONENT_ACTIONS` | `componentId`, `eventType`, `sourceComponentId` |
| Verification-code channels: list | `GET_ALL_SEND_VERIFICATION_CODE_CONFIGS_INFO` | — |
| Verification-code channel detail (+ input paths) | `GET_SEND_VERIFICATION_CODE_CONFIG_DETAIL` | `configId` |
| Create a verification-code channel | `ADD_SEND_VERIFICATION_CODE_CONFIGS` | `items` |
| Update a verification-code channel | `UPDATE_SEND_VERIFICATION_CODE_CONFIG` | `configId` |
| Delete verification-code channels | `DELETE_SEND_VERIFICATION_CODE_CONFIGS` | `configIds` |

Component writes go through the CLI like any other edit: `GET_COMPONENT_TEMPLATE` then `ADD_COMPONENT` to build, `UPDATE_COMPONENT_STYLE` to restyle, `MOVE_COMPONENTS` to restructure, `ADD_COMPONENT_ACTIONS` to wire events, `DELETE_COMPONENTS` to remove — see "Building & Editing the Component Tree" above. One limit is real: you cannot see what you built — the editor canvas and error center are not reachable from here, so a screen can be schema-valid and still look wrong. Verify structurally with `GET_CONTAINER_CHILDREN_INFO` and `GET_INCOMPLETE_STRUCTURES` (its `responsiveRisks` catches the desktop-width-on-phone case), and where appearance is the acceptance criterion, prefer handing the user steps over guessing.

The preset tab-bar icon library is reachable: `npx -y zion-mcp@2.8.2 component tab-bar-icons` lists every icon this deployment ships as a name and the media exId `UPDATE_TAB_BAR_ITEMS` takes. A shown tab needs an icon for both its normal and its selected state or the project reports an error, and the exIds are hashids of media rows that cannot be derived from an icon's name — a value that could not be an exId is refused by the write rather than failing the mini-program build over a file it cannot find, so take them from that listing and never invent one. An icon the library does not ship still has to be brought in through the editor (`web-assets.md`) or picked there by the user: nothing here imports media.

Then ship:

```bash
npx -y zion-mcp@2.8.2 schema validate && npx -y zion-mcp@2.8.2 project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
