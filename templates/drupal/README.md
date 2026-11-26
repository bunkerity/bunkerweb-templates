<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/drupal.svg" alt="Drupal logo" width="96" height="96" />
</p>

<h1 align="center">Drupal</h1>

<p align="center">Secure template with CMS-aware defaults</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Drupal-11.4.13-248651?labelColor=1d6f93&style=flat" alt="Drupal 11.4.13" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#configure-drupal-for-the-proxy">Configure Drupal</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#files">Files</a> ·
  <a href="#validation">Validation</a>
</p>

Generate a Drupal-focused BunkerWeb configuration that keeps TLS automation, sane
reverse proxy defaults, installer rate limiting, and ModSecurity CRS exclusions
aligned with the CMS so logins, content editing, cache management and the JSON:API module
remain reliable.

## At a glance

| Fact        | Value                                        |
| ----------- | -------------------------------------------- |
| Template id | `drupal`                                     |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Drupal 11.4.13 |
| Configs     | 1 (modsec-crs)                               |
| Rule ids    | `9854000` to `9854999`                       |
| Upstream    | `http://drupal:80` (a placeholder, see Install step 4) |

## Prerequisites

- A reachable Drupal application (container, VM, or bare metal) that trusts the
  BunkerWeb proxy IP. Tested with Drupal 11.4.13 (the official `drupal:11-apache` image, SQLite).
  That image serves on port `80`.
- Access to the BunkerWeb UI or environment variables to apply template
  settings.
- Drupal's `settings.php` lines from the next section: `reverse_proxy`, the BunkerWeb address in
  `reverse_proxy_addresses`, and `trusted_host_patterns`.
- A host name that points at BunkerWeb. Let's Encrypt (on by default) needs it to be public. For
  an internal name, use a certificate of your own (Install step 3).

## Configure Drupal for the proxy

Drupal reads the client address, the scheme and the host name from the headers BunkerWeb sends, but
only from a proxy it trusts.

