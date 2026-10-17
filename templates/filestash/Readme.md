# Filestash Template

## Overview

This template proxies Filestash through BunkerWeb with automatic Let's Encrypt certificates, reverse
proxying, a 2g request size limit and custom configuration files.

## Prerequisites

- A running Filestash server that BunkerWeb can reach.
- A public hostname pointing to BunkerWeb.
- Access to the BunkerWeb web UI or service environment variables.

## Setup

1. Import `template.json` (Templates → **Create new template** → **Raw**, then upload or paste it).
2. When the **Missing Custom Configs** dialog appears, upload every file listed in the [Custom
   configs](#custom-configs) section from `configs/`, then save.
3. Assign `USE_TEMPLATE=filestash` to the service, or select **Filestash** in the web UI.
4. Replace `SERVER_NAME`, `EMAIL_LETS_ENCRYPT` and `REVERSE_PROXY_HOST` with values for your
   deployment.
5. Reload BunkerWeb, open the public URL, and test sign-in and the main workflow of the application
   (for example a document, upload or API call).

## Placeholders

The values below are examples. Replace them with your own before using the template:

- `example.com` → your own domain name
- `admin@example.com` → your own email address
- `http://filestash:8334` → the name of the service's container on the shared Docker network, or its
  IP address and port, for example `http://10.0.0.5:8080`
- `replace-with-your-crowdsec-api-key` → the API key generated in CrowdSec (for example with
  `cscli bouncers add <name>`)
- `example-user` → your own username

## Notes

- **Certificates:** the HTTP challenge needs port 80 on BunkerWeb to be reachable from the internet
  for every domain in `SERVER_NAME`.
- **Upstream:** `http://filestash:8334` is a container name and port. It only resolves when
  BunkerWeb and the application share a Docker network (the Compose example joins the external
  `bw-services` network with the alias `filestash`; attach BunkerWeb to the same network); otherwise
  replace it with the IP address and port of the service, for example `http://10.0.0.5:8080`.
- **WebSockets:** enabled on the proxied route.
- **Uploads:** `MAX_CLIENT_SIZE=2g` limits each request, not the total size of a file sent by a
  chunked client. Raise it only if clients need larger single requests.
- **WebDAV:** WebDAV verbs (for example `PROPFIND`, `MKCOL`) are allowed. The application must have
  WebDAV enabled as well; test with a WebDAV client.
- **Bad behaviour:** responses with status 400, 401, 403, 405 and 444 count toward a temporary IP
  ban. Make sure normal clients do not trigger these regularly.
- **Custom configs (`modsec-crs`):** ModSecurity rules loaded before the OWASP Core Rule Set
  (typically rule exclusions / false-positive fixes).
- **CrowdSec:** `127.0.0.1` points to the BunkerWeb container itself. If CrowdSec runs in another
  container or on another host, use its container name or IP address in `CROWDSEC_API` and
  `CROWDSEC_APPSEC_URL`.
- **Anti-bot:** `ANTIBOT_IGNORE_URI` (`^/api/ ^/api2/`) skips the challenge for those paths because
  API clients cannot solve it. Those paths are protected only by the application and the other
  security features.
- **Allowlist:** `WHITELIST_IP` is empty by default. Add only the IP addresses or ranges that may
  bypass the security checks, and review the list regularly.
- **Additional hostnames:** the Compose example also uses `stash.example.com` and
  `office.example.com`. This template only routes `SERVER_NAME`. Create a separate BunkerWeb service
  with its own TLS certificate for every extra hostname that clients must reach, proxy it to the
  matching container, and attach that container to the same Docker network as BunkerWeb. A container
  started with `ssl.enable=false` serves plain HTTP, so proxy it with an `http://` upstream.
- **ModSecurity rule IDs:** rules written for these configs use IDs from the local-use range
  (1–99999), never the ranges reserved for the OWASP Core Rule Set. Keep each template inside its
  own small block of IDs so templates do not collide.

## Settings

### Web service - Front door

Configure server / core settings.

| Setting | Value | Description |
| --- | --- | --- |
| `SERVER_NAME` | `filestash.example.com` | Domain name(s) this service answers for. |

### SSL / Let's Encrypt

Configure ssl / let's encrypt settings.

| Setting | Value | Description |
| --- | --- | --- |
| `AUTO_LETS_ENCRYPT` | `yes` | Whether Let's Encrypt certificates are obtained and renewed automatically. |
| `EMAIL_LETS_ENCRYPT` | `admin@example.com` | Contact email registered with Let's Encrypt. |

### Reverse Proxy

Configure reverse proxy settings.

| Setting | Value | Description |
| --- | --- | --- |
| `USE_REVERSE_PROXY` | `yes` | Whether requests are forwarded to an upstream application. |
| `REVERSE_PROXY_INTERCEPT_ERRORS` | `no` | Whether BunkerWeb replaces upstream error pages with its own error pages. |
| `REVERSE_PROXY_HOST` | `http://filestash:8334` | Upstream application: container name, or IP address and port. |
| `REVERSE_PROXY_WS` | `yes` | Whether WebSocket connections are allowed on the proxied route. |
| `REVERSE_PROXY_BUFFERING` | `no` | Whether BunkerWeb buffers upstream responses before sending them to the client. |
| `REVERSE_PROXY_REQUEST_BUFFERING` | `no` | Whether BunkerWeb buffers the whole client request before forwarding it upstream. |
| `REVERSE_PROXY_READ_TIMEOUT` | `3600s` | Maximum time to wait for the upstream to send data. |
| `REVERSE_PROXY_SEND_TIMEOUT` | `3600s` | Maximum time to wait while sending data to the upstream. |

### Rate Limiting

Configure rate limiting settings.

| Setting | Value | Description |
| --- | --- | --- |
| `LIMIT_CONN_MAX_HTTP1` | `50` | Maximum concurrent HTTP/1 connections per client IP. |
| `LIMIT_CONN_MAX_HTTP2` | `500` | Maximum concurrent HTTP/2 streams per client IP. |
| `LIMIT_CONN_MAX_HTTP3` | `500` | Maximum concurrent HTTP/3 streams per client IP. |
| `LIMIT_REQ_RATE` | `50r/s` | Allowed request rate for the matching URL (for example 15r/s). |
| `LIMIT_REQ_URL_1` | `^/api/files/` | URL path the rate limit applies to. |
| `LIMIT_REQ_RATE_1` | `200r/s` | Allowed request rate for the matching URL (for example 15r/s). |

### Security

Configure security settings.

| Setting | Value | Description |
| --- | --- | --- |
| `USE_ANTIBOT` | `javascript` | Challenge visitors to block bots. |
| `ANTIBOT_IGNORE_URI` | `^/api/ ^/api2/` | URL patterns (regex) that skip the anti-bot challenge, for example API routes. |
| `BAD_BEHAVIOR_STATUS_CODES` | `400 401 403 405 444` | Response status codes that count toward a temporary ban of the client IP. |
| `BAD_BEHAVIOR_THRESHOLD` | `40` | Number of bad responses before the client IP is banned. |
| `USE_CROWDSEC` | `yes` | Whether the CrowdSec integration is enabled. |
| `CROWDSEC_API` | `http://127.0.0.1:8080` | URL of the CrowdSec local API. |
| `CROWDSEC_API_KEY` | `replace-with-your-crowdsec-api-key` | API key generated in CrowdSec for the BunkerWeb bouncer. |
| `CROWDSEC_APPSEC_URL` | `http://127.0.0.1:7422/` | URL of the CrowdSec AppSec component. |

### Performance

Configure performance settings.

| Setting | Value | Description |
| --- | --- | --- |
| `CLIENT_CACHE_ETAG` | `no` | Whether ETag headers are sent for cached files. |
| `GZIP_PROXIED` | `expired no-cache no-store private auth` | Which proxied responses are compressed, based on their caching headers. |
| `MAX_CLIENT_SIZE` | `2g` | Maximum size of a single request body. |
| `SERVE_FILES` | `no` | Whether BunkerWeb serves static files itself (off when proxying an app). |

### Whitelist / Allowlist

Configure whitelist / allowlist settings.

| Setting | Value | Description |
| --- | --- | --- |
| `WHITELIST_IP` |  | IP addresses or ranges that bypass the security checks. |

### Other

Configure other settings.

| Setting | Value | Description |
| --- | --- | --- |
| `KEEP_UPSTREAM_HEADERS` |  | Upstream headers that BunkerWeb passes through instead of overwriting. |
| `PROXY_BUFFERS` | `8 32k` | Number and size of NGINX buffers used for upstream responses. |
| `PROXY_BUFFER_SIZE` | `32k` | Size of the buffer for the first part of the upstream response. |
| `PROXY_BUSY_BUFFERS_SIZE` | `64k` | Limit of buffers busy sending a response to the client. |
| `USE_ROBOTSTXT` | `yes` | Whether BunkerWeb serves a generated robots.txt file. |

### False Positives - False Positives Exclusion config

Modsec CRS Config file to prevent false positives.

Custom configs: `modsec-crs/modsec-crs-exclude-filestash.conf`

### Other settings

Additional settings for this service.

| Setting | Value | Description |
| --- | --- | --- |
| `INTERCEPTED_ERROR_CODES` | `404 405 413 429 500 501 502 503 504` | Upstream status codes replaced by BunkerWeb error pages. |
| `ALLOWED_METHODS` | `GET\|POST\|HEAD\|PUT\|DELETE\|PATCH\|OPTIONS\|PROPFIND\|MKCOL\|COPY\|MOVE\|LOCK\|UNLOCK` | HTTP methods that are accepted; all others are rejected. |

## Custom configs

Files are referenced relative to the template's `configs/` directory and grouped by BunkerWeb config
type.

| Type | File in repository | Reference in `template.json` | Purpose |
| --- | --- | --- | --- |
| `modsec-crs` | `templates/filestash/configs/modsec-crs/modsec-crs-exclude-filestash.conf` | `modsec-crs/modsec-crs-exclude-filestash.conf` | ModSecurity rules loaded before the OWASP Core Rule Set (typically rule exclusions / false-positive fixes). |

## Docker Compose example

The values below are placeholders (domain names such as `stash.example.com` and
`office.example.com`). Replace them with your own values before deploying, and keep the domain names
consistent with `SERVER_NAME` and your TLS routes.

```yaml
services:
  app:
    container_name: filestash
    image: machines/filestash:latest
    restart: always
    environment:
    - APPLICATION_URL=stash.example.com
    - CANARY=true
    - OFFICE_URL=http://wopi_server:9980
    - OFFICE_FILESTASH_URL=http://app:8334
    - OFFICE_REWRITE_URL=https://office.example.com
    ports:
    - "127.0.0.1:8334:8334"
    volumes:
    - filestash:/app/data/state/
    - /root/filestash/plugins:/app/data/state/plugins
    networks:
      - default
      - bw-services
  wopi_server:
    container_name: filestash_wopi
    image: collabora/code:24.04.10.2.1
    restart: always
    environment:
    - "extra_params=--o:ssl.enable=false"
    - "aliasgroup1=https://.*:443"
    command:
    - /bin/bash
    - -c
    - |
         curl -o /usr/share/coolwsd/browser/dist/branding-desktop.css https://gist.githubusercontent.com/mickael-kerjean/bc1f57cd312cf04731d30185cc4e7ba2/raw/d706dcdf23c21441e5af289d871b33defc2770ea/destop.css
         /bin/su -s /bin/bash -c '/start-collabora-online.sh' cool
    user: "1000:1000"
    ports:
    - "127.0.0.1:9980:9980"
volumes:
  filestash: {}

networks:
  bw-services:
    external: true
```

## Validation

Before importing the template:

```bash
jq . template.json
jq -e --arg expected "$(basename "$PWD")" '.id == $expected' template.json
jq -e '(.settings | keys | sort) == ([.steps[] | .settings[]] | sort)' template.json
for f in $(jq -r '.configs[]' template.json); do
  test -f "configs/$f" || echo "missing: $f"
done
```

After applying it:

- Confirm the BunkerWeb scheduler reload completes without a template or unknown-setting error.
- Sign in and test the main workflow of the application (for example an upload, download or API
  call).
- Check BunkerWeb logs for rate-limit, bad-behaviour, and ModSecurity events before changing
  security controls.
- Check BunkerWeb logs for rate-limit, bad-behavior, and ModSecurity events before changing security controls.
