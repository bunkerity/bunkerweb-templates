# DocuSeal Template

## Overview

This template proxies DocuSeal through BunkerWeb with automatic Let's Encrypt certificates, reverse
proxying, API-friendly error handling and a 50m request size limit.

No ModSecurity CRS exclusions ship with this template yet. The DocuSeal API was not observed behind
ModSecurity, so if you use it, test your calls and check the ModSecurity logs for false positives
before relying on the defaults. One known case on BunkerWeb 1.6.17: a JSON field that carries a
`data:` URI (for example `data:image/png;base64,...`) is blocked by CRS rules 934170, 941130, 941170,
932235 and 932260, while plain base64 passes. Send plain base64, or add a scoped exclusion for that
field.

## Prerequisites

- A running DocuSeal server that BunkerWeb can reach.
- A public hostname pointing to BunkerWeb.
- Access to the BunkerWeb web UI or service environment variables.

## Setup

1. Import `template.json` (Templates → **Create new template** → **Raw**, then upload or paste it).
2. Assign `USE_TEMPLATE=docuseal` to the service, or select **DocuSeal** in the web UI.
3. Replace `SERVER_NAME`, `EMAIL_LETS_ENCRYPT` and `REVERSE_PROXY_HOST` with values for your
   deployment.
4. Reload BunkerWeb, open the public URL, and test sign-in and the main workflow of the application
   (for example a document, upload or API call).

## Placeholders

The values below are examples. Replace them with your own before using the template:

- `example.com` → your own domain name
- `admin@example.com` → your own email address
- `http://docuseal:3000` → the name of the service's container on the shared Docker network and the
  port DocuSeal listens on, or its IP address and port, for example `http://10.0.0.5:3000`
- `example-user` → your own username
- `openssl rand -hex 24` → generates a long random secret for each variable

## Notes

- **Certificates:** the HTTP challenge needs port 80 on BunkerWeb to be reachable from the internet
  for every domain in `SERVER_NAME`.
- **Upstream:** DocuSeal's image listens on port `3000`, so `REVERSE_PROXY_HOST` is
  `http://docuseal:3000`. The name only resolves when BunkerWeb and the application share a Docker
  network (the Compose example joins the external `bw-services` network with the alias `docuseal`;
  attach BunkerWeb to the same network). Otherwise use the IP address and port of the service.
- **Anti-bot:** the template leaves `USE_ANTIBOT` off. External signers open `/s/<slug>` and
  `/p/<slug>` links, often on phones or inside mail-app webviews, and the embed script is served
  from `/js/`. A JavaScript challenge in front of those paths breaks them, and API clients cannot
  solve it either. DocuSeal's own authentication and the rate limits protect the service instead.
- **Countries:** no country blacklist ships, because signers can open a signing link from anywhere.
  Add `BLACKLIST_COUNTRY` yourself if every signer is in a known set of countries.
- **Error pages:** `REVERSE_PROXY_INTERCEPT_ERRORS=no` lets DocuSeal's own JSON errors reach API
  clients instead of BunkerWeb's HTML pages. `INTERCEPTED_ERROR_CODES` still applies to errors that
  BunkerWeb itself generates, such as `405`, `413` and `429`.
- **Body limits:** nginx enforces `MAX_CLIENT_SIZE`, but ModSecurity enforces
  `MODSECURITY_REQ_BODY_NO_FILES_LIMIT` first and rejects larger non-file bodies. DocuSeal treats
  every `/api` request as JSON and accepts base64-encoded files inside those bodies, so the default
  of 128 KB is too small. The template sets 10 MB. `MAX_CLIENT_SIZE=50m` limits each request, not
  the total size of a file sent by a chunked client, and it also sets ModSecurity's overall request
  body limit. Raise it only if clients need larger single requests.
- **Methods:** only the listed HTTP methods are accepted; anything else is rejected before reaching
  the application. DocuSeal's API uses `GET`, `POST`, `PUT`, `PATCH` and `DELETE`, and `OPTIONS`
  lets browsers send CORS preflight requests for cross-origin API calls.
- **Headers:** DocuSeal sends its own `Content-Security-Policy` with a per-request script nonce on
  page loads, so `KEEP_UPSTREAM_HEADERS` keeps it instead of replacing it with BunkerWeb's policy.
  The other security headers (`X-Frame-Options`, `Strict-Transport-Security`, `Referrer-Policy`,
  `X-Content-Type-Options`) come from BunkerWeb.
- **Compression:** `GZIP_PROXIED` only takes effect with `USE_GZIP=yes`, which the template sets.
- **Database password:** the database service and the connection URL read the same variable.
  Generate it with `openssl rand -hex 24`; if you choose your own password, percent-encode
  URI-reserved characters (`@ : / ? # %`) in the URL.

## Settings

### Web service - Front service

Point the template at your DocuSeal hostname.

| Setting              | Value                  | Description                                                |
| -------------------- | ---------------------- | ---------------------------------------------------------- |
| `SERVER_NAME`        | `docuseal.example.com` | Domain name(s) this service answers for.                   |
| `AUTO_LETS_ENCRYPT`  | `yes`                  | Obtain and renew Let's Encrypt certificates automatically. |
| `EMAIL_LETS_ENCRYPT` | `admin@example.com`    | Contact email registered with Let's Encrypt.               |

