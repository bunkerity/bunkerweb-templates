<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/atuin.svg" alt="Atuin logo" width="96" height="96" />
</p>

<h1 align="center">Atuin sync server</h1>

<p align="center">Shell history sync server template with API client tuning</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Atuin-18.24.8-248651?labelColor=1d6f93&style=flat" alt="Atuin 18.24.8" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#configure-atuin-for-the-proxy">Configure Atuin</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings-and-tradeoffs">Settings</a> ·
  <a href="#crs-exception-scope-and-cost">CRS exceptions</a> ·
  <a href="#validation">Validation</a> ·
  <a href="#things-to-think-about">Things to think about</a>
</p>

Put an existing [Atuin](https://atuin.sh) sync server behind BunkerWeb with ModSecurity and the
OWASP CRS on: registration and sign-in, end-to-end encrypted history upload and download for
several machines, the key-value store, password change, store and account deletion.

Atuin has no web interface. Its clients (the `atuin` CLI and its daemon) talk to a JSON API, so
this template is tuned for those clients and was tested with the real `atuin` binary.

## At a glance

| Fact        | Value                                        |
| ----------- | -------------------------------------------- |
| Template id | `atuin`                                      |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Atuin 18.24.8 |
| Configs     | 1 (modsec-crs)                               |
| Rule ids    | `9850000` to `9850999`                       |
| Upstream    | `http://atuin:8888`                          |

## Prerequisites

- Run the Atuin sync server `18.24.8` (`ghcr.io/atuinsh/atuin:18.24.8`, command `start`) or a close
  release. BunkerWeb must reach `http://atuin:8888` on a private network; do not expose that port.
  The optional metrics listener (`[metrics]`, `127.0.0.1:9001`) is a separate port and is not proxied.
- The server needs a database (`ATUIN_DB_URI`): PostgreSQL, MySQL or SQLite. The template does not
  touch it; the tests used SQLite. A Compose file for the server with SQLite (run on `18.24.8`):

  ```yaml
  services:
    atuin:
      image: ghcr.io/atuinsh/atuin:18.24.8
      command: start
      environment:
        ATUIN_HOST: 0.0.0.0
        ATUIN_DB_URI: sqlite:///config/atuin.db
        ATUIN_OPEN_REGISTRATION: "true" # back to "false" once your accounts exist
      volumes:
        - atuin-data:/config
      restart: unless-stopped

  volumes:
    atuin-data:
  ```

  The image runs as the user `atuin` (uid 1000), and `/config` is the directory it can write to:
  keep the database file and the volume there. With `sqlite:///data/atuin.db` and a volume on
  `/data` the server stops at start with `unable to open database file`. The service name is the
  host name in `http://atuin:8888`, so put BunkerWeb on the same Compose network.
- Serve Atuin at the root of the hostname. A sub-path (Atuin's `path` setting) was not tested.
- Point your public hostname at BunkerWeb. `AUTO_LETS_ENCRYPT=yes` makes BunkerWeb request its
  certificate (the default HTTP challenge needs inbound port 80). For existing certificate files,
  use `AUTO_LETS_ENCRYPT=no`, `USE_CUSTOM_SSL=yes`, `CUSTOM_SSL_CERT` and `CUSTOM_SSL_KEY`. The
  easy-mode steps have no certificate fields: open the service in **Advanced** mode, **Custom SSL
  certificate**, switch on `USE_CUSTOM_SSL` and enter the two file paths, then switch
  `AUTO_LETS_ENCRYPT` off in **Let's Encrypt** in the same save (BunkerWeb 1.6.17 drops that switch
  when you turn it off while creating the service). For a name that is not public, turn Let's
  Encrypt off in step 1: it cannot get a certificate there.
- Use HTTPS. Registration and sign-in send the password in the request body. Set the client's
  `sync_address = "https://atuin.example.com"` in `config.toml`. Keep TLS 1.2 enabled
  (BunkerWeb's default is TLS 1.2 and 1.3): the macOS client cannot negotiate TLS 1.3
  ([atuin#4584](https://github.com/atuinsh/atuin/issues/4584)). The client speaks HTTP/1.1 over TLS.

> [!IMPORTANT]
> **CRS 4 required.** The exclusions in `configs/modsec-crs/atuin_false_positives.conf` select rules by CRS 4 tag. CRS 3.3.x
> names its tags differently, so on it they match fewer rules or none.

## Configure Atuin for the proxy

Set these on the Atuin container. "Without it" is what was measured on `18.24.8` or read in its
source (`crates/atuin-server/src/settings.rs`, `router.rs`, `client_ip.rs`).

| Atuin setting | Value | Without it |
| --- | --- | --- |
| `ATUIN_HOST` | `0.0.0.0` inside the container | The default `127.0.0.1` is not reachable from BunkerWeb. |
| `ATUIN_DB_URI` | your database | No default: the server does not start. |
| `ATUIN_OPEN_REGISTRATION` | `true` only while people create accounts, then `false` | The default `false` answers `POST /register` with `400 this server is not open for registrations`. With `true`, anyone who reaches the server can create an account and store data; neither this template nor Atuin limits that. Create accounts on the server instead with `atuin-server admin user create --email <email> <name>`. |
| `ATUIN_TRUSTED_PROXIES` | BunkerWeb's address on the Docker network (not a wide range) | The server logs BunkerWeb's address as `client.ip` for every request (measured: `172.21.0.3`). With BunkerWeb's address listed it logs the real client address (`172.21.0.1`). Atuin reads `X-Forwarded-For` from right to left and stops at the first address that is not a trusted proxy, so a forged `X-Forwarded-For: 6.6.6.6` from a client is ignored even though BunkerWeb appends to it (measured: both requests logged `172.21.0.1`). The address is only logged: Atuin does no per-address limiting. |

Security: keep `ATUIN_TRUSTED_PROXIES` to BunkerWeb's address. With a wide range such as
`0.0.0.0/0`, anyone who can reach Atuin's port directly chooses the address that shows in its
logs. `ATUIN_TRUSTED_PROXY_HEADER` (default `X-Forwarded-For`) can name `X-Real-IP` instead; BunkerWeb
sends both.

## Install

Import `template.json` and the file under `configs/` using the repository
[installation guide](../../README.md#install), or its plugin bundle layout. In the web UI the
**Missing Custom Configs** dialog lists the file as `modsec_crs/atuin_false_positives.conf`; upload
`configs/modsec-crs/atuin_false_positives.conf` there. Create the service from the template: in the
service wizard pick it in the **Templates** list above the steps (it is listed as `Atuin`), then
change only `SERVER_NAME` and `REVERSE_PROXY_HOST` (keep the upstream URL without a trailing slash
or path). Saving the service applies it.

Before enabling blocking on your instance:

1. Set the service's `SECURITY_MODE` to `detect`: no easy-mode step has it, so open the service in
   **Advanced** mode, **General**, set **Security mode** to `detect` and save. A new service starts
   in `block`.
2. Register, log in on a second machine, import a large history, run `atuin sync` on both, change
   the password, and read the BunkerWeb logs.
3. Confirm the included exclusions match the targets in your logs. Narrow or add exceptions only
   where your traffic requires them.
4. Set **Security mode** back to `block` after those checks and repeat the workflows.

False positives return `403` in block mode, and the client prints `Invalid request ... 403`. The
default bad-behavior status list includes `400 401 403 404 405 429 444`: ten matching responses in
60 seconds ban the address for one day, which for a household or an office is every machine behind
it. That is another reason for the detect run. A wrong password (`401`), an unknown user (`404`)
and every rate-limit answer (`429`) count too.

## Settings and tradeoffs

Each setting was put back to the BunkerWeb default and the failure below reproduced against the real
client (`LIMIT_REQ_*` as a group, then `LIMIT_REQ_URL_3` alone).

| Setting | Purpose and measurement |
| --- | --- |
| `REVERSE_PROXY_URL=/`, `REVERSE_PROXY_HOST` | Proxy the whole API. There is no WebSocket, SSE or cookie. |
| `REVERSE_PROXY_INTERCEPT_ERRORS=no` | The client prints Atuin's own `{"reason": ...}`. With the default `yes`, BunkerWeb swaps Atuin's `400`, `401`, `404` and `429` answers for its HTML page and the client prints a page dump instead: `login` with a wrong password printed `401 Unauthorized - password is not correct` with the setting and `Status 401 Unauthorized. Response body: <!doctype html>...` without it. BunkerWeb's own `403` and `5xx` pages are unchanged. |
| `ALLOWED_METHODS=GET\|POST\|HEAD\|DELETE\|PATCH` | `atuin account change-password` sends `PATCH /account/password` and `atuin store push --force` and `atuin account delete` send `DELETE`. The default `GET\|POST\|HEAD\|QUERY` answers `405` to all of them (`change-password` printed `unknown error`; `DELETE /api/v0/store` got `405`). |
| `MODSECURITY_REQ_BODY_NO_FILES_LIMIT=2m` | A history upload page is one JSON body of 100 records: median 103,918 bytes, 95 % under 127 KB, largest 163,300 (1,106 pages of two generated histories), and Atuin accepts up to 2 MiB. 33 of those 1,106 pages are above the default `128 KiB`: at the default, 14 of the first 442 got `400` (ModSecurity rule `200002`) and a 22,000-line upload stopped at page 7 with `Status 400 Bad Request` (600 of 21,947 items synced); with `2m` it completed. Bodies of 131,073 and 2,097,152 bytes got `400` at the default and reached Atuin with `2m`. |
| `MAX_CLIENT_SIZE=2m` | Atuin's own request limit is 2 MiB (axum's default, no setting): 2,097,152 bytes reach its JSON parser, 2,097,153 get `413`. With `2m`, nginx answers `413` for 2,097,153 bytes and above, the same status Atuin gives. At the default `10m` and `MODSECURITY_REQ_BODY_NO_FILES_LIMIT=2m` the same bodies got `400` from ModSecurity. Atuin's limit is per request, so one page of 100 records with very long commands (a pasted file in the history) can exceed it; the client then fails with `413` whatever the proxy does ([atuin#2887](https://github.com/atuinsh/atuin/issues/2887)). |
| `LIMIT_REQ_URL_1=^/api/v0/record/next$`, `LIMIT_REQ_RATE_1=600r/s` | Downloading history prefetches 8 pages in parallel: one client made up to 162 requests per second, three at once up to 197. At the default `2r/s` a download failed with `Rate limited; please wait before doing that again`; with `300r/s` three new machines downloading together still got `429`; with `600r/s` all three completed. |
| `LIMIT_REQ_URL_2=^/api/v0/record$`, `LIMIT_REQ_RATE_2=20r/s` | One path for the upload (`POST`, 3 per second at most) and the status calls (`GET`, up to 12 per second with six machines syncing together). At the default `2r/s` the second page of a 22,000-line upload got `429` and the client stopped. |
| `LIMIT_REQ_URL_3=^/api/v0/(?:capabilities\|me\|store)$`, `LIMIT_REQ_RATE_3=20r/s` | Every sync starts with `GET /api/v0/capabilities`. Six machines syncing in the same second made 4 of those answer `429` at the default (the client carries on, but each `429` counts toward the ban above); with `20r/s` none did. |
| `LIMIT_CONN_MAX_HTTP1=40` | One client downloads over up to 8 connections, so the default of 10 per address lets one machine sync at a time. Three new machines downloading together got `429` at the default (two of three over plain HTTP, all three over TLS) and all completed with `40` in both cases (the client speaks HTTP/1.1 over TLS, so the HTTP/2 limit does not apply). |
| `USE_GREYLIST=no`, `GREYLIST_USER_AGENT=^atuin/` | Optional step 7, off by default. The user agent pattern is already filled in, so the switch alone works. See "Only Atuin clients" below. |

`LIMIT_REQ_RATE` itself stays at its default `2r/s` for every other path: sign-in and registration
are limited to 2 per second and address, which also slows password guessing (six machines
signing in at the same second got `429` on 4 of 6 attempts). The limits are per address, so
machines behind one NAT share them. The headroom: the measured single-client peaks are 162 requests
per second (download) and 3 per second (upload); the template allows 600 and 20.

> [!NOTE]
> Both body allowances apply service-wide and raise buffering. They match Atuin's own ceiling, so
> lowering them below 2 MiB makes large upload pages fail.

### Only Atuin clients (optional)

Step 7 ("Security - Atuin clients") holds the switch `USE_GREYLIST` and the field
`GREYLIST_USER_AGENT`, already filled with `^atuin/`. Switch it on and save: BunkerWeb refuses every
client whose user agent does not start with `atuin/` (the real client sends `atuin/18.24.8` or
`atuin/18.25.0`). Measured on an easy-mode service with only the switch turned on: the real client
registered, logged in and synced (`200`), as did `atuin/18.0.0`; `curl`, a browser,
`python-requests`, `ATUIN/18.25.0` (the match is case sensitive) and an empty user agent got
`403`. With the switch off, the pattern is ignored and `curl` and browsers get `200`.

- **Do not empty the field while the switch is on.** The easy-mode field saves empty, and the
  switch with no pattern refuses every client, the Atuin client included. Type `^atuin/` again, or
  switch it off.
- **Refusals count toward the ban.** Each refused request answers `403`, and `403` is in the
  bad-behavior list: the tenth refusal within 60 seconds bans the address from the service for 24
  hours (`86400s`), and from then on every request from that address is refused, the real client
  too. A scanner behind the same address as your machines locks them out.
- **It is spoofable.** A client can send any user agent, so this only removes generic scanners and
  browsers. It also refuses your own `curl` health checks and any third-party client that does not
  send an `atuin/` user agent.

## CRS exception scope and cost

`configs/modsec-crs/atuin_false_positives.conf` is a service-scoped **modsec-crs** config. Its four
rules cover five paths (`/register` and `/login` share one). Every
rule is one chain: the exact path (`@streq`, or an anchored `@rx` for `/register` and `/login`, with
`t:normalizePath`) and the exact method, so the same text on another path, another method, a path
suffix (`/api/v0/record/`) or in another field stays blocked. `949110` and `959100` are never touched.
The record rule lists one `ctl:ruleRemoveTargetById=<id>;<target>` per field. The password rules use
`ctl:ruleRemoveTargetByTag=<tag>;<target>`, one line per CRS family tag. No `ruleRemoveById`, no regex
target (it fails the config test on BunkerWeb 1.6.17) and no id range (it passes the test and removes
nothing); `ARGS_POST:<name>` as a target also removes nothing, only `ARGS:<name>` works.

| Rule | Method and path | Target | Rule ids | Why |
| --- | --- | --- | --- | --- |
| `9850101` | `POST /api/v0/record` | `ARGS:json.array_0.data.data` ... `array_99.data.data` and `array_0.data.content_encryption_key` ... `array_99.data.content_encryption_key` | `930120` | The upload body is a JSON array of up to 100 records, so the argument names carry an index and no regex target works in `ctl` on BunkerWeb 1.6.17 (a regex target fails the config test); the 200 names are listed. The two fields are ciphertext: `data` is a `v4.local.` token and the key a `k4.local-wrap.pie.` wrapped key. `930120` matches the CRS word list `lfi-os-files.data`, which has entries that start with a dot (`.cvs`, `.nsr`, `.aws/`): they match the first characters of the random text right after the dot of `v4.local.` or `.pie.`. 8 of 442 upload pages (about 44,000 records) got `403` that way, always `930120`: 3 on `content_encryption_key`, 5 on `data`. Without the exclusion those pages never upload, and the client retries the same page on every sync. |
| `9850102` | `POST /register`, `POST /login` | `ARGS:json.password` | the ten family tags `OWASP_CRS/PROTOCOL-ATTACK` (921), `ATTACK-LFI` (930), `ATTACK-RFI` (931), `ATTACK-RCE` (932), `ATTACK-PHP` (933), `ATTACK-GENERIC` (934), `ATTACK-XSS` (941), `ATTACK-SQLI` (942), `ATTACK-SESSION-FIXATION` (943), `ATTACK-JAVA` (944) | Passwords are chosen by people and password managers. 209 of 40,000 generated passwords (13 generator shapes: Bitwarden, KeePassXC, 1Password, `openssl rand`, passphrases, text typed at a prompt) got `403` from 22 rule ids of the families 921, 932, 933, 934, 941 and 942, and 260 of 8,000 random 20 to 32 character strings made of all punctuation from 20 ids. A new id was still appearing at request 31,827 of 40,000 and the SQL-like seeds trip 930 and 941 ids, so the whole family is listed. The server only hashes the value. |
| `9850103` | `PATCH /account/password` | `ARGS:json.current_password`, `ARGS:json.new_password` | same ten tags | Same, for the old and the new password. |
| `9850104` | `DELETE /account` | `ARGS:json.password` | same ten tags | Same, confirming the deletion. |

Why tags and the `OWASP_CRS/` form. Listing the 114 paranoia-level-1 ids of those families one by one
takes 656 `ctl` actions. The ten tags remove the same 114 rules at the default paranoia level 1, in
40 lines: every id was checked against the CRS 4.29.0 files. Of the 109 other rules that carry these
tags, 107 run only at paranoia level 2 to 4 and two (`921170`, `941010`) are `pass,nolog` helpers that
score nothing. The short tags `attack-php` and `attack-java` do not exist, `attack-generic`
sits on other rules, and `attack-protocol` and `attack-rce` also carry the 920 rules and the response
web-shell rules, so 28 of the 114 ids have no matching short tag and they are not used (with them,
21 of the 92 ids that fire on a password field kept answering `403`).

Paranoia level. A tag removes every rule of its family from those fields at any level. Measured at
paranoia level 2 with 10,000 of the generated passwords: the list of those 114 ids left 1,839 of them at `403`
(27 ids of the families 920, 932, 934, 941 and 942), the ten tags left 7, all from `920230` and
`920271`, which belong to the 920 family and are not part of the exception. Levels 3 and 4 were not
run. If you raise the level, expect the same removal for every rule those tags carry.

**Cost.** On those five paths, ciphertext text and password text are no longer inspected by the
protocol-attack, LFI, RFI, RCE, PHP, generic, XSS, SQLi, session-fixation and Java rule families for the named fields.
A password that contains an attack string passes, which is harmless: Atuin hashes it and never
interprets it. Every other field is still inspected on those routes (`username`, `email`, the record's `id`, `idx`,
`host`, `parent`, `timestamp`, `version` and `tag`, query arguments, headers), so the same string
in `username`, in the query string, or in the record's `tag` still gets `403`. Ciphertext is stored
and returned as opaque text, so an attack string inside it has no effect. Pages with more than 100
records are not covered (the server advertises a page size of 100, `router.rs:111-114`; a record at
array index 100 still gets `403` from `930120`).

What stays blocked or passes at the edges (206 requests of 7 payload families, benign twins for the path
rows, and the same requests in a run without the exclusion; every row came out as listed):

- **Still `403` in a password field.** Six ids also match `REQUEST_BODY`, which is the whole JSON text, and
  no `ARGS` target removes them: `921110` (a request line such as `GET / HTTP/1.1`), `931110`
  (`include(...)`, `file=http://`) and `944100`-`944130` (`java.lang.Runtime`, `processbuilder`).
  A password that contains one of those strings gets `403`. `920270` (protocol enforcement, not a
  family of this exception) answers the same way to a NUL or to the text `%00` in a password: 1 of
  40,000 generated passwords.
- **Passes, by design.** A payload in the excluded field on its own route (`json.password` on
  `POST /login` and `POST /register`, both password fields on `PATCH /account/password`,
  `json.password` on `DELETE /account`), also when the key is written twice in one body, spelled
  with a JSON escape (`pass\u0077ord`) or sent with `application/json; charset=utf-8`. A query argument or a form field
  literally named `json.password` on those routes is skipped as well; Atuin ignores both.
- **Still `403`.** The same payloads in `username` or `email`; in `user.password` (nested), in an array
  or an object under `password`; in a urlencoded `password=` field or a query `password=`; in
  `new_password` on `/login` and on `DELETE /account`; in `current_password` on `/register`;
  in `password` on `PATCH /account/password`; on `/api/v0/record`, `/api/v0/me`, `/user/<name>` and `/`;
  with the wrong method (`DELETE` or `PATCH` on `/login`, `PATCH` or `DELETE` on `/register`,
  `POST` on `/account` and `/account/password`; nginx answers `405` to `PUT`); with `Password`, `PASSWORD`, `password ` or
  `password.x` as the field name; and with `X-HTTP-Method-Override: DELETE` on `POST /account`. A path
  trick such as `//login`, `/./login`, `/%6cogin` or `/login%00` never reaches an Atuin handler, so
  nothing reads the value (`404`, or `400` from nginx); `/login/`, `/login;x=1`, `/LOGIN`, `/login.json`,
  `/x/../login` and `/login%2f` get `403`.

Known residual: the record exclusion lists only `930120`, the one id that appeared in 1,106 pages
(about 110,000 records). A different rule that fires on some random ciphertext would block one
page the same way; the symptom is `403` on `POST /api/v0/record` with that rule id in the log. Add a
`ctl:ruleRemoveTargetById` line for that id to rule `9850101`.

## Validation

Run on 2026-11-18 against `ghcr.io/atuinsh/atuin:18.24.8` (SQLite) and `bunkerity/bunkerweb-all-in-one:1.6.17`
(CRS 4.29.0, libmodsecurity 3.0.17, default paranoia level and thresholds, `SECURITY_MODE=block`), with the real
`atuin` 18.24.8 binary from the GitHub release.

<details>
<summary>Test results</summary>

| Check | Result |
| --- | --- |
| Request matrix, 145 rows (130 requests the real client sent or that the account calls need, 15 attack controls) | With the template: 145 of 145 as expected. With the CRS config removed: 13 of the 130 requests got `403`; the controls (the same text in `username`, in the query string, on another path or method, at record index 100) stay `403` with the named rule id in both runs. |
| Upload volume through the template | 1,106 upload pages (about 110,000 records) from five generated histories: all `200`. Without the config 8 of the first 442 pages got `403`. |
| Password volume through the template | 40,000 generated passwords on `POST /login` (13 generator shapes, fixed seed): 1 got `403`, a password that contains the text `%00` (rule `920270`, not covered). Without the config 210 got `403`. An earlier run: 20,000 random 20 to 32 character strings, none got `403`; without the config 260 of 8,000 did. |
| One probe per rule id | 92 of the 114 paranoia-level-1 ids of the families trip on a password field with a payload taken from the CRS 4.29.0 regression tests. Each passes on all five password fields (`/login`, `/register`, `current_password`, `new_password`, `DELETE /account`) with the template and gets `403` without it. 16 ids read headers, file names, argument names or the query string and cannot fire on a JSON password field. The 6 that read `REQUEST_BODY` stay `403` (see above). |
| Attack table, 206 requests | Seven payload families (SQLi, XSS, RCE, LFI, PHP, RFI, Java) on the password fields of the four routes, their neighbours (other fields, paths, methods, content types, name variants, nested, array and urlencoded passwords, duplicate keys, path tricks and method override headers): `403` in every row where the paragraphs above say so, and a pass in the rows they call a gap (those got `403` in a run without the exception). |
| Each setting against the BunkerWeb default | See "Settings and tradeoffs": every row was reproduced with the real client with and without the setting. |
| Full smoke with bad behavior on | Register, import 50,000 history lines, sync, second machine logs in with the key and pulls (55,244 items on both, 7 seconds), key-value store with SQL, shell and HTML text, search, five more commands, password change, third machine signs in with the new password: 0 ModSecurity lines, no `403` or `444`, no ban. |
| Same smoke with `USE_GREYLIST=yes` and `GREYLIST_USER_AGENT=^atuin/` (3,000 lines) | Same result. |
| Same smoke again on the final configuration (the ten tags), bad behavior on, the template's limits on, 20,000 lines | Both machines 22,043 items (upload 1 min 26 s, download 3.4 s), five new commands on the second machine, key-value reads returned the stored SQL and HTML text, password change `200`, third machine signed in and synced: 0 ModSecurity lines, no `403` or `444`, no ban. Peak 164 requests per second on `GET /api/v0/record/next`. |
| Web UI import | `template.json` uploaded in Raw mode, the configuration uploaded in the Missing Custom Configs dialog, saved (`Template ... created successfully`); the service wizard lists the eight steps and the configuration in the last one. |
| `pre-commit` and the repository's template checker | Pass; the checker's warnings on `LIMIT_REQ_*` and `LIMIT_CONN_*` are answered by the table above. |

</details>

Not tested: the Let's Encrypt request (the tests ran on `*.localhost`), PostgreSQL and MySQL
backends, HTTP/2 and HTTP/3 clients, a server `path` prefix, Atuin Desktop, the scripts store, and
more than six machines at once.

## Things to think about

- **Large first sync.** Importing 22,000 history lines is 220 upload pages and took 83 to 124
  seconds in the tests (the machine was busy with other runs), and a second machine downloaded them in
  2.6 to 7 seconds at up to 178 requests per second (162 in the limits runs, 178 in the 50,000-line
  smoke). A 100,000-line import is about five times that. The limits above were sized on this; a much slower
  upstream lowers the request rate, not the page size.
- **Several machines behind one address.** The request and connection limits are per address.
  Two or three machines can sync together; the template allows about four first syncs at once
  (`600r/s`, `40` connections). More machines still work, one after the other. The client reports
  `Rate limited; please wait before doing that again` on `429` and tries again at the next sync.
- **`sync_frequency = "0"`** (sync after every command) makes the client call `/api/v0/record` about
  twice per command. A script loop of 60 commands in 8.1 seconds (7 per second) made 118 `GET` and 29
  `POST` on that path, and 60 of the `GET` and 7 of the `POST` got `429`. Keep the default of `5m`, or
  raise `LIMIT_REQ_RATE_2` if you really sync on every command.
- **The daemon** (`[daemon] enabled = true`) syncs on its own timer with the same `atuin/18.24.8` user
  agent through the same paths (measured with a 5-second timer: upload and downloads `200`).
- **Atuin Desktop** and the Atuin Hub login (`atuin account link`) were not tested.
- **A `403` that never goes away.** A client stuck on one `403` page means a CRS rule this template
  does not cover fired on that page (see "Known residual"). Check the BunkerWeb log for the rule id.
- **Sync address.** Write `sync_address = "https://atuin.example.com"` with no path. A server
  `path` prefix was not tested.
- **Rate limits and bans.** `429` is in BunkerWeb's bad-behavior status list. If you lower
  the limits, the same machines that get `429` also get banned after ten answers in a minute.
- **Client trust.** The client uses the system certificate store. A self-signed certificate needs the
  certificate in that store (the harness used `SSL_CERT_FILE`); there is no insecure switch.
- **Monitoring.** `GET /healthz` answers without authentication. Use it for health checks. With the
  optional user-agent step on, a `curl` check gets `403`: set an `atuin/` user agent on it.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
