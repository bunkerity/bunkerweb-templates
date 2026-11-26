<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/seafile.svg" alt="Seafile logo" width="96" height="96" />
</p>

<h1 align="center">Seafile</h1>

<p align="center">File sharing template with WebDAV-aware proxy defaults</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Seafile-11.0.13-248651?labelColor=1d6f93&style=flat" alt="Seafile 11.0.13" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#docker-compose-example">Docker Compose</a> ·
  <a href="#install">Install</a> ·
  <a href="#uploads-webdav-sync-and-notifications">Uploads, WebDAV and sync</a> ·
  <a href="#crs-exception-scope-and-cost">CRS exceptions</a> ·
  <a href="#validation">Validation</a>
</p>

This template proxies Seafile through BunkerWeb with upload, WebDAV, desktop sync, rate-limit,
compression, and client-compatibility defaults, and ships the ModSecurity exceptions that browser
uploads, WebDAV uploads, sync clients and the system admin console need. The deployment example
targets Seafile Community Edition 11.0.x; use Seafile's version-specific Compose file when
deploying another release.

## At a glance

| Fact        | Value                                        |
| ----------- | -------------------------------------------- |
| Template id | `seafile`                                    |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Seafile 11.0.13 |
| Configs     | 1 (modsec-crs)                               |
| Rule ids    | `9882000` to `9882999`                       |
| Upstream    | `http://seafile`                             |

## Prerequisites

- A running Seafile server that BunkerWeb can reach.
- A public hostname pointing to BunkerWeb.
- Access to the BunkerWeb web UI or service environment variables.

## Docker Compose example

