# Tuwunel Template

## Overview

Provision a curated BunkerWeb configuration for [Tuwunel](https://github.com/matrix-construct/tuwunel), a lightweight Matrix homeserver written in Rust and the maintained successor to conduwuit. This template ships sensible defaults for TLS, reverse proxying with WebSocket support, long federation/sync timeouts, request throttling, and `.well-known` client/server delegation so you can expose a Matrix homeserver through BunkerWeb quickly.

## Prerequisites

- A running Tuwunel homeserver reachable on your network (its client/federation API listens on `8008` by default).
- The BunkerWeb UI or the ability to edit multisite settings directly.
- The public domain name that will act as your Matrix server name (the part after the `:` in your user IDs, e.g. `@alice:example.com`).

## Files

- `template.json` – BunkerWeb template definition containing default settings, configs, and guided steps.
- `configs/server-http/well-known.conf` – Serves the `/.well-known/matrix/client` and `/.well-known/matrix/server` delegation endpoints.

## DNS and .well-known Delegation

Matrix separates the **server name** (what appears in user IDs) from the **hostname that actually serves traffic**. Delegation lets `example.com` be your server name while BunkerWeb terminates TLS and proxies to Tuwunel.

1. Point an `A`/`AAAA` record for your server name (e.g. `example.com`) at the host running BunkerWeb.
2. Edit `configs/server-http/well-known.conf` and **replace every `example.com` with your own homeserver domain**. The placeholders are not usable as-is:
   - `/.well-known/matrix/client` advertises the client API base URL to Matrix clients.
   - `/.well-known/matrix/server` advertises the federation endpoint (`your-domain:443`) to other homeservers.

The wildcard `Access-Control-Allow-Origin` header on those endpoints is required by the Matrix `.well-known` spec so browser-based clients can read them cross-origin.

## Federation

This template delegates federation to port `443` via `/.well-known/matrix/server`, so other homeservers reach you over standard HTTPS through BunkerWeb — no need to expose the dedicated Matrix federation port `8448`. If you prefer serving federation directly on `8448` instead of delegating, drop the `/.well-known/matrix/server` block and open that port to Tuwunel separately.

## Setup

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#installing-templates) for the web UI or
     plugin bundle method. When prompted for custom configs, upload `configs/server-http/well-known.conf`.
2. **Assign the template** to your Matrix service via the easy-mode UI or by setting `USE_TEMPLATE=tuwunel`.
3. **Customize the settings** highlighted in the template steps (server name, upstream host, TLS options) and edit `well-known.conf` as described above.
4. **Reload the service** and verify that `https://<your-domain>/.well-known/matrix/client` and `/.well-known/matrix/server` return your delegation JSON.

## Customization Tips

- Update `REVERSE_PROXY_HOST` to point at your Tuwunel instance (e.g. `http://tuwunel:8008`).
- `REVERSE_PROXY_INTERCEPT_ERRORS` is `no` so Matrix clients receive Tuwunel's JSON errors (`M_FORBIDDEN`, `M_NOT_FOUND`) instead of BunkerWeb's HTML error page.
- Adjust `MAX_CLIENT_SIZE` if you need to support larger media uploads.
- Tuwunel's own `max_request_size` defaults to 24 MiB, below `MAX_CLIENT_SIZE`; raise it in `tuwunel.toml` to match when you need larger uploads.
- Tune `LIMIT_REQ_RATE` if your homeserver serves many clients and the default `15r/s` is too strict.
- The crawler whitelist is disabled by default (`USE_WHITELIST=no`): a Matrix homeserver serves an API, not crawlable pages, so rDNS crawler bypass adds no value. Set `USE_WHITELIST=yes` (and add `WHITELIST_RDNS`) only if you have a specific reason.

## Validation

Run `jq . template.json` to confirm the JSON definition is valid before importing via the UI.
