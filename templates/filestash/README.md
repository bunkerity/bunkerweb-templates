<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/filestash.svg" alt="Filestash logo" width="96" height="96" />
</p>

<h1 align="center">Filestash</h1>

<p align="center">File manager template with WebDAV-aware proxy defaults</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#custom-configs">Custom configs</a> ·
  <a href="#docker-compose-example">Docker Compose example</a> ·
  <a href="#validation">Validation</a>
</p>

Provision a BunkerWeb configuration for [Filestash](https://www.filestash.app/), the web file
manager that connects to S3, SFTP, WebDAV and other storage backends. The template proxies
Filestash with automatic Let's Encrypt certificates, WebSocket support, a 100m request size limit,
the verbs Filestash uses for WebDAV on shared links and a ModSecurity exclusion file for sign-in,
folder paths, the web editor and uploads.

Editing Office documents inside Filestash needs a Collabora (WOPI) server on its own hostname. The
[filestash-office](../filestash-office/README.md) template fronts that server and uses the same
Compose example, shown further down on this page.

## At a glance

| Fact        | Value                         |
| ----------- | ----------------------------- |
| Template id | `filestash`                   |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Filestash build `b6fe63f4f01e` |
| Configs     | 1 (modsec-crs)                |
| Rule ids    | `9855000` to `9855999`        |
| Upstream    | `http://filestash:8334`       |

## Prerequisites

- A running Filestash server that BunkerWeb can reach, with `APPLICATION_URL` set to the public
  hostname of the BunkerWeb service (the Compose example below does it). Filestash answers
  `Request coming from ... was blocked` for any other hostname. Tested with the
  `machines/filestash:latest` image built on 2026-11-20 (build `b6fe63f4f01e`, shown on the admin
  console About page), on BunkerWeb 1.6.17 in a real browser.
- A public hostname pointing to BunkerWeb.
- Access to the BunkerWeb web UI.

## Install

1. Import `template.json` (Templates → **Create template** → **Raw**, then upload or paste it).
2. When the **Missing Custom Configs** dialog appears, upload every file listed in the
   [Custom configs](#custom-configs) section from `configs/`, then save.
3. Create a service and pick **Filestash** in the **Templates** menu above the steps (its
   description reads "Filestash file manager with WebDAV-aware proxy defaults"). Plugin users
   assign `USE_TEMPLATE=filestash` instead.
4. Replace `SERVER_NAME`, `EMAIL_LETS_ENCRYPT` and `REVERSE_PROXY_HOST` with values for your
   deployment. Keep the example e-mail only for a lab. Without Office documents, delete the
   `https://filestash-office.example.com` host from `CONTENT_SECURITY_POLICY` (step 6); with
   them, replace it with the public hostname of your Collabora service.
5. Save the service: BunkerWeb applies it on its own. Then do the [first run](#first-run) below.

For both import methods (web UI and plugin), see the repository [installation guide](../../README.md#install).

### First run

A new Filestash server has no admin password and no storage backend yet, and sign-in does not work
until both exist:

1. Open the public URL. Filestash sends you to `/admin/setup`: choose the admin password.
2. Open **Storage** in the admin console (`/admin/storage`) and click a storage tile (for example
   `local`, `sftp` or `s3`) to enable it. Until a backend is enabled, the sign-in page shows
   "Internal Error". The setup page also offers presets that fill in a storage backend and an
   authentication method for you.
3. Open the public URL again, sign in, then test an upload, a download and the web editor.

> [!WARNING]
> The admin console (`/admin`) is reachable from the internet like the rest of the service, and
> the first visitor to open it chooses the admin password. Do step 1 as soon as the service is
> live, and keep the password strong. After that, the admin API answers only to the admin
> password. Restrict `/admin` to your own addresses with a rule of your own if you want more.

### Placeholders

The values below are examples. Replace them with your own before using the template:

- `example.com` → your own domain name
- `admin@example.com` → your own email address
- `http://filestash:8334` → the name of the service's container on the shared Docker network and
  the port Filestash listens on (8334), or the IP address and port of the server, for example
  `http://10.0.0.5:8334`
- `https://filestash-office.example.com` in `CONTENT_SECURITY_POLICY` → the public hostname of your
  [Collabora service](../filestash-office/README.md), or delete it when you do not edit Office
  documents

## Settings

### Notes

- **Certificates:** the HTTP challenge needs port 80 on BunkerWeb to be reachable from the internet
  for every domain in `SERVER_NAME`.
- **Upstream:** `http://filestash:8334` is a container name and port. It only resolves when
  BunkerWeb and the application share a Docker network (the Compose example joins the external
  `bw-services` network with the alias `filestash`; attach BunkerWeb to the same network);
  otherwise replace it with the IP address and port of the service, for example
  `http://10.0.0.5:8334`.
- **WebSockets:** enabled on the proxied route.
- **Uploads:** `MAX_CLIENT_SIZE=100m` caps one request body at 100 MB. BunkerWeb's ModSecurity
  holds the request body in memory: on BunkerWeb 1.6.17, a 300 MB raw upload raised the memory use
  of the BunkerWeb container by about 1.7 GiB (peak 1.8 GiB), and a 90 MB upload peaked at about
  1 GiB. By default (`upload_chunk_size` of `0`, in `server/pkg/config/config.go`) Filestash sends
  each file in a single `POST /api/files/save` (`public/assets/pages/filespage/ctrl_upload.js`,
  `chunkSize === 0`), so without a change a file over 100 MB is answered with a BunkerWeb
  `413 Request Entity Too Large`. To upload larger files, open the Filestash admin console, go to
  the general settings and set `upload_chunk_size` (in MB) to a value below 100, for example `50`.
  Filestash then sends files larger than one chunk as `PATCH /api/files/save` requests of that size
  (tus), and each request stays under the limit. Raising `MAX_CLIENT_SIZE` instead is your call:
  each concurrent upload then costs roughly 5 to 7 times its size in BunkerWeb memory. Filestash
  also uploads up to `upload_pool_size` files in parallel (default `15`, in
  `server/pkg/config/config.go`) and each one costs its own BunkerWeb memory, so size the
  BunkerWeb container for that or lower `upload_pool_size`. Source lines read at Filestash commit
  `76238d7`.
- **WebDAV:** the WebDAV verbs (for example `PROPFIND`, `MKCOL`) are allowed through BunkerWeb.
  Filestash itself has no WebDAV server on `/`, `/webdav` or `/dav` (they answer `405`). It serves
  WebDAV only on shared links (`/s/<share id>`, a `PROPFIND` there answers `401` for an unknown
  link). To mount a shared link from a WebDAV client, add `^/s/` to `ANTIBOT_IGNORE_URI`, because
  a client cannot solve the challenge; that also lets a script open shared-link pages without
  the challenge. Mounting a shared link from a WebDAV client was not tested.
- **Bad behavior:** responses with status 400, 401, 403, 405 and 444 count toward a temporary IP
  ban. Make sure normal clients do not trigger these regularly.
- **Anti-bot:** `ANTIBOT_IGNORE_URI` (`^/api/`) skips the challenge for the Filestash API because
  API clients cannot solve it. That path is protected only by the application and the other
  security features. The admin API (`/admin/api/`) keeps the challenge. A client that connects
  through another path, such as a WebDAV client on a shared link, cannot solve the challenge
  either: add its path to `ANTIBOT_IGNORE_URI` (see **WebDAV** above).
- **Upstream headers:** `KEEP_UPSTREAM_HEADERS` keeps two headers Filestash sets itself. The
  `Content-Security-Policy` locks down the files Filestash serves for preview, so an uploaded HTML
  or SVG file cannot run scripts. `X-Frame-Options` is sent by Filestash unless the admin allows
  embedding in the admin console. BunkerWeb's defaults replace them otherwise.
- **Content security policy:** Filestash sends no policy on the page that opens the Office editor,
  so `CONTENT_SECURITY_POLICY` applies there. BunkerWeb's default `form-action 'self'` blocks the
  form that sends the document to Collabora, and the editor stays on its loading screen. The
  template adds your Collabora hostname to `form-action`: replace the example host, or delete it
  when you do not use Office documents.
- **Compression:** `USE_GZIP` is on so that `GZIP_PROXIED` takes effect.
- **Allowlist:** `WHITELIST_IP` is not set by the template. Set it on the service only for the IP
  addresses or ranges that may bypass the security checks, and review the list regularly.
- **ModSecurity rule IDs:** the template's rules use IDs 9855101 to 9855106, inside its block
  9855000 to 9855999 of the range BunkerWeb reserves for community templates (see "Rule IDs" in
  [`TEMPLATE_GUIDE.md`](../../TEMPLATE_GUIDE.md)). Rules you add yourself belong in 1 to 99999.

### Web service - Front service

Point the template at your Filestash hostname and your own e-mail. Turn Let's Encrypt off for a name
that is not public.

| Setting              | Value                   | Description                                                 |
| -------------------- | ----------------------- | ----------------------------------------------------------- |
| `SERVER_NAME`        | `filestash.example.com` | Domain name(s) this service answers for.                    |
| `AUTO_LETS_ENCRYPT`  | `yes`                   | Whether Let's Encrypt certificates are obtained and renewed automatically. |
| `EMAIL_LETS_ENCRYPT` | `admin@example.com`     | Contact email registered with Let's Encrypt.                |

### Web service - Upstream server

Direct the reverse proxy at Filestash and keep long uploads and WebSockets alive.

| Setting                           | Value                   | Description                                                                         |
| --------------------------------- | ----------------------- | ----------------------------------------------------------------------------------- |
| `USE_REVERSE_PROXY`               | `yes`                   | Whether requests are forwarded to an upstream application.                          |
| `REVERSE_PROXY_HOST`              | `http://filestash:8334` | Upstream application: container name and port, or IP address and port.              |
| `REVERSE_PROXY_WS`                | `yes`                   | Whether WebSocket connections are allowed on the proxied route.                     |
| `REVERSE_PROXY_BUFFERING`         | `no`                    | Whether BunkerWeb buffers upstream responses before sending them to the client.     |
| `REVERSE_PROXY_REQUEST_BUFFERING` | `no`                    | Whether BunkerWeb buffers the whole client request before forwarding it upstream.   |
| `REVERSE_PROXY_INTERCEPT_ERRORS`  | `no`                    | Whether BunkerWeb replaces upstream error pages with its own error pages.           |
| `REVERSE_PROXY_READ_TIMEOUT`      | `3600s`                 | Maximum time to wait for the upstream to send data.                                 |
| `REVERSE_PROXY_SEND_TIMEOUT`      | `3600s`                 | Maximum time to wait while sending data to the upstream.                            |
| `PROXY_BUFFERS`                   | `8 32k`                 | Number and size of NGINX buffers used for upstream responses.                       |
| `PROXY_BUFFER_SIZE`               | `32k`                   | Size of the buffer for the first part of the upstream response.                     |
| `PROXY_BUSY_BUFFERS_SIZE`         | `64k`                   | Limit of buffers busy sending a response to the client.                             |

### HTTP - Filestash specific settings

Allow the verbs Filestash uses for WebDAV on shared links, large uploads, and let Filestash serve its
own files.

| Setting           | Value                                                                       | Description                                              |
| ----------------- | --------------------------------------------------------------------------- | -------------------------------------------------------- |
| `ALLOWED_METHODS` | `GET\|POST\|HEAD\|PUT\|DELETE\|PATCH\|OPTIONS\|PROPFIND\|MKCOL\|COPY\|MOVE\|LOCK\|UNLOCK` | HTTP methods that are accepted; all others are rejected. |
| `MAX_CLIENT_SIZE` | `100m`                                                                      | Maximum size of a single request body.                   |
| `SERVE_FILES`     | `no`                                                                        | Whether BunkerWeb serves static files itself (off when proxying an app). |

### Security - Rate and connection limits

File listings and uploads fan out many parallel requests, so the limits sit above the defaults.

| Setting                | Value          | Description                                                  |
| ---------------------- | -------------- | ------------------------------------------------------------ |
| `LIMIT_CONN_MAX_HTTP1` | `50`           | Maximum concurrent HTTP/1 connections per client IP.         |
| `LIMIT_CONN_MAX_HTTP2` | `500`          | Maximum concurrent HTTP/2 streams per client IP.             |
| `LIMIT_CONN_MAX_HTTP3` | `500`          | Maximum concurrent HTTP/3 streams per client IP.             |
| `LIMIT_REQ_RATE`       | `50r/s`        | Allowed request rate for the default URL (for example 15r/s). |
| `LIMIT_REQ_URL_1`      | `^/api/files/` | URL path the first extra rate limit applies to.              |
| `LIMIT_REQ_RATE_1`     | `200r/s`       | Allowed request rate for that URL (for example 15r/s).       |

### Security - Antibot and bad behavior

Challenge browsers but skip the API, which clients cannot solve, and ban repeated errors.

| Setting                     | Value                 | Description                                                                   |
| --------------------------- | --------------------- | ----------------------------------------------------------------------------- |
| `USE_ANTIBOT`               | `javascript`          | Challenge type shown to visitors to block bots.                               |
| `ANTIBOT_IGNORE_URI`        | `^/api/`              | URL patterns (regex) that skip the anti-bot challenge, for example API routes. |
| `BAD_BEHAVIOR_STATUS_CODES` | `400 401 403 405 444` | Response status codes that count toward a temporary ban of the client IP.     |
| `BAD_BEHAVIOR_THRESHOLD`    | `40`                  | Number of bad responses before the client IP is banned.                       |

### Security - Headers and exposure

Keep the headers Filestash sets itself, and let the editor form reach your Collabora host.

| Setting                   | Value                                   | Description                                                            |
| ------------------------- | --------------------------------------- | ---------------------------------------------------------------------- |
| `KEEP_UPSTREAM_HEADERS`   | `Content-Security-Policy X-Frame-Options` | Upstream headers that BunkerWeb passes through instead of overwriting. |
| `CONTENT_SECURITY_POLICY` | `object-src 'none'; form-action 'self' https://filestash-office.example.com; frame-ancestors 'self';` | Policy BunkerWeb sends on pages that Filestash gives none. Replace the example host with your Collabora hostname, or delete it when you do not edit Office documents. |
| `INTERCEPTED_ERROR_CODES` | `404 405 413 429 500 501 502 503 504`   | Status codes answered with BunkerWeb error pages when BunkerWeb itself generates them (for example 405 from `ALLOWED_METHODS`, 413, 429). Upstream errors pass through unchanged because `REVERSE_PROXY_INTERCEPT_ERRORS` is `no`. |
| `USE_ROBOTSTXT`           | `yes`                                   | Whether BunkerWeb serves a generated robots.txt file.                  |

### Performance - Client optimization

Compress proxied responses; Filestash serves its own files.

| Setting        | Value                                    | Description                                                             |
| -------------- | ---------------------------------------- | ----------------------------------------------------------------------- |
| `USE_GZIP`     | `yes`                                    | Whether BunkerWeb compresses responses with gzip.                       |
| `GZIP_PROXIED` | `expired no-cache no-store private auth` | Which proxied responses are compressed, based on their caching headers. |

### Security - WAF tuning

Let uploads, the editor save, sign-in and folder paths through the checks that reject them, one
request at a time.

Custom configs: `modsec-crs/filestash_false_positives.conf`

## Custom configs

Files are referenced relative to the template's `configs/` directory and grouped by BunkerWeb config
type.

| Type         | File in repository                                                      | Reference in `template.json`                  | Purpose                                                                  |
| ------------ | ----------------------------------------------------------------------- | --------------------------------------------- | ------------------------------------------------------------------------ |
| `modsec-crs` | `templates/filestash/configs/modsec-crs/filestash_false_positives.conf` | `modsec-crs/filestash_false_positives.conf`   | Removes CRS rule 920420 for the editor save and the upload requests only, plus 920340 and 920640 for the single request upload. Removes CRS rule 930120 for the path arguments of sign-in, sharing, browsing and the Office editor, and 942550 for one admin preset field. |

CRS rule 920420 rejects a request whose `Content-Type` is not in the CRS allowed list. Filestash
sends three requests that hit it. Each rule matches one method and one anchored path, so the rest
of the API keeps the full CRS. Rule `9855102` also removes 920340 and 920640 (`Content-Type` header
missing from a request with a body), because the browser sends a file with an unknown MIME type, for
example a small `.bin`, with no `Content-Type` at all:

| Rule ID | Request                    | `Content-Type` sent by the browser                         |
| ------- | -------------------------- | ---------------------------------------------------------- |
| `9855101` | `POST /api/files/cat`      | `text/plain;charset=UTF-8` (web editor save)               |
| `9855102` | `POST /api/files/save`     | the file's own type, for example `application/pdf`, or no header |
| `9855103` | `PATCH /api/files/save`    | `application/offset+octet-stream` (chunked upload)         |

The CRS does not inspect these bodies either way, because no body processor is armed for their
types. The global allowed content type list is not changed.

CRS rule 930120 (OS file access attempt) rejects a path that names a system folder, such as `/tmp/`
or `/var/www/`, in any argument. Filestash is a file manager: users type such a folder as the start
folder at sign-in, open it, upload to it and share it, so the rule fires on normal use. Only the
arguments that carry a folder or file path skip it, on these requests. Every other CRS rule still
reads those arguments (directory traversal, shell, SQL and script text), and the same text in any
other argument or on any other path is still answered `403`:

| Rule ID   | Request                                                                      | Argument that skips 930120 |
| --------- | ---------------------------------------------------------------------------- | -------------------------- |
| `9855104` | `POST /api/session` (sign-in), `POST /api/share/<id>` (create a shared link) | JSON field `path`          |
| `9855105` | any method on `/api/files/<action>`, `/api/metadata/<action>`, `/api/wopi/iframe`, `/api/share` | `path`, `from`, `to` |

Rule `9855106` removes the SQL injection rule 942550 for the field
`middleware.attribute_mapping.params` of `POST /admin/api/config`, which the admin console sends
when you apply the "Local files" or "S3 - Use your AWS credentials" preset (a small JSON text that
the CRS reads as SQL).

## Docker Compose example

This example runs Filestash and the Collabora server used by the
[filestash-office](../filestash-office/README.md) template. Neither container publishes a port:
BunkerWeb reaches them over the external `bw-services` network, and a published port would bypass
TLS, the WAF and the rate limits. Replace `filestash.example.com` and
`filestash-office.example.com` with your own hostnames, and keep them identical to the
`SERVER_NAME` of the matching BunkerWeb service. Skip the `wopi_server` service if you do not edit
Office documents, and remove the three `OFFICE_*` lines from the `app` service too: they point at a
host that no longer exists.

> [!NOTE]
> `bw-services` is an external network. If your BunkerWeb stack does not define it yet, create
> it with `docker network create bw-services` and attach BunkerWeb to it before running
> `docker compose up -d`.

```yaml
services:
  app:
    container_name: filestash
    image: machines/filestash:latest
    restart: always
    environment:
      - APPLICATION_URL=filestash.example.com
      - CANARY=true
      - OFFICE_URL=http://wopi_server:9980
      - OFFICE_FILESTASH_URL=http://app:8334
      - OFFICE_REWRITE_URL=https://filestash-office.example.com
    volumes:
      - filestash:/app/data/state/
    networks:
      default:
      bw-services:
        aliases:
          - filestash
  wopi_server:
    container_name: filestash_wopi
    image: collabora/code:24.04.10.2.1
    restart: always
    environment:
      - "extra_params=--o:ssl.enable=false --o:ssl.termination=true --o:net.frame_ancestors=https://filestash.example.com"
      - aliasgroup1=http://app:8334
    networks:
      default:
      bw-services:
        aliases:
          - filestash-office

volumes:
  filestash: {}

networks:
  bw-services:
    external: true
```

- **Start command:** the upstream Filestash Compose file overrides the Collabora start command with
  `user: root`, a `curl` download of a branding stylesheet from a third-party gist into Collabora's
  web root, and an `su` to the `cool` user. The download is cosmetic, so this example leaves it out
  and uses the image's own start command. That also removes the need for the `root` override.
- **`aliasgroup1`:** it lists the WOPI hosts Collabora accepts. Filestash builds the WOPI address
  from `OFFICE_FILESTASH_URL`, so the host is `http://app:8334`. The upstream example uses a regex
  that accepts any HTTPS host, which lets any site use your Collabora server as a WOPI client.
- **`OFFICE_REWRITE_URL`:** browsers load the editor from this URL, so it must be the public
  hostname of the BunkerWeb service that uses the
  [filestash-office](../filestash-office/README.md) template, over HTTPS. This template only
  routes `SERVER_NAME`: create a separate BunkerWeb service with its own TLS certificate for that
  hostname.
- **`ssl.enable=false` and `ssl.termination=true`:** the first makes Collabora serve plain HTTP
  because BunkerWeb terminates TLS, so you proxy it with an `http://` upstream. The second tells it
  that TLS ends at the proxy, so it hands the browser `wss://` addresses. Without it the editor
  opens but the document never loads: the browser refuses the `ws://` address Collabora announces.
- **`net.frame_ancestors`:** Collabora only lets a page embed the editor when its hostname is
  listed here. Set it to `https://` plus the hostname of the BunkerWeb service that uses the
  filestash template (the one in `APPLICATION_URL`). Without it the browser blocks the editor
  frame and the page stays on its loading screen.
- **Content security policy:** the Filestash service must also allow the form that opens the editor
  to reach the Collabora hostname: `CONTENT_SECURITY_POLICY` of the filestash template does it
  once you put your Collabora hostname in `form-action`.

## Validation

### Checks to run

Run these from the repository root before importing the template:

```bash
(
  set -eu
  t=templates/filestash
  jq . "$t/template.json"
  jq -e --arg expected "$(basename "$t")" '.id == $expected' "$t/template.json"
  jq -e '(.settings | keys | sort) == ([.steps[] | .settings[]] | sort)' "$t/template.json"
  for f in $(jq -r '.configs[]' "$t/template.json"); do
    test -f "$t/configs/$f" || { echo "missing: $f"; exit 1; }
  done
)
```

After applying the template:

- Confirm the BunkerWeb scheduler reload completes without a template or unknown-setting error.
- Sign in, save a file in the web editor, then upload a small file and a file larger than a few
  megabytes, and download them again.
- With Office documents: open a `.docx` from Filestash, type in it and save it with `Ctrl+S`.
- Check BunkerWeb logs for rate-limit, bad behavior, and ModSecurity events before changing
  security controls.

### Checks run for this template

- The four `jq` commands above: JSON syntax, `id` equal to the directory name, every setting listed
  in a step, every config file present.
- `pre-commit run --files` on the template files (Prettier, Codespell, Gitleaks and the whitespace
  hooks).
- The template JSON and the config file imported through a BunkerWeb 1.6.17 web UI that runs its
  own CRS in block mode (Templates → Create template → Raw → Save): no ModSecurity rule fired, and
  a service created from it in easy mode applied.
- A live Filestash (`machines/filestash:latest`, build `b6fe63f4f01e`) behind BunkerWeb 1.6.17 in a
  real browser: admin setup and storage backend, sign-in with the start folders `/tmp/` and
  `/var/www/`, browsing folders, uploads of 2 KB, 6 MB and 120 MB files (the last one chunked with
  `upload_chunk_size` 50), the web editor save and the admin presets. Before the rules
  `9855104` to `9855106` the sign-in, the folder listing and two presets were answered `403`; with
  them they work, and the same text in another argument, another path or another method is still
  `403` (28 requests).
- A live request matrix on `bunkerity/bunkerweb-all-in-one:1.6.17`: the three excluded requests
  (`POST /api/files/cat`, `POST /api/files/save` with and without a `Content-Type`,
  `PATCH /api/files/save`) reach Filestash, and the same payloads on another path, another method
  or another parameter, or with an attack in the query string, are still answered 403.
- A memory measurement on the same image with the template at `MAX_CLIENT_SIZE=2g`: a 300 MB raw
  `POST /api/files/save` and a 300 MB `PATCH /api/files/save` both returned 200, with the BunkerWeb
  container peaking at about 1.8 GiB (idle: about 0.2 GiB).
- The same setup at `MAX_CLIENT_SIZE=100m`: a 150 MB `POST /api/files/save` returned 413 from
  BunkerWeb, a 90 MB `POST /api/files/save` returned 200 (container peak about 1 GiB, idle about
  0.35 GiB), and a 50 MB `PATCH /api/files/save` with `Content-Type: application/offset+octet-stream`
  returned 200 (peak about 0.7 GiB).
- Opening, editing and saving a `.docx` through the [filestash-office](../filestash-office/README.md)
  template on two hostnames, with the Compose example above.

Not tested: a WebDAV client on a shared link, Let's Encrypt, and the other storage backends.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
