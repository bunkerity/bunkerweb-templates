<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/matrix-light.svg">
    <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/matrix.svg" alt="Matrix logo" width="96" height="96" />
  </picture>
</p>

<h1 align="center">Tuwunel</h1>

<p align="center">Matrix homeserver reverse proxy with WebSocket and delegation</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Tuwunel-2.0.0-248651?labelColor=1d6f93&style=flat" alt="Tuwunel 2.0.0" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#dns-and-well-known-delegation">DNS and .well-known</a> ·
  <a href="#federation">Federation</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#files">Files</a> ·
  <a href="#validation">Validation</a>
</p>

Provision a curated BunkerWeb configuration for [Tuwunel](https://github.com/matrix-construct/tuwunel), a lightweight Matrix homeserver written in Rust and the maintained successor to conduwuit. This template ships sensible defaults for TLS, reverse proxying with WebSocket support, long federation/sync timeouts, request throttling, and `.well-known` client/server delegation so you can expose a Matrix homeserver through BunkerWeb quickly.

## At a glance

| Fact        | Value                           |
| ----------- | ------------------------------- |
| Template id | `tuwunel`                       |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Tuwunel 2.0.0 |
| Configs     | 2 (modsec-crs, server-http)     |
| Rule ids    | `9874000` to `9874999`          |
| Upstream    | `http://tuwunel-server-ip:8008` |

## Prerequisites

- A running Tuwunel homeserver reachable on your network. Tested with Tuwunel `2.0.0`
  (`ghcr.io/matrix-construct/tuwunel`), set up with `TUWUNEL_SERVER_NAME`, `TUWUNEL_ADDRESS=0.0.0.0` and
  `TUWUNEL_PORT=8008` (the port the template expects by default). Its `server_name` must be exactly the
  domain you give BunkerWeb: user ids, room aliases and media addresses contain it.
- The BunkerWeb UI or the ability to edit multisite settings directly.
- The public domain name that will act as your Matrix server name (the part after the `:` in your user IDs, e.g. `@alice:example.com`).

## DNS and .well-known Delegation

Matrix separates the **server name** (what appears in user IDs) from the **hostname that actually serves traffic**. Delegation lets `example.com` be your server name while BunkerWeb terminates TLS and proxies to Tuwunel.

1. Point an `A`/`AAAA` record for your server name (e.g. `example.com`) at the host running BunkerWeb.
2. In the last step of the easy-mode wizard, in the editor of `well-known.conf`, **replace `example.com` with your own homeserver domain** in the two `return 200` lines (not in the comment). You can also edit the file before you upload it during the import. The placeholders are not usable as-is: until you replace them the service answers `200` with `example.com` and no warning. When BunkerWeb is not on port `443`, add the port to both lines (`https://host:8448` in the client line, `host:8448` in the server line):
   - `/.well-known/matrix/client` advertises the client API base URL to Matrix clients.
   - `/.well-known/matrix/server` advertises the federation endpoint (`your-domain:443`) to other homeservers.

> [!NOTE]
> The wildcard `Access-Control-Allow-Origin` header on those endpoints is required by the Matrix `.well-known` spec so browser-based clients can read them cross-origin.

## Federation

This template delegates federation to port `443` via `/.well-known/matrix/server`, so other homeservers reach you over standard HTTPS through BunkerWeb — no need to expose the dedicated Matrix federation port `8448`. If you prefer serving federation directly on `8448` instead of delegating, drop the `/.well-known/matrix/server` block and open that port to Tuwunel separately.

## Install

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#install) for the web UI or
     plugin bundle method. When prompted for custom configs, upload `configs/server-http/well-known.conf` and `configs/modsec-crs/tuwunel_false_positives.conf`.

   > [!WARNING]
   > BunkerWeb 1.6.17's web UI answers `403` when you import this template, and when you create or
   > edit a service that uses it, if the web UI is itself protected by BunkerWeb's CRS (the setup
   > wizard default). The JSON in `well-known.conf` trips the CRS. Until a BunkerWeb release fixes
   > it, set the web UI service's Security mode to `detect` (Advanced mode, General) for the import
   > and for each save of this service, then set it back to `block`.

2. **Assign the template** to your Matrix service: choose it in the Templates menu of the service, in easy mode.
3. **Customize the settings** highlighted in the template steps. Step 1: the domain, and turn off Let's Encrypt for a name that is not public (if the toggle comes back on after you create the service, edit the service again). For your own certificate, turn on **Use custom certificate** in the same step and give the certificate and key. Step 2: replace the example host in `REVERSE_PROXY_HOST` with the name or address where BunkerWeb reaches Tuwunel. Last step: replace `example.com` in the editor, as described above.
4. **Save the service.** Saving applies it, there is no separate reload. Verify that `https://<your-domain>/.well-known/matrix/client` and `/.well-known/matrix/server` return your delegation JSON.

## Settings

- Replace the example host in `REVERSE_PROXY_HOST` with your Tuwunel instance (e.g. `http://tuwunel:8008`).
- `REVERSE_PROXY_INTERCEPT_ERRORS` is `no` so Matrix clients receive Tuwunel's JSON errors (`M_FORBIDDEN`, `M_NOT_FOUND`) instead of BunkerWeb's HTML error page.
- Adjust `MAX_CLIENT_SIZE` if you need to support larger media uploads.
- Tuwunel's own `max_request_size` defaults to 24 MiB, below `MAX_CLIENT_SIZE` (`50m`). An upload
  between 24 MiB and 50 MiB gets Tuwunel's `M_TOO_LARGE` JSON answer with a `413` (a client that is
  still sending can see a closed connection instead), and one above `50m` gets BunkerWeb's HTML
  `413` page. To allow larger uploads raise `max_request_size` in `tuwunel.toml` and
  `MAX_CLIENT_SIZE` together.
