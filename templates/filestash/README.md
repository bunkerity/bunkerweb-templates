# Filestash Template

## Overview

Provision a BunkerWeb configuration for [Filestash](https://www.filestash.app/), the web file
manager that connects to S3, SFTP, WebDAV and other storage backends. The template proxies
Filestash with automatic Let's Encrypt certificates, WebSocket support, a 2g request size limit,
the WebDAV verbs and a ModSecurity exclusion file for the web editor and for uploads.

Editing Office documents inside Filestash needs a Collabora (WOPI) server on its own hostname. The
[filestash-office](../filestash-office/README.md) template fronts that server and uses the same
Compose example, shown further down on this page.

## Prerequisites

- A running Filestash server that BunkerWeb can reach.
- A public hostname pointing to BunkerWeb.
- Access to the BunkerWeb web UI or service environment variables.

## Setup

1. Import `template.json` (Templates → **Create new template** → **Raw**, then upload or paste it).
2. When the **Missing Custom Configs** dialog appears, upload every file listed in the
   [Custom configs](#custom-configs) section from `configs/`, then save.
3. Assign `USE_TEMPLATE=filestash` to the service, or select **Filestash file manager with
   WebDAV-aware proxy defaults** in the web UI.
4. Replace `SERVER_NAME`, `EMAIL_LETS_ENCRYPT` and `REVERSE_PROXY_HOST` with values for your
   deployment.
5. Reload BunkerWeb, open the public URL, and test sign-in, an upload, a download and the web
   editor.

## Placeholders

The values below are examples. Replace them with your own before using the template:

- `example.com` → your own domain name
- `admin@example.com` → your own email address
- `http://filestash:8334` → the name of the service's container on the shared Docker network and
  the port Filestash listens on (8334), or the IP address and port of the server, for example
  `http://10.0.0.5:8334`

## Notes

- **Certificates:** the HTTP challenge needs port 80 on BunkerWeb to be reachable from the internet
  for every domain in `SERVER_NAME`.
- **Upstream:** `http://filestash:8334` is a container name and port. It only resolves when
  BunkerWeb and the application share a Docker network (the Compose example joins the external
  `bw-services` network with the alias `filestash`; attach BunkerWeb to the same network);
  otherwise replace it with the IP address and port of the service, for example
  `http://10.0.0.5:8334`.
- **WebSockets:** enabled on the proxied route.
- **Uploads:** `MAX_CLIENT_SIZE=2g` limits each request, not the total size of a file. Filestash
  sends large files in several chunk requests. Raise the limit only if clients need larger single
  requests.
- **WebDAV:** WebDAV verbs (for example `PROPFIND`, `MKCOL`) are allowed. The application must have
  WebDAV enabled as well; test with a WebDAV client.
- **Bad behavior:** responses with status 400, 401, 403, 405 and 444 count toward a temporary IP
  ban. Make sure normal clients do not trigger these regularly.
- **Anti-bot:** `ANTIBOT_IGNORE_URI` (`^/api/`) skips the challenge for the Filestash API because
  API clients cannot solve it. That path is protected only by the application and the other
  security features. A WebDAV client that connects through another path cannot solve the challenge
  either: add its path to `ANTIBOT_IGNORE_URI`.
- **Upstream headers:** `KEEP_UPSTREAM_HEADERS` keeps two headers Filestash sets itself. The
  `Content-Security-Policy` locks down the files Filestash serves for preview, so an uploaded HTML
  or SVG file cannot run scripts. `X-Frame-Options` is sent by Filestash unless the admin allows
  embedding in the admin console. BunkerWeb's defaults replace them otherwise.
- **Compression:** `USE_GZIP` is on so that `GZIP_PROXIED` takes effect.
- **Allowlist:** `WHITELIST_IP` is not set by the template. Set it on the service only for the IP
  addresses or ranges that may bypass the security checks, and review the list regularly.
- **ModSecurity rule IDs:** rules written for these configs use IDs from the local-use range
  (1 to 99999), never the ranges reserved for the OWASP Core Rule Set. This template uses 75001 to
  75003. Keep each template inside its own small block of IDs so templates do not collide.

## Settings

### Web service - Front service

Point the template at your Filestash hostname.

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

Allow the WebDAV verbs and large uploads, and let Filestash serve its own files.

| Setting           | Value                                                                       | Description                                              |
| ----------------- | --------------------------------------------------------------------------- | -------------------------------------------------------- |
| `ALLOWED_METHODS` | `GET\|POST\|HEAD\|PUT\|DELETE\|PATCH\|OPTIONS\|PROPFIND\|MKCOL\|COPY\|MOVE\|LOCK\|UNLOCK` | HTTP methods that are accepted; all others are rejected. |
| `MAX_CLIENT_SIZE` | `2g`                                                                        | Maximum size of a single request body.                   |
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

Keep the security headers Filestash sets itself on served files and pages.

| Setting                   | Value                                   | Description                                                            |
| ------------------------- | --------------------------------------- | ---------------------------------------------------------------------- |
| `KEEP_UPSTREAM_HEADERS`   | `Content-Security-Policy X-Frame-Options` | Upstream headers that BunkerWeb passes through instead of overwriting. |
| `INTERCEPTED_ERROR_CODES` | `404 405 413 429 500 501 502 503 504`   | Upstream status codes replaced by BunkerWeb error pages.               |
| `USE_ROBOTSTXT`           | `yes`                                   | Whether BunkerWeb serves a generated robots.txt file.                  |

### Performance - Client optimization

Compress proxied responses; Filestash serves its own files.

| Setting        | Value                                    | Description                                                             |
| -------------- | ---------------------------------------- | ----------------------------------------------------------------------- |
| `USE_GZIP`     | `yes`                                    | Whether BunkerWeb compresses responses with gzip.                       |
| `GZIP_PROXIED` | `expired no-cache no-store private auth` | Which proxied responses are compressed, based on their caching headers. |

### Security - WAF tuning

Allow the editor save and the uploads that the CRS content type check rejects, one request at a
time.

Custom configs: `modsec-crs/filestash_false_positives.conf`

## Custom configs

Files are referenced relative to the template's `configs/` directory and grouped by BunkerWeb config
type.

| Type         | File in repository                                                      | Reference in `template.json`                  | Purpose                                                                  |
| ------------ | ----------------------------------------------------------------------- | --------------------------------------------- | ------------------------------------------------------------------------ |
| `modsec-crs` | `templates/filestash/configs/modsec-crs/filestash_false_positives.conf` | `modsec-crs/filestash_false_positives.conf`   | Removes CRS rule 920420 for the editor save and the upload requests only, plus 920340 and 920640 for the single request upload. |

CRS rule 920420 rejects a request whose `Content-Type` is not in the CRS allowed list. Filestash
sends three requests that hit it. Each rule matches one method and one anchored path, so the rest
of the API keeps the full CRS. Rule `75002` also removes 920340 and 920640 (`Content-Type` header
missing from a request with a body), because the browser sends a file with an unknown MIME type, for
example a small `.bin`, with no `Content-Type` at all:

| Rule ID | Request                    | `Content-Type` sent by the browser                         |
| ------- | -------------------------- | ---------------------------------------------------------- |
| `75001` | `POST /api/files/cat`      | `text/plain;charset=UTF-8` (web editor save)               |
| `75002` | `POST /api/files/save`     | the file's own type, for example `application/pdf`, or no header |
| `75003` | `PATCH /api/files/save`    | `application/offset+octet-stream` (chunked upload)         |

The CRS does not inspect these bodies either way, because no body processor is armed for their
types. The global allowed content type list is not changed.

## Docker Compose example

This example runs Filestash and the Collabora server used by the
[filestash-office](../filestash-office/README.md) template. Neither container publishes a port:
BunkerWeb reaches them over the external `bw-services` network, and a published port would bypass
TLS, the WAF and the rate limits. Replace `filestash.example.com` and
`filestash-office.example.com` with your own hostnames, and keep them identical to the
`SERVER_NAME` of the matching BunkerWeb service. Skip the `wopi_server` service if you do not edit
Office documents.

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
      - "extra_params=--o:ssl.enable=false"
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
- **`ssl.enable=false`:** Collabora serves plain HTTP because BunkerWeb terminates TLS, so proxy
  it with an `http://` upstream.

## Validation

Before importing the template:

```bash
jq . template.json
jq -e --arg expected "$(basename "$PWD")" '.id == $expected' template.json
jq -e '(.settings | keys | sort) == ([.steps[] | .settings[]] | sort)' template.json
for f in $(jq -r '.configs[]' template.json); do
  test -f "configs/$f" || { echo "missing: $f"; exit 1; }
done
```

After applying it:

- Confirm the BunkerWeb scheduler reload completes without a template or unknown-setting error.
- Sign in, save a file in the web editor, then upload a small file and a file larger than a few
  megabytes, and download them again.
- Check BunkerWeb logs for rate-limit, bad behavior, and ModSecurity events before changing
  security controls.
