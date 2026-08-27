# WeChat mini program source upload (ship local mini program code)

## WeChat Mini Program Source Upload Domain Knowledge
This ships a mini program **you wrote yourself** to WeChat through Zion's third-party platform
authorization. The project must already have granted that authorization in the Zion editor —
without it every call fails with `THIRD_PARTY_AUTHORIZATION_REQUIRED`, and nothing here can grant
it.

### Where the code ends up
Zion commits the code to WeChat as the **trial version** (体验版) of the authorized mini program:
visible in 微信公众平台 → 版本管理 → 体验版, and openable by scanning the trial QR code. Nothing
is public. An upload never submits anything for review and never releases anything.

### The trial QR code
A successful build produces the trial QR code, which Zion fetches from WeChat and stores with the
deployment's artifacts. It is a JPEG, so it is written to a **file** rather than returned as text.
Scanning it only works from a phone whose WeChat id is bound as a **tester** on the mini program —
an unbound account sees a permission error, not the app. Binding testers is done in 微信公众平台
or the Zion editor, not from here.

### Where it stops
The CLI stops at the trial version. Submitting for review (提审) and releasing (发布) are done by
the user in the Zion editor: a submission needs product metadata, category and qualification
material, screenshots, and a privacy declaration that this CLI has no good way to collect, and it
is a billed action. So when the user asks to submit or publish, hand them back to the editor —
do not look for a CLI verb for it, there is none.

### The upload protocol
A deployment is created from a **manifest** (every file's path, size, and md5), uploaded file by
file to object storage via short-lived presigned urls, then **completed**: the server re-reads
object storage and compares each object's checksum against the manifest before it starts the
build. Only after that does it hand off to the build job that talks to WeChat, so a half-landed
upload never reaches WeChat at all.

A project holds at most **one unfinished deployment** — `PENDING`, `UPLOADING`, or `BUILDING`.
An interrupted upload (PENDING / UPLOADING) is either resumed or abandoned before another can
start. A `BUILDING` one is different: it already holds a template-app lock and **cannot be
aborted**, so while a build is running the only option is to wait for it.

### Files the server owns
`project.config.json` and `project.private.config.json` are written by the build job, because they
carry the template appid and the third-party upload settings that make the commit land on the right
mini program. A manifest that declares either one is rejected outright, so a local copy must be left
out of the upload — it is not a file you can override from here.

### Reading a result
- `qrCodePath` — where the trial QR code image was written. `trialVersion` /
  `trialVersionCommittedAt` identify the version that QR code opens; check them against what you
  just uploaded before telling the user it is ready. The QR code is read from the project's latest
  successful build, so a version number that does not match what you uploaded means another build
  finished after yours.
- `deploymentRecordExId` — the deployment record the build runs under; poll it to follow the build.
- `resumed: true` — an interrupted upload was continued rather than restarted.
- `skipped` — files that are not uploadable source types (and pruned directories such as
  `node_modules/`). Always check this list; a mini program that misbehaves after upload is usually
  a file that was silently skipped here.
- `serverManaged` — the `project.config.json` / `project.private.config.json` that were dropped
  because the server writes them. Seeing them listed is expected, not a problem.

## How to drive it (CLI only)

No schema session — this ships files, it does not touch the project schema.

```bash
npx -y zion-mcp@2.7.2 project set-current --projectExId <exId>
npx -y zion-mcp@2.7.2 wechat deploy --dir ./miniprogram   # upload, build, save the trial QR code
```

`wechat deploy` does the whole upload protocol in one call: it scans the directory, declares the
manifest, uploads every file, starts the build that commits the code to WeChat, waits for it, and
writes the trial QR code to a file. Never assemble the individual steps yourself.

| Intent | Command |
|---|---|
| Check a source tree without uploading | `wechat deploy --dir ./miniprogram --dryRun` |
| Upload, build, and get the trial QR code | `wechat deploy --dir ./miniprogram` |
| Re-fetch the trial QR code | `wechat qrcode --out ./trial.jpg` |
| Is the build done / is an upload stuck? | `wechat status` |
| Give up on a stuck upload | `wechat abort` |

- **A multi-client project must name the mini program app**: pass `--appExId <exId>` on every
  `wechat` command, so that status and abort address the same deployment the upload created.
  `project set-current` remembers only the project. Read the exId from `project metadata` →
  `apps[]` (the one with `appType: WECHAT_MINI_PROGRAM`); any other app type fails with
  `UNSUPPORTED_APP_TYPE`, and an exId from another project fails with `APP_NOT_FOUND`.
  Single-client projects need no flag.
- `--dryRun` reports `fileCount`, `totalBytes`, `skipped`, and `serverManaged` from the
  filesystem alone — no deployment is created. Use it to confirm the directory is the mini program
  root before uploading.
- `wechat deploy` **waits for the build by default** and returns `qrCodePath` (the saved trial
  QR image) plus `trialVersion`. `--qrOut` picks the file and
  `--timeoutSeconds` bounds the wait (default 900). On timeout it returns `timedOut: true`
  rather than failing — the build is still running, so poll `wechat status`. Pass
  `--no-wait` to return the moment the build starts instead.
- `ANOTHER_DEPLOYMENT_IN_PROGRESS` names the deployment that is blocking. If it is PENDING or
  UPLOADING, `wechat abort` clears it (or re-run the identical deploy to resume it). If it is
  **BUILDING there is nothing to clear — wait**, because a running build cannot be aborted.
- An interrupted run resumes automatically: re-run the identical command and only the files object
  storage is still missing are re-sent (`resumed: true`). If the directory changed since, the old
  deployment is discarded and a new one starts. `--force` always starts over.
- `--concurrency` (default 8) tunes parallel uploads.

### After the trial version

Hand the user the QR code and stop. Review (提审) and release (发布) are done by them in the
Zion editor — there is no CLI verb for either, and asking for one is not an oversight to work
around. If they ask you to submit or publish, tell them where to go rather than reaching for
`runtime graphql` to do it behind the editor's back: a submission carries product metadata,
category and qualification material, screenshots, and a privacy declaration that only the editor
collects, and it is billed.

## What the directory must contain

- **`app.json` at the root of `--dir`.** Without it the upload is rejected. Point `--dir` at the
  directory WeChat DevTools opens as the project — for a compiling framework (Taro, uni-app) that
  is the built output (e.g. `./dist/weapp`), not the framework source.
- Only known mini program source extensions are uploaded: js/ts/json/wxml/wxss/wxs, png/jpg/jpeg/
  gif/svg/webp/ico, ttf/otf/woff/woff2, mp3/mp4, wasm/cer/txt. **Anything else is silently
  skipped** and listed under `skipped` — check it.
- `node_modules/`, `.git/`, `.idea/`, and `.vscode/` are pruned whole and reported in
  `skipped`. `miniprogram_npm/` is **not** pruned — that is the built npm output the mini program
  actually runs, so build it before uploading.
- `project.config.json` and `project.private.config.json` are dropped and reported under
  `serverManaged`: the build job writes its own, carrying the template appid and upload settings.
  Editing yours changes nothing here.
- Limits are 5000 files and 20 MiB total across the upload. These are Zion's limits on the source
  package; WeChat's own compiled main-package limit is separate and is enforced later, by WeChat,
  during the build.

## What it is not

This uploads a mini program **you wrote yourself** against the project's BaaS runtime — see
`baas-database.md` for querying that project's data from mini program code. It has nothing to do
with the Zion-authored mini program: pages built with `ui-component.md` publish through the
normal deploy, not through here. And it stops at the trial version: review and release are the
Zion editor's job.