This is a small Seafile 11.0 example based on the
[official deployment guide](https://manual.seafile.com/11.0/docker/deploy_seafile_with_docker/). It binds
Seafile only to localhost because BunkerWeb is the public entry point. Attach the BunkerWeb container to
the `seafile-net` network so the default `http://seafile` upstream resolves; otherwise, use an upstream
address reachable from BunkerWeb.

```yaml
services:
  db:
    image: mariadb:10.11
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: "${MYSQL_ROOT_PASSWORD:?Set MYSQL_ROOT_PASSWORD}"
      MYSQL_LOG_CONSOLE: "true"
      MARIADB_AUTO_UPGRADE: "1"
    volumes:
      - /opt/seafile-mysql/db:/var/lib/mysql
    networks:
      - seafile-net

  memcached:
    image: memcached:1.6.18
    restart: unless-stopped
    entrypoint: memcached -m 256
    networks:
      - seafile-net

  seafile:
    image: seafileltd/seafile-mc:11.0-latest
    restart: unless-stopped
    ports:
      - "127.0.0.1:7841:80"
    volumes:
      - /opt/seafile-data:/shared
    environment:
      DB_HOST: db
      DB_ROOT_PASSWD: "${MYSQL_ROOT_PASSWORD:?Set MYSQL_ROOT_PASSWORD}"
      TIME_ZONE: Etc/UTC
      SEAFILE_ADMIN_EMAIL: "${SEAFILE_ADMIN_EMAIL:-admin@example.com}"
      SEAFILE_ADMIN_PASSWORD: "${SEAFILE_ADMIN_PASSWORD:?Set SEAFILE_ADMIN_PASSWORD}"
      SEAFILE_SERVER_LETSENCRYPT: "false"
      SEAFILE_SERVER_HOSTNAME: seafile.example.com
      FORCE_HTTPS_IN_CONF: "true"
    depends_on:
      - db
      - memcached
    networks:
      - seafile-net

networks:
  seafile-net:
    name: seafile-net
```

Set the required environment variables before starting the stack:

```bash
export MYSQL_ROOT_PASSWORD='replace-with-a-long-random-value'
export SEAFILE_ADMIN_PASSWORD='replace-with-a-different-long-random-value'
docker compose config --quiet
docker compose up -d
```

The two environment variables `SEAFILE_SERVER_HOSTNAME` and `FORCE_HTTPS_IN_CONF` are not enough
behind a proxy: sign-in fails until you add the four `seahub_settings.py` lines from
[Install](#install) step 4.

## Install

1. Import `template.json` by following the repository's
   [installation guide](../../README.md#install).
2. In the web UI, create a service in Easy mode and pick **Seafile** in the **Templates** menu above
   the steps. In an environment-variable setup, assign `USE_TEMPLATE=seafile` instead.
3. In the first step, replace `SERVER_NAME` with your hostname and `EMAIL_LETS_ENCRYPT` with your
   own address: the example `admin@example.com` is not yours. For a hostname that is not public,
   turn off **Automatic Let's Encrypt**. BunkerWeb 1.6.17's web UI drops a toggle you switch off
   while creating the service, so save the service, open it again, switch the toggle off and save
   once more. In the second step, replace `REVERSE_PROXY_HOST` if the upstream is not
   `http://seafile`: the default assumes both containers share a Docker network.
4. Tell Seafile its public HTTPS address. This is required on every deployment, new or existing.
   Seafile 11's Docker environment sets only the file-server URL (`FILE_SERVER_ROOT`). It leaves
   `SERVICE_URL` on `http://` and sets no trusted origin, so sign-in answers `403` with
   `Origin checking failed ... does not match any trusted origins` in `seahub.log`. Add these lines
   to `/opt/seafile-data/seafile/conf/seahub_settings.py` and replace the example hostname:

   ```python
   SERVICE_URL = "https://seafile.example.com"
   FILE_SERVER_ROOT = "https://seafile.example.com/seafhttp"
   CSRF_TRUSTED_ORIGINS = ["https://seafile.example.com"]
   SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")
   ```

   Do not add a trailing slash to the trusted origin. Restart Seafile
   (`docker compose restart seafile`) and wait until it answers before you sign in.
5. Save the service: BunkerWeb applies it, there is nothing else to reload. Sign in through the
   public URL, and upload and download a test file.

> [!NOTE]
> The `COOKIE_FLAGS_4` override deliberately omits `HttpOnly` for Seafile's `sfcsrftoken` cookie. It
> narrows BunkerWeb's wildcard cookie policy so Seahub can read the CSRF token without weakening the
> flags on other cookies.

`REVERSE_PROXY_HEADERS` blanks `X-Forwarded-Protocol`. BunkerWeb sends both `X-Forwarded-Proto` and
`X-Forwarded-Protocol`, and Seafile's gunicorn answers `400` to every request that carries the pair.
Seahub only reads `X-Forwarded-Proto`, so removing the duplicate is enough. Keep this header override
if you add routes of your own that proxy to Seahub.

## Uploads, WebDAV, sync, and notifications

> [!WARNING]
> `MAX_CLIENT_SIZE=512m` limits each request, not the total file size used by chunked clients. Raising it
> also raises ModSecurity's in-memory request-body limit, so increase it only for clients that need larger
> single requests.

Browser uploads, WebDAV uploads, desktop sync and the system admin console (`/sys/`) each need one
ModSecurity exception, which the template ships. See
[CRS exception scope and cost](#crs-exception-scope-and-cost).

The template allows WebDAV methods and exempts `/seafdav` from the JavaScript challenge, but SeafDAV is
disabled by default in Seafile. Enable it in `seafdav.conf`, set `share_name = /seafdav`, and test with a
WebDAV client. See Seafile's [WebDAV documentation](https://manual.seafile.com/11.0/extension/webdav/).

Behind a TLS proxy, WebDAV `COPY` and `MOVE` (rename and move in Finder, Windows Explorer and rclone)
fail with `502`. Seafile's own nginx forwards `X-Forwarded-Proto http` to SeafDAV, so SeafDAV rejects
the `https://` address in the `Destination` header. The template cannot change this, so fix it inside
Seafile. On the Docker host, change that header in the `location /seafdav` block only, then reload
the nginx of the Seafile container:

```bash
docker compose exec seafile sed -i \
  '/location \/seafdav {/,/}/ s/X-Forwarded-Proto \$scheme;/X-Forwarded-Proto https;/' \
  /shared/nginx/conf/seafile.nginx.conf
docker compose exec seafile nginx -s reload
```

The change lives in the `/shared` volume and survives a restart of the container.

The Seafile desktop client's sync (`/seafhttp/repo/...`) also goes through the template. The
exceptions for it were tested with the command-line client `seaf-cli` 9.0.4 against Seafile 11.0.13.
The graphical desktop and mobile apps send the same requests but were not run through BunkerWeb.

Public share links (`/d/<token>/` for a folder, `/f/<token>/` for a file) skip the JavaScript
challenge, so `curl`, `wget` and `?dl=1` links in other pages download without a browser. Upload
links (`/u/d/<token>/`) are opened in a browser and still get the challenge.

WebSockets are not enabled on the main `/` upstream. Seafile's optional notification server listens on
a separate port and needs dedicated `/notification/ping` and `/notification` proxy routes; configure
those only after enabling the service by following Seafile's
[notification server documentation](https://manual.seafile.com/11.0/deploy/notification-server/).

The bad-behavior status list excludes `401`, `403`, and `404`. Authentication, permission checks, and
missing-object probes can legitimately return those codes, so counting them toward an IP ban can lock
out normal sync or WebDAV clients.

## CRS exception scope and cost

`configs/modsec-crs/seafile_false_positives.conf` is a service-scoped **modsec-crs** config with
six rules. Each one names a path prefix and a method, and removes only the CRS checks listed here.

| Rule      | Path and method                                            | What it removes                                                                                                                                              |
| --------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `9882100` | `POST /seafhttp/upload-aj/`, `update-aj/`, `upload-api/`, `update-api/` | CRS `920450` for the `Content-Range` header and `920600` for the `Accept` header (the web uploader sends both on every chunk).                        |
| `9882110` | `PUT /seafdav/`                                            | CRS `920340`, `920420` and `920640` (WebDAV clients send a file with its own Content-Type, or with none).                                                    |
| `9882120` | `GET` or `POST` under `/sys/`                              | CRS `930130` (Seafile's system admin pages live under `/sys/`).                                                                                              |
| `9882130` | `PUT /seafhttp/repo/<id>/commit/`, `block/`, `fs/`         | CRS `920340` and `920640` (the sync client stores commits and blocks with no Content-Type).                                                                  |
| `9882131` | `POST /seafhttp/repo/<id>/recv-fs/`                        | The body of the request: the sync client sends binary objects labelled as a web form, and CRS rules such as `920250`, `920270` and `921150` flag some of them. |
| `9882132` | `POST /seafhttp/repo/<id>/recv-fs/` with no query string   | The form fields the CRS reads out of that body, and CRS `920250` as a whole. With a query string this rule does not apply and both checks are back on.            |

Cost: the same checks stay on for every other path and method (each rule has controls that return
`403`). Under `/seafdav/` a `PUT` is accepted with any Content-Type. On the five upload addresses the
two header checks are off, but other restricted headers are still blocked. Rules `9882131` and `9882132`
are the widest: on a request with no query string, which is how the sync client calls `recv-fs`, no CRS
attack rule reads the content of the body, and only a client with a valid repository token reaches the
path. They leave the path, the headers, the cookies and the query string of those requests under the
CRS, so an attack sent there still answers `403`. Rule `9882132` is the one that depends on the query
string. When the request has one, only `9882131` applies: the CRS no longer reads the raw text of the
body, but it reads the form fields, their names and the bytes of the body again (`920250`, `920270`,
`921150` and the attack rules), so a binary body answers `403`. Delete a rule from
the config if you do not use the feature (for example `9882110` without SeafDAV, or `9882130`,
`9882131` and `9882132` without desktop sync).

To adjust a rule in the web UI, open **Configs**, pick the custom config of your service (type
`MODSEC_CRS`), edit it and save. A rule you copy from here for another path needs an id of your own
(1 to 99,999).

## Validation

Before importing the template:

```bash
jq . template.json
jq -e --arg expected "$(basename "$PWD")" '.id == $expected' template.json
jq -e '(.settings | keys | sort) == ([.steps[] | .settings[]] | sort)' template.json
```

Tested on BunkerWeb 1.6.17 with CRS 4.29 and Seafile 11.0.13 (`seafileltd/seafile-mc:11.0-latest`),
with the service created in the web UI from the imported template, in Easy mode.

After applying it:

- Confirm the BunkerWeb scheduler reload completes without a template or unknown-setting error.
- Sign in, browse a library, and upload and download a file larger than the default 10 MiB limit
  (a 15 MB file went up in two chunks through the browser).
- If SeafDAV is enabled, run `curl -u user:password -X PROPFIND -H 'Depth: 1'`
  `https://seafile.example.com/seafdav/` and confirm it is not challenged by Antibot.
- Open **System Admin** as an administrator and confirm its pages load.
- Check BunkerWeb logs for rate-limit, bad-behavior, and ModSecurity events before changing security
  controls.

What the tests showed:

| Check                           | Result                                                                                                                                                            |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Sign-in                         | Without the four `seahub_settings.py` lines: `403 Origin checking failed`. With them: `302` to the library list.                                                  |
| Same requests without the rules | Web upload `403` (`920450`, `920600`), WebDAV `PUT` `403` (`920340`, `920420`, `920640`), `/sys/` `403` (`930130`), sync `PUT` of a commit `403` and `recv-fs` `403` (`920250`, `920270`, `921150`). |
| Same requests with the rules    | Browser upload `200`, WebDAV `PUT` of a small file, a 15 MB file, an image and a PDF, with accents, spaces and an apostrophe in the names `201` or `204`, `COPY` and `MOVE` `204` (after the fix above), `/sys/info/` and `/sys/users/` `200`, `seaf-cli` sync of four files of 0.04 to 2.5 MB `synchronized`, with `recv-fs` answering `200` and no CRS rule logged. |
| Controls                        | The cases listed below the table, all still `403` with the CRS rule that fired before.                                                                             |
| Share links                     | `curl` of `/f/<token>/?dl=1` before: `302` to `/challenge`. After: `302` to the file, which downloaded byte for byte.                                             |

Controls that still answer `403` with the template applied: the two header checks (`Content-Range`,
`Accept` with a charset) on a `GET` or `PUT` of an upload address and on `/api2/` or `/seafhttp/files/`;
a `Proxy` header on an upload address; a `PUT` or `POST` with or without a Content-Type on `/api2/`
and `/seafhttp/other/`; a `POST` to `/seafdav/`; a `PUT` of a sync block that sets a Content-Type;
`/api2/x/sys/`, `/seafhttp/sys/x` and a `.htaccess` path; a script payload in the query of `/sys/info/`,
in the body of an upload address and in the body of `check-fs`; and an attack in the query string,
a cookie, the `Referer` or the `User-Agent` of a `recv-fs` request. The same payload in the body of
`recv-fs` passes the CRS when the request has no query string, which is the cost of rules `9882131`
and `9882132`; with a query string, such as `?v=1`, the same body answers `403`.

## Known limits

- **Not run through BunkerWeb:** the graphical Seafile desktop client, the mobile apps, Finder,
  Windows Explorer and rclone against SeafDAV (only `curl` and `seaf-cli` ran), the web editor's
  file save (`update-aj`, covered by the same rule as the upload), OnlyOffice, and Seafile releases
  other than 11.0.13.
- **Sub-path and multiple hostnames** were not tested.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
