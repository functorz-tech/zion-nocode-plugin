# BaaS runtime — binary asset upload

## Binary asset upload (mandatory two-step)
Assets (images/videos/files) live in object storage; tables reference only the asset id, never a
URL/path.
1. Compute the raw 128-bit MD5 of the file and encode it as standard Base64 — the server decodes
   with the basic decoder, so a URL-safe variant is rejected. Request a presigned URL with the
   QUERY field matching the type — imagePresignedUrlV2(imgMd5Base64, imageSuffix:MediaFormat,
   acl){ imageId uploadUrl uploadHeaders } / videoPresignedUrlV2(videoMd5Base64, videoFormat,
   acl){ videoId … } / filePresignedUrlV2(md5Base64, format, name, suffix, sizeBytes, acl){ fileId
   … }. acl defaults to PRIVATE (recommended). The same fields without the V2 suffix are
   deprecated mutations — do not use them in new code.
2. HTTP PUT the raw bytes to uploadUrl with the returned uploadHeaders. Then use the returned id as
   the value of the corresponding *_id column (e.g. cover_image_id: imageId).

In browsers, crypto.subtle cannot compute MD5 — hash with a JS MD5 library (e.g. spark-md5)
instead. When rendering an asset on the frontend, select its url subfield; when storing or passing
it anywhere else, use the id.

MediaFormat: CSS, CSV, DOC, DOCX, GIF, HTML, ICO, JPEG, JPG, JSON, MOV, MP3, MP4, OTHER, PDF, PNG,
PPT, PPTX, SVG, TXT, WAV, WEBP, XLS, XLSX, XML.
CannedAccessControlList: AUTHENTICATE_READ, AWS_EXEC_READ, BUCKET_OWNER_FULL_CONTROL,
BUCKET_OWNER_READ, DEFAULT, LOG_DELIVERY_WRITE, PRIVATE, PUBLIC_READ, PUBLIC_READ_WRITE.

## Images inside rich-text (RTF) content
A rich-text column stores HTML whose <img> src is an asset placeholder, never a URL:
<img src="fz_image_1020000000002458">. The suffix is either the numeric image id or its
11-char base62 exId; both address the same asset. Asset URLs are presigned and expire, so a
URL written into the body rots — the app's rich-text / Markdown components resolve
placeholders to live URLs at render time, which is why the column read through this API (or
shown in the data table) still holds the raw placeholder.

Reading one from outside the app means resolving them yourself: match
fz_image_([0-9]+|[A-Za-z0-9]{11}) over the HTML, dedupe, batch-resolve with
getImageListByIds(imageIds:_int8){ id exId url urlInfo{ urlExpireAt } } — exIds go through
getImageListByExIds(imageExIds:[String]) — then rewrite each src. Cache by urlExpireAt and
never write a resolved URL back into the column. Writing is the mirror image: upload through
the two-step above, then emit src="fz_image_<imageId>". An external URL left in the body is
stored verbatim and leaves the picture depending on a host nobody here controls.

## Testing against the deployed backend (CLI)

This is a **runtime** spoke — it describes calling a DEPLOYED Zion app's SINGLE auto-generated
GraphQL API, which exposes ALL backend interactions (database, action flows, third-party APIs, AI
agents), not editing the editor schema. Endpoints (`{projectExId}` = the project's external id):
- HTTP (queries + mutations): https://zion-app.functorz.com/zero/{projectExId}/api/graphql-v2
- WebSocket (subscriptions):  wss://zion-app.functorz.com/zero/{projectExId}/api/graphql-subscription

Exercise runtime queries/mutations straight from this CLI — already authenticated with the admin token:

```bash
npx -y zion-mcp@2.7.6 runtime graphql --args '{"query":"query { <root_op> { ... } }","variables":{}}'
npx -y zion-mcp@2.7.6 runtime query   --args '{"tableName":"post","where":{"id":{"_eq":1}},"limit":20,"fields":["id","title"]}'
```
`runtime graphql` sends **raw** GraphQL (use the operator-first `where` grammar in `baas-database.md`); `runtime query/insert/update/delete` are typed helpers that take the **simplified** `where` (see `schema-table.md`). Subscriptions (async action-flow results, AI streaming) run from your generated frontend over the WebSocket endpoint (legacy `subscriptions-transport-ws` framing — see `baas-database.md`) — this CLI does not open runtime subscriptions.

Store the returned asset id in an `IMAGE` / `VIDEO` / `FILE` column defined via `schema-table.md`.
