<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/authentik.svg" alt="authentik logo" width="96" height="96" />
</p>

<h1 align="center">authentik</h1>

<p align="center">Identity provider template with flow and cookie-aware WAF tuning</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#crs-exception-scope">CRS exceptions</a> ·
  <a href="#validation">Validation</a>
</p>

Expose an existing authentik server through BunkerWeb, including its API, admin editors and
WebSockets. This configures authentik itself; applications using its proxy provider need their
own forward-auth configuration.

## At a glance

| Fact        | Value                         |
| ----------- | ----------------------------- |
| Template id | `authentik`                   |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4 |
| Configs     | 1 (modsec-crs)                |
| Rule ids    | `9851000` to `9851999`        |
| Upstream    | `http://authentik:9000`       |

## Prerequisites

- Run authentik with its database and worker (tested with authentik 2026.8.3). BunkerWeb must
  reach the server at `http://authentik:9000`; keep that HTTP upstream on a private network.
- Keep uploaded files (icons, brand logos) across container recreation, on both server and worker.
  authentik 2026.8.3 writes them to `/media`, and `/data/media` is a link to it, so a named Docker
  volume at `/data` does not keep them: Docker fills the volume with that link. Mount a volume at
  `/media` (for example `media:/media` in Compose). An empty host folder at `/data`, owned by user
  `1000`, also keeps them.
- Point your public hostname at BunkerWeb. For a public name, keep **Automatic Let's Encrypt** on:
  the default HTTP challenge needs inbound port 80. For a name that is not public, such as a lab
  host, switch it off. BunkerWeb 1.6.17 ignores that switch on the first save of a new service and
  keeps Let's Encrypt on, so save, open the service again in easy mode, switch it off again and
  save. To keep existing certificates, do that, then switch the service to **Advanced** mode, open
  the **Custom SSL certificate** plugin and set **Use custom certificate** (`USE_CUSTOM_SSL=yes`),
  **Certificate path** (`CUSTOM_SSL_CERT`) and **Key path** (`CUSTOM_SSL_KEY`). The guided steps
  do not show these fields. Mount the files where the scheduler can read them.
