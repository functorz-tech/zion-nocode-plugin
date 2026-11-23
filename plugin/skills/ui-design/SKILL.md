---
name: ui-design
description: Keeps a client's screens on one visual style. Use whenever you build or restyle screens, change the theme, or the user asks for a different look or a DESIGN.md. A client with a DESIGN.md follows it; one without gets its style here — the direction the user gave, or the recommended one for this kind of app from the ui-ux-pro-max design data — written into its DESIGN.md and theme. Only for projects on this no-code platform (Zion / Momen); not for generic web or frontend code.
---

# UI design

The theme is the design system; DESIGN.md is the project's record of the style, one per client: `/project/DESIGN.md`, or `/project/app/<appExId>/DESIGN.md` in a multi-client project. It belongs to the project: one the user wrote is followed the same as one written here.

**If the client has a DESIGN.md and the user has not asked for another look, read it, follow it on every screen, and stop here.** Its values win over your own taste; if they disagree with the theme and the user has not said which wins, ask which to keep.

Otherwise decide the style when the first screen is about to be built, after any plan is approved — not as a question in the first turn. Read only the file for the case you are in:

| Case | What to do | Read |
|---|---|---|
| The user gave a direction: words ("dark", a brand colour), a screenshot or design, HTML/CSS, "like <product>" | Take colours, radius, density, shadow and type mood from it: HTML/CSS values exactly, images by eye, a named product from what you know of it | `assets/theme-mapping.md` |
| No DESIGN.md, and the client has no screens of its own yet (a blank default page counts as none) | Use the recommended direction; do not ask | `assets/pick-style.md`, then `assets/theme-mapping.md` |
| No DESIGN.md, and the client already has screens with a look of their own | Keep that look: capture the main screen, read the theme, write down what is there | `assets/DESIGN.template.md` |
| The user asks for another look, or to choose between looks | Offer or apply directions as they asked | `assets/pick-style.md`, then `assets/theme-mapping.md` |
| The user asks for a DESIGN.md of the current look | Capture and read the theme, write what is there; no new direction | `assets/DESIGN.template.md` |

Then fill `assets/DESIGN.template.md` and set the theme. In the reply, say in one line which look was used, name the two alternatives by feel so the user can switch with a word, and that the style lives in the project's DESIGN.md, which they can edit.

Before replying about screens you built or restyled, read a capture against `assets/checklist.md` and fix what fails.
