<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/opengym.svg" alt="openGym logo" width="96" height="96" />
</p>

<h1 align="center">openGym</h1>

<p align="center">Passkey PWA template with JSON sync and WebAuthn tuning</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/openGym-v1.4.1-248651?labelColor=1d6f93&style=flat" alt="openGym v1.4.1" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#files">Files</a> ·
  <a href="#validation">Validation</a>
</p>

Provision a curated BunkerWeb configuration for openGym, the workout tracking PWA. This
template ships sensible defaults for TLS, reverse proxying, request throttling on the passkey and
pairing endpoints, HSTS, and CRS exclusions tuned for WebAuthn, Web Push and workout notes, so a
self-hosted instance works through BunkerWeb without breaking sign-in or cloud sync.

openGym ships no HSTS and no rate limiting on its passkey and pairing endpoints, leaving both to the
reverse proxy (it only throttles password sign-in and device-link redemption itself). Validated
against openGym v1.4.1 and BunkerWeb 1.6.17 with CRS 4.29.0.

## At a glance

| Fact        | Value                                         |
| ----------- | --------------------------------------------- |
| Template id | `opengym`                                     |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, openGym v1.4.1 |
| Configs     | 1 (modsec-crs)                                |
| Rule ids    | `9866000` to `9866999`                        |
| Upstream    | `http://myopengym:80`                         |

## Prerequisites

- A running openGym upstream accessible on your network. Proxy to the bundled nginx front end (for
  example a container named `myopengym`), never to the API container: the front end serves the
  single-page application and forwards `/api/` itself.
- openGym `1.4.1`, the version this template was tested with. The images are
  `ghcr.io/duartesantos8/opengym-web:1.4.1` and `ghcr.io/duartesantos8/opengym-api:1.4.1`; the tags
  have no `v`.
- On the openGym API container, set `RP_ID` to the bare host name BunkerWeb serves (for example
  `gym.example.com`) and `ORIGIN` to the public HTTPS address of the same name
  (`https://gym.example.com`, with the port if it is not 443). Do this before anyone registers a
  passkey: a passkey only works on the name it was created for, so changing `RP_ID` later invalidates
  every passkey. With a wrong `RP_ID` or `ORIGIN`, creating or using a passkey fails verification.
  Also set `TRUST_PROXY=1`, so the application's own throttle counts each visitor's address instead
  of BunkerWeb's.
- A browser address that is HTTPS: passkeys do not work over plain HTTP.
- The BunkerWeb UI or the ability to edit multisite settings directly.
- Domain name(s) that will serve the openGym instance.

## Install

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#install) for the web
     UI or plugin bundle method.
2. **Assign the template** to your openGym service: in the web UI, open **Services**, **Create new
   service**, and pick **Opengym** in the **Templates** list above the steps.
3. **Customize the settings** highlighted in the template steps (domains, upstream host, TLS
   options). In step 1, switch **Automatic Let's Encrypt** off if the name is not reachable from the
   internet. BunkerWeb 1.6.17 does not keep a toggle you switch off while creating the service, so
   after the first save open the service again, switch it off again and save.
4. **Save the service.** BunkerWeb applies it within about 15 seconds, with no separate reload.
   Verify that registration, sign-in and a workout save all complete.

## Settings

- Update `REVERSE_PROXY_HOST` to the URL of your openGym front end (e.g. `http://myopengym:80`).
  The bundled nginx listens on `NGINX_PORT`, which defaults to `80`.
- Raise `MAX_CLIENT_SIZE` and `MODSECURITY_REQ_BODY_NO_FILES_LIMIT` together if you raise the
  application's own body cap; leaving them out of step is the usual cause of a failing sync.
- `LIMIT_REQ_URL_1` covers `/api/login/`, `/api/register/` and `/api/pair/` at `20r/m`. All three
  are unauthenticated: registration answers whether an invite code is valid, and `/api/pair/redeem`
  exchanges a one-shot five-minute code for a session token. Loosen it only if you have another
  brake in front. `/api/device-link/` is left out on purpose: openGym 1.4.1 already answers `429`
  after ten wrong device-link codes a minute from one address (measured), and those codes are 60 bits
  long.
- `LIMIT_REQ_URL_2` covers `/api/coach/`, the Bearer-token API. Those routes are exempt from the
  application's own cross-origin check, so the proxy is the only limit on token guessing.
- `BLACKLIST_URI` blocks `/api/health`, which reports the total number of registered users. The
  container health check reaches it over loopback, so blocking it externally costs nothing. Point
  external uptime monitoring at `/` instead, or remove the entry if you need the endpoint public.
