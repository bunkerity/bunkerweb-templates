# Synapse Template

## Overview

Provision a BunkerWeb configuration for [Synapse](https://element-hq.github.io/synapse/),
the reference Matrix homeserver. The template terminates HTTPS, reverse proxies
the `/_matrix/` client and federation API, raises the upload limit for media,
applies rate limiting, and serves the `.well-known` delegation documents so
clients and federating servers can discover the homeserver.

## Prerequisites

- A running Synapse instance reachable from the BunkerWeb instance (default
  client/federation listener on port `8008`).
- A domain you control, with DNS pointing the served hostname at BunkerWeb.
- Access to the BunkerWeb UI or environment variables to assign template
  settings.

## Files

- `template.json` – Template definition with steps for TLS, reverse proxy, HTTP
  limits, whitelisting, and well-known delegation.
- `configs/modsec-crs/synapse_false_positives.conf` – ModSecurity CRS exclusions for Matrix clients.
- `configs/server-http/well-known.conf` – Serves `/.well-known/matrix/client`
  and `/.well-known/matrix/server` for delegation. **Replace the placeholder
  `example.com` with your own Matrix hostname before importing.**

## Well-known / DNS delegation

Matrix uses `.well-known` delegation to publish your Matrix hostname. Edit
`configs/server-http/well-known.conf` and replace every `example.com` with the
hostname where Synapse is reachable. The
`server_name` in your Synapse `homeserver.yaml` must match the delegation you
publish here.

## TURN / VoIP (calls)

The TURN server used for Matrix voice and video calls should be set up through
your firewall (direct port forwarding to the TURN/coturn service) rather than
routed through BunkerWeb. TURN/VoIP traffic proxied through the reverse proxy
can cause call interruptions and failed calls; keeping it outside BunkerWeb
gives the best performance and reliability. This template intentionally does not
proxy TURN.

## Upload size

`MAX_CLIENT_SIZE` is set to `50m` so media uploads succeed. Align this with the
`max_upload_size` configured in your Synapse `homeserver.yaml`, and raise both
if you expect larger files.

## ModSecurity exclusions

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

Chat messages that contain shell commands still trigger the `930120`, `932xxx`, and `941xxx`
rules; the template does not exclude them because that would need an exclusion on all chat text.

## Setup

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#installing-templates)
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

## Validation

- Run `jq . template.json` to ensure the definition is valid JSON.
- Confirm `curl https://<server_name>/.well-known/matrix/client` and
  `/.well-known/matrix/server` return the expected JSON.
- Use the [Matrix Federation Tester](https://federationtester.matrix.org/) to
  confirm delegation and federation resolve correctly.
