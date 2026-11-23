# UI components & layout

## Component Domain Knowledge
The UI is a tree of components organized into pages.

### Identity & Tree Rules
Every component has an immutable id and a displayName. Reads return both; write tools verify the {id, displayName} pair and reject a mismatch — re-read, then retry. A container's children order IS the visual order. A component can never be moved into its own subtree.

### Categories
Layout (container — holds children via `children`):
- PAGE: root of each page; created without a parent and registered as a new page.
- LAYOUT_VIEW: generic Flexbox container; the most common container.
- MODAL: popup/dialog container; created without a parent, opened/closed via events.

Special (complex — children live in built-in slots seeded at creation and mutated only via the slot tools, never via ADD_COMPONENT / MOVE_COMPONENTS):
- LIST: repeating list; its "cell" slot renders once per data row (optional header/footer).
- TAB_VIEW: tabbed interface; one child view per tab in tabList.
- SELECT_VIEW: built-in normal/selected state views.
- CONDITIONAL_VIEW: one built-in view per condition branch.
- MAP: optional "marker" slot taking a MAP_MARKER.

Everything else is a leaf that cannot have children; MAP_MARKER only goes inside MAP. `GET_SELECTABLE_COMPONENT_TYPES` returns the full creatable catalog with each type's slot fields.

### Hierarchy Rules
Every component except PAGE and MODAL must have a Layout Component as its parent. Prefer relative positioning; use absolute/fixed only when truly necessary.

### Layout
Zion uses Flexbox. Layout containers configure: direction (row/column), justify-content (start, space-between, space-evenly, space-around), gap, wrap, overflow (scroll/visible/hidden).

### Responsive Breakpoints
Phone: 0–767 px (375 px panel) | Tablet: 768–1279 px (820 px panel) | Desktop: 1280–1920 px (1440 px panel). Desktop is the primary breakpoint and the default for every read and write. A narrower breakpoint stores only its differences from desktop, so a field it does not override renders the desktop value there — an app built without touching phone is not unstyled on phone, it is desktop-sized, which is how a 600 px card runs off a 375 px screen. The schema and the error center do not show this; a read with `breakpoint` or a phone capture does, so look at phone when you build or restyle a layout the user will open on one. Read what phone resolves to by passing `breakpoint` to `GET_COMPONENT_INFO` and `GET_CONTAINER_CHILDREN_INFO`; the read names which fields that breakpoint overrides and which it inherits. Fix one by resending that field with the `breakpoint` param to `UPDATE_COMPONENT_STYLE` — usually flexDirection 'column' on rows, and widths in % or fill rather than px; a phone `component capture-canvas` shows the result. Resending a value unchanged at a breakpoint pins it there and stops it following desktop, so override only what has to differ.

### Reading the Component Tree
Ids are opaque: copy them verbatim from these read tools and never modify them. A page id (from `GET_ALL_ROOTS_INFO`) is not a component id, and breakpoint keys (phone/tablet/desktop) are tool parameters, never part of an id — passing `<pageId>-desktop` as a target fails with "target component not found".

### Building & Editing the Component Tree
Construction workflow: read the target container, pick types with `GET_SELECTABLE_COMPONENT_TYPES`, edit `GET_COMPONENT_TEMPLATE` output and submit it via `ADD_COMPONENT`, then use the ids it echoes instead of re-reading.
Container sizing: a LAYOUT_VIEW hugs its content, so a container created WITH its children inside is sized by them and needs no height of its own; an empty one keeps a small minimum height so it stays visible and droppable, and filling it makes that minimum irrelevant. Give a container an explicit height only for a fixed-size box or a scroll viewport — a fixed height its content outgrows clips that content invisibly, and a fixed width crops text mid-word. `UPDATE_COMPONENT_STYLE` merges, so a size you do not name keeps its old value.
Width/height `fill` means "grow along the flex main axis", so it only makes sense on the axis the parent lays out along: `width: fill` in a row, `height: fill` in a column. On the other axis it would unset the size and let the child shrink to its content, so it is applied as '100%' instead and the result says so. There is no `stretch` alignment to fall back on — in Zion a component declares its own size rather than having its container negotiate one.
A row of cards is sized one axis at a time. Along the row, give every child `fill`, or fr shares — `65fr` and `35fr` — when the split is deliberate; never percentages, which claim the whole width and let the gap overflow it. Across the row, give every child the SAME `minHeight` so they bottom out together: a real `height` would clip the card whose content grows past it, and with no `stretch` a shared floor is as close to equal-height cards as this system gets; pick a floor that clears the tallest copy you expect. Set `minWidth` on a child only where that column has a genuine floor: it, not `flexWrap`, is what makes a row break into 2+1 once the container is narrower than the floors plus the gaps. Size the children this way and the container needs nothing beyond `flexDirection`, `gap` and its own width — `justifyContent` and `alignItems` have no free space left to distribute, so reaching for them to fix a ragged row means the children are the wrong size.
A sentence clipped horizontally is nearly always `textWrap` false (see Typography below): widening the box, or setting `overflowX`, does not make the text reflow.
A TEXT_INPUT collecting a password needs `password: true` via `UPDATE_COMPONENT_PROPERTIES`; a "Password" placeholder alone does not mask anything.