- `BAD_BEHAVIOR_STATUS_CODES` omits `401` on purpose. `GET /api/me` returns `401` to every anonymous
  visitor on first page load, so the stock list bans ordinary users. `400` stays in the list: more
  than ten `400` answers from one address within a minute (for example wrong device-link or pairing
  codes) ban that address from the whole service for 24 hours, which also blocks everyone behind the
  same NAT. Normal use never gets near that. To lift a ban, open the **Bans** page in the web UI and
  unban the address.
- `KEEP_UPSTREAM_HEADERS` preserves the application's own `Content-Security-Policy`,
  `X-Frame-Options` and `Referrer-Policy`, which are stricter than the BunkerWeb defaults, and its
  `Permissions-Policy` if a later release sends one. HSTS is added by BunkerWeb because openGym ships
  none.
- `PERMISSIONS_POLICY` is `publickey-credentials-create=*, publickey-credentials-get=*`, in step 5.
  BunkerWeb's default policy denies both features, and openGym 1.4.1 sends no `Permissions-Policy`
  of its own, so without this value the browser refuses to create or use a passkey and **Create
  passkey** does nothing. This value replaces BunkerWeb's default list of about 90 features: the
  others (camera, microphone, geolocation and the rest) are no longer denied by this header and fall
  back to the browser's own defaults. `*` costs nothing here, because openGym cannot be framed
  (`X-Frame-Options: DENY`). A later BunkerWeb release (core card BW-2367) lets the template carry
  the full default list plus the two passkey entries.
- Since v1.2.11 the cross-site request check reads `Sec-Fetch-Site` first and falls back to
  `Origin`, treating a request with no `Origin` as non-browser traffic. BunkerWeb forwards both
  unchanged, so nothing needs configuring — but do not add a `REVERSE_PROXY_HEADERS` entry that
  sets or rewrites either header, and make sure `Authorization` reaches the upstream: the paired
  mobile client and the coach API both authenticate with a Bearer token.
- Antibot is not enabled. openGym 1.4.1 sends `Content-Security-Policy: frame-ancestors 'none'` and
  nothing stricter, so that policy does not get in the way of a challenge page. The reason is the
  API: with the `javascript` antibot type, API calls such as `/api/me` answer `302` to `/challenge`,
  which the application, the paired mobile app and the Bearer-token coach API cannot solve. If you enable antibot, also set `ANTIBOT_IGNORE_URI` to `^/api/`, and watch that the mobile
  app still pairs.
- This template covers the main application vhost only. An openGym deployment may also expose a
  separate card-preview host and an MCP endpoint; the MCP endpoint uses a long-lived
  `text/event-stream` and is normally kept off the public vhost entirely.
- Review and narrow the WebAuthn exclusions in `opengym_false_positives.conf` for your
  installation. In the web UI, open the service and go to step 7 (**Security - Review WAF tuning and
  exposure**) in easy mode: the file opens in an editor there. The exclusions remove individual JSON
  parameters with `ctl:ruleRemoveTargetById`, scoped by an anchored path and the request method, so
  each CRS rule stays active everywhere else. `PUT /api/data` instead excludes the `attack-sqli` and `attack-rce` families for that exact path
  and method. Routine and workout notes are free text, and openGym stores the state as a per-user
  JSON file (`api/server.js`), never as SQL or a shell command. Array indices vary, and regex
  runtime target exclusions fail to parse on the tested BunkerWeb 1.6.17 build, so this exception
  covers the whole request, including query arguments, headers and sync metadata outside `state`.
  XSS, LFI, PHP and protocol checks remain active, and anomaly scoring is unchanged.
- If you raise the CRS paranoia level to 4, add an exclusion for rule 920273 as well. It matches
  base64url bodies but is inert at the default level 1, and it targets `REQUEST_BODY` rather than a
  named parameter, so it is left out of the shipped file. In the web UI, open **Configs**, **Create
  Custom Config**, choose your service, set the type to `MODSEC_CRS`, name it and paste the rule.

### The session cookie is `__Host-` prefixed

When the application's `ORIGIN` is an `https://` URL, openGym names its session cookie
`__Host-gymsid`. Browsers reject a `__Host-` cookie outright unless it is `Secure`, uses `Path=/`,
and carries **no** `Domain` attribute. The rejection is silent: the application never sees an error,
and the user simply never appears signed in.

> [!WARNING]
> So `COOKIE_FLAGS` may add flags, but it must never introduce a `Domain` or a different `Path`, and
> `COOKIE_AUTO_SECURE_FLAG` must stay `yes`.

Safe (browser keeps the cookie):

```text
"COOKIE_FLAGS": "* HttpOnly SameSite=Lax"
"COOKIE_AUTO_SECURE_FLAG": "yes"
```

