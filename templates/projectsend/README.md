# ProjectSend Template

## Overview

This template puts [ProjectSend](https://www.projectsend.org/) v2 behind BunkerWeb with automatic
Let's Encrypt certificates, a reverse proxy tuned for streamed uploads and downloads, and a
ModSecurity exclusion for the chunked upload endpoint. ProjectSend v2 uploads files in resumable
20 MiB parts, so the request size limit is `MAX_CLIENT_SIZE=100m`, the same value ProjectSend's own
nginx uses.

## Prerequisites

- A running ProjectSend v2 container (`projectsend/projectsend:2`) that BunkerWeb can reach. The
  image listens on port 80.
- A public hostname pointing to BunkerWeb.
- Access to the BunkerWeb web UI or service environment variables.

## Setup

1. Import `template.json` (Templates, **Create new template**, **Raw**, then upload or paste it).
2. When the **Missing Custom Configs** dialog appears, upload every file listed in the
   [Custom configs](#custom-configs) section from `configs/`, then save.
3. Assign `USE_TEMPLATE=projectsend` to the service, or select **ProjectSend** in the web UI.
4. Replace `SERVER_NAME`, `EMAIL_LETS_ENCRYPT` and `REVERSE_PROXY_HOST` with values for your
   deployment.
5. Reload BunkerWeb, open the public URL, then test sign-in, a small upload, a file larger than
   20 MiB (it is sent in several parts) and a public share link.

## Placeholders

The values below are examples. Replace them with your own before using the template:

- `example.com` is your own domain name.
- `admin@example.com` is your own email address.
- `http://projectsend:80` is the name of the ProjectSend container on the shared Docker network
  and its port, or an IP address and port, for example `http://10.0.0.5:80`.
- `change-me-database` and `change-me-root` (Compose example) are secrets you generate, for
  example with `openssl rand -hex 24`.

## Notes

- **Certificates:** the HTTP challenge needs port 80 on BunkerWeb to be reachable from the
  internet for every domain in `SERVER_NAME`.
- **Upstream:** `http://projectsend:80` is a container name and port. It only resolves when
  BunkerWeb and the application share a Docker network (the Compose example joins the external
  `bw-services` network with the alias `projectsend`; attach BunkerWeb to the same network).
  Otherwise use the IP address and port of the service.
- **Uploads:** ProjectSend sends each file as `PUT /uploads/{session}/parts/{part}` requests of
  `upload_part_size_mb` (20 MiB by default) and accepts a part up to twice that size.
  `MAX_CLIENT_SIZE=100m` limits each request, not the size of a file, so keep it at least twice
  the part size. It also sets the ModSecurity request body limit, so do not raise it far beyond
  what you need.
- **Methods:** `ALLOWED_METHODS` includes `PUT` (upload parts), `PATCH` and `DELETE` (file
  management). Anything else is rejected before reaching the application.
- **Custom configs (`modsec-crs`):** ModSecurity rules loaded before the OWASP Core Rule Set. The
  shipped file removes three protocol rules for the upload part request only, see
  [ModSecurity exclusion](#modsecurity-exclusion).
- **Anti-bot:** `ANTIBOT_IGNORE_URI`
  (`^/build/ ^/api/ ^/s/ ^/uploads(?:/|$) ^/zip-downloads(?:/|$)`) skips the challenge for static
  assets, the token API, public share links (`/s/{token}`), the upload endpoints and zip
  downloads, because API clients and download tools cannot solve it. Those paths are protected
  only by the application and the other security features.
- **CrowdSec:** not part of this template. Add `USE_CROWDSEC=yes` with a reachable CrowdSec API
  and your own API key on the service if you run CrowdSec.
- **Cookies:** `COOKIE_FLAGS_2` sets `SameSite=Lax` on `projectsend_xsrf`, the CSRF cookie the web
  app reads.

## ModSecurity exclusion

The browser uploader (Uppy) sends each part as a raw `PUT` body with no `Content-Type` header, and
the signed URL carries no header either. CRS 4 therefore scores rules `920340` and `920640`
(Content-Type missing on a request with a body). Clients that send `application/octet-stream` on
the part trigger `920420` (Content-Type not allowed by policy) instead.

`configs/modsec-crs/projectsend_false_positives.conf` removes these three rules only when the
method is `PUT` and the path matches `^/(?:api/v1/)?uploads/[^/]+/parts/[0-9]+$`, after
`t:normalizePath`. Every other request, including `GET .../parts/{part}/sign`, keeps the full rule
set.

## Settings

### Web service - Front door

Set the public hostname ProjectSend answers on.

| Setting       | Value                     | Description                              |
| ------------- | ------------------------- | ---------------------------------------- |
| `SERVER_NAME` | `projectsend.example.com` | Domain name(s) this service answers for. |

### SSL / Let's Encrypt

Obtain and renew the certificate automatically.

| Setting              | Value               | Description                                                |
| -------------------- | ------------------- | ---------------------------------------------------------- |
| `AUTO_LETS_ENCRYPT`  | `yes`               | Obtain and renew Let's Encrypt certificates automatically. |
| `EMAIL_LETS_ENCRYPT` | `admin@example.com` | Contact email registered with Let's Encrypt.               |

### Reverse Proxy

Point BunkerWeb at the ProjectSend container and tune streaming for large uploads.

| Setting                           | Value                   | Description                                                               |
| --------------------------------- | ----------------------- | ------------------------------------------------------------------------- |
| `USE_REVERSE_PROXY`               | `yes`                   | Forward requests to an upstream application.                              |
| `REVERSE_PROXY_INTERCEPT_ERRORS`  | `no`                    | Replace upstream error pages with BunkerWeb error pages.                  |
| `REVERSE_PROXY_HOST`              | `http://projectsend:80` | Upstream application: container name and port, or IP address and port.    |
| `REVERSE_PROXY_BUFFERING`         | `no`                    | Buffer upstream responses in BunkerWeb before sending them to the client. |
| `REVERSE_PROXY_REQUEST_BUFFERING` | `no`                    | Buffer the whole client request before forwarding it upstream.            |
| `REVERSE_PROXY_READ_TIMEOUT`      | `3600s`                 | Maximum time to wait for the upstream to send data.                       |
| `REVERSE_PROXY_SEND_TIMEOUT`      | `3600s`                 | Maximum time to wait while sending data to the upstream.                  |
| `PROXY_BUFFERS`                   | `8 32k`                 | Number and size of NGINX buffers used for upstream responses.             |
| `PROXY_BUFFER_SIZE`               | `32k`                   | Size of the buffer for the first part of the upstream response.           |
| `PROXY_BUSY_BUFFERS_SIZE`         | `64k`                   | Limit of buffers busy sending a response to the client.                   |

### Web service - ProjectSend specific settings

Allow the methods and the chunk size the resumable uploader needs.

| Setting           | Value                                   | Description                                                      |
| ----------------- | --------------------------------------- | ---------------------------------------------------------------- |
| `ALLOWED_METHODS` | `GET\|POST\|HEAD\|PUT\|DELETE\|PATCH\|OPTIONS` | HTTP methods that are accepted; all others are rejected.         |
| `MAX_CLIENT_SIZE` | `100m`                                  | Maximum size of one request body; keep at least 2x the part size. |

### Rate Limiting

Cap concurrent connections and the request rate per client IP.

| Setting                | Value   | Description                                                    |
| ---------------------- | ------- | -------------------------------------------------------------- |
| `LIMIT_CONN_MAX_HTTP1` | `50`    | Maximum concurrent HTTP/1 connections per client IP.           |
| `LIMIT_CONN_MAX_HTTP2` | `500`   | Maximum concurrent HTTP/2 streams per client IP.               |
| `LIMIT_CONN_MAX_HTTP3` | `500`   | Maximum concurrent HTTP/3 streams per client IP.               |
| `LIMIT_REQ_RATE`       | `25r/s` | Allowed request rate for the matching URL (for example 15r/s). |

### Security

Challenge bots, ban bad behavior and harden cookies and headers.

| Setting                               | Value                                                                                                      | Description                                                                    |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `USE_ANTIBOT`                         | `javascript`                                                                                               | Challenge visitors to block bots.                                              |
| `ANTIBOT_IGNORE_URI`                  | `^/build/ ^/api/ ^/s/ ^/uploads(?:/\|$) ^/zip-downloads(?:/\|$)`                                           | URL patterns (regex) that skip the anti-bot challenge, for example API routes. |
| `BAD_BEHAVIOR_THRESHOLD`              | `20`                                                                                                       | Number of bad responses before the client IP is banned.                        |
| `COOKIE_FLAGS_2`                      | `projectsend_xsrf SameSite=Lax`                                                                            | Flags (HttpOnly, Secure, SameSite...) added to cookies.                        |
| `KEEP_UPSTREAM_HEADERS`               | `Content-Security-Policy Strict-Transport-Security X-Frame-Options X-Content-Type-Options Referrer-Policy` | Upstream headers that BunkerWeb passes through instead of overwriting.         |
| `CONTENT_SECURITY_POLICY_REPORT_ONLY` | `yes`                                                                                                      | Report Content-Security-Policy violations instead of blocking them.            |

### Performance

Compress proxied responses and leave file serving to ProjectSend.

| Setting         | Value                                    | Description                                                             |
| --------------- | ---------------------------------------- | ----------------------------------------------------------------------- |
| `USE_GZIP`      | `yes`                                    | Enable Gzip compression of responses.                                   |
| `GZIP_PROXIED`  | `expired no-cache no-store private auth` | Which proxied responses are compressed, based on their caching headers. |
| `SERVE_FILES`   | `no`                                     | Serve static files from BunkerWeb itself (off when proxying an app).    |

### Error pages and robots

Choose which upstream errors BunkerWeb replaces and serve a robots.txt.

| Setting                   | Value                                 | Description                                              |
| ------------------------- | ------------------------------------- | -------------------------------------------------------- |
| `INTERCEPTED_ERROR_CODES` | `404 405 413 429 500 501 502 503 504` | Upstream status codes replaced by BunkerWeb error pages. |
| `USE_ROBOTSTXT`           | `yes`                                 | Serve a robots.txt file generated by BunkerWeb.          |

### Security - ModSecurity false positives

Let the chunked upload part request through the OWASP Core Rule Set.

Custom configs: `modsec-crs/projectsend_false_positives.conf`

## Custom configs

Files are referenced relative to the template's `configs/` directory and grouped by BunkerWeb
config type.

| Type         | File in repository                                                     | Reference in `template.json`                | Purpose                                                                  |
| ------------ | ---------------------------------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------------------ |
| `modsec-crs` | `templates/projectsend/configs/modsec-crs/projectsend_false_positives.conf` | `modsec-crs/projectsend_false_positives.conf` | Remove CRS rules `920340`, `920640` and `920420` for the upload part `PUT`. |

## Docker Compose example

Based on the official `docker/production/compose.example.yaml` of ProjectSend. The application
publishes no port: BunkerWeb reaches it over the `bw-services` network, so `TRUSTED_PROXIES: "*"`
is only safe while that network contains containers you trust. If it does not, set
`TRUSTED_PROXIES` to the IP address or subnet of BunkerWeb instead. The queue worker and the
scheduler run inside the app container. Passwords below are placeholders, replace them with your
own values before deploying.

```yaml
name: projectsend

services:
  app:
    image: projectsend/projectsend:2
    restart: unless-stopped
    environment:
      # Must be the public URL, the same host as SERVER_NAME in the template.
      APP_URL: "https://projectsend.example.com"
      APP_ENV: production
      APP_DEBUG: "false"

      # Trust X-Forwarded-For from whoever connects. Safe here because the port is not published
      # and only containers on bw-services can reach it. Narrow it to BunkerWeb's IP or subnet
      # if other containers share that network.
      TRUSTED_PROXIES: "*"

      DB_CONNECTION: mysql
      DB_HOST: db
      DB_PORT: "3306"
      DB_DATABASE: projectsend
      DB_USERNAME: projectsend
      DB_PASSWORD: "${DB_PASSWORD:?Set DB_PASSWORD}"

      REDIS_HOST: redis
      CACHE_STORE: redis
      SESSION_DRIVER: redis
      QUEUE_CONNECTION: redis
    volumes:
      # Holds every uploaded file and the generated APP_KEY: back this volume up.
      - storage:/var/www/html/storage
    depends_on:
      db:
        condition: service_healthy
    networks:
      default:
      bw-services:
        aliases:
          - projectsend

  db:
    image: mysql:8.4
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: projectsend
      MYSQL_USER: projectsend
      MYSQL_PASSWORD: "${DB_PASSWORD:?Set DB_PASSWORD}"
      MYSQL_ROOT_PASSWORD: "${DB_ROOT_PASSWORD:?Set DB_ROOT_PASSWORD}"
    volumes:
      - db-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "--silent"]
      interval: 5s
      timeout: 5s
      retries: 20

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis-data:/data

volumes:
  storage:
  db-data:
  redis-data:

networks:
  bw-services:
    external: true
```

Set the required environment variables before starting the stack:

```bash
export DB_PASSWORD="$(openssl rand -hex 24)"
export DB_ROOT_PASSWORD="$(openssl rand -hex 24)"
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
  test -f "configs/$f" || { echo "missing: $f"; exit 1; }
done
```

After applying it:

- Confirm the BunkerWeb scheduler reload completes without a template or unknown-setting error.
- Sign in, upload a file larger than 20 MiB, download it, and open a public share link
  (`/s/{token}`) from a private window.
- Check BunkerWeb logs for rate-limit, bad-behavior, and ModSecurity events before changing
  security controls.
