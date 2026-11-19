# ntfy Template

Expose an existing [ntfy](https://ntfy.sh) server through BunkerWeb: the web app (sign-in, account,
access tokens, web push), publishing (plain text, headers, JSON, message templates, attachments,
UnifiedPush, the Matrix gateway, webhooks from Grafana, Alertmanager, GitHub, Gitea, Uptime Kuma and
Home Assistant), subscriptions over SSE, WebSocket and long-poll, and the account and admin API.

## Prerequisites

- Run ntfy `2.37.2` or a close release with persistent cache, attachment and auth storage. BunkerWeb
  must reach `http://ntfy:80` on a private network. Do not publish that port: ntfy believes the
  forwarded-IP header of whoever connects (next section).
- For a public server set `auth-default-access: deny-all` and create users with `ntfy user add`. This
  template adds no authentication of its own. Leave `enable-signup` off unless anyone may register.
- Point your public hostname at BunkerWeb. `AUTO_LETS_ENCRYPT=yes` makes BunkerWeb request its
  certificate (default HTTP challenge requires inbound port 80). For existing certificate files,
  use `AUTO_LETS_ENCRYPT=no`, `USE_CUSTOM_SSL=yes`, `CUSTOM_SSL_CERT` and `CUSTOM_SSL_KEY`.
- The ntfy web app shows "Notifications not supported" over plain HTTP: browsers only allow
  notifications on HTTPS. That is a browser rule, not a proxy problem.

## Configure ntfy for the proxy

Set these in ntfy's `server.yml` or as `NTFY_*` variables. BunkerWeb sends `Host`, `X-Forwarded-For`
(appended to what the client sent), `X-Real-IP`, `X-Forwarded-Proto` and `X-Forwarded-Host`.

