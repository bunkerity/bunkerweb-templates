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
- Point your public hostname at BunkerWeb. Step 1 of the guided setup has Let's Encrypt on: BunkerWeb
  then requests its certificate (the default HTTP challenge needs inbound port 80). Turn it off for a
  name that is not public (a lab, `.localhost`), or clients are redirected to HTTPS with no usable
  certificate. If the redirect stays after you save, edit the service and save again: BunkerWeb 1.6.17
  can lose the switch at creation. For your own certificate files use the service's Advanced tab
  (these settings are not in the guided steps): switch Let's Encrypt off, then `USE_CUSTOM_SSL`,
  `CUSTOM_SSL_CERT` and `CUSTOM_SSL_KEY`.
- Serve SearXNG at the root of the hostname: the template proxies `/` and was not tested under a
  sub-path.
- SearXNG `2026.11.13` loads no wikipedia, wikidata or openstreetmap engine until it has a contact
  address: set `general.contact_url` (`SEARXNG_CONTACT_URL`). SearXNG logs `wikipedia engine requires
  general.contact_url`; this is not the WAF.
- The limiter, `public_instance` and the link token need Valkey (or Redis) for SearXNG
  (`searx/limiter.py:233-249`): set `SEARXNG_VALKEY_URL` (for example `valkey://valkey:6379/0`) next
  to `SEARXNG_LIMITER=true`. With the limiter on and no Valkey address SearXNG still starts and logs
  `The limiter requires Valkey`. A private instance without the limiter needs none.
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
| `botdetection.trusted_proxies` (`limiter.toml`)      | BunkerWeb's address only: in `limiter.toml` write `[botdetection]` then `trusted_proxies = ['10.0.0.5/32']` | The default is `127.0.0.0/8` and `::1`, so behind BunkerWeb the limiter sees BunkerWeb's address for every client. A range that also covers your clients (the sample comments suggest `192.168.0.0/16`, `172.16.0.0/12`, `10.0.0.0/8`) lets a client on that range choose its own address: see the client-address row in the settings table.        |
| `search.formats`                                     | add `json`, `csv`, `rss` only if clients need them         | The default is `[html]` and `format=json` is answered `403` (`searx/settings.yml`; upstream issues [#3749](https://github.com/searxng/searxng/issues/3749), [#5677](https://github.com/searxng/searxng/issues/5677)). Each `403` counts toward BunkerWeb's bad-behavior ban.                                                                       |
| `server.limiter`, `server.public_instance`           | `true` for an open instance; also set `valkey.url` (`SEARXNG_VALKEY_URL=valkey://valkey:6379/0`) | Off by default. On, the app rate-limits per address, caps `format!=html` at 4 requests per hour, and (with `public_instance`) turns the link token on. See "Things to think about".                                                                                                                                                              |
| `server.image_proxy`                                 | `true` (the container default)                             | With it off, result thumbnails load straight from the engines' hosts in the visitor's browser.                                                                                                                                                                                                                                                   |

Security: `trusted_proxies` must name only BunkerWeb. With a range that includes clients, a request
with `X-Forwarded-For: 6.6.6.6` was reported by SearXNG's own "self information" answer (`q=ip`) as
`Your IP is: 6.6.6.6`, and 20 searches with 20 different forged values all passed a limiter that
blocks the 16th request from one address (`evidence/xff-wide-nosetting.txt`).

## Install

Import `template.json` and the file under `configs/` using the repository
[installation guide](../../README.md#install), or its plugin bundle layout. In the web UI, create
the service in easy mode and pick the SearXNG template, then change only the server name and the
upstream address (keep the upstream URL without a trailing slash or path). In the web UI raw import,
the config dialog lists the file as `modsec_crs/searxng_false_positives.conf`: upload the file under
that name. Saving the service applies it: there is no separate reload.

Before enabling blocking on your instance:

1. Set the service's Security mode to `detect` (Advanced or Raw mode, General: it is not in the
   guided steps) and save.
2. Search for text from your own users (code, error messages, URLs, paths), open the image
   category, page through results, save preferences, and read the BunkerWeb logs.
3. Confirm the included exclusions match the targets in your logs. Narrow or add exceptions only
   where your traffic requires them.
4. Switch to `block` after those checks and repeat the workflows.

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
exclusion is one chained rule: the path (anchored, `t:normalizePath`) and the method. A list of 799
distinct realistic searches (shell commands, code in a dozen languages, SQL questions, markup,
template syntax, URLs, file paths and error messages from databases, PHP, Java, .NET, Node and
Python) was sent to `/search` and `/autocompleter`; every CRS rule that fired on `q` was removed
from `ARGS:q` only. The results of the runs on the shipped file are under "Validation".

| Id        | Method and path         | Targets removed                                                                                                           |
| --------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `9871101` | `GET`, `POST` `/search` | `ARGS:q` for 71 rule ids; the CRS rules tagged `attack-disclosure` on the answer                                          |
| `9871103` | `GET` `/search`         | the whole URI for four rules: `920540` (`REQUEST_URI`), `930100`, `930110` (`REQUEST_URI_RAW`), `944150` (`REQUEST_LINE`) |
| `9871102` | `GET` `/autocompleter`  | the same URI checks as `9871103`, `ARGS:q` for the same 71 rule ids, the answer rules, and `ARGS_NAMES` for 59 of the 71  |

`ARGS:q` carries the text on `GET` and on `POST` (the "method" preference), and for the `json`, `csv`
and `rss` formats. On `GET` the raw URI and request line carry the same query string, so rule `9871103`
(`GET /search`) and `9871102` also remove `920540`, `930100`, `930110` and `944150` from those targets.
On `POST` the browser form sends `q` in the body, so `9871101` leaves the URI inspected: four checks
stay on the URI of a `POST /search` (`920540`, an escaped Unicode character; `930100` and `930110`,
path traversal such as `../../etc/passwd`; `944150`, a `${jndi:` Log4Shell string) and each answers
`403` there. API clients that `POST` to `/search` must put the search parameters in the form body,
not in the URL; see the [SearXNG search API](https://docs.searxng.org/dev/search_api.html).

The answer side: SearXNG prints the query back, and the results carry text from other sites, so a
search for a database or PHP error message (`ORA-00942`, `fopen(` with a file name) makes the CRS
leakage rules score the page and `959100` deny it. On the small autocompleter answer the body came
back empty with status `200`; on a results page the connection was cut. The template removes every
rule tagged `attack-disclosure` for these two paths, not a list of ids, because the page holds
whatever the engines return. `949110` and `959100` are never excluded. To keep those checks on the
results page, delete the `ctl:ruleRemoveByTag=attack-disclosure` line from the two rules.

The autocompleter script puts the typed text in the URL without escaping an ampersand, so the text
after `&` reaches BunkerWeb as argument names. Rule `9871102` therefore removes `ARGS_NAMES` for 59
rule ids. These are the ids that fired on a name when `ARGS_NAMES` was left out of the rule: 4 of
them (`932160`, `932250`, `932260`, `942100`) on the 23 typed texts that hold an ampersand (310
requests, every prefix as typed), and 59 of the 71 in all when each of the 799 searches was put
after `x&` (`x&<text>`, 799 requests). The other 12 ids never fired on a name; four of them
(`930100`, `930110`, `931100`, `941370`) do not read argument names at all. So a text in an
argument name that trips one of those 12 gets `403` on `/autocompleter`, as before this rule
existed. The names of every argument of `/autocompleter` skip the 59 rules, not only the names a
typed ampersand creates; that is the cost of this rule. The value of
an argument other than `q` is not excluded: a typed text such as `(&(uid=*)(userPassword=*))` still
gets `403` on the suggestions request (the search itself works), because rule `921200` reads the
value of the argument that the ampersand creates. So do markup or code typed after an ampersand with
an `=` in it (`x&<a href="...">`), and a line feed (`921120`, `921150`, `921160`). A `403` on
suggestions counts toward the bad-behavior ban.

**Text that still gets `403`:** a real NUL byte in the URL (`%00` in the address, rule `920270` on the
URI; typing the characters `%00` in the box works), the same text outside `q`, and any text that trips
a CRS rule not in the list. If one does, copy a line in each rule, change the id and keep the order
(in the web UI: Configs, open `searxng_false_positives.conf`):

```
    ctl:ruleRemoveTargetById=<id>;ARGS:q,\
```

**Cost:** the query argument is no longer checked by the listed injection, code and protocol rules,
on the two paths and methods only. SearXNG never runs the text: it forwards it URL-encoded to the
engines and prints it HTML-escaped, so a stored or reflected injection through `q` is not the risk
that these rules cover; any other consumer of the query (your own plugin, a log pipeline) has to
handle it. The leakage rules also no longer see the app's own error pages on those two paths. On
`GET`, the four rules `920540`, `930100`, `930110` and `944150` do not read the raw URI or the request
line of `GET /search` and `GET /autocompleter` at all, so a payload in another parameter of that URI
is stopped only by their `ARGS` match. On `GET /autocompleter` the names of all arguments also skip
59 rules (see above).

To roll back, each rule on its own:

- Delete rule `9871103`: the four URI checks come back on `GET /search` only, and path-like
  searches there get `403`.
- Delete the four `ctl:ruleRemoveTargetById=...;REQUEST_URI` / `REQUEST_URI_RAW` / `REQUEST_LINE`
  lines in rule `9871102`: the URI checks come back on `GET /autocompleter`. Deleting `9871103` does
  not do that.
- Delete the 59 lines that end in `;ARGS_NAMES` in rule `9871102`: argument names on
  `/autocompleter` are checked again, and a text with an ampersand can get `403` on suggestions.
- Delete rule `9871102` altogether to remove every exception from `/autocompleter`; the address-bar
  and typed-search suggestions then get `403` for any text that looks like an attack.

The values of other arguments, headers, cookies, other paths and other methods stay under the full
CRS.

Controls (run on the shipped file, 2026-11-26): 132 requests with the same payloads where no
exclusion applies; 126 got `403` from the rule the exclusion removes, and for the other 6 (shell and
SQL text in the `Referer` and `User-Agent`) no CRS rule matched that text in that header either.
Twelve more requests put markup, a PHP setting, a shell string, SQL and a traversal in the *name* of
an argument. On `/search` all 12 got `403`. On `/autocompleter` the 7 that trip one of the ids
the rule leaves out (`941120`, `941230`, `941250`, `941270`, `941280`, `941300`, `933120`) got `403`
from that id, and the 5 that trip an id the rule removes (a `$(wget ...)` string, `x && cat
/etc/passwd`, `<script>`, `' or '1'='1`, `../../etc/passwd`) answered `200`. Before the change
(the 2026-11-25 `9871102`, with 71 ids on names, run again on 2026-11-26) the 7 answered `200` too. The 132 cover another argument on `/search` and `/autocompleter` (`GET`, `POST`), `HEAD`, a path
suffix, a path prefix, `;` in the path, a different case, the text in a cookie, the `Referer` or the
`User-Agent`, eight other paths with `q`, `POST /autocompleter`, a traversal in the image URL, an XSS
blob in the preferences link, and the four checks that stay on the URI of a `POST /search`.

Not covered, so a payload there is blocked: the `q` argument on `/` (the address bar sends
`/search?q=`, not `/?q=`), the `preferences` blob, and any plugin you add.

## Validation

Stack: SearXNG `2026.11.13-8a53a6aa5`, Valkey 8, `bunkerity/bunkerweb-all-in-one:1.6.17` (CRS
4.29.0), default paranoia level and anomaly thresholds, `SECURITY_MODE=block`, the service created
in easy mode from the template imported through a CRS-protected web UI. Each result below is
attributed to its run; the numbers apply to the shipped `searxng_false_positives.conf` only where the
run says so.

- **Run on the shipped file, 2026-11-26** (a Python `requests` script, a Chrome user agent, over HTTPS): the 799
  distinct searches sent as `GET /search?q=` and as `GET /autocompleter?q=`, plus a canary at each
  end: 1,602 requests, all `200`. The 23 typed texts that hold an ampersand, every prefix, sent the
  way the page script does (310 requests): 306 `200`, 4 `403` (`921200`, on the value of the
  argument the ampersand creates, on the last four prefixes of `(&(uid=*)(userPassword=*))`). The
  799 searches put after `x&` (799 requests): 753 `200`, 46 `403` on the value of an argument
  other than `q` or on a line feed or `%00`. The 2026-11-25 `9871102` (71 ids on names), run
  again on 2026-11-26, gave the same 4 and the same 46 on the same texts, so the narrower rule
  loses none of them.
- **Chrome on the shipped file, 2026-11-26** (HTTPS, a desktop user agent, typed key by key so the
  autocompleter runs, then Enter): 13 searches, namely the nine of the earlier browser run
  (`cat /etc/shadow; rm -rf /`, `echo hello && whoami`, `powershell -enc SQBFAFgA`,
  `javascript:alert(document.cookie)`, `select name from sqlite_master`, `fopen("data.txt", "r")`,
  `alert(1)`, `<?php system($_GET[0]); ?>`, `getClass().forName("java.lang.Runtime")`) and four with an
  ampersand (`ls && cat /etc/passwd`, `(&(uid=*)(userPassword=*))`, `cat /etc/shadow; rm -rf / & ls`,
  `a & b=1`). Every search reached the results page and kept its text in the box. The 110
  autocompleter requests answered `200`, except 2 `403` for `(&(uid=*)(userPassword=*))`. Chrome did
  not run the 799-search list: the `requests` run covers it.
- **Earlier runs**, three configurations, none of them the shipped file:
  - **2026-11-18, the first template** (no `ARGS_NAMES` exclusion, 32 distinct ids on `ARGS:q`, each
    listed for both routes, 64 lines): the request matrix, settings, browser smoke and web UI import
    in the details below.
  - **2026-11-25, the template before this change** (71 ids on `ARGS:q` and 71 on `ARGS_NAMES`):
    searches in eight lists of about 860 queries with duplicates. Against the first template,
    101 of the 602 searches in lists 1 to 4 and 6 got `403`. With the 71 ids on `ARGS:q`, the re-run
    of those lists still had one query answered `403` on both endpoints, and so did the full
    eight-list run; both came before the last change of that day. After that change, 177 queries
    selected from the lists and two canary entries passed on both endpoints (358 responses, all
    `200`). The full eight-list run was not repeated on that final file.
  - **2026-11-26, the same 2026-11-25 file**, run again to compare with the shipped file: the
    figures above that name it.

<details>
<summary>Evidence behind the claims above</summary>

- **Request matrix** (2026-11-18, the first template, which had no `ARGS_NAMES` exclusion;
  `evidence/matrix-final2.tsv`): 270 requests
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
- **Settings** (2026-11-18): each non-default setting was switched back to the BunkerWeb default and
  the failure above reproduced.
- **Browser smoke** (2026-11-18, Chrome through Playwright, bad behavior on, the first template, only
  `SERVER_NAME` and `REVERSE_PROXY_HOST` set). With the limiter off: searches for `' OR 1=1 --`,
  `../../etc/passwd`, `<script>alert(1)</script>`, `$(id)` and a 60-character Japanese sentence each
  rendered 8 results with the text kept in the box, the image category loaded, page 2 loaded, the
  preferences form saved and the autocompleter answered while typing; 219 responses, all `200`.
  With the limiter and link token on: the same flow, 232 responses, all `200`. The BunkerWeb log
  for each run shows no ModSecurity denial, no ban and no `403` or `444`
  (`evidence/smoke-limiter-off.summary`, `evidence/smoke-limiter-on.summary`).
- **Web UI import:** on 2026-11-18, raw mode with `template.json` plus the config file imported and
  was listed (7 steps, 11 settings, 1 config). On 2026-11-25 the import went through a web UI protected by
  BunkerWeb's own CRS (JSON, config and full Save): no rule fired.

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
- **Restarting SearXNG.** While the SearXNG container is stopped or starting (about 20 seconds)
  BunkerWeb answers `502`. Stopping it and starting it again, and recreating it with a new address,
  both recovered by themselves with no BunkerWeb restart or reload (checked on 2026-11-25). If a
  `502` stays after SearXNG answers on its own port, check that the name in the upstream address
  still resolves from BunkerWeb.
- **Long searches.** A search by `GET` carries the text in the URL: about 5,000 characters worked,
  20,000 got `414` from nginx before the WAF. A form `POST` (the "method" preference) took 20,000.
- **Not supported through the template:** a sub-path deployment (`X-Script-Name`).

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
