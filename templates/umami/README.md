# Umami Template

Put an existing [Umami](https://umami.is/) analytics server behind BunkerWeb with ModSecurity and the
OWASP CRS on. One service serves both halves of Umami: the dashboard for its owner, and the tracker
script (`/script.js`) plus the collect endpoint (`POST /api/send`) that visitors' browsers call from
other sites.

## Prerequisites

- Run Umami `3.4.1` (PostgreSQL) or a close release. BunkerWeb must reach `http://umami:3000` on a
  private network; do not expose that port publicly.
- Set a strong `APP_SECRET`. A new Umami database has the login `admin` / `umami`: change the
  password before the service is reachable.
- Point your public hostname at BunkerWeb. `AUTO_LETS_ENCRYPT=yes` makes BunkerWeb request its
  certificate (the default HTTP challenge needs inbound port 80). For existing certificate files, use
  `AUTO_LETS_ENCRYPT=no`, `USE_CUSTOM_SSL=yes`, `CUSTOM_SSL_CERT` and `CUSTOM_SSL_KEY`.
- Embed the tracker the usual way:
  `<script defer src="https://umami.example.com/script.js" data-website-id="<id>"></script>`.
- Renamed tracker paths are optional. If you set `COLLECT_API_ENDPOINT`:
  - Do not let the value contain `/api/send`. At every container start Umami runs
    `scripts/update-tracker.js`, which replaces each `/api/send` in the tracker with the value, so a
    value that contains it stacks on every restart.
  - Serve the script from the root (`TRACKER_SCRIPT_NAME=/bw-umami.js`). The tracker builds its collect
    URL from the directory of the script URL plus the endpoint: with the script at `/stats/script.js`
    and the endpoint `/stats/api/collect`, the browser posted to `/stats/stats/api/collect` and got
    `404` (tested). Setting `data-host-url` on the script tag should avoid that; not tested.
  - Add `examples/umami_renamed_paths.conf` yourself: it is not part of the template, so a plain import
    removes nothing for the renamed path and a request to it is inspected like any other. Change the
    path in the file to your value, then add it to the service as a custom config of type `modsec-crs`
    (BunkerWeb UI: Configs, New custom configuration, type `modsec-crs`, this service, the content of
    the file). Without that file, event data with HTML or shell text sent to the renamed path gets `403`.
- Umami drops requests from bots: the collect endpoint answers `200 {"beep":"boop"}` and stores
  nothing for headless browsers (tested) and other bot user agents. Test with a normal browser, or set
  `DISABLE_BOT_CHECK=1` on a test instance only.

Not covered: session replay and heatmaps (`/recorder.js`, `POST /api/record`). A replay body is a DOM
snapshot, and a probe request got `403` from CRS `934100`, `934190`, `941180`, `941390` and `942100`.
The template does not exclude them; leave those features off, or write your own narrow exclusion.

Set the Umami variables in [Configure Umami for the proxy](#configure-umami-for-the-proxy) before you
point visitors at the service.

## Import and initial detect run

Import `template.json` and the file under `configs/modsec-crs/` using the repository
[installation guide](../../README.md#install), or its plugin bundle layout. Assign
`USE_TEMPLATE=umami`, then change only `SERVER_NAME` and `REVERSE_PROXY_HOST` (keep the upstream URL
without a trailing slash or path). In the web UI raw import, the config dialog lists the file as
`modsec_crs/umami_false_positives.conf`: upload the file of the same name there. The renamed-path
file under `examples/` is not imported (see Prerequisites).

Before enabling blocking on your instance:

1. Set service `SECURITY_MODE=detect` and reload.
2. Open a page that embeds the tracker, click an element with `data-umami-event`, then sign in to the
   dashboard, open a website, filter by a path and a query string, and open Events and a report, while
   reading BunkerWeb logs.
3. Confirm the included exclusions match the targets in your logs. Narrow or add exceptions only where
   your traffic requires them.
4. Switch to `SECURITY_MODE=block` after those checks and repeat the workflows.

False positives return `403` in block mode. The template sets the bad-behavior status list to
`401 403 404 405 429 444` (the default also has `400`): about ten matching responses in 60 seconds ban
the client for one day.
The `401` Umami answers before sign-in counts as one hit and decays after about a minute.

## Settings and tradeoffs

| Setting                                         | Purpose                                                                                                                                                                         |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `REVERSE_PROXY_URL=/`                           | Proxy the whole app: dashboard, tracker script and collect endpoint share one host.                                                                                             |
| `REVERSE_PROXY_INTERCEPT_ERRORS=no`             | Keep Umami's own JSON errors. With `yes` BunkerWeb swaps every upstream `401` for its HTML page, and a wrong password shows `Unexpected token '<', "<!doctype "... is not valid JSON`. |
| `ALLOWED_METHODS=GET\|POST\|HEAD\|OPTIONS\|DELETE` | `OPTIONS` is the browser preflight before every cross-origin tracker `POST` (`405` at the default), `DELETE` removes websites, reports and teams in the dashboard. Umami 3.4.1 uses no `PUT` or `PATCH`. |
| `LIMIT_REQ_RATE=100r/s`                         | A cold dashboard page made up to 57 requests in one second. At the default `2r/s` the same flow got `429` and then a ban (see below).                                           |
| `LIMIT_REQ_URL_1=/api/auth/login`, `LIMIT_REQ_RATE_1=5r/m` | Five login tries per minute per client IP. Of 14 wrong logins in a row the first five got `401` and the other nine `429`; without these two settings all 14 got `401`. Other paths keep `100r/s`. |
| `BAD_BEHAVIOR_STATUS_CODES=401 403 404 405 429 444` | The default list also has `400`, which Umami answers to a tracker that sends an unknown website id. 14 such `POST /api/send` requests, then 10 s later `GET /login` and a valid tracker `POST`: `403` for both with the default list (the client was banned), `200` for both with this list. |

The methods apply to the whole service, so the dashboard API accepts `DELETE` and `OPTIONS` as well.
Tracker requests come from each visitor's own IP, so the rate limit counts per visitor, not per site.
`LIMIT_CONN_MAX_HTTP1` (default `10` connections per client IP) is not changed: a headless dashboard
session on plain HTTP/1.1 stayed under it. Visitors who share one IP also share these limits, see
[Things to think about](#things-to-think-about).

Left at their defaults on purpose:

- `USE_CORS` stays `no`. Umami sends its own `Access-Control-Allow-Origin: *` on the script, the collect
  endpoint and the preflight, and BunkerWeb passes it through unchanged.
- `REVERSE_PROXY_WS` stays `no`. Umami 3.4.1 has no WebSocket; its realtime page polls over HTTP.
- No antibot challenge is configured. A tracker `POST` cannot answer a challenge, so if you enable
  `USE_ANTIBOT` on this service, exempt `/script.js` and the collect path (`/api/send`, or your
  renamed one) with `ANTIBOT_IGNORE_URI`. That combination was not tested.
- `MAX_CLIENT_SIZE` stays at `10m`: no request in the matrix was larger than a few KB.

## Configure Umami for the proxy

Set these in Umami's environment. All of them were tested through BunkerWeb 1.6.17 unless the last
column says otherwise.

| Umami variable                                | Value                           | What breaks without it                                                                                                                                                                                                                                                                                                                       |
| --------------------------------------------- | ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CLIENT_IP_HEADER`                            | `X-Real-IP`                     | Umami uses the first of `True-Client-IP`, `CF-Connecting-IP`, `Fastly-Client-IP`, `X-NF-Client-Connection-IP`, `DO-Connecting-IP`, `X-Real-IP`, ... that is present. BunkerWeb passes a client's own `True-Client-IP`, `CF-Connecting-IP`, `Fastly-Client-IP` and `DO-Connecting-IP` through, and each one changed the stored visitor. BunkerWeb overwrites `X-Real-IP` with the client address. With `CLIENT_IP_HEADER=X-Real-IP` none of the forged headers changed the stored visitor. |
| `SKIP_LOCATION_HEADERS`                       | `1`                             | Umami reads country and city from request headers before its own database. A client's `CF-IPCountry: JP` and `CF-IPCity: Tokyo` were stored as `JP` and `Tokyo`. With the variable set (and `CLIENT_IP_HEADER=X-Real-IP`), the same headers stored no location.                                                  |
| `ALLOWED_FRAME_URLS`                          | the origin that embeds a share page | A page on another origin that embeds `/share/<slug>` in an iframe stays blank (Chrome showed an error page). With the origin set the share page rendered, although BunkerWeb adds `X-Frame-Options: SAMEORIGIN`: Umami's `Content-Security-Policy` `frame-ancestors` then lists the origin and Chrome applied it. Other browsers were not tested. The Umami docs call this variable build-time; as a container variable it took effect. |
| `MCP_ENABLED`                                 | `1`, only if you use `/mcp`     | The route answers `404` unless it is `1` (`src/app/mcp/route.ts:41-43`). With it, a client with an API key sent as `Authorization: Bearer <key>` got `200` for `initialize`, `tools/list` and `tools/call` through the service, and the same `POST` without a key got `401`.                                                                                                                      |
| `TRACKER_SCRIPT_NAME`, `COLLECT_API_ENDPOINT` | only for renamed paths          | See Prerequisites. A collect path at the root (`/collect`) answered the preflight and the `POST` with Umami's `Access-Control-Allow-Origin: *` through BunkerWeb.                                                                                                                                                                             |
| `BASE_PATH`                                   | leave empty                     | It is a build argument (`Dockerfile` lines 36-38 at `v3.4.1`), so the official image cannot run under a sub-path. Umami's issue 4064 says the same. Use a hostname of its own. Running it under a sub-path was not verified.                                                                                                                                      |

Security: `CLIENT_IP_HEADER=X-Real-IP` trusts whatever sets that header. Only BunkerWeb may reach
Umami's port. A request sent straight to Umami with `X-Real-IP: 8.8.8.8` and one with `9.9.9.9` got two
different visitor ids, so a client that reaches the port picks its own identity. Umami 3.4.1 has no
trusted-proxy list (`TRUSTED_PROXIES` appears nowhere in its source). BunkerWeb's `X-Forwarded-For`
appends the client address to a client-supplied value (`6.6.6.6, 172.21.0.1`); Umami checks that
header after `X-Real-IP`, and a forged value did not change the stored visitor in the test.

## Things to think about

- **Visitors who share one IP (office, school, mobile carrier).** The limits count per client IP.
  600 tracker `POST` requests from one IP in one HTTP/2 burst (100 streams in parallel) got 200
  `200` and 400 `429` at `LIMIT_REQ_RATE=100r/s`, and all 600 `200` at `1000r/s`. Over plain HTTP/1.1, 50
  parallel connections got 455 `429` out of 600 from the connection limit
  (`LIMIT_CONN_MAX_HTTP1=10`, log line `limiting connections by zone v1ips`). `429` is in the default
  bad-behavior list. For a site with that much traffic from one address, raise `LIMIT_REQ_RATE`;
  raising `LIMIT_CONN_MAX_HTTP1` was not tested.
- **Stale or mistyped website id in a tracker snippet.** Umami answers `400`, and with the default
  bad-behavior list ten of them ban the visitor from the whole service for a day, dashboard
  included. The template drops `400` from the list (`BAD_BEHAVIOR_STATUS_CODES`). Fix the snippet
  anyway: those visits are not recorded.
- **Bad behavior reacts a few seconds late.** The counters were logged about five seconds after the
  responses, so 14 wrong logins in a row all got an answer, and the right password worked right
  after. The login limit above answers at once; keep it.
- **`IGNORE_IP` in Umami.** Umami answers `403` to tracker requests from an ignored IP. 14 of them
  from one address got `403` and ten seconds later `GET /login` got `403` too (banned). With that
  address in `WHITELIST_IP` on the service, `GET /login` got `200`. Whitelist the address in
  BunkerWeb, or do not use `IGNORE_IP`.
- **IP and user agent in the request body.** A `payload.ip` field in the JSON `POST /api/send`
  overrides every header (the stored visitor changed with `CLIENT_IP_HEADER` set). No BunkerWeb
  setting stops it, and the website id is public in each page's source. Treat unique-visitor and
  location counts as untrusted against deliberate abuse.
- **Uptime monitors and API clients.** `GET /api/heartbeat` returned `200` without a sign-in for
  `curl`, Uptime-Kuma, UptimeRobot, Prometheus, `node`, `axios`, `undici`, `python-requests`,
  `Go-http-client`, `okhttp`, Googlebot, a mobile Safari agent and an empty agent. Only `sqlmap/1.7.2`
  got `403` (BunkerWeb's user-agent blacklist). Umami accepted an API key as
  `Authorization: Bearer <key>` (`200`) and answered `401` to an `x-umami-api-key` header, so an
  client that sends the second form fails.
- **Browsers behind BunkerWeb's headers.** BunkerWeb adds `X-Frame-Options: SAMEORIGIN` and, over
  HTTPS, `Strict-Transport-Security` (`max-age=63072000; includeSubDomains; preload`). Umami's own
  `Content-Security-Policy` arrives once. Tested over HTTPS with a self-signed certificate; a
  Let's Encrypt certificate was not part of the run.
- **Ad blockers.** The renamed-path example exists for them. It was tested only against a headless
  browser that has no blocker, so whether a given blocker list lets the renamed script through is
  not verified.
- **Tracking pages served from `localhost`.** A tracker `POST` whose page URL or referrer holds
  `localhost/` (`http://localhost/`, `http://x.localhost/`) is blocked by CRS `934190` (SSRF,
  scheme-less localhost) unless that id is skipped; rule `9875100` now skips it, so those pageviews, a
  `http://localhost/` referrer and event data holding that URL are stored. Other places that carry a
  URL you type stay blocked on purpose, with `403` from `934190`: a short-link target
  `http://localhost/`, a website domain typed as the URL `http://localhost/`, and a dashboard
  filter typed as `http://localhost/` (the dashboard itself sends the bare domain,
  `referrer=eq.x.localhost`, which got `200`). A website domain of `localhost` or `x.localhost` got
  `200` without any exclusion.
- **Session replay, heatmaps.** Not supported through this template: see Prerequisites.

## CRS exception scope and cost

`configs/modsec-crs/umami_false_positives.conf` is a service-scoped **modsec-crs** config. Every
exclusion is one chained rule with the HTTP method and an anchored path. `949110` and `959100` are
never touched. A name below that says "by id" skips whole rules for that request; the others skip one
argument.

| Id            | Method and path                                                              | Skipped                                                 | Rules                                                                              |
| ------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `9875100`       | `POST /api/send`                                                             | by id, the whole request                                | 22 ids from `930`, `931`, `932`, `934`, `941`, `942`                                 |
| `9875101`       | `POST /api/auth/login`                                                       | `json.password`                                         | `930100 930110 930120 932130 932160 932235 942100`                                 |
| `9875102-9875104` | `POST /api/users`, `/api/users/<id>`, `/api/me/password`                     | `json.password`, `json.currentPassword`, `json.newPassword` | the same seven                                                                 |
| `9875110-9875124` | `GET /api/websites/<id>/...` and the dashboard pages `/websites/<id>/...`    | one filter argument per rule                            | 16 ids from `930`, `932`, `941`, `942`                                             |
| `9875125`       | the same paths                                                               | `REQUEST_URI_RAW`                                       | `930100 930110`                                                                    |
| `9875127`       | `GET`/`POST` for assets, API and pages, with a dashboard page as `Referer`   | the `Referer` header                                    | `941110 941120 941160 942100`                                                      |
| `9875150`       | `POST /api/reports`, `/api/reports/<type>`                                   | by id                                                   | 18 ids                                                                             |
| `9875151`       | `POST /api/websites/<id>/(funnels\|goals\|segments\|reports)[/<id>]`         | by id                                                   | the same 18                                                                        |

The filter arguments are `path query referrer title event utmSource utmMedium utmCampaign utmContent
utmTerm tag hostname distinctId language search`. The example file `examples/umami_renamed_paths.conf`
(not imported with the template) holds rule `9875160`, the same exclusion as `9875100` for
`POST /stats/api/collect`; it is active only after you add it as a custom config.

Why each one exists:

- **Tracker (`9875100`).** A pageview carries the page's URL with its query string, its title and the
  referrer; an event carries a name and free-form data. Real pages hold `?q=' OR 1=1`, `<script>`
  examples and `../` in all of them. Event data keys are chosen by the site owner and are unbounded,
  and BunkerWeb 1.6.17 rejects regex targets in `ctl:` (the reload fails), so no target can be named.
  The 22 ids are the union the matrix recorded for URL, referrer, title, event name, event data and
  identify data; `934190` is the one for a page URL or referrer on `localhost`. **Cost:** those rules no longer inspect any part of a `POST /api/send` request, the
  query string and headers included. Other rules still do (protocol, scanner user agents, PHP, Java
  and log4j patterns, SSRF ids outside the list).
- **Passwords (`9875101-9875104`).** Password managers produce `a;b|c&d$HOME1` (CRS `932160`), `$(x)`
  (`932130`, `932235`), `' or '1'='1` (`942100`) and `../` runs (`930100`). Only the password arguments
  are skipped, the username is still inspected.
- **Dashboard filters (`9875110-9875125`).** Every filter value is data a visitor sent, and clicking it
  in the dashboard sends it back as an argument. A bot's `?file=../../etc/passwd` appears in the top
  queries, and filtering by it needs `930110` and `930100` skipped on the raw URI too (`9875125`).
  Search boxes on the data pages use `search`. Lists of websites, users and teams are not excluded.
- **Referer (`9875127`).** After a filtered dashboard page loads, the browser sends its URL as `Referer`
  on every asset and API call, and CRS `942100`, `941110`, `941120`, `941160` read that header: the
  page's own assets got `403`. Browsers send cross-site requests an origin-only `Referer` by default, so this
  does not touch tracker traffic. A forged `Referer` can only skip those four rules for its own header.
- **Reports and definitions (`9875150`, `9875151`).** Funnel steps, journey and attribution steps, goal
  values and segment filters are arrays of unbounded length (`json.parameters.steps.array_N.value`)
  holding the same paths and event names. By id again, only on those endpoints. Annotations needed
  nothing and are not listed.

**Overall cost:** a stored-XSS or SQL injection string inside tracker data, a dashboard filter, a
password or a report is not stopped by the WAF. Umami has to handle that content safely; in the
smoke test the dashboard showed a website named `<b>1=1</b>` as plain text. Everything else
(other arguments, paths, methods, headers, cookies and every other CRS rule) stays active.

## Validation and test history

Tested on 2026-11-17: Umami `3.4.1` (PostgreSQL 16), `bunkerity/bunkerweb-all-in-one:1.6.17`
(CRS 4.29.0), default paranoia level and anomaly thresholds,
`SECURITY_MODE=block`, plain HTTP on a local test host.

- **Request matrix:** 219 requests that must succeed and 37 controls, one at a time, with ModSecurity
  lines attributed per request. Without the config 174 of the first 210 got `403` with the ids above;
  the nine `localhost` seeds were run against the config before `934190` was added: the five
  pageview, referrer and event-data seeds got `403` from `934190`, the four others passed. With the
  final config all 219 pass and read back (database rows for pageviews, referrers, events, event
  data and identify data, API reads of websites, teams, reports and segments, logins, website
  domains). The other 40 need no exclusion; `S01` (preflight) and `D17` (delete) need
  `ALLOWED_METHODS` instead and got `405` without it. All 37 controls (the same payload on
  `/api/send/`, `/api/Send`, `GET`, another path or another argument, a class outside the list such
  as log4j, PHP or a scanner user agent, and a `localhost` URL on a path or in a field that is not
  excluded) still got `403` from the named rule. The final run raised `LIMIT_REQ_RATE_1` to
  `1000r/s` (test only), because its login requests come within one minute; without that, five
  login rows got `429`.
- **Renamed paths:** with `TRACKER_SCRIPT_NAME=/bw-umami.js` and `COLLECT_API_ENDPOINT=/stats/api/collect`,
  event data with HTML and shell text got `403` without `umami_renamed_paths.conf` and `200` with it
  (as a custom config on the service),
  and a `http://localhost/` pageview got `403` from `934190` before that id was added to the file and
  `200` after. Browser smoke
  recorded the pageview through the renamed paths.
- **Settings:** `ALLOWED_METHODS`, `REVERSE_PROXY_INTERCEPT_ERRORS` and `LIMIT_REQ_RATE` were each
  switched off alone and the failures above reproduced. At `2r/s` the dashboard flow got 11 `429`,
  then 14 `403` once bad behavior banned the client. `LIMIT_REQ_URL_1` with `LIMIT_REQ_RATE_1` and
  `BAD_BEHAVIOR_STATUS_CODES` were each removed from the template in turn: the login limit and the
  stale-tracker ban behaved as the table above says in both states.
- **Browser smoke (bad behavior on):** a page on another origin embedded the tracker, a visitor
  clicked a tracked element, and the dashboard showed the pageview and both events. The owner signed
  in (a wrong password first showed Umami's message), opened a website, a filtered Events page whose
  URL held `' OR 1=1--` and `../`, and the funnels page. The BunkerWeb log showed no ModSecurity
  line, no `403`, no `429` and no ban for either path set. A `400` came from Umami for a
  half-defined test funnel.
- **Web UI import:** raw mode with `template.json` and both config files imports and is listed with
  7 steps, 11 settings and 2 configs.
- **Proxy research:** forged client-IP and location headers, HTTPS and HTTP/2 headers and bursts,
  login and stale-tracker bans, `IGNORE_IP`, a share page in an iframe, `/mcp` and a root-level
  collect path were each measured through the service; the numbers are in the sections above.

Not verified: Let's Encrypt issuance (HTTPS ran on a self-signed certificate), BunkerWeb behind
another proxy or CDN (`USE_REAL_IP` and the real client address), embedding a share page in Firefox
or Safari, running under a sub-path, `data-host-url`, renamed paths against real ad-blocker lists,
Redis or ClickHouse backends, a PostgreSQL server whose default time zone is not UTC (Umami 3 on a
non-UTC cluster stored events an hour or two early for the maintainers), SSO and two-factor sign-in,
link and pixel URLs (`/q/<slug>`, `/p/<slug>`) with attack-like query strings (they are plain GETs and
stay inspected), and Umami versions other than `3.4.1`. Validate your own tracked sites.
