# Projectsend Template

## Overview

This template proxies Projectsend through BunkerWeb with automatic Let's Encrypt certificates, reverse proxying, a 8192m request size limit and custom configuration files.

## Prerequisites

- A running Projectsend server that BunkerWeb can reach.
- A public hostname pointing to BunkerWeb.
- Access to the BunkerWeb web UI or service environment variables.

## Setup

1. Import `template.json` (Templates → **Create new template** → **Raw**, then upload or paste it).
2. When the **Missing Custom Configs** dialog appears, upload every file listed in the [Custom configs](#custom-configs) section from `configs/`, then save.
3. Assign `USE_TEMPLATE=projectsend` to the service, or select **Projectsend** in the web UI.
4. Replace `SERVER_NAME`, `EMAIL_LETS_ENCRYPT` and `REVERSE_PROXY_HOST` with values for your deployment.
5. Reload BunkerWeb, open the public URL, and test sign-in and the main workflow of the application (for example a document, upload or API call).

## Placeholders

The values below are examples. Replace them with your own before using the template:

- `example.com` → your own domain name
- `admin@example.com` → your own email address
- `http://projectsend:80` → the name of the service's container on the shared Docker network, or its IP address and port, for example `http://10.0.0.5:8080`
- `replace-with-your-crowdsec-api-key` → the API key generated in CrowdSec (for example with `cscli bouncers add <name>`)
- `example-user` → your own username
- `openssl rand -hex 24` → generates a long random secret for each variable
- the example addresses `192.0.2.x`, `198.51.100.x` and `203.0.113.x` → the IP addresses or ranges that should be allowed

## Notes

- **Certificates:** the HTTP challenge needs port 80 on BunkerWeb to be reachable from the internet for every domain in `SERVER_NAME`.
- **Upstream:** `http://projectsend:80` is a container name and port. It only resolves when BunkerWeb and the application share a Docker network (the Compose example joins the external `bw-services` network with the alias `projectsend`; attach BunkerWeb to the same network); otherwise replace it with the IP address and port of the service, for example `http://10.0.0.5:8080`.
- **Uploads:** `MAX_CLIENT_SIZE=8192m` limits each request, not the total size of a file sent by a chunked client. Raise it only if clients need larger single requests.
- **Methods:** only the listed HTTP methods are accepted; anything else is rejected before reaching the application.
- **Custom configs (`modsec-crs`):** ModSecurity rules loaded before the OWASP Core Rule Set (typically rule exclusions / false-positive fixes).
- **CrowdSec:** `127.0.0.1` points to the BunkerWeb container itself. If CrowdSec runs in another container or on another host, use its container name or IP address in `CROWDSEC_API` and `CROWDSEC_APPSEC_URL`.
- **Anti-bot:** `ANTIBOT_IGNORE_URI` (`^/build/ ^/api/ ^/share/ ^/uploads ^/zip-downloads`) skips the challenge for those paths because API clients cannot solve it. Those paths are protected only by the application and the other security features.
- **Allowlist:** the addresses in the allowlist are documentation examples. Replace them with the IP addresses or ranges that may bypass the security checks.

## Settings

### Web service - Front door

Configure server / core settings.

| Setting | Value | Description |
| --- | --- | --- |
| `SERVER_NAME` | `projectsend.example.com` | Domain name(s) this service answers for. |

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
| `REVERSE_PROXY_HOST` | `http://projectsend:80` | Upstream application: container name, or IP address and port. |
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
| `USE_ANTIBOT` | `javascript` | Challenge visitors to block bots. |
| `ANTIBOT_IGNORE_URI` | `^/build/ ^/api/ ^/share/ ^/uploads ^/zip-downloads` | URL patterns (regex) that skip the anti-bot challenge, for example API routes. |
| `BAD_BEHAVIOR_THRESHOLD` | `20` | Number of bad responses before the client IP is banned. |
| `USE_CROWDSEC` | `yes` | Enable the CrowdSec integration. |
| `CROWDSEC_API` | `http://127.0.0.1:8080` | URL of the CrowdSec local API. |
| `CROWDSEC_API_KEY` | `replace-with-your-crowdsec-api-key` | API key generated in CrowdSec for the BunkerWeb bouncer. |

### Performance

Configure performance settings.

| Setting | Value | Description |
| --- | --- | --- |
| `CLIENT_CACHE_ETAG` | `no` | Send ETag headers for cached files. |
| `GZIP_PROXIED` | `expired no-cache no-store private auth` | Which proxied responses are compressed, based on their caching headers. |
| `MAX_CLIENT_SIZE` | `8192m` | Maximum size of a single request body. |
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
| `KEEP_UPSTREAM_HEADERS` | `Content-Security-Policy Strict-Transport-Security X-Frame-Options X-Content-Type-Options Referrer-Policy` | Upstream headers that BunkerWeb passes through instead of overwriting. |
| `PROXY_BUFFERS` | `8 32k` | Number and size of NGINX buffers used for upstream responses. |
| `PROXY_BUFFER_SIZE` | `32k` | Size of the buffer for the first part of the upstream response. |
| `PROXY_BUSY_BUFFERS_SIZE` | `64k` | Limit of buffers busy sending a response to the client. |
| `USE_ROBOTSTXT` | `yes` | Serve a robots.txt file generated by BunkerWeb. |

### Fale Positives - False Positives Exclusion config

Modsec CRS Config file to prevent false positives.

Custom configs: `modsec-crs/modsec-crs-exclude-projectsend.conf`

### Other settings

Additional settings for this service.

| Setting | Value | Description |
| --- | --- | --- |
| `CROWDSEC_APPSEC_URL` | `http://127.0.0.1:7422/` | URL of the CrowdSec AppSec component. |
| `INTERCEPTED_ERROR_CODES` | `404 405 413 429 500 501 502 503 504` | Upstream status codes replaced by BunkerWeb error pages. |
| `CONTENT_SECURITY_POLICY_REPORT_ONLY` | `yes` | — |
| `COOKIE_FLAGS_2` | `projectsend_xsrf SameSite=Lax` | Flags (HttpOnly, Secure, SameSite…) added to cookies. |
| `ALLOWED_METHODS` | `GET\|POST\|HEAD\|PUT\|DELETE\|PATCH\|OPTIONS` | HTTP methods that are accepted; all others are rejected. |

## Custom configs

Files are referenced relative to the template's `configs/` directory and grouped by BunkerWeb config type.

| Type | File in repository | Reference in `template.json` | Purpose |
| --- | --- | --- | --- |
| `modsec-crs` | `templates/projectsend/configs/modsec-crs/modsec-crs-exclude-projectsend.conf` | `modsec-crs/modsec-crs-exclude-projectsend.conf` | ModSecurity rules loaded before the OWASP Core Rule Set (typically rule exclusions / false-positive fixes). |

## Docker Compose example

Usernames and passwords below are placeholders. Replace them with your own values before deploying.

```yaml
services:
  app:
    image: projectsend/projectsend:latest
    container_name: projectsend-app
    restart: unless-stopped
    ports:
      - "9999:80"                                # Maps host port 8080 to container port 80
    environment:
      # -- App --
      APP_URL: "https://filo.example.com"     # ASSUMPTION: pick the real hostname (matches SERVER_NAME below)
      TZ: "Europe/Berlin"

      # -- Tells ProjectSend a reverse proxy (BunkerWeb) sits in front. --
      # Safe as "*" here because only BunkerWeb can reach this container's
      # port on the bw-services network below — nobody else can connect
      # directly and spoof X-Forwarded-For.
      TRUSTED_PROXIES: "*"

      # -- Database --
      DB_HOST: "db"
      DB_NAME: "projectsend"
      DB_USER: "example-user"
      DB_PASSWORD: "${DB_PASSWORD:?Set DB_PASSWORD}"
      DB_ROOT_PASSWORD: "${DB_ROOT_PASSWORD:?Set DB_ROOT_PASSWORD}"

      # -- Redis (sessions, cache, queue) --
      REDIS_HOST: "redis"
    volumes:
      # Bind-mounted per the persistence doc: this ONE mount carries both
      # the uploaded files AND storage/.env (APP_KEY). Do not mount just
      # storage/app/files/ — that leaves APP_KEY inside the container.
      - ./projectsend-default.conf:/etc/nginx/http.d/default.conf:ro
      - /srv/projectsend/storage:/var/www/html/storage
    depends_on:
      - db
      - redis
    networks:
      default:
      bw-services:
        aliases:
          - projectsend

  db:
    image: mysql:8.0
    container_name: projectsend-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: "${MYSQL_ROOT_PASSWORD:?Set MYSQL_ROOT_PASSWORD}"
      MYSQL_DATABASE: "projectsend"
      MYSQL_USER: "example-user"
      MYSQL_PASSWORD: "${MYSQL_PASSWORD:?Set MYSQL_PASSWORD}"
    volumes:
      - /srv/projectsend/mysql:/var/lib/mysql

  redis:
    image: redis:7-alpine
    container_name: projectsend-redis
    restart: unless-stopped

networks:
  bw-services:
    external: true
```

Set the required environment variables before starting the stack:

```bash
export DB_PASSWORD="$(openssl rand -hex 24)"
export DB_ROOT_PASSWORD="$(openssl rand -hex 24)"
export MYSQL_ROOT_PASSWORD="$(openssl rand -hex 24)"
export MYSQL_PASSWORD="$(openssl rand -hex 24)"
docker compose config
docker compose up -d
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


## Additional information
For uploads above 9MB you'll need to remove ```env CROWDSEC_APPSEC_URL=http://127.0.0.1:7422/ ``` along with increasing the `MAX_CLIENT_SIZE` but that is on your own risk and you should know what you do.
