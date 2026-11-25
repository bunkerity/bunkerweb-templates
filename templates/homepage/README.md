<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/png/homepage.png" alt="Homepage logo" width="93" height="96" />
</p>

<h1 align="center">Homepage</h1>

<p align="center">Dashboard template with room for every tile, widget-safe ban rules and an optional authentik sign-in</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Homepage-2.8.0-248651?labelColor=1d6f93&style=flat" alt="Homepage 2.8.0" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#configure-homepage-for-the-proxy">Configure Homepage</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#sign-in-with-authentik-optional">authentik</a> ·
  <a href="#crs-exception-scope-and-cost">CRS exceptions</a> ·
  <a href="#validation">Validation</a> ·
  <a href="#known-limits">Known limits</a>
</p>

Put an existing [Homepage](https://gethomepage.dev) dashboard behind BunkerWeb with ModSecurity and
the OWASP CRS on: service tiles, status dots, service widgets, information widgets, the search box,
live config reload, the optional built-in sign-in and the optional MCP endpoint. The template lets a
big dashboard open without `429` answers, keeps a failing widget from getting you banned, shows the
real widget error text, limits password tries, and removes the few CRS rules that block normal use.

> [!WARNING]
> Homepage has no sign-in unless you turn on its built-in one: the default install is open to
> anyone who reaches it. The dashboard lists your internal service names and links, and its widgets
> reach every service in `services.yaml` from inside the container. Do not publish it without the
> built-in sign-in, the [authentik step](#sign-in-with-authentik-optional) or a VPN.

## At a glance

| Fact        | Value                                                 |
| ----------- | ----------------------------------------------------- |
| Template id | `homepage`                                            |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Homepage 2.8.0         |
| Configs     | 1 (modsec-crs)                                        |
| Rule ids    | `9881000` to `9881999`                                |
| Upstream    | `http://homepage:3000`                                |

## Prerequisites

- Run Homepage `2.8.0` or a close release (tested image
  `ghcr.io/gethomepage/homepage:v2.8.0`). BunkerWeb must reach it at `http://homepage:3000` on a
  private network. Do not publish port 3000.
- Point your public hostname at BunkerWeb. `AUTO_LETS_ENCRYPT=yes` makes BunkerWeb request its
  certificate, and the default HTTP challenge needs inbound port 80. For your own certificate files
  switch the service to Advanced mode (the guided steps do not show them) and set
  `AUTO_LETS_ENCRYPT=no`, `USE_CUSTOM_SSL=yes`, `CUSTOM_SSL_CERT` and `CUSTOM_SSL_KEY`.
- Serve Homepage at the root of the hostname. Version 2.8.0 has no base path setting, and the
  template proxies `/`.

## Configure Homepage for the proxy

Set these on the Homepage container. The first row was measured on `2.8.0`; the others come from
reading its source and documentation at that tag.

| Homepage setting                                       | Value                                    | Without it                                                                                                                                                                                                                     |
| ------------------------------------------------------ | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `HOMEPAGE_ALLOWED_HOSTS`                               | `homepage.example.com`                   | Every request answers `400` with `Host validation failed`, and the Homepage log names the host it saw. BunkerWeb passes the host name without a port, so list the bare name even when BunkerWeb listens on another port (an entry with the port gave `400`). |
| `HOMEPAGE_AUTH_ENABLED`, `HOMEPAGE_AUTH_SECRET`, `HOMEPAGE_EXTERNAL_URL`, `HOMEPAGE_AUTH_PASSWORD` | `true`, 32 or more random characters, `https://homepage.example.com`, a long password | The optional built-in sign-in. Without it, anyone who reaches the dashboard sees and uses everything. `HOMEPAGE_EXTERNAL_URL` must be the public address with `https://`, or the session cookie lacks `Secure` and callbacks use the wrong origin. |
| `HOMEPAGE_MCP_ENABLED`                                 | leave unset                              | The MCP endpoint (`/api/mcp`) stays off. With it on, a token of 32 or more characters (`HOMEPAGE_MCP_TOKEN`) reads every service credential, and `HOMEPAGE_MCP_ALLOW_WRITE=true` also lets it replace `custom.js`, which runs in every browser. |
| `HOMEPAGE_VAR_<NAME>` or `HOMEPAGE_FILE_<NAME>`        | your widget keys                         | Keys written straight into `services.yaml` are readable through MCP. Reference them as `{{HOMEPAGE_VAR_NAME}}`.                                                                                        |
| `AUTH_TRUST_HOST`                                      | do not set                               | With `HOMEPAGE_EXTERNAL_URL` unset it makes the sign-in library trust a forwarded host name.                                                                                                                                  |

Security: Homepage never reads a client address. It ignores `X-Forwarded-For`, `X-Real-IP` and
`Forwarded` (no use in the source), so its log shows no address for a failed sign-in and fail2ban or
CrowdSec cannot ban from it. Limits and bans live in BunkerWeb.

## Install

1. Import the template. In the web UI use Templates, Create template, Raw, and upload
   `template.json`; the dialog then lists one missing file, `modsec_crs/homepage_false_positives.conf`,
   so upload `configs/modsec-crs/homepage_false_positives.conf` there.
2. Create the service: Services, Create new service, Templates, `Homepage`, Easy mode.
3. Step 1: enter your public hostname in `SERVER_NAME`. Keep `AUTO_LETS_ENCRYPT` on for a public
   site and turn it off for a name that is not public (a lab name such as `homepage.test`), since no
   certificate can be issued for it. List the same name in `HOMEPAGE_ALLOWED_HOSTS`.
4. Step 2: change `REVERSE_PROXY_HOST` only if your container is not named `homepage`. Keep the
   address without a path or trailing slash.
5. Steps 3 to 6: the defaults are tuned (see [Settings](#settings)). Raise the step 3 values if your
   dashboard has well over 60 tiles.
6. Step 7 (authentik sign-in): leave it empty to skip it.
7. Step 8 only shows the CRS exclusions. Save, wait for the reload, open `https://<your hostname>`.

> [!NOTE]
> Easy mode does not label the numbered fields in steps 5 and 7, it shows the setting name. The
> step subtitles name them. Entering a value equal to a default that the template changes does not
> stick: to go back to BunkerWeb's own default, type that default instead of clearing the field
> (`BAD_BEHAVIOR_STATUS_CODES` is `400 401 403 404 405 429 444`).

Before you turn on blocking on a live instance, set the service's Security mode to `detect` (Advanced
mode, General tab; a new service starts in `block`), use the tiles and widgets you rely on, read the
BunkerWeb log, then switch it to `block`.

> [!IMPORTANT]
> False positives answer `403` in block mode. The template's ban list does not count `403`, but a
> blocked request still fails.

For both import methods (web UI and plugin), see the repository [installation guide](../../README.md#install).

## Settings

Each setting was put back to the BunkerWeb default and the failure below reproduced on the real
app, in headless Chrome behind one address. Tiles are the services of a generated `services.yaml`,
each with a widget and a status check.

| Setting                                                         | Purpose and measurement |
| --------------------------------------------------------------- | ----------------------- |
| `REVERSE_PROXY_URL=/`, `REVERSE_PROXY_HOST=http://homepage:3000` | Proxy the app. Homepage `2.8.0` polls with plain requests: a 180 second dashboard session (809 requests) opened no WebSocket and no event stream, so there is no `REVERSE_PROXY_WS`. |
| `LIMIT_REQ_RATE=500r/s`                                         | BunkerWeb counts requests per address and per route, and a dashboard asks for every tile on the same route at once when it opens. At the default `2r/s` a 60 tile dashboard got 433 `429` answers out of 530 requests, and `429` counts toward a ban. `50r/s` gave 116 out of 471, `100r/s` gave none for one browser (411 requests). Over HTTPS with `200r/s`, three browsers opening 60 tiles together got 37 `429` out of 1776 requests. At `500r/s` three browsers (1782 requests) and five (2975 requests) got none; the busiest second held 213 and 197 requests. Raise it if you run more tiles or more browsers than that. |
| `LIMIT_CONN_MAX_HTTP2=500`                                      | Over HTTPS the browser sends all tile requests at once on one connection. At the default `100`, 61 requests got `429` and the address was banned (every later request answered `403`). `200` passed for one browser (592 requests) and gave 46 `429` for three. With `500` and `500r/s` together, three and five browsers of 60 tiles passed. |
| `LIMIT_REQ_URL_1=^/api/auth/callback/credentials$`, `LIMIT_REQ_RATE_1=10r/m` | Homepage does not limit password tries itself. Without the slot, 25 wrong passwords in a second all got `401`. With it, 10 got `401` and 15 got `429`, and the sign-in page itself stayed reachable. Harmless while the built-in sign-in is off. |
| `BAD_BEHAVIOR_STATUS_CODES=405 429 444`                         | A widget that fails (wrong key, stopped service) and a status dot of a service that is down answer `400`, `401`, `403` or `404` on every refresh. With the default list (`400 401 403 404 405 429 444`) a dashboard with failing widgets banned the address during the run. With the template list the same run was not banned. The cost: repeated `400`, `401`, `403` or `404` no longer ban on their own, and the password slot above guards sign-in instead. |
| `REVERSE_PROXY_INTERCEPT_ERRORS=no`                             | A failing widget tile shows its own error text. With the default `yes`, BunkerWeb swaps the widget's `4xx` or `5xx` answer for its HTML page and the tile reads `Unexpected token '<', "<!doctype " ... is not valid JSON`. The host name check error also stays readable. |

Measured and left at the default: `LIMIT_CONN_MAX_HTTP1` (three browsers of 60 tiles over plain
HTTP, and two browsers with hung widgets, got no `429`), `COOKIE_FLAGS` (the sign-in cookies
already carry `HttpOnly` and `SameSite=Lax`, one copy each), the security headers (one copy each,
no duplicate), `Cache-Control: private, no-store` on the signed-in dashboard (passes through
unchanged) and `X-Forwarded-For` (Homepage ignores it).

> [!TIP]
> With search suggestions on (`showSearchSuggestions: true` on the search widget, or under
> `quicklaunch` in `settings.yaml`), Homepage asks the search provider on every typed character. At 10 characters a second the
> template's rate is no limit. Without that option Homepage never calls the suggestion route, and
> the search rules below are idle. The default `Permissions-Policy` of BunkerWeb denies camera,
> microphone and fullscreen: Homepage uses none, but a `custom.js` that does needs a change in the
> `PERMISSIONS_POLICY` setting.

## Sign in with authentik (optional)

Step 7 puts an authentik forward-auth sign-in in front of Homepage, so only authentik users reach
the dashboard. Skip it if you use Homepage's own sign-in or a VPN. Both can run together: authentik
first, then the Homepage password page.

In authentik, create an application with a **Proxy Provider** in "Forward auth (single
application)" mode, set its external host to the public address including the port if it is not
443 (for example `https://homepage.example.com`) and assign it to a standalone proxy outpost, not the embedded one (sign-out answered `502` through
the embedded outpost of authentik 2025.10.1, an upstream fault). BunkerWeb must reach the outpost
(the tests used `http://outpost:9000`). Then fill these fields in step 7:

| Field (setting)                         | Value                                                                                   |
| --------------------------------------- | --------------------------------------------------------------------------------------- |
| `REVERSE_PROXY_HOST_1`                  | `http://outpost:9000` (the outpost address)                                             |
| `REVERSE_PROXY_URL_1`                   | keep `/outpost.goauthentik.io`                                                          |
| `REVERSE_PROXY_HEADERS_1`               | keep the value: it passes the public host and the original URL to the outpost           |
| `REVERSE_PROXY_PASS_REQUEST_BODY_1`     | keep `no`                                                                               |
| `REVERSE_PROXY_AUTH_REQUEST`            | `/outpost.goauthentik.io/auth/nginx`                                                    |
| `REVERSE_PROXY_AUTH_REQUEST_SIGNIN_URL` | `$scheme://$http_host/outpost.goauthentik.io/start?rd=$scheme://$http_host$request_uri` |

Homepage reads no `X-authentik-*` header, so the dashboard does not show who signed in.

What the tests showed, in a browser over HTTPS with the three fields filled in easy mode: an
unauthenticated visit went to the authentik sign-in, came back to the dashboard (169 requests
answered `200`, 21 tiles), and the outpost cookie was `HttpOnly`, `SameSite=Lax` and `Secure`. With
the fields empty the dashboard opened directly.

- **An open dashboard whose session ends keeps polling.** Each refresh is redirected to authentik
  and fails in the browser (102 failed requests in 20 seconds), so tiles show errors until you reload
  the page. Reloading signs in again at once while the authentik session lives.
- **Sign out** (`/outpost.goauthentik.io/sign_out`) lands on the authentik sign-out page. The authentik
  session stays, so the next visit signs in again without a prompt.
- **Every path is covered, `/api/mcp` and `/api/healthcheck` included.** An MCP client or an uptime
  check that calls the public address gets a `302` to the sign-in. Point them at Homepage's private
  address (`POST /api/mcp` with the token answered `200` there).

## CRS exception scope and cost

`configs/modsec-crs/homepage_false_positives.conf` is a service-scoped **modsec-crs** config with five
rules. Each one names an exact path and the method `GET`.

| Rule      | Path                          | What it removes |
| --------- | ----------------------------- | --------------- |
| `9881100` | `/api/search/searchSuggestion` | CRS `930100`, `930110`, `930120`, `932120`, `932125`, `932130`, `932160`, `932230`, `932235`, `932250`, `932260`, `932340`, `932370`, `932380`, `933120`, `933150`, `933160`, `941100`, `941110`, `941160`, `941390`, `942100`, `942190`, `942320`, `942550` and `944130` for the `query` argument (what you type in the search box, one request per character, only with search suggestions on). `930100` and `930110` also skip the raw query string of this route. |
| `9881110` | `/api/search/searchSuggestion` | The leaked-error-page checks for the answer: CRS `950150`, `951110` to `951260` (the SQL error rules of every database), `952110`, `953100`, `954120`, `954130` and `956100`. Suggestions repeat the typed text first, so a search for a PHP, SQL, Java, ASP.NET, IIS or Ruby error message looks like a leaked error page, and the CRS cut the connection. The check for leaked PHP source code (`953110`) stays on. |
| `9881120` | `/api/ping`                    | CRS `932250` for `groupName` and `serviceName`, and `932260` for `serviceName`. A tile called `docker-registry`, `Wget Mirror` or `ssh bastion` reads as a command. |
| `9881130` | `/api/siteMonitor`             | The same two rules and arguments, for the status check. |
| `9881140` | `/api/services/proxy`          | `932250` for `group` and `service`, and `932260` for `service`, for widget requests. |

Cost: the same text in another argument, path or method is still blocked (each rule has controls
that return `403`). The checks named above skip the search text on its route and the tile names on
theirs, but the app only uses both as keys: the search text goes URL-encoded to your provider, and an
unknown name is refused. Every other check still reads the search text, so a PHP stream wrapper, a
`union select` query or a serialized object typed into the search box still answers `403`. The
suggestion answer is not scanned for leaked error text, and it holds only the typed text and the
suggestions of your search provider. The check for leaked PHP source code (`953110`) still reads it.

To remove one exception, delete its rule block (the comment lines and the rule below them) in the
web UI: Configs, `homepage_false_positives` of your service (type `modsec-crs`), then Save. Other
words in a tile name or in a search can need another rule: add the CRS ids the BunkerWeb log names to
the matching block, or write a new block in a config of your own: Configs, Create Custom Config,
Service = your service, Config Type `MODSEC_CRS`.

## Files

- `template.json` - BunkerWeb template definition: settings, configs and the guided steps.
- `configs/modsec-crs/homepage_false_positives.conf` - the five CRS exclusions above.

## Validation

Run `jq . template.json` to check the JSON before importing.

Everything ran against the real app (`ghcr.io/gethomepage/homepage:v2.8.0`) behind BunkerWeb
`1.6.17` with CRS `4.29`, one request at a time, with the log read back after each run.

| Set                                 | Requests | Result                                                                                           |
| ----------------------------------- | -------- | ------------------------------------------------------------------------------------------------ |
| Dashboard, widgets, search, names   | 115      | All as expected: 104 pass, 11 give the app's own `403`, `404` or `500`. 19 were blocked without the config. |
| Controls (other argument, path, method) | 13   | All still blocked or cut: `403` from the same CRS rules, or the connection closed for the answer rule. |
| Search box, error text typed one character at a time (47 texts: PHP, MySQL, PostgreSQL, SQL Server, Oracle, Java, Python, ASP.NET, IIS, Ruby, Node) | 2846 | 786 were cut or blocked without the search rules above, none with them: every answer is `200` with the typed text first. |
| Search box, shell commands typed one character at a time (27 texts) | 919 | 142 blocked without the rules, none with them. |
| Search box, every entry of the CRS error text lists (SQL, Ruby, IIS, ASP.NET, 250 of the PHP list) | 639 | 492 cut or blocked without the rules, none with them. |
| Search box with the real DuckDuckGo suggestions | 47 | 22 cut or blocked without the rules, none with them. |
| Search box in Chrome, five error and command texts typed | 167 | 82 requests failed or answered `403` without the rules, none with them. |
| Controls of the search rules (other argument, path or method, and attack payloads in `query`) | 19 | All still blocked (`403`) or cut. |
| Line feed before the path           | 4 + 4    | The exclusion does not match, so all 4 stay blocked, by `921190` and by the rule the exclusion would lift. With `921190` taken out for the test, only the lifted rule blocks them. |
| Built-in sign-in                    | 16       | All as expected: page, CSRF, session cookie, redirect, sign out, wrong password. |
| Sign-in with symbol-heavy password  | 2        | Blocked with `403`, as listed under known limits.                                                |
| Passwords                           | 51       | 46 reach Homepage, 5 blocked with `403`.                                                         |
| MCP endpoint                        | 18       | 16 as expected, 1 `403` for JavaScript written into `custom.js`, 1 `400` for a body over 128 KiB. |

Browser runs, with Chrome driving the dashboard and the web UI, back the settings and sign-in
sections above.

Test the dashboard again after a Homepage upgrade, since the exclusions name routes and arguments of
`2.8.0`.

## Known limits

- **Code-like text in a few places is blocked.** Of 51 test passwords for the built-in sign-in, 46
  reached Homepage (a wrong one got its own `401`), and five that read as shell substitution, SQL or
  a path answered `403`. Two sign-in tries with a password full of symbols were blocked the same
  way. Pick a password without such patterns, for example random words.
- **JavaScript through MCP.** A `write_config_file` MCP call that writes JavaScript into `custom.js`
  answers `403` (CRS `933160`, `934100`, `941180`, `941390` and `942151`). No exclusion ships for it,
  because that file runs in every browser.
- **Large MCP writes.** A body over 128 KiB (BunkerWeb's `MODSECURITY_REQ_BODY_NO_FILES_LIMIT`)
  answers `400`. A 64 KiB write passed.
- **Sub-paths.** Homepage `2.8.0` cannot serve under a path, so use a host name of its own.
- **Embedding.** BunkerWeb's default `X-Frame-Options: SAMEORIGIN` and `frame-ancestors 'self'`
  keep Homepage out of another site's iframe.
- **Easy mode.** Switching `AUTO_LETS_ENCRYPT` off while you create the service is not kept. Check
  the setting after the first save if you bring your own certificate.
- **Not verified:** the built-in OIDC sign-in, the Docker, Kubernetes and Proxmox integrations
  (their routes answer `500` on a test setup without a cluster), weather and
  stock widgets that call the internet from the server, installing the dashboard as a web app,
  HTTP/3, a real Let's Encrypt certificate, and more than five browsers behind one address.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
