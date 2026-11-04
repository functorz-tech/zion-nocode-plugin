# Global theme (variables & scrollbar)

# Global theme

A themed project keeps its colours and sizes in one place and components reference them, so restyling is one edit rather than a sweep over every component. GET_THEME_INFO reports every variable in all eight categories with its id, displayName, isPreset flag and value under the default theme — read it before editing, because the add and update tools take a value in the category's own shape and this is where to copy that shape from.

## Referencing a variable
A variable's referenceValue is `$$` followed by its id, in every category, and that is what UPDATE_COMPONENT_STYLE takes: colours, spacings, radii, font sizes, weights and families on the field of that class, a shadow as `shadow: {variable}`. Only a component holding a reference follows when the variable changes, which is what turns a restyle into a few edits here. There is no line-height category, so a text role is a size and a weight variable paired by name, with the line height sent as px alongside.

## Font families
The theme starts with none, so text renders in the platform default until one is added and referenced. GET_THEME_INFO lists the families this client offers (offeredFontFamilies); a FONT_FAMILIES value outside that list is rejected.

## Editing
ADD_THEME_VARIABLES returns each new variable's id, which is what component bindings reference. UPDATE_THEME_VARIABLES is incremental — omitted fields keep their current value. DELETE_THEME_VARIABLES refuses preset variables: their ids carry semantics the palette algorithm and existing bindings depend on, and a component left bound to a deleted variable falls back to no value, so check usages first.

## Palettes and levels
A palette is one basic colour and the twelve levels derived from it, level 1 lightest to level 12 darkest; GET_THEME_INFO lists the levels as hex so one can be chosen by sight. A colour variable's value should be `{kind:'paletteRef', paletteId, level}` whenever the colour is a shade of something the theme already has — a page ground, a lifted surface, a border and a divider are Neutral levels; a hover or pressed state is a level of the palette its resting colour follows — because that is what lets a new brand colour restyle it along with everything else. A `{kind:'custom', color}` hex stays put through every recolour, so it belongs only to a colour that genuinely must not move. A paletteRef without a level means the palette's basic colour itself: the semantic presets (Primary, Success, …) take that form because a brand colour's lightness is arbitrary and no fixed level is guaranteed to equal it.

## Scrollbar
SET_SCROLLBAR_STYLE is web-only; other clients draw the scrollbar with the OS.

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
| Every variable, all categories, plus the scrollbar | `GET_THEME_INFO` | — |
| Add theme variables | `ADD_THEME_VARIABLES` | `items` |
| Rename a variable or replace its value | `UPDATE_THEME_VARIABLES` | `items` |
| Delete theme variables (presets rejected) | `DELETE_THEME_VARIABLES` | `items` |
| Scrollbar appearance (web only) | `SET_SCROLLBAR_STYLE` | — |

Theme variables carry a value whose shape differs per category, and that shape is not documented per category — read `GET_THEME_INFO` first and copy the shape of an existing variable in the same category, the same way component styles are copied from `GET_COMPONENT_TYPE_CAPABILITIES`. Preset variables cannot be deleted: their ids are what bindings and the theme algorithm resolve against. A component still bound to a deleted variable falls back to no value rather than erroring, so it goes wrong silently — check usages before removing one.

Components reference a variable by its `referenceValue` (`$$<id>`), never by its id, and that is what `UPDATE_COMPONENT_STYLE` takes in a colour field (`ui-component.md`). `GET_THEME_INFO` lists the ones this client defines — read the ids there rather than from memory.

## Arguments (generated from ztype)

Shapes and field docs below are generated from ztype's `tool-schemas.json` (the source of truth) — never hand-built. `schemaPath` is a `DiffPathComponents` array (`{key}` for an object step, `{index}` for an array step) and is always read back from a discovery call (see above), never fabricated.

### `ADD_THEME_VARIABLES`

Add theme variables. The result echoes each new variable's id, which is what component bindings reference.
- `items` *(required)*: `array<{category: enum(PALETTES|COLOR_VARIABLES|FONT_FAMILIES|FONT_SIZES|FONT_WEIGHTS|RADII|SPACINGS|SHADOWS), displayName: string, value: string | number | object}>`
  - `items[].category` — Which category to add to.
  - `items[].displayName` — Label shown in the editor. Need not be unique.
  - `items[].value` — The variable's value under the default theme, as JSON matching the category: a color string for PALETTES (the palette's basic color, the 12 steps are derived) and FONT_FAMILIES; a `{kind, ...}` color value for COLOR_VARIABLES — `{kind:'paletteRef', paletteId, level}` follows one of that palette's 12 derived levels (1 lightest, 12 darkest; GET_THEME_INFO lists them), omit `level` to follow the basic colour itself, and `{kind:'custom', color}` pins a hex that no palette recolour will move — prefer paletteRef for any shade of a colour the theme already has (surfaces, borders and dividers are Neutral levels); a number for FONT_WEIGHTS; a `{desktop, tablet, phone}` object for FONT_SIZES / RADII / SPACINGS (non-web clients take `{phone}` only); a `{type, offsetX, offsetY, blur, spread, color}` object for SHADOWS. Read GET_THEME_INFO first to copy an existing entry's shape.

### `UPDATE_THEME_VARIABLES`

Rename a theme variable and/or replace its value under the default theme (incremental — omitted fields keep their current value).
- `items` *(required)*: `array<{category: enum(PALETTES|COLOR_VARIABLES|FONT_FAMILIES|FONT_SIZES|FONT_WEIGHTS|RADII|SPACINGS|SHADOWS), displayName?: string, value?: string | number | object, variableId: string}>`
  - `items[].category` — Which category the variable is in.
  - `items[].displayName` — New label; omit to keep the current one.
  - `items[].value` — New value under the default theme, same shape as in ADD_THEME_VARIABLES; omit to keep the current one.
  - `items[].variableId` — The variable's id (from GET_THEME_INFO).

Then ship:

```bash
npx -y zion-mcp@2.8.2 schema validate && npx -y zion-mcp@2.8.2 project sync-backend
```
`project sync-backend` aborts with `SAVE_SCHEMA_WITHOUT_PATCHES` when nothing is pending — make at least one change before shipping.
