# Radius and shadow by style

Spacing per client is in the component plugin. Use the theme's radius (sm 4, md 8, lg 16, xl 24, full) and shadow presets; retune a preset's value rather than writing a literal when a style needs a different number.

## Radius by style

| Style radius | Buttons and inputs | Cards | Pills and tags |
|---|---|---|---|
| none | 0 | 0 | 0 |
| sm | sm | sm | full |
| md | md | md | full |
| lg | md | lg | full |
| xl | lg | xl | full |
| full | full | xl | full |

## Shadow by style

- none: cards sit on a hairline border (Border).
- sm: cards take the theme's sm shadow and no border.
- md: cards take md; at most one more level (modals, popovers) above it.
