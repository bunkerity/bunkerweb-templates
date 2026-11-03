# authentik Template

Expose an existing authentik server through BunkerWeb, including its API, admin editors and
WebSockets. This configures authentik itself; applications using its proxy provider need their
own forward-auth configuration.

## Prerequisites

- Run authentik with its database and worker. BunkerWeb must reach the server at
  `http://authentik:9000`; keep that HTTP upstream on a private network.
- Mount persistent storage at `/data` for both server and worker before using file uploads.
  The historical test returned `500` without a file management backend.
- Point your public hostname at BunkerWeb. `AUTO_LETS_ENCRYPT=yes` requests a certificate;
  the default HTTP challenge needs inbound port 80. To retain existing certificates, use
  `AUTO_LETS_ENCRYPT=no`, `USE_CUSTOM_SSL=yes`, `CUSTOM_SSL_CERT` and `CUSTOM_SSL_KEY`;
  mount those files where the scheduler can read them.
- Configure authentik's `AUTHENTIK_LISTEN__TRUSTED_PROXY_CIDRS` for the network/address from
  which BunkerWeb connects. Trust only actual proxies. See the upstream
  [reverse proxy guide](https://docs.goauthentik.io/install-config/reverse-proxy/).

## Import

1. Follow the repository [installation guide](../../README.md#installing-templates).
   Import `template.json` and upload `configs/modsec/authentik_false_positives.conf` when
   prompted, or use the documented plugin bundle layout.
2. Assign `USE_TEMPLATE=authentik` to your service. Change `SERVER_NAME` and **both**
   `REVERSE_PROXY_HOST` and `REVERSE_PROXY_HOST_1` to your upstream, including port `9000`.
   Keep upstream URLs without a trailing slash or path.
3. Review the guided steps, reload, and test login, API writes, WebSockets and uploads.
   Start with service `SECURITY_MODE=detect`, inspect BunkerWeb logs, then switch to `block`
   after verifying your flows. Detect mode still enforces nginx upload limits.

## Settings and tradeoffs

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

`configs/modsec/authentik_false_positives.conf` is a service-scoped **modsec** config.
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

## Validation and test history

On 2026-11-03, the packaged template was imported through a local test plugin into BunkerWeb
`:tests` (1.6.18-rc2, libmodsecurity 3.0.17, CRS 4.30.0). Applied defaults matched the JSON and
nginx/ModSecurity loading passed. Against an echo backend, Python policy and HTML prompt
requests passed while another path or field remained blocked. This does not revalidate all
exclusions, cookie/CSP browser behavior or authentik itself.

Check JSON locally with `jq . templates/authentik/template.json`, then validate the import
and your flows on your own instance. JSON parsing is not runtime or browser proof.

Historical scripted tests: **authentik 2026.8.3**, PostgreSQL **16.15**, BunkerWeb **1.6.15**,
**2026-10-06**, using a self-signed certificate. They covered session login, bearer API calls,
CRUD/preflights, Python/CSS/HTML editors, both WebSocket routes, a 21 MiB upload using a
path-specific allowance, OIDC discovery/JWKS, authorization code, tokens, userinfo, refresh,
introspection and client credentials. Wrong-target and wrong-path exclusion controls returned
`403`. This template extraction is **not a fresh full-app E2E run**.

Not verified: real browsers (including the cookie/CSP changes above), SAML, LDAP/proxy outpost
flows, Kubernetes, multiple instances, and authentik's forwarded-client-IP trust. Test those
before relying on them. Repeated authentik `400`, `401`, `403`, `404` and rate-limit `429`
responses can trigger BunkerWeb's bad-behavior ban (default ten failures in 60 seconds,
24-hour ban); preserving app error bodies does not disable that accounting.
