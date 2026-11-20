<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/filestash.svg" alt="Filestash logo" width="96" height="96" />
</p>

<h1 align="center">Filestash Office</h1>

<p align="center">Collabora (WOPI) host template with WebSocket proxy defaults</p>

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

Provision a BunkerWeb configuration for the Collabora Online (WOPI) server that Filestash uses to
edit Office documents. Filestash and Collabora run on two hostnames: users open Filestash, and
Filestash loads the editor in an iframe from the Collabora hostname. This template fronts the
Collabora hostname. The Filestash hostname uses the [filestash](../filestash/README.md) template,
and the editor only works when the settings in [Both hostnames](#both-hostnames) are in place on
both sides.

The template proxies the Collabora container on port 9980 with automatic Let's Encrypt
certificates, WebSocket support (Collabora's `/cool/.../ws` connections) and a 50m request size
limit. It allows only `GET`, `POST`, `HEAD` and `OPTIONS`.

## At a glance

| Fact        | Value                          |
| ----------- | ------------------------------ |
| Template id | `filestash-office`             |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Collabora `24.04.10.2.1` |
| Configs     | 0                              |
| Upstream    | `http://filestash-office:9980` |

## Prerequisites

- A running Collabora server (`collabora/code`) and a Filestash server that BunkerWeb can reach. The
  [Compose example](../filestash/README.md#docker-compose-example) in the filestash README starts
  both and pins the Collabora image `collabora/code:24.04.10.2.1`. Tested with that image and
  Filestash `machines/filestash:latest` (build `b6fe63f4f01e`): a `.docx` opens, is edited and
  saved through BunkerWeb on two hostnames.
- A public hostname for Collabora pointing to BunkerWeb, separate from the Filestash hostname.
- Access to the BunkerWeb web UI.

## Install

1. Import `template.json` (Templates → **Create template** → **Raw**, then upload or paste it).
2. Create the service that fronts Collabora and pick **Filestash-office** in the **Templates** menu
   above the steps (its description reads "Filestash Collabora (WOPI) host with WebSocket proxy
   defaults"). Plugin users assign `USE_TEMPLATE=filestash-office` instead.
3. Replace `SERVER_NAME`, `EMAIL_LETS_ENCRYPT` and `REVERSE_PROXY_HOST` with values for your
   deployment. Keep the example e-mail only for a lab.
4. In the Filestash container set `OFFICE_REWRITE_URL` to `https://` plus this `SERVER_NAME`, as in
   the Compose example.
5. Set the three settings of [Both hostnames](#both-hostnames): two on the Collabora container and
   one on the Filestash service in BunkerWeb.
6. Save the service: BunkerWeb applies it on its own. Open a document from Filestash and check
   that the editor loads.

### Both hostnames

The editor needs all three of these. Each one fixes a different failure of the same loading screen:

| Where | Setting | Without it |
| ----- | ------- | ---------- |
| Collabora container, `extra_params` | `--o:net.frame_ancestors=https://<Filestash hostname>` | The browser refuses to show the editor frame (`frame-ancestors` in the console). |
| Collabora container, `extra_params` | `--o:ssl.termination=true` (next to `--o:ssl.enable=false`) | The editor opens but the document stays blank: Collabora announces `ws://` addresses and the browser refuses them on an HTTPS page. |
| Filestash service in BunkerWeb (template [filestash](../filestash/README.md)) | `CONTENT_SECURITY_POLICY` with `form-action 'self' https://<this SERVER_NAME>` | The browser refuses the form that opens the editor (`form-action` in the console). |

The [Compose example](../filestash/README.md#docker-compose-example) already carries the first two.

For both import methods (web UI and plugin), see the repository [installation guide](../../README.md#install).

### Placeholders

The values below are examples. Replace them with your own before using the template:

- `example.com` → your own domain name
- `admin@example.com` → your own email address
- `http://filestash-office:9980` → the name of the Collabora container on the shared Docker network
  and its port (9980), or the IP address and port of the server, for example `http://10.0.0.5:9980`

## Settings

### Notes

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
- **TLS:** Collabora runs with `ssl.enable=false` and `ssl.termination=true` because BunkerWeb
  terminates TLS: the first makes it serve plain HTTP, so the upstream is `http://`, the second
  makes it hand the browser `wss://` addresses. Without `ssl.termination=true` the editor opens but
  the document never loads.
- **Upstream headers:** `KEEP_UPSTREAM_HEADERS` keeps `Content-Security-Policy` and
  `X-Frame-Options` from Collabora. Collabora lists the Filestash hostname in its `frame-ancestors`
  policy only when you set `--o:net.frame_ancestors=https://<Filestash hostname>` on the Collabora
  container, as in the Compose example. BunkerWeb's default `frame-ancestors 'self'` would replace
  that policy, so the template keeps Collabora's header. On a few pages Collabora sends no policy
  and BunkerWeb's default applies; the browser console then logs a blocked frame of the Collabora
  root page, which does not affect the editor.
- **Compression:** `USE_GZIP` is on so that `GZIP_PROXIED` takes effect.
- **Allowlist:** `WHITELIST_IP` is not set by the template. Set it on the service only for the IP
  addresses or ranges that may bypass the security checks, and review the list regularly.
- **WOPI hosts:** the `aliasgroup1` variable of the Collabora container lists the WOPI hosts it
  accepts. Keep it limited to the Filestash address, as in the Compose example, so other sites
  cannot use your Collabora server.

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
| `INTERCEPTED_ERROR_CODES` | `404 405 413 429 500 501 502 503 504`     | Status codes answered with BunkerWeb error pages when BunkerWeb itself generates them (for example 405 from `ALLOWED_METHODS`, 413, 429). Upstream errors pass through unchanged because `REVERSE_PROXY_INTERCEPT_ERRORS` is `no`. |
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

Run these from the repository root before importing the template:

```bash
(
  set -eu
  t=templates/filestash-office
  jq . "$t/template.json"
  jq -e --arg expected "$(basename "$t")" '.id == $expected' "$t/template.json"
  jq -e '(.settings | keys | sort) == ([.steps[] | .settings[]] | sort)' "$t/template.json"
  for f in $(jq -r '.configs[]' "$t/template.json"); do
    test -f "$t/configs/$f" || { echo "missing: $f"; exit 1; }
  done
)
```

After applying it:

- Confirm the BunkerWeb scheduler reload completes without a template or unknown-setting error.
- Open an Office document from Filestash, type in it and save it with `Ctrl+S`: the editor loads
  and the file in Filestash holds your text.
- Check BunkerWeb logs for rate-limit, bad behavior, and ModSecurity events before changing
  security controls.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
