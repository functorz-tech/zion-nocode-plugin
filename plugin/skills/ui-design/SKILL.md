---
name: ui-design
description: Makes the app's screens look finished. Settles a client's visual style (from what the user gave, its DESIGN.md, or three directions drawn from the ui-ux-pro-max design data for this kind of app), writes the DESIGN.md and the theme, then builds and self-checks. Use whenever you create pages, build or restyle UI, change the theme, or the user asks to create or update a DESIGN.md. Only for projects on this no-code platform (Zion / Momen); not for generic web or frontend code.
---

# UI design

The theme is the design system: every colour, radius, spacing, font and shadow a component uses references a theme variable, so a style change is a theme edit. DESIGN.md is the project's record of the decision behind those values, one per client: `/project/DESIGN.md`, or `/project/app/<appExId>/DESIGN.md` in a multi-client project; working outside the editor, keep it beside the project's other notes. It belongs to the project, not to this skill: one the user wrote or uploaded is followed the same as one written here. This skill is how one gets made or changed.

`assets/uupm/` is the ui-ux-pro-max design data, unmodified. It is written for HTML, CSS and Tailwind; this file says how to read it and how to carry it onto this platform. Read rows with your file search (`grep_files` in the editor); never run its scripts.

## 1. Settle the style before the first component

Stop at the first that applies.

1. **The user gave a direction this turn** — words ("dark", "warmer", a brand colour), a screenshot or design file, HTML/CSS, or "like <product>". Extract colours, radius, spacing density, shadow and type mood: read HTML/CSS values exactly, estimate from images, and for a named product use what you know of its look. Go to section 2 with that.
2. **The user asked for a DESIGN.md.** If they want the current look written down, capture the client's main screens and read the theme, then write what is there into the template — colours by role, radius, spacing density, shadow, type — without asking for a direction. If they want a new look, offer directions as in item 4. If it is unclear which, ask.
3. **The client has a DESIGN.md.** Follow it. If its values disagree with the theme and the user has not said which wins, ask once which to keep.
4. **No DESIGN.md.** Ask once with your multiple-choice question tool (`ask_user_question` in the editor), then write DESIGN.md; never ask again while it exists. Ask in the first turn of the work, in the same message that states the first-release cut and before building anything, with that turn's other questions in the same card, so the style never costs a round trip of its own. In that message, say in one line where each part lives: the style they pick is written to the project's DESIGN.md for this client, which they can open and edit like any project file; how the options were chosen comes from the ui-design skill in their Skills, which they can read or change.

### Finding the three directions

1. **The product row.** Search `assets/uupm/products.csv` on the app's subject in English (`Product Type`, `Keywords`), e.g. `beauty|salon|spa` for a nail studio. Take the closest row. Then read the row of the same name in `assets/uupm/ui-reasoning.csv`: `Style_Priority`, `Color_Mood`, `Typography_Mood`, `Key_Effects` and `Anti_Patterns` hold for every direction you offer.
2. **The three styles.** The first is the row's `Primary Style Recommendation`; the next two come from its `Secondary Styles`. If those repeat the first in feel, take one from `assets/uupm/styles.csv` whose `Best For` fits the subject and whose `Keywords` differ. Never offer a style the product's `Anti_Patterns` or the style's `Do Not Use For` rules out. Read each chosen style's row in `styles.csv` for its keywords, colours, effects and `Design System Variables`.
3. **Colours per direction.** The first direction takes the product's row in `assets/uupm/colors.csv` (same `Product Type`). The others take the style's own `Primary Colors` from `styles.csv`, tuned to the product's `Color Palette Focus`.
4. **Type per direction.** Match `Typography_Mood` and the style's keywords against `Mood/Style Keywords` in `assets/uupm/typography.csv` for a pairing's character (serif or sans, weight, contrast); the faces themselves are chosen in section 2 below.
5. **The card.** Label each option with a short feel name and one plain sentence in the user's language — ground, accent colour, corner shape, depth, spacing, type — written for someone who knows no design terms, recommended first. Do not name the data source or English style names.

- If the client already has pages with content (a blank default page does not count), offer "keep the current look" plus two directions. When they pick a new style, ask in the same card whether existing pages switch too: that means rewriting their literal colours, radii and spacings as theme references.
- Which option to recommend for a client with pages: capture its main page first and read it against `assets/checklist.md`. A plain screen (grey boxes, default colours, no hierarchy, the accent spent nowhere in particular) gets a new style recommended; a screen with a look of its own (its own colours, clear hierarchy) gets "keep the current look" recommended. Judge by the capture, not the theme: older pages often carry literal styles the theme does not show.
- If the user answers something else instead, a new client takes the recommended direction and a client with pages keeps its look; say which in the reply.

## 2. Write the style

- **DESIGN.md**: fill `assets/DESIGN.template.md` from the chosen direction (or what you extracted), including its do's and don'ts from the style row and the product's `Anti_Patterns`. Update it whenever the style changes later.
- **Colours** (theme plugin, `update_theme_variables`): set the PALETTES basic colours — Primary, Secondary and Tertiary (the data's Accent), Error (Destructive), Neutral from the muted foreground. Then read the levels back with `get_theme_info` and point Background, Surface (the data's Card), Border, Text_primary (Foreground) and Text_secondary (Muted Foreground) at the closest levels; in a dark direction the ground and surface take dark levels and the text light ones. Use a custom hex only where no level comes close.
- **Buttons**: text on a primary-filled button is white. When white on the primary falls below 4.5:1, fill primary buttons with a darker level of the primary palette that reaches it instead of switching to dark text; only a very light primary such as a bright yellow keeps dark text. The data's `On Primary` column picks black whenever black contrasts more, which reads wrong on pinks and pastels.
- **Radius, spacing, shadow**: carry the style's `Design System Variables` (border radius, shadow, spacing) onto the theme's preset variables, retuning their values rather than writing literals; `assets/sizes.md` maps radius and shadow levels to components.
- **Fonts**: the pairing gives a character, not a face — its Google Fonts are not on this platform. Pick the nearest of `offeredFontFamilies` from `get_theme_info`; Chinese body text needs a CJK face. Mini-program and mobile clients offer only system faces, so carry the character through weight and size.
- **Effects the platform draws differently**: glass becomes a translucent surface with `backdropBlur` on overlays only; neumorphic or 3D depth becomes the theme's soft shadows; motion follows the component plugin's animation rules. Drop what cannot be drawn rather than faking it.

## 3. Build

Follow DESIGN.md. Reference theme variables for every styled value, size per client by the component plugin's spacing numbers, and map the style's radius and shadow to components with `assets/sizes.md`. `assets/uupm/ux-guidelines.csv` and `assets/uupm/quick-reference.md` hold the UX rules (touch targets, forms, feedback, navigation); read the rows for what you are building when a choice is not obvious.

## 4. Check before replying

Capture the screen and go through `assets/checklist.md` for that client, then the pre-delivery checklist in `assets/uupm/pro-rules.md` where it applies to what this platform can draw; fix what fails. Tell the user which style was used and that they can ask to change it.