- Configure authentik's `AUTHENTIK_LISTEN__TRUSTED_PROXY_CIDRS` for the network/address from
  which BunkerWeb connects. Trust only actual proxies. See the upstream
  [reverse proxy guide](https://docs.goauthentik.io/install-config/reverse-proxy/).

## Install

1. Follow the repository [installation guide](../../README.md#install).
   Import `template.json` and upload `configs/modsec-crs/authentik_false_positives.conf` when
   prompted, or use the documented plugin bundle layout.
2. Create the service: **Services**, **New**, then pick **Authentik** in the **Templates** list
   above the steps. Change `SERVER_NAME` in step 1 and **both** `REVERSE_PROXY_HOST` and
   `REVERSE_PROXY_HOST_1` in step 2 to your upstream, including port `9000`. Keep upstream URLs
   without a trailing slash or path. The WebSocket field of the `/ws/` entry
   (`REVERSE_PROXY_WS_1`) is a text box in easy mode: keep `yes`.
3. Review the guided steps and save the service: BunkerWeb applies it, and nothing else needs a
   reload. Test login, API writes, WebSockets and uploads. A new service starts in `block` mode. To
   start in detect mode, switch the service to **Advanced** mode, open **General** and set
   **Security mode** to `detect`. Inspect the BunkerWeb logs, then set it back to `block` after
   verifying your flows. Detect mode still enforces nginx upload limits.

## Settings

| Setting | Purpose |
| --- | --- |
| `REVERSE_PROXY_URL=/` | Serve authentik at the public hostname root. |
| `REVERSE_PROXY_URL_1=/ws/`, `REVERSE_PROXY_WS_1=yes` | Upgrade `/ws/client/` and `/ws/outpost/<id>/` on the same upstream; without this the historical test got `200` instead of `101`. |
| `REVERSE_PROXY_INTERCEPT_ERRORS=no` | Keep authentik's JSON API/OAuth errors; the default replaced `400`/`403` bodies with BunkerWeb HTML. |
| `ALLOWED_METHODS=GET\|POST\|HEAD\|OPTIONS\|PUT\|PATCH\|DELETE` | Permit API edits, deletes and preflights that otherwise get `405`. `QUERY` is omitted because no source flow used it. |
| `MAX_CLIENT_SIZE=64m` | Carry the source upload allowance; the default `10m` returns `413` above 10 MiB, even in detect mode. This template raises the limit for the whole service. |
| `LIMIT_REQ_RATE=50r/s` | Avoid the static/login burst `429` responses seen at the default `2r/s`; rate limiting remains enabled. |
| `SERVE_FILES=no` | Route content to authentik rather than local files. |
| `COOKIE_FLAGS="* SameSite=Lax"` | Apply SameSite without making the JavaScript-readable CSRF cookie HttpOnly. |
| `CONTENT_SECURITY_POLICY=""` | Let authentik supply its own CSP, as recommended by the stack's integration documentation. |

The [authentik browser API client](https://github.com/goauthentik/authentik/blob/version/2026.8.3/web/src/common/api/middleware.ts)
reads `authentik_csrf` to populate `X-authentik-CSRF`. Adding `HttpOnly` to that cookie prevents
that read. The template retains the stack's wildcard `SameSite=Lax` policy without adding
`HttpOnly`; authentik remains responsible for marking its sensitive cookies HttpOnly.
BunkerWeb's default `COOKIE_AUTO_SECURE_FLAG=yes` still adds `Secure` on HTTPS.
Clear cookies from any earlier proxy configuration before browser validation.

An empty BunkerWeb CSP avoids imposing its default `form-action 'self'` on flows that post to
another origin. BunkerWeb's default `KEEP_UPSTREAM_HEADERS` preserves upstream CSP, including
authentik's restrictions on uploaded content. If authentik sends no CSP, this template adds none;
check the response headers after import. See authentik's
[header guidance](https://docs.goauthentik.io/security/security-hardening/#http-headers).
Cookie/CSP compatibility still needs **browser validation** on your installed version.

The larger upload allowance increases potential request buffering and memory use. If only
`/api/v3/admin/file/` needs it, restore service `MAX_CLIENT_SIZE=10m` and add a separate proxy
entry for that path with the same upstream and `REVERSE_PROXY_MAX_CLIENT_SIZE_N=64m`.
Do not change the `/ws/` entry when doing so.

## CRS exception scope

`configs/modsec-crs/authentik_false_positives.conf` is a service-scoped **modsec-crs** config.
Its phase 1 chains match only `POST`, `PUT` or `PATCH` to the following collection or UUID
detail endpoints (each ending in `/`):

| Endpoint | Literal target | Rules skipped for that target |
| --- | --- | --- |
| `/api/v3/policies/expression/` | `ARGS:json.expression` | `932140`, `933160`, `934100`, `941390`, `942100`, `942190` |
| `/api/v3/core/brands/` | `ARGS:json.branding_custom_css` | `941100`, `941130`, `941160`, `941170` |
| `/api/v3/stages/prompt/prompts/` | `ARGS:json.placeholder` | `941100`, `941120`, `941160`, `941390` |
| Same prompt endpoint | `ARGS:json.sub_text` | `941160` |

These fields intentionally contain Python, CSS or HTML. The listed rules lose inspection of
those fields on matching writes; other fields, paths, methods and CRS rules remain inspected.
Literal `ARGS:json.*` targets also cover query/form arguments with that exact name, not only
JSON fields. Rule `949110` remains active. Do not apply these configs globally.

The exception does not cover every rule that can match these fields. A literal `<script>` tag
in a policy expression or prompt text, or a `javascript:` URL in brand CSS, is still denied with
`403`. Realistic CSS (imports, gradients, `data:` images), HTML help text and Python policies
pass. To turn one exception off, open step 4 of the service in easy mode and delete that rule
block in the editor.

## Validation

On 2026-11-03, the packaged template was imported through a local test plugin into BunkerWeb
`:tests` (1.6.18-rc2, libmodsecurity 3.0.17, CRS 4.30.0). Applied defaults matched the JSON and
nginx/ModSecurity loading passed. Against an echo backend, Python policy and HTML prompt
requests passed while another path or field remained blocked. This does not revalidate all
exclusions, cookie/CSP browser behavior or authentik itself.

> [!NOTE]
> Check JSON locally with `jq . templates/authentik/template.json`, then validate the import
> and your flows on your own instance. JSON parsing is not runtime or browser proof.

Historical scripted tests: **authentik 2026.8.3**, PostgreSQL **16.15**, BunkerWeb **1.6.15**,
**2026-10-06**, using a self-signed certificate. They covered session login, bearer API calls,
CRUD/preflights, Python/CSS/HTML editors, both WebSocket routes, a 21 MiB upload using a
path-specific allowance, OIDC discovery/JWKS, authorization code, tokens, userinfo, refresh,
introspection and client credentials. Wrong-target and wrong-path exclusion controls returned
`403`.

On 2026-11-20 and 2026-11-23 the template was imported through the web UI of BunkerWeb 1.6.17
(all-in-one, CRS 4.29, the web UI protected by its own CRS), and a service was created in easy mode
for authentik 2026.8.3 with PostgreSQL 16. In Chrome, login, the admin interface, API writes with
Python, CSS and HTML in the documented fields, both WebSocket routes, a 21 MiB upload and the cookie
flags behaved as described. Uploads survived recreating the containers with a `/media` volume or
an empty host folder at `/data`, and did not survive with a named volume at `/data`.

Not verified: SAML, LDAP/proxy outpost flows, Kubernetes, multiple instances, a real outpost
connection, a custom certificate, and authentik's forwarded-client-IP trust. Test those before
relying on them. Repeated authentik `400`, `401`, `403`, `404` and rate-limit `429`
responses can trigger BunkerWeb's bad-behavior ban (default ten failures in 60 seconds,
24-hour ban); preserving app error bodies does not disable that accounting.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
