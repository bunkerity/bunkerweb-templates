<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/netbird.svg" alt="NetBird logo" width="96" height="96" />
</p>

<h1 align="center">NetBird</h1>

<p align="center">Self-hosted template with gRPC and websocket routing</p>

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

Provision a BunkerWeb configuration tailored for a self-hosted NetBird stack.
The template wires relay, websocket, REST API, OAuth2, and gRPC endpoints with
custom headers and long-lived timeout defaults suited for control-plane traffic.

## At a glance

| Fact        | Value                         |
| ----------- | ----------------------------- |
| Template id | `netbird`                     |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4 |
| Requires    | `BunkerWeb >= 1.6.9~rc1`      |
| Configs     | 1 (modsec-crs)                |
| Rule ids    | `9862000` to `9862999`        |
| Upstream    | `http://netbird-server`       |

## Prerequisites

- BunkerWeb `>= 1.6.9~rc1` (gRPC proxy settings used by this template were
  introduced in that release line).
- A reachable `netbird-server` upstream for relay, API, OAuth2, and gRPC
  routes.
- A reachable `netbird-dashboard` upstream for the fallback `/` route.
- Access to the BunkerWeb UI or multisite environment variables.

## Install

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#install)
     for the web UI or plugin bundle method.
2. **Assign the template** to your NetBird site (`USE_TEMPLATE=netbird` or
   select it in the UI).
3. **Adjust domains and TLS** so `SERVER_NAME` and certificate options match
   your deployment.
4. **Confirm upstreams** by updating `REVERSE_PROXY_HOST*`, `GRPC_HOST*`, and
   `REVERSE_PROXY_HOST_999` if your service names or ports differ.
5. **Reload BunkerWeb** and verify dashboard access, OAuth2 flows, relay
   traffic, and websocket connectivity.

## Settings

- Keep `REVERSE_PROXY_WS=yes` and `REVERSE_PROXY_WS_1=yes` for relay and
  websocket routes.
- Use `REVERSE_PROXY_HEADERS` to pass custom headers like `X-Real-Port` to upstreams (the relay needs to know the client's original port / address).
- If you run NetBird on non-default ports, update both HTTP and gRPC upstream
  hosts together.
- Lower `LIMIT_REQ_RATE` or `LIMIT_CONN_MAX_HTTP*` only after confirming your
  expected number of peers and API clients.
- Adjust `configs/modsec-crs/netbird_false_positives.conf` if CRS rule IDs change
  in your deployment.

## Files

- `template.json` - Template definition with guided steps for TLS, proxy, and
  gRPC settings.
- `configs/modsec-crs/netbird_false_positives.conf` - CRS exclusions for known
  NetBird OAuth2 false positives. NetBird's embedded identity provider (Dex)
  serves `/oauth2/`. The file removes `931100` and `934110` (a redirect URI that
  holds an IP address or a URL) for `GET` and `POST` on `/oauth2/auth` and the
  connector paths below it (`/oauth2/auth/<connector>`,
  `/oauth2/auth/<connector>/login`), and for `POST /oauth2/token`. Other
  methods, names that only start the same way (`/oauth2/authx`,
  `/oauth2/tokens`) and the other `/oauth2/` paths stay under the full CRS.
  Both rules remove the two rule ids for the whole request on those paths, so
  the other arguments of those requests are not checked by them either.

## Validation

- Run `jq . template.json` to verify JSON syntax.
- Test `/relay`, `/ws-proxy/`, `/api/`, `/oauth2/`, and dashboard `/` through
  BunkerWeb to confirm routing and timeout behavior.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
