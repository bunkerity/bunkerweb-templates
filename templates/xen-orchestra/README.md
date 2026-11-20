<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/xen-orchestra.svg" alt="Xen Orchestra logo" width="96" height="96" />
</p>

<h1 align="center">Xen Orchestra</h1>

<p align="center">Reverse proxy template with WebSocket and rate limiting defaults</p>

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

Provision a BunkerWeb configuration tailored for Xen Orchestra so HTTPS
automation, reverse proxy, WebSocket upgrades, and request rate limiting work
out of the box. Bad behavior detection is disabled to avoid false positives
caused by XO's frequent API polling.

## At a glance

| Fact        | Value                         |
| ----------- | ----------------------------- |
| Template id | `xen-orchestra`               |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4 |
| Configs     | 1 (modsec-crs)                |
| Rule ids    | `9879000` to `9879999`        |
| Upstream    | `https://192.168.1.1`         |

## Prerequisites

- A Xen Orchestra backend reachable from the BunkerWeb instance (typically
  HTTPS with a self-signed certificate).
- Access to the BunkerWeb UI or environment variables to assign template
  settings.
- Confirm XO listens on HTTPS and that `REVERSE_PROXY_SSL_SNI` is set to `no`
  when using a self-signed upstream certificate.

## Install

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#install)
     for the web UI or plugin bundle method.
2. **Assign the template** to the site serving Xen Orchestra
   (`USE_TEMPLATE=xen-orchestra` or choose it in the UI).
3. **Adjust TLS automation** so `SERVER_NAME` and certificate options reflect
   the domain you expose.
4. **Update upstream target**: point `REVERSE_PROXY_HOST` to your Xen Orchestra
   instance (e.g. `https://192.168.1.1`) and confirm connectivity from
   BunkerWeb.
5. **Review rate limits**: `LIMIT_REQ_RATE` applies to every URL; raise it if many users or
   automation scripts share the proxy.
6. **Reload BunkerWeb** and log into Xen Orchestra to verify dashboard access,
   VM console websockets, and live migration actions work through the proxy.

## Settings

- Raise or lower `MAX_CLIENT_SIZE` depending on whether you transfer ISOs or
  large files through the proxy.
- Adjust `LIMIT_REQ_RATE` if you have many concurrent users or heavy automation scripts. XO's
  JSON-RPC API runs over a WebSocket on `/api/`, so only the upgrade request counts toward it.
- `USE_BAD_BEHAVIOR` is set to `no` because XO's polling pattern triggers false
  positives; re-enable only after confirming it does not block legitimate
  traffic.
- Modify `CONTENT_SECURITY_POLICY` and `PERMISSIONS_POLICY` if you embed XO in
  an iframe or need additional origins.
- Set `REVERSE_PROXY_SSL_SNI` to `yes` if your upstream uses a valid
  certificate matching the hostname.
- Edit `configs/modsec-crs/xen-orchestra_false_positives.conf` if future CRS
  updates require different rule IDs.

## Files

- `template.json` – Template definition with steps for TLS, reverse proxy,
  rate limiting, and security tuning.
- `configs/modsec-crs/xen-orchestra_false_positives.conf` – CRS exclusion of rule `920420` for
  `POST /api/<token>`, the `application/octet-stream` upload XO uses for config, VM and disk
  imports. It is a `modsec-crs` config on purpose: those load before the CRS rules, which is the
  only point where the removal still applies.

## Validation

- Run `jq . template.json` to ensure the definition is valid JSON.
- Test a dashboard login, VM console session (websocket), and a live migration
  or snapshot action through BunkerWeb to confirm proxying, rate limits, and
  security settings behave as expected.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
