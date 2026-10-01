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
4. Reload BunkerWeb, open the public URL, and test sign-in plus an upload and download.

## Placeholders

The values below are examples. Replace them with your own before using the template:

- `example.com` → your own domain name
- `admin@example.com` → your own email address
- `http://docuseal` → the name of the service's container on the shared Docker network, or its IP address and port, for example `http://10.0.0.5:8080`
- `replace-with-your-crowdsec-api-key` → the API key generated in CrowdSec (for example with `cscli bouncers add <name>`)
- `example-user` → your own username
- `replace-with-a-long-random-value` → your own long random secret

## Notes

- **Certificates:** the HTTP challenge needs port 80 on BunkerWeb to be reachable from the internet for every domain in `SERVER_NAME`.
- **Upstream:** `http://docuseal` is a container name. It only resolves when BunkerWeb and the application share a Docker network; otherwise replace it with the IP address and port of the service, for example `http://10.0.0.5:8080`. Add the port if the application does not listen on 80.
- **Uploads:** `MAX_CLIENT_SIZE=50m` limits each request, not the total size of a file sent by a chunked client. Raise it only if clients need larger single requests.
- **Methods:** only the listed HTTP methods are accepted; anything else is rejected before reaching the application.

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
| `REVERSE_PROXY_HOST` | `http://docuseal` | Upstream application: container name, or IP address and port. |
| `REVERSE_PROXY_BUFFERING` | `no` | — |

### Rate Limiting

Configure rate limiting settings.

| Setting | Value | Description |
| --- | --- | --- |
| `LIMIT_CONN_MAX_HTTP1` | `25` | — |
| `LIMIT_CONN_MAX_HTTP2` | `200` | — |
| `LIMIT_CONN_MAX_HTTP3` | `200` | — |
| `LIMIT_REQ_RATE` | `60r/s` | Allowed request rate for the matching URL (for example 15r/s). |

### Security

Configure security settings.

| Setting | Value | Description |
| --- | --- | --- |
| `USE_ANTIBOT` | `javascript` | Challenge visitors to block bots. |
| `CROWDSEC_API` | `http://127.0.0.1:8080` | — |
| `CROWDSEC_API_KEY` | `replace-with-your-crowdsec-api-key` | API key generated in CrowdSec for the BunkerWeb bouncer. |
| `CROWDSEC_APPSEC_URL` | `http://127.0.0.1:7422` | — |

### Performance

Configure performance settings.

| Setting | Value | Description |
| --- | --- | --- |
| `GZIP_PROXIED` | `expired no-cache no-store private auth` | — |
| `MAX_CLIENT_SIZE` | `50m` | Maximum size of a single request body. |
| `SERVE_FILES` | `no` | Serve static files from BunkerWeb itself (off when proxying an app). |

### Whitelist / Allowlist

Configure whitelist / allowlist settings.

| Setting | Value | Description |
| --- | --- | --- |
| `WHITELIST_IP` | `194.163.173.229 57.129.118.35 57.129.118.37` | — |

### Other

Configure other settings.

| Setting | Value | Description |
| --- | --- | --- |
| `KEEP_UPSTREAM_HEADERS` | `Content-Security-Policy Strict-Transport-Security X-Frame-Options X-Content-Type-Options Referrer-Policy` | — |

### Other settings

Additional settings for this service.

| Setting | Value | Description |
| --- | --- | --- |
| `ANTIBOT_IGNORE_URI` | `^/api/` | — |
| `CLIENT_CACHE_CONTROL` | `public, max-age=604800` | — |
| `BLACKLIST_COUNTRY` | `CF CN IN RU ZA` | — |
| `INTERCEPTED_ERROR_CODES` | `404 405 413 429 500 501 502 503 504` | — |
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
      - DATABASE_URL=postgresql://example-user:example-password@postgres:5432/docuseal
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
```

Set the required environment variables before starting the stack:

```bash
export POSTGRES_PASSWORD='replace-with-a-long-random-value'
docker compose config
docker compose up -d
```

## Validation

Before importing the template:

```bash
jq . template.json
jq -e --arg expected "$(basename "$PWD")" '.id == $expected' template.json
jq -e '(.settings | keys | sort) == ([.steps[] | .settings[]] | sort)' template.json
```

After applying it:

- Confirm the BunkerWeb scheduler reload completes without a template or unknown-setting error.
- Sign in and test an upload and download.
- Check BunkerWeb logs for rate-limit, bad-behavior, and ModSecurity events before changing security controls.