- `LIMIT_REQ_RATE` is `20r/s` for each client address, with no burst allowance. Element Web opens many
  parallel requests when it starts and when you switch rooms: with `15r/s` a start-up of an account
  with ten rooms got `429` answers, with `20r/s` it got none. Raise the value when many users share
  one address (an office behind NAT) or when remote homeservers federate with you, since they arrive
  from few addresses. BunkerWeb answers `429` with an HTML page, not Matrix JSON
  (`M_LIMIT_EXCEEDED`), so a client shows a generic error. Repeated `429` answers also count toward
  the bad behavior ban.
- `LISTEN_HTTP` is `no`: the service itself does not answer on plain HTTP. BunkerWeb's default server
  answers instead (a redirect to HTTPS with the default settings, a `404` on some setups). The
  Let's Encrypt HTTP challenge was not tested end to end, so check that issuance works before you
  rely on it, or use the DNS challenge.
- In step 4, a whitelisted address (`WHITELIST_IP`, `WHITELIST_ASN`) skips the whole WAF, not only
  the rate limit: every exception and every CRS rule is off for it.
- The crawler whitelist is disabled by default (`USE_WHITELIST=no`): a Matrix homeserver serves an API, not crawlable pages, so rDNS crawler bypass adds no value. Set `USE_WHITELIST=yes` (and add `WHITELIST_RDNS`) only if you have a specific reason.

### ModSecurity exclusions

`configs/modsec-crs/tuwunel_false_positives.conf` removes three CRS false positives that Matrix
clients trigger. Each rule is limited to one method and one path shape:

- `920440` (restricted file extension) on `GET`, `POST`, `PUT`, `DELETE` and `OPTIONS` of any path under
  `/_matrix/client/`, `/_matrix/federation/` and `/_matrix/key/` whose last segment ends in `.com`.
  A homeserver name ending in `.com` puts that ending on user ids, room aliases and room ids, so
  profile, typing, join (by alias or room id), knock, directory, room state and push rule requests,
  the browser's preflight for them and the federation requests of other homeservers all need it.
  Identifiers ending in any other restricted extension (`.bak`, `.log`, `.sql`), a `.com` segment
  under `/_matrix/media/` or the authenticated media paths, and a `.com` segment outside
  `/_matrix/` stay blocked.
- `920420` (content type) on `POST` and `PUT` of `/_matrix/media/{r0,v3}/upload`, the media
  upload sent as `application/octet-stream`.
- `942100` (SQL injection) on the `body` and `formatted_body` JSON fields of
  `PUT /_matrix/client/{r0,v3}/rooms/<room>/send/<type>/<txn>`, which hold chat text that can
  mention SQL. Every other SQL injection rule, field, and path stays active.

### Chat messages the WAF still blocks

The template does not exclude ordinary chat text from the CRS: that would turn the checks off for
everything people type. A message that reads like an attack gets a `403` page from BunkerWeb, and
the Matrix client shows a send error. In the tests these kinds of text were blocked, the same on
both Matrix homeservers:

| Chat text                                                                         | Rules fired                     |
| --------------------------------------------------------------------------------- | ------------------------------- |
| a system path: `/etc/hosts`, `/var/log/syslog`, `/tmp/x`, `~/.bashrc`, `C:\Windows\...` | `930120`, `932160`              |
| a command: `ls -la`, `sudo apt install foo`, `rm -rf /tmp/x`, `chmod`, `wget`, `python3 -c`, `curl ... \| bash`, `a && b`, `` `cmd` ``, `$(cmd)` | `932250`, `932260`, `932370`, `932235`, `932130` |
| code: `os.system(...)`, `eval(...)`, `console.log(...)`, `<script>`, a `#!` line  | `933160`, `934100`, `941xxx`, `932160` |
| SQL after a semicolon (`; drop table`) or `UNION SELECT`                          | `942350`, `942190`, `942270`    |
| `../` in text                                                                     | `930100`, `930110`              |
| a code block that holds a command, code or `UNION SELECT`                         | the rules above                 |

In Element the message shows "Some of your messages have not been sent", and Element holds every
later message in that room until you delete the failed one. Tell your users to rewrite the message
or to put it in a file. These pass: plain text, links, emoji, bold, mentions, replies, reactions,
a command name inside a sentence (`git push origin main`, `curl -X POST https://...`), and SQL
without a semicolon before the keyword (`SELECT * FROM users`, `admin' or '1'='1`), which is
rule `942100` in the `body` and `formatted_body` of a message.

## Files

- `template.json` – BunkerWeb template definition containing default settings, configs, and guided steps.
- `configs/modsec-crs/tuwunel_false_positives.conf` – ModSecurity CRS exclusions for Matrix clients.
- `configs/server-http/well-known.conf` – Serves the `/.well-known/matrix/client` and `/.well-known/matrix/server` delegation endpoints. Replace the placeholder `example.com` in the wizard's editor, or before you upload it.

## Validation

Run `jq . template.json` to confirm the JSON definition is valid before importing via the UI.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
