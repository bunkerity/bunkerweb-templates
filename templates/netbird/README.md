<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/netbird.svg" alt="NetBird logo" width="96" height="96" />
</p>

<h1 align="center">NetBird</h1>

<p align="center">Self-hosted template with gRPC and websocket routing</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/NetBird-0.83.1-248651?labelColor=1d6f93&style=flat" alt="NetBird 0.83.1" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#files">Files</a> ·
  <a href="#validation">Validation</a>
</p>

Put a self-hosted [NetBird](https://netbird.io/) behind BunkerWeb with ModSecurity and the OWASP
CRS on: the dashboard, the REST API, the embedded identity provider (`/oauth2/`), the relay and
websocket routes, and the gRPC Management and Signal services that peers use. The template adds
long-lived timeouts, the `X-Real-Port` header the relay needs, and CRS exceptions for NetBird's
sign-in, for object names typed in the dashboard and for posture-check paths.

## At a glance

| Fact        | Value                                   |
| ----------- | --------------------------------------- |
| Template id | `netbird`                               |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, NetBird 0.83.1 |
| Requires    | `BunkerWeb >= 1.6.9~rc1`                |
| Configs     | 1 (modsec-crs)                          |
| Rule ids    | `9862000` to `9862999`                  |
| Upstream    | `http://netbird-server`                 |

## Prerequisites

- BunkerWeb `>= 1.6.9~rc1` (gRPC proxy settings used by this template were
  introduced in that release line).
- A reachable `netbird-server` upstream: NetBird's combined server (`netbirdio/netbird-server`,
  tested with 0.83.1). It serves the REST API, gRPC (Management and Signal), the relay and the
  embedded identity provider (`/oauth2/`) on one HTTP port, 80 here. Older multi-container setups
  (separate management, signal, relay and coturn containers) and external identity providers are
  not covered by these routes.
- A reachable `netbird-dashboard` upstream (`netbirdio/dashboard`, tested with 2.98.0) for the
  fallback `/` route.
- In NetBird's own configuration, `exposedAddress`, `auth.issuer`, the dashboard redirect URIs and
  the dashboard's API endpoints use the public HTTPS address that BunkerWeb serves.
- STUN does not go through BunkerWeb, because it is UDP: publish the STUN port of the NetBird host
  (`stunPorts`, 3478 by default) on the public address. Relaying over QUIC (UDP) is not proxied
  either: peers use the websocket relay through `/relay`.
- Access to the BunkerWeb web UI.

## Install

1. **Raise the global request body timeout.** In the web UI open **Global settings**, **General**,
   set `CLIENT_BODY_TIMEOUT` to `1d` and save. BunkerWeb's default is `10s`, and it resets NetBird's
   idle gRPC streams every 10 seconds: peers log `stream terminated by RST_STREAM` and BunkerWeb
   logs `408` on `/management.ManagementService/Job`. A template cannot set it, because it is a
   global setting. Any finite value ends an idle stream at that interval (one reset a minute with
   `60s`), so `1d` is the smallest value that held in testing. It applies to every service of the
   instance: a client that opens a request and stops sending is now kept for up to a day, so keep
   `LIMIT_CONN_MAX_HTTP*` on your other services.
2. **Import the template.** Follow the repository's [installation guide](../../README.md#install)
   for the web UI or plugin bundle method.
3. **Create the service.** Pick `Netbird` (the template id) in the **Templates** dropdown of a new
   service.
4. **Set the domain and TLS** in step 1: replace the example domain with the public NetBird
   address, and turn Let's Encrypt off if that name is not public. A custom certificate goes in the
   same step.
5. **Confirm the upstreams** in step 2. The boxes have no label; they are, in order, `/relay`,
   `/ws-proxy/`, `/api/`, `/oauth2/`, the gRPC Signal service, the gRPC Management service and the
   dashboard (`/`). Change them together if your service names or ports differ.
6. **Save the service.** Saving applies it; there is no separate reload. Then check the dashboard,
   the sign-in, a peer registration and the relay.

## Settings

- Keep `REVERSE_PROXY_WS=yes` and `REVERSE_PROXY_WS_1=yes` for relay and websocket routes. The
  second one is a plain text box in easy mode: keep `yes`.
- Use `REVERSE_PROXY_HEADERS` to pass custom headers like `X-Real-Port` to upstreams (the relay needs to know the client's original port / address).
- If you run NetBird on non-default ports, update both HTTP and gRPC upstream
  hosts together.
- `LIMIT_REQ_RATE` (`15r/s`) counts requests per client address and per URL. A peer that
  (re)connects sends about eight gRPC requests in a few seconds, three of them to
  `/management.ManagementService/GetServerKey`. Peers behind one public address that reconnect
  together (a NetBird restart, a network blip) multiply that: in testing, 8 peers at once got
  `429` answers, and 12 peers at once got the address banned by the Bad behavior plugin (`403` for
  every peer behind it, 24 hours by default). Set `LIMIT_REQ_RATE` to at least three times the
  number of peers behind one address, for example `60r/s` for 20 peers. Lift a ban in **Bans**.
- Lower `LIMIT_CONN_MAX_HTTP*` only after confirming your expected number of peers and API clients.
- The gRPC routes (Management and Signal) run without ModSecurity: BunkerWeb turns it off on gRPC
  locations because it cannot inspect gRPC traffic reliably. The CRS covers the dashboard, the
  REST API, `/oauth2/` and the relay paths.
- To turn off one of the CRS exceptions, open step 5 of the service (or **Configs**, then the
  `netbird_false_positives` config) and delete its rule block.

## Files

- `template.json` - Template definition with guided steps for TLS, proxy, and
  gRPC settings.
- `configs/modsec-crs/netbird_false_positives.conf` - CRS exceptions for NetBird:
  - **OAuth2 sign-in.** NetBird's embedded identity provider (Dex) serves `/oauth2/`. The file
    removes `931100` and `934110` (a redirect URI that holds an IP address or a URL) for `GET` and
    `POST` on `/oauth2/auth` and the connector paths below it (`/oauth2/auth/<connector>`,
    `/oauth2/auth/<connector>/login`), and for `POST /oauth2/token`. Other methods, names that only
    start the same way (`/oauth2/authx`, `/oauth2/tokens`) and the other `/oauth2/` paths stay
    under the full CRS. Both rules are removed for the whole request on those paths, so they do
    not check the other arguments of those requests either.
  - **Names typed in the dashboard.** A group, policy, setup key, network, DNS zone or device
    called `SSH to prod servers` or `wget mirror` looks like a shell command to rules `932250` and
    `932260`. They skip the `name`, `description` and `network_id` fields (and the first four
    rules of a policy) on `POST` and `PUT` under `/api/groups`, `/api/policies`,
    `/api/setup-keys`, `/api/networks`, `/api/posture-checks`, `/api/routes`, `/api/dns`,
    `/api/peers` and `/api/users`, and only while every one of those fields has no pipe, semicolon,
    backtick, dollar sign, angle bracket, backslash or line break.
  - **Posture-check paths.** Rule `930120` skips the Windows path of the first ten processes on
    `POST` and `PUT` under `/api/posture-checks`, while each path holds only letters, digits,
    spaces, dots, dashes, underscores, slashes, backslashes, colons and brackets.

## Validation

- Tested on BunkerWeb 1.6.17 with NetBird 0.83.1 (combined server) and dashboard 2.98.0, two real
  peers over the gRPC routes, in easy mode.
- Test `/relay`, `/ws-proxy/`, `/api/`, `/oauth2/`, and dashboard `/` through
  BunkerWeb to confirm routing and timeout behavior.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
