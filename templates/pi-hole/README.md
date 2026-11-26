<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/pi-hole.svg" alt="Pi-hole logo" width="96" height="96" />
</p>

<h1 align="center">Pi-hole</h1>

<p align="center">Reverse proxy template with admin UI and API tuning</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#host-header">Host header</a> ·
  <a href="#validation">Validation</a> ·
  <a href="#modsecurity-exceptions">ModSecurity exceptions</a> ·
  <a href="#root-domain-caveat">Root domain caveat</a>
</p>

This template reverse-proxies the Pi-hole v6 **web admin UI and REST API** (paths `/admin/`
and `/api/`) through BunkerWeb. Pi-hole's DNS service is out of scope: keep serving DNS directly
from Pi-hole on port 53 and only front the HTTP interface with BunkerWeb. Pi-hole v5 serves its
API at `/admin/api.php`, which the template does not cover.

## At a glance

| Fact        | Value                                      |
| ----------- | ------------------------------------------ |
| Template id | `pi-hole`                                  |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4              |
| Pi-hole     | v6.6 (core v6.6, web v6.6.1, FTL v6.9)     |
| Configs     | 1 (modsec-crs)                             |
| Rule ids    | `9868000` to `9868999`                     |

## Install

1. Follow the repository's [installation guide](../../README.md#install)
   for the web UI or plugin bundle method.
2. In the web UI, open **Services**, create a service in easy mode and pick **Pi-hole** in the
   **Templates** dropdown above the steps.
3. Fill in the steps:
   - **Domain and TLS**: replace the example domain and the example Let's Encrypt e-mail with your
     own. If the name is not public (a lab or LAN name), switch Let's Encrypt off. BunkerWeb 1.6.17
     drops that switch when it creates the service: save once, open the service again, switch it
     off and save again.
   - **Reverse proxy**: replace the upstream with the address of the Pi-hole web port, for example
     `http://pihole:80` when BunkerWeb and Pi-hole share a Docker network (the container name
     works as the host name), or `http://192.168.1.10:80` for a Pi-hole on another machine.
4. Save the service. BunkerWeb applies it on its own, no reload needed. Then verify the admin UI
   and REST API through the proxy (see [Validation](#validation)).

## Settings

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

Pi-hole v6.6 answers `404` on `/` unless the request carries `Host: pi.hole`, because its web
server redirects by host name (`/admin/` answers with any host name). The template sets
`REVERSE_PROXY_CUSTOM_HOST=pi.hole`, so BunkerWeb sends `pi.hole` as the upstream `Host` header
instead of the public host name, and `/` redirects to `/admin/`. The setting is unsuffixed:
`REVERSE_PROXY_CUSTOM_HOST_1` has no effect.

## Validation

- Run `jq . template.json` to confirm the template definition is valid JSON.
- Sign in to the admin UI and exercise its API-backed dashboard through
  BunkerWeb to confirm routing and rate limits.
- After you save settings in Pi-hole, expect about a second of `502` while Pi-hole restarts its
  web server (measured: it starts about 1 s after the save and lasts 1 to 1.5 s).

## ModSecurity exceptions

The config `pihole_false_positives.conf` has three rules, all for `/api/` only (the web UI sends
everything it saves there):

- `9868101` turns off the file inclusion checks, for paths in dnsmasq lines, group names and
  filters.
- `9868102` turns off the command injection checks, for regex filters with command-like words.
- `9868107` lets a free-text comment contain a semicolon followed by an SQL word, on create and
  edit of domains, adlists, groups and clients. Other fields, other routes and the other SQL
  checks stay on.

The template no longer exempts `/admin/` (the pages themselves) or any content type: the web UI
works without that. A script or app that calls the API with a `text/plain` or
`application/octet-stream` body gets `403`: send `application/json`, or a multipart form for a
Teleporter file.

To change a rule, open **Configs** in the web UI, create a custom config for your service with
type `MODSEC_CRS`, and paste the rule there.

## Root Domain Caveat

It is possible to serve Pi-hole at the root of your domain, but this adds an
extra attack surface and is **not advised**. Additionally, you will always be
redirected to `/admin/` — this is how Pi-hole is built and cannot be changed;
using BunkerWeb, you would need to modify Pi-hole itself to achieve otherwise.

If you still want to proceed, use the raw config below instead of the
`template.json`: paste it in the service's Raw mode in the web UI.

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

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
