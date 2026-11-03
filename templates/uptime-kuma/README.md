# Uptime Kuma Template

## Overview

Proxy Uptime Kuma's dashboard, status pages, push monitors and metrics through BunkerWeb.
The template enables WebSockets on `/socket.io/`, allows its polling requests, and preserves
upstream JSON errors. It keeps the CRS enabled.

## Prerequisites

- Uptime Kuma reachable from BunkerWeb at `http://uptime-kuma:3001`. With Docker, attach both
  containers to the same network; keep the application port private.
- A dedicated hostname pointing to BunkerWeb. Uptime Kuma does not support hosting the full
  application under a subdirectory; see the upstream
  [reverse proxy guide](https://github.com/louislam/uptime-kuma/wiki/Reverse-Proxy).
- Port 80 reachable for the Let's Encrypt HTTP challenge. For an existing certificate, disable
  `AUTO_LETS_ENCRYPT` and configure BunkerWeb's `USE_CUSTOM_SSL`, `CUSTOM_SSL_CERT` and
  `CUSTOM_SSL_KEY` settings instead.

## Import and configure

1. Follow the repository's [installation guide](../../README.md#installing-templates).
   Import `template.json` and upload `configs/modsec-crs/uptime-kuma_false_positives.conf` when
   prompted. Keep the config type `modsec-crs`: rule 920420 runs in phase 1.
2. Select this template for the service, or set `USE_TEMPLATE=uptime-kuma`.
3. Replace `SERVER_NAME`, `REVERSE_PROXY_HOST` and `REVERSE_PROXY_HOST_1`. Both upstream entries
   must point to the same Uptime Kuma instance, including its port.
4. Begin with `SECURITY_MODE=detect`, exercise setup, login, status pages and monitor callbacks,
   and inspect BunkerWeb logs. Switch to `block` after resolving any additional false positives.

## Settings

| Setting | Default | Purpose |
| --- | --- | --- |
| `SERVER_NAME` | `uptime-kuma.example.com` | Dedicated public hostname. |
| `AUTO_LETS_ENCRYPT` | `yes` | Automatic HTTPS certificates. |
| `USE_REVERSE_PROXY` | `yes` | Proxy requests to Uptime Kuma. |
| `REVERSE_PROXY_URL` | `/` | Dashboard, status pages and REST endpoints. |
| `REVERSE_PROXY_HOST` | `http://uptime-kuma:3001` | Main upstream. |
| `REVERSE_PROXY_INTERCEPT_ERRORS` | `no` | Preserve API JSON errors, including invalid push tokens. |
| `REVERSE_PROXY_URL_1` | `/socket.io/` | Socket.IO polling and upgrade path. |
| `REVERSE_PROXY_HOST_1` | `http://uptime-kuma:3001` | Same upstream for Socket.IO. |
| `REVERSE_PROXY_WS_1` | `yes` | Forward WebSocket upgrades on that path. |
| `USE_LIMIT_REQ` | `yes` | Keep request throttling enabled. |
| `LIMIT_REQ_URL` | `/` | Fallback request-limit rule. |
| `LIMIT_REQ_RATE` | `50r/s` | Allow polling handshakes and page-load bursts. |
| `SERVE_FILES` | `no` | Serve application content through the upstream. |

## Security and compatibility notes

- **Polling exception:** only `POST /socket.io/` with `Content-Type: text/plain` (optionally
  followed by parameters) removes rule 920420. Other paths, methods and content types keep that
  rule. This is a rule removal, not a target removal: the latter did not work in the historical
  test. CRS does not parse these text packets as JSON or forms, so their application payloads
  are not inspected. WebSocket frames also bypass HTTP request-body inspection after upgrade.
- **Request limits:** the default BunkerWeb `2r/s` limit interrupted the polling handshake with
  `429` in the historical test; `50r/s` passed. Repeated blocked requests can still trigger
  BunkerWeb's bad-behavior ban. Tune limits against your actual monitor and client load.
- **Uploads:** status-page logos travel inside Socket.IO messages. Raising `MAX_CLIENT_SIZE`
  does not raise the application's WebSocket message limit.
- **Authentication:** keep Uptime Kuma's own authentication enabled. External authentication
  must account for public status pages, metrics clients and `/api/push/` callbacks separately.
  The template supplies no external authentication policy or antibot challenge.
- **Client IPs:** enable Uptime Kuma's **Settings → Reverse Proxy → HTTP Headers → Trust Proxy**
  only when the application is reachable exclusively through trusted proxies. See the upstream
  guide linked above.

## Validation

On 2026-11-03, the packaged settings and exclusion were checked with the local BunkerWeb
`:tests` image (1.6.18-rc2, libmodsecurity 3.0.17, CRS 4.30.0) against an echo backend, not a
running Kuma instance. The polling assertion failed with `403` before the exclusion and passed
with `200` after it. Other paths, a near path, `PUT`, other media types and a SQLi form remained
blocked. WebSocket upgrade returned `101` with a valid accept header, and the upstream JSON
`404` stayed JSON. Generated nginx configuration passed its syntax check.

Check JSON syntax from the repository root:

```bash
jq . templates/uptime-kuma/template.json
```

After import, check database setup and login, the polling handshake, a WebSocket upgrade on
`/socket.io/` returning `101`, a public status page, `/metrics` with its configured credentials,
and both valid and invalid push tokens. An invalid token should retain Uptime Kuma's JSON `404`.
In block mode, verify that the content-type exception does not extend to another path, method
or media type. Inspect BunkerWeb logs for rule IDs and `429` responses.

Historical scripted tests used Uptime Kuma 2.5.5 with SQLite and BunkerWeb 1.6.15 on 2026-10-06.
They covered setup, login, polling, WebSockets, HTTP and push monitors, metrics, status-page
editing and negative exclusion boundaries. Notification providers, Docker/database monitors,
status-page domain mapping, MariaDB mode and a real browser were not tested. Those results
predate this packaged template and do not certify another version or deployment.
