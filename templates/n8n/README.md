<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/n8n.svg" alt="n8n logo" width="96" height="96" />
</p>

<h1 align="center">n8n</h1>

<p align="center">Workflow automation template with editor and webhook tuning</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/n8n-2.48.4-248651?labelColor=1d6f93&style=flat" alt="n8n 2.48.4" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#configure-n8n-for-the-proxy">Configure n8n</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#crs-exception-scope-and-cost">CRS exceptions</a> ·
  <a href="#things-to-think-about">Things to think about</a> ·
  <a href="#validation">Validation</a>
</p>

Expose an existing n8n instance through BunkerWeb: the editor (saving and running workflows, the
push channel), production, test and waiting webhooks with JSON and file bodies, forms, the Chat
Trigger, MCP (the MCP Server Trigger and n8n's instance-level MCP server), the public API, and an
optional webhook-only mode. ModSecurity and the OWASP CRS stay on.

## At a glance

| Fact        | Value                                     |
| ----------- | ----------------------------------------- |
| Template id | `n8n`                                     |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, n8n 2.48.4 |
| Configs     | 5 (modsec-crs)                            |
| Rule ids    | `9861000` to `9861999`                    |
| Upstream    | `http://n8n:5678`                         |

## Prerequisites

- Run n8n with persistent storage. BunkerWeb must reach `http://n8n:5678` on a private network;
  do not publish port `5678`.
- Point your public hostname at BunkerWeb. `AUTO_LETS_ENCRYPT=yes` asks BunkerWeb for a
  certificate (the default HTTP challenge needs inbound port 80). For your own certificate use
  `AUTO_LETS_ENCRYPT=no`, `USE_CUSTOM_SSL=yes`, `CUSTOM_SSL_CERT` and `CUSTOM_SSL_KEY`.
- Serve n8n at the root of its hostname. The CRS exceptions and rate-limit slots anchor on
  `/rest/`, `/webhook/`, `/api/v1/` and the other default paths. A sub-path deployment was not
  tested.

## Configure n8n for the proxy

The template cannot set these n8n variables for you.

| n8n variable                   | Value                           | What breaks without it                                                                                                                                                                                                                                  |
| ------------------------------ | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `N8N_EDITOR_BASE_URL`          | `https://n8n.example.com`       | The push channel compares the browser `Origin` with the forwarded host and answers `Invalid origin!` on a mismatch (`push/index.ts:126-140`). Include the port when you serve a non-standard one: BunkerWeb forwards `X-Forwarded-Host` with the port. |
| `N8N_WEBHOOK_URL`              | `https://n8n.example.com/`      | Base URL n8n shows for webhooks and forms, and signs into waiting links (`resumeUrl`). n8n 2.48.4 reports `WEBHOOK_URL` as deprecated in favour of this one. In webhook-only mode, use the public webhook host.                                       |
| `N8N_PROXY_HOPS`               | `1`                             | n8n then reads the client address from `X-Forwarded-For` (`abstract-server.ts:95-96`). With `0` n8n sees BunkerWeb's address for every client: a Webhook node IP whitelist rejected everyone (`403`, see below).                                       |
| `N8N_SECURE_COOKIE`            | default (`true`)                | Over HTTPS the login cookie must be `Secure`. Set `false` only for a plain-HTTP test.                                                                                                                                                                 |
| `N8N_DEFAULT_BINARY_DATA_MODE` | `filesystem`                    | Optional, for large uploads. In the default `memory` mode a 199 MiB upload ran n8n out of Node heap (`FATAL ERROR: Zone Allocation failed`, exit 134) in the test setup.                                                                               |
| `N8N_PAYLOAD_SIZE_MAX`         | `16` (MiB, default)             | JSON bodies. `MODSECURITY_REQ_BODY_NO_FILES_LIMIT=16m` follows it; change both together.                                                                                                                                                              |
| `N8N_FORMDATA_FILE_SIZE_MAX`   | `200` (MiB, default)            | Form-data files. `MAX_CLIENT_SIZE=200m` follows it; change both together.                                                                                                                                                                             |
| `N8N_METRICS`                  | off, or scrape privately        | `/metrics` has no authentication in n8n. The template answers `403` for it at BunkerWeb (see below).                                                                                                                                                   |
| `N8N_PUSH_BACKEND`             | `websocket` (default)           | `sse` opens and pings through the template (measured, not tried in a browser). Keep the default.                                                                                                                                                        |
| `N8N_ENDPOINT_*`               | defaults                        | The CRS exceptions and the rate-limit slots name the default paths (`webhook`, `rest`, `form`, `api`). If you rename one, change the same path in the configs and in `LIMIT_REQ_URL_1` to `_3`.                                                        |

Keep `N8N_PROXY_HOPS` at `1`: BunkerWeb appends the client address to a client-supplied
`X-Forwarded-For` (`6.6.6.6, <client>`). With `1` n8n takes the last entry and ignores the forged
one. With `2` it takes the forged first entry: a Webhook node whitelisted for `6.6.6.6` accepted a
request that carried `X-Forwarded-For: 6.6.6.6`. Raise it only when another proxy that you control
sits in front of BunkerWeb, and then set `USE_REAL_IP` on the BunkerWeb service too (not tested).

## Install

Import `template.json` and the five files under `configs/modsec-crs/` using the repository
[installation guide](../../README.md#install), or its plugin bundle layout. Assign
`USE_TEMPLATE=n8n`, then change `SERVER_NAME` and `REVERSE_PROXY_HOST` (keep port `5678`, no
trailing path). Nothing else needs to change for the full app.

Before enabling blocking on your instance:

1. Set service `SECURITY_MODE=detect` and reload.
2. Create, save, run and publish a workflow that holds your usual code, SQL and HTML. Call each
   production webhook with the bodies your callers send. Read the BunkerWeb log meanwhile.
3. Add an exclusion only for a request you recognise as legitimate (see the last paragraph of
   the CRS section).
4. Switch to `SECURITY_MODE=block`.

> [!IMPORTANT]
> False positives return `403`.

### Optional: webhook-only exposure

To publish only what outside callers use, set
`REVERSE_PROXY_URL=~ ^/(webhook|webhook-waiting|form|form-waiting)/` on the public service and reach
the editor through another service, a VPN or an internal hostname. Measured with that value on the
final template: the production webhook, the hosted chat page and its API, the Form Trigger page
(a browser submit returned `200`), the waiting form and a signed resume link answer; `/`,
`/signin`, `/rest/*`, `/assets/`, `/types/*`, `/healthz`, `/api/v1/`, `/mcp/`, `/mcp-server/`,
`/webhook-test/` and `/form-test/` return `404`, and `/metrics` stays `403`. MCP and public API
clients, and test webhooks from the editor, need their own `REVERSE_PROXY_URL` entry. The
exclusions for `/webhook-test/` and the editor paths then have nothing to apply to.

## Settings

Every non-default value was checked against the BunkerWeb default on the same app. The numbers
come from `curl`, a Node WebSocket client and n8n's own HTTP Request node through BunkerWeb.

| Setting | Without it | With it |
| --- | --- | --- |
| `REVERSE_PROXY_WS=yes` | `/rest/push` upgrade refused (`401`). | Upgrade accepted, stays open 130 s idle (n8n pings every 60 s). |
| `REVERSE_PROXY_READ_TIMEOUT=1h` | The default is 60 s: a webhook whose workflow takes 70 s gets `504`. An MCP Server Trigger stream has no keep-alive and was cut at 300.1 s under `5m`. | A 70 s workflow gets `200`. The MCP stream stayed open to the 400 s test limit and the editor socket for 340 s idle. |
| `REVERSE_PROXY_INTERCEPT_ERRORS=no` | A wrong login gets BunkerWeb's HTML page. | The editor and webhook callers get n8n's JSON error. BunkerWeb's own blocks keep their page. |
| `ALLOWED_METHODS` with `PUT\|DELETE\|PATCH\|OPTIONS` | `PATCH` (save), `DELETE`, `PUT` (share) and `OPTIONS` (CORS preflight) get `405`. | `200`, `200`, n8n's own `403`, `204`. |
| `MAX_CLIENT_SIZE=200m` | A 150 MiB multipart upload gets `413`. | 150 and 199 MiB get `200`, 201 MiB gets `413`. |
| `MODSECURITY_REQ_BODY_NO_FILES_LIMIT=16m` | A 200 KB workflow save gets `400` (rule `200002`). | `200`. 15.5 MB passes ModSecurity, 17.5 MB gets `400`. |
| `LIMIT_REQ_URL_1` / `RATE_1` = webhooks, forms, waiting and test URLs, `100r/s` | 100 calls from one address: 2 x `200`, 98 x `429`. | 100 x `200`. |
| `LIMIT_REQ_URL_2` / `RATE_2` = `/mcp`, `/mcp-server`, `/api/v1`, `200r/s` | 100 API calls: 2 x `200`, 98 x `429`. An MCP client start-up got `200 202 429 429`. | 100 x `200`. The same start-up got `200 202 200 200`. |
| `LIMIT_REQ_URL_3` / `RATE_3` = `^/rest/push$`, `30r/s`, with `LIMIT_CONN_MAX_HTTP1=100` | 14 editor sockets from one address: 4 open with the default rate (`2r/s`), 10 open with the default connection limit (10). | 14 and 40 sockets open. |
| `BAD_BEHAVIOR_STATUS_CODES=400 401 403 405 444` | 12 calls to an unpublished webhook (`404`) banned the sender: the published webhook and `/healthz` then got `403`. | The same 12 calls: no ban, `200` on both 25 s later. |
| `BLACKLIST_URI=^/metrics$` | With `N8N_METRICS=true`, `/metrics` answers `200` with Prometheus text. | `403`. `/healthz` stays `200`. |
| `X_FRAME_OPTIONS`, `CONTENT_SECURITY_POLICY` (equal to BunkerWeb's defaults) | See "Embedding". | See "Embedding". |

Costs and choices:

- `16m` equals n8n's JSON limit. ModSecurity buffers and scans up to that much per JSON request:
  the 15.5 MB request took 6 s end to end. Lower it if your callers send small bodies.
- `ALLOWED_METHODS` adds `PUT`, `DELETE`, `PATCH` and `OPTIONS` for the whole service,
  including the webhooks.
- The rate slots limit one client address and one URL. The values are about three times the fastest
  rates n8n's HTTP Request node reached against its own public URL: 33 requests per second on
  webhooks and 66 on the API. A fleet of callers behind one address, or a CDN that hides client
  addresses, needs higher values (`USE_REAL_IP` was not tested).
- `LIMIT_CONN_MAX_HTTP1=100` counts every open connection, push sockets included. A fan-out of
  300 HTTP Request items to a public n8n URL got 100 x `200` and 200 x `429` from the connection
  limit, with the node default and with a batch size of 20; 100 items, with or without batching,
  all passed. Raise the value if your workflows fan out wider.
- `BAD_BEHAVIOR_STATUS_CODES` drops `404` and `429` from the default list. A scanner that
  floods `404` is then no longer banned, and a client that is rate limited does not escalate to
  a ban. `400`, `401`, `403`, `405` and `444` still count.

## CRS exception scope and cost

Every file is a service-scoped **modsec-crs** config (rule ids `9861100` to `9861399`).

**`n8n_editor_false_positives.conf`**, five exact method and path pairs:

| Id | Request | What is removed |
| --- | --- | --- |
| `9861100` | `POST /rest/workflows` | `ARGS` from 89 injection rules |
| `9861101` | `PATCH /rest/workflows/<id>` | same |
| `9861102` | `POST /rest/workflows/<id>/run` | same |
| `9861103` | `POST /rest/dynamic-node-parameters/<kind>` | same |
| `9861104` | `GET /types/credentials.json` | the file name target of `930130` only |

A workflow is code: Code nodes (JavaScript with `require`), SQL queries, shell commands, HTML and
expressions are argument values, so the SQL, shell, XSS, PHP and Node injection rules fire on
content the user is meant to write. The 89 ids are every paranoia-level-1 rule of the CRS 4.29
families `930` to `934`, `941` and `942` that inspect `ARGS`, plus `921120` and `921130`. The list
comes from the shipped rule files, not from test workflows. `ARGS` is excluded as a whole because
the argument name carries the node position and pinned data carries the node name; regex targets
fail to parse in 1.6.17 and `ARGS_POST` does not match JSON arguments.

**Cost:** on these requests the listed rules stop inspecting body and query arguments. They need
a signed-in n8n user, who can already run code on the n8n host. Cookies, headers, every other
rule and every other path stay inspected. `949110` is never excluded. Rules of paranoia level 2
and above are not in the list. `9861104` exists because the editor cannot start without that file
and its name is on the CRS restricted-file list.

**`n8n_api_false_positives.conf`**, the same 89 ids on `ARGS`, for scripts and AI clients that
send workflow code: `9861105` `POST /api/v1/workflows`, `9861106` `PUT /api/v1/workflows/<id>`,
`9861107` `POST /mcp-server/http`. Same cost; the callers hold an API key.

**`n8n_webhook_false_positives.conf`**, `POST`, `PUT` and `PATCH` to `/webhook/`,
`/webhook-test/` and `/webhook-waiting/`:

| Id | Rule | Condition |
| --- | --- | --- |
| `9861110` | `920420` on `Content-Type` | the type contains no `json` or `xml` and does not start with `text/` |
| `9861111` | `920340` and `920640` | the request has a body and no `Content-Type` at all |
| `9861112` | `920450` on the `Expect` header name only | `Expect: 100-continue`; every other restricted header stays blocked, with or without `Expect` |

These let file uploads through: `application/octet-stream`, `application/pdf`, images, zips, and
the `Expect: 100-continue` that `curl` sends for bodies over 1 MiB. **Cost:** a body of such a type
has no ModSecurity body processor, so it is never inspected (with or without these rules); the
workflow must treat it as untrusted binary. The JSON, form and multipart bodies on the same path
are inspected as before. Still blocked, by design: `text/plain`, `application/atom+xml`,
`application/vnd.api+json` (`920420`), any request with `X-HTTP-Method-Override` and the other
restricted headers (`920450`), and gzip or deflate request bodies (`Content-Encoding`). The
`Expect` exception names the header as `Expect` (HTTP/1) and `expect` (HTTP/2); a header
spelled `eXpEcT` is still blocked.

**`n8n_forms_false_positives.conf`**: rule `9861122` removes BunkerWeb's rules `200002` and
`200003` for `POST` with a `multipart/form-data` body on `/form/<id>`, `/form-test/<id>` and
`/form-waiting/<id>`. Rule `200004` stays active: it never fired on a browser post. **Cost:** once
the parser fails on an empty-name part it builds no argument list, so the CRS inspects none of
the fields of that request (measured: SQL injection, XSS, shell, path traversal and PHP payloads
in the first or a later field all passed). A truncated or badly bounded multipart body on these
paths is no longer refused by the WAF either, n8n answers it. A form post without an empty-name
part is inspected and blocked as usual, and so are the query string, cookies and headers; a
malformed JSON, XML or `multipart/mixed` body on a form path keeps its block. Only the CRS rules
that also read the raw body (Java `944xxx`, some `920`, `921`, `931` and `934` rules) still
fire. No narrower version held: BunkerWeb defines `200002` and `200003` before the custom
configs, so a rule here cannot lift them after reading the body, and rescanning the raw body
with own rules reproduced only 19 of the 38 payloads and blocked file uploads. The cause is
upstream: the form page appends every form element, nameless submit button included
(`packages/cli/templates/form-trigger.handlebars` in n8n `2.48.4`). Drop this file if you use no
browser forms.

**`n8n_clients_false_positives.conf`**: `9861120` (Chat Trigger) and `9861121` (MCP Server Trigger)
remove the argument name `sessionId` from CRS `943120`, for those exact paths and `POST` only. The
name is a conversation key chosen by the client, not a login session.

When a new node or caller gets `403`, read the rule id and the target in the BunkerWeb log, then
add one `ctl:ruleRemoveTargetById=<id>;<target>` line to the matching chain. Do not remove
`949110`.

## Things to think about

- **Chat Trigger (hosted page and embedded widget).** The browser page sends a `Referer`; curl, a
  server-side integration or a mobile app does not, and CRS `943120` rejects the `sessionId` in
  the body. Rule `9861120` removes that one argument name for `POST /webhook/<webhookId>/chat`. The
  trigger's `Allowed Origins` option sets CORS; the preflight is allowed by the template and n8n
  answers `204`. A message typed in the hosted page in headless Chrome got its reply. The
  streaming response mode was not tested.
- **Forms.** `GET` and `POST /form/<webhookId>` (text, a textarea with SQL-looking text, a file up
  to 20 MiB) and the signed `/form-waiting/<id>` links of a Wait node pass. A browser adds
  empty-name parts to the POST, which ModSecurity cannot parse: `n8n_forms_false_positives.conf`
  (rule `9861122`) removes two BunkerWeb body-parser rules for the form paths. Cost: those posts
  get no CRS inspection of their fields, wherever the empty-name part sits (of 38 attack
  payloads the CRS blocks in a normal post, 2 are still blocked in one with an empty-name
  part), so validate form input in the workflow. Without that file every browser submit fails (the browser reports a CORS
  error, the WAF answered `400`). The signature of waiting links is built from the public URL, so
  a wrong `N8N_WEBHOOK_URL` breaks every link.
- **Wait nodes and "send and wait" links.** `POST /webhook-waiting/<id>` with JSON, a raw body or
  a `GET` with a query string (an approval link) pass. Raw bodies use rules `9861110` to `9861112`,
  which the template extends to `/webhook-test/` and `/webhook-waiting/`.
- **MCP.** Two servers exist: the MCP Server Trigger node (`/mcp/<path>`) and the instance-level
  server (`/mcp-server/http`, enabled in Settings, bearer API key or OAuth). Clients such as
  Claude, Cursor and `mcp-remote` send no `Referer`; the SSE transport posts every message to
  `/mcp/<path>?sessionId=<uuid>`, which `943120` blocks. Rule `9861121` removes that argument name.
  The instance-level server carries workflow code in tool arguments: rule `9861107` gives
  `POST /mcp-server/http` the editor's exception. Tool arguments on the trigger endpoint stay
  inspected, so a tool call with SQL text gets `403` by design. The trigger sends no keep-alive:
  with a 5-minute read timeout its stream was cut at 300.1 s, which the template's `1h` avoids.
- **Public API.** Scripts and CI jobs that sync workflows (header `X-N8N-API-KEY`) send the same
  workflow documents as the editor: `POST /api/v1/workflows` and `PUT /api/v1/workflows/<id>` get
  rules `9861105` and `9861106`. Reads, activation and credentials need nothing. Twenty-one client
  user agents (axios, python-requests, Go, PostmanRuntime, HTTPie, okhttp, Stripe, GitHub, Slack
  and others) all passed. n8n itself answers `204` to bot-like user agents (Slackbot,
  TelegramBot, UptimeRobot) on `GET /api/...`: monitor `/healthz` instead.
- **Many editors behind one address.** Each open editor tab holds a push socket. With BunkerWeb's
  default connection limit (10 per address) 14 tabs from one office address left 4 of them
  refused, and the default `2r/s` on `/rest/push` left 4 open. The template slot and
  `LIMIT_CONN_MAX_HTTP1=100` opened 40.
- **Embedding.** The default `X-Frame-Options` and `frame-ancestors` block the form page, the
  chat page and the editor in an iframe on another site (the browser shows an error page). With
  `X_FRAME_OPTIONS` empty and `frame-ancestors` removed from `CONTENT_SECURITY_POLICY`, the form
  and the chat page load in an iframe. The editor stays blocked by n8n's own
  `X-Frame-Options: SAMEORIGIN`. Do this only when you embed forms or chat. A widget that loads
  `@n8n/chat` from a script tag was not tried in a browser.
- **Webhook JSON that carries code or SQL text.** The CRS still inspects JSON, form and multipart
  bodies of production webhooks: a caller that posts `DROP TABLE` text in a JSON field gets
  `403`. That is intended. Add a narrow `ctl:ruleRemoveTargetById` for one webhook path when you
  know the caller (see the last paragraph of the CRS section).
- **IP whitelist on a Webhook node.** It sees the client address only with `N8N_PROXY_HOPS=1`.
  A whitelisted client address passed with a forged `X-Forwarded-For`; an address that was only
  forged got n8n's `403`.
- **Cookies set by a workflow.** BunkerWeb's default `COOKIE_FLAGS=* HttpOnly SameSite=Lax` adds
  `HttpOnly; SameSite=Lax` to a cookie that a Respond to Webhook node sets (`bwt_pref=dark;
  Path=/` became `bwt_pref=dark; Path=/; HttpOnly; SameSite=Lax`). A cookie meant for browser
  JavaScript needs `COOKIE_FLAGS_1=<cookie name> SameSite=Lax` on the service.
- **Client address and headers.** BunkerWeb sends `X-Forwarded-For` (appended), `X-Real-IP`,
  `X-Forwarded-Proto` and `X-Forwarded-Host` (with the port), and passes a client `Forwarded`
  header through. n8n prefers `Forwarded` over `X-Forwarded-Host` when it checks the push origin
  (`origin-validator.ts:59-84`); a browser cannot set that header on a cross-site request.
  n8n sends `X-Frame-Options: SAMEORIGIN` and a report-only CSP; BunkerWeb adds its own CSP and
  no duplicate `X-Frame-Options` appears. `Referrer-Policy` is BunkerWeb's value, `strict-origin-when-cross-origin`;
  n8n sends `no-referrer`. The `n8n-auth` cookie arrives unchanged.
- **Permissions-Policy.** BunkerWeb's default denies camera, microphone, fullscreen, geolocation
  and display capture. The editor code of n8n 2.48.4 does not call those APIs itself (only
  bundled libraries mention them); voice input in the chat widget was not tried.
- **Metrics.** `N8N_METRICS=true` publishes `/metrics` with no authentication. The template
  returns `403` for it at BunkerWeb. Scrape `http://n8n:5678/metrics` on the private network.
- **Banned senders.** A caller of a webhook that is not published (a typo, an unpublished
  workflow) gets `404`; the default list counts those and bans the address for a day, health
  checks included. The template list drops `404`.
- **Long LLM runs.** n8n's own HTTP client gives up after 5 minutes on one AI request (n8n issue
  11886); BunkerWeb's `1h` timeout is for the idle streams and slow webhooks.
- **Queue mode.** n8n's docs route `/webhook/*` and `/webhook-waiting/*` to webhook processors
  and everything else, `/webhook-test/*` and the editor included, to the main instance. This
  template points all paths at one upstream, so a queue-mode deployment needs a second
  `REVERSE_PROXY_URL` entry. Form paths are not listed in that doc.
- **SSO, LDAP, MFA, OAuth callback.** Not verified (see the end).

## Validation

Run on 2026-11-17 and 2026-11-18: n8n `2.48.4` (SQLite, internal task runner), BunkerWeb
all-in-one `1.6.17` (CRS `4.29.0`), default paranoia level and thresholds,
`SECURITY_MODE=block`, template packed as an external plugin.

- **Request matrix:** 138 requests (71 seeds, 67 controls) and 18 state checks (the saved workflow reads back, the
  execution returns, the webhook echoes or counts the bytes), 0 mismatches on the final files.
  Without the configs the editor requests, the raw-body
  webhooks, the public API and MCP writes, forms and chat get `403` or `400`; with them all pass.
  Controls (another path, method, trailing segment, `..%2f` traversal, other content type,
  header, cookie, scanner agent, an empty-name part on a non-form path) keep the block, and the
  log shows the CRS rule. Every log line was matched to its request by a `?bwt=<name>` marker.
  A second round re-ran the matrix on the final files, with the three form pages in a browser
  (red `5/5`, `1/1` and `1/1` failing without `9861122`, green `200` on all) and 44 webhook rows
  for the per-name `Expect` exception.
- **Settings:** the table above, each setting set back alone to its BunkerWeb default.
- **Browser smoke (headless Chrome, bad behavior and limit_req on, only `SERVER_NAME` and
  `REVERSE_PROXY_HOST` changed):** owner setup, import and save of a workflow with Code nodes
  holding SQL, shell and HTML strings, run from the editor, publish, `POST /webhook/...`, then a
  form submit in the browser (`Form Submitted`) and a message in the hosted chat page (reply
  received). The whole BunkerWeb log of that fresh container holds no ModSecurity denial, no
  warning and no ban.
- **Optional step:** webhook-only mode as described above.
- **BunkerWeb web UI:** the template, packed as an external plugin, is listed on the Templates page with
  15 steps, 22 settings and 5 configs and its steps open. The Raw-mode upload was not repeated
  on this version.

Not verified: SSO, SAML, OIDC and LDAP (the test build has no licence for them), MFA, the
OAuth credential callback and the instance-level MCP OAuth flow with a real provider, queue mode
and workers, the external task runner, community nodes, the chat widget script tag and streaming
responses, HTTPS, HTTP/3 and Let's Encrypt (the test hostname was plain HTTP on `localhost`),
`USE_REAL_IP` behind a CDN, binary downloads, a sub-path deployment, `N8N_PUSH_BACKEND=sse` in a
browser, voice input, a real browser idle past 130 s (a Node client was used), gzip request
bodies and `text/plain` webhooks.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
