<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/nextcloud.svg" alt="Nextcloud logo" width="96" height="96" />
</p>

<h1 align="center">Nextcloud</h1>

<p align="center">Secure template with WebDAV-aware defaults</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Nextcloud-35.0.3-248651?labelColor=1d6f93&style=flat" alt="Nextcloud 35.0.3" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#reverse-proxy-settings">Reverse proxy settings</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#files">Files</a> ·
  <a href="#validation">Validation</a>
</p>

Generate a hardened BunkerWeb configuration for Nextcloud. The template keeps TLS automation, WebDAV
verbs, uploads, caching, and ModSecurity CRS exclusions aligned with common Nextcloud guidance so your
instance stays reachable and protected.

## At a glance

| Fact        | Value                                           |
| ----------- | ----------------------------------------------- |
| Template id | `nextcloud`                                     |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Nextcloud 35.0.3 |
| Configs     | 1 (modsec-crs)                                  |
| Rule ids    | `9863000` to `9863999`                          |
| Upstream    | `http://nextcloud`                              |

## Prerequisites

- A reachable Nextcloud backend (container, VM, or service) that accepts traffic from BunkerWeb. Tested
  with the official `nextcloud:apache` image, Nextcloud 35.0.3, which listens on port 80.
- Nextcloud already set up (admin account and database): a fresh official image shows the web installer.
- Access to the BunkerWeb UI or the ability to apply settings via environment variables.
- Nextcloud configured for life behind a reverse proxy (see [Reverse proxy settings](#reverse-proxy-settings)).
- Give the BunkerWeb container at least 2 GiB of memory (see [Upload sizing](#upload-sizing)).

## Reverse proxy settings

BunkerWeb forwards `X-Forwarded-For`, `X-Real-IP`, and `X-Forwarded-Proto` to the upstream. Nextcloud
ignores all three until you tell it which proxy to trust, so set the following in `config.php` (or the
matching environment variables on the official image):

| `config.php` key    | Official image env var      | Why                                                               |
| ------------------- | --------------------------- | ----------------------------------------------------------------- |
| `trusted_domains`   | `NEXTCLOUD_TRUSTED_DOMAINS` | Hostnames Nextcloud will answer for.                              |
| `trusted_proxies`   | `TRUSTED_PROXIES`           | BunkerWeb's address, so `X-Forwarded-For` is honoured.            |
| `overwriteprotocol` | `OVERWRITEPROTOCOL`         | Set to `https`; BunkerWeb terminates TLS, the upstream sees HTTP. |
| `overwritehost`     | `OVERWRITEHOST`             | Public hostname used in generated URLs.                           |
| `overwrite.cli.url` | `OVERWRITECLIURL`           | Base URL for `occ`, cron, and notification links.                 |

> [!WARNING]
> Keep `trusted_proxies` as narrow as possible — the BunkerWeb container IP or its `/32`, not `0.0.0.0/0`.
> Nextcloud treats a trusted proxy's `X-Forwarded-For` as authoritative, so an over-broad value lets any
> client spoof its source IP and defeat both Nextcloud's brute-force protection and BunkerWeb's bans.

On the BunkerWeb side, leave `USE_REAL_IP` at its default `no`. Only enable it (with a matching
`REAL_IP_FROM`) when BunkerWeb is itself behind another proxy, load balancer, or CDN — otherwise you are
trusting client-supplied `X-Forwarded-For` headers.

## Install

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#install) for the web UI or
     plugin bundle method.
2. **Assign the template** to the service that fronts Nextcloud (set `USE_TEMPLATE=nextcloud` or pick it
   in the **Templates** dropdown of the easy-mode UI).
3. **Adjust the TLS and domain entries** so `SERVER_NAME` and certificate options match your deployment.
   `AUTO_LETS_ENCRYPT` is on: for a name that is not public, turn it off. BunkerWeb 1.6.17 keeps the
   template's value when you create the service, so save once, open the service again, turn it off and
   save again. A custom certificate is not in the guided steps: use the **Advanced** tab.
4. **Point the reverse proxy** at your upstream host (the template uses `http://nextcloud`, the official
   image on port 80; use for example `http://nextcloud:8080` if your container publishes another port)
   and confirm the service resolves from the BunkerWeb container or pod.
5. **Configure Nextcloud itself** for the proxy – see [Reverse proxy settings](#reverse-proxy-settings).
   Skipping this yields `http://` links, wrong client IPs in the logs, and broken federation.
6. **Review the HTTP hardening options** – keep the WebDAV verbs, upload size, rate limit, caching, and
   CRS plugin values or tailor them to your workload.
7. **Save the service**: BunkerWeb applies it once the scheduler has reloaded. Then run the checks in
   [Validation](#validation).

## Settings

- Set `REVERSE_PROXY_HOST` to the correct scheme and port for your Nextcloud backend.
- Keep `ALLOWED_METHODS` aligned with the WebDAV verbs required by Nextcloud; remove verbs only if you are
  sure the clients do not need them. The easy-mode step shows each verb in its own box.
- `LIMIT_CONN_MAX_HTTP2` (`200`) and the `LIMIT_REQ_*` pairs are in the same step, see
  [Rate limiting](#rate-limiting).
- `SERVE_FILES=no` ensures Nextcloud handles static assets; change it only if you delegate asset delivery
  elsewhere.
- Set `REVERSE_PROXY_WS` to `"yes"` to enable WebSocket support, which is required by some Nextcloud
  applications (for example Talk or Collabora).

### Upload sizing

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
The practical ceiling is therefore a few times `MAX_CLIENT_SIZE` per concurrent upload of resident
memory, and values in the multi-gigabyte range are an out-of-memory risk rather than a capability.
Prefer leaving Nextcloud's chunked uploads enabled over raising this value.

The template's upload rules (see [ModSecurity and the CRS](#modsecurity-and-the-crs)) keep the content of a
file out of the CRS checks, but ModSecurity still reads the whole request body into memory. Measured on
BunkerWeb 1.6.17, a 100 MiB `PUT` of `application/octet-stream` raised the BunkerWeb container by 310,
596 and 581 MiB in three runs, so plan for three to six times the size of each concurrent upload. Give
the BunkerWeb container at least 2 GiB. When a worker runs out of memory,
BunkerWeb logs `worker process ... exited on signal 9` and the web UI upload ends with no file and no
error message, so look for that line first when an upload vanishes.

To cap the memory, set `MODSECURITY_SEC_REQUEST_BODY_LIMIT` (it defaults to `MAX_CLIENT_SIZE`) together
with `MODSECURITY_SEC_REQUEST_BODY_LIMIT_ACTION` set to `ProcessPartial`. With `20m` and `ProcessPartial`,
the same 100 MiB `PUT` raised the container by 70, 81 and 24 MiB. The cost: ModSecurity reads
only the first 20 MiB of any larger request body and the rest is not checked. Leave the action at `Reject`
and the limit is a size limit: a request body above it answers `413`. Both settings apply to every request
of the service.

### Rate limiting

| Slot | Path pattern                                           | Rate    | Covers                                   |
| ---- | ------------------------------------------------------ | ------- | ---------------------------------------- |
| base | `/`                                                    | `15r/s` | Anything the patterns below do not match |
| 1    | `/apps/` and `/index.php/apps/`, except Mail and below | `5r/s`  | App endpoints                            |
| 2    | `/apps/text/session/sync`                              | `8r/s`  | Collaborative editing poll               |
| 3    | `/core/preview`                                        | `20r/s` | Thumbnail generation                     |
| 4    | `/apps/mail`                                           | `40r/s` | Nextcloud Mail                           |

These are starting points — expect to tune them for your client population. Three things about how
BunkerWeb counts, so you tune the right knob:

- The counter is keyed per **(service, client IP, exact request path)**; the query string is not part of
  the key. It is not a per-client budget across the whole site, but every `/core/preview?fileId=…`
  request does share one bucket.
- `LIMIT_REQ_URL=/` is the catch-all for every path no `LIMIT_REQ_URL_*` pattern matches. With
  `USE_LIMIT_REQ=yes` and no explicit `/` entry, BunkerWeb still applies its own default of `2r/s`, which
  throttles ordinary Nextcloud traffic. Always set `/` explicitly.
- Patterns are PCRE and the first match wins in **undefined order**, so a path must match only one
  pattern. The template anchors its patterns (for example `^/(index\.php/)?apps/mail`) and slot 1
  leaves out `/apps/mail` and `/apps/text/session/sync`: each of those two keeps its own rate. An
  unanchored `/apps` would also match them, and Mail would end up on `5r/s` instead of `40r/s`. Keep
  your own patterns disjoint the same way. The OCS API (`/ocs/v2.php/apps/...`) is not in slot 1: it
  uses the base rate.

The preview rate is `20r/s` because opening a folder of images asks for one `/core/preview` per
thumbnail within a second, and they all share one counter. At `5r/s` most of the thumbnails of a folder
came back `429`; at `10r/s` a folder of 60 images on a tall screen still lost 14 to 24 of them.

Connections are limited too, separately from the request rate. `LIMIT_CONN_MAX_HTTP2` (BunkerWeb's
default is `100` open HTTP/2 requests per client IP) is set to `200` in the same step. The Files app
asks for about 300 files on a cold start, and with the default 33 to 42 of them, mostly the JavaScript
chunks of the viewer, comments and activity, were answered `429`; `150` was the lowest round value
without any. Without TLS the browser speaks HTTP/1.1 and `LIMIT_CONN_MAX_HTTP1` (default `10`)
applies instead.

Raise the values if legitimate clients trip them. Common triggers: a Files or Photos grid pulling many
thumbnails from `/core/preview` at once, Talk or collaborative editing polling one endpoint, and several
users sharing a NAT or CGNAT egress address. `429` responses in the BunkerWeb logs are the signal.

> [!IMPORTANT]
> Do not treat any of this as brute-force protection. It is per-IP, so it is weak against distributed
> attempts. Nextcloud's own brute-force protection and `AnonRateLimit` are the controls for authentication
> abuse.

`BAD_BEHAVIOR_STATUS_CODES` omits `401` for the same reason: Nextcloud answers `401` on unauthenticated
WebDAV requests, and WebDAV clients that probe before authenticating would accumulate enough of them to
earn a ban during normal operation.

### ModSecurity and the CRS

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
- The content of uploaded files is not scanned. A file sent with `PUT` to `/remote.php/dav/files/`,
  `/remote.php/dav/uploads/`, `/remote.php/webdav/` or a public share (`/public.php/dav/files/`) is
  stored, not run, but ModSecurity reads a `.json` or `.xml` file (the browser sends it as
  `application/json` or `application/xml`) as if it were a request, and a file that only mentions a
  shell path, a snippet of HTML or a SQL query got a `403` (CRS `932160`, `941160`, `930120` and more).
  Rules `9863110` and `9863111` keep the content of those `PUT` requests out of the CRS attack checks.
  The query string, the cookies, the headers and the path of those requests are still checked, so an
  attack sent there still answers `403`. The second rule only applies to a request with no query string:
  the web UI and the WebDAV clients tested send none, and a `PUT` that has one is checked in full, file
  content included. The cost is that the content of uploaded files is not checked by the WAF: use an
  anti-virus on the Nextcloud side if you need it. The OCS API, forms, other methods and every other path
  stay inspected in full. The Content-Type and file name checks for these uploads (CRS `920340`,
  `920420`, `920440`, `920640`) come from the `nextcloud-rule-exclusions` plugin, so keep
  `MODSECURITY_CRS_PLUGINS` set. If BunkerWeb cannot download the plugin (the scheduler log shows a
  `ConnectionError` for `download-crs-plugins`), every upload answers `403` with those ids.
- `ALLOWED_METHODS` is enforced by BunkerWeb itself (405 on anything not listed), not by the CRS.
  BunkerWeb also generates the CRS `tx.allowed_methods` rule (id `900200`) from that same setting, so
  redefining `900200` in a custom config is redundant.
- Service-scoped `modsec-crs` configs are unsupported when `USE_MODSECURITY_GLOBAL_CRS=yes` (default is
  `no`). On a global-CRS deployment, promote the snippet to a global config scoped with a `Host` rule.

### CalDAV / CardDAV discovery

Calendar and contacts clients expect `/.well-known/caldav` and `/.well-known/carddav` to redirect to
`/remote.php/dav/`. The template does not add those redirects, because the official Nextcloud images
already serve them and BunkerWeb owns `/.well-known/acme-challenge/` for HTTP-01 certificate issuance.

Verify with:

```bash
curl -sI https://nextcloud.example.com/.well-known/caldav | head -n 5
curl -sI https://nextcloud.example.com/.well-known/carddav | head -n 5
```

Both should return `301` with a `Location` that ends in `/remote.php/dav/`. The official image builds that
URL itself, as an absolute one from the host name BunkerWeb passes on: it reads `http://` and has no port,
whatever `overwriteprotocol` says (Apache answers before Nextcloud's PHP runs, so `overwriteprotocol` and
`overwritehost` do not reach it). Calendar and contacts clients follow it to that plain-HTTP address,
so publish ports 80 and 443 and keep BunkerWeb's HTTP to HTTPS redirect on: it is automatic with
`AUTO_LETS_ENCRYPT` or a custom certificate, and `REDIRECT_HTTP_TO_HTTPS=yes` turns it on otherwise
(then `http://<host>/remote.php/dav/` answers `301` to the `https` address). If they `404`, fix it in
Nextcloud's own web server layer (the `.htaccess` or nginx snippet shipped with your image) rather than in BunkerWeb, so the
ACME challenge path stays intact. The DAV verbs those clients need — `PROPFIND`, `REPORT`, `MKCALENDAR`,
`ACL` — are already in `ALLOWED_METHODS`.

## Files

- `template.json` – Template definition with default settings and the guided step layout.
- `configs/modsec-crs/nextcloud_false_positives.conf` – Enables the CRS v3 Nextcloud exclusion set.
  It is a `modsec-crs` config on purpose: those load **before** the CRS rules, which is the only point
  where `tx.crs_exclusions_nextcloud` is still read. The same file under `modsec/` would load after the
  CRS and do nothing.
  It also lets WebDAV `UNLOCK` under `/remote.php/dav/files/` carry a `Lock-Token` header: the plugin
  covers `If` but not `Lock-Token`, so CRS rule `920450` would otherwise answer `403`.
  The third and fourth rules (`9863110`, `9863111`) keep the content of file uploads by `PUT` under the
  WebDAV file and upload paths out of the CRS attack checks, so JSON and XML files are not blocked. The
  query string, cookies and headers of those requests are still checked.
  To change it later in the web UI, open **Configs**, pick the service and the `MODSEC_CRS` type.

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
- Upload a file larger than one chunk from both the web UI and a desktop sync client, and upload a
  `.json` and an `.xml` file that mention a shell path or an HTML tag: they must arrive, not `403`.
- Check that the upload exceptions stay narrow: the same `PUT` with an attack in the query string must
  still answer `403`:
  ```bash
  curl -u user:pass -X PUT --data hello \
    'https://nextcloud.example.com/remote.php/dav/files/user/x.txt?q=<script>alert(1)</script>'
  ```
- Open a folder of 24 or more images: every thumbnail loads and the BunkerWeb log shows no `429` for
  `/core/preview`. Reload the Files app with the browser cache off: no `limiting connections` lines.
- Confirm `/.well-known/caldav` and `/.well-known/carddav` redirect as shown above.
- Watch for `429` responses and ModSecurity denials in the BunkerWeb logs while doing the above; both
  indicate a value that needs raising for your workload.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
