# Forgejo Template

Proxy Forgejo at a dedicated hostname, including Git smart HTTP, LFS, generic packages and
code-bearing issue/file forms. This template extracts a historically tested recipe; it does not
install Forgejo or configure SSH.

## Prerequisites and import

1. Run Forgejo on a network reachable by BunkerWeb. The default upstream is
   `http://forgejo:3000`; change its hostname and port for your deployment.
2. Set Forgejo's `[server] ROOT_URL` to the public URL, for example
   `https://forgejo.example.com/`, including any public nonstandard port. Generated LFS links
   depend on this URL. See the upstream [configuration reference](https://forgejo.org/docs/latest/admin/config-cheat-sheet/#server-server)
   and [reverse-proxy guide](https://forgejo.org/docs/latest/admin/setup/reverse-proxy/).
3. Follow the repository [installation guide](../../README.md#installing-templates). For Raw
   import, upload `template.json` and both referenced configs when prompted. For a plugin bundle,
   preserve the guide's template/config layout.
4. Select `forgejo` for the service (or set `USE_TEMPLATE=forgejo`), change `SERVER_NAME`, review
   every guided step and reload. Automatic Let's Encrypt requires suitable DNS and HTTP-01
   reachability; otherwise set `AUTO_LETS_ENCRYPT=no` and configure BunkerWeb's `USE_CUSTOM_SSL`
   and certificate settings, or `GENERATE_SELF_SIGNED_SSL` for local testing.

The recipe serves the app at `/`. Subpath hosting and encoded-slash edge cases require separate
validation against Forgejo's proxy guide; they were not exercised in the source tests.

## Settings and limits

- `ALLOWED_METHODS=GET|POST|HEAD|OPTIONS|PUT|PATCH|DELETE` supports API edits, LFS and packages.
  It replaces the default list, dropping `QUERY`. Append it if needed; `OPTIONS` was not exercised.
- `MAX_CLIENT_SIZE=100m` raises the service body limit for pushes, release assets and attachments.
  Historical tests used 21–25 MB bodies, not the full 100 MiB limit. Choose a finite limit above
  your largest request and use LFS for large Git files. Larger accepted bodies increase buffering,
  temporary-disk and inspection costs. This does not raise the separate ModSecurity non-file
  body limit: large JSON/base64 API file writes can still exceed its default 128 KiB.
- `USE_LIMIT_REQ=yes`, `LIMIT_REQ_URL=/`, `LIMIT_REQ_RATE=20r/s` retain request limiting while
  allowing repository-fetch and login bursts. Raising the rate reduces throttling protection.
- `SERVE_FILES=no` keeps local file serving off. The historical `/user/events` SSE flow passed
  with default proxy settings; no WebSocket or buffering override is included.
- BunkerWeb's default bad-behavior policy can ban a client after repeated authentication,
  missing-resource or rate-limit errors. Review logs while validating; protection remains enabled.

## CRS exception scope and cost

`configs/modsec-crs/forgejo_protocol.conf` loads before the CRS:

- Rule `77300` removes only target `REQUEST_FILENAME` from CRS `930130` for `GET|POST|PUT` on
  `/<owner>/<repo>.git/info/refs`, `git-upload-pack`, `git-receive-pack` and `info/lfs/...`.
- Rule `77301` removes content-type rule `920420` for `POST|PUT` on the Git pack/LFS paths and
  `/api/packages/<owner>/...`. Any content type is accepted by that rule on these requests;
  this is not restricted to known Git/LFS MIME types and does not add a body parser.
- Rule `77302` removes extension rule `920440` for `GET|HEAD|POST|PUT|PATCH|DELETE` on
  `/api/v1/repos/<owner>/<repo>/contents/...` and
  `/<owner>/<repo>/{src,raw,blame,_new,_edit,_delete,_upload,commits}/...`.
  Other fields on the matched request also lose this rule's coverage.

`configs/modsec/forgejo_false_positives.conf` expands all tested literal fields and tags:

- Rule `77303`: `POST|PUT|PATCH` on `/api/v1/repos/<owner>/<repo>/{issues,pulls,contents}`
  and their subpaths; fields `json.body`, `json.title`, `json.content`, `json.message`.
- Rule `77304`: `POST` on `/<owner>/<repo>/issues/new`,
  `/<owner>/<repo>/{issues,pulls}/<number>/comments`, `_new/...` and `_edit/...`;
  fields `content`, `title`, `commit_summary`, `commit_message`.

Those fields skip every rule tagged `attack-rce`, `attack-lfi`, `attack-xss` or
`attack-injection-php`. Forgejo must enforce authorization and safely render/store submitted code.
SQLi rules remain active. Literal `ARGS` targets can also match query parameters with those exact
names on an allowed write request. These are service-scoped exceptions, not authentication checks.

## Validation and historical evidence

On 2026-11-03, the packaged template was imported through a local test plugin into BunkerWeb
`:tests` (1.6.18-rc2, libmodsecurity 3.0.17, CRS 4.30.0). Applied defaults matched the JSON and
nginx/ModSecurity loading passed. Echo-backend probes passed for Git discovery, Git content
types, source-file extensions and an API code field; paired other-path/field requests stayed
blocked. This does not prove Git/LFS transfers or application behavior on your installed version.

Historical scripted tests: **2026-10-06**, **Forgejo 16.0.5** with SQLite,
**BunkerWeb 1.6.15**, self-signed TLS. Final block-mode tests passed pages/login, API repository,
issue/comment/contents edits, web code forms, HTTPS clone and a 22 MB push/re-clone with matching
hashes, LFS batch/upload/verify/download, generic package PUT/GET/DELETE, 25 MB release assets,
21 MB attachments, SSE, archives and raw files. Initial detect-mode LFS had a script error;
the final block-mode run is the exclusion evidence.

Historical negative controls blocked `.git/config`, wrong methods, other paths, other JSON/form
fields, wiki content and SQLi in an excluded field. Recheck these after upgrades. Not tested:
SSH, Actions/runners, webhooks, OAuth/OIDC, non-generic package registries, mirrors/migrations,
wiki workflows or real client-IP forwarding. Validate the flows you use after importing.
