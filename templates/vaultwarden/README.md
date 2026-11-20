<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/vaultwarden-light.svg">
    <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/vaultwarden.svg" alt="Vaultwarden logo" width="96" height="96" />
  </picture>
</p>

<h1 align="center">Vaultwarden</h1>

<p align="center">Password vault template with encrypted-field WAF tuning</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#crs-exception-scope-and-cost">CRS exceptions</a> ·
  <a href="#validation">Validation</a>
</p>

Expose an existing Vaultwarden server through BunkerWeb, including encrypted vault writes,
imports, attachments and WebSocket sync notifications.

## At a glance

| Fact        | Value                         |
| ----------- | ----------------------------- |
| Template id | `vaultwarden`                 |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4 |
| Configs     | 1 (modsec-crs)                |
| Rule ids    | `9877000` to `9877999`        |
| Upstream    | `http://vaultwarden:80`       |

## Prerequisites

- Run Vaultwarden (tested with 1.37.4) with persistent `/data` storage. BunkerWeb must reach
  `http://vaultwarden:80` on a private network; do not expose that port publicly.
- Set Vaultwarden's `DOMAIN=https://vaultwarden.example.com` to your actual public URL.
  See upstream [proxy examples](https://github.com/dani-garcia/vaultwarden/wiki/Proxy-examples).
  Modern Vaultwarden notifications use `/notifications/hub` on the **same port 80** as the
  web vault; no separate notification port or BunkerWeb proxy entry is needed.
- Test over HTTPS. The web vault and the apps refuse to sign up or log in over plain HTTP
  (`Insecure URL not allowed`), so a lab needs BunkerWeb's self-signed certificate at least.
- Point your public hostname at BunkerWeb. `AUTO_LETS_ENCRYPT=yes` makes BunkerWeb request
  its certificate (default HTTP challenge requires inbound port 80). For a name that is not
  public, switch Let's Encrypt off in step 1. For an existing certificate, switch it off, then
  open the service in Advanced mode, tab **Custom SSL certificate**, switch on **Use custom
  certificate** and give the certificate and key (paths the scheduler can read, or pasted data).

## Install

Import `template.json` and `configs/modsec-crs/vaultwarden_false_positives.conf` using the
repository [installation guide](../../README.md#install), or its plugin bundle
layout. Pick **Vaultwarden** in the **Templates** list of the new service, update `SERVER_NAME`
and `REVERSE_PROXY_HOST` including port `80`, and review the guided steps. Keep the upstream
URL without a trailing slash or path. The Missing Custom Configs dialog names the folder
`modsec_crs`: upload `vaultwarden_false_positives.conf` there anyway.

Before enabling blocking on your instance:

1. Open the service in Advanced mode, tab **General**, set **Security mode** to `detect` and
   save the service (saving applies it).
2. Use the web vault and browser extension while reading BunkerWeb logs; exercise saves,
   attachments and sync notifications, and import a large file several times. The data is
   random, so a rule can match on one import and not on the next: one clean import proves
   nothing.
3. Confirm the included exclusions match actual log targets; narrow or add targeted
   exceptions only where your traffic requires them. To add one in the web UI, open
   **Configs**, **Create Custom Config**, pick the service, set the type to `MODSEC_CRS`, name
   it and paste the rule.
4. Switch **Security mode** back to `block` only after those checks and repeat your workflows.

False positives return `403` in block mode. The default bad-behavior status list includes
`400 401 403 404 405 429 444`: about ten matching responses in 60 seconds ban the client
for 86400 seconds (one day). Bulk-write `429` responses count too. This is why the detect
run matters. Detect mode still enforces nginx body limits. Use the explicit HTTPS URL when
testing a nonstandard public HTTPS port; the HTTP redirect omits that port.

## Settings

| Setting | Purpose |
| --- | --- |
| `REVERSE_PROXY_URL=/`, `REVERSE_PROXY_WS=yes` | Proxy the vault and upgrade its notification hub through the same upstream. |
| `ALLOWED_METHODS=GET\|POST\|HEAD\|OPTIONS\|PUT\|DELETE` | Allow item/folder updates and deletes; defaults omit `PUT`/`DELETE`, causing `405`. `OPTIONS` is included but historically untested; no source flow used `QUERY`. In easy mode the list is folded: `DELETE`, the sixth value, is behind **See More**. |
| `MAX_CLIENT_SIZE=100m` | Permit attachments such as the source's 25 MB upload, rejected with nginx `413` at the default `10m`. |
| `MODSECURITY_REQ_BODY_NO_FILES_LIMIT=5m` | Permit JSON imports above the default 128 KiB; a 354 KB import otherwise got `400` with rule `200002`. |
| `LIMIT_REQ_RATE=20r/s` | Support scripted bulk writes that returned `429` at the default `2r/s`. |
| `SERVE_FILES=no` | Fetch content from Vaultwarden rather than local files. |

> [!NOTE]
> Both body allowances apply service-wide and increase potential buffering/memory use.
> Keep lower limits or the default request rate if your clients do not need larger uploads,
> imports or bulk writes. ModSecurity remains enabled.

## CRS exception scope and cost

`configs/modsec-crs/vaultwarden_false_positives.conf` is a service-scoped **modsec-crs** config:

- For **POST or PUT** to `/api/ciphers`, `/api/folders`, `/api/sends` or their descendants,
  `ctl:ruleRemoveTargetById` removes **the entire `ARGS` collection** from eleven rules:
  `930120`, `932125`, `932160`, `932230`, `932235`, `932250`, `932260`, `932370`, `932380`,
  `941120` and `942100`.
- For **POST only** to `/api/ciphers/<36 hex-or-hyphen characters>/attachment/<hex ID>`,
  it removes only `FILES:data` from rule `920120`.

Vaultwarden's EncStrings are base64 pieces joined by `|`, and the client encrypts every field
before it sends it. The CRS can only read ciphertext there, and random ciphertext and
encrypted filenames can resemble file, command or script attacks: the `|` followed by a few
random letters is enough for the shell rules. The eleven rules are the ones that matched random
EncStrings in bulk imports on BunkerWeb 1.6.17, and a 300-item import carries about two thousand
strings, so each import is another draw. A literal list cannot cover arbitrary array indices,
and regex targets and `ARGS_POST` do not work in a `ctl` action on BunkerWeb 1.6.17.

**Cost:** on those writes the eleven rules stop inspecting all query-string, form and JSON
arguments, not only the encrypted fields. An attack hidden in such an argument is then seen only
by the other CRS rules. Those still inspect the arguments, headers, cookies, other methods and
other paths. `949110` is never excluded. The prefix requires `/` or end-of-path, so
`/api/ciphers-other` does not match. Argument names stay inspected too, since only `ARGS` is
removed. Do not make these exceptions global or expand their method/path scope. Folders and
Sends carry EncStrings, but the source saw no hit there and did not stress-test those paths.

A shorter list is not an option: with only the original five rules, about one bulk import in
four got `403`. Removing the `attack-rce` family from `ARGS` by tag also gave no `403` in the
same test, but it takes all 25 paranoia-level-1 command-injection rules off those arguments
instead of the nine in the list, so the list is the narrower choice.

## Validation

On 2026-11-03, the packaged template was imported through a local test plugin into BunkerWeb
`:tests` (1.6.18-rc2, libmodsecurity 3.0.17, CRS 4.30.0). Applied defaults matched the JSON and
nginx/ModSecurity loading passed. An echo-backend probe confirmed that `930120` no longer
flagged the item-write argument while independent RCE rule `932160` still blocked that
plaintext attack. On another path both rules fired. Encrypted vault traffic, other excluded
rules and attachment boundaries were not freshly exercised.

On 2026-11-20, BunkerWeb 1.6.17 (all-in-one, CRS 4.29.0, libmodsecurity 3.0.17) in front of
Vaultwarden 1.37.4 over its self-signed HTTPS, `block` mode, service created in easy mode from
the packaged template, template imported through the web UI while it was protected by the CRS.
Random EncStrings of the web vault's format, in 300-item bulk imports (about 2,100 strings each):
with the original five-rule list 27 of 100 imports got `403`; with the eleven rules in this
file none of 100 did (7 got `502` from nginx after the WAF let them through, because
Vaultwarden closes the connection on an unauthenticated body). 400 single item creates, edits,
folder and Send writes got no `403` with either list. One 300-item import through the real
web vault in `block` mode returned `200`. The same attack text that the eleven rules skip in an
argument of an item write still got `403` in a JSON key, in a cookie, in a query on a `GET`
or `DELETE`, in the `Referer` header, and on any path outside the item, folder and Send writes.
Rule `932340`, which is not in the list, still denied command text in the same argument.

Check JSON with `jq . templates/vaultwarden/template.json`, then validate the import and
your own clients. JSON parsing does not prove runtime behavior.

Historical scripted tests: **Vaultwarden 1.37.4**, SQLite, BunkerWeb **1.6.15**,
**2026-10-06**, self-signed TLS. Scripts constructed master keys and EncStrings in the web
vault's format. They covered page/assets/config, registration/prelogin/login/token refresh,
profile/sync/folders, item CRUD, a 300-item 354 KB import, a 25 MB attachment upload/download
with matching SHA-256, WebSocket hub/update push, and admin login/users/config/diagnostics.
With the exception, 1,200 item creates (about 8,400 EncStrings) returned `200`.

Historical controls for each of the original five `ARGS` rules blocked the same EncString on another
path, in a cookie or in a GET query. SQL injection in the excluded field still got `403`
from other rules. The attachment `920120` exception only had real-upload pass/fail proof,
**no negative boundary pair**. This extraction is **not a fresh full-app E2E run**.

Not verified: public Send access (app returned `404` for tested routes), SSO, two-factor,
mail, mobile and browser-extension clients, organizations/collections, icon downloads or
`OPTIONS`. Real browser behavior and those clients require deployment-specific validation.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
