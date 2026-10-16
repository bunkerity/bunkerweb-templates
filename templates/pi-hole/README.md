# Pi-hole Template

This template reverse-proxies the Pi-hole **web admin UI and REST API** through
BunkerWeb. Pi-hole's DNS service is out of scope: keep serving DNS directly from
Pi-hole on port 53 and only front the HTTP interface with BunkerWeb.

## Setup

1. Follow the repository's [installation guide](../../README.md#installing-templates)
   for the web UI or plugin bundle method.
2. Assign the template to the Pi-hole service (`USE_TEMPLATE=pi-hole` or select
   it in the easy-mode UI).
3. Replace the example domain and upstream with your Pi-hole deployment, then
   review the real-IP settings described below.
4. Reload BunkerWeb and verify the admin UI and REST API through the proxy.

## Security tuning notes

This template deliberately relaxes a few defaults so a self-hosted Pi-hole admin
does not get locked out of their own dashboard. Each is safe to tighten once your
environment allows it:

- **`USE_DNSBL=no`** — administrators often connect from residential IPs that
  appear on public DNS blocklists, and leaving DNSBL on can lock you out of your
  own panel. Re-enable it if your admin IPs are static and known to be clean.
- **`BAD_BEHAVIOR_THRESHOLD=20`, `BAD_BEHAVIOR_BAN_TIME=3600`, and 401 left out of
  `BAD_BEHAVIOR_STATUS_CODES`** — the admin UI legitimately produces bursts of
  404/403 responses while you navigate it, and failed logins (401) are already
  rate-limited elsewhere, so they are not counted here. The one-hour ban keeps a
  genuine bad actor out without causing long self-lockouts on a fat-fingered day.
- **`LIMIT_REQ_RATE=20r/s`** — the admin dashboard polls its own API several times
  a second, so a lower rate would throttle normal use.
- **`ALLOWED_METHODS=GET|POST|HEAD|OPTIONS|PUT|PATCH|DELETE`**: the Pi-hole API changes
  settings with `PATCH`, and creates, updates and deletes entries with `PUT` and `DELETE`.
  BunkerWeb's default list answers `405` to all three.
- **`MAX_CLIENT_SIZE=100m`**: a Teleporter backup import can reach 20 MB or more, above the
  `10m` default that answers `413`.
- **`USE_METRICS=no`** — metrics are disabled to keep small instances light.
  Re-enable it if you want BunkerWeb's metrics page.

## Host header

Pi-hole answers `403` on `/` unless the request carries `Host: pi.hole`, because its web
server redirects by host name. The template sets `REVERSE_PROXY_CUSTOM_HOST=pi.hole`, so
BunkerWeb sends `pi.hole` as the upstream `Host` header instead of the public host name. The
setting is unsuffixed: `REVERSE_PROXY_CUSTOM_HOST_1` has no effect.

## Cloudflare real IP assumption

`REAL_IP_FROM` and `REAL_IP_HEADER` default to Cloudflare's IP ranges and the
`CF-Connecting-IP` header, assuming Pi-hole sits behind Cloudflare. If you are
**not** behind Cloudflare, edit `REAL_IP_FROM`/`REAL_IP_HEADER` to match your own
proxy or set `USE_REAL_IP=no`, otherwise client IPs will be mis-attributed.

## Root Domain Caveat

It is possible to serve Pi-hole at the root of your domain, but this adds an
extra attack surface and is **not advised**. Additionally, you will always be
redirected to `/admin/` — this is how Pi-hole is built and cannot be changed;
using BunkerWeb, you would need to modify Pi-hole itself to achieve otherwise.

If you still want to proceed, use the raw config below instead of the
`template.json`.

## Raw Config

```env
SERVER_NAME=example.com
BAD_BEHAVIOR_STATUS_CODES=400 403 404 405 429 444
BAD_BEHAVIOR_THRESHOLD=20
BAD_BEHAVIOR_BAN_TIME=3600
USE_BUNKERNET=yes
USE_DNSBL=no
GZIP_PROXIED=expired no-cache no-store private auth
REMOVE_HEADERS=Server Expect-CT X-Powered-By X-AspNet-Version X-AspNetMvc-Version Public-Key-Pins
AUTO_LETS_ENCRYPT=yes
EMAIL_LETS_ENCRYPT=registration@example.com
LIMIT_CONN_MAX_HTTP2=50
LIMIT_CONN_MAX_HTTP3=50
LIMIT_REQ_RATE=20r/s
MAX_CLIENT_SIZE=100m
ALLOWED_METHODS=GET|POST|HEAD|OPTIONS|PUT|PATCH|DELETE
USE_METRICS=no
USE_REAL_IP=yes
REAL_IP_FROM=172.64.0.0/13 104.16.0.0/13
REAL_IP_HEADER=CF-Connecting-IP
USE_REVERSE_PROXY=yes
REVERSE_PROXY_URL_1=/
REVERSE_PROXY_HOST_1=http://pihole-server-ip:80
REVERSE_PROXY_CUSTOM_HOST=pi.hole
REVERSE_PROXY_WS_1=yes
REVERSE_PROXY_KEEPALIVE_1=yes
REVERSE_PROXY_HIDE_HEADERS_1=
REVERSE_PROXY_CONNECT_TIMEOUT_1=30s
REVERSE_PROXY_READ_TIMEOUT_1=90s
```

## Validation

- Run `jq . template.json` to confirm the template definition is valid JSON.
- Sign in to the admin UI and exercise its API-backed dashboard through
  BunkerWeb to confirm routing, client IP attribution, and rate limits.
