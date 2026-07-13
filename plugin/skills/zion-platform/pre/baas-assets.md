# BaaS runtime — binary asset upload

## Binary asset upload (mandatory two-step)
Assets (images/videos/files) live in object storage; tables reference only the asset id, never a
URL/path.
1. Compute the raw 128-bit MD5 of the file and Base64-encode it. Request a presigned URL with the
   mutation matching the type — imagePresignedUrl(imgMd5Base64, imageSuffix:MediaFormat, acl){ imageId
   uploadUrl uploadHeaders } / videoPresignedUrl(videoMd5Base64, videoFormat, acl){ videoId … } /
   filePresignedUrl(md5Base64, format, name, suffix, sizeBytes, acl){ fileId … }. acl defaults to
   PRIVATE (recommended).
2. HTTP PUT the raw bytes to uploadUrl with the returned uploadHeaders. Then use the returned id as
   the value of the corresponding *_id column (e.g. cover_image_id: imageId).

In browsers, crypto.subtle cannot compute MD5 — hash with a JS MD5 library (e.g. spark-md5)
instead. When rendering an asset on the frontend, select its url subfield; when storing or passing
it anywhere else, use the id.

MediaFormat: CSS, CSV, DOC, DOCX, GIF, HTML, ICO, JPEG, JPG, JSON, MOV, MP3, MP4, OTHER, PDF, PNG,
PPT, PPTX, SVG, TXT, WAV, WEBP, XLS, XLSX, XML.
CannedAccessControlList: AUTHENTICATE_READ, AWS_EXEC_READ, BUCKET_OWNER_FULL_CONTROL,
BUCKET_OWNER_READ, DEFAULT, LOG_DELIVERY_WRITE, PRIVATE, PUBLIC_READ, PUBLIC_READ_WRITE.

## Testing against the deployed backend (CLI)

This is a **runtime** spoke — it describes calling a DEPLOYED Zion app's SINGLE auto-generated
GraphQL API, which exposes ALL backend interactions (database, action flows, third-party APIs, AI
agents), not editing the editor schema. Endpoints (`{projectExId}` = the project's external id):
- HTTP (queries + mutations): https://zion-app.functorz.com/zero/{projectExId}/api/graphql-v2
- WebSocket (subscriptions):  wss://zion-app.functorz.com/zero/{projectExId}/api/graphql-subscription

Exercise runtime queries/mutations straight from this CLI — already authenticated with the admin token:

```bash
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" runtime graphql --args '{"query":"query { <root_op> { ... } }","variables":{}}'
"${PLUGIN_ROOT:-${CLAUDE_PLUGIN_ROOT}}/bin/zion-mcp" runtime query   --args '{"tableName":"post","where":{"id":{"_eq":1}},"limit":20,"fields":["id","title"]}'
```
`runtime graphql` sends **raw** GraphQL (use the operator-first `where` grammar in `baas-database.md`); `runtime query/insert/update/delete` are typed helpers that take the **simplified** `where` (see `schema-table.md`). Subscriptions (async action-flow results, AI streaming) run from your generated frontend over the WebSocket endpoint (legacy `subscriptions-transport-ws` framing — see `baas-database.md`) — this CLI does not open runtime subscriptions.

Store the returned asset id in an `IMAGE` / `VIDEO` / `FILE` column defined via `schema-table.md`.
