# Carrying a direction onto the theme

Use the theme plugin; reference variables everywhere, write literals nowhere.

- **Colours** (`update_theme_variables`): set the PALETTES basic colours — Primary, Secondary, Tertiary (the data's accent), Error, and Neutral from `text_secondary`. Read the levels back with `get_theme_info` and point Background, Surface, Border, Text_primary and Text_secondary at the closest levels; in a dark direction ground and surface take dark levels and text light ones. Use a custom hex only where no level comes close.
- **Buttons**: text on a primary fill is white. When white on the primary is below 4.5:1, fill primary buttons with a darker level of the primary palette that reaches it rather than switching to dark text; only a very light primary such as a bright yellow keeps dark text.
- **Radius, spacing, shadow**: carry the style's `variables` onto the theme's preset variables by retuning their values (radius sm 4, md 8, lg 16, xl 24, full).

  | Style radius | Buttons and inputs | Cards | Pills and tags |
  |---|---|---|---|
  | none | 0 | 0 | 0 |
  | sm | sm | sm | full |
  | md | md | md | full |
  | lg | md | lg | full |
  | xl | lg | xl | full |
  | full | full | xl | full |

  Shadow none: cards sit on a hairline Border. sm: cards take the sm shadow and no border. md: cards take md, and at most one level more (modals, popovers) above it.
- **Fonts**: the data gives a character, not a face — its Google Fonts are not on this platform. Pick the nearest of `offeredFontFamilies` from `get_theme_info`; Chinese body text needs a CJK face. Mini-program and mobile clients offer only system faces, so carry the character through weight and size.
- **Effects**: glass becomes a translucent surface with `backdropBlur` on overlays only; neumorphic or 3D depth becomes the theme's soft shadows; motion follows the component plugin's animation rules. Drop what cannot be drawn.
