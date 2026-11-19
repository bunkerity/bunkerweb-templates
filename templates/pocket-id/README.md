# Pocket ID Template

Put an existing [Pocket ID](https://pocket-id.org/) server behind BunkerWeb with ModSecurity and the
OWASP CRS on: passkey sign-in, the admin interface and API, OpenID Connect clients (authorization
code with PKCE, PAR, device flow, client credentials, refresh tokens, logout), single sign-on
through a relying party such as oauth2-proxy, and clients whose callback is a LAN, loopback or
IPv6 address.

## Prerequisites

- Run Pocket ID `v2.19.3` or a close release. BunkerWeb must reach `http://pocket-id:1411` on a
  private network; do not publish that port.
- Pocket ID is passkey-only and passkeys bind to the host name. Pick the final public host name
  **before** the first administrator registers a passkey: a passkey made on `id.example.com` does
  not work on another name. WebAuthn also needs a secure context, so the site must be served over
  HTTPS. `AUTO_LETS_ENCRYPT=yes` makes BunkerWeb request its certificate (the default HTTP
  challenge needs inbound port 80). For existing certificate files use `AUTO_LETS_ENCRYPT=no`,
  `USE_CUSTOM_SSL=yes`, `CUSTOM_SSL_CERT` and `CUSTOM_SSL_KEY`.
- `APP_URL` on the Pocket ID container must be the exact public HTTPS origin, for example
  `https://id.example.com` (see the next section).
- Set `ENCRYPTION_KEY` on the Pocket ID container and keep it with the data directory. Pocket ID
  needs it to read what it stored encrypted; a recreated container without it cannot.
- Open the setup page (`/setup`) to create the first administrator before the site is public.
- Serve Pocket ID at the root of the host name: the template proxies `/` and was not tested under
  a sub-path.

## Configure Pocket ID for the proxy

Set these on the Pocket ID container. "Without it" is what was measured on `v2.19.3`, or what the
Pocket ID documentation and source at `v2.19.3` say where marked (docs / source).

| Pocket ID setting                  | Value                                                  | Without it                                                                                                                                                                                                                                                                                                                                                           |
| ---------------------------------- | ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `APP_URL`                          | `https://id.example.com` (no path, no trailing slash)  | Issuer, redirects and the WebAuthn relying party come from it (docs). With an `https` value the session cookies are named `__Host-...` (measured: `__Host-access_token` with `HttpOnly; Secure; SameSite=Lax`, identical through BunkerWeb and direct, `cookies.txt`). A different host name than the browser uses breaks passkey sign-in.                             |
| `TRUST_PROXY`                      | BunkerWeb's address or network on the Docker network   | Pocket ID takes the client address from `X-Forwarded-For` only from trusted senders (docs). See the security line below.                                                                                                                                                                                                                                              |
| `INTERNAL_APP_URL` (only if used)  | an address Pocket ID can reach itself on               | Used when the server must call its own public address and cannot reach it (docs). Leave it unset unless your container cannot resolve the public name.                                                                                                                                                                                                                |
| `TRUSTED_PLATFORM` (only if used)  | `CF-Connecting-IP` behind Cloudflare, otherwise unset  | Reads the client address from that header (docs). Do not set it for plain BunkerWeb: a client could then send the header. The documentation also says to switch Cloudflare Rocket Loader off for the web app.                                                                                                                                                         |
| `ALLOW_INSECURE_CALLBACK_URLS`     | leave unset                                            | Allows insecure (`http`) callback URLs (docs). Only a client that really needs `http://<LAN host>/callback` needs it. The CRS exclusions in this template let such a URL through the WAF; the app's own check is separate.                                                                                                         |
| `DISABLE_RATE_LIMITING`            | leave unset                                            | The app has its own limiter: 100 requests a second with a burst of 300 per client address (source). It sees BunkerWeb's view of the address only with `TRUST_PROXY` set as above.                                                                                                                                                                                     |
| `LOG_LEVEL`                        | `info` or higher, but know what it prints              | Pocket ID logs the request path, the query string and the `Referer` at `INFO` (upstream issue 1903, fixed in the next major version). One-time login links (`/lc/<code>`, `/st/<token>`) and authorization URLs (`state`, `nonce`) are in those fields: protect the log.                                                                                                                                       |

New OIDC clients are group-restricted by default (source): check the client's allowed groups
before you test a sign-in.

Security: `TRUST_PROXY` must name only BunkerWeb, never `0.0.0.0/0`. With a wide value, anyone who
reaches the Pocket ID port directly chooses the client address shown in its audit log and used by
its rate limiter. Measured with a forged `X-Forwarded-For: 6.6.6.6` sent through BunkerWeb:

| `TRUST_PROXY`           | BunkerWeb default `X-Forwarded-For` | With this template                       |
| ----------------------- | ----------------------------------- | ---------------------------------------- |
| BunkerWeb's address     | app saw the real client address     | app saw the real client address          |
| `true` (everyone)       | app saw `6.6.6.6`, the forged one   | app saw the real client address          |

The template's `REVERSE_PROXY_HEADERS=X-Forwarded-For $remote_addr` makes the header exactly the
address BunkerWeb saw (`xff.txt`). It protects a request that goes through BunkerWeb; a client that
reaches the app's port directly can still send anything, so keep that port private.

## Import and initial detect run

Import `template.json` and the files under `configs/` using the repository
[installation guide](../../README.md#install), or its plugin bundle layout. Assign
`USE_TEMPLATE=pocket-id`, then change only `SERVER_NAME` and `REVERSE_PROXY_HOST` (keep the
upstream URL without a trailing slash or path). In the web UI raw import, the config dialog lists
the file as `modsec_crs/pocket-id_false_positives.conf`: upload it under that name.

Before enabling blocking on your instance:

1. Set service `SECURITY_MODE=detect` and reload.
2. Register a passkey, sign in, create a client with your real callback URLs, run one sign-in
   from your relying party, sign out, and read the BunkerWeb logs.
3. Confirm the included exclusions match the targets in your logs. Narrow or add exceptions only
   where your traffic requires them.
4. Switch to `SECURITY_MODE=block` after those checks and repeat the workflow.

False positives return `403` in block mode. `BAD_BEHAVIOR_THRESHOLD` matching responses in 60
seconds ban the client for one day, so the detect run matters.

## Settings and tradeoffs

Each setting was put back to the BunkerWeb default (or removed) and the failure below reproduced,
on the real image, through BunkerWeb with a certificate trusted by the test client.

| Setting                                              | Purpose and measurement                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `USE_REVERSE_PROXY=yes`, `REVERSE_PROXY_URL=/`, `REVERSE_PROXY_HOST=http://pocket-id:1411` | Proxy the app. Keep the host path-less.                                                                                                                                                                                                                                                                                                                                                                                                       |
| `ALLOWED_METHODS=GET\|POST\|HEAD\|OPTIONS\|PUT\|PATCH\|DELETE` | The admin API uses `PUT`, `PATCH` and `DELETE`; the token, userinfo and discovery endpoints answer browser preflights with `OPTIONS`. The default `GET\|POST\|HEAD\|QUERY` answered `405` to all four (`PUT /api/users/<id>/user-groups`, `PATCH /api/webauthn/credentials/<id>`, `DELETE /api/oidc/clients/<id>`, `OPTIONS /api/oidc/token`). With the setting the app answered `400`, `400`, `404` and `204`.                                                |
| `REVERSE_PROXY_INTERCEPT_ERRORS=no`                  | Keep Pocket ID's own JSON errors. At the default, BunkerWeb swapped every upstream `400` and `401` for its HTML page (`content-type: text/html`), including `authorization_pending` on `POST /api/oidc/token`, `request_unauthorized` on `GET /api/oidc/userinfo` and `not_signed_in` on `GET /api/users/me`. Relying parties and the web app read those bodies. With `no` they arrive as `application/json`.                                                    |
| `REVERSE_PROXY_HEADERS=X-Forwarded-For $remote_addr` | Replaces BunkerWeb's `X-Forwarded-For $proxy_add_x_forwarded_for`, which appends to what the client sent. With `TRUST_PROXY=true` and the default, a client `X-Forwarded-For: 6.6.6.6` reached the app as the client address. With the setting the app saw the real client address in both cases (table above).                                                                                                                                                     |
| `MAX_CLIENT_SIZE=11m`                                | Pocket ID limits an upload to 10 MiB of the whole body and answers `413` with `file_too_large` in JSON. At the default `10m`, files of 10485700 and 10486000 bytes got BunkerWeb's HTML `413`; with `11m` the same two files got the app's JSON `413`, and 11 MiB (11534336 bytes) and 12 MiB still got BunkerWeb's `413`. Files up to 10485500 bytes passed with both values.                                                                                     |
| `LIMIT_REQ_RATE=20r/s`                               | The web app asks for the same URL several times at once: in the final browser run `GET /api/users/me` was requested 7 times inside one second, `GET /api/application-configuration` 6 times. BunkerWeb counts per client address and exact URL. At the default `2r/s` a signed-out page load got `429` on both URLs (6 `429` lines in `spa-429-default-limit-req.console.log`) and 10 of a burst of 12 requests to one URL got `429`; with `20r/s` all 12 passed. A burst of 30 got 20 passes and 10 `429`, so the limit still caps a flood. Many users behind one address (a NAT) share the budget: raise it above 20. |
| `BAD_BEHAVIOR_STATUS_CODES=403 405 429 444`          | BunkerWeb bans an address for one day after `BAD_BEHAVIOR_THRESHOLD` (10) answers with a status in this list within a minute. The default list is `400 401 403 404 405 429 444`. Pocket ID answers `401` (`/api/users/me` while signed out), `404` (the default logo) and `400` (`authorization_pending` while a device waits for approval) in normal use. At the default, 6 signed-out page loads (3 counted answers each) banned the address (the next `GET /healthz` got `403`); with the setting 20 loads left it unbanned. A device-flow client polling `POST /api/oidc/token` every 5 seconds was banned at the 11th poll (`+50s`, `403`); with the setting 14 polls over 65 seconds were never counted. `403` and `429` still count, so ModSecurity blocks, rate-limit hits and scanners are banned as before. |
| `PERMISSIONS_POLICY` (BunkerWeb's 1.6.17 default list with two entries changed) | The default sets `publickey-credentials-create=()` and `publickey-credentials-get=()`, which turns passkeys off for the page itself. In headless Chrome `navigator.credentials.create` failed with `NotAllowedError: The 'publickey-credentials-create' feature is not enabled in this document`. The template sets both to `(self)` and leaves the other 92 entries as they are; `featurePolicy.allowsFeature` then returned `true` for both and passkey registration and sign-in worked (`permissions-policy-diff.txt`). If you update BunkerWeb, compare the default list again. |
| `configs/modsec-crs/pocket-id_false_positives.conf`  | See the next section.                                                                                                                                                                                                                                                                                                                                                                                                                          |

Measured and dropped because they made no difference on `v2.19.3`: `SERVE_FILES` (eight paths,
the web app, discovery documents, a missing file and the admin API, returned the same status and
size with `yes` and `no` because `/` is proxied), and `COOKIE_FLAGS` (the app's `__Host-` cookies
arrive unchanged). `AUTO_LETS_ENCRYPT=yes` is the usual public setting; it was not exercised
offline (the test host used a custom certificate).

`LIMIT_REQ_*` is set only for the reason in the table above.

curl and some scripts send `Expect: 100-continue` on bodies above 1 MB, which CRS rule `920450`
blocks. Browsers and relying-party libraries do not send it.

## CRS exception scope and cost

`configs/modsec-crs/pocket-id_false_positives.conf` is a service-scoped **modsec-crs** config: one
rule per argument, ids `9869100` to `9869160`. A rule removes CRS rules from its one argument only when
**all** of these hold, checked in this order:

1. the path is exactly the endpoint's path;
2. the method is the listed one;
3. the argument occurs exactly once, and where the client sends it (query string for a `GET`, form
   body for a `POST`; a JSON argument must not also be in the query string);
4. for a URL argument, the value is a plain `http` or `https` URL whose host is an IPv4 literal
   (not `169.254.x.x`), an IPv6 literal (not a spelling of `169.254.0.0/16`: no IPv4-mapped
   `::ffff:`, no NAT64 `64:ff9b::/32`, no 6to4 `2002::/16`, no literal whose last 32 bits are
   `169.254.x.x`, that is whose last two groups are `a9fe:x`; and not the AWS and Google Cloud
   metadata prefixes `fd00:ec2:` and `fd20:ce:`) or a local
   development name (`localhost`, `localhost.localdomain`, `ip6-localhost`, `host.docker.internal`,
   `gateway.docker.internal`, `host.containers.internal`, `lvh.me`, `localtest.me`), then an
   optional port, path and query made of URL-safe characters only (no quote, space, `;`, `|`,
   backquote, `$`, parenthesis, brace, bracket, backslash or `#`).

A value that fails a condition is inspected by every CRS rule, the removed ones included. Anchors
are `\A` and `\z`: `@rx` is multiline on BunkerWeb 1.6.17, so `^` and `$` would also match at a
line break inside a value or a path.

| Ids                | Method and path                                           | Argument                                                | Removed CRS rules                                                              |
| ------------------ | --------------------------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `9869100`            | `GET /authorize`                                          | `redirect_uri` (query)                                  | `931100 932130 934110 934190`                                                  |
| `9869101`            | `POST /authorize`                                         | `redirect_uri` (form body)                              | same                                                                           |
| `9869102`            | `POST /api/oidc/par`                                      | `redirect_uri` (form body)                              | same                                                                           |
| `9869103`            | `POST /api/oidc/token`                                    | `redirect_uri` (form body)                              | same                                                                           |
| `9869104`            | `GET /api/oidc/end-session`                               | `post_logout_redirect_uri` (query)                      | same                                                                           |
| `9869105`            | `POST /api/oidc/end-session`                              | `post_logout_redirect_uri` (form body)                  | same                                                                           |
| `9869110` to `9869119` | `POST`, `PUT /api/oidc/clients`, `/api/oidc/clients/<id>` | `callbackURLs`, entries 0 to 9 (JSON)                   | same                                                                           |
| `9869120` to `9869129` | same                                                      | `logoutCallbackURLs`, entries 0 to 9 (JSON)             | same                                                                           |
| `9869130`, `9869131`   | same                                                      | `launchURL`, `backchannelLogoutURL` (JSON)              | same                                                                           |
| `9869140`            | `POST`, `PUT /api/scim/service-provider`, `.../<id>`      | `endpoint` (JSON)                                       | same                                                                           |
| `9869150`, `9869151`   | `PUT /api/application-configuration`                      | `smtpPassword`, `ldapBindPassword` (JSON, no URL check) | `932125 932130 932250 932270 933100 933180 933210 942100 942190 942290 942540` |
| `9869160`            | `PUT /api/oidc/clients/<id>`                              | the argument **names** `credentials.secrets[0..4].id`, `.prefix`, `.createdAt`, `.expiresAt`, `.isActive` | `930120` (names only)                |

How the ids were chosen, all measured on the real image:

- On every URL argument the same four rules fire on the same URL shapes: `931100` on an IPv4 literal
  and most IPv6 literals, `932130` on any bracketed IPv6 host (`//[`), `934110` on `http://127.0.0.1`,
  `http://localhost` and `http://[::1]` (its SSRF phrase list), `934190` on the scheme-less
  `localhost/`, `host.docker.internal/` and `lvh.me/` (a callback such as `http://localhost/cb`
  without a port).
- On the two password arguments, 23 of 840 passwords generated from the usual symbol sets (the
  Bitwarden, KeePassXC and 1Password character sets, 12 to 64 characters) were blocked, by ten of the
  listed ids, each on fewer than one password in a hundred. `942190` is the eleventh: it fires on
  `union select` and on a password with `'`, `<>`, `&`, `;` and `$(id)`. Those two arguments are
  not screened for SQL, shell or PHP patterns by these rules.
- Saving an OIDC client in the Pocket ID web interface sends the client's secrets back
  (`credentials.secrets`). CRS `930120` reads the argument name `json.credentials.secrets.array_0.id`
  as an OS file access (`.credentials`). Only these literal names are excluded, only on this path
  and `PUT`; their values are still inspected.

Array arguments are listed for the literal indices `0` to `9`, and secrets for `0` to `4`: regex
targets, a target without the index, `ARGS_GET:` and `ARGS_POST:` targets, and macros in a target do
not work in `ctl:ruleRemoveTargetById` on this BunkerWeb. **An eleventh callback or logout URL is
blocked again**, and so is a client with more than five secrets. To raise a limit, add the index in
the same way (indices 10 and 11 were tested and work).

**Cost:** a URL that passes the four conditions is not screened for inclusion or SSRF by `931100`,
`932130`, `934110` or `934190` on that argument. Pocket ID matches `redirect_uri` against the
client's registered callbacks and the client list is admin-only, which limits what that exposes.
Everything else still inspects those arguments: the other RFI rules (`931120` and the rest),
path traversal and file inclusion (`930xxx`), shell, PHP, XSS, SQL and Java rules, and the header,
cookie and protocol rules. Cloud metadata addresses do not satisfy the grammar, so they stay
blocked: `169.254.x.x` and its decimal, octal and hexadecimal spellings, `fd00:ec2::254` (also
written `fd00:0ec2::254`, AWS), `fd20:ce::254` (also written `fd20:0ce::254` and
`fd20:00ce::254`, Google Cloud, IPv6-only instances), `metadata.*` and `instance-data`, and every IPv6 spelling of
`169.254.0.0/16` that embeds the address in its last 32 bits or in a NAT64 or 6to4 prefix:
`[64:ff9b::a9fe:a9fe]`, `[64:ff9b:1::a9fe:a9fe]` (local-use NAT64), `[2002:a9fe:a9fe::]`,
`[::a9fe:a9fe]` (IPv4-compatible), `[::ffff:a9fe:a9fe]`, `[::ffff:0:a9fe:a9fe]`, ISATAP
`…:5efe:a9fe:a9fe`, a network-specific NAT64 `/96` prefix such as `[2001:db8:1::a9fe:a9fe]`, and
the dotted-quad tails `[::169.254.169.254]` and `[64:ff9b::169.254.169.254]`, in compressed,
expanded and zero-padded writing, upper and lower case, with or without a zone. A NAT64 prefix
that is not a `/96` and a Teredo address are not recognised, and every other IPv6 literal is
accepted, including `fe80::/10` (link-local, which needs a zone and is a different range from
`169.254.0.0/16`), `fd00::/8` and `::1`. Pocket ID still compares the value with the client's
registered callbacks. `949110` and `959100` are never excluded.

Not covered, and blocked when a legitimate value trips CRS: other URL-bearing admin fields
(`logoUrl`, `darkLogoUrl`, `credentials.federatedIdentities[].issuer` and `.jwks`), custom claim
values, user and group names, and passwords made only of symbols. At paranoia level 2 and above many
more rules fire on generated passwords (see "Things to think about").

Controls: the same payload in another argument, another path, another method, a second occurrence
of the argument, a value that is not a plain URL (`;id`, a line break, a cloud metadata address) or
an index past the list still gets `403` from the rule the exclusion removes (`C01` to `C25`).

## Things to think about

- **Server-to-server endpoints.** Relying parties call the token, userinfo, JWKS and discovery
  endpoints from a server, not a browser. Keep antibot, a country list and similar browser
  challenges off these paths: an antibot page cannot be solved by `oauth2-proxy`. The template
  turns none of them on. 31 client user agents (curl, Go, Python, Node, Java, Ruby, `okhttp`,
  Postman and others, and an empty one) got `200` or the app's own `401`, never `403`, on
  discovery, JWKS, the token endpoint, userinfo and `/`.
- **Cross-origin browser clients.** Pocket ID answers `CORS: *` on the token, userinfo,
  end-session, introspection and well-known documents. The preflight (`OPTIONS /api/oidc/token`)
  passes (`204`) because `OPTIONS` is allowed. The rest of the API is not cross-origin: the app
  checks `Origin` and `Sec-Fetch-Site` itself, with `APP_URL` as the trusted origin.
- **No WebSocket or server-sent events** are used, so `REVERSE_PROXY_WS` stays off.
- **Silent token renewal in a hidden iframe** (`prompt=none`) does not work with Pocket ID: its own
  `frame-ancestors 'none'` header refuses the frame, with or without BunkerWeb (source). Use
  refresh tokens.
- **Device flow.** A device polls every 5 seconds for up to 15 minutes. The default bad-behavior
  list bans it (table above), so keep `BAD_BEHAVIOR_STATUS_CODES` as set here.
- **Many users behind one address.** Pocket ID's own limiter (100 per second, burst 300) and
  BunkerWeb's `LIMIT_REQ_RATE` are both per address. Raise `LIMIT_REQ_RATE` for an office or school;
  a `429` also feeds the bad-behavior ban.
- **BunkerWeb behind another proxy.** Set `USE_REAL_IP=yes` and the trusted networks first;
  otherwise `$remote_addr` is the outer proxy and every client looks the same to Pocket ID and to
  the limiters.
- **The admin interface and the API share one path space.** `/settings/...` and `/api/...` serve
  administrators and users alike, so BunkerWeb cannot expose the sign-in and OIDC endpoints while
  hiding the admin pages by path (upstream issues 1875 and 1527 plan separation for the next major
  version). Restrict the admin pages by network on another layer if you need to.
- **Backups.** Back up the data directory together with `ENCRYPTION_KEY`. Pocket ID takes an
  instance lock on its data volume: after a quick restart wait for the old process to exit before
  starting a new one, or the new one stops.
- **Outbound features.** LDAP sync, SMTP, the GeoLite database download and SCIM provisioning leave
  the Pocket ID container directly; BunkerWeb does not see them.
- **Upload size.** Client logos stop at 2 MiB and other uploads at 10 MiB, both answered by the app.
  `MAX_CLIENT_SIZE=11m` keeps the app's JSON error visible.

- **Paranoia level 2 and above.** The password exclusions list the ids that fire at paranoia level 1.
  At level 2, 517 of 840 generated passwords (letters, digits and symbols) tripped at least one more
  rule (for example `920230`, `932200`, `932240`, `942131`, `942430`), and the config does not cover
  them. Use passwords of letters and digits with a few symbols for the SMTP and LDAP accounts, or add
  the ids your detect run shows.
- **Pocket ID v3.** The next major version (not released: `v2.19.3` is the latest release,
  `main` is v3) moves the admin endpoints under `/api/admin/`
  ([issue 1527](https://github.com/pocket-id/pocket-id/issues/1527)). The paths in this config are
  the v2 paths and need updating when you upgrade.

Not verified: see the end of the next section.

## Validation and test history

Tested on 2026-11-18: Pocket ID `v2.19.3` (`ghcr.io/pocket-id/pocket-id`, digest
`sha256:189e03bb73bb86d7108fe81cbe2237c408cff2176b14cd7a39cc12a2ee8b2762`),
`bunkerity/bunkerweb-all-in-one:1.6.17` (CRS 4.29.0), `oauth2-proxy` `v7.15.6` as the relying party,
default paranoia level and anomaly thresholds, `SECURITY_MODE=block`, HTTPS on a local test host
with a certificate signed by a test CA, headless Chrome with a virtual passkey authenticator. Every
CRS exception was proven again from a clean data volume, once without the config and once with the
shipped file. After the IPv6 grammar change, the request matrix, the URL corpora, the attack table,
and the passwords were run again on the final file, byte for byte the one in this directory, on a
fresh data volume. The browser smoke ran on the previous revision of the file, which differs only by
the `fd20:ce:` alternative of the IPv6 host rule. The settings check and the two items marked
"earlier run" were not repeated.

- **Request matrix**, 158 requests, one at a time, each with a `bwt=<name>` tag so its ModSecurity
  lines are attributed per request: 61 seeds (admin requests that must succeed and read back from the
  app), 73 usage requests (what a relying party or browser sends) and 24 controls. Without the
  config 62 of the 134 seeds and usage requests got `403` (37 seeds, 25 usage), with the rule ids in
  the file; with it all 158 pass and every state change reads back (clients, claims, users, groups,
  SMTP and LDAP settings, uploads, SCIM, API keys, sign-up). All 24 controls got `403` both without
  and with the config, from the rule the exclusion removes.
- **URL corpus:** 40 callback shapes (public and public IP, LAN, loopback, `localhost` with and
  without a port, IPv6 with and without a zone, custom schemes, wildcards, `host.docker.internal`,
  `lvh.me`) on each of the 11 URL arguments, 440 requests: 209 got `403` without the config, none
  with it.
- **IPv6 spellings of the metadata address:** 38 hosts (NAT64, local-use NAT64, 6to4,
  IPv4-compatible, IPv4-mapped and IPv4-translated, ISATAP, a network-specific `/96`, dotted-quad
  tails, zoned forms, `fd00:ec2::254`, `fd20:ce::254` in six writings, `169.254.169.254`) and 9 legitimate hosts (`[fd00::1]`,
  `[::1]`, a documentation and a global address, a zoned `fe80::1`, `a9fe` elsewhere in the
  address, `fd20:cf::254`, `fd21:ce::254`) on each of the 11 URL arguments, 517 requests: all 418 requests with a metadata host got
  `403` (`931100`, `932130` or `934110`, the rules the exception removes for a legitimate URL), and
  the 99 requests with a legitimate host got no `403`. Offline, 5637 spellings of the address (every compression, with and without
  leading zeros, upper and lower case, four zone forms, 45096 URLs) are all rejected by the
  grammar, and 40 legitimate IPv6 hosts and 19999 random addresses are not.
- **Attack and smuggling table:** 499 rows (one payload per CRS family on every excluded argument;
  nine IPv6 spellings of the metadata address on each of the 11 URL arguments; the same URL
  payloads on other arguments, paths, methods, a second occurrence, a duplicate JSON key,
  a line break before, inside and after a value and a path, path variants, method override headers,
  other content types, the eleventh index). All 449 rows that must be blocked are blocked. Seven
  rows pass on purpose: `//authorize`, `/%61uthorize`, `/x/../authorize`, `/./authorize`,
  `//api/oidc/clients`, `//api/oidc/token` and `/api/oidc/tok%65n` reach the same endpoint, so
  the exception applies to them with the same value check.
- **Passwords:** 840 passwords generated from the usual symbol sets on `smtpPassword` and on
  `ldapBindPassword`: 23 blocked without the exclusions, none with them. Passwords made only of
  symbols are not covered (15 of 210 blocked, on both arguments).
- **Settings:** each non-default setting was switched back to the BunkerWeb default and the
  failure above reproduced. `SERVE_FILES` was tested and dropped.
- **Browser smoke** (bad behavior and rate limit on, template as shipped): without the config, creating
  a client with a LAN callback in the web interface got `403` (`931100`) and `oauth2-proxy` got a `403`
  page on `/authorize` (callback on `localhost`). With the config: passkey registration, sign-out
  and passkey sign-in; a client created with a LAN, a port-less `localhost` and an IPv6 callback and
  then saved with a launch URL, a back-channel logout URL and a logout URL; SCIM enabled with an IPv6
  endpoint; an SMTP password with `"`, `'`, `;`, `$(id)`, backquote and `|`; `oauth2-proxy` signed in
  through the authorization code flow with PKCE (callback on `localhost`) and signed out through a
  back-channel `end-session` request with a LAN `post_logout_redirect_uri`. The BunkerWeb access log of the
  run has 341 requests for the service (325 `200`, 5 `204`, 3 `401`, 3 `404`, 2 `201`, 2 `303`,
  1 `302`), no ModSecurity line, no ban and no `429`.
- **User agents** (earlier run, config unchanged on those paths): 31 client agents against
  discovery, JWKS, token, userinfo and `/`: 155 requests, 93 `200` and 62 `401` (the app's own answer
  to a bad token), none `403`.
- **Web UI import** (earlier run, `template.json` unchanged): raw mode with a copy of `template.json`
  (`id` and `name` changed, because the original id is already loaded as a plugin) plus the config
  file imports and is listed with 9 steps, 12 settings and 1 config next to the plugin copy.

Not verified: SMTP and LDAP (outbound mail, sync) and the e-mail sign-in and one-time-token
flows that depend on them, the GeoLite download, SCIM provisioning to a live target, back-channel
logout delivery to a live relying party, the mobile apps, CIMD client ids (the client metadata
document is fetched by the app), a client logo loaded from a URL, federated client credentials,
`form_post` response mode through a real relying party, a hidden-iframe renewal,
`AUTO_LETS_ENCRYPT` (the test host used a custom certificate), BunkerWeb behind another reverse
proxy, many browsers behind one address, paranoia level 2 and above, and Pocket ID versions other
than `v2.19.3`. Validate your own clients.
