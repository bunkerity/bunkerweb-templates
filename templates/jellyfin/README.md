<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/jellyfin.svg" alt="Jellyfin logo" width="96" height="96" />
</p>

<h1 align="center">Jellyfin</h1>

<p align="center">Media streaming template with reverse proxy tuning</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#files">Files</a> ·
  <a href="#validation">Validation</a>
</p>

Provision a BunkerWeb configuration tailored for Jellyfin so HTTPS automation,
reverse proxy buffering, websocket upgrades, payload limits, and CRS exclusions
match typical media streaming workloads without sacrificing security headers.

## At a glance

| Fact        | Value                         |
| ----------- | ----------------------------- |
| Template id | `jellyfin`                    |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4 |
| Configs     | 1 (modsec-crs)                |
| Rule ids    | `9859000` to `9859999`        |
| Upstream    | `http://jellyfin:8096`        |

## Prerequisites

- A Jellyfin backend reachable from the BunkerWeb instance (container hostname
  or static IP).
- Access to the BunkerWeb UI or environment variables to assign template
  settings.
- Confirm Jellyfin trusts the proxy IP (set `EnableReverseProxySupport` in
  Jellyfin if you use the built-in setting).

## Install

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

## Settings

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

## Files

- `template.json` – Template definition with steps for TLS, upstreams, and
  header tuning.
- `configs/modsec-crs/jellyfin_false_positives.conf` – Removes CRS rules that
  interfere with the Jellyfin configuration API.

## Validation

- Run `jq . template.json` to ensure the definition is valid JSON.
- Test a dashboard login, library scan, and multiple video streams through
  BunkerWeb to confirm timeouts, websockets, and rate limits behave as expected.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
