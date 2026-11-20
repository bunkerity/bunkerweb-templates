<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/xen-orchestra.svg" alt="Xen Orchestra logo" width="96" height="96" />
</p>

<h1 align="center">Xen Orchestra</h1>

<p align="center">Reverse proxy template with WebSocket and rate limiting defaults</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Xen_Orchestra-5.213.0-248651?labelColor=1d6f93&style=flat" alt="Xen Orchestra xo-server 5.213.0" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#crs-exception-scope-and-cost">CRS exceptions</a> ·
  <a href="#files">Files</a> ·
  <a href="#validation">Validation</a>
</p>

Provision a BunkerWeb configuration tailored for Xen Orchestra so HTTPS
automation, reverse proxy, WebSocket upgrades, and request rate limiting work
out of the box. Bad behavior detection is disabled to avoid false positives
caused by XO's frequent API polling.

## At a glance

| Fact        | Value                                                |
| ----------- | ---------------------------------------------------- |
| Template id | `xen-orchestra`                                      |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Xen Orchestra 5.213.0 |
| Configs     | 1 (modsec-crs)                                       |
| Rule ids    | `9879000` to `9879999`                               |
| Upstream    | `https://192.168.1.1`                                |

## Prerequisites

- A Xen Orchestra backend reachable from the BunkerWeb instance (typically
  HTTPS with a self-signed certificate). The template was tested with the
  `ronivay/xen-orchestra` image (Xen Orchestra from sources, built 2026-11-14:
  xo-server 5.213.0, xo-web 5.208.0, XO 6 web 0.65.1) started with `HTTP_PORT=80` and
  `HTTPS_PORT=443`. Both interfaces work through the template: XO 6 at `/` and XO 5 at `/v5`.
- Access to the BunkerWeb web UI to create the service from the template.
- Confirm XO listens on HTTPS and that `REVERSE_PROXY_SSL_SNI` is set to `no`
  when using a self-signed upstream certificate.

## Install

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#install)
     for the web UI or plugin bundle method.
2. **Create the service from the template**: **Services**, **Create new service**, then pick
   `Xen-orchestra` in the **Templates** dropdown above the steps (or set
   `USE_TEMPLATE=xen-orchestra`).
3. **Set the name and the certificate** in step 1. Replace `SERVER_NAME` with the name you
   expose. Let's Encrypt works only for a public name that reaches BunkerWeb on port 80. For a
   lab or an internal name, turn **Automatic Let's Encrypt** off and tick **Activate
   self-signed certificate** (the browser then shows a certificate warning). BunkerWeb 1.6.17
   can drop a switch you turned off while creating the service: open the service again and
   check it, or set `AUTO_LETS_ENCRYPT=no` in Raw mode. A custom certificate is not in the
   guided steps: use the **Advanced** tab, **Custom SSL certificate**.
4. **Update upstream target**: point `REVERSE_PROXY_HOST` to your Xen Orchestra
   instance (e.g. `https://192.168.1.1`) and confirm connectivity from
   BunkerWeb.
5. **Review rate limits**: `LIMIT_REQ_RATE` applies to each URL separately, per client address;
   raise it if automation scripts call the same URL more often (see Settings).
6. **Save the service**: BunkerWeb applies it when you save, there is nothing to reload. Open the
   address, sign in and check the XO 6 dashboard and the tree view. The VM console, live
   migration and snapshots were not tested (see Validation).

## Settings

- Raise or lower `MAX_CLIENT_SIZE` depending on whether you transfer ISOs or
  large files through the proxy.
- `LIMIT_REQ_RATE` is `10r/s`, counted per URL and per client address. XO 5 talks JSON-RPC over
  a WebSocket on `/api/`, so only the upgrade request counts. XO 6 uses REST on `/rest/v0/`:
  every page load posts nine event subscriptions to the same URL within one second, and a
  subscription refused with `429` leaves that part of the page without live updates. With
  `8r/s` four of 69 subscriptions got `429`; `9r/s` and `10r/s` got none. `10r/s` is the
  lowest round value with no `429`, and it leaves little room: if the BunkerWeb log shows
  `429` on `/rest/v0/events/.../subscriptions`, raise it (`15r/s` for a busy pool).
- `USE_BAD_BEHAVIOR` is set to `no` because XO's polling pattern triggers false
  positives; re-enable only after confirming it does not block legitimate
  traffic.
