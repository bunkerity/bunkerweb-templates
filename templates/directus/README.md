# Directus Template

## Overview

Provision a BunkerWeb configuration for [Directus](https://directus.io/), the headless CMS that
turns an existing SQL database into a REST and GraphQL API plus an admin app. The admin dashboard
and the public API share one hostname, so this template tunes the reverse proxy, request limits and
bad-behavior counting around API traffic rather than around a classic web page.

Because the API surface depends entirely on your collections and permissions, treat the defaults
below as a starting point and expect to adjust body size, rate limits and CORS.

Validated against Directus 11 and BunkerWeb 1.6.14 with OWASP CRS 4.29: the admin app, schema
and field creation, item reads and writes, Directus `filter` syntax, GraphQL, multipart uploads
and anonymous access were all exercised through the proxy.

## Prerequisites

- A reachable Directus instance (container, VM, or bare metal) that trusts the BunkerWeb proxy IP.
- Directus configured with `PUBLIC_URL` set to the public `https://` URL, so generated asset and
  OAuth redirect links point at BunkerWeb instead of the internal address.
- Access to the BunkerWeb UI or environment variables to apply template settings.
- `MAX_PAYLOAD_SIZE` raised on the Directus side if you send bodies over 1 MB. Directus caps
  JSON payloads at `1mb` by default and answers `400 INVALID_PAYLOAD` above it, whatever the
  proxy allows. Verified by sending the same 2 MB body straight to Directus with BunkerWeb out
  of the path: identical `400`.

## Setup

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#installing-templates) for the web
     UI or plugin bundle method.
2. **Assign the template** to the service that fronts Directus (`USE_TEMPLATE=directus` via
   environment variables, or pick it in the UI).
3. **Set `SERVER_NAME`** to your public hostname(s) and point `REVERSE_PROXY_HOST` at the Directus
   backend (for example `http://directus:8055`).
4. **Reload BunkerWeb** and verify you can sign in to the admin app and call the API through the
   proxy.

## Antibot is not usable here

Directus serves the admin dashboard and the API from the same hostname, so any antibot challenge
placed in front of the site also lands in front of `/items`, `/auth` and `/graphql`, where API
clients cannot solve it. Leave `USE_ANTIBOT` at `no` and rely on Directus authentication plus the
rate limits below.

## Bad behavior counts API status codes

Directus answers `401` on every unauthenticated call (the admin app polls `/auth/refresh` and
`/users/me` before sign-in), `403` whenever the public role lacks a permission, and `404` for any
item id that does not exist. All three are in BunkerWeb's default `BAD_BEHAVIOR_STATUS_CODES`, so
the defaults ban ordinary visitors:

```text
"BAD_BEHAVIOR_STATUS_CODES": "400 401 403 404 405 429 444"
```

This template counts only the codes Directus does not emit during normal use:

```text
"BAD_BEHAVIOR_STATUS_CODES": "400 405 444"
```

## The two body limits move together

nginx enforces `MAX_CLIENT_SIZE`, but ModSecurity enforces `MODSECURITY_REQ_BODY_NO_FILES_LIMIT`
first and rejects anything above it, so a generous upload size alone still fails at 128 KB on a
JSON body. `MAX_CLIENT_SIZE` additionally generates ModSecurity's `SecRequestBodyLimit`, which runs
with `SecRequestBodyLimitAction Reject`, so raising it costs memory on every buffered request.

The defaults here allow 1 GB asset uploads and 10 MB non-file bodies (bulk `PATCH /items/...`
writes, imports, large GraphQL queries). Lower `MAX_CLIENT_SIZE` if you do not serve large assets.

Measured on `POST /items/<collection>` with a JSON body, changing nothing but this one setting:

| JSON body | `MODSECURITY_REQ_BODY_NO_FILES_LIMIT=131072` (default) | `=10485760` (this template) |
| --------- | ------------------------------------------------------ | --------------------------- |
| 100 KB    | `200`                                                  | `200`                       |
| 500 KB    | `400`                                                  | `200`                       |
| 900 KB    | `400`                                                  | `200`                       |

The rejection is a `400`, not a `413`: ModSecurity truncates the body at the limit and rule
200002 then fails to parse it. Nothing in the error mentions a size, which is what makes this
one expensive to diagnose from the Directus side.

## The Expect header, and the CRS exclusion that ships with this template

CRS 4 lists `/expect/` among its always-forbidden headers, so rule 920450 answers `403` to any
request carrying `Expect: 100-continue`. curl adds that header by itself once a body passes 1 KB,
as do several HTTP client libraries, so scripted imports and bulk writes fail with a `403` that
looks like it came from Directus.

`configs/modsec-crs/directus_crs_tuning.conf` re-sets the CRS policy variable with `/expect/`
removed and every other entry kept. It is a `modsec-crs` config because rule 901165 defaults that
variable only while it is unset; the same rule placed in `modsec/` would load after CRS
initialisation and do nothing.

Verified after applying it — `Expect: 100-continue` passes, and every other restricted header is
still refused:

| Request                                       | Result |
| --------------------------------------------- | ------ |
| `Expect: 100-continue`                        | `200`  |
| `Proxy: http://evil.example`                  | `403`  |
| `X-HTTP-Method-Override: DELETE`              | `403`  |
| `Lock-Token`, `Content-Range`                 | `403`  |

## Rich text and code samples will trip the CRS

Directus stores whatever your editors type, and a CMS field holding HTML or a code sample reads
exactly like an attack payload. Measured on `POST /items/<collection>`:

| Field content                                        | Rules that fired                     | Score |
| ---------------------------------------------------- | ------------------------------------ | ----- |
| `<h2>..</h2><p>..<img src="..">..</p>` (WYSIWYG)     | 941160                               | 5     |
| `document.cookie` / `alert(1)` in a code block       | 941180, 941390                       | 10    |
| `cat /etc/passwd \| grep root` in a code block        | 930120, 932160, 932235, 932260       | 20    |

No exclusion for these ships with the template, and that is deliberate: the target is
`ARGS:json.<field>`, and the field name is whatever you called it, so there is nothing generic to
exclude. A blanket removal of the XSS and RCE rules on `/items/` would strip protection from the
main write endpoint of the API.

If your collections hold rich text, scope an exclusion to the one field, taking the target name
from `matched_var_name` in the logs:

```conf
# configs/modsec/directus_false_positives.conf
# Editors post rich HTML into articles.body. POST/PUT/PATCH under /items/articles only.
# t:normalizePath is required: REQUEST_FILENAME is URL-decoded but not normalised.
SecRule REQUEST_FILENAME "@rx ^/items/articles(?:/|$)" \
    "id:3000110,phase:1,pass,nolog,t:none,t:normalizePath,chain,\
    ctl:ruleRemoveTargetById=941160;ARGS:json.body,\
    ctl:ruleRemoveTargetById=941180;ARGS:json.body"
SecRule REQUEST_METHOD "@rx ^(?:POST|PUT|PATCH)$" \
    "t:none,nolog"
```

Remove targets, never whole rules, and confirm the same payload is still refused on another
field, another path and another method before you keep it.

## CORS is off-origin only

`USE_CORS` matters only when a decoupled front end on another origin calls the API. The template
ships `CORS_ALLOW_ORIGIN=self`, which is same-origin and effectively a no-op; replace it with your
front end's origin (a PCRE regex is accepted).

`CORS_ALLOW_CREDENTIALS` is `yes` because Directus session auth relies on cookies. Never combine
that with `CORS_ALLOW_ORIGIN=*` — browsers reject the pair, and any origin that slipped through
would carry the user's session. If no separate front end exists, set `USE_CORS=no`.

## Customization Tips

- Review the rate limits: `LIMIT_REQ_RATE=25r/s` covers an admin app that fans out many parallel
  requests per page view, but a busy public API may need more.
- ModSecurity runs in blocking mode. Directus `filter` syntax, GraphQL, deep `fields=*.*` reads,
  aggregates and multipart uploads were all exercised against CRS 4.29 and none of them tripped a
  rule, so no exclusion ships for them.
- Directus sets its own security headers, so `KEEP_UPSTREAM_HEADERS` preserves them and BunkerWeb's
  own CSP stays report-only. Review what the application sends at
  `https://<your-domain>/admin/settings/project` before tightening either side.

## Validation

- Run `jq . templates/directus/template.json` to confirm the template definition is valid JSON.
- Sign in to the admin app and exercise a collection through the API to confirm routing, client IP
  attribution, upload size and rate limits.

## Raw Config

```env
SERVER_NAME=example.com
AUTO_LETS_ENCRYPT=yes
EMAIL_LETS_ENCRYPT=registration@example.com
USE_REVERSE_PROXY=yes
REVERSE_PROXY_URL=/
REVERSE_PROXY_HOST=http://directus:8055
REVERSE_PROXY_WS=yes
REVERSE_PROXY_BUFFERING=no
REVERSE_PROXY_REQUEST_BUFFERING=no
REVERSE_PROXY_INTERCEPT_ERRORS=no
REVERSE_PROXY_READ_TIMEOUT=3600s
REVERSE_PROXY_SEND_TIMEOUT=3600s
ALLOWED_METHODS=GET|POST|HEAD|PUT|DELETE|PATCH|OPTIONS
MAX_CLIENT_SIZE=1024m
MODSECURITY_REQ_BODY_NO_FILES_LIMIT=10485760
USE_MODSECURITY=yes
MODSECURITY_SEC_RULE_ENGINE=On
SERVE_FILES=no
USE_CORS=yes
CORS_ALLOW_ORIGIN=self
CORS_ALLOW_METHODS=GET, POST, HEAD, PUT, DELETE, PATCH, OPTIONS
CORS_ALLOW_HEADERS=Content-Type,Authorization,X-Requested-With,Accept,Origin,Cache-Control
CORS_ALLOW_CREDENTIALS=yes
USE_LIMIT_REQ=yes
LIMIT_REQ_URL=/
LIMIT_REQ_RATE=25r/s
LIMIT_CONN_MAX_HTTP1=50
LIMIT_CONN_MAX_HTTP2=500
LIMIT_CONN_MAX_HTTP3=500
BAD_BEHAVIOR_STATUS_CODES=400 405 444
BAD_BEHAVIOR_THRESHOLD=20
KEEP_UPSTREAM_HEADERS=Content-Security-Policy Strict-Transport-Security X-Frame-Options X-Content-Type-Options Referrer-Policy Access-Control-Allow-Origin Access-Control-Allow-Credentials Access-Control-Allow-Headers Access-Control-Allow-Methods Access-Control-Expose-Headers Access-Control-Max-Age
CONTENT_SECURITY_POLICY_REPORT_ONLY=yes
INTERCEPTED_ERROR_CODES=404 405 413 429 500 501 502 503 504
USE_ROBOTSTXT=yes
USE_GZIP=yes
GZIP_PROXIED=expired no-cache no-store private auth
```

## Disclaimer

We are not the Directus support team; for questions about Directus itself, use their official
channels.
