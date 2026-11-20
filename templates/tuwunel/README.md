<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/matrix-light.svg">
    <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/matrix.svg" alt="Matrix logo" width="96" height="96" />
  </picture>
</p>

<h1 align="center">Tuwunel</h1>

<p align="center">Matrix homeserver reverse proxy with WebSocket and delegation</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#dns-and-well-known-delegation">DNS and .well-known</a> ·
  <a href="#federation">Federation</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#files">Files</a> ·
  <a href="#validation">Validation</a>
</p>

Provision a curated BunkerWeb configuration for [Tuwunel](https://github.com/matrix-construct/tuwunel), a lightweight Matrix homeserver written in Rust and the maintained successor to conduwuit. This template ships sensible defaults for TLS, reverse proxying with WebSocket support, long federation/sync timeouts, request throttling, and `.well-known` client/server delegation so you can expose a Matrix homeserver through BunkerWeb quickly.

## At a glance

| Fact        | Value                           |
| ----------- | ------------------------------- |
| Template id | `tuwunel`                       |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4   |
| Configs     | 2 (modsec-crs, server-http)     |
| Rule ids    | `9874000` to `9874999`          |
| Upstream    | `http://tuwunel-server-ip:8008` |

## Prerequisites

- A running Tuwunel homeserver reachable on your network (its client/federation API listens on `8008` by default).
- The BunkerWeb UI or the ability to edit multisite settings directly.
- The public domain name that will act as your Matrix server name (the part after the `:` in your user IDs, e.g. `@alice:example.com`).

## DNS and .well-known Delegation

Matrix separates the **server name** (what appears in user IDs) from the **hostname that actually serves traffic**. Delegation lets `example.com` be your server name while BunkerWeb terminates TLS and proxies to Tuwunel.

1. Point an `A`/`AAAA` record for your server name (e.g. `example.com`) at the host running BunkerWeb.
2. Edit `configs/server-http/well-known.conf` and **replace every `example.com` with your own homeserver domain**. The placeholders are not usable as-is:
   - `/.well-known/matrix/client` advertises the client API base URL to Matrix clients.
   - `/.well-known/matrix/server` advertises the federation endpoint (`your-domain:443`) to other homeservers.

> [!NOTE]
> The wildcard `Access-Control-Allow-Origin` header on those endpoints is required by the Matrix `.well-known` spec so browser-based clients can read them cross-origin.

## Federation

This template delegates federation to port `443` via `/.well-known/matrix/server`, so other homeservers reach you over standard HTTPS through BunkerWeb — no need to expose the dedicated Matrix federation port `8448`. If you prefer serving federation directly on `8448` instead of delegating, drop the `/.well-known/matrix/server` block and open that port to Tuwunel separately.

## Install

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#install) for the web UI or
     plugin bundle method. When prompted for custom configs, upload `configs/server-http/well-known.conf` and `configs/modsec-crs/tuwunel_false_positives.conf`.
2. **Assign the template** to your Matrix service via the easy-mode UI or by setting `USE_TEMPLATE=tuwunel`.
3. **Customize the settings** highlighted in the template steps (server name, upstream host, TLS options) and edit `well-known.conf` as described above.
4. **Reload the service** and verify that `https://<your-domain>/.well-known/matrix/client` and `/.well-known/matrix/server` return your delegation JSON.

## Settings

- Update `REVERSE_PROXY_HOST` to point at your Tuwunel instance (e.g. `http://tuwunel:8008`).
- `REVERSE_PROXY_INTERCEPT_ERRORS` is `no` so Matrix clients receive Tuwunel's JSON errors (`M_FORBIDDEN`, `M_NOT_FOUND`) instead of BunkerWeb's HTML error page.
- Adjust `MAX_CLIENT_SIZE` if you need to support larger media uploads.
- Tuwunel's own `max_request_size` defaults to 24 MiB, below `MAX_CLIENT_SIZE`; raise it in `tuwunel.toml` to match when you need larger uploads.
- Tune `LIMIT_REQ_RATE` if your homeserver serves many clients and the default `15r/s` is too strict.
- The crawler whitelist is disabled by default (`USE_WHITELIST=no`): a Matrix homeserver serves an API, not crawlable pages, so rDNS crawler bypass adds no value. Set `USE_WHITELIST=yes` (and add `WHITELIST_RDNS`) only if you have a specific reason.

### ModSecurity exclusions

`configs/modsec-crs/tuwunel_false_positives.conf` removes three CRS false positives that Matrix
clients trigger. Each rule is limited to one method and one path shape:

- `920440` (restricted file extension) on `GET`, `PUT`, and `DELETE` of
  `/_matrix/client/{r0,v3}/profile/<id>`, `.../rooms/<room>/typing/<id>`, and
  `.../directory/room/<alias>` when the user id or alias ends in `.com`.
  Identifiers ending in any other restricted extension stay blocked.
- `920420` (content type) on `POST` and `PUT` of `/_matrix/media/{r0,v3}/upload`, the media
  upload sent as `application/octet-stream`.
- `942100` (SQL injection) on the `body` and `formatted_body` JSON fields of
  `PUT /_matrix/client/{r0,v3}/rooms/<room>/send/<type>/<txn>`, which hold chat text that can
  mention SQL. Every other SQL injection rule, field, and path stays active.

> [!NOTE]
> Chat messages that contain shell commands still trigger the `930120`, `932xxx`, and `941xxx`
> rules; the template does not exclude them because that would need an exclusion on all chat text.

## Files

- `template.json` – BunkerWeb template definition containing default settings, configs, and guided steps.
- `configs/modsec-crs/tuwunel_false_positives.conf` – ModSecurity CRS exclusions for Matrix clients.
- `configs/server-http/well-known.conf` – Serves the `/.well-known/matrix/client` and `/.well-known/matrix/server` delegation endpoints.

## Validation

Run `jq . template.json` to confirm the JSON definition is valid before importing via the UI.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
