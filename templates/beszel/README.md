<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/beszel.svg" alt="Beszel logo" width="96" height="96" />
</p>

<h1 align="center">Beszel</h1>

<p align="center">Monitoring hub template with WebSocket agent connections</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Beszel-0.21.2-248651?labelColor=1d6f93&style=flat" alt="Beszel 0.21.2" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#configure-beszel-for-the-proxy">Configure Beszel</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#crs-exception-scope-and-cost">CRS exceptions</a> ·
  <a href="#validation">Validation</a> ·
  <a href="#things-to-think-about">Things to think about</a>
</p>

Expose an existing Beszel hub through BunkerWeb: the dashboard and its live charts, the
PocketBase API and superuser UI under it, and agents that connect to the hub over WebSocket.

## At a glance

| Fact        | Value                                        |
| ----------- | -------------------------------------------- |
| Template id | `beszel`                                     |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Beszel 0.21.2 |
| Configs     | 1 (modsec-crs)                               |
| Rule ids    | `9852000` to `9852999`                       |
| Upstream    | `http://beszel:8090`                         |

## Prerequisites

- Run the Beszel hub with persistent `/beszel_data`. BunkerWeb must reach `http://beszel:8090` on
  a private network; do not publish port `8090`.
- Set the hub's `APP_URL=https://beszel.example.com` to your public URL. Alert links and the agent
  setup text use it (see the
  [hub environment variables](https://beszel.dev/guide/environment-variables)). Serving the hub
  under a sub-path is not covered by this template.
- Create the admin account before the site is reachable. The first visitor to `/` creates it
  (`POST /api/beszel/create-user` works only while no user exists,
  `internal/users/users.go:90-150`), or set `USER_EMAIL` and `USER_PASSWORD` on the hub.
- Point your public hostname at BunkerWeb. `AUTO_LETS_ENCRYPT=yes` makes BunkerWeb request its
  certificate (the default HTTP challenge needs inbound port 80). For your own certificate files
  use `AUTO_LETS_ENCRYPT=no`, `USE_CUSTOM_SSL=yes`, `CUSTOM_SSL_CERT` and `CUSTOM_SSL_KEY`.
- Agents in **WebSocket mode** dial the hub: give them `HUB_URL=https://beszel.example.com`, the
  `TOKEN` and the `KEY` from the Add System dialog (or a universal token). Agents in **SSH mode**
  are dialled by the hub on port `45876` and never cross BunkerWeb.

## Configure Beszel for the proxy

| Beszel setting | Value | What was measured or read |
| --- | --- | --- |
| `APP_URL` (hub) | `https://beszel.example.com` | Used for alert and notification links and for the agent setup text (upstream docs). Required when the hub is served under a sub-path. |
| _Settings, Application, Trusted proxy headers_ in the superuser UI (`/_/#/settings`) | `X-Real-IP` | Without it PocketBase logs BunkerWeb's address as the client of every request (`userIP` `172.22.0.4`, the BunkerWeb container). With it the log shows the real client (`172.22.0.1`) even when the request carries a forged `X-Real-IP: 8.8.8.8` and `X-Forwarded-For: 6.6.6.6`, because BunkerWeb replaces both (`r5-pbip.log`, `r5-EF.log`). |
| `TRUSTED_AUTH_HEADER` (hub, only if you use it) | the header your authenticating proxy sets | **BunkerWeb passes a client-sent header on.** With `TRUSTED_AUTH_HEADER=X-Auth-Email` and `TRUSTED_PROXY_IPS` set to the Docker network, `GET /api/collections/systems/records` with only `X-Auth-Email: <admin address>` returned the admin's systems through BunkerWeb (2 systems in total, 0 without the header, `r5-trusted.log`). `TRUSTED_PROXY_IPS` matches the connection address, which is BunkerWeb's, so it does not help. A `REVERSE_PROXY_HEADERS` entry `X-Auth-Email ""` blanked the header (the same request then returned 0 systems), but then the header cannot carry a login either. Forward authentication in front of Beszel was not tested. |
| `AUTO_LOGIN` (hub) | leave unset | Logs every request in as that user (upstream docs). Never set it on an exposed hub. |
| OAuth2 or OIDC | redirect URI `https://beszel.example.com/api/oauth2-redirect` | Set `USER_CREATION=true` for automatic account creation. With `OAUTH_DISABLE_POPUP=true` also register the base URL with a trailing slash (upstream issue #2170). Editing the users collection needs "hide collection create and edit controls" switched off in `/_/#/settings`, and BunkerWeb needs rule `9852104` of this template for the save (see below). The flow against a real identity provider was not tested. |

Security: set the trusted proxy header only when the hub is reachable through BunkerWeb alone.
If a client can reach port `8090` directly it chooses the address PocketBase records. The hub also
takes the address of an agent that registers with a universal token from `CF-Connecting-IP`, then
`X-Real-IP`, then the first `X-Forwarded-For` entry (`internal/hub/agent_connect.go:314-337`);
see the header setting below.

## Install

Import `template.json` and `configs/modsec-crs/beszel_false_positives.conf` using the repository
[installation guide](../../README.md#install), or its plugin bundle layout. Assign
`USE_TEMPLATE=beszel`, update `SERVER_NAME` and `REVERSE_PROXY_HOST` (keep the port, `8090`, and
no trailing slash), and review the guided steps.

Before enabling blocking on your instance:

1. Set the service `SECURITY_MODE=detect` and reload.
2. Use the dashboard, a system page, the notification settings, the superuser UI and your agents
   while reading BunkerWeb logs.
3. Confirm the included exclusions match the logged targets; narrow or extend them only where your
   traffic requires it.
4. Switch to `SECURITY_MODE=block` and repeat your workflows.

In block mode a false positive returns `403`, and the default bad-behavior list
(`400 401 403 404 405 429 444`, ten matching responses in 60 seconds) bans the client for a day.
Beszel itself answers a few `404` and `400` in normal use: a new system page asks for
`system_details` (one `404` in the smoke) and a wrong password is a `400`.

## Settings

Every row was measured with the setting and with BunkerWeb's value (`evidence/r5-*.log` in the
lane folder; hub and agent `0.21.2`, BunkerWeb `1.6.17`, plain HTTP on loopback unless stated).

| Setting | Why | Measured with and without |
| --- | --- | --- |
| `REVERSE_PROXY_URL=/`, `REVERSE_PROXY_HOST` | Proxy the dashboard, the API and the superuser UI. | |
| `REVERSE_PROXY_WS=yes` | Agents upgrade `/api/beszel/agent-connect` to a WebSocket. | With it the agent connects (`101`). Without it every attempt gets `400`, repeated every 10 seconds (`r5-NOWS.log`). |
| `REVERSE_PROXY_INTERCEPT_ERRORS=no` | Keep PocketBase's own error answers. | A wrong password returns `400` with `{"message":"Failed to authenticate."}` as JSON; with BunkerWeb's default the same request returns its own HTML `400` page (`r5-F.log`, `r5-W.log`). |
| `ALLOWED_METHODS=GET\|POST\|HEAD\|PATCH\|DELETE` | PocketBase edits with `PATCH` and removes with `DELETE`. | `PATCH` of a system `200` and `DELETE` `204`; with the default list both get `405`. |
| `LIMIT_REQ_RATE=50r/s` | A page load is a burst of small requests. | A cold sign-in and system page is 42 requests with 22 in one second (`coldload-signin.log`); the sign-in page alone is 11. At `50r/s` a burst of 160 requests over 16 connections gets 50 `200` and 110 `429`, and at BunkerWeb's `2r/s` it gets 2 `200` and 158 `429`. The value is about 2.3 times the observed peak. Several tabs loading at once behind one address were not measured. |
| `REVERSE_PROXY_READ_TIMEOUT=360s` | The upstream nginx and Caddy examples use `360s`. | The live-update stream (`/api/realtime`) sends nothing while idle. With `360s` it stayed open for the 130 second probe; at the default `60s` the proxy closed it at 60.19 seconds. The browser then reconnects by itself and the update still arrived (2 stream requests and 5 subscribe requests in 140 seconds, `r5-T60.log`), so this setting saves a reconnect each minute and covers a slow agent collection, it is not what makes the dashboard work. The real agent stayed connected for 600 seconds at both values. |
| `REVERSE_PROXY_HEADERS=X-Forwarded-For $remote_addr;CF-Connecting-IP $remote_addr` | The hub reads `CF-Connecting-IP` first. | Against an echo backend a client that sends `X-Forwarded-For: 6.6.6.6` and `CF-Connecting-IP: 7.7.7.7` gets `xff 172.22.0.1` and `cf 172.22.0.1`; with BunkerWeb's value the upstream receives `6.6.6.6, 172.22.0.1` and `7.7.7.7` (`r5-EF.log`, `r5-EW.log`). A second real agent that registered itself with a universal token and sent those headers was stored with host `7.7.7.7` without the setting and `172.22.0.6` (its real address) with it (`r5-W.log`, `r5-F.log`). `X-Real-IP` is overwritten by BunkerWeb in both. |
| `MAX_CLIENT_SIZE=20m` | The superuser UI uploads a backup ZIP to `/api/backups/upload`. | The hub itself rejects a body above 16 MiB with `400` (reproduced against the hub directly). 9, 11 and 16 MiB uploads returned `204`, 17 MiB `400` from the hub, 21 MiB `413` from BunkerWeb. At BunkerWeb's `10m`, 11 MiB and above get `413`. Enter `10m`, BunkerWeb's default, if you never upload a backup; an empty field keeps `20m`. |

Not set by the template, with the numbers to know:

- `LIMIT_CONN_MAX_HTTP1` and `LIMIT_CONN_MAX_HTTP2` count held streams per address, and every open
  dashboard tab holds one (`/api/realtime`). Over plain HTTP the `HTTP1` limit applies: 10 streams
  passed and 4 got `429` at the default `10`, and while the limit was reached a plain request got
  `429` too. Over HTTPS the browser speaks HTTP/2 and the `HTTP2` limit applies instead: 100 of 130
  streams passed at the default `100`, and `LIMIT_CONN_MAX_HTTP1=5` did not touch 20 streams
  (`r5-C10.log`, `r5-C100.log`, `r5-h2.log`). The default is enough for a household; raise
  `LIMIT_CONN_MAX_HTTP2` for an office with more than 100 open dashboards behind one address. 110
  agent WebSocket upgrades from one address passed with the limit at `100`, and 14 with it at `10`,
  so agents are not counted by it.
- No `COOKIE_FLAGS` change: PocketBase authenticates with an `Authorization` header stored in the
  browser, and the sign-in answer sets no cookie (0 `Set-Cookie` headers).
- No proxy buffering change: the live update reached the browser in the same second with
  `REVERSE_PROXY_BUFFERING=yes`, and `USE_GZIP` and `USE_BROTLI` are off by default, which is what
  upstream asks for on `text/event-stream`.
- BunkerWeb's default CSP (`object-src 'none'; form-action 'self'; frame-ancestors 'self'`) did
  not break the dashboard, and the hub sets none unless you give it `CSP`. The page cannot be put
  in an iframe on another origin.

## CRS exception scope and cost

`configs/modsec-crs/beszel_false_positives.conf` holds nine chained rules: ids `9852101` to `9852109`.
Each one matches an anchored path and a method, then removes one rule id from literal arguments
with `ctl:ruleRemoveTargetById`. BunkerWeb `1.6.17` does not accept regex targets there, so lists
of positions are written out. The three dashboard `filter` rules (`9852101`, `9852107`, `9852108`) add
two more links: the request has exactly one `filter` argument, and its whole value matches the one
shape the dashboard sends. The two logs page rules (`9852103`, `9852109`) add one link, exactly one
`filter` argument: what a superuser types in the logs search box is free text, so no value shape
can be pinned there.

| Id | Method and path | Removed | Why (PocketBase filter or JSON the app sends) |
| --- | --- | --- | --- |
| `9852101` | `GET /api/collections/(system_stats\|container_stats)/records`, argument `filter`, only for `system='<15 characters>' && created > '<YYYY-MM-DD hh:mm:ss>' && type='<1m\|10m\|20m\|120m\|480m>'` | `942100` | Dashboard charts filter with that expression, which libinjection reads as SQL. Upstream issue henrygd/beszel#2684 reports the same. |
| `9852107` | `GET /api/collections/network_monitor_stats/records`, argument `filter`, only for `(monitor='<6 to 10 characters>' \|\| ...) && created><13 digits> && type='<same types>'` | `942100` | The network monitor charts, one monitor or a comparison of several. |
| `9852108` | `GET /api/collections/network_monitors/records`, argument `filter`, only for `system!='<15 characters>' && protocol='<http\|icmp\|tcp\|dns>'` | `942100` | The compare list of the monitor dialog. The plain `system='<id>'` list needs no exception. |
| `9852102` | `POST /api/realtime`, `json.subscriptions.array_0` to `array_15` | `942100` | The browser subscribes to `systems/*` and `alerts/*`; `/*` looks like an SQL comment. A tab with more than 16 topics gets `403`. |
| `9852103` | `GET /api/logs` (exact path, no `normalizePath`), one `filter` argument with any value; also the response body | `942100`, `932235` on `filter`; `953100` on the response body | The superuser logs page list: a typed filter such as `data.status>=400 && data.method="GET"` reads as SQL, and `data.method="POST"`, `"PATCH"` or `"HEAD"` as a Unix command. The answer holds the hub's own error text (`unexpected character`), which `953100` reads as a PHP error, and `959100` then empties a small answer. |
| `9852109` | `GET /api/logs/stats` (exact path, no `normalizePath`), one `filter` argument with any value | `942100`, `932235` on `filter` | The chart above the logs list sends the same filter. Its answer is a date and a count, so nothing is skipped on the response. |
| `9852104` | `PATCH /api/collections/users`, `json.indexes.array_0` to `array_5` | `942190` | Saving the users collection (the documented OAuth2 step) sends its index definitions, which are SQL text. Only `users`. |
| `9852105` | `PATCH /api/collections/user_settings/records/<15 characters>`, `json.settings.webhooks.array_0` to `array_9` | `931100`, `932130`, `934110` | A notification target on a LAN address (`http://192.168.1.10:8080/notify`) reads as remote file inclusion, an IPv6 address as SSRF and shell syntax. |
| `9852106` | `POST /api/beszel/test-notification`, `json.url` | `931100`, `932130`, `934110` | The Test button posts the same kind of URL. |

> [!WARNING]
> **Cost.** The listed rule ids stop looking at the listed argument on that method and path, and
> nothing else changes. `949110` is never excluded and every other rule keeps scoring the argument.

For the three dashboard `filter` rules the exception reaches only the exact shape above. Measured
against the final files (the first 365 requests of `evidence/logs/matrix-final3.tsv` in the lane
folder, 0 failures; the same requests were `evidence/verify/matrix-final.tsv`):

- The 30 filters the dashboard sends pass: every chart range (`1m` to `480m`) for system and
  container stats, one monitor and several monitors, the plain monitor list, the four compare
  protocols, and the container, S.M.A.R.T., systemd and ZFS lists that need no exception.
- 176 attack requests were stopped: 174 got `403`, a `PUT` got `405` from the method list and a
  `%00` path got `400`. They put `name='x' OR 1=1 --` (single, double and no quotes, `#`, `/**/`
  and `/*` comments), `UNION SELECT`, `DROP TABLE`, `sleep(5)` and stacked queries in the `filter`
  of each of the four collections, and XSS, shell, path traversal, PHP and Java payloads in the
  `filter` of `system_stats`. Others appended to or broke out of a dashboard-shaped filter, sat in
  `sort`, `fields`, `expand` or `q`, targeted other collections, used `POST`, `PATCH` or `DELETE`,
  hid behind path tricks (`//`, `%2f`, `;`, case, `..`, `%0a`), came as a second or renamed `filter`
  argument or as a body on `GET`, or followed a newline or carriage return.
- What still passes: any filter in exactly the dashboard shape, whatever the 15-character id and
  the time (`system='zzzzzzzzzzzzzzz' && created > '2000-01-01 00:00:00' && type='1m'` got `200`).
  It carries no SQL a person could use, and PocketBase decides what a signed-in user may list.
  The earlier version of this rule let `name='x' OR 1=1 --` through to the hub (`400` from
  PocketBase's parser); that is now `403`.
- A filter you write by hand outside the shape is inspected and gets `403` when `942100` reads it
  as SQL: double quotes (`system="<id>" && created > "<time>" && type="1m"`), the same without
  spaces, an ISO timestamp with `T` and `Z`, the terms in another order, a monitor filter without
  `created`, a doubled `filter` argument. `system='<id>'` alone, `system!='<id>'` alone and
  `system="<id>"` on the monitors list do not trigger `942100` and need no exception. A script
  that needs a filter outside the shape gets a narrower rule of its own in a custom config: copy
  one of the three rules, give it a new id, and anchor its path, method and that one literal filter.

For the two logs page rules (`9852103`, `9852109`) the value is free text, so the exception reaches
every typed filter on two exact paths and nothing else. Measured with 161 more requests in the same
matrix (526 in all, pass `526`, fail `0`; evidence `evidence/logs/matrix-final3.tsv`, the rows named
`VL-*`, 45 seeds, 80 attacks, 8 gap rows, 1 measure row, 27 probes):

- What the page sends passes: the untyped view, and 13 typed filters (a status and a method,
  the search box's `login` form, a url, a run time, a level, an address, a message, an auth value,
  `data.method="POST"`, `"PATCH"` and `"HEAD"`, a user agent), each as the list (with the
  `created <=` clause) and as the chart (with and without it). All 45 seeds got `200`.
- Of the 80 attacks, 74 got `403`, 2 got `405` (`PUT`), and 4 did not reach a rule:
  `/api/logs/./stats` and `/api/logs/../logs/stats` (curl collapsed them into the exact path, so it
  is the gap below, `400` from the hub), `/api/logs%2fstats` (ModSecurity decodes it into the exact
  path, the hub answered `404`) and a `%00` path (`400` from BunkerWeb). The same dot segments sent
  as written (`curl --path-as-is`) got `403`. The attacks were XSS, `;cat /etc/passwd`, `$(id)`,
  `../../etc/passwd` and `${jndi:...}` in `filter`; the same payloads on `/api/logs/stats/x`,
  `/api/logsx`, `/api/logs//stats`, `/api/logs/stats/`, `/api/logs/`, `/API/logs`, `;x=1`, `%0a`,
  `%2f` and dot-segment variants; `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, a method override and
  a filter in a body; a second, renamed or array `filter`; and `q`, `sort` or `fields` instead of
  `filter`.
- What still passes, on `GET /api/logs` and `GET /api/logs/stats` with one `filter`: `942100` and
  `932235` no longer look at it, so `name='x' OR 1=1 --` reaches the hub (`400` from PocketBase's
  parser). That is what `9852103` already allowed for the list; the chart is new. `UNION SELECT`
  still gets `403` (`942190`, `942270`), and so do the XSS, shell, traversal and Java payloads.
  Only the superuser can call these paths and PocketBase parses the filter itself.
- The paths use no `normalizePath`. With it, `/api/logs//stats` and `/api//logs` also carried the
  exception (`307` from the hub, measured with a variant of the file); without it they get `403`.
- A typed filter that holds an SQL keyword such as `data.url~"select"` still gets `403`
  (`942190`). The words `wget`, `bash`, `python`, `passwd`, `union` and `script` passed.
- The hub's own error text no longer empties a small list answer. A list that holds one entry with
  the hub's `unexpected character '$'` error: before the change 644 bytes straight from the hub and
  `200` with 0 bytes through BunkerWeb (`953100` and `959100` logged); after it 644 bytes, the
  same bytes and no rule line. The same for `unexpected character ';'` (646 bytes). The other two
  hub error texts tried (`invalid quoted text`, `expected && or ||`) never triggered `953100`
  (622 and 626 bytes before and after). A list request without any `filter` is not covered by
  the rule, and its small answer is emptied the same way; the logs page always sends one.

PocketBase's list rules need a signed-in user (an unauthenticated list returned 0 items). Creating
a settings record with a LAN webhook through `POST` is not excluded (control `C05c`, `403`); the
dashboard creates its settings record at the first sign-in (a `POST` that passed) and saves later
changes with `PATCH`.

Old Komodo labels (March 2026) removed `942100`, `934110`, `931100` and `942550` for the whole app.
The first three reproduce and are replaced by the argument-level rules above; `942550` did not
fire on any request of the matrix.

## Validation

Tested on 2026-11-17 and 2026-11-18 with the Beszel hub and agent `0.21.2` (images
`ghcr.io/henrygd/beszel/beszel:0.21.2` and `ghcr.io/henrygd/beszel/beszel-agent:0.21.2`), PocketBase
`0.42.2`, and `bunkerity/bunkerweb-all-in-one:1.6.17` with CRS `4.29.0`, `SECURITY_MODE=block`,
default paranoia level.

<details>
<summary>Test evidence</summary>

- **Request matrix.** The final matrix has 526 requests (`evidence/logs/matrix-final3.tsv`): 176
  seeds (each with a status and a state check), 29 controls, 256 attack requests, 10 hand-written
  filters or other requests whose status is recorded, 46 path and encoding probes whose status is
  recorded, and 9 requests for the gaps above. Without the config file the dashboard's chart,
  monitor chart and compare requests all got `403` from `942100`, and the container, S.M.A.R.T.,
  systemd and ZFS lists passed. With the config on the final files: pass `526`, fail `0`. Every
  control (other collection, argument, method or path suffix, `..%2f` traversal, one position past
  the listed range, plain SQL injection and XSS) is `403` from the specific rule.
- **Browser smoke** (headless Chrome, bad behavior on, only `SERVER_NAME` and the upstream address
  changed, final files): sign-in, the system page in every chart range, the container and monitor
  pages, a monitor dialog compared with another monitor in every range, notification settings
  saved with a LAN webhook and its Test button, 125 seconds idle and then a rename made through
  the API showed up in the open page, the superuser UI signed in and its logs page opened. The
  real agent stayed connected through BunkerWeb. Every filter the dashboard sent on the way (22
  distinct shapes, `evidence/verify/ui-green-filters.txt`) got `200`. The BunkerWeb log slice of 215 requests had
  `200` 178 times, `204` 31 times, `101` once and `499` twice (the page left while a
  `POST /api/realtime` was open). The 22 ModSecurity lines in that slice all belonged to the logs
  page before the rules `9852103` and `9852109` were changed. No ban. The first admin and the Add
  System dialog were exercised in the earlier smoke of 2026-11-18 on the previous rule version;
  the filter rules do not touch those requests.
- **Logs page in a browser** (headless Chrome, bad behavior on, final files, superuser UI):
  signed in, opened `/_/#/logs` and typed five filters in its search box:
  `data.status>=400 && data.method="GET"`, `login`, `data.method="POST"`, `data.method="HEAD"` and
  `data.status=400 && message~"jndi"`. The page sent 12 requests (the untyped list and chart, then the chart and the
  list for each of the five filters), and all 12 got `200`. The list showed the hub's own error
  entries. The browser had no console error, and the BunkerWeb log slice has no ModSecurity line,
  no ban and 10 of those 12 requests (the last two were not in the relay yet). Evidence
  `ui-logs-requests.txt`, `ui-logs-page-typed.png` and `browser-bw-all.log` in
  `evidence/logs/`.
- **Web UI import.** The template with its config file was imported through BunkerWeb's web UI
  (Templates, Raw mode) and is listed with 7 steps, 12 settings and 1 config.

</details>

> [!NOTE]
> Check JSON with `jq . templates/beszel/template.json`. JSON parsing does not prove runtime
> behavior; repeat your own workflows before enabling block mode.

## Things to think about

- **Agents behind one address.** WebSocket agents are not counted by the connection limit and
  their handshake user agent is a Chrome string (`agent/client.go:491`), so user-agent lists do not
  see them. Several misconfigured agents that fail the handshake repeatedly (`400` or `401` every
  10 seconds each) add up in the bad-behavior counter of their shared address.
- **Expose only what agents need.** The dashboard and the agent endpoint share one port
  (upstream issue #2044 asks for a separate one). To publish only `/api/beszel/agent-connect` to
  the outside, give that path its own BunkerWeb service or `REVERSE_PROXY_URL_<n>` location and
  keep the dashboard on a private name. Not tested here.
- **The superuser UI (`/_/`) is the full database admin.** It is open to the internet by default
  here. Restrict it (`REVERSE_PROXY_URL` entries for the paths you want, or an allow list of your
  own) if you do not need it from outside. Backups, collection edits and the logs page live there.
- **The logs page works with a typed filter.** Both of its requests have a rule: `9852103` for the
  list and `9852109` for the chart above it. Before, the chart got `403` from `942100` with any
  typed filter, and the list answer, which holds the hub's own error text, made `953100` and
  `959100` log (the body of 21 597 bytes still arrived, but a small one was emptied, see the
  measurement above). Limits: a typed filter with an SQL keyword such as `select` still gets
  `403`, a list request without a `filter` (a script) can still be emptied, and the rules skip
  `942100` for any value on those two paths, so keep `/_/` and `/api/logs` for superusers only.
- **Uploading a backup** works up to 16 MiB; the hub refuses larger bodies itself. Restore a
  larger backup by copying it into `/beszel_data/backups` on the hub (not tested).
- **Homepage and other API clients.** The Homepage Beszel widget signs in with
  `POST /api/collections/_superusers/auth-with-password` and lists
  `/api/collections/systems/records` with a bearer token; that request shape passed. Eight client
  user agents (`curl`, `Go-http-client`, `python-requests`, `Uptime-Kuma`, an Android Chrome,
  `Homepage`, `Prometheus`, `node-fetch`) were not blocked.
- **No sub-path.** Beszel generates absolute URLs, so `https://host/beszel/` needs a rewrite and
  `APP_URL` with the path (upstream issue #33); the exclusions here match `/api/...` from the root.
- **Not verified:** a real identity provider, forward authentication (Authelia, authentik),
  Cloudflare or other tunnels in front of BunkerWeb, ZFS and systemd detail pages (the test agent
  runs in a container without systemd), real notification delivery, restoring a backup, HTTPS from
  a browser (HTTP/2 was measured only for the stream limit), and several browsers loading the
  dashboard at once.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
