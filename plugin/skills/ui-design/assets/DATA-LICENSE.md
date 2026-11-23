# Data source

`directions.csv` and `styles.csv` are column extracts of ui-ux-pro-max (nextlevelbuilder/ui-ux-pro-max-skill, commit 4f2dbb67e64969a9a30ab00742f2d55d66ddec2b, `.claude/skills/ui-ux-pro-max/data/`), with no values rewritten:

- `directions.csv`: one row per `products.csv` row — `Product Type`, `Keywords`, `Primary Style Recommendation`, `Secondary Styles`; from the `ui-reasoning.csv` row of the same name `Color_Mood`, `Typography_Mood`, `Key_Effects`, `Anti_Patterns`; from the `colors.csv` row of the same name `Primary`, `Secondary`, `Accent`, `Background`, `Card`, `Foreground`, `Muted Foreground`, `Border`, `Destructive`.
- `styles.csv`: every `styles.csv` row not `deprecated` — `Style Category`, `Keywords`, `Primary Colors`, `Secondary Colors`, `Effects & Animation`, `Do Not Use For`, `Preferred Mode`, `Design System Variables`.

Its licence:

MIT License

Copyright (c) 2024 Next Level Builder

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
