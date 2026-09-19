# NetBird Template

## Overview

Provision a BunkerWeb configuration tailored for a self-hosted NetBird stack.
The template wires relay, websocket, REST API, OAuth2, and gRPC endpoints with
custom headers and long-lived timeout defaults suited for control-plane traffic.

## Prerequisites

- BunkerWeb `>= 1.6.9~rc1` (gRPC proxy settings used by this template were
  introduced in that release line).
- A reachable `netbird-server` upstream for relay, API, OAuth2, and gRPC
  routes.
- A reachable `netbird-dashboard` upstream for the fallback `/` route.
- Access to the BunkerWeb UI or multisite environment variables.

## Files

- `template.json` - Template definition with guided steps for TLS, proxy, and
  gRPC settings.
- `configs/modsec/netbird_false_positives.conf` - CRS exclusions for known
  NetBird OAuth2 false positives.

## Setup

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#installing-templates)
     for the web UI or plugin bundle method.
2. **Assign the template** to your NetBird site (`USE_TEMPLATE=netbird` or
   select it in the UI).
3. **Adjust domains and TLS** so `SERVER_NAME` and certificate options match
   your deployment.
4. **Confirm upstreams** by updating `REVERSE_PROXY_HOST*`, `GRPC_HOST*`, and
   `REVERSE_PROXY_HOST_999` if your service names or ports differ.
5. **Reload BunkerWeb** and verify dashboard access, OAuth2 flows, relay
   traffic, and websocket connectivity.

## Customization Tips

- Keep `REVERSE_PROXY_WS=yes` and `REVERSE_PROXY_WS_1=yes` for relay and
  websocket routes.
- Use `REVERSE_PROXY_HEADERS` to pass custom headers like `X-Real-Port` to upstreams (the relay needs to know the client's original port / address).
- If you run NetBird on non-default ports, update both HTTP and gRPC upstream
  hosts together.
- Lower `LIMIT_REQ_RATE` or `LIMIT_CONN_MAX_HTTP*` only after confirming your
  expected number of peers and API clients.
- Adjust `configs/modsec/netbird_false_positives.conf` if CRS rule IDs change
  in your deployment.

## Validation

- Run `jq . template.json` to verify JSON syntax.
- Test `/relay`, `/ws-proxy/`, `/api/`, `/oauth2/`, and dashboard `/` through
  BunkerWeb to confirm routing and timeout behavior.
