# Docuseal Template

## Important the Docuseal Api was not tested against Modsec, when you use the api make sure to test it and cross check the Modsec logs for false positives!

## Overview

This template proxies Docuseal through BunkerWeb with automatic Let's Encrypt certificates, reverse proxying and a 50m request size limit.

## Prerequisites

- A running Docuseal server that BunkerWeb can reach.
- A public hostname pointing to BunkerWeb.
- Access to the BunkerWeb web UI or service environment variables.

## Setup

1. Import `template.json` (Templates → **Create new template** → **Raw**, then upload or paste it).
2. Assign `USE_TEMPLATE=docuseal` to the service, or select **Docuseal** in the web UI.
3. Replace `SERVER_NAME`, `EMAIL_LETS_ENCRYPT` and `REVERSE_PROXY_HOST` with values for your deployment.
4. Reload BunkerWeb, open the public URL, and test sign-in and the main workflow of the application (for example a document, upload or API call).

## Placeholders

The values below are examples. Replace them with your own before using the template:

- `example.com` → your own domain name
- `admin@example.com` → your own email address
- `http://docuseal:3000` → the name of the service's container on the shared Docker network, or its IP address and port, for example `http://10.0.0.5:8080`
- `replace-with-your-crowdsec-api-key` → the API key generated in CrowdSec (for example with `cscli bouncers add <name>`)
- `example-user` → your own username
- `openssl rand -hex 24` → generates a long random secret for each variable
- the example addresses `192.0.2.x`, `198.51.100.x` and `203.0.113.x` → the IP addresses or ranges that should be allowed

## Notes

- **Certificates:** the HTTP challenge needs port 80 on BunkerWeb to be reachable from the internet for every domain in `SERVER_NAME`.
- **Upstream:** `http://docuseal:3000` is a container name and port. It only resolves when BunkerWeb and the application share a Docker network (the Compose example joins the external `bw-services` network with the alias `docuseal`; attach BunkerWeb to the same network); otherwise replace it with the IP address and port of the service, for example `http://10.0.0.5:8080`.
- **Uploads:** `MAX_CLIENT_SIZE=50m` limits each request, not the total size of a file sent by a chunked client. Raise it only if clients need larger single requests.
- **Methods:** only the listed HTTP methods are accepted; anything else is rejected before reaching the application.
- **Anti-bot:** `ANTIBOT_IGNORE_URI` (`^/api/`) skips the challenge for those paths because API clients cannot solve it. Those paths are protected only by the application and the other security features.
- **CORS:** `OPTIONS` is not in `ALLOWED_METHODS`, so browser preflight requests for cross-origin API calls are rejected. Add it if a web client calls the API from another origin.
- **Allowlist:** the addresses in the allowlist are documentation examples. Replace them with the IP addresses or ranges that may bypass the security checks.
- **Database password:** the database service and the connection URL read the same variable. Generate it with `openssl rand -hex 24`; if you choose your own password, percent-encode URI-reserved characters (`@ : / ? # %`) in the URL.

## Settings

### Web service - Front door

Configure server / core settings.

| Setting | Value | Description |
| --- | --- | --- |
| `SERVER_NAME` | `docuseal.example.com` | Domain name(s) this service answers for. |

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
| `REVERSE_PROXY_HOST` | `http://docuseal:3000` | Upstream application: container name, or IP address and port. |
| `REVERSE_PROXY_BUFFERING` | `no` | Buffer upstream responses in BunkerWeb before sending them to the client. |

### Rate Limiting

Configure rate limiting settings.

| Setting | Value | Description |
| --- | --- | --- |
| `LIMIT_CONN_MAX_HTTP1` | `25` | Maximum concurrent HTTP/1 connections per client IP. |
| `LIMIT_CONN_MAX_HTTP2` | `200` | Maximum concurrent HTTP/2 streams per client IP. |
| `LIMIT_CONN_MAX_HTTP3` | `200` | Maximum concurrent HTTP/3 streams per client IP. |
| `LIMIT_REQ_RATE` | `60r/s` | Allowed request rate for the matching URL (for example 15r/s). |

### Security

Configure security settings.

| Setting | Value | Description |
| --- | --- | --- |
| `USE_ANTIBOT` | `javascript` | Challenge visitors to block bots. |
| `CROWDSEC_API` | `http://127.0.0.1:8080` | URL of the CrowdSec local API. |
| `CROWDSEC_API_KEY` | `replace-with-your-crowdsec-api-key` | API key generated in CrowdSec for the BunkerWeb bouncer. |
| `CROWDSEC_APPSEC_URL` | `http://127.0.0.1:7422` | URL of the CrowdSec AppSec component. |

### Performance

Configure performance settings.

| Setting | Value | Description |
| --- | --- | --- |
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
| `KEEP_UPSTREAM_HEADERS` | `Content-Security-Policy Strict-Transport-Security X-Frame-Options X-Content-Type-Options Referrer-Policy` | Upstream headers that BunkerWeb passes through instead of overwriting. |

### Other settings

Additional settings for this service.

| Setting | Value | Description |
| --- | --- | --- |
| `ANTIBOT_IGNORE_URI` | `^/api/` | URL patterns (regex) that skip the anti-bot challenge, for example API routes. |
| `CLIENT_CACHE_CONTROL` | `public, max-age=604800` | — |
| `BLACKLIST_COUNTRY` | `CF CN IN RU ZA` | — |
| `INTERCEPTED_ERROR_CODES` | `404 405 413 429 500 501 502 503 504` | Upstream status codes replaced by BunkerWeb error pages. |
| `ALLOWED_METHODS` | `GET\|PUT\|POST\|HEAD\|QUERY\|DELETE\|PATCH` | HTTP methods that are accepted; all others are rejected. |
| `WHITELIST_URI` | `acme-v01.api.letsencrypt.org acme-staging.api.letsencrypt.org acme-v02.api.letsencrypt.org acme-staging-v02.api.letsencrypt.org` | — |

## Docker Compose example

Usernames and passwords below are placeholders. Replace them with your own values before deploying.

```yaml
services:
  app:
    depends_on:
      postgres:
        condition: service_healthy
    image: docuseal/docuseal:latest
    restart: unless-stopped
    ports:
      - 3030:3000
    volumes:
      - ./docuseal:/data/docuseal
    env_file:
      - .env
    environment:
      - HOST=${HOST}
      - DATABASE_URL=postgresql://example-user:${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD}@postgres:5432/docuseal
    networks:
      default:
      bw-services:
        aliases:
          - docuseal
  postgres:
    image: postgres:18
    restart: unless-stopped
    volumes:
      - './pg_data:/var/lib/postgresql/18/docker'
    environment:
      POSTGRES_USER: "example-user"
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD}"
      POSTGRES_DB: docuseal
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

networks:
  bw-services:
    external: true
```

Set the required environment variables before starting the stack:

```bash
export POSTGRES_PASSWORD="$(openssl rand -hex 24)"
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
