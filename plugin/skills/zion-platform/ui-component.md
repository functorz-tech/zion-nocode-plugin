# UI components & layout

## Component Domain Knowledge
The UI is a tree of components organized into pages.

### Categories
Layout (container — can hold other components):
- PAGE: root of each page; no parent.
- LAYOUT_VIEW: generic Flexbox container; the most common container.
- MODAL: popup/dialog container.

Special (complex — add children to their built-in slots, NOT directly to the component):
- LIST: repeating list; has a built-in "List Item View" slot.
- TAB_VIEW: tabbed interface; has a built-in view per tab.
- SELECT_VIEW: has built-in selected/unselected state views.
- CONDITIONAL_VIEW: has a built-in view per condition branch.

Leaf/Business (cannot have children): BUTTON, TEXT, IMAGE, VIDEO, LOTTIE, RICH_TEXT, TEXT_INPUT, NUMBER_INPUT, VIDEO_PICKER, FILE_PICKER, SWITCH, MAP, MAP_MARKER, CALENDAR, HORIZONTAL_LINE, SHEET, HTML, RICH_TEXT_EDITOR, MIX_IMAGE_PICKER, DATE_TIME_PICKER, DATA_SELECTOR, PROGRESS_BAR. MAP_MARKER can only be placed inside MAP.

### Hierarchy Rules
Every component except PAGE and MODAL must have a Layout Component as its parent. Prefer relative positioning; use absolute/fixed only when truly necessary.

### Layout
Zion uses Flexbox. Layout containers configure: direction (row/column), justify-content (start, space-between, space-evenly, space-around), gap, wrap, overflow (scroll/visible/hidden).

### Responsive Breakpoints
Phone: 0–767 px | Tablet: 768–1279 px | Desktop: 1280–1920 px. Primary breakpoint is Desktop; changes propagate unless overridden per breakpoint.

### Component Configuration (right sidebar in editor)
- Design tab: layout, spacing, size, visual styles.
- Data tab: bind data sources, conditional visibility.
- Action tab: event triggers (onClick, onChange, onLoad, etc.).

### Reading the Component Tree
These read tools inspect the live component tree (component construction and editing are done in the editor, not here):
- `GET_ALL_PAGE_NAMES` — display names of all pages in the project.
- `GET_COMPONENT_INFO` — one component's type and config, by its component id.
- `GET_CONTAINER_CHILDREN_INFO` — the direct children of a container (or a special component's slot), by the container's id.
- `GET_COMPONENT_CONTEXT_INFO` — a component together with its ancestor chain and siblings, so you can see where it sits in the page.

## How to drive it (CLI only)

All commands are `"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" <verb>`. A long-lived daemon holds the in-memory CRDT schema session
between calls. **Edits do NOT go live until `project sync-backend`.**

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" whoami                                    # check auth; if needed: "${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" login
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project set-current --projectExId <exId>  # pin the project ("${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" projects search to find it)
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema load                               # warm the schema session
```

Operations run through one verb:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema tool-call --toolCalls '[{"name":"<TOOL_NAME>","args":{ ... }}]'
```
Each call is applied immediately — any resulting CRDT patch is uploaded. Batch several calls in one array; use `schema undo` to revert the last change.
A batch is all-or-nothing: when any call in the array fails, the whole batch's changes are discarded even though the other calls returned success — only the failing call's error is reported, so after a batch error re-read (`GET_*`) before assuming anything persisted.

## Operation reference (`schema tool-call` names)

| Intent | `name` | Required `args` |
|---|---|---|
| List pages | `GET_ALL_PAGE_NAMES` | — |
| Component info | `GET_COMPONENT_INFO` | `componentId` |
| Data/vars in scope at a component | `GET_COMPONENT_CONTEXT_INFO` | `componentId` |
| Container children info | `GET_CONTAINER_CHILDREN_INFO` | `componentId` |

Building or changing UI is the user's job, done in the editor — not yours. Component create / update / delete is intentionally unavailable through the CLI, so for anything visual (pages, components, layout, styling) **only instruct the user**: give concrete, numbered editor steps for them to perform, and never attempt to build UI on their behalf. Through the CLI you can only *inspect* the component tree (above) and bind already-placed component properties to data with `data-binding.md`.

Then ship:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" schema validate && "${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
