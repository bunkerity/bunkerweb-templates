<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/outline.svg" alt="Outline logo" width="96" height="96" />
</p>

<h1 align="center">Outline</h1>

<p align="center">Wiki template with realtime collaboration and MCP support</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Outline-1.12.0-248651?labelColor=1d6f93&style=flat" alt="Outline 1.12.0" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#configure-outline-for-the-proxy">Configure Outline</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#crs-exception-scope-and-cost">CRS exceptions</a> ·
  <a href="#validation">Validation</a> ·
  <a href="#things-to-think-about">Things to think about</a>
</p>

Put an existing [Outline](https://www.getoutline.com/) wiki behind BunkerWeb with ModSecurity and the
OWASP CRS on: document editing with code, SQL and HTML text, realtime collaboration, attachments,
imports and exports, search, the Slack `/outline` command, passkey and email sign-in, and the `/mcp`
endpoint and OAuth flow that MCP clients (ChatGPT, Claude, Cursor) use.

## At a glance

| Fact        | Value                                         |
| ----------- | --------------------------------------------- |
| Template id | `outline`                                     |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Outline 1.12.0 |
| Configs     | 1 (modsec-crs)                                |
| Rule ids    | `9867000` to `9867999`                        |
| Upstream    | `http://outline:3000`                         |

## Prerequisites

- Run Outline `1.12.0` or a close release with PostgreSQL and Redis. BunkerWeb must reach
  `http://outline:3000` on a private network. Do not publish that port: Outline has no list of
  trusted proxy addresses, so anyone who can reach port 3000 directly can forge the client IP and
  scheme (see the next section).
- Outline has no local passwords. Configure an authentication provider (OIDC, Google, Slack, Azure,
  Discord, passkeys, email). Email sign-in only works for invited users who have no SSO account, so
  the first workspace needs one of the other providers.
- Point your public hostname at BunkerWeb. `AUTO_LETS_ENCRYPT=yes` makes BunkerWeb request its
  certificate (the default HTTP challenge needs inbound port 80). For existing certificate files,
  use `AUTO_LETS_ENCRYPT=no`, `USE_CUSTOM_SSL=yes`, `CUSTOM_SSL_CERT` and `CUSTOM_SSL_KEY`.

## Configure Outline for the proxy

Set these in Outline's environment. BunkerWeb sends `Host`, `X-Forwarded-For` (appended to what the
client sent), `X-Real-IP` (the TCP peer, always replaced), `X-Forwarded-Proto` (always its own scheme)
and `X-Forwarded-Host` (with the port).

