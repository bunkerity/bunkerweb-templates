# Filestash Office Template

## Overview

Provision a BunkerWeb configuration for the Collabora Online (WOPI) server that Filestash uses to
edit Office documents. Filestash and Collabora run on two hostnames: users open Filestash, and
Filestash loads the editor in an iframe from the Collabora hostname. This template fronts the
Collabora hostname. The Filestash hostname uses the [filestash](../filestash/README.md) template.

The template proxies the Collabora container on port 9980 with automatic Let's Encrypt
certificates, WebSocket support (Collabora's `/cool/.../ws` connections) and a 50m request size
limit. It allows only `GET`, `POST`, `HEAD` and `OPTIONS`.

## Prerequisites

- A running Collabora server (`collabora/code`) and a Filestash server that BunkerWeb can reach. The
  [Compose example](../filestash/README.md#docker-compose-example) in the filestash README starts
  both.
- A public hostname for Collabora pointing to BunkerWeb, separate from the Filestash hostname.
- Access to the BunkerWeb web UI or service environment variables.

## Setup

1. Import `template.json` (Templates → **Create new template** → **Raw**, then upload or paste it).
2. Assign `USE_TEMPLATE=filestash-office` to the service that fronts Collabora, or select
   **Filestash Collabora (WOPI) host with WebSocket proxy defaults** in the web UI.
3. Replace `SERVER_NAME`, `EMAIL_LETS_ENCRYPT` and `REVERSE_PROXY_HOST` with values for your
   deployment.
4. In the Filestash container set `OFFICE_REWRITE_URL` to `https://` plus this `SERVER_NAME`, as in
   the Compose example.
5. Reload BunkerWeb, open a document from Filestash, and check that the editor loads.

## Placeholders

The values below are examples. Replace them with your own before using the template:

- `example.com` → your own domain name
- `admin@example.com` → your own email address
- `http://filestash-office:9980` → the name of the Collabora container on the shared Docker network
  and its port (9980), or the IP address and port of the server, for example `http://10.0.0.5:9980`

## Notes

- **Certificates:** the HTTP challenge needs port 80 on BunkerWeb to be reachable from the internet
  for every domain in `SERVER_NAME`.
- **Upstream:** `http://filestash-office:9980` is a container name and port. It only resolves when
  BunkerWeb and Collabora share a Docker network (the Compose example joins the external
  `bw-services` network and gives the Collabora container the alias `filestash-office`; attach
  BunkerWeb to the same network); otherwise replace it with the IP address and port of the
  service, for example `http://10.0.0.5:9980`. The alias is on the Collabora container, not on
  Filestash.
- **WebSockets:** enabled on the proxied route. Collabora keeps one WebSocket per open document.
- **Uploads:** `MAX_CLIENT_SIZE=50m` limits each request. Raise it only if users insert larger
  files into documents.
- **Methods:** only the listed HTTP methods are accepted; anything else is rejected before reaching
  Collabora.
- **TLS:** Collabora runs with `ssl.enable=false` because BunkerWeb terminates TLS, so the upstream
  is `http://`.
- **Upstream headers:** `KEEP_UPSTREAM_HEADERS` keeps `Content-Security-Policy` and
  `X-Frame-Options` from Collabora. Collabora sets the `frame-ancestors` policy that lets the
  Filestash hostname embed the editor, and BunkerWeb's default `frame-ancestors 'self'` would
  block it.
- **Compression:** `USE_GZIP` is on so that `GZIP_PROXIED` takes effect.
- **Allowlist:** `WHITELIST_IP` is not set by the template. Set it on the service only for the IP
  addresses or ranges that may bypass the security checks, and review the list regularly.
- **WOPI hosts:** the `aliasgroup1` variable of the Collabora container lists the WOPI hosts it
  accepts. Keep it limited to the Filestash address, as in the Compose example, so other sites
  cannot use your Collabora server.

## Settings

### Web service - Front service

Point the template at the hostname that fronts Collabora.

| Setting              | Value                          | Description                                                                |
| -------------------- | ------------------------------ | -------------------------------------------------------------------------- |
| `SERVER_NAME`        | `filestash-office.example.com` | Domain name(s) this service answers for.                                   |
| `AUTO_LETS_ENCRYPT`  | `yes`                          | Whether Let's Encrypt certificates are obtained and renewed automatically. |
| `EMAIL_LETS_ENCRYPT` | `admin@example.com`            | Contact email registered with Let's Encrypt.                               |

### Web service - Upstream server

Direct the reverse proxy at the Collabora container and keep its WebSockets alive.

| Setting                           | Value                          | Description                                                                       |
| --------------------------------- | ------------------------------ | --------------------------------------------------------------------------------- |
| `USE_REVERSE_PROXY`               | `yes`                          | Whether requests are forwarded to an upstream application.                        |
| `REVERSE_PROXY_HOST`              | `http://filestash-office:9980` | Upstream application: Collabora container name and port, or IP address and port.  |
| `REVERSE_PROXY_WS`                | `yes`                          | Whether WebSocket connections are allowed on the proxied route.                   |
| `REVERSE_PROXY_BUFFERING`         | `no`                           | Whether BunkerWeb buffers upstream responses before sending them to the client.   |
| `REVERSE_PROXY_REQUEST_BUFFERING` | `no`                           | Whether BunkerWeb buffers the whole client request before forwarding it upstream. |
| `REVERSE_PROXY_INTERCEPT_ERRORS`  | `no`                           | Whether BunkerWeb replaces upstream error pages with its own error pages.         |
| `REVERSE_PROXY_READ_TIMEOUT`      | `3600s`                        | Maximum time to wait for the upstream to send data.                               |
| `REVERSE_PROXY_SEND_TIMEOUT`      | `3600s`                        | Maximum time to wait while sending data to the upstream.                          |
| `PROXY_BUFFERS`                   | `8 32k`                        | Number and size of NGINX buffers used for upstream responses.                     |
| `PROXY_BUFFER_SIZE`               | `32k`                          | Size of the buffer for the first part of the upstream response.                   |
| `PROXY_BUSY_BUFFERS_SIZE`         | `64k`                          | Limit of buffers busy sending a response to the client.                           |

### HTTP - Collabora specific settings

Collabora only needs GET, POST, HEAD and OPTIONS and modest request bodies.

| Setting           | Value                | Description                                                              |
| ----------------- | -------------------- | ------------------------------------------------------------------------ |
| `ALLOWED_METHODS` | `GET\|POST\|HEAD\|OPTIONS` | HTTP methods that are accepted; all others are rejected.           |
| `MAX_CLIENT_SIZE` | `50m`                | Maximum size of a single request body.                                   |
| `SERVE_FILES`     | `no`                 | Whether BunkerWeb serves static files itself (off when proxying an app). |

### Security - Rate and connection limits

The editor loads many assets and keeps WebSockets open, so the limits sit above the defaults.

| Setting                | Value   | Description                                                   |
| ---------------------- | ------- | ------------------------------------------------------------- |
| `LIMIT_CONN_MAX_HTTP1` | `50`    | Maximum concurrent HTTP/1 connections per client IP.          |
| `LIMIT_CONN_MAX_HTTP2` | `500`   | Maximum concurrent HTTP/2 streams per client IP.              |
| `LIMIT_CONN_MAX_HTTP3` | `500`   | Maximum concurrent HTTP/3 streams per client IP.              |
| `LIMIT_REQ_RATE`       | `25r/s` | Allowed request rate for the default URL (for example 15r/s). |

### Security - Bad behavior

Ban clients that repeatedly trigger error responses.

| Setting                  | Value | Description                                         |
| ------------------------ | ----- | --------------------------------------------------- |
| `BAD_BEHAVIOR_THRESHOLD` | `20`  | Number of bad responses before the client IP is banned. |

### Security - Headers and exposure

Keep the frame headers Collabora sets so Filestash can embed the editor.

| Setting                   | Value                                     | Description                                                            |
| ------------------------- | ----------------------------------------- | ---------------------------------------------------------------------- |
| `KEEP_UPSTREAM_HEADERS`   | `Content-Security-Policy X-Frame-Options` | Upstream headers that BunkerWeb passes through instead of overwriting. |
| `INTERCEPTED_ERROR_CODES` | `404 405 413 429 500 501 502 503 504`     | Upstream status codes replaced by BunkerWeb error pages.               |
| `USE_ROBOTSTXT`           | `yes`                                     | Whether BunkerWeb serves a generated robots.txt file.                  |

### Performance - Client optimization

Compress proxied responses; Collabora serves its own files.

| Setting        | Value                                    | Description                                                             |
| -------------- | ---------------------------------------- | ----------------------------------------------------------------------- |
| `USE_GZIP`     | `yes`                                    | Whether BunkerWeb compresses responses with gzip.                       |
| `GZIP_PROXIED` | `expired no-cache no-store private auth` | Which proxied responses are compressed, based on their caching headers. |

## Custom configs

This template ships no custom config files.

## Docker Compose example

The [filestash README](../filestash/README.md#docker-compose-example) holds the full Compose file.
It starts Filestash and Collabora together, publishes no port, joins both to the external
`bw-services` network, and gives the Collabora container the alias `filestash-office` that
`REVERSE_PROXY_HOST` uses. Replace the placeholder hostnames in it with your own before deploying.

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
- Open an Office document from Filestash and check that the editor loads and saves.
- Check BunkerWeb logs for rate-limit, bad behavior, and ModSecurity events before changing
  security controls.
