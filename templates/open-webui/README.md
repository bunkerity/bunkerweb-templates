<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/open-webui-light.svg">
    <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/open-webui.svg" alt="Open WebUI logo" width="96" height="96" />
  </picture>
</p>

<h1 align="center">Open WebUI</h1>

<p align="center">AI chat template with WebSocket and prompt-aware WAF tuning</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Open_WebUI-v0.13.3-248651?labelColor=1d6f93&style=flat" alt="Open WebUI v0.13.3" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#configure-open-webui-for-the-proxy">Configure Open WebUI</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#crs-exception-scope-and-cost">CRS exceptions</a> ·
  <a href="#validation">Validation</a> ·
  <a href="#things-to-think-about">Things to think about</a>
</p>

Put an existing [Open WebUI](https://openwebui.com/) server behind BunkerWeb with ModSecurity and the
OWASP CRS on: chats with SQL, shell, HTML and code in the prompt, streamed replies, the Socket.IO
connection, voice and camera, file uploads, notes, prompts, model settings, admin code editing,
OpenAI-compatible API clients and single sign-on.

## At a glance

| Fact        | Value                                             |
| ----------- | ------------------------------------------------- |
| Template id | `open-webui`                                      |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Open WebUI v0.13.3 |
| Configs     | 2 (modsec-crs, server-http)                       |
| Rule ids    | `9865000` to `9865999`                            |
| Upstream    | `http://open-webui:8080`                          |

## Prerequisites

- Run Open WebUI `v0.13.3` or a close release. BunkerWeb must reach `http://open-webui:8080` on a
  private network; do not expose that port publicly. Any OpenAI-compatible backend works; the
  template does not touch the connection between Open WebUI and its models.
- The first account created through the sign-up form becomes the administrator. Open WebUI then
  switches sign-up off by itself (a second sign-up gets `403`); turn it back on in the admin panel
  or with `ENABLE_SIGNUP` only if you want open registration. Create the admin account before the
  site is public.
- Open WebUI's Socket.IO server accepts the `websocket` transport only (`transports=['websocket']`
  in `backend/open_webui/socket/main.py`). Without the WebSocket upgrade the chat page cannot
  receive replies, so do not put another proxy in front of BunkerWeb that strips `Upgrade`.
- Point your public hostname at BunkerWeb. `AUTO_LETS_ENCRYPT=yes` makes BunkerWeb request its
  certificate (the default HTTP challenge needs inbound port 80). For existing certificate files,
  use `AUTO_LETS_ENCRYPT=no`, `USE_CUSTOM_SSL=yes`, `CUSTOM_SSL_CERT` and `CUSTOM_SSL_KEY`.
- Serve Open WebUI at the root of the hostname: the template proxies `/` and was not tested under
  a sub-path.

## Configure Open WebUI for the proxy

Set these on the Open WebUI container. The "without it" column is what was measured or read in
the source of `v0.13.3` (`backend/open_webui/env.py` and `main.py` in the Open WebUI repository).

| Open WebUI setting                                | Value                                                    | Without it                                                                                                                                                                                                                                                       |
| ------------------------------------------------- | -------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `FORWARDED_ALLOW_IPS`                             | BunkerWeb's address on the Docker network (not `*`)      | The default `*` trusts every sender, and uvicorn takes the first `X-Forwarded-For` entry: a client that reaches the app's port directly picks the address the app sees (`6.6.6.6` in the test, `uvicorn-ip.log`). Through BunkerWeb the template replaces the header (see the settings table). |
| `WEBUI_URL`                                       | `https://open-webui.example.com`                         | OAuth and OpenID Connect redirect addresses are built from `WEBUI_URL`, or from the request address when it is empty (`utils/oauth.py:575,694,2256`). Behind another TLS-terminating proxy the request address is `http`, so the redirect address is wrong.     |
| `WEBUI_SESSION_COOKIE_SECURE`                     | `true` once the site is HTTPS                            | The default is `false` (`env.py:782`): the session and token cookies are also sent over plain HTTP. `WEBUI_AUTH_COOKIE_SECURE` follows it.                                                                                                                       |
| `WEBUI_SECRET_KEY`                                | a long random value, kept across container recreation    | Without it `start.sh` writes `.webui_secret_key` into `/app/backend`, which is not the `/app/backend/data` volume: a recreated container gets a new key and every session and every encrypted value is lost (`start.sh:29-46`).                                  |
| `CORS_ALLOW_ORIGIN`                               | `https://open-webui.example.com`                         | The default `*` (`config.py:2202`) answers any `Origin` with `access-control-allow-origin: <that origin>` and `access-control-allow-credentials: true` (measured). With the public URL set, a request from `https://evil.example` got no `access-control-allow-origin` header and the allowed origin got it.                                                                                          |
| `WEBUI_AUTH_TRUSTED_EMAIL_HEADER` (only if used)  | the header your authenticating proxy sets                | BunkerWeb passes a client-sent header on. Measured with the variable set: `POST /api/v1/auths/signin` with only `X-Forwarded-Email: <the admin's address>` and no credentials returned an admin token through BunkerWeb, and an unknown address returned a token for a new account in the `pending` role.                                                                                                                                                                                                                  |

