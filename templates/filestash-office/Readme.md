# Filestash-office Template

## Overview

This template proxies Filestash-office through BunkerWeb with automatic Let's Encrypt certificates, reverse proxying and a 50m request size limit.

## Prerequisites

- A running Filestash-office server that BunkerWeb can reach.
- A public hostname pointing to BunkerWeb.
- Access to the BunkerWeb web UI or service environment variables.

## Setup

1. Import `template.json` (Templates → **Create new template** → **Raw**, then upload or paste it).
2. Assign `USE_TEMPLATE=filestash-office` to the service, or select **Filestash-office** in the web UI.
3. Replace `SERVER_NAME`, `EMAIL_LETS_ENCRYPT` and `REVERSE_PROXY_HOST` with values for your deployment.
4. Reload BunkerWeb, open the public URL, and test sign-in and the main workflow of the application (for example a document, upload or API call).

## Placeholders

The values below are examples. Replace them with your own before using the template:

- `example.com` → your own domain name
- `admin@example.com` → your own email address
- `http://filestash-office:8334` → the name of the service's container on the shared Docker network, or its IP address and port, for example `http://10.0.0.5:8080`
- `replace-with-your-crowdsec-api-key` → the API key generated in CrowdSec (for example with `cscli bouncers add <name>`)
- `example-user` → your own username
- the example addresses `192.0.2.x`, `198.51.100.x` and `203.0.113.x` → the IP addresses or ranges that should be allowed

## Notes

- **Certificates:** the HTTP challenge needs port 80 on BunkerWeb to be reachable from the internet for every domain in `SERVER_NAME`.
- **Upstream:** `http://filestash-office:8334` is a container name and port. It only resolves when BunkerWeb and the application share a Docker network (the Compose example joins the external `bw-services` network with the alias `filestash-office`; attach BunkerWeb to the same network); otherwise replace it with the IP address and port of the service, for example `http://10.0.0.5:8080`.
- **WebSockets:** enabled on the proxied route.
- **Uploads:** `MAX_CLIENT_SIZE=50m` limits each request, not the total size of a file sent by a chunked client. Raise it only if clients need larger single requests.
- **Methods:** only the listed HTTP methods are accepted; anything else is rejected before reaching the application.
- **CrowdSec:** `127.0.0.1` points to the BunkerWeb container itself. If CrowdSec runs in another container or on another host, use its container name or IP address in `CROWDSEC_API` and `CROWDSEC_APPSEC_URL`.
- **Allowlist:** the addresses in the allowlist are documentation examples. Replace them with the IP addresses or ranges that may bypass the security checks.

## Settings

### Web service - Front door

Configure server / core settings.

| Setting | Value | Description |
| --- | --- | --- |
| `SERVER_NAME` | `filestash-office.example.com` | Domain name(s) this service answers for. |

### SSL / Let's Encrypt

Configure ssl / let's encrypt settings.

| Setting | Value | Description |
| --- | --- | --- |
| `AUTO_LETS_ENCRYPT` | `yes` | Obtain and renew Let's Encrypt certificates automatically. |
| `EMAIL_LETS_ENCRYPT` | `admin@example.com` | Contact email registered with Let's Encrypt. |

### Reverse Proxy

Configure reverse proxy settings.

| Setting | Value | Description |
| --- | --- | --- |
| `USE_REVERSE_PROXY` | `yes` | Forward requests to an upstream application. |
| `REVERSE_PROXY_INTERCEPT_ERRORS` | `no` | Replace upstream error pages with BunkerWeb error pages. |
| `REVERSE_PROXY_HOST` | `http://filestash-office:8334` | Upstream application: container name, or IP address and port. |
| `REVERSE_PROXY_WS` | `yes` | Allow WebSocket connections on the proxied route. |
| `REVERSE_PROXY_BUFFERING` | `no` | Buffer upstream responses in BunkerWeb before sending them to the client. |
| `REVERSE_PROXY_REQUEST_BUFFERING` | `no` | Buffer the whole client request before forwarding it upstream. |
| `REVERSE_PROXY_READ_TIMEOUT` | `3600s` | Maximum time to wait for the upstream to send data. |
| `REVERSE_PROXY_SEND_TIMEOUT` | `3600s` | Maximum time to wait while sending data to the upstream. |

### Rate Limiting

Configure rate limiting settings.

