<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/searxng.svg" alt="SearXNG logo" width="96" height="96" />
</p>

<h1 align="center">SearXNG</h1>

<p align="center">Metasearch template with query-aware WAF tuning</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/SearXNG-2026.11.13-248651?labelColor=1d6f93&style=flat" alt="SearXNG 2026.11.13" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#configure-searxng-for-the-proxy">Configure SearXNG</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#crs-exception-scope-and-cost">CRS exceptions</a> ·
  <a href="#validation">Validation</a> ·
  <a href="#things-to-think-about">Things to think about</a>
</p>

Put an existing [SearXNG](https://docs.searxng.org/) instance behind BunkerWeb with ModSecurity and
the OWASP CRS on: searches for SQL, shell, markup and path strings, the autocompleter, the image and
favicon proxies, preferences, the `json`, `csv` and `rss` formats, the address-bar search, and
SearXNG's own limiter and link token.

## At a glance

| Fact        | Value                                             |
| ----------- | ------------------------------------------------- |
| Template id | `searxng`                                         |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, SearXNG 2026.11.13 |
| Configs     | 1 (modsec-crs)                                    |
| Rule ids    | `9871000` to `9871999`                            |
| Upstream    | `http://searxng:8080`                             |

## Prerequisites

- Run SearXNG `2026.11.13` (`searxng/searxng:2026.11.13-8a53a6aa5`) or a close release. BunkerWeb must
  reach `http://searxng:8080` on a private network; do not publish that port.
- Point your public hostname at BunkerWeb. `AUTO_LETS_ENCRYPT=yes` makes BunkerWeb request its
  certificate (the default HTTP challenge needs inbound port 80). For existing certificate files use
  `AUTO_LETS_ENCRYPT=no`, `USE_CUSTOM_SSL=yes`, `CUSTOM_SSL_CERT` and `CUSTOM_SSL_KEY`.
- Serve SearXNG at the root of the hostname: the template proxies `/` and was not tested under a
  sub-path.
- The limiter, `public_instance` and the link token need Valkey (or Redis) for SearXNG
  (`searx/limiter.py:233-249`). A private instance without the limiter needs none.
- If BunkerWeb itself sits behind another proxy or a CDN, set `USE_REAL_IP=yes` and the matching
  `REAL_IP_FROM` first. Otherwise `$remote_addr` is the proxy, SearXNG sees one address for everybody
  and its bot defence misfires (upstream issue
  [#3191](https://github.com/searxng/searxng/issues/3191), not reproduced here).

## Configure SearXNG for the proxy

Set these in SearXNG's `settings.yml` (or the environment variable in brackets) and `limiter.toml`.
The "without it" column is what was measured or read in the source of `2026.11.13`.

| SearXNG setting                                      | Value                                                      | Without it                                                                                                                                                                                                                                                                                                                                       |
| ---------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `server.base_url` (`SEARXNG_BASE_URL`)               | `https://searxng.example.com/`                             | `base_url` takes precedence over `X-Forwarded-Proto` and `X-Forwarded-Host` (`searx/flaskfix.py:40-67`). A wrong value builds wrong absolute links and the redirect after saving preferences (upstream issue [#1852](https://github.com/searxng/searxng/issues/1852)); it also fills the OpenSearch description (`/opensearch.xml`) the browser reads for address-bar search. |
| `server.secret_key` (`SEARXNG_SECRET`)               | a long random value                                        | The image and favicon proxies sign their URLs with it (`/image?url=...&h=<hmac>`). The shipped `ultrasecretkey` lets anyone forge a signed proxy URL.                                                                                                                                                                                              |
| `botdetection.trusted_proxies` (`limiter.toml`)      | BunkerWeb's address only, for example `['10.0.0.5/32']`    | The default is `127.0.0.0/8` and `::1`, so behind BunkerWeb the limiter sees BunkerWeb's address for every client. A range that also covers your clients (the sample comments suggest `192.168.0.0/16`, `172.16.0.0/12`, `10.0.0.0/8`) lets a client on that range choose its own address: see the client-address row in the settings table.        |
| `search.formats`                                     | add `json`, `csv`, `rss` only if clients need them         | The default is `[html]` and `format=json` is answered `403` (`searx/settings.yml`; upstream issues [#3749](https://github.com/searxng/searxng/issues/3749), [#5677](https://github.com/searxng/searxng/issues/5677)). Each `403` counts toward BunkerWeb's bad-behavior ban.                                                                       |
| `server.limiter`, `server.public_instance`           | `true` for an open instance (needs Valkey)                 | Off by default. On, the app rate-limits per address, caps `format!=html` at 4 requests per hour, and (with `public_instance`) turns the link token on. See "Things to think about".                                                                                                                                                              |
| `server.image_proxy`                                 | `true` (the container default)                             | With it off, result thumbnails load straight from the engines' hosts in the visitor's browser.                                                                                                                                                                                                                                                   |

Security: `trusted_proxies` must name only BunkerWeb. With a range that includes clients, a request
with `X-Forwarded-For: 6.6.6.6` was reported by SearXNG's own "self information" answer (`q=ip`) as
`Your IP is: 6.6.6.6`, and 20 searches with 20 different forged values all passed a limiter that
blocks the 16th request from one address (`evidence/xff-wide-nosetting.txt`).

## Install

Import `template.json` and the file under `configs/` using the repository
[installation guide](../../README.md#install), or its plugin bundle layout. Assign
`USE_TEMPLATE=searxng`, then change only `SERVER_NAME` and `REVERSE_PROXY_HOST` (keep the upstream URL
without a trailing slash or path). In the web UI raw import, the config dialog lists the file as
`modsec_crs/searxng_false_positives.conf`: upload the file under that name.

Before enabling blocking on your instance:

1. Set service `SECURITY_MODE=detect` and reload.
2. Search for text from your own users (code, error messages, URLs, paths), open the image
   category, page through results, save preferences, and read the BunkerWeb logs.
3. Confirm the included exclusions match the targets in your logs. Narrow or add exceptions only
   where your traffic requires them.
4. Switch to `SECURITY_MODE=block` after those checks and repeat the workflows.

> [!IMPORTANT]
> False positives return `403` in block mode. The default bad-behavior status list is
> `400 401 403 404 405 429 444`: 10 matching responses in one minute ban the client for one day.

## Settings

Each non-default setting was put back to the BunkerWeb default and the failure below reproduced
(`evidence/r5-*.txt`, `evidence/xff-*.txt`).

| Setting                                                                                        | Purpose and measurement                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `REVERSE_PROXY_URL=/`                                                                          | Proxy the whole site. SearXNG has no WebSocket or event stream (the theme scripts contain no `WebSocket`, `EventSource`, `getUserMedia`, `geolocation` or `document.cookie`), so `REVERSE_PROXY_WS` stays off.                                                                                                                                                                                                                                                           |
| `REVERSE_PROXY_HEADERS=X-Forwarded-For $remote_addr`                                           | Replaces BunkerWeb's default `X-Forwarded-For $proxy_add_x_forwarded_for`, which appends to what the client sent. With a `trusted_proxies` range that includes the client, `X-Forwarded-For: 6.6.6.6` became the client address and 20 searches with rotating forged values all got `200`; with the setting the address stayed `10.86.60.1` and the 16th request got `429`. With only BunkerWeb trusted the forged header was already ignored without the setting, so it is defence in depth for a wide range. |
| `LIMIT_REQ_RATE=100r/s`                                                                        | A page makes its requests in a burst. Cold loads in a fresh browser: image results 37 requests in the first second (28 in 250 ms), search results 22, home 11, preferences 8. At the default `2r/s` an image results page got 27 `429` of 43 responses; with `100r/s` all 37 got `200`. `429` also feeds the bad-behavior ban. Many people behind one address can exceed `100r/s`: see "Things to think about".                                                          |
| `PROXY_BUFFER_SIZE=16k`, `PROXY_BUFFERS=8 16k`, `PROXY_BUSY_BUFFERS_SIZE=32k`                  | A preferences save sets 22 cookies in one response: 2,658 bytes of `Set-Cookie` for the form defaults, 4,383 with every engine disabled, 6,488 with none disabled (260 engine toggles). At the defaults the two larger saves got `502` and the 2,658-byte one passed; with these values all three got `302`. The three settings go together because nginx rejects a busy size that is not smaller than the buffers. The largest `Cookie` header the browser then sent back was 4,242 bytes and got `200`; nginx's default limit is 8 KB (a 9,000-byte header got `400`). |
| `CONTENT_SECURITY_POLICY=object-src 'none'; frame-ancestors 'self';`                           | BunkerWeb's default adds `form-action 'self'`. Chrome applies it to the redirect after the search form is submitted, so an external bang (`!!g hello`) was blocked ("Sending form data to ... violates the following Content Security Policy directive: form-action 'self'"). Without the directive the browser followed the bang to the other site. SearXNG sends no CSP of its own. Drop this setting if you disable external bangs and want the stricter default.        |

Measured and dropped because they made no difference or the default is harmless:

- `REVERSE_PROXY_BUFFERING=no` (the old labels used it). The image proxy, favicon proxy, search and
  `json` seeds, 32 requests, all passed with buffering off and on.
- `REFERRER_POLICY`. BunkerWeb replaces SearXNG's `no-referrer` with `strict-origin-when-cross-origin`
  (measured on the response headers), but every HTML page carries `<meta name="referrer"
  content="no-referrer">`; a search for `' OR 1=1 --` loaded 32 requests and none sent a `Referer`
  with the text in it.
- `COOKIE_FLAGS`. BunkerWeb adds `HttpOnly; SameSite=Lax` to SearXNG's cookies; the theme scripts do
  not read cookies, and saved preferences were still in effect after a reload.
- `PERMISSIONS_POLICY`. The default denies camera, microphone, geolocation and more; the theme uses
  none of them (clipboard is not in the default list).
- `LIMIT_CONN_MAX_HTTP1` and `LIMIT_CONN_MAX_HTTP2`, kept at their defaults: see "Things to think
  about" for the numbers.

## CRS exception scope and cost

`configs/modsec-crs/searxng_false_positives.conf` is a service-scoped **modsec-crs** config. Each
exclusion is one chained rule: the path (anchored, `t:normalizePath`) and the method. The matrix
recorded which rules matched which target on the real app and only those pairs are removed. The
search box takes any text, so the list covers SQL, shell and PHP code, path traversal, template
syntax, HTML and script markup, URLs and error messages.

| Id      | Method and path          | Targets removed                                                              | Rules                    |
| ------- | ------------------------ | ---------------------------------------------------------------------------- | ------------------------ |
| `9871101` | `GET`, `POST` `/search`  | `ARGS:q`; `RESPONSE_BODY` for 6                                              | 38 rule ids, by target |
| `9871103` | `GET` `/search`          | `REQUEST_URI_RAW` and `REQUEST_LINE` for 3 rules (the whole URI)             | `930100 930110 944150` |
| `9871102` | `GET` `/autocompleter`   | the `9871101` set plus the `9871103` set                                     | 38 rule ids, by target |

`ARGS:q` carries the text on `GET` and on `POST` (the "method" preference), and for the `json`, `csv`
and `rss` formats. On `GET` the raw URI and request line carry the same query string, so rule `9871103`
(`GET /search`) and `9871102` also remove `930100`, `930110` and `944150` from those two targets. On
`POST` the browser form sends `q` in the body (the theme's search forms post to `/search` with no
query string), so `9871101` leaves the raw URI and the request line inspected. Three checks stay on
the URI of a `POST /search`: `930100` and `930110` (path traversal such as `../../etc/passwd`) and
`944150` (a `${jndi:` Log4Shell string). Each answers `403` there. Shell text such as `$(id)` in the `q`
of the URL of a `POST /search` is not stopped: the `ARGS:q` exclusion covers it and none of these three
checks matches it (it answered `200` in the tests). API clients that `POST` to `/search` must put the search parameters in the form body, not in
the URL, or a query that looks like path traversal gets `403`; see the
[SearXNG search API](https://docs.searxng.org/dev/search_api.html). The response-side rules
(`951220`, `951230`, `952110`, `953100`, `953120`, `956100`) are removed from `RESPONSE_BODY` because
SearXNG prints the query back: a search for `You have an error in your SQL syntax` or `<?php` makes
those leakage rules score the answer and `959100` deny it. On the small autocompleter answer the
body came back empty (7 requests, `Content-Length: 282`, status `200`); on the 60 KB HTML page the
body arrived complete although the deny was logged. `949110` and `959100` are never excluded.

**Cost:** the query argument is no longer checked by the listed injection rules, on the two paths
and methods only. SearXNG never runs the text: it forwards it URL-encoded to the engines and prints
it HTML-escaped, so a stored or reflected injection through `q` is not the risk that these rules
cover; any other consumer of the query (your own plugin, a log pipeline) has to handle it. The
leakage rules also no longer see the app's own error pages on those two paths, which is the only
content they would have caught there. On `GET`, `REQUEST_URI_RAW` and `REQUEST_LINE` are whole targets:
`930100`, `930110` and `944150` do not read the raw URI or the request line of `GET /search` and
`GET /autocompleter` at all, so a payload in another parameter of that URI is stopped only by their
`ARGS` match, not by these two targets. On `POST /search` the whole URI stays inspected. Every other
argument, header, cookie, path and method stay under the full CRS.

Controls: 28 requests with the same payloads where no exclusion applies all got `403` from the
specific rule the exclusion removes: another argument on `/search` (`GET`, `POST`), another method
(`HEAD`), a path suffix, a path prefix, `;` in the path, a different case, the text in a cookie, the
`Referer` or the `User-Agent`, nine other paths with `q`, five autocompleter variants, a traversal in
the image URL, and an XSS blob in the preferences link.

Not covered, so a payload there is blocked: the `q` argument on `/` (the address bar sends
`/search?q=`, not `/?q=`), the `preferences` blob, and any plugin you add.

## Validation

Tested on 2026-11-18: SearXNG `2026.11.13-8a53a6aa5` (two mock engines behind a stub, the container's
other defaults; the full engine set for the preference-size probes), Valkey 8 for the limiter,
`bunkerity/bunkerweb-all-in-one:1.6.17` (CRS 4.29.0), default paranoia level and anomaly thresholds,
`SECURITY_MODE=block`, plain HTTP on a local test host and one HTTPS and HTTP/2 run with the
self-signed certificate.

<details>
<summary>Evidence behind the claims above</summary>

- **Request matrix** (final run on the shipped files, `evidence/matrix-final2.tsv`): 270 requests
  that must succeed and 28 controls, one at a time with their ModSecurity lines attributed per
  request. The same 120 seeds of the first run all pass straight to SearXNG without the WAF; the
  first two runs without the config got `403` for 49 of 145 requests, every one on the query text
  of `/search` or `/autocompleter`. With the config all 270 pass and read back (the stub engine
  received the query unmodified, the page rendered the mock results, the saved preferences came
  back), and all 28 controls got `403` from the specific rule. The other seeds need no exclusion:
  the instance pages, the image and favicon proxies, a preferences save and link, cookies, an
  external bang redirect, the `html`, `json`, `csv` and `rss` formats, and 14 client user agents
  (curl, `python-requests`, `httpx`, `aiohttp`, `axios`, `node-fetch`, Go, OkHttp, Perplexica, a
  SearXNG client, LiteLLM, Homepage, Wget, Java; run with the limiter off).
- **Settings:** each non-default setting was switched back to the BunkerWeb default and the failure
  above reproduced.
- **Browser smoke** (Chrome through Playwright, bad behavior on, template as shipped, only
  `SERVER_NAME` and `REVERSE_PROXY_HOST` set). With the limiter off: searches for `' OR 1=1 --`,
  `../../etc/passwd`, `<script>alert(1)</script>`, `$(id)` and a 60-character Japanese sentence each
  rendered 8 results with the text kept in the box, the image category loaded, page 2 loaded, the
  preferences form saved and the autocompleter answered while typing; 219 responses, all `200`.
  With the limiter and link token on: the same flow, 232 responses, all `200`. The BunkerWeb log
  for each run shows no ModSecurity denial, no ban and no `403` or `444`
  (`evidence/smoke-limiter-off.summary`, `evidence/smoke-limiter-on.summary`).
- **Web UI import:** raw mode with `template.json` plus the config file imports and is listed
  (7 steps, 11 settings, 1 config).

</details>

Not verified: real search engines, Open WebUI, Perplexica or any client beyond their user agent
strings, Let's Encrypt, BunkerWeb behind another proxy or a CDN, a sub-path, many users on real
networks, Redis instead of Valkey, SearXNG versions other than `2026.11.13`. Validate your own
clients.

## Things to think about

- **Limiter, link token and API clients.** With `server.limiter` on, SearXNG allows 15 requests
  per 20 seconds and 150 per 10 minutes per address (`searx/botdetection/ip_limit.py`). Through
  BunkerWeb, with the client address trusted correctly: 20 searches in a row gave 15 `200` and five
  `429`. `format=json`, `csv` and `rss` are capped at 4 per hour (measured: 4 `200`, then `429`),
  and the limiter rejects requests whose `Accept` lacks `text/html` (`http_accept`) or whose user
  agent looks like a script (`http_user_agent`): a client sending `Accept: application/json` got
  `429` on the first request, and so did `python-requests`. Put the client's address in
  `[botdetection.ip_lists] pass_ip` (all six requests of the same client then got `200`), or leave
  the limiter off for a private instance. `pass_ip` only holds if the address SearXNG sees cannot be
  forged: keep `trusted_proxies` narrow.
- **Link token.** With `link_token = true` a visitor who has not loaded a page yet counts as
  suspicious (2 requests per 20 seconds): a browser that opened 18 searches in a row by URL got
  `429`, `429`, `429`, then `200` for the rest, because the first page's `client<token>.css` request
  had marked it. Visitors who start at the home page are not affected. SearXNG's own `429` page is
  replaced by BunkerWeb's (69,441 bytes instead of 17); the status is the same.
- **BunkerWeb antibot.** Not part of the template. With `USE_ANTIBOT=cookie` every client without
  the cookie is redirected to `/challenge`: `GET /search`, `format=json`, `POST /search`,
  `/autocompleter` and `/healthz` all got `302`. A browser passes once and then searches normally
  (autocompleter and `json` fetches got `200` with the cookie), but scripts, API clients and uptime
  checks break. BunkerWeb can exempt paths (`ANTIBOT_IGNORE_URI`); that was not tested here, and
  the limiter and link token already cover most of what a challenge would add.
- **Many people behind one address.** Each results page is a burst of requests: with the template
  values, 2 and 3 browsers loading the image results at once over HTTPS and HTTP/2 all got `200`,
  and 5 at once got `429` for three of them (`limiting connections by zone "v2ips"`, 18 events,
  `LIMIT_CONN_MAX_HTTP2=100`). On plain HTTP and HTTP/1.1 the default `LIMIT_CONN_MAX_HTTP1=10` is
  the limit: 3 browsers at once got 15, 19 and 21 `429`. Raise `LIMIT_REQ_RATE` and the matching
  `LIMIT_CONN_*` for offices and schools, and serve HTTPS.
- **External bangs.** `!!g text` redirects the browser to the other site. The template lets that
  through (see the CSP row). The redirect leaves SearXNG with a `302`, so BunkerWeb and the CRS
  never see the other site; the text you typed is sent to it.
- **Address-bar search.** Browsers read `/opensearch.xml` and send `GET /search?q=<text>` and
  `GET /autocompleter?q=<text>`; both are covered by the exclusions. Set `server.base_url` so the
  XML carries the public address.
- **Instance pages.** `/stats`, `/stats/errors` and `/config` are public and list your engines and
  their error rates; `/metrics` answers `404` until `general.open_metrics` is set, then asks for
  that password. The template leaves them open (`/stats` and `/config` answered `200` in the
  matrix); block them at BunkerWeb if you do not want that exposed.
- **Preference link.** `/preferences?preferences=<blob>&save=1` applies a preference set in one
  request. It passed the CRS in the matrix and is how people share settings.
- **Search engines and the offline harness.** The matrix used two mock engines. A real engine
  answers slower, which lowers the number of concurrent requests per user, and engines have their
  own captcha and rate limits that BunkerWeb does not see.
- **Not supported through the template:** a sub-path deployment (`X-Script-Name`).

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
