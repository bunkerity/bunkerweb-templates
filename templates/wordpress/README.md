<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/wordpress.svg" alt="WordPress logo" width="96" height="96" />
</p>

<h1 align="center">WordPress</h1>

<p align="center">Secure template with essential hardening defaults</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/WordPress-6.9.4-248651?labelColor=1d6f93&style=flat" alt="WordPress 6.9.4" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#files">Files</a> ·
  <a href="#validation">Validation</a>
</p>

Provision a curated BunkerWeb configuration for WordPress. This template ships sensible defaults for TLS, reverse proxying, request throttling, crawler whitelisting, and CRS exclusions tuned for WordPress traffic so you can get a secure site online quickly.

## At a glance

| Fact        | Value                          |
| ----------- | ------------------------------ |
| Template id | `wordpress`                    |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, WordPress 6.9.4 |
| Configs     | 2 (modsec-crs)                 |
| Rule ids    | `9878000` to `9878999`         |
| Upstream    | `http://mywp`                  |

## Prerequisites

- A running WordPress upstream accessible on your network (for example a container named `mywp`).
  Tested with the official `wordpress:6-apache` image, WordPress 6.9.4.
- The BunkerWeb web UI (easy mode) or the ability to edit multisite settings directly.
- Domain name(s) that will serve the WordPress instance.
- WordPress must know its public address: set the WordPress Address and Site Address (`home` and
  `siteurl`) to the public name with its scheme, for example `https://www.example.com`.
  BunkerWeb passes the host name on to WordPress, which builds its links and redirects from these two
  values.

## Install

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#install) for the web UI or
     plugin bundle method.
2. **Assign the template** to your WordPress service: on the new service page, pick it in the
   **Templates** dropdown above the steps (the default is `Low`).
3. **Customize the settings** highlighted in the template steps (domains, upstream host, TLS options).
   Replace the example `www.example.com` in step 1 with your own hostname(s). Turn Let's Encrypt off if
   the name is not public. If it is still on after you save, edit the service and turn it off again.
4. **Save** the service, wait a few seconds while BunkerWeb applies it, and check that WordPress loads
   through BunkerWeb.

## Settings

- Update `REVERSE_PROXY_HOST` to the URL of your WordPress upstream (e.g. `http://wordpress:80`).
- `MAX_CLIENT_SIZE` (`50m`) caps the request size at BunkerWeb. It is not the only upload limit: the
  official WordPress image runs PHP with `upload_max_filesize` of `2M` and `post_max_size` of `8M`, and
  PHP refuses a bigger file first ("exceeds the upload_max_filesize directive"), so raising
  `MAX_CLIENT_SIZE` alone changes nothing. To allow larger media, add a PHP file to the WordPress
  container, for example `./uploads.ini` mounted at `/usr/local/etc/php/conf.d/uploads.ini` with
  `upload_max_filesize = 64M` and `post_max_size = 64M`, restart the container, and keep
  `MAX_CLIENT_SIZE` at least as large.
- `INTERCEPTED_ERROR_CODES` (`403 405 413 429 501 502 503 504`) lists the answers BunkerWeb replaces
  with its own error page. It leaves out `400`, `401` and `404`, so the REST API (`/wp-json/`) keeps
  WordPress's JSON errors (`code` and `message`, which the block editor and REST clients read) and
  your theme keeps its own 404 page. `403` stays in the list because BunkerWeb's own denials, such as a
  CRS block, need it to show BunkerWeb's page. The cost is that a `403` from WordPress itself (for
  example an expired REST nonce) is replaced by that page as well. A WordPress or upstream `502` or
  `503` also gets BunkerWeb's page.
- Tune `LIMIT_REQ_RATE` or change the protected `LIMIT_REQ_URL` (defaults to `/`) if you want to rate-limit only `wp-login.php` or `xmlrpc.php`. `LIMIT_REQ_RATE` is `50r/s` on `/`: a single page load fetches dozens of assets, and a lower rate answers `429` and counts toward a bad-behavior ban. `wp-login.php` has its own `2r/s` limit (`LIMIT_REQ_URL_1`).
- The classic media uploader (`/wp-admin/async-upload.php`) is blocked by CRS rule `932180` for any file whose name contains `class`. The template removes that rule for the `async-upload` field of that one `POST` path. The same field then also skips the restricted-name check for names such as `.htaccess`, so keep WordPress's own upload type filter (and `uploads/` execution rules) in place.
- Apps that upload raw images (`Content-Type: image/png` or `image/jpeg`) to `POST /wp-json/wp/v2/media` are blocked by CRS rule `920420`. The template removes that rule for that method and path only. With plain permalinks (Settings, Permalinks, "Plain"), WordPress serves the same route as `/?rest_route=/wp/v2/media`, and a second rule covers it: `POST` on `/` with exactly one `rest_route` equal to `/wp/v2/media`. Every other route and path keeps the check.
- Add or prune domains in `WHITELIST_RDNS` if you want to control which crawlers bypass security checks. A
  client matched by `WHITELIST_IP`, `WHITELIST_RDNS` or `WHITELIST_ASN` skips every check of the service,
  so list only addresses and domains you trust.
- `MODSECURITY_CRS_PLUGINS` keeps the `wordpress-rule-exclusions` plugin of the OWASP CRS enabled. It
  removes CRS hits on the WordPress login form, the block editor and its REST calls, live preview, and
  the admin screens for installation, users, content, settings, site health and plugins. It does not
  exclude the comment field, so comment text stays fully checked.
- Review and narrow the `admin-ajax` exclusions for your installation: in the web UI open **Configs**,
  then the `wordpress_false_positives` config of your service (type `MODSEC_CRS`), edit it and save.

### REST API methods

The default `ALLOWED_METHODS` value is `GET|POST|HEAD|OPTIONS|PUT|PATCH|DELETE`, so the WordPress REST
API works for clients that edit and delete with `PUT`, `PATCH` and `DELETE` (headless integrations,
mobile apps, block-editor plugins). If your site does not need them, narrow the list:

```text
"ALLOWED_METHODS": "GET|POST|HEAD|OPTIONS"
```

## Files

- `template.json` – BunkerWeb template definition containing default settings, configs, and guided steps.
- `configs/modsec-crs/wordpress_false_positives.conf` – ModSecurity CRS tuning for WordPress admin, cron, and XML-RPC traffic.
- `configs/modsec-crs/wordpress_crs_tuning.conf` – CRS settings that must load before the CRS, plus the exclusions for the classic uploader and raw REST media uploads (pretty and plain permalinks).

## Validation

Run `jq . template.json` to confirm the JSON definition is valid before importing via the UI.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