Breaks every sign-in (browser drops the cookie):

```text
"COOKIE_FLAGS": "* HttpOnly SameSite=Lax Domain=example.com"
```

Set the application's own `ORIGIN` to the public `https://` URL as well. If it is left on `http://`,
openGym falls back to the unprefixed `gymsid` name and sends it without `Secure`.

### PUT and the two body limits

Cloud sync writes the whole application state with a single `PUT /api/data`, capped at 5 MB by the
application. Two BunkerWeb defaults block that, and they have to be raised together: nginx enforces
`MAX_CLIENT_SIZE`, but ModSecurity enforces `MODSECURITY_REQ_BODY_NO_FILES_LIMIT` first, so a
generous `MAX_CLIENT_SIZE` alone still fails at 128 KB.

Default (sync returns 405, then 413 once the method is allowed):

```text
"ALLOWED_METHODS": "GET|POST|HEAD|QUERY"
"MAX_CLIENT_SIZE": "10m"
"MODSECURITY_REQ_BODY_NO_FILES_LIMIT": "131072"
```

Updated (sync works):

```text
"ALLOWED_METHODS": "GET|POST|HEAD|PUT|OPTIONS|DELETE"
"MAX_CLIENT_SIZE": "6m"
"MODSECURITY_REQ_BODY_NO_FILES_LIMIT": "6291456"
```

`OPTIONS` reaches the API so the official mobile app can complete its CORS preflight. The API
reflects `Origin` without allowing credentials, and the paired app authenticates with a Bearer
token. `DELETE` serves the account password, email and passkey removal routes. These handlers live
in `api/server.js`; the web PWA itself remains same-origin.

## Files

- `template.json` – BunkerWeb template definition containing default settings, configs, and guided
  steps.
- `configs/modsec-crs/opengym_false_positives.conf` – ModSecurity CRS tuning for the base64url WebAuthn
  ceremony bodies, the Web Push subscription endpoint and user-authored state sync.

## Validation

Run `jq . template.json` to confirm the JSON definition is valid before importing via the UI.

Load the snippet as a service-scoped `modsec-crs` custom config on a multisite BunkerWeb instance,
then run `nginx -t` in its BunkerWeb container before reloading it. A syntax error names the
file and line; a successful parse alone does not prove an exclusion matches a request.

Local validation used `bunkerity/bunkerweb-all-in-one:1.6.17`, libmodsecurity 3.0.17, CRS 4.29.0,
and a dummy upstream that always answered 200. All four chains parsed and reloaded together.
JSON state bodies included `state`, `stamped`, `baseRev` and `baseWid`:

| Request or note | Before sync exclusion | After |
| --- | --- | --- |
| `PUT /api/data`, note containing `; history` | 403 | 200 |
| `PUT /api/data`, `21–15–9; load not reported` | 403 | 200 |
| `PUT /api/data`, French workout note containing `; wall` | 403 | 200 |
| `PUT /api/data`, reconstructed `20 kg bar` note with `; load not reported; sos` | 403 | 200 |
| Same blocked note on `PUT /api/other` | 403 | 403 |
| Same blocked note on `POST /api/data` | 403 | 403 |
| `<script>alert(1)</script>` in a note on `PUT /api/data` | 403 | 403 |

The bar note's full production text was unavailable in the truncated audit log. The reconstruction
reproduced a SQLi-family block, not the original libinjection fingerprint `sos` (`942100`).

The passkey headers were checked in headless Chrome 158 with a virtual authenticator, on a service
created in easy mode from this template, over HTTPS with `RP_ID` and `ORIGIN` set to its host:

| Step | BunkerWeb's default `PERMISSIONS_POLICY` | This template |
| --- | --- | --- |
| `navigator.credentials.create` and `.get` | `NotAllowedError`, feature not enabled | succeed |
| **Create passkey** | `POST /api/register/options` `200`, no `register/verify` | `register/verify` `200` |
| **Sign in with passkey** | not run | `login/verify` `200` |
| **Add another device** code | not run | `account/device-link` `200` |

> [!TIP]
> When rechecking a deployment, keep bad-behavior bans disabled on the isolated test instance so
> negative controls do not ban the test client. Reproduce the note's 403 before adding the exclusion;
> afterwards, require the path, method and XSS controls above to remain blocked by CRS.

After a real sign-in, confirm the WebAuthn exclusions matched the parameters they were meant to.
Set `SecDebugLogLevel 9` and look for `Skipped rule id` on `/api/login/verify`. If a CRS rule still
fires there, read `matched_var_name` from the log line: that is the exact target name to add to the
chain, and it is the only reliable source for it.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
