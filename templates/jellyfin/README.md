# Jellyfin Template

## Overview

Provision a BunkerWeb configuration tailored for Jellyfin so HTTPS automation,
reverse proxy buffering, websocket upgrades, payload limits, and CRS exclusions
match typical media streaming workloads without sacrificing security headers.

## Prerequisites

- A Jellyfin backend reachable from the BunkerWeb instance (container hostname
  or static IP).
- Access to the BunkerWeb UI or environment variables to assign template
  settings.
- Confirm Jellyfin trusts the proxy IP (set `EnableReverseProxySupport` in
  Jellyfin if you use the built-in setting).

## Files

- `template.json` – Template definition with steps for TLS, upstreams, and
  header tuning.
- `configs/modsec-crs/jellyfin_false_positives.conf` – Removes CRS rules that
  interfere with the Jellyfin configuration API.

## Setup

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#install)
     for the web UI or plugin bundle method.
2. **Assign the template** to the site serving Jellyfin (`USE_TEMPLATE=jellyfin`
   or choose it in the UI).
3. **Adjust TLS automation** so `SERVER_NAME` and certificate options reflect
   the domains you expose.
4. **Update upstream targets**: point `REVERSE_PROXY_HOST` (and the websocket
   entry) to your Jellyfin service and confirm connectivity from BunkerWeb.
5. **Review streaming limits**: keep `REVERSE_PROXY_BUFFERING=no` and the
   elevated timeouts unless your deployment has specific limits.
6. **Reload BunkerWeb** and play a test video to ensure websocket connections,
   transcoding, and dashboard actions work through the proxy.

## Customization Tips

- Raise `MAX_CLIENT_SIZE` if you proxy uploads larger than 20 MiB (for example
  library imports).
- Keep `REVERSE_PROXY_WS_1=yes` for `/socket` so Jellyfin websockets upgrade
  correctly.
- Adjust `LIMIT_REQ_RATE` to match your concurrent stream expectations; set it
  higher for multi-user households.
- Modify `CONTENT_SECURITY_POLICY` and `PERMISSIONS_POLICY` only if you embed
  Jellyfin in another application and need additional origins.
- Edit `configs/modsec-crs/jellyfin_false_positives.conf` if future CRS updates
  require different rule IDs.

## Validation

- Run `jq . template.json` to ensure the definition is valid JSON.
- Test a dashboard login, library scan, and multiple video streams through
  BunkerWeb to confirm timeouts, websockets, and rate limits behave as expected.
