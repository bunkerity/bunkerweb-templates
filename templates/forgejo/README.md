<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/forgejo.svg" alt="Forgejo logo" width="96" height="96" />
</p>

<h1 align="center">Forgejo</h1>

<p align="center">Git forge template with HTTPS, LFS and package API tuning</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Forgejo-16.0.5-248651?labelColor=1d6f93&style=flat" alt="Forgejo 16.0.5" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#crs-exception-scope-and-cost">CRS exceptions</a> ·
  <a href="#validation">Validation</a>
</p>

Proxy Forgejo at a dedicated hostname, including Git smart HTTP, LFS, generic packages, the
viewing of repository files and the forms and API routes where you type code or shell commands
(issues, comments, pull requests, releases, wiki pages, file edits). It does not install Forgejo
or configure SSH. Read [CRS exception scope and cost](#crs-exception-scope-and-cost) before
importing.

## At a glance

| Fact        | Value                         |
| ----------- | ----------------------------- |
| Template id | `forgejo`                     |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Forgejo 16.0.5 |
| Configs     | 2 (modsec-crs)                |
| Rule ids    | `9856000` to `9856999`        |
| Upstream    | `http://forgejo:3000`         |

## Prerequisites

1. Run Forgejo on a network reachable by BunkerWeb. The default upstream is
   `http://forgejo:3000`; change its hostname and port for your deployment.
2. Set Forgejo's `[server] ROOT_URL` to the public URL, for example
   `https://forgejo.example.com/`, including any public nonstandard port. Generated LFS links
   depend on this URL. See the upstream [configuration reference](https://forgejo.org/docs/latest/admin/config-cheat-sheet/#server-server)
   and [reverse-proxy guide](https://forgejo.org/docs/latest/admin/setup/reverse-proxy/).
3. Git LFS is off by default in Forgejo. To use LFS through this template, set
   `LFS_START_SERVER = true` in the `[server]` section of `app.ini`, or the environment variable
   `FORGEJO__server__LFS_START_SERVER=true`. Without it, LFS calls get a Forgejo `404` page, not a
   WAF block.

## Install

1. Follow the repository [installation guide](../../README.md#install). For Raw
   import, upload `template.json` and both referenced configs when prompted. For a plugin bundle,
   preserve the guide's template/config layout.
2. In the web UI, create the service in easy mode and pick **Forgejo** in the **Templates** list.
   Replace the `forgejo.example.com` server name with yours, go through the four guided steps and
   save the service: BunkerWeb applies it on save. Automatic Let's Encrypt (step 1) requires public
   DNS and HTTP-01 reachability. For a LAN or test host, turn it off in step 1, save, then switch
   the service to Advanced mode: the **Custom SSL certificate** tab takes your certificate and the
   **Self-signed certificate** tab generates one for local testing.

> [!NOTE]
> The recipe serves the app at `/`. Subpath hosting and encoded-slash edge cases require separate
> validation against Forgejo's proxy guide; they were not exercised in the source tests.

## Settings

- `ALLOWED_METHODS=GET|POST|HEAD|OPTIONS|PUT|PATCH|DELETE` supports API edits, LFS and packages.
  It replaces the default list, dropping `QUERY`. Append it if needed; `OPTIONS` was not exercised.
- `MAX_CLIENT_SIZE=100m` raises the service body limit for pushes, release assets and attachments.
  Historical tests used 21–25 MB bodies, not the full 100 MiB limit. Choose a finite limit above
  your largest request and use LFS for large Git files. Larger accepted bodies increase buffering,
  temporary-disk and inspection costs. This does not raise the separate ModSecurity non-file
  body limit: large JSON/base64 API file writes can still exceed its default 128 KiB.
- Package and LFS uploads must send a `Content-Type` header. A `PUT` without one is blocked
  (`403`, CRS `920340` and `920640`), and `curl -T file` sends none: add
  `-H "Content-Type: application/octet-stream"`. Check what your own client sends. This template
  ships no exception for it.
- `USE_LIMIT_REQ=yes`, `LIMIT_REQ_URL=/`, `LIMIT_REQ_RATE=20r/s` retain request limiting while
  allowing repository-fetch and login bursts. Raising the rate reduces throttling protection.
- `SERVE_FILES=no` keeps local file serving off. The historical `/user/events` SSE flow passed
  with default proxy settings; no WebSocket or buffering override is included.
- BunkerWeb's default bad-behavior policy can ban a client after repeated authentication,
  missing-resource or rate-limit errors. Review logs while validating; protection remains enabled.

## CRS exception scope and cost

`configs/modsec-crs/forgejo_protocol.conf` loads before the CRS:

- Rule `9856100` removes only target `REQUEST_FILENAME` from CRS `930130` for `GET|POST|PUT` on
  `/<owner>/<repo>.git/info/refs`, `git-upload-pack`, `git-receive-pack` and `info/lfs/...`.
- Rule `9856101` removes content-type rule `920420` for `POST|PUT` on the Git pack/LFS paths and
  `/api/packages/<owner>/...`. Any content type is accepted by that rule on these requests;
  this is not restricted to known Git/LFS MIME types and does not add a body parser.
- Rule `9856102` removes extension rule `920440` for `GET|HEAD|POST|PUT|PATCH|DELETE` on
  `/api/v1/repos/<owner>/<repo>/{contents,raw}/...` and
  `/<owner>/<repo>/{src,raw,blame,_new,_edit,_delete,_upload,commits}/...`.
  Other fields on the matched request also lose this rule's coverage.
- Rule `9856105` lets you view repository files: `GET|HEAD` on `/<owner>/<repo>/{src,raw,blame}/...`
  and `/api/v1/repos/<owner>/<repo>/{raw,contents}/...`. A stored file can hold text that the CRS
  takes for a leak when it appears in a response: PHP source (`<?php`), SQL or Java error text,
  stack traces, a directory listing. Without the rule, such a file is cut or returns `403`
  (`953120`, `951230`, `952110` and others, then `959100`). The rule removes every CRS rule tagged
  `attack-disclosure` (response data leakage) and the path check of rule `930130` (files named
  `.env.example` look like restricted files) on these requests. **Cost:** on file views the CRS no
  longer reads the response for data leakage. The request itself is still checked in full, and
  other pages, other methods and plain-text patches (`.diff`, `.patch`) are not covered.

`configs/modsec-crs/forgejo_false_positives.conf` expands all tested literal fields and tags:

- Rule `9856103`: `POST|PUT|PATCH` on `/api/v1/repos/<owner>/<repo>/{issues,pulls,contents}`
  and their subpaths; fields `json.body`, `json.title`, `json.content`, `json.message`.
- Rule `9856104`: `POST` on `/<owner>/<repo>/issues/new`,
  `/<owner>/<repo>/{issues,pulls}/<number>/comments`, `_new/...` and `_edit/...`;
  fields `content`, `title`, `commit_summary`, `commit_message`.
- Rule `9856106`: `POST` on `/<owner>/<repo>/issues/<number>/content`,
  `/<owner>/<repo>/issues/<number>/title`, `/<owner>/<repo>/comments/<id>` (comment edit),
  `/<owner>/<repo>/compare/...` (new pull request), `/<owner>/<repo>/releases/new` and
  `releases/edit/...`, `/<owner>/<repo>/wiki` and `wiki/...` (new and edit), the review comment and
  review submit routes of a pull request, and `/<owner>/<repo>/markup` (the editors' preview);
  fields `content`, `title`, `message`, `text`.
- Rule `9856107`: `POST|PATCH` on `/api/v1/repos/<owner>/<repo>/releases` and
  `/api/v1/repos/<owner>/<repo>/releases/<id>`; fields `json.body`, `json.name`.

Those fields skip every rule tagged `attack-rce`, `attack-lfi`, `attack-xss` or
`attack-injection-php`. Forgejo must enforce authorization and safely render/store submitted code.
SQLi rules remain active. Literal `ARGS` targets can also match query parameters with those exact
names on an allowed write request. These are service-scoped exceptions, not authentication checks.

## Validation

On 2026-11-20 (Forgejo 16.0.5 with SQLite, BunkerWeb 1.6.17 all-in-one, CRS 4.29.0, block mode) the
files of this template were imported through the web UI behind the CRS (`POST /templates/create`:
the JSON alone, each config alone and everything together, as Save sends it: `200`, no rule
fired) and the service was created in easy mode from the template. Request matrices ran against
the service before and after the file-view and edit rules:

- Shell commands, `../` and `<?php` in an issue body or title, a comment, a new pull request, a
  release (web form and API), a wiki page (new and edit), a review comment and the editor preview
  answered `403` before (CRS `932230`, `932235`, `932125`, `932250`, `933100`) and `200`, `201` or
  `303` after. Each of the four fields is needed on its routes: removing one from the rule gave `403`
  again on the route that uses it.
- 139 repository files (113 real files from public repositories: PHP, SQL dumps, Java stack traces,
  `.env.example`, docs quoting database and PHP errors, plus 26 small files holding one error text
  per database, language or server), each through `raw`, `src`, `blame`, the API `raw` and the API
  `contents` routes (695 requests): 432 were cut or `403` before, none after, and the `raw` and
  API `raw` bodies were byte-equal to what Forgejo returns directly. A list of the 13 rule ids that
  fired on the real files still left 7 of the 26 other files blocked (`951110`, `951130`, `951150`,
  `951170`, `951190`, `951260`, `954130`), so the rule uses the `attack-disclosure` tag instead.
- Still blocked after the change: the same SQL error text in an issue or a wiki page (`951230`),
  `<script>`, SQL injection, a traversal and a scanner user agent on the file-view routes,
  `/.git/config` and `.env` outside a file view, shell text in a field or route the rules do not
  list, and SQL injection in a listed field.
- Not covered: plain-text patches (`/commit/<sha>.diff`, `.patch`) of a file that holds such text,
  the LFS `media/` route, and editing a file named `.env` (`_edit` is still checked by `930130`).
- Package and LFS object uploads without a `Content-Type` answer `403` (`920340`, `920640`) and
  work with `application/octet-stream`.

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

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
