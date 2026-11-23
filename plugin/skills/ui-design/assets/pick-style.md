# Picking a direction

`directions.csv` has one row per kind of app; `styles.csv` one row per style. Read rows with your file search (`grep_files` in the editor), never whole files.

1. **The app's row.** Search `directions.csv` on the app's subject in English (`product`, `keywords`), e.g. `beauty|salon|spa` for a nail studio, and take the closest row. Its `color_mood`, `type_mood`, `effects` and `avoid` hold for any direction you use.
2. **The recommended direction** is the row's `style` with the row's own colours (`primary` … `error`). Read the `styles.csv` row of each style it names (a `+` joins two) for `effects`, `not_for` and `variables` (radius, shadow, spacing). The search is a regular expression: anchor the name at the line start and escape its brackets and dots, e.g. `^Dark Mode \(OLED\),`.
3. **The two alternatives** come from `alternatives`. Name each by its feel in the user's language for the reply; read its `styles.csv` row only when the user picks it, and then take its `colors`, tuned to the app's `color_mood`.
4. **When the user wants to choose** before anything is applied, put the three as options of your multiple-choice question tool (`ask_user_question` in the editor), recommended first: a short feel name and one plain sentence each — ground, accent colour, corner shape, depth, spacing, type — with no English style names and no data source.

Never use a style the app's `avoid` or the style's `not_for` rules out; take the next alternative instead.
