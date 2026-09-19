# Seafile Template

## Overview

This template proxies Seafile through BunkerWeb with upload, WebDAV, rate-limit, compression, and
client-compatibility defaults. The deployment example targets Seafile Community Edition 11.0.x; use
Seafile's version-specific Compose file when deploying another release.

## Prerequisites

- A running Seafile server that BunkerWeb can reach.
- A public hostname pointing to BunkerWeb.
- Access to the BunkerWeb web UI or service environment variables.

## Setup

1. Import `template.json` by following the repository's
   [installation guide](../../README.md#installing-templates).
2. Assign `USE_TEMPLATE=seafile` to the service, or select **Seafile** in the web UI.
3. Replace `SERVER_NAME`, `EMAIL_LETS_ENCRYPT`, and `REVERSE_PROXY_HOST` with values for your
   deployment. The default upstream, `http://seafile`, assumes both containers share a Docker network.
4. Configure Seafile's public URL for HTTPS. For a fresh Seafile 11 Docker deployment, set
   `SEAFILE_SERVER_HOSTNAME` and `FORCE_HTTPS_IN_CONF=true`. For an existing deployment, verify:

   ```python
   SERVICE_URL = "https://seafile.example.com"
   FILE_SERVER_ROOT = "https://seafile.example.com/seafhttp"
   CSRF_TRUSTED_ORIGINS = ["https://seafile.example.com"]
   SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
   ```

   Put manual overrides in `/opt/seafile-data/seafile/conf/seahub_settings.py`, replace the example
   hostname, and restart Seafile. Do not add a trailing slash to the trusted origin.
5. Reload BunkerWeb, sign in through the public URL, and upload and download a test file.

The `COOKIE_FLAGS_4` override deliberately omits `HttpOnly` for Seafile's `sfcsrftoken` cookie. It
narrows BunkerWeb's wildcard cookie policy so Seahub can read the CSRF token without weakening the
flags on other cookies.

## Uploads, WebDAV, and notifications

`MAX_CLIENT_SIZE=512m` limits each request, not the total file size used by chunked clients. Raising it
also raises ModSecurity's in-memory request-body limit, so increase it only for clients that need larger
single requests.

The template allows WebDAV methods and exempts `/seafdav` from the JavaScript challenge, but SeafDAV is
disabled by default in Seafile. Enable it in `seafdav.conf`, set `share_name = /seafdav`, and test with a
WebDAV client. See Seafile's [WebDAV documentation](https://manual.seafile.com/11.0/extension/webdav/).

WebSockets are not enabled on the main `/` upstream. Seafile's optional notification server listens on
a separate port and needs dedicated `/notification/ping` and `/notification` proxy routes; configure
those only after enabling the service by following Seafile's
[notification server documentation](https://manual.seafile.com/11.0/deploy/notification-server/).

The bad-behavior status list excludes `401`, `403`, and `404`. Authentication, permission checks, and
missing-object probes can legitimately return those codes, so counting them toward an IP ban can lock
out normal sync or WebDAV clients.

## Docker Compose example

This is a small Seafile 11.0 example based on the
[official deployment guide](https://manual.seafile.com/11.0/docker/deploy_seafile_with_docker/). It binds
Seafile only to localhost because BunkerWeb is the public entry point. Attach the BunkerWeb container to
the `seafile-net` network so the default `http://seafile` upstream resolves; otherwise, use an upstream
address reachable from BunkerWeb.

```yaml
services:
  db:
    image: mariadb:10.11
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: "${MYSQL_ROOT_PASSWORD:?Set MYSQL_ROOT_PASSWORD}"
      MYSQL_LOG_CONSOLE: "true"
      MARIADB_AUTO_UPGRADE: "1"
    volumes:
      - /opt/seafile-mysql/db:/var/lib/mysql
    networks:
      - seafile-net

  memcached:
    image: memcached:1.6.18
    restart: unless-stopped
    entrypoint: memcached -m 256
    networks:
      - seafile-net

  seafile:
    image: seafileltd/seafile-mc:11.0-latest
    restart: unless-stopped
    ports:
      - "127.0.0.1:7841:80"
    volumes:
      - /opt/seafile-data:/shared
    environment:
      DB_HOST: db
      DB_ROOT_PASSWD: "${MYSQL_ROOT_PASSWORD:?Set MYSQL_ROOT_PASSWORD}"
      TIME_ZONE: Etc/UTC
      SEAFILE_ADMIN_EMAIL: "${SEAFILE_ADMIN_EMAIL:-admin@example.com}"
      SEAFILE_ADMIN_PASSWORD: "${SEAFILE_ADMIN_PASSWORD:?Set SEAFILE_ADMIN_PASSWORD}"
      SEAFILE_SERVER_LETSENCRYPT: "false"
      SEAFILE_SERVER_HOSTNAME: seafile.example.com
      FORCE_HTTPS_IN_CONF: "true"
    depends_on:
      - db
      - memcached
    networks:
      - seafile-net

networks:
  seafile-net:
    name: seafile-net
```

Set the required environment variables before starting the stack:

```bash
export MYSQL_ROOT_PASSWORD='replace-with-a-long-random-value'
export SEAFILE_ADMIN_PASSWORD='replace-with-a-different-long-random-value'
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
- Sign in, browse a library, and upload and download a file larger than the default 10 MiB limit.
- If SeafDAV is enabled, run `curl -u user:password -X PROPFIND -H 'Depth: 1'`
  `https://seafile.example.com/seafdav/` and confirm it is not challenged by Antibot.
- Check BunkerWeb logs for rate-limit, bad-behavior, and ModSecurity events before changing security
  controls.
