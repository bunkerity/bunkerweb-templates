<p align="center">
  <img src="https://raw.githubusercontent.com/usekaneo/kaneo/main/apps/web/public/favicon.svg" alt="Kaneo logo" width="96" height="96" />
</p>

<h1 align="center">Kaneo</h1>

<p align="center">Task board template with WebSocket and task-text WAF tuning</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Kaneo-2.49.0-248651?labelColor=1d6f93&style=flat" alt="Kaneo 2.49.0" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#configure-kaneo-for-the-proxy">Configure Kaneo</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings-and-tradeoffs">Settings</a> ·
  <a href="#crs-exception-scope-and-cost">CRS exceptions</a> ·
  <a href="#validation">Validation</a> ·
  <a href="#things-to-think-about">Things to think about</a>
</p>

Put an existing [Kaneo](https://kaneo.app/) server behind BunkerWeb with ModSecurity and the OWASP
CRS on: task titles, descriptions and comments that hold SQL, shell and HTML, search, the board's
live updates over WebSocket, file and image uploads, the CLI's device login, MCP clients such as
Claude and ChatGPT, Git forge webhooks and single sign-on.

## At a glance

| Fact        | Value                                       |
| ----------- | ------------------------------------------- |
| Template id | `kaneo`                                     |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Kaneo 2.49.0 |
| Configs     | 2 (modsec-crs, server-http)                 |
| Rule ids    | `9860000` to `9860999`                      |
| Upstream    | `http://kaneo:5173`                         |

## Prerequisites

- Run Kaneo `2.49.0` or a close release (the all-in-one image `ghcr.io/usekaneo/kaneo`). BunkerWeb
  must reach `http://kaneo:5173` on a private network; do not publish that port. The image already
  proxies `/api/` to its own API on `127.0.0.1:1337`, so BunkerWeb talks to one port only.
- The first account created through the sign-up form becomes the instance administrator. Create it
  before the site is public.
- Kaneo needs PostgreSQL and an S3-compatible bucket for uploads. The browser uploads straight to
  the S3 endpoint with a presigned `PUT`; that request never passes BunkerWeb (confirmed with a
  Garage bucket: the BunkerWeb log shows `PUT /api/task/image-upload/<id>` and its `finalize` call,
  not the file).
- Point your public hostname at BunkerWeb. `AUTO_LETS_ENCRYPT=yes` makes BunkerWeb request its
  certificate (the default HTTP challenge needs inbound port 80). For existing certificate files,
  use `AUTO_LETS_ENCRYPT=no`, `USE_CUSTOM_SSL=yes`, `CUSTOM_SSL_CERT` and `CUSTOM_SSL_KEY`.
- Serve Kaneo at the root of the hostname: the template proxies `/` and was not tested under a
  sub-path.
- Do not put an authentication proxy (forward auth, basic auth) in front of `/api/mcp*`,
  `/.well-known/*` or the three webhook paths below. MCP clients send a bearer token, and GitHub,
  Forgejo or Gitea and GitLab call the webhook URLs without a session.

## Configure Kaneo for the proxy

Set these on the Kaneo container. The "without it" column is what was measured, or read in the
source of `v2.49.0` (`apps/api/src` in the Kaneo repository) when marked "source".

| Kaneo setting                | Value                                              | Without it                                                                                                                                                                                                                                                                                                                                      |
| ---------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `KANEO_CLIENT_URL`           | the public address, `https://kaneo.example.com`    | The WebSocket check compares the `Origin` header with this value and answers `403` otherwise; links in emails and the OAuth redirect addresses are built from it (source: `ws/security.ts`, `auth.ts`).                                                                                                                                       |
| `KANEO_API_URL`              | leave unset                                        | The bundled image derives it as `KANEO_CLIENT_URL` plus `/api`. A value that points elsewhere breaks sign-in with `Unexpected token '<'` errors (upstream issues 1454 and 1218).                                                                                                                                                                |
| `TRUSTED_PROXIES`            | BunkerWeb's own address only, for example `172.20.0.5/32`, never a shared private range | The default is loopback only. The app then sees BunkerWeb's address as the client of every request (measured: the stored address was `172.29.93.10`, the BunkerWeb container). With BunkerWeb's address listed, the stored address is the real client (`172.29.93.1`); with `none` it is `127.0.0.1`. See the security note below.            |
| `CORS_ORIGINS`               | only extra origins, if any                         | The browser app is same-origin and needs none. A web page on another origin needs `OPTIONS` allowed as well (see "Settings and tradeoffs").                                                                                                                                                                                                  |
| `AUTH_SECRET`                | a long random value, kept across container recreation | It signs the sessions: change it and every session ends. Not measured.                                                                                                                                                                                                                                                              |
| S3 endpoint and bucket       | an HTTPS endpoint the browser can reach, with a bucket CORS rule that allows `PUT`, the `Content-Type` header and your exact origin | Uploads fail in the browser with `NetworkError` (upstream issue 2191). The harness used Garage with that rule.                                                                                                                                                                                                                              |
| Registration switches        | as you need                                        | Anyone who can reach the sign-up form can create an account until you close it.                                                                                                                                                                                                                                                              |
| SMTP                         | as you need                                        | Not tested through this template (no mail server in the harness).                                                                                                                                                                                                                                                                          |
| `KANEO_ALLOW_PRIVATE_WEBHOOK_DESTINATIONS` | `true` only for a Forgejo or Gitea on a private network | Kaneo refuses to deliver to private addresses (source). Not tested.                                                                                                                                                                                                                                                                  |

Security: `TRUSTED_PROXIES` must name only BunkerWeb. The template sets
`REVERSE_PROXY_HEADERS=X-Forwarded-For $remote_addr`, so BunkerWeb replaces the header instead of
appending to what the client sent. That keeps a forged `X-Forwarded-For` out even when the trusted
range is wider than it should be (see the settings table). It does not help a client that reaches
the Kaneo port directly: keep that port private.

## Install

Import `template.json` and the files under `configs/` using the repository
[installation guide](../../README.md#install), or its plugin bundle layout. Assign
`USE_TEMPLATE=kaneo`, then change only `SERVER_NAME` and `REVERSE_PROXY_HOST` (keep the upstream
URL without a trailing slash or path). In the web UI raw import, the config dialog lists the files
as `modsec_crs/kaneo_false_positives.conf` and `server_http/kaneo_large_cookies.conf`: upload each
file under its listed name.

Before enabling blocking on your instance:

1. Set service `SECURITY_MODE=detect` and reload.
2. Sign in, create tasks and comments with code, SQL and markup, search, attach an image, open the
   board in two tabs, log in once with the CLI and connect one MCP client, then read the BunkerWeb
   logs.
3. Confirm the included exclusions match the targets in your logs. Narrow or add exceptions only
   where your traffic requires them.
4. Switch to `SECURITY_MODE=block` after those checks and repeat the workflows.

> [!IMPORTANT]
> False positives return `403` in block mode. The template removes `400` from the bad-behavior status
> list (see below); the remaining codes still ban a client after `BAD_BEHAVIOR_THRESHOLD` matching
> responses in 60 seconds (default 10, one day), so the detect run matters.

## Settings and tradeoffs

Each setting was put back to the BunkerWeb default (or removed) and the failure below reproduced.
Numbers come from the lane evidence for BunkerWeb `1.6.17` with the template as shipped.

| Setting                                          | Purpose and measurement                                                                                                                                                                                                                                                                                                                         |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `REVERSE_PROXY_URL=/`, `REVERSE_PROXY_WS=yes`   | Proxy the app and upgrade `/api/ws/user` and `/api/ws/<projectId>` (two WebSockets per tab). Without it the handshake answers `404`; with it twelve held sockets all got `101`. A socket whose client sends the app's 30 second ping stayed open for 140 seconds at the default 60 second timeout; one that sent nothing was closed at 60 seconds. |
| `REVERSE_PROXY_INTERCEPT_ERRORS=no`              | Keep Kaneo's JSON errors (`{"message":...,"code":...}`, the MCP `401` with `WWW-Authenticate`, the device-flow `{"error":"invalid_grant"}`). With `yes`, BunkerWeb swaps every upstream `400`, `401` and `404` for its HTML page (`content-type: text/html`), which the SPA and the CLI cannot parse.                                      |
| `ALLOWED_METHODS=GET\|POST\|HEAD\|PUT\|PATCH\|DELETE` | Task, comment, column, label and project updates use `PUT`; notifications and bulk edits use `PATCH`; deletes use `DELETE`. The default `GET\|POST\|HEAD\|QUERY` answered `405` to all three; with the setting all three got `200`.                                                                                                       |
| `LIMIT_REQ_RATE=100r/s`                          | BunkerWeb counts per exact URI. At the default `2r/s`, 30 requests in a burst to one URI got 2 times `200` and 28 times `429`, and 20 to `/api/mcp` got 2 and 18; with `100r/s` all pass. Distinct URIs were not limited (80 different requests all passed at `2r/s`). The browser smoke peaked at 5 requests per second to one URI (`GET /api/auth/get-session`), above the default. `429` also feeds the bad-behavior ban. |
| `LIMIT_CONN_MAX_HTTP1=200`                       | Every open tab holds two WebSockets, each on its own HTTP/1.1 connection. At the default `10`, the 11th and 12th socket from one address got `429`; with `200`, all twelve got `101`. Raise it further for an office behind one address.                                                                                                  |
| `MAX_CLIENT_SIZE=25m`                            | Matches the 25 MiB limit of the nginx inside the Kaneo image. A task import of 25 425 KiB passed and one of 25 825 KiB got `413`; at `10m`, a 24 023 KiB import got `413` (5 005 KiB passed).                                                                                                                                        |
| `MODSECURITY_REQ_BODY_NO_FILES_LIMIT=25m`        | Task descriptions have no size limit in the API. At the default `128 KiB`, a 300 KiB description got `400` (100 KiB passed); with `25m`, 8 000 KiB passed. The limit applies to every JSON body, so ModSecurity buffers up to that much per request.                                                                                           |
| `REVERSE_PROXY_HEADERS=X-Forwarded-For $remote_addr` | Replaces BunkerWeb's default `$proxy_add_x_forwarded_for`, which appends to the client's value. With `TRUSTED_PROXIES` set to BunkerWeb's `/32` both settings stored the real client. With a wider trusted range (`172.29.93.0/24` in the test) and the default append, a client `X-Forwarded-For: 6.6.6.6` was stored as `6.6.6.6`; with the template setting the same request stored the real client. |
| `BAD_BEHAVIOR_STATUS_CODES=401 403 404 405 429 444` | The default list also holds `400`. The CLI's device login polls `POST /api/auth/device/token` every 5 seconds and the pending answer is `400`. Fourteen polls: with the default list the 11th and later got `403` and the next page request was banned (`GET /api/health` got `403`); with this list all fourteen got `400` and `GET /api/health` got `200`. `401` stays in the list: failed sign-ins and MCP requests without a token still count. |
| `BLACKLIST_IGNORE_USER_AGENT=^openai-(?:mcp\|connectors-oauth)/` | ChatGPT's connector calls Kaneo with the user agents `openai-mcp/<version>` and `openai-connectors-oauth/<version>`, which BunkerWeb's default user agent blacklist denies. `POST /api/mcp` with each of them got `403` without the setting and `401` (the app's own answer) with it. Browsers and `curl` got `401` both times.        |
| `COOKIE_FLAGS_1=better-auth.last_used_login_method SameSite=Lax` | BunkerWeb's default `COOKIE_FLAGS=* HttpOnly SameSite=Lax` adds `HttpOnly` to every cookie. The better-auth sign-in cookie `better-auth.last_used_login_method` is set `HttpOnly` by default (measured), which hides it from the page script that shows the last sign-in method; the app sets the session cookie `HttpOnly` itself and that stays. With the setting the cookie is readable from JavaScript (`httpOnly=false` in the browser). The effect on the sign-in page itself was not compared. |
| `configs/server-http/kaneo_large_cookies.conf` (`large_client_header_buffers 4 32k`) | Single sign-on sets large cookies. With the nginx default a 9 000 byte `Cookie` header got `400` (7 000 passed); with the file 9 000 passed. A 20 000 byte header got `431` from the app itself, not from BunkerWeb.                                                                                                                                |

Measured and not needed, so not set: `REVERSE_PROXY_READ_TIMEOUT` (the app's own ping keeps the
sockets open, see the first row), `REVERSE_PROXY_BUFFERING` and the `PROXY_BUFFER*` settings (no
identity provider was available to produce a large `Set-Cookie` response header, so they were not
exercised). Kaneo's nginx and BunkerWeb both add `X-Content-Type-Options`, `X-Frame-Options`,
`Referrer-Policy` and `X-XSS-Protection`; through BunkerWeb each arrives once (counted on `/`,
`/api/health` and `/api/config`).

`OPTIONS` is not allowed. With the template, a preflight got `405` from BunkerWeb (the app answers
`204` directly). The browser app is same-origin and never sends one. Add `OPTIONS` to
`ALLOWED_METHODS` only for a web page on another origin, and list that origin in `CORS_ORIGINS`.

> [!NOTE]
> curl and some scripts send `Expect: 100-continue` on large bodies, which CRS rule `920450` blocks.
> Browsers do not. Add `-H 'Expect:'` to API calls with big bodies.

## CRS exception scope and cost

`configs/modsec-crs/kaneo_false_positives.conf` is a service-scoped **modsec-crs** config with 27
exclusions (ids `9860101` to `9860127`, inside BunkerWeb's block `9860000`-`9860999`). Every
exclusion is one chained rule: the path (anchored with `\A` and `\z`, `t:normalizePath`) and the
method. The request matrix recorded which rule matched which argument on the real app; only those
are removed. The text the rules cover is what people paste into a tracker: SQL statements, shell
pipelines, path traversal, HTML and script markup, PHP, Python, Node and Java code, XML, URLs, raw
HTTP requests and template syntax.

| Id | Method and path | Where | Rules |
| --- | --- | --- | --- |
| `9860101` | `POST /api/project` | `json.name` | 29 ids, by target |
| `9860102` | `PUT /api/project/<id>` | `json.description` (+ `REQUEST_BODY`, `RESPONSE_BODY`) | 53 ids, by target |
| `9860103` | `POST /api/task/<id>` | `json.description`, `json.title` (+ `REQUEST_BODY`, `RESPONSE_BODY`) | 53 ids, by target |
| `9860104` | `PUT /api/task/title/<id>` | `json.title` | 29 ids, by target |
| `9860105` | `PUT /api/task/description/<id>` | `json.description` (+ `REQUEST_BODY`, `RESPONSE_BODY`) | 52 ids, by target |
| `9860106` | `PUT /api/task/<id>` | `json.description`, `json.title` (+ `REQUEST_BODY`, `RESPONSE_BODY`) | 53 ids, by target |
| `9860107` | `POST /api/task/import/<id>` | the whole request body (by id) | 52 ids, by id |
| `9860108` | `POST/PUT /api/comment/<id>` | `json.content` (+ `REQUEST_BODY`, `RESPONSE_BODY`) | 52 ids, by target |
| `9860109` | `POST/PUT /api/activity/comment` | `json.comment` (+ `REQUEST_BODY`, `RESPONSE_BODY`) | 52 ids, by target |
| `9860110` | `GET /api/search` | `q` in the query string (+ `REQUEST_URI_RAW`, `RESPONSE_BODY`) | 47 ids, by target |
| `9860111` | `GET /api/task/description-matches/<id>` | `query` (query string) | 2 ids, by target |
| `9860112` | `GET /api/task/tasks/<id>` | `assigneeId`, `dueBefore`, `priority`, `status` | 12 ids, by target |
| `9860113` | `POST /api/label` | `json.name` | 29 ids, by target |
| `9860114` | `PUT /api/label/<id>` | `json.name` | 10 ids, by target |
| `9860115` | `POST/PUT /api/column/<id>` | `json.name` | 31 ids, by target |
| `9860116` | `POST /api/custom-field` | `json.name`, `json.options.array_0` to `array_9` | 31 ids, by target |
| `9860117` | `PUT /api/custom-field/value` | `json.value` (+ `REQUEST_BODY`, `RESPONSE_BODY`) | 52 ids, by target |
| `9860118` | `POST /api/time-entry` | `json.description` (+ `REQUEST_BODY`, `RESPONSE_BODY`) | 52 ids, by target |
| `9860119` | `PUT /api/time-entry/<id>` | `json.description` (+ `REQUEST_BODY`, `RESPONSE_BODY`) | 53 ids, by target |
| `9860120` | `POST /api/external-link/task/<id>` | `json.title` | 2 ids, by target |
| `9860121` | `POST /api/mcp/register` | `json.redirect_uris.array_0` to `array_9` | 2 ids, by target |
| `9860122` | `POST /api/github-integration/webhook` | the whole request body (by id) | 52 ids, by id |
| `9860123` | `POST /api/gitea-integration/webhook/<id>` | the whole request body (by id) | 52 ids, by id |
| `9860124` | `POST /api/gitlab-integration/webhook/<id>` | the whole request body (by id) | 52 ids, by id |
| `9860125` | `GET /api/mcp/authorize` | `redirect_uri` | 1 id, by target |
| `9860126` | `POST /api/mcp/token` | `redirect_uri` | 1 id, by target |
| `9860127` | `POST /api/mcp` | `json.params.arguments.content`, `.description`, `.q`, `.title` (+ `REQUEST_BODY`) | 52 ids, by target |

The counts are the rule ids in the file, each one seen on that argument in the matrix. Four rules read
more than the argument: `921110`, `944100`, `944110` and `944130` also read the raw JSON body
(`REQUEST_BODY`), `930100` and `930110` read the raw query (`REQUEST_URI_RAW`), and `953120` reads
the response (`RESPONSE_BODY`) because the app returns the text it just saved. Each is removed for
that one target on its own path only.

**Four exclusions remove whole rule ids for their path (marked "by id" above).** The task import
list (`json.tasks.array_<N>.*`) and the signed webhook bodies have member names that grow with the
payload. This BunkerWeb rejects regex targets in `ctl:` (a target with `^`, `\`, `+` or `|` fails
the config test, and a `/.../` target parses but never matches), so no target can be named. Those
entries use `ctl:ruleRemoveById`, with a comment above each, on those exact paths and the method
`POST`. The signature of a webhook covers the raw body, so the body cannot be changed, and issue text with
code in it would otherwise be blocked. If
you prefer a narrower rule, enumerate literal targets for the few indexes your users reach.

**Cost:** the excluded arguments (and, on the "by id" entries, the whole request body) are no longer
checked by the listed injection rules, so a stored-XSS or SQL injection payload in a task, comment,
label or search is not stopped by the WAF. Kaneo renders that text in the browser and sends it to
notification channels, so it has to handle it safely itself. Headers, cookies, other arguments,
other paths, other methods and every rule outside the lists stay active. `949110` and `959100` are
never excluded.

List members: Kaneo caps `redirect_uris` at 10 entries, so `json.redirect_uris.array_0` to
`array_9` cover every allowed index. A custom field's dropdown has no bound in the API; the
template covers the first ten options (`json.options.array_0` to `array_9`), because `ctl` cannot
name an index range or a pattern. An eleventh option that holds SQL or script text is blocked
(`403`, seen in the matrix at index 10).

Not covered, so a payload there is blocked: dropdown options from the eleventh on, other endpoints
the matrix did not exercise, and the same text in a field the table does not name. Add one
exclusion per argument your logs name.

## Validation

Tested on 2026-11-19: Kaneo `2.49.0` (`ghcr.io/usekaneo/kaneo:2.49.0@sha256:cd1826b1aba3ce736015cadf6f6c975d923aac00f4b6aaa7aedb29bc722dae49`, PostgreSQL 16,
Garage `v2.5.0` as the S3 service), `bunkerity/bunkerweb-all-in-one:1.6.17` (libmodsecurity
3.0.17, CRS 4.29.0), default paranoia level and anomaly thresholds, `SECURITY_MODE=block`, plain
HTTP on a local test host.

- **Request matrix** (final run on the shipped files): 147 requests that must succeed and 59
  controls, one at a time with their ModSecurity lines attributed per request. Without the config
  29 of the first 77 seeds got `403` with the rule ids in the file; each later round of seeds
  (more payload classes) was run until no seed was blocked. With the config all 147 pass and 59
  of 59 controls still got `403` (the same payload in another argument, path or method, a path
  suffix, a short id or a `..` traversal, a dropdown option or a redirect URI at index 10; and one line-feed row per
  path-anchored exclusion: the path prefixed with `/x%0a` gets `403` from rule `921190` as shipped, and from the lifted
  rule when `921190` is removed for the test).
- **Settings:** each non-default setting was switched back to the BunkerWeb default and the failure
  above reproduced.
- **Realtime:** WebSocket handshakes get `101` through BunkerWeb; the browser smoke showed a task
  created in one tab in a second tab without reload.
- **Browser smoke** (bad behavior on, template as shipped, only `SERVER_NAME` and
  `REVERSE_PROXY_HOST` set): the first administrator signs up through BunkerWeb, the workspace and
  first project are created, a task title with SQL, a description with SQL, a shell pipeline, a
  `<script>` tag and a URL and a comment with the same classes are saved, a second tab sees a new
  task, and an image attached to a task uploads through Garage and loads back. The BunkerWeb log
  for the 1 069 requests of the run shows 527 `200`, 534 `304` and 8 `101`; no ModSecurity
  denial, no ban.
- **Web UI import:** raw mode with `template.json` plus both config files imports, saves
  (10 steps, 16 settings, 2 configs) and is listed next to the plugin copy.

Not verified: see the list at the end of "Things to think about". Validate your own clients.

## Things to think about

- **MCP clients.** Claude Code, Cursor and ChatGPT use `/.well-known/oauth-*`, `/api/mcp/register`,
  `/api/mcp/authorize`, `/api/mcp/token` and `POST /api/mcp`. The full flow passed for a generic
  client and for ChatGPT's user agents. `localhost` redirect addresses of desktop clients trip rule
  `934110` and are excluded on the three OAuth paths. The tool call reply is one
  `text/event-stream` message, so no buffering setting is needed.
- **Device login of the CLI.** `kaneo login` polls every 5 seconds and the pending answer is `400`;
  the template keeps `400` out of the ban list for that reason.
- **Many people behind one address.** Each tab holds two WebSockets (`LIMIT_CONN_MAX_HTTP1`), and the board
  polls a few URIs several times a second (`LIMIT_REQ_RATE` counts per URI; the smoke peaked at 5). A
  `429` also feeds the bad-behavior ban, which bans the shared address. Raise both for offices.
- **Git forge webhooks.** `POST /api/github-integration/webhook`,
  `/api/gitea-integration/webhook/<id>` and `/api/gitlab-integration/webhook/<id>` verify a
  signature over the raw body. They pass the CRS with the by-id exclusions and answer the app's own
  `401` for a bad signature. Only replays with a computed signature were run, not a real forge.
- **Uploads.** The presigned `PUT` goes to the S3 endpoint, outside BunkerWeb. The download
  `GET /api/asset/<id>` streams through BunkerWeb and worked in the browser smoke.
- **25 MiB.** Bigger imports need both `MAX_CLIENT_SIZE` and
  `MODSECURITY_REQ_BODY_NO_FILES_LIMIT` raised, and Kaneo's own nginx limit too.
- **Board filters.** `GET /api/task/tasks/<projectId>?dueBefore=...` with a non-date value answers
  `500` from the app itself. That is an app bug, not a proxy matter.
- **Not verified:** a real OpenID Connect provider (only the callback request shapes were
  replayed), SMTP, a real GitHub, Forgejo or GitLab sending webhooks, the mobile app, the real
  `kaneo` CLI binary (a scripted device flow ran instead), large `Set-Cookie` response headers,
  HTTPS and HTTP/2, a sub-path deployment, several Kaneo replicas with Redis.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
