# Vaultwarden Template

Expose an existing Vaultwarden server through BunkerWeb, including encrypted vault writes,
imports, attachments and WebSocket sync notifications.

## Prerequisites

- Run Vaultwarden with persistent `/data` storage. BunkerWeb must reach
  `http://vaultwarden:80` on a private network; do not expose that port publicly.
- Set Vaultwarden's `DOMAIN=https://vaultwarden.example.com` to your actual public URL.
  See upstream [proxy examples](https://github.com/dani-garcia/vaultwarden/wiki/Proxy-examples).
  Modern Vaultwarden notifications use `/notifications/hub` on the **same port 80** as the
  web vault; no separate notification port or BunkerWeb proxy entry is needed.
- Point your public hostname at BunkerWeb. `AUTO_LETS_ENCRYPT=yes` makes BunkerWeb request
  its certificate (default HTTP challenge requires inbound port 80). For existing certificate
  files, use `AUTO_LETS_ENCRYPT=no`, `USE_CUSTOM_SSL=yes`, `CUSTOM_SSL_CERT` and
  `CUSTOM_SSL_KEY`, mounted so the scheduler can read them.

## Import and initial detect run

Import `template.json` and `configs/modsec/vaultwarden_false_positives.conf` using the
repository [installation guide](../../README.md#installing-templates), or its plugin bundle
layout. Assign `USE_TEMPLATE=vaultwarden`, update `SERVER_NAME` and `REVERSE_PROXY_HOST`
including port `80`, and review the guided steps. Keep the upstream URL without a trailing
slash or path.

Before enabling blocking on your instance:

1. Set service `SECURITY_MODE=detect` and reload.
2. Use the web vault and browser extension while reading BunkerWeb logs; exercise saves,
   imports, attachments and sync notifications.
3. Confirm the included exclusions match actual log targets; narrow or add targeted
   exceptions only where your traffic requires them.
4. Switch to `SECURITY_MODE=block` only after those checks and repeat your workflows.

False positives return `403` in block mode. The default bad-behavior status list includes
`400 401 403 404 405 429 444`: about ten matching responses in 60 seconds ban the client
for 86400 seconds (one day). Bulk-write `429` responses count too. This is why the detect
run matters. Detect mode still enforces nginx body limits. Use the explicit HTTPS URL when
testing a nonstandard public HTTPS port; the HTTP redirect omits that port.

## Settings and tradeoffs

| Setting | Purpose |
| --- | --- |
| `REVERSE_PROXY_URL=/`, `REVERSE_PROXY_WS=yes` | Proxy the vault and upgrade its notification hub through the same upstream. |
| `ALLOWED_METHODS=GET\|POST\|HEAD\|OPTIONS\|PUT\|DELETE` | Allow item/folder updates and deletes; defaults omit `PUT`/`DELETE`, causing `405`. `OPTIONS` is included but historically untested; no source flow used `QUERY`. |
| `MAX_CLIENT_SIZE=100m` | Permit attachments such as the source's 25 MB upload, rejected with nginx `413` at the default `10m`. |
| `MODSECURITY_REQ_BODY_NO_FILES_LIMIT=5m` | Permit JSON imports above the default 128 KiB; a 354 KB import otherwise got `400` with rule `200002`. |
| `LIMIT_REQ_RATE=20r/s` | Support scripted bulk writes that returned `429` at the default `2r/s`. |
| `SERVE_FILES=no` | Fetch content from Vaultwarden rather than local files. |

Both body allowances apply service-wide and increase potential buffering/memory use.
Keep lower limits or the default request rate if your clients do not need larger uploads,
imports or bulk writes. ModSecurity remains enabled.

## CRS exception scope and cost

`configs/modsec/vaultwarden_false_positives.conf` is a service-scoped **modsec** config:

- For **POST or PUT** to `/api/ciphers`, `/api/folders`, `/api/sends` or their descendants,
  `ctl:ruleRemoveTargetById` removes **the entire `ARGS` collection** from rules `930120`,
  `932125`, `932370`, `932380` and `941120`.
- For **POST only** to `/api/ciphers/<36 hex-or-hyphen characters>/attachment/<hex ID>`,
  it removes only `FILES:data` from rule `920120`.

Vaultwarden's EncStrings are base64 pieces joined by `|`; random ciphertext and encrypted
filenames can resemble file, command or script attacks. The broader `ARGS` exception is the
source-tested compromise: literal lists cannot cover arbitrary array indices, while regex
targets and `ARGS_POST` did not work with BunkerWeb 1.6.15.

**Cost:** those five rules also stop inspecting all query-string and form arguments on the
matching POST/PUT paths, not only encrypted JSON fields. Headers, cookies, other methods,
other paths and every other CRS rule remain active. `949110` is never excluded. The prefix
requires `/` or end-of-path, so `/api/ciphers-other` does not match. Do not make these
exceptions global or expand their method/path scope. Folders and Sends carry EncStrings,
but the source saw no hit there and did not stress-test those paths.

## Validation and test history

On 2026-11-03, the packaged template was imported through a local test plugin into BunkerWeb
`:tests` (1.6.18-rc2, libmodsecurity 3.0.17, CRS 4.30.0). Applied defaults matched the JSON and
nginx/ModSecurity loading passed. An echo-backend probe confirmed that `930120` no longer
flagged the item-write argument while independent RCE rule `932160` still blocked that
plaintext attack. On another path both rules fired. Encrypted vault traffic, other excluded
rules and attachment boundaries were not freshly exercised.

Check JSON with `jq . templates/vaultwarden/template.json`, then validate the import and
your own clients. JSON parsing does not prove runtime behavior.

Historical scripted tests: **Vaultwarden 1.37.4**, SQLite, BunkerWeb **1.6.15**,
**2026-10-06**, self-signed TLS. Scripts constructed master keys and EncStrings in the web
vault's format. They covered page/assets/config, registration/prelogin/login/token refresh,
profile/sync/folders, item CRUD, a 300-item 354 KB import, a 25 MB attachment upload/download
with matching SHA-256, WebSocket hub/update push, and admin login/users/config/diagnostics.
With the exception, 1,200 item creates (about 8,400 EncStrings) returned `200`.

Historical controls for each of the five `ARGS` rules blocked the same EncString on another
path, in a cookie or in a GET query. SQL injection in the excluded field still got `403`
from other rules. The attachment `920120` exception only had real-upload pass/fail proof,
**no negative boundary pair**. This extraction is **not a fresh full-app E2E run**.

Not verified: public Send access (app returned `404` for tested routes), SSO, two-factor,
mail, mobile and browser-extension clients, organizations/collections, icon downloads or
`OPTIONS`. Real browser behavior and those clients require deployment-specific validation.