| Setting | Value | Description |
| --- | --- | --- |
| `LIMIT_CONN_MAX_HTTP1` | `50` | Maximum concurrent HTTP/1 connections per client IP. |
| `LIMIT_CONN_MAX_HTTP2` | `500` | Maximum concurrent HTTP/2 streams per client IP. |
| `LIMIT_CONN_MAX_HTTP3` | `500` | Maximum concurrent HTTP/3 streams per client IP. |
| `LIMIT_REQ_RATE` | `25r/s` | Allowed request rate for the matching URL (for example 15r/s). |

### Security

Configure security settings.

| Setting | Value | Description |
| --- | --- | --- |
| `BAD_BEHAVIOR_THRESHOLD` | `20` | Number of bad responses before the client IP is banned. |
| `USE_CROWDSEC` | `yes` | Enable the CrowdSec integration. |
| `CROWDSEC_API` | `http://127.0.0.1:8080` | URL of the CrowdSec local API. |
| `CROWDSEC_API_KEY` | `replace-with-your-crowdsec-api-key` | API key generated in CrowdSec for the BunkerWeb bouncer. |
| `CROWDSEC_APPSEC_URL` | `http://127.0.0.1:7422/` | URL of the CrowdSec AppSec component. |

### Performance

Configure performance settings.

| Setting | Value | Description |
| --- | --- | --- |
| `CLIENT_CACHE_ETAG` | `no` | Send ETag headers for cached files. |
| `GZIP_PROXIED` | `expired no-cache no-store private auth` | Which proxied responses are compressed, based on their caching headers. |
| `MAX_CLIENT_SIZE` | `50m` | Maximum size of a single request body. |
| `SERVE_FILES` | `no` | Serve static files from BunkerWeb itself (off when proxying an app). |

### Whitelist / Allowlist

Configure whitelist / allowlist settings.

| Setting | Value | Description |
| --- | --- | --- |
| `WHITELIST_IP` | `192.0.2.10 198.51.100.10 203.0.113.10` | IP addresses or ranges that bypass the security checks. |

### Other

Configure other settings.

| Setting | Value | Description |
| --- | --- | --- |
| `KEEP_UPSTREAM_HEADERS` | `Content-Security-Policy X-Frame-Options` | Upstream headers that BunkerWeb passes through instead of overwriting. |
| `PROXY_BUFFERS` | `8 32k` | Number and size of NGINX buffers used for upstream responses. |
| `PROXY_BUFFER_SIZE` | `32k` | Size of the buffer for the first part of the upstream response. |
| `PROXY_BUSY_BUFFERS_SIZE` | `64k` | Limit of buffers busy sending a response to the client. |
| `USE_ROBOTSTXT` | `yes` | Serve a robots.txt file generated by BunkerWeb. |

### Other settings

Additional settings for this service.

| Setting | Value | Description |
| --- | --- | --- |
| `INTERCEPTED_ERROR_CODES` | `404 405 413 429 500 501 502 503 504` | Upstream status codes replaced by BunkerWeb error pages. |
| `ALLOWED_METHODS` | `GET\|POST\|HEAD\|OPTIONS` | HTTP methods that are accepted; all others are rejected. |

## Docker Compose example

Usernames and passwords below are placeholders. Replace them with your own values before deploying.

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
    - "8334:8334"
    volumes:
    - filestash:/app/data/state/
    - /root/filestash/plugins:/app/data/state/plugins
    networks:
      default:
      bw-services:
        aliases:
          - filestash-office
  wopi_server:
    container_name: filestash_wopi
    image: collabora/code:24.04.10.2.1
    restart: always
    environment:
    - "extra_params=--o:ssl.enable=false"
    - aliasgroup1="https://.*:443"
    command:
    - /bin/bash
    - -c
    - |
         curl -o /usr/share/coolwsd/browser/dist/branding-desktop.css https://gist.githubusercontent.com/mickael-kerjean/bc1f57cd312cf04731d30185cc4e7ba2/raw/d706dcdf23c21441e5af289d871b33defc2770ea/destop.css
         /bin/su -s /bin/bash -c '/start-collabora-online.sh' cool
    user: "1000:1000"
    ports:
    - "9980:9980"
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
- Sign in and test the main workflow of the application (for example an upload, download or API call).
- Check BunkerWeb logs for rate-limit, bad-behavior, and ModSecurity events before changing security controls.