Keep `ENABLE_WEBSOCKET_SUPPORT` at its default `true`. The documentation's fallback (long polling)
needs sticky sessions with more than one replica.

Security: `FORWARDED_ALLOW_IPS` must name only BunkerWeb. With `*`, anyone who can reach the
Open WebUI port directly chooses the client address shown in the app's audit trail and used by
its per-address limiters
(`routers/sessions.py:86`, `routers/auths.py:1623`). Open WebUI's sign-in limiter is keyed by
e-mail address (`routers/auths.py:99`), not by address.

## Install

Import `template.json` and the files under `configs/` using the repository
[installation guide](../../README.md#install), or its plugin bundle layout. Assign
`USE_TEMPLATE=open-webui`, then change only `SERVER_NAME` and `REVERSE_PROXY_HOST` (keep the upstream
URL without a trailing slash or path). In the web UI raw import, the config dialog lists the files
as `modsec_crs/open-webui_false_positives.conf` and `server_http/open-webui_large_cookies.conf`: upload
each file under its listed name.

Before enabling blocking on your instance:

1. Set service `SECURITY_MODE=detect` and reload.
2. Sign in, send prompts with code, SQL and markup, use a temporary chat, upload a file, write a
   note, save a prompt and a custom model, and read the BunkerWeb logs.
3. Confirm the included exclusions match the targets in your logs. Narrow or add exceptions only
   where your traffic requires them.
4. Switch to `SECURITY_MODE=block` after those checks and repeat the workflows.

> [!IMPORTANT]
> False positives return `403` in block mode. The default bad-behavior status list includes
> `400 401 403 404 405 429 444`: `BAD_BEHAVIOR_THRESHOLD` matching responses in 60 seconds ban the
> client for one day (the template sets 30, the BunkerWeb default is 10), so the detect run matters.

## Settings

Each setting was put back to the BunkerWeb default and the failure below reproduced. The settings
added in the second pass were reverted together; `REVERSE_PROXY_READ_TIMEOUT` and
`BAD_BEHAVIOR_THRESHOLD` were reverted on their own.

| Setting                                      | Purpose and measurement                                                                                                                                                                                                                                                                                                    |
| -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `REVERSE_PROXY_URL=/`, `REVERSE_PROXY_WS=yes` | Proxy the app and upgrade `/ws/socket.io` and `/api/v1/audio/realtime` (voice calls). Without it the Socket.IO handshake answers `400`. The voice-call handshake returns `101` through BunkerWeb and directly.                                                                                                              |
| `REVERSE_PROXY_INTERCEPT_ERRORS=no`          | Keep Open WebUI's own JSON errors (`{"detail": ...}`). With `yes`, BunkerWeb swaps every upstream `400`/`401` for its HTML page (`content-type: text/html`).                                                                                                                                                                |
| `REVERSE_PROXY_READ_TIMEOUT=30m`     | A reply that streams over plain HTTP (temporary chats, API clients) waits for a slow model. A model that stays silent for 75 s was cut at the default 60 s (`200`, no end of stream). A 330 s silence was cut at 300 s with `5m` and completed (333 s) with `30m`. The official nginx example uses `1800` seconds.                                                                                                                                                                                                                           |
| `REVERSE_PROXY_HEADERS=X-Forwarded-For $remote_addr` | Replaces BunkerWeb's default `X-Forwarded-For $proxy_add_x_forwarded_for`, which appends to what the client sent: a client `X-Forwarded-For: 6.6.6.6` reached the app as `6.6.6.6, <client address>`. With the setting the same request reached the app as `172.21.0.1`, the real client, and nothing else.                                                                                                          |
| `ALLOWED_METHODS=GET\|POST\|HEAD\|PUT\|PATCH\|DELETE` | Deleting a chat, changing chat parameters and notification targets use `DELETE`, `PATCH` and `PUT`. The default `GET\|POST\|HEAD\|QUERY` answers `405` to all three.                                                                                                                                                |
| `LIMIT_REQ_RATE=100r/s`                      | The single page app loads hundreds of small assets at once: a cold load made 122 requests in one second. The default `2r/s` answered 78 of 80 burst requests with `429`; with `100r/s` all 80 pass. One address shared by several people that open the app together can exceed it (not measured with several browsers). |
| `LIMIT_CONN_MAX_HTTP1=200`                   | Every open tab holds one WebSocket on its own HTTP/1.1 connection, also when the page uses HTTP/2. At the default `10`, the 11th WebSocket from one address got `429` and so did an ordinary request made while they were open; with `200`, 100 held WebSockets all got `101`. `429` also counts toward bad behavior. |
| `BAD_BEHAVIOR_THRESHOLD=30`                  | BunkerWeb bans an address for one day after this many responses with a status from its bad-behavior list (`403` is in it) within a minute; the default is 10. While channels are off (the default, `config.py:2084`) Open WebUI answers `GET /api/v1/channels/` with `403` ("Channels is disabled") on every page load. Fourteen signed-in page loads in a row banned the address at the default (the next `GET /api/version` got `403`); with `30` the same fourteen left it unbanned. The cleaner fix is `ENABLE_CHANNELS=true` on the app. ModSecurity `403`s count toward the same ban: a scanner now needs 30 blocked requests in a minute. |
| `MAX_CLIENT_SIZE=100m`                       | File uploads (Open WebUI sets no size limit of its own). A 99 MiB upload passes and a 100 MiB one gets `413`; at the default `10m` an 11 MiB upload gets `413`.                                                                                                                                                              |
| `MODSECURITY_REQ_BODY_NO_FILES_LIMIT=10m`    | A chat resends its whole history as one JSON body on every turn, images included. At the default `128 KiB`, a 300 KiB prompt gets `400`; with `10m`, 9000 KiB passes and 10500 KiB gets `400`. A 200 KB image in a vision chat is about 200 KB of that budget.                                                               |
| `PERMISSIONS_POLICY` (default list with `microphone`, `camera`, `display-capture`, `fullscreen`, `geolocation`, `screen-wake-lock`, `speaker-selection` and `autoplay` set to `(self)`) | The BunkerWeb default denies all of them, and the bundle uses `getUserMedia`, `getDisplayMedia`, `requestFullscreen`, `geolocation`, `wakeLock`, `setSinkId` and `new Audio`. In headless Chrome `document.permissionsPolicy.allowsFeature()` returned `false` for all eight with the default and `true` with the setting; `clipboard-read` and `clipboard-write` are not in the list and were `true` both times. |
| `COOKIE_FLAGS_1=token SameSite=Lax`          | BunkerWeb's `COOKIE_FLAGS=* HttpOnly SameSite=Lax` adds `HttpOnly` to every cookie. Open WebUI's SSO callback sets `token` without `HttpOnly` on purpose because its `/auth` page reads it with `document.cookie` (`utils/oauth.py:2320`). With the default, the page stayed on `/auth` with no token in `localStorage`; with this setting it signed in and moved to `/`. The app sets its other cookies `HttpOnly` itself; the default flags still apply to them. The browser leg used a stub that plays the callback; no identity provider was run. |
| `PROXY_BUFFER_SIZE=16k`, `PROXY_BUFFERS=8 16k`, `PROXY_BUSY_BUFFERS_SIZE=32k` | A response header larger than the buffer is a `502`. With the defaults a 6 KB `Set-Cookie` (Open WebUI keeps the provider's ID token in an `oauth_id_token` cookie) got `502`; with these values 12 KB passes and 20 KB gets `502`. All three are set together because nginx rejects a busy size that is not smaller than the total. |
| `configs/server-http/open-webui_large_cookies.conf` (`large_client_header_buffers 4 32k`) | The browser sends the token cookies back in one `Cookie` header. With the nginx default a 9 KB `Cookie` header got `400`; with the file 30 KB passes and 40 KB gets `400`. |

Both body allowances apply service-wide and raise buffering and memory use. Lower them if your users
never paste large documents or upload large files.

Measured and dropped because they made no difference on `v0.13.3`: `REVERSE_PROXY_BUFFERING=no`
(a streamed reply arrives chunk by chunk with buffering on or off), `REVERSE_PROXY_CONNECT_TIMEOUT`,
`REVERSE_PROXY_SEND_TIMEOUT` and `SERVE_FILES`. The rest of `COOKIE_FLAGS` stays at its default.

> [!NOTE]
> curl and some scripts send `Expect: 100-continue` on bodies above 1 MB, which CRS rule `920450`
> blocks. Browsers do not send it. Add `-H 'Expect:'` to API calls with large bodies.

`OPTIONS` is not allowed. A cross-origin preflight (`POST` with `Authorization`) from a web page on
another origin got `405` from BunkerWeb. With `OPTIONS` added to `ALLOWED_METHODS` the same preflight got `200` from Open WebUI. Add it only for a browser client on another origin, and set `CORS_ALLOW_ORIGIN` to that origin first.

## CRS exception scope and cost

`configs/modsec-crs/open-webui_false_positives.conf` is a service-scoped **modsec-crs** config. Every
exclusion is one chained rule: the path (anchored, `t:normalizePath`) and the method. The matrix
recorded which rules matched which argument on the real app; only those are removed. The text class
the rules cover is what people paste into a chat: SQL, shell pipelines, path traversal, HTML and
script markup, PHP, Python, Node and Java code, XML, URLs, raw HTTP requests and template syntax.

| Id | Method and path | Where | Rules |
| --- | --- | --- | --- |
| `9865101` | `POST /api/v1/auths/signup`, `/api/v1/auths/update/profile` | `json.profile_image_url` (data: URI) | 5 rules, by target |
| `9865102` | `POST /api/v1/users/user/settings/update` | `json.ui.system` | 45 rules, by target |
| `9865103` | `POST /api/v1/chats/new`, `/import`, `/<uuid>` | the whole request (by id) | 46 rules, by id |
| `9865104` | `GET /api/v1/chats/search` | `text` (query string) | 47 rules, by target |
| `9865105` | `POST /api/chat/completions`, `/api/v1/chat/completions`, `/api/chat/completed`, `/api/(v1/)messages`, `/api/(v1/)embeddings`, `/openai/chat/completions`, `/ollama/api/chat`, `/ollama/api/generate`, `/ollama/v1/chat/completions` | the whole request (by id) | 48 rules, by id |
| `9865106` | `POST /api/v1/notes/create`, `/api/v1/notes/<uuid>/update` | `json.data.content.md`, `.html` | 45 rules, by target |
| `9865107` | `GET /api/v1/notes/search` | `query` (query string) | 47 rules, by target |
| `9865108` | `POST /api/v1/prompts/create`, `/api/v1/prompts/id/<id>/update` | `json.content` | 45 rules, by target |
| `9865109` | `POST /api/v1/models/create`, `/api/v1/models/model/update` | `json.params.system` | 45 rules, by target |
| `9865110` | `POST /api/v1/memories/add`, `/api/v1/memories/query` | `json.content` | 46 rules, by target |
| `9865111` | `POST /api/v1/knowledge/create`, `/api/v1/knowledge/<uuid>/update` | `json.description` | 46 rules, by target |
| `9865112` | `POST /api/v1/functions/create`, `/api/v1/tools/create` | `json.content` (Python source, admin) | 44 rules, by target |
| `9865113` | `POST /api/v1/utils/code/format`, `/api/v1/utils/code/execute` | `json.code` (admin) | 43 rules, by target |
| `9865114` | `POST /api/v1/tasks/title/completions` | the whole request (by id) | 45 rules, by id |
| `9865115` | `POST /api/v1/tasks/auto/completions`, `/api/v1/images/generations` | `json.prompt` | 7 rules, by target |
| `9865116` | `POST /api/v1/audio/speech` | `json.input` | 45 rules, by target |
| `9865117` | `POST /api/v1/retrieval/process/text`, `/api/v1/files/<uuid>/data/content/update`, `/api/v1/chats/<uuid>/messages/<id>` | `json.content` | 45 rules, by target |
| `9865118` | `POST /api/v1/models/import` | the whole request (by id) | 45 rules, by id |
| `9865119` | `POST /api/v1/folders/<uuid>/update` | `json.data.system_prompt` | 45 rules, by target |
| `9865120` | `POST /api/v1/evaluations/feedback` | `json.data.comment` | 45 rules, by target |
| `9865121` | `GET /` (address-bar search) | `q` (query string) | 47 rules, by target |
| `9865122` | `POST /api/v1/retrieval/config/update` | the URL fields of the Documents page | 1 rule, by target |
| `9865123` | `POST /api/v1/images/config/update` | `json.COMFYUI_WORKFLOW` | 1 rule, by target |
| `9865124` | `POST /api/v1/configs/import` | the URL and workflow fields of an exported config | 2 rules, by target |

The rule counts are the ids in the file, each one seen on that argument in the matrix. A few of them
(`921110`, `944100`, `944110`, `944130`) read the raw JSON body next to the arguments, so the
literal-argument rules also remove them for `REQUEST_BODY` on their own path.

**Several rules remove whole rule ids for their path (marked "by id" above).** The prompt travels
as `json.messages.array_<N>.content` with `N` growing with the chat, a saved chat uses a UUID key per
message (`json.chat.history.messages.<uuid>.content`), an imported file lists its models under
`json.models.array_<N>`. This BunkerWeb rejects regex targets in `ctl:` (a target with `^`, `\`, `+`
or `|` fails the config test, and `/json.messages.*/` parses but never matches: probes in the lane
evidence), so no target can be named. Those rules use `ctl:ruleRemoveById`, with a comment above each,
on those exact paths and the method `POST`. Open WebUI hands the text to the model and does not
execute it. If you prefer a narrower rule, enumerate literal targets for the few `N` your users
reach; the list is unbounded.

**Cost:** the excluded arguments (and, on the "by id" rules, the whole request body) are no longer
checked by the listed injection rules, so a stored-XSS or SQL injection payload in a prompt, note,
system prompt or function source is not stopped by the WAF. Open WebUI has to handle that content
safely itself, and an agent with tools can still act on a prompt-injected instruction. Headers,
cookies, other arguments, other paths, other methods and every rule outside the lists stay active.
`949110` and `959100` are never excluded. The same payload on the same path in the **query string**
is also not inspected by the ids removed on the "by id" paths (measured, `G05f` in the matrix); the
app ignores unknown query parameters.

Not covered, so a payload there is blocked: other search boxes (`/api/v1/knowledge/search`,
workspace model and prompt lists), channel messages, admin prompt suggestions with SQL in them
(`C23a`: `403`), tool valves, and every endpoint the matrix did not exercise. Add one exclusion per
argument your logs name.

## Validation

Tested on 2026-11-17: Open WebUI `v0.13.3` (SQLite, offline mode, a stub OpenAI-compatible backend
that streams a fixed reply), `bunkerity/bunkerweb-all-in-one:1.6.17` (libmodsecurity 3.0.17, CRS
4.29.0), default paranoia level and anomaly thresholds, `SECURITY_MODE=block`, plain HTTP on a
local test host.

- **Request matrix** (final run on the shipped files): 135 requests that must succeed and 72 controls, one at a time with
  their ModSecurity lines attributed per request. Without the config 38 of the first 68 and 31 of the 67 added in the second
  pass got `403` with the rule ids in the file; with it all 135 pass and read back (accounts, chats, completions that reach the
  model with the prompt unmodified, an image in a chat, tool calls, uploads, notes, prompts, models, memories, functions, admin
  settings pages), and all 72 controls (the same payload in another argument, path, method, a path suffix, a non-UUID id or a
  `..%2f` traversal) still got `403` from the specific rule the exclusion removes. The other seeds need no exclusion: the page and
  its discovery files, sign-in and sign-up with a punctuation-heavy password, uploads, `PATCH` and `DELETE`, twelve client user
  agents, the OIDC callback and back-channel logout shapes, the shared-chat page.
- **Settings:** each non-default setting was switched back to the BunkerWeb default and the failure above reproduced.
- **Realtime:** a WebSocket handshake gets `101` on `/ws/socket.io` and on `/api/v1/audio/realtime`. The browser smoke kept the
  chat open for 125 seconds without input and the next reply still arrived.
- **Browser smoke** (bad behavior on, template as shipped, only `SERVER_NAME` and `REVERSE_PROXY_HOST` set): the first
  administrator signs up through BunkerWeb (the avatar goes out as a `data:` URI), a prompt with a SQL statement, a shell pipeline,
  a `<script>` tag and a URL gets its streamed reply, the chat idles for 125 seconds and a second prompt still gets its reply, a
  temporary chat streams its reply over plain HTTP, a text file attached to a chat uploads and is processed, a page opened from
  `/?q=<SQL statement>` loads with the text in the prompt box, and seven admin and workspace pages open. The BunkerWeb log for the
  623 requests of the run shows 548 `200`, 46 `304`, 10 `101`, 7 `302`, 1 `206`, 1 `499` and ten `403`, all ten
  `GET /api/v1/channels/` answered by Open WebUI itself (`Channels is disabled`, no ModSecurity line). No ModSecurity line, no ban.
- **Web UI import:** raw mode with `template.json` plus both config files imports and is listed next to the plugin copy.

Not verified: HTTPS and HTTP/2, a real OpenID Connect or LDAP provider, a real model backend
(Ollama, a hosted API), the notes editor in a browser (its text travels over Socket.IO), knowledge
base indexing with a real embedding model, web search and image generation engines, a live
voice call, channels, API key creation, mobile apps, several browsers behind one address, Open WebUI
versions other than `v0.13.3`. Validate your own clients.

## Things to think about

- **Single sign-on.** After an OpenID Connect sign-in the app's `/auth` page must read the `token`
  cookie from JavaScript; that is the `COOKIE_FLAGS_1` row above, and with plain passwords you can
  drop that step. Set `WEBUI_URL` to the public HTTPS address first, or the callback is built from
  the proxy's own scheme. The callback
  request itself (`code`, `state`, `iss` URL, `session_state`) passes the CRS (`U06b`). The IdP's
  back-channel logout `POST /oauth/backchannel-logout` also passes. No identity provider was run.
- **Bans from normal navigation.** Each page load of a signed-in user adds a `403` from
  `/api/v1/channels/` while channels are off. Admin pages opened in quick succession reached 10 of
  them inside a minute in the smoke. Other `4xx` answers from the app count the same way; only
  this `403` was measured.
- **Many people behind one address.** Each tab is one WebSocket (`LIMIT_CONN_MAX_HTTP1`), each cold
  page load is about 120 requests (`LIMIT_REQ_RATE`). A `429` also feeds BunkerWeb's bad-behavior
  ban, which bans the shared address. Raise both for offices and schools.
- **Voice and camera.** The Open WebUI documentation notes that microphone input over a LAN
  generally needs HTTPS (docs.openwebui.com, HTTPS page), so test these features on the HTTPS site.
- **Long silent replies.** Open WebUI sends nothing while a model prefills or reasons (upstream issue 31408 asks for a heartbeat). `REVERSE_PROXY_READ_TIMEOUT=30m` covers a 330 s silence; raise it if your model is slower. Replies over the Socket.IO connection are not affected: the socket pings every 25 s.
- **Vision and call mode.** Each turn resends all earlier images; the JSON body ceiling is
  `MODSECURITY_REQ_BODY_NO_FILES_LIMIT`. At `10m` a chat of about fifty 200 KB images reaches it
  (calculated from the 10 MiB limit and the size of one test image).
- **API clients and scripts.** Twelve user agents (OpenAI Python and JS SDKs, `python-requests`,
  `Go-http-client`, Continue, `litellm`, Postman, axios, `node-fetch`, aiohttp, curl, a phone
  browser) all got `200` on `/api/models` through the default blacklist. `/api/chat/completions`,
  `/openai/chat/completions`, `/api/v1/messages` (Anthropic style), `/api/v1/embeddings` and the
  Ollama paths accept code in the prompt (the "by id" rules). A client on another origin needs
  `OPTIONS` and `CORS_ALLOW_ORIGIN`.
- **Admin settings pages.** Saving the Documents and Images pages and importing a config file sent
  the app's own defaults (`http://localhost:...` URLs, a ComfyUI workflow JSON) and got `403`
  (rules `934110`, `942151`) until the three exclusions at the end of the table were added.
- **Address-bar search.** The browser search entry from `opensearch.xml` opens `/?q=<text>`.
  Text with SQL in it is blocked without the `/?q=` exclusion; with it, the page and its assets
  load. A `Referer` carrying the same text did not raise a rule (`U07d`).
- **Trusted-header sign-in.** If you use `WEBUI_AUTH_TRUSTED_EMAIL_HEADER`, the header must be set
  by an authenticating proxy that sits in front of BunkerWeb and strips it from clients.
  Without that, any client signs in as any user by sending the header (measured). Leave the variable unset if you use passwords or SSO.
- **Not supported through the template:** a sub-path deployment, and `ENABLE_WEBSOCKET_SUPPORT=false`
  with several Open WebUI replicas (needs sticky sessions that BunkerWeb does not set here).
- **`ENABLE_API_KEYS`.** API keys use the same `Authorization: Bearer` header; the bearer path was
  tested with a session token. Key creation itself was not.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
