<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/drupal.svg" alt="Drupal logo" width="96" height="96" />
</p>

<h1 align="center">Drupal</h1>

<p align="center">Secure template with CMS-aware defaults</p>

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

Generate a Drupal-focused BunkerWeb configuration that keeps TLS automation, sane
reverse proxy defaults, installer rate limiting, and ModSecurity CRS exclusions
aligned with the CMS so logins, content editing, and cache management remain
reliable.

## At a glance

| Fact        | Value                         |
| ----------- | ----------------------------- |
| Template id | `drupal`                      |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4 |
| Configs     | 1 (modsec-crs)                |
| Rule ids    | `9854000` to `9854999`        |
| Upstream    | `http://mydrupal`             |

## Prerequisites

- A reachable Drupal application (container, VM, or bare metal) that trusts the
  BunkerWeb proxy IP.
- Access to the BunkerWeb UI or environment variables to apply template
  settings.
- Updated Drupal `settings.php` so `$settings['reverse_proxy'] = TRUE;` and the
  proxy address is listed in `$settings['reverse_proxy_addresses']`.

## Install

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#install)
     for the web UI or plugin bundle method.
2. **Assign the template** to the service that fronts Drupal (`USE_TEMPLATE=drupal`
   via environment variables or pick it in the UI).
3. **Review TLS automation** and ensure `SERVER_NAME` and certificate options
   reflect your public hostname(s).
4. **Point the reverse proxy** at the Drupal backend (for example
   `http://drupal:8080`) and confirm BunkerWeb can resolve the target.
5. **Keep the installer rate limits** on `/core/install.php` unless you are
   performing automated installs.
6. **Reload BunkerWeb** and verify you can browse the Drupal admin dashboard
   through the proxy.

## Settings

- Set `REVERSE_PROXY_HOST` to the internal scheme and port of your Drupal
  service.
- Increase `LIMIT_REQ_RATE_*` temporarily if scripted migrations or updates need
  higher throughput. `LIMIT_REQ_RATE` is `50r/s` on `/`: a lower rate answers `429` while the
  installer or an admin page loads its assets.
- `MAX_CLIENT_SIZE=100m` lets large image and file uploads through (the BunkerWeb default of
  `10m` answers `413`). Keep it in line with PHP's `upload_max_filesize` and `post_max_size`.
- `ALLOWED_METHODS` includes `PATCH`, `DELETE`, and `OPTIONS`, which JSON:API clients and CORS
  preflights need. If you do not expose JSON:API, enter `GET|POST|HEAD|QUERY`, BunkerWeb's default.
- `modsec-crs/drupal_false_positives.conf` allows the `application/vnd.api+json` content type
  and arms the JSON body parser for it, so request bodies stay inspected. It also lets
  `application/octet-stream` through for `POST /jsonapi/<entity>/<bundle>[/<uuid>]/field_*`,
  the JSON:API file upload route, and nowhere else.
- Leave `SERVE_FILES=no` so Drupal handles asset delivery; enable it only if you
  move static files to an object store behind BunkerWeb.
- `MODSECURITY_CRS_PLUGINS=drupal-rule-exclusions` keeps Drupal management
  actions functional—change it only if you tune CRS rules manually.

## Files

- `template.json` – Template definition with guided steps for TLS, upstream, and
  hardening options.
- `configs/modsec-crs/drupal_false_positives.conf` – ModSecurity CRS tuning for the JSON:API
  content type and file uploads.

## Validation

- Run `jq . template.json` to confirm the template definition is valid JSON.
- Check the Drupal status report and attempt cache clears or content edits to
  ensure rate limits and CRS exclusions do not block normal operations.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