### Web service - Upstream server

Send traffic to the DocuSeal container on port 3000 and let its JSON API errors reach clients
unchanged.

| Setting                          | Value                  | Description                                                               |
| -------------------------------- | ---------------------- | ------------------------------------------------------------------------- |
| `USE_REVERSE_PROXY`              | `yes`                  | Forward requests to an upstream application.                              |
| `REVERSE_PROXY_HOST`             | `http://docuseal:3000` | Upstream application: container name and port, or IP address and port.    |
| `REVERSE_PROXY_BUFFERING`        | `no`                   | Buffer upstream responses in BunkerWeb before sending them to the client. |
| `REVERSE_PROXY_INTERCEPT_ERRORS` | `no`                   | Replace upstream error responses with BunkerWeb error pages.              |

### HTTP - API and upload constraints

Allow the REST verbs DocuSeal uses and keep both body limits above your largest payload.
`MAX_CLIENT_SIZE` also drives ModSecurity's `SecRequestBodyLimit`.

| Setting                               | Value                                          | Description                                                            |
| ------------------------------------- | ---------------------------------------------- | ---------------------------------------------------------------------- |
| `ALLOWED_METHODS`                     | `GET\|PUT\|POST\|HEAD\|DELETE\|PATCH\|OPTIONS` | HTTP methods that are accepted; all others are rejected.               |
| `MAX_CLIENT_SIZE`                     | `50m`                                          | Maximum size of a single request body.                                 |
| `MODSECURITY_REQ_BODY_NO_FILES_LIMIT` | `10485760`                                     | Maximum size of a request body that carries no file uploads, in bytes. |
| `SERVE_FILES`                         | `no`                                           | Serve static files from BunkerWeb itself (off when proxying an app).   |

### Security - Rate and connection limits

Limits per client IP, sized for the signing and admin pages.

| Setting                | Value   | Description                                                    |
| ---------------------- | ------- | -------------------------------------------------------------- |
| `LIMIT_CONN_MAX_HTTP1` | `25`    | Maximum concurrent HTTP/1 connections per client IP.           |
| `LIMIT_CONN_MAX_HTTP2` | `200`   | Maximum concurrent HTTP/2 streams per client IP.               |
| `LIMIT_CONN_MAX_HTTP3` | `200`   | Maximum concurrent HTTP/3 streams per client IP.               |
| `LIMIT_REQ_RATE`       | `60r/s` | Allowed request rate for the matching URL (for example 15r/s). |

### Security - Headers and error pages

DocuSeal sends its own nonce-based Content-Security-Policy, so BunkerWeb keeps it instead of
replacing it.

| Setting                   | Value                                 | Description                                                            |
| ------------------------- | ------------------------------------- | ---------------------------------------------------------------------- |
| `KEEP_UPSTREAM_HEADERS`   | `Content-Security-Policy`             | Upstream headers that BunkerWeb passes through instead of overwriting. |
| `INTERCEPTED_ERROR_CODES` | `404 405 413 429 500 501 502 503 504` | Status codes replaced by BunkerWeb error pages.                        |

### Performance - Client optimisation

Compress proxied responses; static files are served by DocuSeal, not BunkerWeb.

| Setting        | Value                                    | Description                                                             |
| -------------- | ---------------------------------------- | ----------------------------------------------------------------------- |
| `USE_GZIP`     | `yes`                                    | Compress responses sent to clients.                                     |
| `GZIP_PROXIED` | `expired no-cache no-store private auth` | Which proxied responses are compressed, based on their caching headers. |

## Docker Compose example

The layout follows DocuSeal's official `docker-compose.yml`, without the Caddy service and without
publishing the app port: BunkerWeb reaches DocuSeal over the `bw-services` network. Usernames and
passwords below are placeholders. Replace them with your own values before deploying.

```yaml
services:
  app:
    depends_on:
      postgres:
        condition: service_healthy
    image: docuseal/docuseal:latest
    restart: unless-stopped
    volumes:
      - ./docuseal:/data/docuseal
    environment:
      - FORCE_SSL=${HOST:?Set HOST}
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
      - "./pg_data:/var/lib/postgresql/18/docker"
    environment:
      POSTGRES_USER: "example-user"
      POSTGRES_PASSWORD: "${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD}"
      POSTGRES_DB: docuseal
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U example-user -d docuseal"]
      interval: 5s
      timeout: 5s
      retries: 5

networks:
  bw-services:
    external: true
```

`FORCE_SSL` makes DocuSeal treat requests as HTTPS and redirect plain HTTP, which is what it needs
behind a TLS-terminating proxy. Any value other than empty or `false` turns it on, and DocuSeal's
own Compose file passes the public hostname. Compose reads `HOST` and `POSTGRES_PASSWORD` from a
`.env` file next to `docker-compose.yml`. Create it before starting the stack:

```bash
cat > .env <<EOF
HOST=docuseal.example.com
POSTGRES_PASSWORD=$(openssl rand -hex 24)
EOF
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
- Sign in and test the main workflow of the application (for example an upload, download or API
  call).
- Check BunkerWeb logs for rate-limit, bad-behavior, and ModSecurity events before changing security
  controls.
