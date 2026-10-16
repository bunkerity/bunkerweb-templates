# WordPress Template

## Overview

Provision a curated BunkerWeb configuration for WordPress. This template ships sensible defaults for TLS, reverse proxying, request throttling, crawler whitelisting, and CRS exclusions tuned for WordPress traffic so you can get a secure site online quickly.

## REST API methods

The default `ALLOWED_METHODS` value is `GET|POST|HEAD|OPTIONS|PUT|DELETE`, so the WordPress REST
API works for clients that edit and delete with `PUT` and `DELETE` (headless integrations,
mobile apps, block-editor plugins). If your site does not need them, narrow the list:

```text
"ALLOWED_METHODS": "GET|POST|HEAD|OPTIONS"
```

## Prerequisites

- A running WordPress upstream accessible on your network (for example a container named `mywp`).
- The BunkerWeb UI or the ability to edit multisite settings directly.
- Domain name(s) that will serve the WordPress instance.

## Files

- `template.json` – BunkerWeb template definition containing default settings, configs, and guided steps.
- `configs/modsec/wordpress_false_positives.conf` – ModSecurity CRS tuning for WordPress admin, cron, and XML-RPC traffic.
- `configs/modsec-crs/wordpress_crs_tuning.conf` – CRS settings that must load before the CRS, plus the exclusions for the classic uploader and raw REST media uploads.

## Setup

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#installing-templates) for the web UI or
     plugin bundle method.
2. **Assign the template** to your WordPress service via the easy-mode UI or by setting `USE_TEMPLATE=wordpress`.
3. **Customize the settings** highlighted in the template steps (domains, upstream host, TLS options).
4. **Reload the service** and verify WordPress loads through BunkerWeb.

## Customization Tips

- Update `REVERSE_PROXY_HOST` to the URL of your WordPress upstream (e.g. `http://wordpress:80`).
- Adjust `MAX_CLIENT_SIZE` if you need to support larger media uploads.
- Tune `LIMIT_REQ_RATE` or change the protected `LIMIT_REQ_URL` (defaults to `/`) if you want to rate-limit only `wp-login.php` or `xmlrpc.php`. `LIMIT_REQ_RATE` is `50r/s` on `/`: a single page load fetches dozens of assets, and a lower rate answers `429` and counts toward a bad-behavior ban. `wp-login.php` has its own `2r/s` limit (`LIMIT_REQ_URL_1`).
- The classic media uploader (`/wp-admin/async-upload.php`) is blocked by CRS rule `932180` for any file whose name contains `class`. The template removes that rule for the `async-upload` field of that one `POST` path. The same field then also skips the restricted-name check for names such as `.htaccess`, so keep WordPress's own upload type filter (and `uploads/` execution rules) in place.
- Apps that upload raw images (`Content-Type: image/png` or `image/jpeg`) to `POST /wp-json/wp/v2/media` are blocked by CRS rule `920420`. The template removes that rule for that method and path only.
- Add or prune domains in `WHITELIST_RDNS` if you want to control which crawlers bypass security checks.
- Review and narrow the `admin-ajax` exclusions in `configs/modsec/wordpress_false_positives.conf` for your installation.

## Validation

Run `jq . template.json` to confirm the JSON definition is valid before importing via the UI.
