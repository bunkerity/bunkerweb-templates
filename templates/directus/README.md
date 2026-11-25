<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/directus.svg" alt="Directus logo" width="96" height="96" />
</p>

<h1 align="center">Directus</h1>

<p align="center">Headless CMS template with REST and GraphQL API defaults</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Directus-11.17.4-248651?labelColor=1d6f93&style=flat" alt="Directus 11.17.4" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#validation">Validation</a> ·
  <a href="#raw-config">Raw Config</a>
</p>

Provision a BunkerWeb configuration for [Directus](https://directus.io/), the headless CMS that
turns an existing SQL database into a REST and GraphQL API plus an admin app. The admin dashboard
and the public API share one hostname, so this template tunes the reverse proxy, request limits and
bad-behavior counting around API traffic rather than around a classic web page.

Because the API surface depends entirely on your collections and permissions, treat the defaults
below as a starting point and expect to adjust body size, rate limits and CORS.

Validated against Directus 11.17.4 and BunkerWeb 1.6.17 with OWASP CRS 4.29: the admin app, schema
and field creation, item reads and writes, Flows, Directus `filter` syntax, GraphQL reads,
multipart uploads and anonymous access were all exercised through the proxy.

## At a glance

| Fact        | Value                                         |
| ----------- | --------------------------------------------- |
| Template id | `directus`                                    |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Directus 11.17.4 |
| Configs     | 3 (modsec-crs)                                |
| Rule ids    | `9853000` to `9853999`                        |
| Upstream    | `http://directus:8055`                        |

## Prerequisites

- A reachable Directus instance (container, VM, or bare metal) that trusts the BunkerWeb proxy IP.
- Directus configured with `PUBLIC_URL` set to the public `https://` URL, so generated asset and
  OAuth redirect links point at BunkerWeb instead of the internal address.
- Access to the BunkerWeb web UI.
- `MAX_PAYLOAD_SIZE` raised on the Directus side if you send bodies over 1 MB. Directus caps
  JSON payloads at `1mb` by default and answers `400 INVALID_PAYLOAD` above it, whatever the
  proxy allows. Verified by sending the same 2 MB body straight to Directus with BunkerWeb out
  of the path: identical `400`.

## Install

1. **Import the template.** Follow the repository's [installation guide](../../README.md#install).
   For the Raw import, upload `template.json` and then the three configs when the **Missing Custom
   Configs** dialog lists them.
2. **Create the service in easy mode.** Pick **Directus** in the **Templates** list at the top of
   the service page.
3. **Go through the guided steps.** In **Web service - Front service**, replace `example.com` with
   your public host name and the example e-mail with a real one. Let's Encrypt needs public DNS and
   HTTP-01 reachability: for a host that is not public, turn it off and read
   [CORS](#cors-the-default-admits-this-services-own-address-only) first. In **Web service -
   Upstream server**, point `REVERSE_PROXY_HOST` at your Directus (the default is
   `http://directus:8055`).
4. **Save the service.** BunkerWeb applies it on save. BunkerWeb 1.6.17 can keep a switch you
   turned off while creating the service (Let's Encrypt, for example): open the service again after
   the first save and check it. Then sign in at `https://<your host>/admin` and call the API
   through the proxy.

## Settings

### Antibot is not usable here

Directus serves the admin dashboard and the API from the same hostname, so any antibot challenge
placed in front of the site also lands in front of `/items`, `/auth` and `/graphql`, where API
clients cannot solve it. Leave `USE_ANTIBOT` at `no` and rely on Directus authentication plus the
rate limits below.

### Bad behavior counts API status codes

Directus answers `401` on unauthenticated calls such as `/users/me`, `403` whenever the public
role lacks a permission, and `404` for any item id that does not exist. All three are in
BunkerWeb's default `BAD_BEHAVIOR_STATUS_CODES`, so the defaults ban ordinary visitors:

```text
"BAD_BEHAVIOR_STATUS_CODES": "400 401 403 404 405 429 444"
```

This template drops `401`, `403` and `404`:

```text
"BAD_BEHAVIOR_STATUS_CODES": "400 405 444"
```

`400` stays counted, and Directus 11.17.4 answers it on ordinary requests too:

| Request                                                         | Directus answer          | Counted |
| --------------------------------------------------------------- | ------------------------ | ------- |
| `POST /auth/refresh` with no session cookie (every load of the sign-in page) | `400 INVALID_PAYLOAD`    | yes     |
| `POST /auth/refresh` with an expired session cookie             | `401 TOKEN_EXPIRED`      | no      |
| `POST /auth/refresh` with an unknown or malformed session cookie | `401` or `403`          | no      |
| `POST /auth/login` with a wrong password                        | `401 INVALID_CREDENTIALS` | no      |
| A query Directus cannot parse (`?limit=abc`), a body without a required field, a user with an invalid e-mail | `400`  | yes     |

The default is 20 counted answers within a minute (`BAD_BEHAVIOR_THRESHOLD`), then a ban of 24
hours for that address on this service. One person opening the sign-in page adds one. Twenty
people behind one address, an SDK client that refreshes without holding a token, or a bulk import
that sends invalid rows can reach it: 24 anonymous `POST /auth/refresh` in about ten seconds ended
in `403` for the sender, and the BunkerWeb log showed `increased counter for IP ... (19/20)`
first. To avoid it, raise **BAD_BEHAVIOR_THRESHOLD** in the **Security - Bad behavior** step, or
remove `400` from **BAD_BEHAVIOR_STATUS_CODES** there. The cost is that an address can send more
malformed requests before it is banned. To lift a ban, open **Bans** in the web UI, select it and
choose **Unban**.

### The two body limits move together

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

> [!NOTE]
> The rejection is a `400`, not a `413`: ModSecurity truncates the body at the limit and rule
> 200002 then fails to parse it. Nothing in the error mentions a size, which is what makes this
> one expensive to diagnose from the Directus side.

### The Expect header, and the CRS exclusion that ships with this template

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

### Rich text and code samples will trip the CRS

Directus stores whatever your editors type, and a CMS field holding HTML or a code sample reads
exactly like an attack payload. Measured on `POST /items/<collection>`:

| Field content                                        | Rules that fired                     | Score |
| ---------------------------------------------------- | ------------------------------------ | ----- |
| `<h2>..</h2><p>..<img src="..">..</p>` (WYSIWYG)     | 941160                               | 5     |
| `document.cookie` / `alert(1)` in a code block       | 941180, 941390                       | 10    |
| `cat /etc/passwd \| grep root` in a code block        | 930120, 932160, 932235, 932260       | 20    |

The editor's image button adds an `<img>` tag, so saving an item after inserting an image is
refused with a `403` (rule `941160`) until the field is excluded.

No exclusion for these ships with the template, and that is deliberate: the target is
`ARGS:json.<field>`, and the field name is whatever you called it, so there is nothing generic to
exclude. A blanket removal of the XSS and RCE rules on `/items/` would strip protection from the
main write endpoint of the API.

If your collections hold rich text, scope an exclusion to the one field, taking the target name
from `matched_var_name` in the logs. In the web UI, open **Configs**, then **Create Custom
Config**. Set **Service** to your Directus service (the default, Global, applies the rule to every
service) and **Config Type** to `MODSEC_CRS` (the default, HTTP, leaves the rule without effect).
Name it `directus_false_positives`, paste the block below with your collection and field names,
and save. BunkerWeb applies it on save.

```conf
# directus_false_positives
# Editors post rich HTML into articles.body. POST/PUT/PATCH under /items/articles only.
# t:normalizePath is required: REQUEST_FILENAME is URL-decoded but not normalised.
# Use an id from 1 to 99,999 (yours to pick), never one from a template block.
SecRule REQUEST_FILENAME "@rx \A/items/articles(?:/|\z)" \
    "id:10110,\
    phase:1,\
    pass,\
    t:none,\
    t:normalizePath,\
    nolog,\
    ctl:ruleRemoveTargetById=941160;ARGS:json.body,\
    ctl:ruleRemoveTargetById=941180;ARGS:json.body,\
    chain"
    SecRule REQUEST_METHOD "@rx \A(?:POST|PUT|PATCH)\z" \
        "t:none,\
        nolog"
```

> [!IMPORTANT]
> Remove targets, never whole rules, and confirm the same payload is still refused on another
> field, another path and another method before you keep it.

GraphQL mutations that write HTML are refused too, and a field-scoped exclusion only fits one of
the two ways to send them:

| GraphQL mutation                                                        | Rules and target                              |
| ----------------------------------------------------------------------- | --------------------------------------------- |
| HTML written inline in the document, `create_articles_item(data: {body: "<p>..."})` | `941160`, `941170` on `ARGS:json.query` |
| HTML passed as a variable, `variables: {d: {body: "<p>..."}}`           | `941160` on `ARGS:json.variables.d.body`      |

An inline document is one argument that holds every field, so no per-field exclusion exists for
it: write rich text through the REST endpoints. A client that passes values as variables can use the
recipe above with `REQUEST_FILENAME` set to `\A/graphql(?:/|\z)`, `REQUEST_METHOD` set to `POST`
and the target copied from the log. Checked that way, the variable path returned `200` while the
same HTML in another variable and the inline document stayed `403`.

### The Flows Run Script exclusion that ships with this template

A Flow's **Run Script** operation stores JavaScript in the field `options.code`, and JavaScript
reads like an attack to the CRS. Without an exception, saving a Flow that holds the sample script
Directus pre-fills is refused with a `403` and a generic error toast (rules `934100`, `941390`,
`942151` on `ARGS:json.options.code`; longer scripts also fire `932130`, `932140`, `933160`,
`942170`, `942190` and `942230`).

`configs/modsec-crs/directus_flows_script.conf` removes the XSS, SQL injection, command injection,
PHP injection, Node.js injection and file access families from that one target, `options.code`,
for `POST`, `PUT` and `PATCH` on `/operations` only (rule `9853100`). The same script text in an
item field, in another field of an operation, on `/flows`, or on another path is still refused.

**Cost:** the JavaScript of a Run Script operation is no longer inspected by those checks, so code
that holds `child_process` or `cat /etc/passwd` is saved too. Directus runs it in a sandbox and only
users who can edit Flows can write it: keep that permission for administrators. To refuse Run
Script code again, remove the config from the service.

### Passwords with symbols

A password generated with every symbol can look like an attack to the CRS, and it is refused
before Directus answers, so the user can neither set it nor sign in with it. Sent to
`POST /auth/login`, 42 of 1500 random 24 character passwords (letters, digits and all punctuation)
got `403`, from rules `942100`, `942540`, `942190`, `932130` and others; sent to `POST /users`, 20
of 600 did. None of 300 passwords made of letters, digits and `!@#$%^&*` was refused. The error
the user sees is a generic one.

`configs/modsec-crs/directus_passwords.conf` removes the SQL injection, command injection, PHP
injection, Java injection and template injection families from the `password` field of the
sign-in, password reset and `/users` routes, for `POST`, `PUT` and `PATCH` only (rule `9853110`).
With it, the same 1500 passwords, 1500 passwords of 32 characters and 600 `POST /users` calls all
passed the WAF. The same text in another field of the sign-in route, in the `password` field of
another route, in a query string, or in a `DELETE` is still refused, and so is a password that
holds `<script>`, because the cross-site scripting checks still read the field.

**Cost:** SQL, shell, PHP, Java and template text in a password is no longer inspected. Directus
hashes the password and never runs it as a query, a command or a template. To refuse those
passwords again, remove the config from the service.

### CORS: the default admits this service's own address only

`USE_CORS` matters only when a decoupled front end on another origin calls the API. The template
ships `CORS_ALLOW_ORIGIN=self`, which is not a no-op: BunkerWeb turns it into
`https://<server name>` when it serves HTTPS with a Let's Encrypt certificate and into
`http://<server name>` when it does not, and answers `403` to every request whose `Origin`
header does not match. Measured with Let's Encrypt on: a preflight and a plain `GET` from another
origin both got `403`, and a request without an `Origin` header got `200`.

> [!WARNING]
> With Let's Encrypt off and BunkerWeb's default certificate, `self` became `http://<server name>`,
> so the admin app's own requests (they carry `Origin: https://<your host>`) got `403` and the
> app showed no page. The BunkerWeb log says `denied access from cors : origin ... is not
> allowed`. A setup where TLS ends before BunkerWeb was not tested, but BunkerWeb then serves
> plain HTTP too. Replace `self` with a regular expression for your site's address, for example
> `^https://directus\.example\.com$`.

To let a front end on another origin call the API, put its address in the same expression, for
example `^https://(directus|app)\.example\.com$`, in the **HTTP - Cross-origin access** step. With
it, a preflight from the second address answered `204` with that address in
`Access-Control-Allow-Origin` and `Access-Control-Allow-Credentials: true`. The cross-origin call
from a real browser was not verified.

> [!WARNING]
> `CORS_ALLOW_CREDENTIALS` is `yes` because Directus session auth relies on cookies. Never combine
> that with `CORS_ALLOW_ORIGIN=*` — browsers reject the pair, and any origin that slipped through
> would carry the user's session. If no separate front end exists, set `USE_CORS=no`: switch it
> off, save, then open the service again and check it stayed off.

### Customization Tips

- Review the rate limits: `LIMIT_REQ_RATE=25r/s` covers an admin app that fans out many parallel
  requests per page view, but a busy public API may need more.
- ModSecurity runs in blocking mode. Directus `filter` syntax, GraphQL reads, deep `fields=*.*`
  reads, aggregates and multipart uploads were all exercised against CRS 4.29 and none of them
  tripped a rule, so no exclusion ships for them. Writes that carry HTML or code are the exception,
  see the sections above.
- `ALLOWED_METHODS` and `CORS_ALLOW_METHODS` include `SEARCH`, the method Directus 11 accepts for
  queries too long for a URL, with the query in a JSON body. A `SEARCH /items/<collection>` returned
  `200`; the same call with `<script>alert(1)</script>` in the query body was refused with `403`, so
  the body is inspected like any other. Remove `SEARCH` from both settings if no client uses it.
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
ALLOWED_METHODS=GET|POST|HEAD|PUT|DELETE|PATCH|OPTIONS|SEARCH
MAX_CLIENT_SIZE=1024m
MODSECURITY_REQ_BODY_NO_FILES_LIMIT=10485760
USE_MODSECURITY=yes
MODSECURITY_SEC_RULE_ENGINE=On
SERVE_FILES=no
USE_CORS=yes
CORS_ALLOW_ORIGIN=self
CORS_ALLOW_METHODS=GET, POST, HEAD, PUT, DELETE, PATCH, OPTIONS, SEARCH
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

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
