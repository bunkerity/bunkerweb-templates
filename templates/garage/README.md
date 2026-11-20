<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/garage.svg" alt="Garage logo" width="96" height="96" />
</p>

<h1 align="center">Garage</h1>

<p align="center">S3 object storage template with signed-request WAF tuning</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Garage-v2.5.0-248651?labelColor=1d6f93&style=flat" alt="Garage v2.5.0" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#configure-garage-for-the-proxy">Configure Garage</a> ·
  <a href="#import-and-initial-detect-run">Import</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#crs-exception-scope-and-cost">CRS exceptions</a> ·
  <a href="#things-to-think-about">Things to think about</a> ·
  <a href="#validation">Validation</a>
</p>

Expose the S3 API of an existing [Garage](https://garagehq.deuxfleurs.fr) server through
BunkerWeb: signed uploads and downloads, multipart uploads, bulk deletes, presigned URLs, browser
uploads with CORS, in path-style and virtual-host-style addressing, plus a token-free `/health`
endpoint for uptime monitors.

## At a glance

| Fact        | Value                                        |
| ----------- | -------------------------------------------- |
| Template id | `garage`                                     |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Garage v2.5.0 |
| Configs     | 1 (modsec-crs)                               |
| Rule ids    | `9857000` to `9857999`                       |
| Upstream    | `http://garage:3900`                         |

## Prerequisites

- Garage `v2.5.0` or compatible. BunkerWeb must reach the S3 API at `http://garage:3900` on a private
  network. Never publish Garage's RPC port (`3901`) through BunkerWeb or anywhere else. The admin
  port (`3903`) is reachable through the template for one path only, `/health` (see "Things to
  think about"). The template proxies nothing else of it.
- Point your S3 hostname (for example `s3.example.com`) at BunkerWeb. `AUTO_LETS_ENCRYPT=yes` makes
  BunkerWeb request its certificate (default HTTP challenge requires inbound port 80). For existing
  certificate files, use `AUTO_LETS_ENCRYPT=no`, `USE_CUSTOM_SSL=yes`, `CUSTOM_SSL_CERT` and
  `CUSTOM_SSL_KEY`.
- Set Garage's `[s3_api]` `s3_region` and `root_domain` as described in "Configure Garage for the
  proxy".
- Every client signs its requests (SigV4). Garage verifies the signature and refuses an unsigned
  request, so the S3 API needs no further login in front of it.
- Keep your clients' single-request size below `MAX_CLIENT_SIZE` (`256m`): set the multipart part
  size, and for rclone the `--s3-upload-cutoff`, under it. rclone's default cutoff is 200 MiB, which
  fits.

## Configure Garage for the proxy

| Garage setting                                    | Value                                  | What breaks without it                                                                                                                                                  |
| ------------------------------------------------- | -------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `[s3_api]` `s3_region`                            | The region your clients sign with      | Garage answers a request signed for another region with `AuthorizationHeaderMalformed` ([configuration reference][garage-conf]). The tests used `garage`.               |
| `[s3_api]` `root_domain`                          | `".s3.example.com"`                    | Virtual-host-style requests (`mybucket.s3.example.com`) are not mapped to a bucket. Path-style needs no `root_domain`.                                                  |
| One BunkerWeb service per bucket host             | See "Import and initial detect run"    | A wildcard `SERVER_NAME` is refused by BunkerWeb 1.6.17, so each virtual-host bucket needs its own service with `USE_TEMPLATE=garage`.                                  |
| Client endpoint scheme                            | `https://s3.example.com`               | With TLS on, BunkerWeb answers plain HTTP with a `301` to HTTPS (`AUTO_REDIRECT_HTTP_TO_HTTPS` defaults to `yes`), so a client left on `http://` gets a redirect instead of S3.       |
| Client `Host` with a port, when not 80 or 443     | Nothing to change on Garage            | BunkerWeb's default `Host $host` would drop the port and break the signature. The template sets `Host $http_host`, so any port works.                                    |

[garage-conf]: https://garagehq.deuxfleurs.fr/documentation/reference-manual/configuration/

Garage reads the client address only to label its own log line, so no trusted-proxy setting exists
on the Garage side (`src/api/common/generic_server.rs` and `src/util/forwarded_headers.rs` at
`v2.5.0`). The template still sends `X-Forwarded-For
$remote_addr`, see "Things to think about".

## Import and initial detect run

Import `template.json` and `configs/modsec-crs/garage_false_positives.conf` using the repository
[installation guide](../../README.md#install), or its plugin bundle layout. Assign
`USE_TEMPLATE=garage`, update `SERVER_NAME` and `REVERSE_PROXY_HOST` (keep the port, no trailing
slash or path), and review the guided steps.

**Virtual-host-style buckets.** BunkerWeb 1.6.17 refuses a wildcard `SERVER_NAME`
(`*.s3.example.com` fails the setting's regex and the whole configuration is not saved). List every
bucket host as a service of its own and give each one the template:

```text
SERVER_NAME=s3.example.com mybucket.s3.example.com
mybucket.s3.example.com_USE_TEMPLATE=garage
mybucket.s3.example.com_REVERSE_PROXY_HOST=http://garage:3900
```

Each service gets its own copy of the exclusions. Path-style addressing needs only the first host.

Before enabling blocking on your instance:

1. Set the service `SECURITY_MODE=detect` and reload.
2. Run your real clients (backup tool, application SDK, browser uploads) while reading BunkerWeb's
   ModSecurity log lines; compare the rule ids with the list in "CRS exception scope and cost".
3. Switch to `SECURITY_MODE=block` and repeat the workflows.

> [!IMPORTANT]
> False positives return `403` and BunkerWeb counts them for bad behavior: about ten matching
> responses in 60 seconds ban the client for one day. That includes a backup job that fails the same
> way for every object.

## Settings

| Setting                                                | Purpose                                                                                                 |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| `REVERSE_PROXY_URL=/`, `REVERSE_PROXY_HOST`            | Proxy the whole S3 API. Keep the host path-less.                                                        |
| `REVERSE_PROXY_HEADERS=Host $http_host;X-Forwarded-For $remote_addr` | Forward the client's `Host` header with its port (SigV4 signs it) and a single client address. |
| `REVERSE_PROXY_INTERCEPT_ERRORS=no`                    | Let S3 clients read Garage's XML errors (`NoSuchKey`, `AccessDenied`) instead of BunkerWeb's HTML page. |
| `REVERSE_PROXY_BUFFERING=no`                           | Stream downloads instead of spooling them to a temporary file, as Garage's reverse-proxy guide advises. |
| `REVERSE_PROXY_URL_1=^/health$`, `REVERSE_PROXY_HOST_1=http://garage:3903` | Token-free health check for uptime monitors. |
| `ALLOWED_METHODS=GET\|HEAD\|POST\|PUT\|DELETE\|OPTIONS` | S3 uses `PUT` and `DELETE`; `OPTIONS` carries CORS preflights. The default answers `405`.              |
| `MAX_CLIENT_SIZE=256m`                                 | Largest single request body, a `PutObject` or one multipart part.                                       |
| `MODSECURITY_REQ_BODY_NO_FILES_LIMIT=2m`               | Room for `DeleteObjects` (1000 keys) and `CompleteMultipartUpload` XML bodies.                          |
| `LIMIT_REQ_RATE=1000r/s`                               | S3 clients send bursts of small requests on one path.                                                   |
| `LIMIT_CONN_MAX_HTTP1=200`                             | S3 clients open many parallel connections.                                                              |
| `BAD_BEHAVIOR_STATUS_CODES=400 401 403 405 429 444`    | Drop `404` from the list: clients probe keys that do not exist yet.                                     |
| `SERVE_FILES=no`                                       | Fetch content from Garage rather than local files.                                                      |

What each one costs, and the measurement behind it:

- **`Host $http_host`.** BunkerWeb's default forwards `Host $host`, which drops the port. A client
  that signed `s3.example.com:8443` then gets Garage's `403`. On the standard ports 80 and 443 both
  forms are identical; with the setting, any port works.
- **`REVERSE_PROXY_INTERCEPT_ERRORS=no`.** BunkerWeb's own pages (a ModSecurity block, a `413`) are
  still BunkerWeb's HTML, which S3 SDKs cannot parse: rclone reports "XML syntax error". Only
  upstream replies become transparent.
- **`REVERSE_PROXY_BUFFERING=no`** affects responses only. ModSecurity needs the whole request body,
  so BunkerWeb spools every upload to disk first (`a client request body is buffered to a temporary
  file` in the log, 121 times in the smoke run) and `REVERSE_PROXY_REQUEST_BUFFERING=no` would do
  nothing. Plan disk space under BunkerWeb's temp directory for your largest upload times the number
  of parallel uploads.
- **`MAX_CLIENT_SIZE=256m`** is also the largest object a client can upload in one request. Garage
  accepts up to 5 GiB per object; larger objects need multipart with parts below this limit. One
  MiB more than the limit gets BunkerWeb's `413` page.
- **`MODSECURITY_REQ_BODY_NO_FILES_LIMIT=2m`**: the measured case is a `DeleteObjects` of 1000
  keys, 198084 bytes, which the default `131072` rejects. 2m is sized, not measured, for 1000 keys
  of 1024 bytes (about 1.1 MB) and the 10000 parts of a `CompleteMultipartUpload`. It applies to every parsed
  body of the service (XML, JSON, forms), so it also raises the limit of those. Raw object bodies are
  not affected: ModSecurity does not parse them.
- **`LIMIT_REQ_RATE=1000r/s`.** The counter is per client address and path. The heaviest measured
  case is `rclone sync` of a 300-directory tree with 64 checkers: 457 requests per second in total
  and 284 on a single path (the per-directory listings). At `100r/s` that run got 78 answers of
  `429` and 13 rclone errors; at `1000r/s` it finished with 300 of 300 checks and no error. The
  value is about 3.5 times the same-path peak. A single bucket path with many clients behind one
  address needs more.
- **`LIMIT_CONN_MAX_HTTP1=200`.** With the default `10`, `rclone --transfers 16 --checkers 32` made 32
  requests fail with `429` ("limiting connections by zone"). The largest measured load, rclone with
  32 transfers and 64 checkers, held 96 parallel connections and passed at `100`; `200` leaves about
  twice that, not measured separately. HTTP/2 already allows 100.
- **`BAD_BEHAVIOR_STATUS_CODES`** without `404`: an application that checks whether 12 objects exist
  (`HeadObject` on keys that are not there) is banned by the default list; the next ordinary request
  answers `403`. Garage only answers `404` to a signed, authorized request, so scanners still count
  through `400`, `401`, `403` and `405`.
- No timeout setting is shipped: the 200 MiB uploads and a 60 MiB virtual-host-style multipart upload
  ran inside the defaults, so a timeout setting would be unproven.
- CORS is Garage's own per-bucket configuration; the preflight passes through unchanged.

## CRS exception scope and cost

`configs/modsec-crs/garage_false_positives.conf` is a service-scoped **modsec-crs** config with
five chained rules, ids `9857100` to `9857104`. S3 object keys and content types are arbitrary, so no
path can be named. Every exclusion therefore requires, together:

- a **signed S3 request**: an `Authorization: AWS4-HMAC-SHA256` header, or the `X-Amz-Algorithm`
  query argument of a presigned URL;
- the **S3 method** that needs it;
- a **non-root path** (`\A/[^/]`, normalized).

Garage checks the signature itself. A forged marker only skips the listed checks for a request that
Garage then rejects. Requests without the marker keep every rule. Each rule removes one rule's
target, never the rule, and `949110` and `959100` are never touched.

| Id      | Methods                           | What is skipped                                                                                             | Why                                                                                                  |
| ------- | --------------------------------- | ----------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `9857100` | `PUT`                             | `920420` content-type policy                                                                                | An object carries its own type: `text/plain`, `image/png`, `application/octet-stream`, `video/mp4`.  |
| `9857104` | `POST` with the `uploads` keyword | `920420` content-type policy                                                                                | `CreateMultipartUpload` carries the object's type (rclone and the AWS CLI both send it).             |
| `9857101` | `PUT`, `POST`                     | `920340` and `920640` (body without `Content-Type`)                                                         | The AWS CLI sends `DeleteObjects` without a `Content-Type`; some SDKs send unlabelled uploads.       |
| `9857102` | `PUT`                             | `920450` for the header names `Expect` and `Content-Encoding` only                                          | SDKs add `Expect: 100-continue`, and `Content-Encoding` for `aws-chunked`, gzip or the like.         |
| `9857103` | `GET HEAD PUT POST DELETE`        | `920440` (restricted extension) on the basename, `930130` (restricted file) and `933160` (PHP call) on path | Keys like `backup.sql`, `tls.key`, `.env`, `.DS_Store`, `package.json`, `Dockerfile`, `file (1).txt`. |

**Cost:**

- The content-type allow-list no longer applies to signed uploads, and a body with no
  `Content-Type` is accepted. Every injection rule (`93x`, `94x`, `942x`) still inspects the
  request, and `930100` and `930110` still read the path, so a traversal in a key stays blocked.
- A restricted extension or file name in an object key is no longer a signal. `Proxy`,
  `Content-Range`, `If` and every other header name stay restricted.
- **Objects labelled as JSON, XML, urlencoded or multipart are parsed and inspected like any
  request body.** A `PutObject` with `Content-Type: application/json` that contains SQL-looking text
  gets `403` (rule `942100`), an XML object with `<script>` text gets `403` (`941100`), and a body
  that is not valid JSON under a JSON label (NDJSON, a template) gets `400` (rule `200002`). No
  literal target can name an object's content, so the template does not exclude it. Label such
  uploads `application/octet-stream` in your client or accept the check. Benign JSON and XML
  objects pass.
- A key that is itself an attack string, such as `eval (x).js` (also `934100`, `941390` on the path)
  or a traversal, is still blocked.
- The exclusions rely on ModSecurity matching `Content-Type`, `Expect` and `Content-Encoding` in the
  case the client sends them: rclone, the AWS CLI and a raw client sending lowercase names all passed.

## Things to think about

- **Client IP in Garage's log.** Garage parses the whole `X-Forwarded-For` header as one address; a
  list makes the parse fail and it logs BunkerWeb's address instead. BunkerWeb's default appends to
  a client-supplied value. The template replaces it with `$remote_addr`: with it Garage logged the
  real client for no header, one address and a list; without it, a client-supplied header made
  Garage log BunkerWeb. The address is only a log label, so nothing can be spoofed into access. If
  BunkerWeb itself sits behind another proxy, set `USE_REAL_IP` first, or `$remote_addr` is that
  proxy (not tested).
- **HTTPS with an HTTP upstream.** The signature binds the `Host` header, not the scheme: a
  presigned `GET` and a signed listing over HTTPS to BunkerWeb with `http://garage:3900` behind it
  returned `200` (HTTP/2), and rclone and the AWS CLI listed a bucket over HTTPS. Every client must use the `https`
  endpoint: after TLS is on, plain HTTP gets a `301`. BunkerWeb also sends HSTS
  (`includeSubDomains; preload`), which browsers apply to every subdomain.
- **Health check.** The step "HTTP - Health check" maps `^/health$` to Garage's admin port, which
  answers `GET /health` without a token (`200`, "Garage is fully operational"). `/metrics`,
  `/v1/health`, `/health/x` and `/check` through BunkerWeb are answered by the S3 side with `403`,
  so the rest of the admin API stays private. A bucket named `health` is the cost: with the step on,
  `GET /health?list-type=2` reached the admin port (`200`); without it the same request reached the
  S3 API. Clear `REVERSE_PROXY_URL_1` and `REVERSE_PROXY_HOST_1` if you have such a bucket or no
  monitor.
- **Website hosting (`3902`) and K2V.** Not covered and not tested. Serve a website
  endpoint from a separate BunkerWeb service of its own, never from the S3 hostname, and do not
  enable K2V behind this template without testing it.
- **Browser uploads.** Garage's per-bucket CORS passes through unchanged. In real Chrome, from
  another origin, a presigned `PUT` (`200`, ETag readable through `ExposeHeader`), a presigned
  `GET`, an `<img>` and a presigned `POST` (PostObject, as Outline uploads) with a concrete key
  (`201`, filename `upload (1).png`) all passed through the template's defaults. A key with the
  `${filename}` placeholder in a PostObject policy returns `403` (rules `932130` and `933135` on
  `ARGS:key`): use a concrete key.
- **Embedding an object in an iframe.** BunkerWeb's default `Content-Security-Policy` is
  `object-src 'none'; form-action 'self'; frame-ancestors 'self';`, so a browser refuses to frame an
  object from another origin ("violates CSP directive frame-ancestors 'self'"). Set
  `CONTENT_SECURITY_POLICY=object-src 'none'; form-action 'self'; frame-ancestors https://app.example.com`
  on the S3 service to allow one embedding origin; `X-Frame-Options: SAMEORIGIN` was left in place in the test and the frame loaded. The setting is not shipped: it would apply to every object.
- **Rate limits count per client address and path.** `LIMIT_REQ_RATE` keys on the client IP and the
  request path without its query. A sync touching many distinct object paths never reaches it; the
  same bucket path does, as each listing (`GET /bucket?prefix=...`) and `HEAD /bucket` counts on
  `/bucket`. Many clients behind one NAT add up on that path.
- **Caching proxies in front.** Garage users report `HEAD` returning `403` behind Cloudflare and
  Traefik with caching ([#1410][g1410], [#1226][g1226]). Not tested here; do not cache S3 requests
  between the client and BunkerWeb.
- **SDK uploads that Garage rejects.** Upstream issue [#1549][g1549] (open): AWS SDK for Java v2
  from `2.30` sends `STREAMING-AWS4-HMAC-SHA256-PAYLOAD-TRAILER` uploads over plain HTTP and Garage
  answers "Invalid payload signature". The report is about Garage and that SDK; it was not
  reproduced through BunkerWeb here.

[g1410]: https://git.deuxfleurs.fr/Deuxfleurs/garage/issues/1410
[g1226]: https://git.deuxfleurs.fr/Deuxfleurs/garage/issues/1226
[g1549]: https://git.deuxfleurs.fr/Deuxfleurs/garage/issues/1549

## Validation

Tested on 2026-11-17: Garage `v2.5.0` (single node), BunkerWeb `1.6.17` (OWASP CRS 4.29.0,
libmodsecurity 3.0.17), default paranoia level and anomaly thresholds, `SECURITY_MODE=block`.
The matrix and the smoke run used HTTP; the HTTPS checks used a self-signed certificate.
Clients: rclone `1.69.1`, AWS CLI `2.22.35`, a SigV4 client that sends exact headers, and Chrome for
the browser checks.

<details>
<summary>Test evidence</summary>

- **Request matrix.** 133 requests, one at a time: 97 seeds, 32 controls and
  4 documented limits. Each seed passes on its status and on a read-back from Garage (the
  object bytes, the listing, the bucket, the CORS and lifecycle configuration). Without the
  exclusion config, 32 of 97 seeds passed and 65 failed: 54 got `403` (`920420`, `920340`, `920640`,
  `920450`, `920440`, `930130`, `933160`), and 11 more failed because the object or upload they
  depend on was blocked (nine reads and copies of an object whose `PUT` was blocked, a listing page
  with no objects to page through, and a multipart completion after blocked parts). The PostObject
  seeds and `/health` passed without the config. With the config, 133 of 133 passed. Each control (the same payload unsigned, on another method, on the root path, with another
  header, with an injection in a query argument, or an admin path other than `/health`) stayed `403`.
  Covered: `PutObject` with 12 content types and with no type, keys with spaces, quotes,
  parentheses, unicode, a double slash, `.sql`/`.log`/`.env` names, `GetObject` with `Range` and
  response overrides, `HeadObject`, `ListObjects` v1 and v2 with prefix, delimiter, continuation
  token and `start-after`, `CopyObject`, `DeleteObject`, `DeleteObjects` (including 1000 keys,
  198084 bytes), multipart (initiate, parts, list, complete, abort), presigned `GET` and `PUT`,
  PostObject form uploads, `CreateBucket`, `PutBucketCors`, `PutBucketLifecycleConfiguration`, a
  CORS preflight, `GET /health` and both addressing styles.
- **Settings.** Each setting was switched back to its default alone: `MAX_CLIENT_SIZE` (20 MiB `413`
  at `10m`; 256 MiB `200`, 257 MiB `413` with the template), `ALLOWED_METHODS` (`PUT`, `DELETE`,
  `OPTIONS` all `405`), error interception (`NoSuchKey` and `AccessDenied` arrive as BunkerWeb HTML),
  the body limit (`DeleteObjects` `400`, rule `200002`), buffering (one "upstream response is
  buffered to a temporary file" line for a slow reader of a 200 MiB object, none with the template),
  `Host` with a port (`403` against `200`), the request rate (see above), the connection limit, the
  `404` ban described above, the forwarded address (Garage's log with and without the header) and the
  health step (`200` against `403`).
- **Real clients, bad behavior on, template as shipped** (only the hostnames and the harness's
  port-80 forwarder differ from a user setup): rclone `mkdir`, `copyto`, `copy` of 50 files with
  names such as `.env`, `.DS_Store`, `package.json` and `my file (1).txt`, a 200 MiB upload in one
  request and another as multipart, `ls`, `lsd`, `cat`, `check`, `delete`, `purge`; the two 200 MiB
  objects read back with the same SHA-256 as the source; a 60 MiB multipart upload in virtual-host
  style; AWS CLI `cp` of 30 MB (multipart), `sync`, `ls`, `head-object`, `delete-objects`, `rm
  --recursive`, and a virtual-host-style `cp`, `ls`, `rm`; an unsigned `GET /health`. 27 checks
  passed. 382 requests reached the two services: `200` (268), `204` (103), `404` (7) and `409` (4).
  No ModSecurity line, no `403`, `429` or `444`, no ban.
- **Other user agents.** A signed listing through BunkerWeb with the template as shipped and the user agents of
  restic, minio-go, aws-sdk-go-v2, Boto3, s3cmd, aws-sdk-java, Cyberduck, curl, python-requests,
  Go-http-client, rclone and s3fs: `200` each. Only the empty `User-Agent` request left an
  audit-log entry, and it also returned `200`. The strings are representative ones typed from each
  client's usual format, not captured from the real tools.
- **Browser.** Real Chrome, page on `http://app.localhost:19110`, S3 on `http://s3.garage.localhost:19100`:
  presigned `PUT`, `GET`, PostObject and `<img>` passed; the iframe was blocked by `frame-ancestors`
  until the `CONTENT_SECURITY_POLICY` override above.
- **Web UI import.** Templates, Create template, Raw: `template.json` and the config uploaded,
  saved, and listed next to the plugin copy (done on the first build of the template, before the
  health and forwarding settings were added; the final file was packed as a plugin for every run).

</details>

Not verified: HTTP/3; a multi-node Garage cluster; clients other than those named (minio-go and
restic, boto3 and s3cmd with real transfers, not only a listing); streaming-signed uploads with a
literal `Content-Encoding: aws-chunked` (the header name is excluded, the value is not what rule
`920450` reads); SSE-C; Garage's website endpoint (`3902`) and K2V; `USE_REAL_IP` with
BunkerWeb behind another proxy; uploads above 256 MiB; idle connections (S3 has no long-lived
channel). Validate your own clients before enabling blocking.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