### Bottom Navigation Is The Client's Tab Bar
On a mini-program or mobile client, the row of tabs along the bottom — Home / Categories / Cart / Me — is the client's native tab bar: a client setting the runtime draws itself, not components on a page. Whenever such an app has a few top-level sections the user moves between, that is a tab bar, whether or not the request says "tab bar" or "bottom navigation". Build it with one top-level PAGE per section, then `SET_TAB_BAR_ENABLED` and `UPDATE_TAB_BAR_ITEMS` in the same batch to point a slot at each page, icons from `component tab-bar-icons` — two to five tabs, one page per tab. Read `GET_TAB_BAR_INFO` first when the client may already have one. Do NOT build the bar out of components — not a LAYOUT_VIEW pinned to the bottom of each page, not a TAB_VIEW holding every section on one page: a component copy repeats on every page and drifts apart, and it is not the bar a mini-program user expects. Leave the bottom of each tab page free for the bar. Build one from components only when the user explicitly asks for a custom bar the native one cannot draw. A web client has no native tab bar; its navigation is components.

### The Style Is Settled Before The First Component
A client's look is written down in its DESIGN.md — `/project/DESIGN.md`, or `/project/app/<appExId>/DESIGN.md` in a multi-client project — and carried by the theme. When it exists, follow it; if its values disagree with the theme and the user has not said which wins, ask. A client without one gets its look when its first screen is about to be built, not as a question beforehand: the direction the user gave (words, a screenshot, a design, HTML), else the look its existing screens already have, else a recommended one for this kind of app — then the DESIGN.md and the theme are written, and the reply names the look used and what else is on offer, since the look is the user's to change. The platform's creation defaults are what produce the grey-box build, so a request that names no visual direction still gets one. Once settled, the style holds on every screen: a stated direction on two screens and another on the rest reads as a bug rather than a choice.

### The Theme Is The Design System
Every style value that has a theme category is sent as that variable's referenceValue, `$$<id>`, never as a literal: colours (backgroundColor, textColor, borderColor, gradient stops, shadow.color) from colour variables, padding/margin/gap from spacings, borderRadius from radii, fontSize, fontWeight and fontFamily from their own categories, and a whole shadow as `shadow: {variable: '$$<id>'}`. The id resolves against the field's category, so `$$md` under padding is the spacing md and under borderRadius the radius md. A literal renders the same today but is pinned to the component it was written on; a reference follows the theme. That is the point: "make it rounder" or "a different font" is then a few `UPDATE_THEME_VARIABLES` edits every screen follows, instead of a sweep that touches every component and still misses some. Only lineHeight, letterSpacing, borderWidth, opacity and sizes have no category and stay literal.
That is the default, not a rule over the user:
- A value the user gives for one component ("make this button 13px", "this banner #ff6600") is written as that literal on that component — not rounded to a variable, not turned into a new variable, and not applied by retuning a shared one. Ask when it is unclear whether they mean this component or the whole app.
- Change only what the request touches; existing literals stay literal unless the user asks for the restyle.
- Add a variable only for a value several components share, never for a one-off.
`GET_THEME_INFO` shows what the theme holds: colour roles (Primary, Secondary, Tertiary, Text_primary, Text_secondary, Background, Surface, Border, Success, Alert, Error), font sizes h1 through caption that already shrink per breakpoint, weights 100–900, radii sm–full, spacings xs–xl3 and shadows xs–xl2 — and no font family. Retune those to the style DESIGN.md records rather than working around them, and add a variable only for a role the theme lacks.
On a web client a font size, radius or spacing variable holds one value per breakpoint (`{desktop, tablet, phone}`). When one must differ on phone — an h1 at 28 desktop, 24 phone — set that on the variable, not as a breakpoint override of fontSize, padding, gap or borderRadius on the components that reference it: an override replaces the reference, so the component stops following the theme there.

