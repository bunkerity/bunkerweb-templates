<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/jellyfin.svg" alt="Jellyfin logo" width="96" height="96" />
</p>

<h1 align="center">Jellyfin</h1>

<p align="center">Media streaming template with reverse proxy tuning</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Jellyfin-10.11.6-248651?labelColor=1d6f93&style=flat" alt="Jellyfin 10.11.6" />
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

| Fact        | Value                                          |
| ----------- | ---------------------------------------------- |
| Template id | `jellyfin`                                     |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Jellyfin 10.11.6 |
| Configs     | 1 (modsec-crs)                                 |
| Rule ids    | `9859000` to `9859999`                         |
| Upstream    | `http://jellyfin:8096`                         |

## Prerequisites

- A Jellyfin backend reachable from the BunkerWeb instance (container hostname
  or static IP). Tested with the official `jellyfin/jellyfin:10.11.6` image.
- Access to the BunkerWeb web UI (easy mode) to assign template settings.
- In Jellyfin, open Dashboard, Networking, and add the BunkerWeb container's
  hostname or IP address to **Known proxies**, then restart Jellyfin (the page
  says it needs a reboot). Without it Jellyfin shows every client under the
  address of the BunkerWeb container.

## Install

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#install)
     for the web UI or plugin bundle method.
2. **Assign the template** to the site serving Jellyfin: when you create the
   service in easy mode, pick Jellyfin in the Templates menu above the steps.
3. **Adjust TLS automation** in step 1: type your Jellyfin host name in
   Server name. For a host with no public DNS (a LAN name), turn Automatic
   Let's Encrypt off, or turn on Use custom certificate and fill the
   certificate and key. BunkerWeb 1.6.17 ignores the Let's Encrypt switch when
   you first create the service: save, open the service again, switch it off
   and save again.
4. **Update upstream targets** in step 2: point the first Host field at your
   Jellyfin service and confirm connectivity from BunkerWeb. The second Host,
   URL and websocket fields are the `/socket` entry: change only its host, to
   the same address.
5. **Review streaming limits**: keep `REVERSE_PROXY_BUFFERING=no` and the
   elevated timeouts unless your deployment has specific limits.
6. **Save the service.** Saving applies it, with no reload step: the scheduler
   pushes the change to BunkerWeb within about a minute. Then play a test
   video to ensure websocket connections, transcoding, and dashboard actions
   work through the proxy.

## Settings

- `MAX_CLIENT_SIZE` also limits pictures. Jellyfin sends a poster or backdrop
  as base64 text, a third larger than the file, so the largest picture you can
  upload is about three quarters of it: 15 MiB at the default `20M`. A larger
  upload gets `413`. Raise it for larger pictures.
- Keep `REVERSE_PROXY_WS_1=yes` for `/socket` so Jellyfin websockets upgrade
  correctly.
- `LIMIT_REQ_RATE` limits requests per client address to one exact path
  (15r/s by default), not per stream. Raise it if several users behind one
  address open the same URL. Requests refused with `429` count as bad
  behavior, and repeated ones ban that address for a day.
- `CONTENT_SECURITY_POLICY` carries `media-src 'self' blob:`. Direct play
  needs it: without it, a service on plain HTTP silently falls back to
  transcoding on every play. Modify the policy and `PERMISSIONS_POLICY` only if
  you embed Jellyfin in another application and need additional origins.
- A custom CSS import line (Dashboard, Branding) saves, but the browser loads the
  stylesheet only if `CONTENT_SECURITY_POLICY` allows its host. With the default
  `style-src 'self' 'unsafe-inline'` the theme does not load. Add the host after
  `'unsafe-inline'`, for example `https://cdn.jsdelivr.net`, and add the host that
  serves the web fonts to `font-src`.
- To add an exclusion of your own, open Configs in the web UI, choose Create
  Custom Config, pick your service, set the type to `MODSEC_CRS`, name it,
  paste the rule and save.

### What the exceptions allow

Every rule in the config is limited to one request shape. A request that
differs in path, method or field keeps the full CRS checks.

| Rule      | Lets through                                                         | Rule turned off, where                              |
| --------- | -------------------------------------------------------------------- | --------------------------------------------------- |
| `9859100` | The web client start-up file `/web/config.json`                      | `930130` on that exact path                         |
| `9859110` | Poster, backdrop and profile picture uploads (Edit images, profile)  | `920420`, on `POST` to the picture routes, `image/*` only |
| `9859120` | Branding custom CSS with import lines for fonts and community themes | `941130` and `941170` on the `CustomCss` field of `POST /System/Configuration/branding` |

Images written inline in the custom CSS text are still refused: host the file
and link to it.

## Files

- `template.json` – Template definition with steps for TLS, upstreams, and
  header tuning.
- `configs/modsec-crs/jellyfin_false_positives.conf` – Removes the CRS rules
  that block the web client start-up file, picture uploads and custom CSS
  with import lines.

## Validation

- Run `jq . template.json` to ensure the definition is valid JSON.
- Test a dashboard login, library scan, and multiple video streams through
  BunkerWeb to confirm timeouts, websockets, and rate limits behave as expected.
- Tested on BunkerWeb 1.6.17 with Jellyfin 10.11.6, in easy mode, over plain HTTP
  and HTTPS: sign-in, direct play of a WebM clip, poster and backdrop upload
  through Edit images, and a Branding custom CSS save with two import lines.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
