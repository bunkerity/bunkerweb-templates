<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/synapse-light.svg">
    <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/synapse.svg" alt="Synapse logo" width="96" height="96" />
  </picture>
</p>

<h1 align="center">Synapse</h1>

<p align="center">Matrix homeserver template with well-known delegation</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#well-known--dns-delegation">Well-known / DNS</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#files">Files</a> ·
  <a href="#validation">Validation</a>
</p>

Provision a BunkerWeb configuration for [Synapse](https://element-hq.github.io/synapse/),
the reference Matrix homeserver. The template terminates HTTPS, reverse proxies
the `/_matrix/` client and federation API, raises the upload limit for media,
applies rate limiting, and serves the `.well-known` delegation documents so
clients and federating servers can discover the homeserver.

## At a glance

| Fact        | Value                          |
| ----------- | ------------------------------ |
| Template id | `synapse`                      |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4  |
| Configs     | 2 (modsec-crs, server-http)    |
| Rule ids    | `9872000` to `9872999`         |
| Upstream    | `http://mysynapse-server:8008` |

## Prerequisites

- A running Synapse instance reachable from the BunkerWeb instance (default
  client/federation listener on port `8008`).
- A domain you control, with DNS pointing the served hostname at BunkerWeb.
- Access to the BunkerWeb UI or environment variables to assign template
  settings.

## Well-known / DNS delegation

Matrix uses `.well-known` delegation to publish your Matrix hostname. Edit
`configs/server-http/well-known.conf` and replace every `example.com` with the
hostname where Synapse is reachable. The
`server_name` in your Synapse `homeserver.yaml` must match the delegation you
publish here.

## Install

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#install)
     for the web UI or plugin bundle method.
2. **Assign the template** to the site serving Synapse
   (`USE_TEMPLATE=synapse` or choose it in the UI).
3. **Adjust TLS automation** so `SERVER_NAME` and certificate options reflect
   the domain you expose.
4. **Update upstream target**: point `REVERSE_PROXY_HOST` at your Synapse
   listener (e.g. `http://mysynapse-server:8008`).
5. **Edit the delegation config** as described above so the well-known
   documents advertise your real Matrix hostname.
6. **Reload BunkerWeb** and verify login, federation, and media upload work
   through the proxy.

## Settings

### TURN / VoIP (calls)

The TURN server used for Matrix voice and video calls should be set up through
your firewall (direct port forwarding to the TURN/coturn service) rather than
routed through BunkerWeb. TURN/VoIP traffic proxied through the reverse proxy
can cause call interruptions and failed calls; keeping it outside BunkerWeb
gives the best performance and reliability. This template intentionally does not
proxy TURN.

### Upload size

> [!TIP]
> `MAX_CLIENT_SIZE` is set to `50m` so media uploads succeed. Align this with the
> `max_upload_size` configured in your Synapse `homeserver.yaml`, and raise both
> if you expect larger files.

### ModSecurity exclusions

`configs/modsec-crs/synapse_false_positives.conf` removes three CRS false positives that Matrix
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

- `template.json` – Template definition with steps for TLS, reverse proxy, HTTP
  limits, whitelisting, and well-known delegation.
- `configs/modsec-crs/synapse_false_positives.conf` – ModSecurity CRS exclusions for Matrix clients.
- `configs/server-http/well-known.conf` – Serves `/.well-known/matrix/client`
  and `/.well-known/matrix/server` for delegation. **Replace the placeholder
  `example.com` with your own Matrix hostname before importing.**

## Validation

- Run `jq . template.json` to ensure the definition is valid JSON.
- Confirm `curl https://<server_name>/.well-known/matrix/client` and
  `/.well-known/matrix/server` return the expected JSON.
- Use the [Matrix Federation Tester](https://federationtester.matrix.org/) to
  confirm delegation and federation resolve correctly.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
