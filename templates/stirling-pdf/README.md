<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/stirling-pdf.svg" alt="Stirling-PDF logo" width="96" height="96" />
</p>

<h1 align="center">Stirling-PDF</h1>

<p align="center">PDF toolbox template with big uploads, long jobs and tuned WAF rules</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Stirling_PDF-3.4.0-248651?labelColor=1d6f93&style=flat" alt="Stirling-PDF 3.4.0" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#configure-stirling-pdf-for-the-proxy">Configure Stirling-PDF</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#sign-in-with-authentik-optional">authentik</a> ·
  <a href="#crs-exception-scope-and-cost">CRS exceptions</a> ·
  <a href="#validation">Validation</a> ·
  <a href="#known-limits">Known limits</a>
</p>

Put an existing [Stirling-PDF](https://github.com/Stirling-Tools/Stirling-PDF) server behind
BunkerWeb with ModSecurity and the OWASP CRS on: merge, split, compress, convert, OCR, redact,
sign, the Automate pipeline, saved signatures, server-side storage and the phone scanner page.
The template raises the upload size and the job timeout, lets the web app send one request per
file, and removes the few CRS rules that block normal use.

## At a glance

| Fact        | Value                                                |
| ----------- | ---------------------------------------------------- |
| Template id | `stirling-pdf`                                       |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Stirling-PDF 3.4.0    |
| Configs     | 1 (modsec-crs)                                       |
| Rule ids    | `9880000` to `9880999`                               |
| Upstream    | `http://stirling-pdf:8080`                           |

## Prerequisites

- Run Stirling-PDF `3.4.0` or a close release (tested image `stirlingtools/stirling-pdf:3.4.0-fat`).
  BunkerWeb must reach it at `http://stirling-pdf:8080` on a private network. Do not publish port
  8080: the app trusts forwarded headers from any sender (see the security note below).
- Give the Stirling-PDF container 3 GB of memory or more. The tests ran with 3 GB.
- Give BunkerWeb about 8 times `MAX_CLIENT_SIZE` of memory on top of its normal use (see
  [Settings](#settings)). The separate `bunkerweb` container idles near 190 MiB, so the default
  `100m` fits in 1.5 GB there. The all-in-one image (scheduler, web UI, Redis and nginx in one
  container) idles near 1.1 GiB by itself: with `100m` it needs about 2 GB (a 100 MB upload peaked
  at about 1.8 GiB). A container that is too small does not answer `413`: the nginx worker is
  killed and the client sees a dropped connection (`curl: (52) Empty reply from server`). Lower
  `MAX_CLIENT_SIZE` instead of guessing.
- Point your public hostname at BunkerWeb. The template turns on Let's Encrypt, so BunkerWeb
  requests the certificate and the default HTTP challenge needs inbound port 80. For your own
  certificate, or a name that is not public, see [Known limits](#known-limits).
- Serve Stirling-PDF at the root of the hostname. The template proxies `/` and was not tested
  under a sub-path.

## Configure Stirling-PDF for the proxy

Set these on the Stirling-PDF container. The measured rows were reproduced on `3.4.0`; the others
come from reading the `3.4.0` source and settings template.

| Stirling-PDF setting                                  | Value                                      | Without it                                                                                                                                                                                                                                                  |
| ----------------------------------------------------- | ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SYSTEM_FRONTENDURL`                                  | `https://stirling-pdf.example.com`         | Invite e-mails, share links and the phone scanner QR code use the wrong address (source). The template step "Domains and TLS" holds the host name only, without `https://`.                                                                                    |
| `SECURITY_ENABLELOGIN`                                | `true` (the default)                       | With `false` every route is open, including the API documentation. Use `false` only behind another sign-in (see [authentik](#sign-in-with-authentik-optional)). Both modes ran through the template.                                                      |
| `SECURITY_INITIALLOGIN_USERNAME` and `_PASSWORD`      | an account name and a long password        | Without both, the first start creates `admin` with the password `stirling` (source). Set them before the first start.                                                                                                                                     |
| `SYSTEM_CORSALLOWEDORIGINS`                           | `https://stirling-pdf.example.com`         | Empty means every origin may call the API with credentials (source). Set it to your public address. A non-empty list also blocks the desktop app: add `tauri://localhost`, `http://tauri.localhost` and `https://tauri.localhost`, or its preflight answers `403 Invalid CORS request`. The app reads the list at start, so restart it after a change (measured). |
| `SYSTEM_ENABLEURLTOPDF`                               | `false` (the default)                      | The URL to PDF route stays off; with it on the app fetches addresses the client chooses. In the tests the route answered a redirect to `/error` while off.                                                                                                |

Security: Stirling-PDF always believes `X-Forwarded-For` and `Forwarded` from whoever connects
(Jetty, no trusted-proxy list). Straight to port 8080, a login with `X-Forwarded-For: 6.6.6.6`
was logged as coming from `6.6.6.6`, and a `Forwarded: for=7.7.7.7` header won over
`X-Forwarded-For`. Let only BunkerWeb reach the port. Through BunkerWeb the template replaces both
headers (see `REVERSE_PROXY_HEADERS` in [Settings](#settings)).

## Install

1. Import the template. In the web UI use Templates, Create template, Raw, and upload
   `template.json`; the dialog then lists one missing file, `modsec_crs/stirling-pdf_false_positives.conf`,
   so upload `configs/modsec-crs/stirling-pdf_false_positives.conf` there.
   For both import methods (web UI and plugin), see the repository
   [installation guide](../../README.md#install).
2. Create the service: Services, Create new service, Templates, `Stirling-pdf`, Easy mode.
3. Step 1: enter your public hostname in `SERVER_NAME`, for example `pdf.example.com`: the name
   only, with no `https://`. Stirling-PDF's `SYSTEM_FRONTENDURL` takes the full address,
   `https://pdf.example.com`. Keep `AUTO_LETS_ENCRYPT` on for a public site. For a name that is not
   public, see [Known limits](#known-limits).
4. Step 2: change `REVERSE_PROXY_HOST` only if your container is not named `stirling-pdf`. Keep the
   address without a path or trailing slash.
5. Steps 3 to 10: the defaults are tuned (see [Settings](#settings)). Lower `MAX_CLIENT_SIZE` if
   BunkerWeb has less memory. Step 10 (phone camera) lets the scanner page use the camera; to
   opt out, see the `PERMISSIONS_POLICY` row in [Settings](#settings).
6. Step 11 (authentik sign-in): leave it empty to skip it.
7. Step 12 only shows the CRS exclusions. Save: saving the service applies it, with no separate
   reload. Open `https://<your hostname>` and sign in with the Stirling-PDF account.

> [!WARNING]
> BunkerWeb 1.6.17's web UI answers `403` when you import this template, and when you create or edit
> a service that uses it, if the web UI is itself protected by BunkerWeb's CRS (the setup wizard
> default). The long `PERMISSIONS_POLICY` value trips the CRS (rules `941370` and `942151`). Until a
> BunkerWeb release fixes it, set the web UI service's Security mode to `detect` (Advanced mode,
> General) for the import and for each save of this service, then set it back to `block`.

Before you turn on blocking on a live instance, set the service's Security mode to `detect`
(Advanced or Raw mode, General; easy mode has no field for it), use the tools you rely on (OCR,
sign, Automate, your own file names), read the BunkerWeb log, then switch to `block`.

> [!IMPORTANT]
> False positives answer `403` in block mode. `403` is in the bad-behavior list, so repeated blocked
> requests ban the client (`BAD_BEHAVIOR_THRESHOLD` matches in a minute, ten by default).

## Settings

Each setting was put back to the BunkerWeb default and the failure below reproduced on the real
app.

| Setting                                                   | Purpose and measurement                                                                                                                                                                                                                                                                                                                           |
| --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `REVERSE_PROXY_URL=/`, `REVERSE_PROXY_HOST=http://stirling-pdf:8080` | Proxy the app. Stirling-PDF `3.4.0` uses plain HTTP request and response pairs: no WebSocket or event-stream route was found, so no `REVERSE_PROXY_WS`.                                                                                                                                                                      |
| `MAX_CLIENT_SIZE=100m`                                    | Uploads. At the default `10m` an 11 MiB PDF got `413` (a 9 MiB one passed). With `100m` a 104,850,000 byte file passed and a 105,906,176 byte one got `413`, not `403` or a reset. Memory: a 100 MiB upload raised BunkerWeb from about 190 MiB to a peak of about 1.1 GiB (50 MiB: about 650 MiB), roughly 8 times the upload. A 200 MiB upload on a container limited to 1.5 GB was killed by the kernel and the connection reset, so size the container before you raise the value. |
| `REVERSE_PROXY_READ_TIMEOUT=20m`                          | Stirling-PDF ends a synchronous job after 20 minutes. OCR of a 2000-page scan took 189 seconds straight to the app and 256 seconds through BunkerWeb; with the default `60s` BunkerWeb answered `504` after 60 seconds.                                                                                                                          |
| `LIMIT_REQ_RATE=100r/s`                                   | The web app sends one request per file, all at the same moment, to the same address. At the default `2r/s` rotating 11 files gave 2 successes and 9 `429` answers (the page showed 9 errors, and `429` counts toward a ban). With `100r/s` 40 files all went through. One address shared by many people that run batches together can exceed it. |
| `ALLOWED_METHODS=GET\|POST\|HEAD\|OPTIONS\|PUT\|PATCH\|DELETE` | Server-side storage, saved signatures and folder changes use `PUT`, `PATCH` and `DELETE`, and the desktop app sends `OPTIONS`. The default list answered `405` to all of them. Narrow the list if you turn storage off.                                                                                                                  |
| `REVERSE_PROXY_INTERCEPT_ERRORS=no`                       | Keep Stirling-PDF's own error messages (for example "pageNumbers must list the pages to remove" or "Invalid username or password"). With `yes` BunkerWeb swapped each upstream `400`, `401` and `404` for its HTML page, and the web app and scripts lost the message.                                                                       |
| `REVERSE_PROXY_HEADERS=X-Forwarded-For $remote_addr;Forwarded ""` | BunkerWeb's default appends to a client-sent `X-Forwarded-For`, and `Forwarded` passes through. With the default, forged logins were logged as `6.6.6.6` and `7.7.7.7`; with the setting all five forged logins were logged as the real client address.                                                                                      |
| `BAD_BEHAVIOR_STATUS_CODES=403 405 429 444`               | Stirling-PDF answers `401` to every page load without a session (and `400` to bad input). With the default list, eleven `401` answers (expired-session reloads or wrong passwords) banned the address: the next request got `403`. With this list sixteen `401` answers left it unbanned. The app locks an account after five failed logins for 120 minutes, so sign-in guessing stays limited. |
| `PERMISSIONS_POLICY` (default list with `camera=(self)`)  | The phone scanner page asks for the camera. BunkerWeb's default sends `camera=()`, and the page could not use it (`allowsFeature(camera)` was `false`; with the setting it was `true`). If you never use the scanner, open the step "Security - Phone camera" and change the entry `camera=(self)` to `camera=()` (type the value: clearing the field keeps the template value). That is BunkerWeb's default for the camera. The default list also blocks `ch-ua-high-entropy-values`, so the browser console shows a harmless `Permissions policy violation` line from Stirling-PDF's start page.                                                         |

Measured and left at the default: the BunkerWeb ModSecurity body limits (a 100 MiB upload passes
with them as they are) and the connect and send timeouts. Large JSON bodies sent by the text editor
were not measured (see [Known limits](#known-limits)).

> [!NOTE]
> curl and some scripts send `Expect: 100-continue` on bodies above 1 MB, which CRS rule `920450`
> blocks. Browsers do not send it. Add `-H 'Expect:'` to API calls with large bodies.

## Sign in with authentik (optional)

Step 11 puts an authentik forward-auth sign-in in front of Stirling-PDF, so only authentik users
reach the app. Skip it if Stirling-PDF's own accounts are enough.

In authentik, create an application with a **Proxy Provider** in "Forward auth (single
application)" mode, set its external host to the public address (for example
`https://stirling-pdf.example.com`) and assign it to an outpost. BunkerWeb must reach the outpost
(the tests used `http://outpost:9000`). Then fill these fields in step 11:

| Field (setting)                                                  | Value                                                                                       |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `REVERSE_PROXY_HOST_1`                                           | `http://outpost:9000` (the outpost address)                                                 |
| `REVERSE_PROXY_URL_1`                                            | keep `/outpost.goauthentik.io`                                                              |
| `REVERSE_PROXY_HEADERS_1`                                        | keep the value: it passes the public host and the original URL to the outpost               |
| `REVERSE_PROXY_PASS_REQUEST_BODY_1`                              | keep `no`: uploads stay out of the check request                                            |
| `REVERSE_PROXY_AUTH_REQUEST`                                     | `/outpost.goauthentik.io/auth/nginx`                                                        |
| `REVERSE_PROXY_AUTH_REQUEST_SIGNIN_URL`                          | `$scheme://$http_host/outpost.goauthentik.io/start?rd=$scheme://$http_host$request_uri`     |

The easy-mode form shows the first four fields without a label, only the setting name. In the
tests a browser was sent to the authentik sign-in, came back to Stirling-PDF, signed in with the
app account and rotated three files; with the three fields empty the app was reachable directly.
Scripts and health probes that call the public address with `X-API-KEY` are redirected to the
sign-in too (a `302`), so point them at the app's private address instead.

## CRS exception scope and cost

`configs/modsec-crs/stirling-pdf_false_positives.conf` is a service-scoped **modsec-crs** config
with three rules. Each one names an exact path (or the `/api/v1/` prefix) and the method.

| Rule      | Path and method                                | What it removes                                                                                                                                  |
| --------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `9880100` | `POST /api/v1/pipeline/handleData`             | CRS `942550` for the form field `json` (the Automate pipeline definition, sent as JSON text with placeholders such as `{outputFolder}`).         |
| `9880110` | `POST /api/v1/proprietary/signatures`          | CRS `932235`, `932260`, `934170`, `941130` and `941170` for the JSON field `dataUrl` (the `data:image/png;base64,...` signature image).          |
| `9880120` | `POST` or `PUT` under `/api/v1/`               | CRS `920120` for the names of the upload parts (`fileInput`, `file`, `files`, `attachments`, `imageFile` and so on): file names with `'`, `;`, `=`. |

Cost: the same text in another field, path or method is still blocked (each rule has a control
that returns `403`). The file-name rule covers every upload route, so a quote in a file name is
checked by the app and not the WAF on those routes. To narrow rule `9880120` to the routes you
use, edit the path in your copy of `stirling-pdf_false_positives.conf` before you upload it in the
import dialog: the config of a service created from the template is read-only in the web UI.

## Files

- `template.json` - BunkerWeb template definition: settings, configs and the guided steps.
- `configs/modsec-crs/stirling-pdf_false_positives.conf` - the three CRS exclusions above.

## Validation

Run `jq . template.json` to check the JSON before importing.

Everything ran against the real app (`stirlingtools/stirling-pdf:3.4.0-fat`) behind BunkerWeb
`1.6.17` with CRS `4.29`, one request at a time, with the log read back after each run.

| Check                                 | Result                                                                                                                                                                                          |
| ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Request matrix with the template      | 136 valid requests (page tools, conversions, OCR, redaction, signing, passwords, Automate, signatures, storage, file names with accents, quotes and `;`, mobile scanner, async jobs, `PUT`, `PATCH`, `DELETE`, `OPTIONS`) passed with the expected status and result. |
| Same matrix without the exclusions    | The requests behind the three rules answered `403` with the CRS rules named above.                                                                                                             |
| Controls                              | 20 requests with the same text in another field, on another path or with another method, plus a line-feed form of each path, all answered `403`.                                               |
| Web UI, easy mode, clean BunkerWeb    | Service created from the template with nothing else set, then three files rotated through the browser: all `200`. Repeated with step 11 filled in: authentik sign-in, app sign-in, three files rotated. |
| Upload edge and settings              | Numbers in [Settings](#settings).                                                                                                                                                               |

Not part of the request matrix, because ModSecurity does not read file contents: the bytes of the
PDFs and office files you upload.

Rerun the matrix after a Stirling-PDF upgrade, since the exclusions name fields and paths of
`3.4.0`.

## Known limits

- **Code-like text in a few free-text fields is blocked.** Ordinary text passes (names with `'`,
  `&` and `;`, URLs, brackets), but a watermark or stamp text, a comment, a custom metadata value or
  a plain-text redaction list that holds script or SQL-like code answers `403` (CRS `941100` and
  related rules). No exclusion ships for them: that text looks like an attack. To allow one field,
  open Configs, Create Custom Config, set Service to your service and Config Type to `MODSEC_CRS`,
  paste a copy of one rule from `stirling-pdf_false_positives.conf` with a new id, your path and your
  field name, then save.
- **URL to PDF stays blocked.** The route is off in the app and the template does not open it.
- **A name that is not public (a lab, an internal host).** Step 1 ships `AUTO_LETS_ENCRYPT=yes`,
  and in easy mode switching it off at service creation does not stick: the service keeps `yes`.
  Create the service, then open it again: Services, the service, Advanced mode. In the tab
  "Let's Encrypt" switch off "Automatic Let's Encrypt" and save. On plain HTTP also switch off
  "Auto redirect HTTP to HTTPS" in the tab "SSL"; without it every request gets a `301` to
  `https://<host>/`, which fails with no certificate or on another port.
- **Your own certificate.** No easy-mode step has it. Open the service in Advanced mode, switch
  off "Automatic Let's Encrypt" (tab "Let's Encrypt"), then in the tab "Custom SSL certificate" set
  `USE_CUSTOM_SSL=yes` with `CUSTOM_SSL_CERT` and `CUSTOM_SSL_KEY`.
- **Not verified:** single sign-on (OIDC, SAML), invite e-mails, the MCP and AI features, the
  desktop app against the proxy, the timestamp server, a real phone camera, the large JSON the text
  editor sends (ModSecurity's no-files body limit is `128 KiB` by default), several people sharing
  one address, a sub-path, and releases other than `3.4.0`. Teams and other licence features were
  not run.
- **Tested once:** a 200 MiB upload on a 1.5 GB container (see [Settings](#settings)).

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