- `KEEP_UPSTREAM_HEADERS` is `*`, so XO's own `Content-Security-Policy` and `X-Frame-Options`
  (`SAMEORIGIN`) reach the browser, and the `CONTENT_SECURITY_POLICY` field of the service has
  no effect. `X-Frame-Options` comes from XO itself, not from BunkerWeb. To let another site
  show XO in a frame: type `Vary` in `KEEP_UPSTREAM_HEADERS` (a field cleared in easy mode
  keeps the template value), type `frame-ancestors https://portal.example.org` in
  `CONTENT_SECURITY_POLICY`, and in the **Advanced** tab, **Headers**, set `X-Frame-Options`
  to the empty choice. BunkerWeb's policy then replaces XO's own, which lists the scripts XO
  may load. `PERMISSIONS_POLICY` works as it is.
- Set `REVERSE_PROXY_SSL_SNI` to `yes` if your upstream uses a valid
  certificate matching the hostname.
- To change or remove an exception, open **Configs** in the web UI, then the
  `xen-orchestra_false_positives` config of your service (type `MODSEC_CRS`), and edit or delete
  its rules.

## CRS exception scope and cost

`configs/modsec-crs/xen-orchestra_false_positives.conf` holds two rules. Each one names the
method and the path, and a request that does not match both gets no exception.

| Rule      | Path and method                                 | What it removes                                                                                        |
| --------- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `9879100` | `POST /api/<token>` (one segment after `/api/`) | CRS `920420` (content type). XO sends config, VM and disk imports as `application/octet-stream`.       |
| `9879101` | `GET` under `/rest/v0/`                         | CRS `932235` (Unix command) for the `fields` parameter only. XO 6 lists the properties it wants there. |

- `9879100` cost: any content type is accepted on that path, and a raw upload body is not
  inspected. `POST /api/<a>/<b>` and `POST /api/` stay blocked.
- `9879101` cost: the `fields` parameter of a `GET` under `/rest/v0/` is no longer checked for
  Unix commands. The host list asks for `startTime`, `version` and `bios_strings`, which the
  rule reads as a command, so without the exception the dashboard, the tree view and the host
  pages stay empty (`403` on every load). Rule `932235` still checks every other parameter on
  those paths, and `fields` on every other path or method. Every other CRS rule still checks
  `fields`: a path such as `/etc/passwd` in it still gets `403`.

## Files

- `template.json` – Template definition with steps for TLS, reverse proxy,
  rate limiting, and security tuning.
- `configs/modsec-crs/xen-orchestra_false_positives.conf` – The two CRS exceptions above: rule
  `9879100` (`920420` for `POST /api/<token>`) and rule `9879101` (`932235` for `fields` on
  `GET /rest/v0/`).
  It is a `modsec-crs` config on purpose: those load before the CRS rules, which is the only point
  where the removal still applies. The import dialog may write the folder as `modsec_crs`; upload
  the file from `configs/modsec-crs/`.

## Validation

Run `jq . template.json` to check the JSON before importing.

On 2026-11-20 (BunkerWeb 1.6.17 all-in-one, CRS 4.29.0, the `ronivay/xen-orchestra` image above
with no pool attached) the three files were imported through the web UI behind the CRS (the JSON
alone, the config alone and everything together, as Save sends it: `200`, no rule fired), and
the service was created in easy mode from the template. In headless Chrome on that service:

- five XO 6 page loads, each followed by three page changes (dashboard, hosts, VMs, pools),
  gave `63` subscriptions answered `201`, no `429` and no ModSecurity denial. The host list,
  the tree view and every other `/rest/v0/<type>?fields=...` list the pages request (alarms,
  backup jobs and logs, networks, pbds, pifs, plugins, pools, schedules, servers, SRs, tasks,
  users, VBDs, VDIs, VMs, VM controllers) answered `200`. With the old `6r/s` the same pages got
  `429` on some of the nine subscriptions at every load, and `GET /rest/v0/hosts` got `403`
  (rule `932235` on `fields`) at every load;
- thirteen XO 5 pages under `/v5`, the JSON-RPC calls over the `/api/` WebSocket (sign in,
  `server.add`, `server.getAll`, `server.remove`, `xo.exportConfig`) and the config export download
  answered with no denial, and `POST /api/<token>` with `application/octet-stream` reached XO.

A curl matrix confirmed the limits of each rule. `403` stayed in every row that is outside an
exception: the same text in another parameter of `/rest/v0/hosts` (`932235`), in `fields` on
`/rest/v1/`, `/api/` and `/rest/v0x/`, in a `POST` or `HEAD` to `/rest/v0/hosts`, a `/../` path,
a path such as `/etc/passwd` or a script tag in `fields` itself (`930120`, `932160`, `941100`),
and `POST /api/<a>/<b>` or `POST /api/` with `application/octet-stream` (`920420`).

**Not tested** (no XCP-ng or XenServer host was available): the VM console WebSocket, live
migration and snapshot actions, and every XO page that needs a connected pool. Test them on your
pool before relying on the proxy; the list pages above only show what XO asks for with no pool.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