1. **Find BunkerWeb's address** on the network it shares with Drupal. Replace `<network>` with that
   Docker network (`docker network ls` lists them) and `<bunkerweb>` with the BunkerWeb container:

   ```sh
   docker inspect -f '{{(index .NetworkSettings.Networks "<network>").IPAddress}}' <bunkerweb>
   ```

   It prints one address such as `172.29.27.10`, without a `/24` suffix. Recreating the container can
   give it a new address, and Drupal then stops trusting BunkerWeb. Pin the address in Compose
   (`ipv4_address` under the service's network) to keep it.

2. **Add these lines** to Drupal's `sites/default/settings.php`, with your address and host name
   (a dot in the host name is written `\.`):

   ```php
   $settings['reverse_proxy'] = TRUE;
   $settings['reverse_proxy_addresses'] = ['172.29.27.10'];
   $settings['trusted_host_patterns'] = ['^drupal\.example\.com$'];
   ```

   List the address of BunkerWeb only, not its whole subnet: every container on the subnet could
   then forge the client address.

3. **Do it before the installer runs on a new site.** Copy `default.settings.php` to
   `settings.php` first, make it writable by the web server, and add the lines. Without them Drupal
   builds `http://` addresses behind BunkerWeb's HTTPS, and BunkerWeb's
   `Content-Security-Policy: form-action 'self'` blocks the redirect after each installer step. The
   installer then looks frozen: the **Save and continue** button does nothing and the browser console
   says `Sending form data ... violates the following Content Security Policy directive`. On a site
   that is already installed, Drupal made `settings.php` read-only: run `chmod u+w settings.php`,
   edit it, then `chmod 444 settings.php`.

Check it in **Reports, Status report**: no Error, and **Trusted Host Settings** reads **Enabled**.
Open an entry of **Reports, Recent log messages**: its **Hostname** is the visitor's address, not
BunkerWeb's.

## Install

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#install)
     for the web UI or plugin bundle method.
2. **Assign the template** to the service that fronts Drupal: in the web UI, **Services**,
   **Create new service**, Easy mode, **Templates**, **Drupal**.
3. **Review TLS automation** in step 1 of the form. Replace `www.example.com` with your host name.
   Let's Encrypt is on by default (HTTP challenge) and needs that name to resolve publicly to
   BunkerWeb on port 80. For an internal name turn **Automatic Let's Encrypt** off and set
   **Use custom certificate**, in the same step: give the certificate and key paths, or choose the two
   files. BunkerWeb 1.6.17 can keep Let's Encrypt on when you switch it off while creating the
   service: edit the service, switch it off again and save.
4. **Point the reverse proxy** (`REVERSE_PROXY_HOST`, step 2) at the Drupal backend: scheme, host
   and the port its web server listens on. The default `http://drupal:80` is a placeholder: replace
   `drupal` with your container name. The official `drupal:11-apache` image listens on `80`, so
   `http://drupal:8080` ends in a `502`. BunkerWeb must resolve that name from its own container.
5. **Keep the installer rate limits** on `/core/install.php` (`LIMIT_REQ_URL_1` and
   `LIMIT_REQ_RATE_1`, the pair without a title in step 4, below the labelled request limit)
   unless you are performing automated installs. Repeated `429` answers feed BunkerWeb's
   bad-behavior counter: about ten in a minute ban the client for one day.
6. **Save** the service, wait a few seconds for BunkerWeb to apply it, and verify you can browse
   the Drupal admin dashboard through the proxy.

## Settings

- Set `REVERSE_PROXY_HOST` to the internal scheme, host and port of your Drupal service.
- Increase `LIMIT_REQ_RATE_*` temporarily if scripted migrations or updates need
  higher throughput. `LIMIT_REQ_RATE` is `50r/s` on `/`: a lower rate answers `429` while the
  installer or an admin page loads its assets.
- `MAX_CLIENT_SIZE=100m` lets large image and file uploads through (the BunkerWeb default of
  `10m` answers `413`). Keep it in line with PHP's `upload_max_filesize` and `post_max_size`.
- `ALLOWED_METHODS` includes `PATCH`, `DELETE`, and `OPTIONS`, which JSON:API clients and CORS
  preflights need. If you do not expose JSON:API, enter `GET|POST|HEAD|QUERY`, BunkerWeb's default.
  Drupal's JSON:API is read-only until you select **Accept all JSON:API create, read, update, and
  delete operations** at `/admin/config/services/jsonapi`. Until then `POST`, `PATCH` and `DELETE`
  answer `405` from Drupal itself, and BunkerWeb shows its own error page instead of Drupal's body.
- `modsec-crs/drupal_false_positives.conf` makes three exceptions, all for JSON:API:
  - It accepts the `application/vnd.api+json` content type on paths that start with `/jsonapi`, and
    reads those bodies as JSON, so the SQL injection and script checks still look at every field.
    The same content type on any other path (`/node/add/article`, `/user/login`) still answers
    `403`. Cost: the list of accepted types on `/jsonapi` is a copy of the CRS 4.29 default plus the
    new type, so it does not follow CRS upgrades. Compare it with the CRS default (rule 901162)
    after you upgrade BunkerWeb.
  - It lets `application/octet-stream` through for
    `POST /jsonapi/<entity>/<bundle>[/<uuid>]/field_*`, the file upload route, and nowhere else.
    Cost: an upload on that route may use any content type.
  - If you changed the JSON:API path prefix (the JSON:API Extras module can), replace `/jsonapi`
    in the file. Delete the file if you do not use JSON:API.
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
- Check the Drupal status report (no Error, **Trusted Host Settings** enabled) and attempt cache
  clears or content edits to ensure rate limits and CRS exclusions do not block normal operations.
- With JSON:API writes enabled, create (`POST`), upload to a `field_*` file field, update (`PATCH`)
  and delete (`DELETE`) a node. A `vnd.api+json` request on `/node/add/article`, or SQL or script
  text in a JSON:API body, answers `403`.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