| Outline variable                                                 | Value                         | What breaks without it                                                                                                                                                                                                  |
| ---------------------------------------------------------------- | ----------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `URL`                                                            | `https://outline.example.com` | Sign-in redirects, cookies, the CSP and the WebSocket origin are built from it. No trailing slash. Hosting under a sub-path is not supported upstream ([issue 9161](https://github.com/outline/outline/issues/9161)).    |
| `PROXY_IP_HEADER`                                                | `X-Real-IP`                   | Outline reads the first `X-Forwarded-For` entry, which a client can supply. See below.                                                                                                                                  |
| `PROXY_HEADERS_TRUSTED`                                          | `true` (the default)          | With `false` Outline ignores the forwarded scheme and IP: every user appears as BunkerWeb, and its CSRF cookie falls back to a weaker form (Outline logs a warning, `server/services/web.ts:34-46`).                                                                   |
| `FORCE_HTTPS`                                                    | `true` (the default)          | Keep it when BunkerWeb serves HTTPS on port 443. Set `false` when another proxy ends TLS in front of BunkerWeb or HTTPS uses a custom port (see below).                                                                 |
| `FILE_STORAGE_UPLOAD_MAX_SIZE`                                   | `26214400`                    | Unset, Outline accepts attachments up to 1,000,000 bytes only (`server/env.ts:748-751`). `26214400` (25 MiB) fits the template's `MAX_CLIENT_SIZE=26m`.                                                                |
| `FILE_STORAGE_IMPORT_MAX_SIZE`, `FILE_STORAGE_WORKSPACE_IMPORT_MAX_SIZE` | `26214400` or lower   | Both fall back to the upload limit (`server/env.ts:758-762`, `775-779`). A value above `MAX_CLIENT_SIZE` makes BunkerWeb answer `413` first.                                                                            |
| `RATE_LIMITER_*`                                                 | keep the defaults             | Outline's own limiter is on by default at 1000 requests per 60 seconds (`server/env.ts:570-582`). Signed-in browser sessions are keyed on the user id, so people behind one address do not share a bucket.             |

**Client IP.** BunkerWeb's `X-Forwarded-For` is the client's own value plus the peer address, and
Outline takes the first entry (Koa `app.proxy`, `server/services/web.ts:31-37`). Outline uses that IP
for the audit log, the last-active IP, its rate limiter for unauthenticated and API-key requests
(`server/middlewares/rateLimiter.ts:30-52`) and to bind an email sign-in link to the address that
asked for it (`server/utils/jwt.ts:138`). Measured through BunkerWeb:

- A request with `X-Forwarded-For: 9.9.9.9` stored `9.9.9.9` as the IP of the audit event; without the
  header it stored the real address.
- With Outline's limiter at 5 requests per minute, 8 sign-in-page requests with 8 different forged
  addresses all got `200`; 8 requests from one address got `200` five times, then `429`.
- With `PROXY_IP_HEADER=X-Real-IP` the stored IP stayed the real one even with both `X-Real-IP` and
  `X-Forwarded-For` forged, and the 8 forged requests shared one bucket (five `200`, then `429`).

> [!NOTE]
> Keep `USE_REAL_IP` at its default `no` on BunkerWeb unless BunkerWeb itself sits behind another proxy
> or CDN, with a matching `REAL_IP_FROM`.

**HTTPS.** With `FORCE_HTTPS=true`, BunkerWeb's HTTPS port answered `200`. A plain HTTP `GET` got
Outline's `301` to `https://host/` (the port of a custom public port is dropped) and a plain HTTP
`POST` got `405`. A forged `X-Forwarded-Proto: https` over HTTP changed nothing, because BunkerWeb
always sets that header from its own connection. So when another proxy terminates TLS in front of
BunkerWeb, Outline sees `http`: set `FORCE_HTTPS=false` there and let that proxy redirect. Over HTTPS
Outline names its CSRF cookie `__Host-csrfToken` and BunkerWeb adds `Secure`.

## Install

Import `template.json` and `configs/modsec-crs/outline_false_positives.conf` using the repository
[installation guide](../../README.md#install), or its plugin bundle layout. Assign
`USE_TEMPLATE=outline`, then change only `SERVER_NAME` and `REVERSE_PROXY_HOST` (keep the upstream
URL without a trailing slash or path). In the web UI raw import, the template name must be unique,
and the config dialog lists the file as `modsec_crs/outline_false_positives.conf`: upload
`outline_false_positives.conf` there and press Done before Save.

Before enabling blocking on your instance:

1. Set service `SECURITY_MODE=detect` and reload.
2. Sign in, write documents with code blocks and tables, comment, search for code, open a second
   tab, upload an attachment, and connect an MCP client while reading BunkerWeb logs.
3. Confirm the included exclusions match the targets in your logs. Narrow or add exceptions only
   where your traffic requires them.
4. Switch to `SECURITY_MODE=block` after those checks and repeat the workflows.

False positives return `403` in block mode. The default bad-behavior status list includes
`400 401 403 404 405 429 444`: about ten matching responses in 60 seconds ban the client for one
day, so the detect run matters. The unauthenticated `401` Outline answers before sign-in counts
as one hit and decays after about a minute.

## Settings

| Setting                                                       | Purpose                                                                                                                                                                                                                        |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `REVERSE_PROXY_URL=/`, `REVERSE_PROXY_WS=yes`                 | Proxy the app and upgrade `/realtime` (Socket.IO) and `/collaboration` (live editing). Without it both answer `400`/`200`.                                                                                                       |
| `REVERSE_PROXY_INTERCEPT_ERRORS=no`                           | Keep Outline's own JSON errors. With `yes`, BunkerWeb swaps every upstream `401`/`404` for its HTML page and the editor breaks.                                                                                                  |
| `COOKIE_FLAGS_1=csrfToken SameSite=Lax`, `COOKIE_FLAGS_2=__Host-csrfToken SameSite=Lax`, `COOKIE_FLAGS_3=lastSignedIn SameSite=Strict` | The editor reads its CSRF cookie in JavaScript (`csrfToken` over HTTP, `__Host-csrfToken` over HTTPS) and the sign-in page reads `lastSignedIn`. A named flag replaces BunkerWeb's default `* HttpOnly SameSite=Lax` for that cookie only: every other cookie keeps `HttpOnly`, and the session cookie `accessToken` is `HttpOnly` as Outline sets it. With the default on the CSRF cookie too, Outline rejects every write. |
| `LIMIT_REQ_RATE=100r/s`                                       | A cold page load made up to 185 requests in one second. The default `2r/s` answers `429` after the second request.                                                                                                               |
| `LIMIT_CONN_MAX_HTTP1=100`                                    | Every open document holds two WebSockets (`/realtime` and `/collaboration`; two connections established after 100 seconds on one tab). The default `10` per client address refused the 11th held socket with `429`. `100` is 50 open documents per address. |
| `MAX_CLIENT_SIZE=26m`                                         | Attachments up to Outline's 25 MiB limit. At the default `10m` a 20 MiB upload gets `413`.                                                                                                                                       |
| `MODSECURITY_REQ_BODY_NO_FILES_LIMIT=5m`                      | Outline saves documents as JSON (its own limit is 5 MB). At the default `128 KiB` a 1 MB document gets `400` (rule `200002`).                                                                                                    |
| `PERMISSIONS_POLICY` (BunkerWeb's default plus five changes)  | The default denies `fullscreen`, `picture-in-picture`, `encrypted-media` and both passkey features. The template sets `fullscreen=*`, `picture-in-picture=*`, `encrypted-media=*` and `publickey-credentials-create/get=(self)`. |
| `BLACKLIST_IGNORE_USER_AGENT=^openai-mcp/ ^openai-connectors-oauth/` | BunkerWeb's user-agent blacklist matches `openai`, so ChatGPT's MCP connector got `403` on `/mcp` and the OAuth endpoints.                                                                                              |
| `SERVE_FILES=no`                                              | Fetch content from Outline rather than local files.                                                                                                                                                                              |

The body limits apply service-wide and raise buffering and memory use. Lower both if your users
never write large documents or upload large files.

`LIMIT_CONN_MAX_HTTP1` counts WebSockets even when the page itself loads over HTTP/2: the 11th held
socket was refused over BunkerWeb's TLS port too. `429` is in the default bad-behavior status list,
so refused sockets also count toward a ban. `LIMIT_CONN_MAX_HTTP2` (default `100`) is unchanged.

`PERMISSIONS_POLICY` uses `*` for fullscreen, picture-in-picture and encrypted media because embeds
can come from any host (Outline puts `allow="fullscreen; encrypted-media; picture-in-picture"` on
every embed, `shared/editor/components/Frame.tsx:82`). Camera, microphone and geolocation stay denied.
The value copies BunkerWeb 1.6.17's default list: if a later BunkerWeb adds features, merge them in.

> [!NOTE]
> curl and some scripts send `Expect: 100-continue` on bodies above 1 MB, which CRS rule `920450`
> blocks. Browsers do not send it. Add `-H 'Expect:'` to API calls with large bodies.

## CRS exception scope and cost

`configs/modsec-crs/outline_false_positives.conf` is a service-scoped **modsec-crs** config. Every
exclusion is one chained rule: the exact path (anchored, `t:normalizePath`) and the method
`POST` (`GET` for the search page). It removes named rules from one argument.

| Id      | Path                                | Argument skipped                      | Rules removed from it                                          |
| ------- | ----------------------------------- | ------------------------------------- | -------------------------------------------------------------- |
| `9867100` | `/api/auth.config`                  | request basename                      | `920440` (the sign-in page asks for `auth.config`)             |
| `9867101` | `/api/documents.create`, `.update`  | `json.text`                           | 23 rules from the `930`, `932`, `933`, `941`, `942` families   |
| `9867102` | `/api/documents.create`, `.update`  | `json.title`                          | `930120 932160 941100 941110 941160 941390 942100 942350`      |
| `9867103` | `/api/documents.search`, `.search_titles` | `json.query`                    | ten `930`, `932`, `941`, `942` rules                           |
| `9867104` | `/api/collections.create`, `.update` | `json.description`                   | twelve `930`, `932`, `941`, `942` rules                        |
| `9867105` | `/api/comments.create`, `.update`   | the whole body (see below)            | 21 injection rules, by id                                      |
| `9867106` | `/mcp`                              | `json.params.arguments.text`          | the same 23 rules as `9867101`                                   |
| `9867107` | `/mcp`                              | `json.params.arguments.query`         | `942100 942350`                                                |
| `9867108` | `/oauth/register`                   | first four `json.redirect_uris` items | `931100 934110` (loopback redirect URIs)                       |
| `9867109` | `/oauth/authorize`, `/oauth/token`  | `redirect_uri`                        | `931100 934110`                                                |
| `9867110` | `/api/oauthClients.info`            | `json.redirectUri`                    | `931100 934110`                                                |
| `9867111` | `/api/oauthClients.create`, `.update` | first four `json.redirectUris` items | `931100 934110`                                               |
| `9867112` | `/search` (the page, `GET`)         | `query`                               | `942100`                                                       |
| `9867113` | `/api/hooks.slack`                  | `text` (the `/outline` command)       | 14 `930`, `932`, `941`, `942` rules                            |

Wiki text holds SQL, shell pipelines, HTML and URLs by nature, and the CRS reads them as attacks.
The matrix recorded which rules matched which argument; only those pairs are removed.

**Comments are the broad one.** A comment is ProseMirror JSON, so its text sits at
`json.data.content.array_N.content.array_M.text` with unbounded `N` and `M`, and this BunkerWeb
rejects regex targets in `ctl:` (the config test fails). No target can be named, so rule `9867105`
uses `ctl:ruleRemoveById` for 21 injection rules on the two comment-writing endpoints, with a
comment above it. Outline validates every other field there (UUIDs). Only these two endpoints are
affected. If you prefer a narrower rule, enumerate literal targets for the few `N` and `M` your users
produce.

**Cost:** the excluded arguments are no longer checked by the listed rules, so a stored-XSS or SQL
injection payload in a document, title, comment, search, Slack command or MCP argument is not
stopped by the WAF. The application then has to handle that content safely on its own. Headers,
cookies, other arguments, other paths, other methods and every other CRS rule stay active, with one
exception: `9867105` removes its 21 rules by id for the whole request, so on `POST /api/comments.create`
and `POST /api/comments.update` those 21 rules no longer read the headers, cookies, query string or any
other argument either. `949110` and `959100` are never excluded. `/mcp` is fully inspected apart from the two arguments above; the
template ships no detection-only mode for it. Rule `9867113` only matters when Outline's Slack
integration is configured: without it the path answers `404`.

## Validation

Tested on 2026-11-17: Outline `1.12.0` (PostgreSQL 16, Redis 7, local file storage, Dex OIDC, Mailpit),
`bunkerity/bunkerweb-all-in-one:1.6.17` (libmodsecurity 3.0.17, CRS 4.29.0), default paranoia level
and anomaly thresholds, `SECURITY_MODE=block`, plain HTTP on a local test host (HTTPS probed
separately, see below).

- **Request matrix:** 72 requests that must succeed and 39 controls, one at a time with their
  ModSecurity lines attributed per request; all 111 pass on the shipped files. Without the config 33
  of the 72 got `403` with the rule ids above (the 29 documents, comments, search and OAuth seeds
  plus the 4 Slack command seeds); with it all 72 pass and read back, and all 39 controls (the same
  payload in another argument, path, method, a fifth redirect URI, a path suffix or a `..%2f`
  traversal) still got `403` from the specific rule the exclusion removes. The newer seeds cover
  Slack commands, interactive payloads and unfurls, signed and unsigned GitHub webhooks, passkey
  options and a verify attempt, an email magic link redeemed into a session, a 24 MiB video with
  full and `Range` downloads, a collection export, a 21.8 MB workspace import, a 26 MiB upload
  (`413`), the ChatGPT user agents on `/mcp` and `/oauth/token`, and six link-preview bots on a share
  page.
- **Settings:** each non-default setting was switched off alone and the failure above reproduced
  (the three newest against a real browser, a real passkey flow with a virtual authenticator, and
  the real user agent strings).
- **Realtime:** a WebSocket handshake gets `101` on `/realtime` and `/collaboration`. Two browser
  tabs edited one document live, and the edit made after 130 seconds without input still reached
  the other tab with the default 60 second read timeout.
- **Browser smoke (bad behavior on, template as shipped):** OIDC sign-in, a document with SQL, a
  shell pipeline, an HTML example, a URL and a table, a second session seeing it and the live edits
  (also after 130 seconds idle), then passkey registration, sign-out and sign-in with the passkey.
  1,132 requests for the service: 1,110 `200`, 8 `101`, 9 `302`, 4 `401`, 1 `204`; no ModSecurity
  line, no `403`, `444` or `429`, no ban.
- **Web UI import:** raw mode with `template.json` plus the config file imports and is listed.
- **HTTPS:** only header, cookie, `FORCE_HTTPS` and connection-limit probes were run against
  BunkerWeb's TLS port (self-signed certificate, `curl` from inside the container).

Not verified: the full application flow over HTTPS and HTTP/2, `FILE_STORAGE=s3`, Slack, Google,
Azure and Discord sign-in, a real Slack or GitHub app (the routes were exercised with placeholder
credentials, Slack's own signature was not computed), the desktop and mobile apps and the service
worker, Outline's collaboration service run on its own (`SERVICES`, upstream `docs/SERVICES.md`), a
real ChatGPT or Claude connector session (their user agents and endpoints were), embeds other than
one YouTube iframe, the presentation-mode fullscreen action itself (the feature is allowed, the
action was not clicked), imports above 26 MiB, Safari video playback, `REFERRER_POLICY=no-referrer`,
and Outline versions other than `1.12.0`. `ALLOWED_METHODS` is left at its default (`GET|POST|HEAD|QUERY`),
which covers the web app, the API, `/mcp` and OAuth registration; the optional OAuth client
management calls `PUT` and `DELETE /oauth/register/<id>` get `405` until you add those methods.
Validate your own clients.

## Things to think about

- **Passkeys, presentation mode and video embeds.** BunkerWeb's default `Permissions-Policy` blocks
  them, and the page shows "The 'publickey-credentials-create' feature is not enabled in this
  document". The template relaxes only those features. Passkey origins include the port: Outline reads
  BunkerWeb's `X-Forwarded-Host` (which has it), so a custom public port works
  (`plugins/passkeys/server/auth/passkeys.ts:42-72`, measured on port 18100).
- **ChatGPT, Claude and other MCP clients.** ChatGPT sends `openai-mcp/1.0.0` for MCP calls and
  `openai-connectors-oauth/1.0` for the token exchange
  ([OpenAI forum](https://community.openai.com/t/mcp-connector-creation-times-out-despite-successful-initialize-tools-list-openai-mcp-1-0-0/1369456),
  [openai/codex#47513](https://github.com/openai/codex/issues/47513)). The template lets those two
  through the blacklist; clear `BLACKLIST_IGNORE_USER_AGENT` if you do not use ChatGPT. Claude.ai's
  connector reportedly sends `Claude-User` or `python-httpx`
  ([write-up](https://www.brendanlong.com/debugging-claude-ai-mcp-connectors.html)); both and `node`
  passed. Clients must send `Accept: application/json, text/event-stream`, or Outline
  answers `406`. Other crawlers on the same list (`GPTBot`, `ChatGPT-User`, `ClaudeBot`,
  `anthropic-ai`, `Scrapy`) stay blocked, which also keeps them off public share pages. Outline's
  [issue 14239](https://github.com/outline/outline/issues/14239) blames a reverse proxy that reuses
  upstream connections for ChatGPT's intermittent `400`: BunkerWeb's `REVERSE_PROXY_KEEPALIVE`
  defaults to `no`, and 100 consecutive `initialize` calls all answered `200` (also with `yes`).
- **Many people behind one address.** See `LIMIT_CONN_MAX_HTTP1`. Outline's own limiter keys signed-in
  sessions on the user id. Unauthenticated sign-in and API-key traffic is keyed on the client IP, so
  set `PROXY_IP_HEADER` or all of it shares BunkerWeb's address. Email sign-in is limited to 10 per
  hour per IP (`plugins/email/server/auth/email.ts:26-30`).
- **Slack.** The `/outline <words>` command posts the typed words as the form field `text`. Searching
  for code ("SELECT * FROM users") got `403`, so rule `9867113` skips 14 injection rules for that one
  field. Interactive payloads and link unfurls passed unchanged. Slack sign-in callbacks have the same
  `GET` shape as the OIDC one that was tested.
- **GitHub app.** A correctly signed `installation` event got `202`, an unsigned one `401` from
  Outline. Outline only acts on `installation`, `installation_repositories` and `repository` events
  (`plugins/github/server/GitHubIssueProvider.ts:71-90`); a repository description holding SQL or
  script text would get `403` and is not excluded.
- **Email magic links.** Invited users (no SSO account) get a link; users with an SSO account are
  sent to their provider (`plugins/email/server/auth/email.ts:65-74`). The link works only from the
  address that requested it, so keep `PROXY_IP_HEADER` consistent. Request and redemption through
  BunkerWeb both worked.
- **Large imports and exports.** A Markdown workspace zip of 21.8 MB (422 documents, 22 of them 1.3
  MB) uploaded and imported into 423 documents. Outline refuses a single imported document above
  1.46 MB. `collections.import` is limited to ten per hour. A zip above `MAX_CLIENT_SIZE` gets `413`:
  raise `MAX_CLIENT_SIZE` and the Outline limits together for a one-off import. A collection export
  downloaded through BunkerWeb as a valid zip.
- **Attachments and video.** A 24 MiB video uploaded (`25165824` bytes, the README maximum is 25 MiB),
  downloaded byte-identical, and `Range` requests answered `206`. A file of 26 MiB gets `413` from
  BunkerWeb, not `403`.
- **S3 storage.** With `FILE_STORAGE=s3` browsers and API clients upload straight to the bucket from a
  presigned URL, so BunkerWeb never sees the file and `AWS_S3_UPLOAD_BUCKET_URL` must be reachable by
  clients ([issue 13650](https://github.com/outline/outline/issues/13650)). Outline's file-storage
  guide asks for a bucket CORS policy. Not tested.
- **Embeds and framing.** Outline's own `Content-Security-Policy` (with `frame-src *`) and
  `X-Frame-Options: SAMEORIGIN` pass through unchanged (BunkerWeb keeps upstream values), and a
  YouTube iframe loaded. Other sites cannot frame Outline. BunkerWeb replaces Outline's
  `Referrer-Policy: no-referrer` with `strict-origin-when-cross-origin`, so external links receive
  the wiki's origin (never a path); embeds set their own policy. `REFERRER_POLICY=no-referrer`
  restores Outline's choice; it was not tested.
- **Link previews and search engines.** `Slackbot`, `Twitterbot`, `facebookexternalhit`,
  `Discordbot`, `WhatsApp` and `Googlebot` all got `200` for a public `/s/<id>` page.
- **Automation.** Ten `4xx` answers in a minute ban the client, and `404` is in the list: a script
  that probes for documents that do not exist collects them. Outline's own limiter (1000 requests per
  60 seconds per user, or per credential and address for API keys) is the effective cap long before
  `LIMIT_REQ_RATE`.
- **Endpoints.** `/_health` answers `200` with a two-byte body to anyone. The first provider sign-in
  creates the workspace and its admin (seen in the test), so sign in yourself right after the first
  start.
- **Not supported.** Hosting at a sub-path (`URL` with a path), per [issue 9161](https://github.com/outline/outline/issues/9161).

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
