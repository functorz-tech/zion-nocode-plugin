# Static site hosting (deploy a built frontend)

## Static Site Domain Knowledge
A project can host one static site per **target**: `BETA` (a preview url) and `PROD` (the live
url). Each target holds exactly one active deployment; deploying again archives the previous one
and switches the edge routing over, so a deploy is a full replacement, not a merge — files removed
from the build output disappear from the site.

### The deployment protocol
A deployment is created from a **manifest** (every file's path, size, and md5), uploaded file by
file to object storage via short-lived presigned urls, then **completed**: the server re-reads
object storage and compares each object's checksum against the manifest before it switches the
site over. A deployment whose files did not all land stays in progress and serves nothing — the
previously active deployment keeps serving until a complete succeeds. This is why a failed deploy
never takes a site down.

Only one in-progress deployment per (project, target) is allowed; an interrupted one is either
resumed or abandoned before another can start. An in-progress deployment that goes **one hour**
without requesting an upload url is reclaimed (aborted, its uploaded objects deleted) by a
server-side sweep, so a deploy left half-finished overnight is gone the next morning — re-run it.

### Reading a result
- `siteUrl` — where the site is now served.
- `resumed: true` — an interrupted upload was continued rather than restarted.
- `skipped` — files in the directory that are not uploadable asset types. Always check this list;
  a missing asset at runtime is usually a file that was silently skipped here.
- `previousDeploymentExId` — the deployment that was archived by this one.

## How to drive it (CLI only)

No schema session — this ships files, it does not touch the project schema.

```bash
npx -y zion-mcp@2.6.0 project set-current --projectExId <exId>
npx -y zion-mcp@2.6.0 site deploy --dir ./dist --target BETA
```

`site deploy` does the whole protocol in one call: it scans the directory, declares the
manifest, uploads every asset, and switches the site over. Never assemble the individual
steps yourself.

| Intent | Command |
|---|---|
| Check a build output without deploying | `site deploy --dir ./dist --dryRun` |
| Deploy to the beta URL | `site deploy --dir ./dist --target BETA` |
| Take it live | `site deploy --dir ./dist --target PROD` |
| Where is the site / is an upload stuck? | `site status --target BETA` |
| Give up on a stuck upload | `site abort --target BETA` |

- **A multi-client project must name the web app**: pass `--appExId <exId>` on every `site` command,
  so that status and abort address the same site the deploy created. `project set-current` remembers
  only the project, so without it `site deploy` resolves to the project's backend-only app — which has
  no site domain — and fails with `UNSUPPORTED_APP_TYPE`. Read the exId from `project metadata` →
  `apps[]` (the one with `appType: WEB`); an exId from another project fails with `APP_NOT_FOUND`.
  Single-client projects need no flag.
- `--dryRun` reports `fileCount`, `totalBytes`, and `skipped` from the filesystem alone —
  no deployment is created. Use it to confirm the directory is the build output before deploying.
- `--target PROD` **takes the site live and requires a human to approve a GUI dialog**; the agent
  cannot self-approve, and `--force` does not approve it (`MCP_HITL=off` downgrades the gate to
  `--confirmProjectExId <exId>`). Deploy to BETA first and let the user check the beta URL.
- An interrupted run resumes automatically: re-run the identical command and only the files object
  storage is still missing are re-sent (`resumed: true`). If the directory changed since, the old
  deployment is discarded and a new one starts. `--force` always starts over.
- `--concurrency` (default 8) tunes parallel uploads.

## What the directory must contain

- **`index.html` at the root of `--dir`.** Without it the deploy is rejected. Point `--dir` at the
  build output (`./dist`, `./build`, `./out`), never at the project root.
- Only known web asset extensions are uploaded: html/js/mjs/cjs/css/map/json/webmanifest/txt/xml/
  csv/wasm/pdf, svg/png/jpg/jpeg/gif/webp/avif/ico/bmp, woff/woff2/ttf/otf/eot, mp4/webm/mp3/wav/ogg.
  **Anything else is silently skipped** and listed under `skipped` in the result — check it. Pre-compressed
  `.gz` / `.br` siblings are not uploadable, so turn build-time compression off.
- Per-file limit 2 GB. **Total size must fit the project's remaining object storage**, so a
  `QUOTA_EXCEEDED` error is about the plan's storage allowance, not a fixed deploy cap — report the
  limit from the error rather than guessing.

## What it is not

This deploys a **frontend you built yourself** (the output of a Vite / Next export / CRA build) against
the project's BaaS runtime — see `baas-database.md` for querying that project's data from the site's
code. It has nothing to do with the Zion-authored app: pages built with `ui-component.md` publish
through the normal deploy, not through here.
