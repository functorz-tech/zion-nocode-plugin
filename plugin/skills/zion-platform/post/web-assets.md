# Web pages & asset import

## Web Domain Knowledge
Two tools, both reaching the public internet through the same guard: internal and private addresses are refused, redirects are followed to a limit, and a response over 10MB is cut off.

### Reading a page
`tpa fetch-doc` returns a page as Markdown plus three link lists. `pageLinks` are same-site pages worth following; `specLinks` are OpenAPI/Swagger documents, which this project has no bulk importer for — read the spec and build the endpoints with the `api` plugin; `imageLinks` are the absolute URLs of the images the page renders. Long pages arrive windowed — read on with `offset` rather than re-fetching, since the 15-fetch session budget counts every call.

### Copying a page
Asked to copy or reproduce a page, build what the source actually uses, not something that resembles it. Every `imageLinks` entry is an asset the page really renders, so a picture in the source becomes a picture here — importing it and putting it in an IMAGE component. Standing in a text glyph, an emoji or a coloured box for one is a different page that only photographs well: it carries none of the artwork, it cannot be swapped for the real asset without rebuilding the component, and the person who asked for a copy has to notice the substitution and ask again. The same holds the other way — where the source really does set a character in text, that is a TEXT component, not an image of one. Decide per element by what the fetched page shows, and if a needed asset did not come back in `imageLinks`, say so rather than approximating it.

The feature icons are the ones most often lost this way. They arrive as `format: svg` with no `alt`, which reads like page furniture next to the labelled hero shots, and a card built with '◉' where the source has an icon looks finished in a screenshot while carrying none of the source's artwork. Count the icons the page draws and import that many.

Import every asset the copy needs before building, and check each result. An import that failed leaves you an asset short: rebind nothing, re-read the URL against `imageLinks` and retry it. Binding one assetExId to two components to cover the gap puts the same picture in both places, which reads as a build that finished rather than one that lost a file.

### Putting a picture on the canvas
Import it. `import_asset` with `mode: STATIC` copies the file into the project's own storage and returns an `assetExId`; pass that to `CREATE_CONST_BINDING`, or to the `backgroundImage` style field, as the constant value. This is the default for every picture you put on a page — the editor's own image picker has no way to bind a remote URL at all, and a project built by hand therefore never contains one.

`const_value` decides between the two forms by what you send: a string starting `http://` or `https://` binds as a remote URL, anything else is read as the exId of a stored asset. Binding the URL is quicker and it renders immediately, which is why it is tempting and why it is wrong for anything but a placeholder you intend to replace in the same session. The user does not own that URL. When the source site expires the link, blocks hotlinking or goes away, the picture disappears from their live app, the editor still shows a binding that looks correct, and nothing in the project says what broke.

A VIDEO property behaves the same way. A FILE property has no URL form at all — it can only point at a stored asset, so a file must be imported before it can be bound.

### Two kinds of stored asset, and the `mode` that picks between them
One import produces one asset, but that asset is addressed by two different identifiers, and the two places you might put it want different ones. `mode` says which you are after:
- `STATIC` is the app's design — the picture is part of what you are building, the same for everyone who opens the app. It returns an `assetExId`, which is what a component binding stores. This is the mode for copying a page.
- `RUNTIME` is the app's data — the asset belongs in a row of a table, the way a product's photo does. It returns a numeric `assetId`, which is what an IMAGE/VIDEO/FILE column stores as `[col]_id` through `runtime.insert` or `runtime.update`.

Both come from the same copy, so a `STATIC` import is not wasted if the asset later has to go in a row — re-import it as `RUNTIME` and the ids agree. What does not work is spending one mode's identifier in the other's place: an exId in a media column and a number in a component binding are both accepted, and both render nothing without reporting an error.

Assets the app itself received — an end user's upload, a row edited in the data table — exist only in the running app. `runtime.query` on `fz_images` / `fz_videos` / `fz_files` lists them, and their ids are `RUNTIME` ids; the editor has no record of them, so none of them can be bound to a component.

This tool imports at BUILD time, while you are working. For an app that must import a file while it runs — a URL a user pastes, an image an AI node returned — the app needs the **Files** action flow node, which does the same job inside a flow and hands the following nodes a native image/video/file. Build that into the flow rather than importing the asset yourself; an import you run now happens once, on your machine's say-so, and the app still cannot do it for the next user.

## How to drive it (CLI)

No CLI verb yet — `web.fetch_page` and `web.import_asset` run inside the editor agent, which
reaches the public internet through a guarded fetcher (private addresses refused, redirects capped,
responses truncated at 10MB) and writes the imported bytes straight into the project's asset
storage.

From this CLI, bring an asset in through the editor, then reference the identifier its `mode`
returned: `STATIC` yields an `assetExId` for a component binding, `RUNTIME` a numeric
`assetId` for a table's `[col]_id`. They are not interchangeable — each is accepted in the
other's place and renders nothing.

For an app that must import a URL while it runs, build the **Files** action flow node into the
flow instead; it performs the same import at run time and yields a native image/video/file.

An imported asset is stored in the project, so it survives the source URL going away — binding a remote URL instead leaves the picture depending on a site nobody here controls.
