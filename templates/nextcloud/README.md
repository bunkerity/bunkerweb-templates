# Nextcloud Template

## Overview

Generate a hardened BunkerWeb configuration for Nextcloud. The template keeps TLS automation, WebDAV
verbs, uploads, caching, and ModSecurity CRS exclusions aligned with common Nextcloud guidance so your
instance stays reachable and protected.

## Prerequisites

- A reachable Nextcloud backend (container, VM, or service) that accepts traffic from BunkerWeb.
- Access to the BunkerWeb UI or the ability to apply settings via environment variables.
- Nextcloud configured for life behind a reverse proxy (see [Reverse proxy settings](#reverse-proxy-settings)).

## Files

- `template.json` – Template definition with default settings and the guided step layout.
- `configs/modsec-crs/nextcloud_false_positives.conf` – Enables the CRS v3 Nextcloud exclusion set.
  It is a `modsec-crs` config on purpose: those load **before** the CRS rules, which is the only point
  where `tx.crs_exclusions_nextcloud` is still read. The same file under `modsec/` would load after the
  CRS and do nothing.
  It also lets WebDAV `UNLOCK` under `/remote.php/dav/files/` carry a `Lock-Token` header: the plugin
  covers `If` but not `Lock-Token`, so CRS rule `920450` would otherwise answer `403`.

## Setup

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#installing-templates) for the web UI or
     plugin bundle method.
2. **Assign the template** to the service that fronts Nextcloud (set `USE_TEMPLATE=nextcloud` or pick it
   from the easy-mode UI).
3. **Adjust the TLS and domain entries** so `SERVER_NAME` and certificate options match your deployment.
4. **Point the reverse proxy** at your upstream host (for example `http://nextcloud:8080`) and confirm the
   service resolves from the BunkerWeb container or pod.
5. **Configure Nextcloud itself** for the proxy – see [Reverse proxy settings](#reverse-proxy-settings).
   Skipping this yields `http://` links, wrong client IPs in the logs, and broken federation.
6. **Review the HTTP hardening options** – keep the WebDAV verbs, upload size, rate limit, caching, and
   CRS plugin values or tailor them to your workload.
7. **Reload BunkerWeb** and run the checks in [Validation](#validation).

## Reverse proxy settings

BunkerWeb forwards `X-Forwarded-For`, `X-Real-IP`, and `X-Forwarded-Proto` to the upstream. Nextcloud
ignores all three until you tell it which proxy to trust, so set the following in `config.php` (or the
matching environment variables on the official image):

| `config.php` key   | Official image env var      | Why                                                        |
| ------------------ | --------------------------- | ---------------------------------------------------------- |
| `trusted_domains`  | `NEXTCLOUD_TRUSTED_DOMAINS` | Hostnames Nextcloud will answer for.                        |
| `trusted_proxies`  | `TRUSTED_PROXIES`           | BunkerWeb's address, so `X-Forwarded-For` is honoured.      |
| `overwriteprotocol`| `OVERWRITEPROTOCOL`         | Set to `https`; BunkerWeb terminates TLS, the upstream sees HTTP. |
| `overwritehost`    | `OVERWRITEHOST`             | Public hostname used in generated URLs.                     |
| `overwrite.cli.url`| `OVERWRITECLIURL`           | Base URL for `occ`, cron, and notification links.           |

Keep `trusted_proxies` as narrow as possible — the BunkerWeb container IP or its `/32`, not `0.0.0.0/0`.
Nextcloud treats a trusted proxy's `X-Forwarded-For` as authoritative, so an over-broad value lets any
client spoof its source IP and defeat both Nextcloud's brute-force protection and BunkerWeb's bans.

On the BunkerWeb side, leave `USE_REAL_IP` at its default `no`. Only enable it (with a matching
`REAL_IP_FROM`) when BunkerWeb is itself behind another proxy, load balancer, or CDN — otherwise you are
trusting client-supplied `X-Forwarded-For` headers.

## Upload sizing

`MAX_CLIENT_SIZE` defaults to `512m`, matching the `client_max_body_size 512M` in Nextcloud's own nginx
reference configuration. This is not the maximum file size your users can upload:

- Nextcloud's web UI and sync clients upload in **chunks** (`files.chunked_upload.max_size`, default
  100 MiB), and each chunk is a separate request. Upstream is explicit that "the Nextcloud sync client is
  not affected by these upload limits as it is uploading files in smaller chunks". `512m` leaves ample
  headroom over the default chunk size.
- Only clients that skip chunking — plain WebDAV mounts, `rclone`, `davfs2`, some third-party apps — send
  a whole file in one request and are actually bound by `MAX_CLIENT_SIZE`.

Raise it only if you need those non-chunked clients, and raise it deliberately. With ModSecurity enabled
(the BunkerWeb default), `MAX_CLIENT_SIZE` also drives `SecRequestBodyLimit`, and libmodsecurity 3
inspects request bodies **entirely in memory** — `SecRequestBodyInMemoryLimit` was removed in
ModSecurity 3.0, so there is no disk-backed inspection buffer. NGINX spills a large body to a temp file
and the connector then reads that file back into a single in-process buffer before any limit is applied.
The practical ceiling is therefore roughly `MAX_CLIENT_SIZE` × concurrent uploads of resident memory, and
values in the multi-gigabyte range are an out-of-memory risk rather than a capability. Prefer leaving
Nextcloud's chunked uploads enabled over raising this value.

## Rate limiting

| Slot | Path pattern              | Rate    | Covers                                    |
| ---- | ------------------------- | ------- | ----------------------------------------- |
| base | `/`                       | `15r/s` | Anything the patterns below do not match  |
| 1    | `/apps`                   | `5r/s`  | App endpoints                             |
| 2    | `/apps/text/session/sync` | `8r/s`  | Collaborative editing poll                |
| 3    | `/core/preview`           | `5r/s`  | Thumbnail generation                      |
| 4    | `/apps/mail`              | `40r/s` | Nextcloud Mail                            |

These are starting points — expect to tune them for your client population. Three things about how
BunkerWeb counts, so you tune the right knob:

- The counter is keyed per **(service, client IP, exact request path)**; the query string is not part of
  the key. It is not a per-client budget across the whole site, but every `/core/preview?fileId=…`
  request does share one bucket.
- `LIMIT_REQ_URL=/` is the catch-all for every path no `LIMIT_REQ_URL_*` pattern matches. With
  `USE_LIMIT_REQ=yes` and no explicit `/` entry, BunkerWeb still applies its own default of `2r/s`, which
  throttles ordinary Nextcloud traffic. Always set `/` explicitly.
- Patterns are unanchored PCRE and the first match wins in **undefined order**. `/apps` therefore also
  matches `/index.php/apps/...` and both `/apps/text/session/sync` and `/apps/mail`, so Mail may end up
  on `5r/s` rather than `40r/s`. Anchor a pattern (`^/(index\.php/)?apps/mail`) if you need its rate to
  win deterministically.

Raise the values if legitimate clients trip them. Common triggers: a Files or Photos grid pulling many
thumbnails from `/core/preview` at once, Talk or collaborative editing polling one endpoint, and several
users sharing a NAT or CGNAT egress address. `429` responses in the BunkerWeb logs are the signal.

Do not treat any of this as brute-force protection. It is per-IP, so it is weak against distributed
attempts. Nextcloud's own brute-force protection and `AnonRateLimit` are the controls for authentication
abuse.

`BAD_BEHAVIOR_STATUS_CODES` omits `401` for the same reason: Nextcloud answers `401` on unauthenticated
WebDAV requests, and WebDAV clients that probe before authenticating would accumulate enough of them to
earn a ban during normal operation.

## ModSecurity and the CRS

- `MODSECURITY_CRS_PLUGINS=nextcloud-rule-exclusions` pulls the
  [CRS plugin](https://github.com/coreruleset/nextcloud-rule-exclusions-plugin) that carries the Nextcloud
  exclusions for CRS v4 (BunkerWeb's default `MODSECURITY_CRS_VERSION`).
- The value is unpinned, so BunkerWeb installs the plugin's latest release. Append a tag
  (`nextcloud-rule-exclusions/v1.0.0`) if you need reproducible rollouts.
- CRS plugins require CRS v4. If you drop the service to `MODSECURITY_CRS_VERSION=3`, the plugin setting
  is ignored and `configs/modsec-crs/nextcloud_false_positives.conf` takes over by setting
  `tx.crs_exclusions_nextcloud=1`, which activates the exclusion set shipped inside CRS v3.
- The `UNLOCK` exception rewrites `tx.restricted_headers_basic` for that one request, with the list
  copied from the CRS bundled with BunkerWeb. Re-check the list in
  `configs/modsec-crs/nextcloud_false_positives.conf` after a BunkerWeb upgrade that bumps the CRS.
- `ALLOWED_METHODS` is enforced by BunkerWeb itself (405 on anything not listed), not by the CRS.
  BunkerWeb also generates the CRS `tx.allowed_methods` rule (id `900200`) from that same setting, so
  redefining `900200` in a custom config is redundant.
- Service-scoped `modsec-crs` configs are unsupported when `USE_MODSECURITY_GLOBAL_CRS=yes` (default is
  `no`). On a global-CRS deployment, promote the snippet to a global config scoped with a `Host` rule.

## CalDAV / CardDAV discovery

Calendar and contacts clients expect `/.well-known/caldav` and `/.well-known/carddav` to redirect to
`/remote.php/dav/`. The template does not add those redirects, because the official Nextcloud images
already serve them and BunkerWeb owns `/.well-known/acme-challenge/` for HTTP-01 certificate issuance.

Verify with:

```bash
curl -sI https://nextcloud.example.com/.well-known/caldav | head -n 5
curl -sI https://nextcloud.example.com/.well-known/carddav | head -n 5
```

Both should return `301` with `Location: /remote.php/dav/`. If they `404`, fix it in Nextcloud's own web
server layer (the `.htaccess` or nginx snippet shipped with your image) rather than in BunkerWeb, so the
ACME challenge path stays intact. The DAV verbs those clients need — `PROPFIND`, `REPORT`, `MKCALENDAR`,
`ACL` — are already in `ALLOWED_METHODS`.

## Customization Tips

- Set `REVERSE_PROXY_HOST` to the correct scheme and port for your Nextcloud backend.
- Keep `ALLOWED_METHODS` aligned with the WebDAV verbs required by Nextcloud; remove verbs only if you are
  sure the clients do not need them.
- `SERVE_FILES=no` ensures Nextcloud handles static assets; change it only if you delegate asset delivery
  elsewhere.
- Set `REVERSE_PROXY_WS` to `"yes"` to enable WebSocket support, which is required by some Nextcloud
  applications (for example Talk or Collabora).

## Validation

Before importing:

```bash
jq . template.json                                   # template definition parses
jq -r '.configs[]' template.json | while read -r c; do test -f "configs/$c" || echo "missing: $c"; done
```

After applying the template:

- Check the scheduler logs for `nextcloud-rule-exclusions` being downloaded, and confirm the generated
  `modsecurity-rules.conf` includes the `modsec-crs` directory ahead of the CRS rules.
- Sign in through the proxy, then confirm Nextcloud reports no reverse-proxy warnings under
  **Administration settings → Overview**.
- Exercise WebDAV directly:
  ```bash
  curl -u user:pass -X PROPFIND -H 'Depth: 1' \
    https://nextcloud.example.com/remote.php/dav/files/user/
  ```
- Upload a file larger than one chunk from both the web UI and a desktop sync client, and open a folder
  of images so the preview endpoint is exercised in a burst.
- Confirm `/.well-known/caldav` and `/.well-known/carddav` redirect as shown above.
- Watch for `429` responses and ModSecurity denials in the BunkerWeb logs while doing the above; both
  indicate a value that needs raising for your workload.