### The Palette Is Named Before It Is Spent
Give a new colour variable a palette level (the theme plugin explains which), not a hex, so it recolours with the rest. A working set is roughly eight roles, named by ROLE — "surface", not "light-grey":
- Four neutrals: the page ground, a surface lifted off it, a border a step darker again, and a hairline weaker than the border. Two neutrals is the single most common cause of a flat screen: every card, input and divider drawn with the same one.
- Two inks: primary for content, muted for secondary text and labels. Do NOT make muted text by dropping `opacity` on a TEXT — opacity fades the whole component and composites against what is behind it, so the same "muted" reads differently on a card than on the page. Muted is a colour.
- One accent, and one status colour (danger) if the app writes data.
Spend the accent only on the primary action and the active/selected state; the moment it also appears on a heading, an icon and a border, nothing is being pointed at.

### A Text Role Is Three Values Sent Together
The theme has font sizes and weights but no line height and no combined text style, so a role — page title, section heading, body, label, caption, rarely more than five — is a pairing you keep yourself: a size variable, a weight variable and a lineHeight, sent together every time. A heading size at body weight is two roles blended. lineHeight is absolute px: a fontSize sent without it rewrites it to 1.2× the size rounded up (the result's note gives the px), which suits a heading, so send body copy's lineHeight, nearer 1.5×, with its size. Display sizes shrink on phone while the lineHeight sent at the main breakpoint does not, so give a heading role a phone lineHeight too. For the same reason, changing a font size variable does not move the lineHeight of the text using it: in the same turn, re-send lineHeight (and the phone lineHeight) on every TEXT and BUTTON of that role, or the larger text crops and overlaps.

### Typography Carries Most Of The Impression
Type is the largest coloured area on almost every screen, so it decides the character of the page before any accent does.
- Typeface is a real choice: the default (DM Sans on web, pingFang SC on mobile) is the face every other project ships with. Choose by the type mood DESIGN.md records: reference the theme's FONT_FAMILIES variable when one fits, otherwise add one, and reference it as fontFamily on every TEXT, BUTTON and TEXT_INPUT — switching the face later is then one edit. `GET_THEME_INFO` lists the families this client offers under offeredFontFamilies, and only those have font files; any other name falls back to the system font. One family is a complete answer; two only when clearly distinct in role (a display face and a text face). Chinese copy needs a CJK BODY face, not just a CJK headline: a Latin family carries no Chinese glyphs, so every Chinese character falls back to the device font.
- Hierarchy comes from weight and space as much as size: three roles that differ only in size read as one undifferentiated block, while a heading at 600–700 against body at 400 separates at a glance.
- `letterSpacing` is in PIXELS, not em, and goes NEGATIVE on large display type — roughly -0.5px at a 28–32px page title, -1px or beyond as the size grows — because a face designed at body size is too loose when set large. Body text wants 0. A fractional -0.02 changes nothing visible, so reporting the tracking as tightened after sending one is a false claim. It goes slightly POSITIVE only on small uppercase labels.
- On a web client, set `maxWidth` on any TEXT holding a paragraph: unbounded body text stretches to the container and a 1200px line is unreadable however well styled. Hold it near 60–70 characters, roughly 600–680px at body size. A capped column hugs the left edge until its PARENT centres it — in a column parent (a PAGE's default) that is `alignItems: 'center'`.
- Body text needs `textWrap: true` — false pins it to one line and ellipsises it at the box edge however wide the sentence is. Add `lineClamp` when it must stop after N lines.

### Depth Is One Decision, Applied Once
- A card lifts off the page with EITHER a hairline border OR a soft shadow, not both. Pick one for the whole app.
- Elevation is at most two levels — resting surface, and the raised thing (modal, dropdown, the one hero card) — so reference two of the theme's shadow variables and no more. A third level means the second is not doing its job.
- A real shadow is large, soft and nearly transparent, offset downward only — light comes from above: `blur` well above `offsetY`, a colour at low alpha.
- `inset: true` is for a button's pressed state and for a well an input sits in — the only correct way to make something look recessed.
- `backdropBlur` belongs on overlay chrome over content — a sticky header, a modal scrim. On an ordinary card it costs paint and buys nothing.
- `borderRadiusCorners` is for what a single radius cannot express: a segmented control, a sheet rounded only at the top, a cell at the end of a row. Nowhere else.
- Radius is hierarchy. If the card, the input, the badge and the button all carry the same radius variable, nothing separates them.

### Space Is The Cheapest Thing That Reads As Finished
The common failure is far too little of it. On a web desktop, section padding starts at 64px vertical and goes up; card padding is 20–24px; the page gutter sits at least two spacing steps above the in-card gap, so a card reads as sitting on the page rather than pasted onto it; the phone breakpoint drops sections to 24–32px and cards to 16px. A 375px mini-program or mobile screen keeps 16px side margins and 16px card padding, 8px inside a card, 12–16px between cards and 24–32px between sections — never the desktop 64 — with tap targets at least 44 × 44. When a screen feels cramped, the fix is more space AROUND the groups, not less inside them. Proximity is the signal: things that belong together sit closer to each other than to anything else, and a label equidistant from its own field and the next one has lost the reader.

### Every Interactive Thing Shows It Is One
On a web client `UPDATE_COMPONENT_STYLE` takes `variant: 'hover'`, and a build that never sends it is a build where nothing responds to the mouse. A hover state on buttons, clickable cards and list rows — a surface a step lighter or darker is enough — costs one extra call per component type, not per component. A LAYOUT_VIEW with an ON_CLICK handler still shows an arrow cursor until it gets `cursor: 'pointer'`, so the user cannot tell it is a target. A mini-program or mobile client has no cursor, so send no hover variant there: a tappable thing shows itself by shape — a fill or outline on a button, a trailing chevron on a row that opens something. On every client, style the selected state of a SELECT_VIEW or TAB_VIEW. The default selected state is not a design decision, and "the active tab looks like the inactive ones" is a bug the user will report in different words.

### Three Things Can Be Conditional — Pick The Smallest One
When a screen has two modes, do NOT reach for a conditional container first. Ask what actually differs, and make only that conditional:
- Conditional DATA (`CREATE_CONDITIONAL_BINDING`) when only VALUES differ — a label, a title, a placeholder, a colour. One component, one conditional binding keyed off the deciding value. This is the default answer.
- Conditional ACTION when only BEHAVIOR differs — one button that submits, signs in or signs up depending on the mode. One button, one handler that branches.
- Conditional VIEW (CONDITIONAL_VIEW) only when the STRUCTURE differs — different fields, different counts of children. Branches do NOT share children, so anything in a branch is duplicated in every other branch: keep a shared form OUTSIDE the CONDITIONAL_VIEW and put only the differing part inside. Its branch conditions start always-true, so the first branch shows until you narrow them; make the mode a page variable the conditions read and change it with SET_VARIABLE_DATA.
A switchable signup / login is therefore conditional data plus a conditional action over ONE shared form — not two branches each holding a copy of the same fields. Wrapping shared UI in branches is the classic wrong answer: it looks right and silently doubles the tree.
TAB_VIEW is a different question: use it only when the tab strip itself is the affordance the user should see and click. It renders tab chrome you cannot remove.
Both TAB_VIEW and CONDITIONAL_VIEW take `preserveStateOnSwitch`: true keeps every case alive, so a scrolled list, a half-filled input or a loaded query survives switching away and back; false tears the inactive case down and rebuilds it. Both are created with it ON. Turn it off only when stale state would be wrong (a form that must reset), and remember the cost: every case stays mounted.

### Components That Display a Field of Their Data
DATA_SELECTOR and SELECT_VIEW name the field they display with `displayDataField`. Binding the dataSource is not enough — one whose data source resolves to a list of objects is invalid until its displayDataField is set. Set it with `UPDATE_COMPONENT_PROPERTIES`, passing the field's plain NAME as a string; the full field reference is resolved for you, so never hand-assemble one.
LIST has no display field: its cell child renders each row, and that content binds to the list's current item.
The candidates appear as selectableDataFields in `GET_COMPONENT_INFO` once the dataSource is bound, so the order is always: create → bind dataSource → read the component → name the field.

### Page Data: Queries, Variables & Inputs
Pages have NO separate query list — a page query IS a read-only page variable whose dataSource is a database query. It is for data more than one component reads, or a filter the page owns centrally; a single list showing one filtered table binds that table itself (see Data Bindings below):
- `ADD_COMPONENT_QUERY` attaches a table query to a PAGE or MODAL (limit 1 = single row, >1 = list). Components consume it through data bindings at the echoed dataSourceSchemaPath; narrow filters and sorting with the request-filter tools at that same path. The result also reports rolesWithoutSelect: those roles see no rows until granted SELECT with the permission plugin's table-permission tool. Tune with `UPDATE_COMPONENT_QUERY` — which also sets or clears the row limit of the query a LIST, SELECT_VIEW, DATA_SELECTOR or SHEET owns; remove queries and variables alike with `DELETE_COMPONENT_VARIABLES`.
- When the page has to show changes other users make while it is open (a chat, a live dashboard, an order's status), make the query a subscription: queryType SUBSCRIPTION on `ADD_COMPONENT_QUERY` or `SET_SHEET_DATA_SOURCE`, or switch an existing page query, LIST or SHEET with `UPDATE_COMPONENT_QUERY`. Leave everything else a plain query.
- Page-local mutable state: `ADD_COMPONENT_VARIABLES`, each typeIdentifier picked from `GET_COMPONENT_VARIABLE_SELECTABLE_TYPES` — never hand-assembled. A new variable starts with no value, so give it an initial one at the echoed `defaultValueSchemaPath` whenever anything reads it: conditional cases, branch conditions and SET_VARIABLE_DATA values all test against specific values and none match an empty variable, so the page loads with every dependent case unmatched.
- Page parameters: `ADD_COMPONENT_INPUTS` / `UPDATE_COMPONENT_INPUT` / `DELETE_COMPONENT_INPUTS`. WEB pages take URL query/path params; WECHAT/MOBILE pages and MODALs take general inputs; navigation and show-modal call sites pass one value per input. A URL_QUERY input can set `ignoreWhenEmpty` so an empty value is left out of the link instead of sent blank; both input tools take it, and sending it for any other category is refused.
- `SET_INITIAL_PAGE_ID` sets the app's home page for the current client.

### SEO Path Ranges
A WEB page's URL path parameters are its URL_PATH inputs; under SSR/SSG each needs a value range so the publisher can generate URLs: literal values via `SET_SEO_PATH_STATIC_OPTIONS`, database-backed via `ADD_SEO_PATH_DB_QUERY` (the chosen column's values become the segments; narrow that source's rows with `GET_REQUEST_FILTER_CONTEXT` and the where tools its `whereTrack` names). Start from `GET_SEO_INFO`, then `GET_SEO_PAGE_DETAIL` for one page. Ranges are hidden and ignored under CSR — switch rendering mode first with `SET_SEO_RENDERING_METHOD` (CSR to SSG; SSR is a project setting). Renaming a URL_PATH input migrates its range; deleting it removes the range.

### Global SEO
Rendering mode, sitemap mode and robots.txt are client-wide, not per page. Read them with `GET_SEO_INFO`; switch the mode with `SET_SEO_RENDERING_METHOD` or replace robots.txt with `SET_SEO_ROBOTS_TXT`. The sitemap mode is derived, not set: CSR → AUTO, SSG → BY_PATH_DATA, SSR keeps the current value. SSG needs the project's SEO capability — refused when the capability data reports none, allowed when that data is unavailable; SSR needs a single-tenant server and is always refused here, so ask the user to switch it in the project settings. Leaving SSG clears the pages generated at publish time.

### Page SEO Fields
Per-page SEO lives on the page's `PageSeoConfig`, not the client: TDK (seoTitle / seoDescription / seoKeywords), seoThumbnail (og:image), canonicalUrl, noindex, excludeFromSitemap, structuredData and sitemapLastmod. Static pages carry them too, so this is not path-range work. **Read first** with `GET_SEO_PAGE_DETAIL` — each field's binding kind and value, the structured-data shape, the lastmod source — then set only what is unset, never overwriting a configured field. noindex keeps the page out of search indexes but still reachable; excludeFromSitemap only drops it from the sitemap. A user center, search-result or draft page usually wants both; a crawlable page that should not compete in the sitemap wants excludeFromSitemap alone. canonicalUrl points search engines at the main URL of content reachable at more than one URL. These slots are data-binding leaves, not plain properties: take their schemaPaths from `GET_COMPONENT_BINDABLE_PROPERTIES` and write them with the bindings plugin's `CREATE_*_BINDING` tools, adding `ADD_CUSTOM_ARRAY_BINDING_ITEMS` first when structuredData has no items. Never pass `seoConfig` to a generic property-setting tool — it is refused, because a partial blob would drop the other fields. sitemapLastmod is not a binding (a fixed date or a table column) and has no write tool yet. Look types up with `GET_ALL_SEO_STRUCTURED_DATA_TYPES_INFO` and `GET_SEO_STRUCTURED_DATA_TYPE_DETAIL`. **Do not touch these fields unless the user asks for SEO** — an unrequested change to indexing is user-visible.

### Data Bindings on Components
`GET_COMPONENT_BINDABLE_PROPERTIES` lists every place a component can be bound — always start there and copy the paths verbatim, never hand-assemble one. It returns `properties`, the component's own slots, each with its ready-to-use schemaPath, expected type, current binding and scope; and `actionSlots`, the parameter slots of the actions already wired to its events, chained success/failure branches included. `ADD_COMPONENT_ACTIONS` echoes an action's slots when it creates it — bind them in the same turn when you can, and use `actionSlots` to find them again later. Fill the echoed paths with the bindings plugin's tools: `BROWSE_DATA_BINDING_OPTIONS` + `CREATE_OPTION_BINDING` for dataSource slots and other dynamic values, `CREATE_CONST_BINDING` for literals. A LIST, SELECT_VIEW or DATA_SELECTOR can take its dataSource either way, and the options tree offers both in one answer: the tables under 'Table', and any page query under 'Context > Current page > Data source'. **Prefer the table** — a component that owns its data source is filtered, sorted and retyped in one place, and `GET_REQUEST_FILTER_CONTEXT` narrows it at the component's own path. Reach for a page query (`ADD_COMPONENT_QUERY` first) only when the SAME result set is genuinely shared — two components reading one query, or a filter the page tunes centrally. One component, one filtered list is not a sharing case: a page variable there buys nothing and leaves a second thing to keep in step. Inside a list cell (scope "list-cell:<listId>") the options tree additionally offers the current list item's fields under 'item-data' — bind cell children to those instead of re-querying the table.

### Editing Protocol
- One read of a container covers adding children to it and editing its own configuration, so `GET_COMPONENT_INFO` and `GET_CONTAINER_CHILDREN_INFO` on the same component is one call wasted. After a conflict, re-read and retry.
- Wiring behavior: `GET_COMPONENT_SELECTABLE_EVENTS`, then `GET_SELECTABLE_COMPONENT_ACTIONS` for what that event accepts, then `ADD_COMPONENT_ACTIONS`. Anything a kind returns beyond its bindableSchemaPaths arrives under that handler's `details`.
- Pick the kind from `GET_SELECTABLE_COMPONENT_ACTIONS` and its parameter shape from `ADD_COMPONENT_ACTIONS`'s parameter schema (fetch it first if it was left out as "schema on request") — the only complete list of kinds and fields. The editor offers far more than the handful named below — SHOW_TOAST, OPEN_EXTERNAL_LINK, SET_CLIPBOARD, DOWNLOAD_FILE, CALL_PHONE, the payment and animation kinds — so do not conclude an interaction is impossible because it is not discussed here. What follows is only the judgment the schema cannot carry.
- ALERT_DIALOG is the runtime's own confirm box — title, message and two buttons it draws itself, no MODAL to build. Use it for "are you sure?"; use SHOW_MODAL when the dialog needs your own components. Chain what confirming does onto its ON_CONFIRM branch, or the buttons do nothing.
- Building a MODAL does not make it reachable: without a SHOW_MODAL action pointing at it, nothing opens it and the modal is dead UI.
- An action that produces a RESULT — GET_LOCATION, UPLOAD_FILE, GENERATE_QR_CODE and the other result-bearing kinds — keeps it only through assignToVariable, a writable page variable created with `ADD_COMPONENT_VARIABLES` on the page the component sits on. Their success branches cannot see the result, so without assignToVariable the action throws its answer away. Create the variable first.
- Which sign-up / sign-in leaves GET_SELECTABLE_COMPONENT_ACTIONS offers follows the project's sign-in settings. When the one a request needs is missing, the method is off: read GET_AUTHENTICATION_CONFIG_INFO and turn it on with UPDATE_AUTHENTICATION_CONFIG, then list again. Ask the user before turning WeChat on (it needs their AppID and AppSecret) and before turning off a method the app already uses.
- SIGN_UP and USER_LOGIN are separate actions, and the password slot on a SIGN_UP decides what the account can sign in with later: leave it unbound and the account has no password, so USER_LOGIN with credentialType PASSWORD can never succeed for anyone who signed up there. Build the pair together, or sign in with VERIFICATION_CODE.
- A WeChat mini-program signs its user in by itself: the client is created with a silent WeChat login in its own app-did-load configuration, which the editor will not let anyone delete, so whoever opens the app is already signed in and the account is created there. Do NOT build a sign-in or sign-up screen or wire a login action: USER_LOGIN offers no WeChat mode, the silent login belongs to that configuration alone, and SIGN_IN / SIGN_UP / SIGN_OUT are absent from the mini-program's selectable actions. When the app needs the phone number, use OBTAIN_PHONE_NUMBER.
- MUTATION writes the database without an action flow. rolesWithoutPermission in its result is report-only — grant the table permission with the permission plugin or the write is denied at runtime, which looks like a silent no-op. rowScope CURRENT_ITEM is valid only inside a LIST cell; refreshOnSuccess beats a separate REFRESH. `UPDATE_COMPONENT_MUTATION_ACTION` adds what the editor's panel configures: batch insert (one row per list item), per-column validation checks shown before the request, and an insert's conflict handling on a unique or primary key. Prefer batch insert to a loop of single inserts, and validation to a CONDITIONAL in front of the MUTATION.
- CONDITIONAL is how ONE event does different things in different cases: branches in evaluation order, first match wins, each narrowed at the echoed conditionSchemaPath and filled by a further `ADD_COMPONENT_ACTIONS` carrying that branch's conditionalBranchId. Leave the last branch always-true as the else. Without it every action on the event fires on every click — ON_SUCCESS / ON_FAILURE branch on whether a call worked, never on a data condition. Do NOT reach for an action flow to get branching: a flow's BRANCH node runs on the server and cannot navigate, toast or open a modal. This decides what HAPPENS; a CONDITIONAL_VIEW decides what is DISPLAYED.
- There is no per-component conditional visibility — an ordinary component has no data-driven show/hide flag. To show content only when a condition holds, wrap it in a CONDITIONAL_VIEW and gate the branch with the bindings plugin's condition tools.

### Motion Is One Moment, Not A Coat Of Paint
Zion has animation as ACTIONS, not style fields, so they are attached with `ADD_COMPONENT_ACTIONS` like any other handler: ANIMATION_FADE_EFFECT and ANIMATION_SLIDE_EFFECT (APPEAR / DISAPPEAR, eight directions, a distance defaulting to 150px), ANIMATION_SCALE_EFFECT, ANIMATION_COMMON_EMPHASIS (BOUNCE / PULSE / BLINK), ANIMATION_FLIP_EMPHASIS, and ANIMATION_SCROLLING_INTERACTION. Each takes delay; the emphasis kinds take repeat or loop.
Breaking these rules shows up as a project error rather than a subtle defect: exactly ONE animation per event, and only on onClick, onHover or onScrollIntoPage — the runtime reads the last effect on the event and ignores any attached elsewhere. ANIMATION_SCROLLING_INTERACTION is the exception: it goes ONLY on onPageScroll, which carries nothing else, because the curve is driven by scroll distance. None take attachTo and none take bindings, so they cannot be chained onto a success branch.
Spend motion the way you spend the accent. Movement that answers what a person just did — a press, an open, a confirm — is worth it, because it shows what changed. Movement that merely announces that content exists is the generated-page tell: a fade-and-slide on every section reads as a template. One orchestrated moment on the screen that matters beats an effect on every card, and no animation at all is a perfectly good answer. The emphasis kinds are for something genuinely needing attention, never decoration, and BLINK almost never earns it.

### Reading A Screenshot
A `component capture-canvas` shows what the user will see, which `schema validate`, knowing only invalid schema, cannot. Take one when the request calls for seeing the screen: you built a screen the user will judge, or they asked about its look; a change that only adds or rewires something on an existing screen does not need one. Name the page with `--args '{"pageId":"<id>"}'`, adding `"breakpoint":"phone"` for the phone layout. It saves a PNG and prints its path: open that file to look at it. When it reports that no browser could be started, say that you could not see the screen rather than describing it. Read it against: one clear focal point rather than everything at the same weight; related things closer together than unrelated ones; at least three levels of text that differ in weight, not only size; gutters where things would otherwise touch an edge; on web, lines under ~70 characters and something that responds to hover; the accent only on the primary action and the active state; colour, spacing, radius, type and shadow values that are theme references, so a restyle reaches them. Report only what you actually changed. Every claim about appearance — a heading looks larger, a selected state looks different, spacing was tightened — must correspond to an `UPDATE_COMPONENT_STYLE` (or creation-time style) call you made in this session. If a requested look was not achievable with the available tools, say which part you skipped and why; do not describe a component's built-in default styling as if you had applied it.

## How to drive it (CLI only)

All commands are `npx -y zion-mcp@2.8.7 <verb>`. A long-lived daemon holds the in-memory CRDT schema session
between calls. **Edits do NOT go live until `project sync-backend`.**

```bash
npx -y zion-mcp@2.8.7 whoami                                    # check auth; if needed: npx -y zion-mcp@2.8.7 login
# create a NEW project (auto-pins it and its client; BACKEND_ONLY only if the front end is hosted elsewhere):
# clients: WEB | WECHAT_MINI_PROGRAM | MOBILE | BACKEND_ONLY
npx -y zion-mcp@2.8.7 project create --projectName "My App" --appType WEB
# …or pin an EXISTING one (find its exId with npx -y zion-mcp@2.8.7 projects search):
npx -y zion-mcp@2.8.7 project set-current --projectExId <exId>
npx -y zion-mcp@2.8.7 schema load                               # warm the schema session
```

`project set-current` also makes the project's only client the current one, which is what pages and
components are written to. A project with several clients is refused with its client list (a project created before multi-client has a web client **and** a mini program), so pin the one to work on: `project set-current --projectExId <exId> --appExId <appExId>`.
Pass `--appExId null` to address the project's backend document alone (data model, action flows,
APIs, permissions, secrets).

Operations run through one verb:

```bash
npx -y zion-mcp@2.8.7 schema tool-call --toolCalls '[{"name":"<TOOL_NAME>","args":{ ... }}]'
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
| Batch insert, column validation, conflict handling on a MUTATION action | `UPDATE_COMPONENT_MUTATION_ACTION` | `schemaPath` |
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
| Sign-in methods: which are on, validity, SSO providers | `GET_AUTHENTICATION_CONFIG_INFO` | — |
| Turn sign-in methods on or off, set WeChat credentials | `UPDATE_AUTHENTICATION_CONFIG` | — |
| SEO context: rendering mode, pages with path params | `GET_SEO_INFO` | — |
| One page's SEO path ranges | `GET_SEO_PAGE_DETAIL` | `pageId` |
| Set literal values for a URL path parameter | `SET_SEO_PATH_STATIC_OPTIONS` | `options`, `pageId`, `pathData` |
| Add a database-backed path value source | `ADD_SEO_PATH_DB_QUERY` | `columnDisplayName`, `pageId`, `pathData`, `tableDisplayName` |
| Update a database-backed path value source | `UPDATE_SEO_PATH_DB_QUERY` | `index`, `pageId`, `pathData` |
| Remove a database-backed path value source | `DELETE_SEO_PATH_DB_QUERY` | `index`, `pageId`, `pathData` |
| Remove a URL path parameter's whole range | `DELETE_SEO_PATH_RANGE` | `pageId`, `pathData` |
| Switch the rendering method | `SET_SEO_RENDERING_METHOD` | `renderingMethod` |
| Replace the robots.txt content | `SET_SEO_ROBOTS_TXT` | `content` |
| schema.org vocabulary: every type, top-level flag, what it extends | `GET_ALL_SEO_STRUCTURED_DATA_TYPES_INFO` | — |
| One schema.org type's properties and their ranges | `GET_SEO_STRUCTURED_DATA_TYPE_DETAIL` | `typeName` |
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

Component writes go through the CLI like any other edit: `GET_COMPONENT_TEMPLATE` then `ADD_COMPONENT` to build, `UPDATE_COMPONENT_STYLE` to restyle, `MOVE_COMPONENTS` to restructure, `ADD_COMPONENT_ACTIONS` to wire events, `DELETE_COMPONENTS` to remove — see "Building & Editing the Component Tree" above. A screen can be schema-valid and still look wrong: verify structurally with `GET_CONTAINER_CHILDREN_INFO` and `GET_INCOMPLETE_STRUCTURES` (its `responsiveRisks` catches the desktop-width-on-phone case), and where appearance is the acceptance criterion, look at it with `component capture-canvas` (see "Reading A Screenshot" above) rather than guessing.

The preset tab-bar icon library is reachable: `npx -y zion-mcp@2.8.7 component tab-bar-icons` lists every icon this deployment ships as a name and the media exId `UPDATE_TAB_BAR_ITEMS` takes. A shown tab needs an icon for both its normal and its selected state or the project reports an error, and the exIds are hashids of media rows that cannot be derived from an icon's name — a value that could not be an exId is refused by the write rather than failing the mini-program build over a file it cannot find, so take them from that listing and never invent one. An icon the library does not ship still has to be brought in through the editor (`web-assets.md`) or picked there by the user: nothing here imports media.

Then ship:

```bash
npx -y zion-mcp@2.8.7 schema validate && npx -y zion-mcp@2.8.7 project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