| ntfy setting | Value | What breaks without it |
| --- | --- | --- |
| `base-url` | `https://ntfy.example.com`, no trailing slash, no path | ntfy writes it into attachment links, `/config.js` and the UnifiedPush discovery reply. Running under a sub-path such as `https://host/ntfy` is not supported upstream ([issue 398](https://github.com/binwiederhier/ntfy/issues/398), [issue 1009](https://github.com/binwiederhier/ntfy/issues/1009)). |
| `behind-proxy` | `true` | ntfy keys every limit on the address that connects, which is BunkerWeb's: "all visitors are rate limited as if they are one" ([config docs](https://docs.ntfy.sh/config/#behind-a-proxy-tls-etc)). With it on, ntfy logged the address BunkerWeb saw as the visitor. |
| `proxy-forwarded-header` | keep `X-Forwarded-For` | ntfy takes the right-most address in the header that is not listed in `proxy-trusted-hosts` (`server/util.go:111-145`, v2.37.2). BunkerWeb appends the address it saw, so that is the one ntfy uses. |
| `proxy-trusted-hosts` | empty when BunkerWeb faces the clients | Only list the proxies that sit in front of BunkerWeb. See the warning below. |
| `keepalive-interval` | default `45s`, always below 60 s | BunkerWeb closes a response that is silent for `REVERSE_PROXY_READ_TIMEOUT` (60 s). With `75s` a JSON stream was closed after 60 s. The Android app drops a stream after 77 s of silence (`HttpUtil.kt:43`). |
| `attachment-file-size-limit` | default `15M`, with `MAX_CLIENT_SIZE=15m` | A larger ntfy limit still gets `413` from BunkerWeb first. Raise both together, also for a tier's larger limit. |
| `visitor-subscription-limit` | default `30` | Stays below the template's `LIMIT_CONN_MAX_HTTP1=40`. Above 30 held streams from one address ntfy answers `429` itself. |
| `enable-metrics` | with `metrics-listen-http` on a private port | Prometheus metrics are "sensitive" ([config docs](https://docs.ntfy.sh/config/#monitoring)). Scrape the private port directly; the template blocks `/metrics` on the public name. |
| `upstream-base-url` | `https://ntfy.sh`, for iOS instant notifications | The iOS app only gets prompt pushes if your server forwards poll requests to ntfy.sh. That call goes out from the ntfy container, not through BunkerWeb. Not tested. |
| `web-push-*` | optional | Needed only for web app notifications while the tab is closed (HTTPS only). |

**Never put the clients' own network in `proxy-trusted-hosts`.** ntfy strips trusted addresses from the
right of the header and uses the first one left over. With `proxy-trusted-hosts` set to the network
BunkerWeb saw the test client in, `X-Forwarded-For: 6.6.6.6` made ntfy log `6.6.6.6` as the visitor,
so that client could dodge every per-visitor limit (and without the header ntfy fell back to the
BunkerWeb container's address). With the setting empty, forged `X-Forwarded-For`, `X-Real-IP`,
`Forwarded` and `X-Client-IP` headers all left the logged visitor at the real address.

**BunkerWeb behind another proxy or a CDN.** BunkerWeb then sees the CDN's address and ntfy would key
every visitor on it. Set `USE_REAL_IP=yes` and `REAL_IP_FROM` to the CDN's ranges on the BunkerWeb
service (keep `USE_REAL_IP=no` otherwise). In the test, with `REAL_IP_FROM` covering the client's
network, `X-Forwarded-For: 5.5.5.5` and `6.6.6.6, 5.5.5.5` both reached ntfy as `5.5.5.5`
([issue 905](https://github.com/binwiederhier/ntfy/issues/905) is this setup).

## Import and initial detect run

Import `template.json` and `configs/modsec-crs/ntfy_false_positives.conf` using the repository
[installation guide](../../README.md#install), or its plugin bundle layout. Assign
`USE_TEMPLATE=ntfy`, update `SERVER_NAME` and `REVERSE_PROXY_HOST` (keep the port, no trailing
slash or path), and review the guided steps. Those two are the only values most setups change.

Before enabling blocking on your instance:

1. Set service `SECURITY_MODE=detect` and reload.
2. Sign in to the web app, change a setting, create a token, subscribe, publish from the dialog,
   publish with `curl -d`, `curl -T file` (an attachment), the JSON form, your webhook senders and
   your real clients (Android, iOS, UnifiedPush, scripts) while reading BunkerWeb logs.
3. Confirm the exclusions match the targets you see in the log.
4. Switch to `SECURITY_MODE=block` and repeat.

In block mode a false positive returns `403`. The default bad-behavior status list includes
`400 401 403 404 405 429 444`: ten matching responses in 60 seconds ban the client for one day. For
ntfy that means a publisher (a cron job, a router, a sensor) whose messages are refused is banned and
its alerts stop without any visible error, and 13 wrong-password logins from one address were enough
to ban it. This is why the detect run matters.

## Settings and tradeoffs

| Setting | Purpose and proof |
| --- | --- |
| `REVERSE_PROXY_URL=/`, `REVERSE_PROXY_WS=yes` | Proxy everything and upgrade `/<topic>/ws`. Without it the WebSocket handshake gets `400`; with it `101`. |
| `ALLOWED_METHODS=GET\|POST\|HEAD\|OPTIONS\|PUT\|DELETE\|PATCH` | ntfy uses `PUT` (publish, `curl -T`), `DELETE` (delete a message), `PATCH` (account settings and tokens) and `OPTIONS` (CORS preflight). The default `GET\|POST\|HEAD\|QUERY` answers `405` to all four. |
| `MAX_CLIENT_SIZE=15m` | Matches ntfy's 15M attachment limit. With the default `10m`, an 11 MiB upload gets `413`; with `15m`, 15 MiB passes and 16 MiB gets `413` from nginx (not `403`). |
| `LIMIT_REQ_URL=/`, `LIMIT_REQ_RATE=100r/s` | BunkerWeb allows 2 requests per second by default (`2r/s`, a sliding one-second window), counted per client address **and exact path** (read from `core/limit/limit.lua`). Publishers that post to one topic, a monitor that polls one URL and a Matrix homeserver hit it; page loads and polls of different topics do not. At the default, 40 publishes to one topic gave 3×`200` and 37×`429`, and 150 gave 6×`200` and 144×`429`; 150 publishes to 150 different topics all gave `200`. 30 parallel pushes to the Matrix gateway gave 2×`200` and 28×`429` at the default and 30×`200` with this template. ntfy's own burst allowance is 60 requests (`config.go:106`), so `100r/s` lets ntfy decide. With `100r/s` all 190 same-topic requests passed. Not measured above 100 requests per second. |
| `LIMIT_CONN_MAX_HTTP1=40` | BunkerWeb allows 10 held connections per address by default, and a held stream or WebSocket counts for each. The web app opens one WebSocket per subscribed topic (`web/src/app/ConnectionManager.js`). With 12 held SSE streams, 9 to 10 got `200` and 2 to 3 got `429` at the default; all 12 got `200` with `40`. In the browser, reloading the web app with 12 subscribed topics showed BunkerWeb's `429` page at the default, and with `40` all 12 topics received a live message. Past 30 streams from one address ntfy answers `429` itself (`visitor-subscription-limit`, `config.go:105`; 45 streams gave 29 to 30×`200`). Over HTTPS a browser speaks HTTP/2 and `LIMIT_CONN_MAX_HTTP2` applies instead; the lab ran plain HTTP, so that value was not measured. |
| `REVERSE_PROXY_INTERCEPT_ERRORS=no` | BunkerWeb swaps every upstream `400`, `401`, `404`, `429` and `5xx` for its own HTML page. ntfy's apps read its JSON error (`{"code":40026,...}`) and the CLI prints it. With the setting, curl got `{"code":40007,...}`, `{"code":40101,...}` and `{"code":40401,...}`, and `ntfy publish` printed `invalid delay parameter`; without it the CLI printed a whole HTML page. In the web app's change-password dialog a wrong current password shows `Unexpected response 400` without the setting and `Password incorrect` with it. A WAF `403` and a stopped upstream (`502`) still show BunkerWeb's page. |
| `BLACKLIST_URI=^/metrics$` | Keeps Prometheus metrics off the public name when ntfy serves them on the main port: `/metrics` answered `200` direct and `403` through BunkerWeb, while a topic named `metrics2` still got `200`. Drop it if you scrape through BunkerWeb. |
| `AUTO_LETS_ENCRYPT=yes` | Convenience for a public hostname. Not exercised in the lab (needs public DNS). |

Measured and deliberately **not** set:

- `REVERSE_PROXY_BUFFERING`: ntfy's nginx example sets `proxy_buffering off`, but a published message
  reaches an SSE subscriber in 0.01 s with BunkerWeb's default `yes`, and the ntfy CLI received a live
  message through it, so `no` is not needed.
- `REVERSE_PROXY_*_TIMEOUT`: SSE, JSON stream and WebSocket subscribers idle for 330 s were still
  connected and received the next message with the default 60 s timeouts, because ntfy sends a
  keepalive every 45 s. If you raise ntfy's `keepalive-interval` above 60 s, raise
  `REVERSE_PROXY_READ_TIMEOUT` above it too. The nginx example's `proxy_request_buffering off` and
  `client_max_body_size 0` do not apply: attachments are capped at 15 MiB.
- `BAD_BEHAVIOR_STATUS_CODES`: dropping `401` (a wrong token or password) only moved the ban from
  the 14th wrong login to the 29th. ntfy answers `429` once its failed-login budget is spent (30,
  refilled one per minute, `config.go:115-116`) and `429` is in the list. The default stays.
- `LIMIT_CONN_MAX_HTTP2`, `USE_CORS`, `COOKIE_FLAGS`: ntfy's own CORS headers pass through, ntfy sets no
  cookies, and the web app logged no Content-Security-Policy violation.
- The request-rate and connection limits are per client address. People behind one NAT or one VPN
  exit share them.

## Things to think about

- **Web app.** Sign-in, account, change password, access tokens, settings, synced subscriptions, topic
  reservation, sign-up, web push and the publish dialog all worked with `SECURITY_MODE=block`, the limiter
  and bad behavior on. The app sends its JSON as `text/plain;charset=UTF-8` (`9864170`), opens one WebSocket
  per subscribed topic (see `LIMIT_CONN_MAX_HTTP1`), and sends the message and title in the query of
  `PUT /<topic>` when you attach a file, because the body is the file (`9864131` covers `message`; shell
  text in the title, or SQL text in the message, is still blocked with `403`). It requests
  `/static/langs/en-GB.json`, which ntfy answers `404`; that counts toward bad behavior (3 of 10 in the
  smoke, back to 0 within a minute). Notifications need HTTPS.
- **Android app.** It publishes with `PUT /<topic>?priority=...&title=...` (`ApiService.kt` publish),
  polls one `GET /<topic>/json?poll=1&since=...` per subscription and holds a streaming `GET
  /<topics>/json` with a 77 s read timeout. Its user agent is `ntfy/<version> (<flavor>; Android
  <release>; SDK <n>)` (`HttpUtil.kt:19`); with that user agent and the F-Droid one, publish and poll
  both answered `200`. A real phone was not tested.
- **iOS app.** It relies on `upstream-base-url` (see the table). Not tested.
- **ntfy CLI.** The official image's `ntfy publish` and `ntfy subscribe` worked through BunkerWeb;
  a subscription received a published message live. With `REVERSE_PROXY_INTERCEPT_ERRORS=no` the CLI
  prints ntfy's JSON error (`invalid delay parameter`); with BunkerWeb's default it printed a whole
  HTML page. `curl -T file` sends `Expect: 100-continue` above 1 MiB, which `9864100` allows.
- **Webhook senders.** Grafana, Alertmanager and GitHub post JSON to `/<topic>?template=grafana`,
  `alertmanager` or `github`; Gitea, Uptime Kuma (webhook and native ntfy provider) and Home Assistant
  post JSON to a topic or to `/`. Their text is shell, SQL, paths and cron lines, which the CRS reads as
  attacks. The exclusions `9864160` and `9864180` cover payloads built from the senders' documented
  shapes (see the CRS section). Shell or SQL text in a `title`, or in a GitHub field other than
  an issue or pull request title or body or a comment body, is still blocked with `403`.
- **Access tokens and basic auth.** `Authorization: Bearer tk_...`, basic auth and the `?auth=`
  query parameter all worked. `?auth=` puts the credential in the URL, and BunkerWeb's access log
  records the whole request line, so the credential is in the log: prefer the header.
- **Email and phone.** ntfy sends them from its own SMTP and Twilio settings, outbound. The `Email` and
  `Call` publish headers pass the WAF (ntfy answered `400` because the lab has neither configured).
  Publishing by e-mail (`smtp-server-listen`) is a plain TCP port that BunkerWeb does not carry.
- **Attachments.** Uploads up to 15 MiB pass, and a message links to `/file/<id>`. ntfy deletes the file
  after `attachment-expiry-duration` (3 h by default) and the link then fails; the expiry itself was
  not waited out. ntfy's own JSON `404` reaches the client with the template (tested on an unknown path). `Range` requests are answered `200` with the full file by ntfy itself.
- **UnifiedPush and the Matrix gateway.** `POST /up<id>?up=1` and `/_matrix/push/v1/notify` pass.
  A Matrix homeserver sends all its pushes from one address to one path, so it exceeds BunkerWeb's
  default `2r/s` at once (28 of 30 parallel pushes got `429`); with `LIMIT_REQ_RATE=100r/s` all 30
  passed. ntfy.sh whitelists such senders for the same reason
  ([issue 144](https://github.com/binwiederhier/ntfy/issues/144)).
- **Scheduled and cached messages.** `Delay`, `At`, `Cache: no` and `Firebase: no` headers passed.
- **Metrics and statistics.** `/metrics` is blocked by the template. `/v1/stats` (message counters)
  and `/v1/health` stay public by ntfy's design; add `/v1/stats` to `BLACKLIST_URI` if you want it
  private (no ntfy client calls it).
- **Embedding.** BunkerWeb adds `X-Frame-Options: SAMEORIGIN` and `frame-ancestors 'self'`, which ntfy
  does not set, so other sites cannot frame the web app. Not tested in a page that embeds it.
- **Web push.** `POST` and `DELETE /v1/webpush` and `/manifest.webmanifest` passed with test keys; the
  service worker `/sw.js` registered. The browser permission prompt and a delivered push were not
  tested (headless Chrome, no push service).
- **Not supported.** Hosting at a sub-path (see `base-url`).

## CRS exception scope and cost

`configs/modsec-crs/ntfy_false_positives.conf` is a service-scoped **modsec-crs** config. Every
chain is scoped by method and path, and uses literal targets unless the table says why not; no rule
is removed for a whole request and `949110` is never touched.

| Id | Scope | What is removed |
| --- | --- | --- |
| `9864100` | `PUT`/`POST` on `/<topic>` and `/<topic>/<id>` (not `/v1/...`) | Rules `920420` (Content-Type not allowed), `920340`/`920640` (no Content-Type with a body) and `920450` for the `Expect` header name only. |
| `9864110` | `PUT`/`POST` on `/<topic>`, urlencoded body | `921150` on argument names, and `930120`, `932125`, `932230`, `932235`, `932250`, `932260`, `932370` on `ARGS` and `ARGS_NAMES`. |
| `9864120` | `POST /` (JSON publish) | The seven rules above on `ARGS:json.message` only. |
| `9864130` | `GET /<topic>/publish\|send\|trigger` (scripts, message in the query) | The seven rules above on `ARGS:message` and `ARGS:m` only. |
| `9864131` | `PUT /<topic>` (the web app's publish dialog with an attachment, message in the query) | The seven rules above on `ARGS:message` only. |
| `9864140` | `POST /_matrix/push/v1/notify` | The seven rules above on `ARGS:json.notification.content.body` only. |
| `9864150` | `POST`/`PUT`/`DELETE` on `/v1/account`, `/v1/account/password`, `/v1/account/password/reset`, `/v1/users` | `932130`, `932230`, `932250`, `933180`, `942100`, `942540` on `ARGS:json.password` and `ARGS:json.new_password` only. |
| `9864160` | `PUT`/`POST` on `/<topic>` with `Content-Type: application/json` | The seven rules above and `932160` on `ARGS`. |
| `9864170` | `PUT`/`POST`/`PATCH`/`DELETE` on `/` and the account, token, settings, subscription, reservation, phone, e-mail and web push paths, with `Content-Type: text/plain` | `920420` for the `Content-Type` header only. |
| `9864180` | `POST /<topic>?template=github`, JSON body | `942100`, `942190`, `934110` on `ARGS:json.issue.body`, `json.issue.title`, `json.pull_request.body`, `json.pull_request.title`, `json.comment.body`. |

Why each exists:

- **Raw bodies (`9864100`).** ntfy takes the message or attachment as the raw body, with whatever
  `Content-Type` the client sends or none (the ntfy CLI and UnifiedPush send none). The four rules only
  judge those headers. curl sends `Expect: 100-continue` for any body over 1 MiB, so the documented
  `curl -T file https://host/topic` is refused without the `920450` entry. ntfy parses nothing from
  these bodies, so no inspection is lost.
- **Message text (`9864110` to `9864140`).** Alert text is shell and log output: paths, `ssh`,
  `docker compose`, `systemctl`, tracebacks, `&&` and `;`. These seven RCE rules fire on it. Only the
  field that carries the message loses them. SQL injection, XSS and every other rule still inspect it,
  and `title`, `tags`, filters and every other field keep all rules.
- **`9864110` is wider than the others.** `curl -d "message"` is urlencoded, so the whole text becomes an
  argument *name* when it has no `=`. Names are arbitrary and cannot be listed, so `ARGS` and
  `ARGS_NAMES` are excluded for those seven rules on that path and method.
- **Message in the query (`9864130`, `9864131`).** ntfy reads `message` from the query on two routes: `GET
  /<topic>/publish|send|trigger` (scripts, ntfy's "publish via GET"; `m` is its alias) and `PUT /<topic>`,
  which the web app's publish dialog uses when a file is attached because the body is the file
  (`PublishDialog.jsx`: always `PUT`, only `message`). Each route has its own rule with its own method and
  path. The ntfy CLI sends `POST` with the message in the body (or an `X-Message` header with `--file`) and the
  Android app a raw `PUT` body, so neither needs these rules. A `POST` with the message in the query, `GET /<topic>?message=`, any deeper
  path and any other query parameter stay inspected. The same text under another name (`x-message`,
  `message[]`, `message2`) is blocked too. A `message` repeated in the query is excluded alike: ntfy keeps
  the first value. `ARGS_GET:message` as a `ctl` target removes nothing on BunkerWeb 1.6.17 (the rules still
  fired on `ARGS:message`), so the exclusion cannot be limited to the query string of a request that also has
  a body.
- **Passwords (`9864150`).** In 300 random 20-character passwords from all printable ASCII, 6 (2%) were
  blocked by `942100`, `932130` or `942540`, so a password manager hits this regularly. This covers
  clients that send JSON with `Content-Type: application/json` (curl, scripts).
- **Webhooks (`9864160`).** The CRS turns a JSON body into `ARGS:json.<path>` including every array
  element (`json.alerts.array_0...`). The length is unbounded and `ctl` takes no pattern, so `ARGS` is
  the narrowest target. The same text sent as `text/plain` is not inspected at all (`9864100`), so this
  opens nothing a plain-text post has not. SQL injection, XSS and every other rule still inspect it.
- **The web app (`9864170`).** `fetch()` with a string body and no header sends
  `text/plain;charset=UTF-8`, and ntfy's web app never sets one (`AccountApi.js`, `Api.js`). Without
  this entry change password, tokens, settings, subscription sync, topic reservation, sign-up, account
  deletion, web push and the publish dialog's `PUT /` all got `403`. Text/plain is not parsed, so
  nothing else is inspected there either way.
- **GitHub text (`9864180`).** An issue or pull request written by a developer holds SQL statements and
  `curl ... | sh` lines, which also trip `942100`, `942190` and `934110`. Only the five fields the
  github template renders lose them, and only when the URL asks for that template.

**Cost:** the seven or eight rules no longer inspect the message text, webhook bodies and password
fields, and `920420` no longer judges text/plain on the web app's API paths. ntfy stores and relays
the text; it does not execute it. The risk moves to whatever *renders* messages (web app, mail, your own
consumers): keep ntfy's web app and consumers escaping output. A `title` or a token label that
contains shell text is still blocked with `403`, as is a subscribe filter with shell-like text; that is
a limit of this template, not a bug.

## Validation and test history

Tested on 2026-11-17 and 2026-11-18 with **ntfy v2.37.2** behind **BunkerWeb 1.6.17** all-in-one
(nginx 1.30.6, libmodsecurity 3.0.17, OWASP CRS 4.29.0), default paranoia level and anomaly thresholds,
`SECURITY_MODE=block`, plain HTTP on `ntfy.localhost`, the template mounted as an external plugin.

- **Request matrix on the final files:** 182 requests, all as expected. 106 are requests that must pass,
  each with a check of the stored message, attachment or account state read back from ntfy directly; 76
  are controls that must stay `403` from a named CRS rule (the same payload on another path, method or
  field, or under a path or name trick). They cover raw text, headers, query and urlencoded publishing, JSON and templates, webhook
  payloads, attachments up to 15 MiB, every subscribe form (SSE, WebSocket, long poll, raw, several topics,
  filters, `since`, `?auth=`), the account and admin API, the web app's own calls, UnifiedPush and the
  Matrix gateway.
- **Without the config, the same requests fail:** the 12 web app calls got `403` from `920420`; 8
  webhook payloads got `403` from `930120`, `932xxx`, `932160`, `942100`, `942190` or `934110`; the
  dialog's attachment publish and a `?m=` publish got `403` from `932235`. With the config they pass.
- **Message in the query, re-proved on 2026-11-18:** in the web app, the publish dialog with a 300 KB file
  and the message `Fix: docker compose pull && docker compose up -d; ssh root@db01 systemctl restart
  postgresql` showed "An error occurred" without `9864130` and `9864131` (`PUT /<topic>?...message=...` got `403`
  from `932230`, `932235`, `932250` and `932370`), and closed with the message and file stored with them.
  63 realistic alert messages (shell, paths, tracebacks, URLs, SQL, HTML, emoji, German, Chinese) sent as
  `GET /<topic>/publish?message=` and as `PUT /<topic>?message=` (for `PUT`, newlines as `\n` like the
  dialog): 53 of 63 pass in each form with the two rules; without them 23 of 59 passed (the four `; ssh
  host` messages added later were not run without). The other 10 trip rules outside the seven (`932120`,
  `932160`, `932380`, `933160`, `934110`, `941xxx`, `942100`, `952110`) and stay `403`. Dropping any one of
  the seven makes a message fail: `932125` (a German alert quoting `ls -la`), `932370` (`; ssh root@db01
  ...`). Before this round the rule covered `GET`, `PUT` and `POST` on `/<topic>` and on the suffixes.
  The narrowing closed `GET /<topic>?message=`, `POST ...?message=` and a urlencoded body `message=` on
  `/<topic>/publish` (all `200` before, `403` now).
- **Settings:** each of `ALLOWED_METHODS`, `MAX_CLIENT_SIZE`, `LIMIT_REQ_RATE`, `LIMIT_CONN_MAX_HTTP1`,
  `REVERSE_PROXY_INTERCEPT_ERRORS` and `BLACKLIST_URI` was run with and without its value; the numbers
  are in the table above.
- **Proxy behaviour:** forged `X-Forwarded-For`, `X-Real-IP`, `Forwarded` and `X-Client-IP`; 20 client user
  agents (Android, iOS, the ntfy CLI, curl, Python, Go, Grafana, Alertmanager, Home Assistant, GitHub,
  Gitea and others, all `200`); 330 s idle SSE, JSON and WebSocket streams; the ntfy CLI.
- **Browser smoke** (headless Chrome, limiter and bad behavior on): cold load, subscribe, live messages
  from `curl -d`, a 2 MiB `curl -T` attachment, a publish from the dialog with a 300 KB attachment,
  admin sign-in, access token creation, and a message after 125 s idle. The BunkerWeb log slice had 73
  requests, no `403` or `444` and no ModSecurity line. The 2026-11-18 re-run on the final files (dialog
  publish with shell text in the message, `GET /<topic>/publish?message=` with shell text, `curl -d`,
  a 2 MiB `curl -T`, admin sign-in, token creation, 125 s idle) had 147 requests, no `403` or `444`, no
  ModSecurity line, and four `404` (`/static/langs/en-GB.json`).

Not verified: Let's Encrypt, the Android and iOS apps on real devices, UnifiedPush distributors,
e-mail and phone delivery, web push delivery and the browser permission prompt, Stripe billing,
the `upstream-base-url` forwarding to ntfy.sh, authentication through an upstream SSO, TLS and HTTP/2
(the lab uses plain HTTP), and request rates other than the cases above. Real clients and deployments
need their own detect run.
