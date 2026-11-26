<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/synapse-light.svg">
    <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/synapse.svg" alt="Synapse logo" width="96" height="96" />
  </picture>
</p>

<h1 align="center">Synapse</h1>

<p align="center">Matrix homeserver template with well-known delegation</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
  <img src="https://img.shields.io/badge/Synapse-1.165.0-248651?labelColor=1d6f93&style=flat" alt="Synapse 1.165.0" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#well-known--dns-delegation">Well-known / DNS</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#files">Files</a> ·
  <a href="#validation">Validation</a>
</p>

Provision a BunkerWeb configuration for [Synapse](https://element-hq.github.io/synapse/),
the reference Matrix homeserver. The template terminates HTTPS, reverse proxies
the `/_matrix/` client and federation API, raises the upload limit for media,
applies rate limiting, and serves the `.well-known` delegation documents so
clients and federating servers can discover the homeserver.

## At a glance

| Fact        | Value                          |
| ----------- | ------------------------------ |
| Template id | `synapse`                      |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Synapse 1.165.0 |
| Configs     | 2 (modsec-crs, server-http)    |
| Rule ids    | `9872000` to `9872999`         |
| Upstream    | `http://mysynapse-server:8008` |

## Prerequisites

- A running Synapse instance reachable from the BunkerWeb instance (default
  client/federation listener on port `8008`). Tested with Synapse `1.165.0`
  (`matrixdotorg/synapse`) with `x_forwarded: true` on its listener, which the generated
  `homeserver.yaml` sets.
- A domain you control, with DNS pointing the served hostname at BunkerWeb.
- Access to the BunkerWeb UI or environment variables to assign template
  settings.

## Well-known / DNS delegation

Matrix uses `.well-known` delegation to publish your Matrix hostname. In the last step of the
easy-mode wizard, in the editor of `well-known.conf`, replace `example.com` with the hostname
where clients and other servers reach BunkerWeb in the two `return 200` lines (not in the
comment). Delegation names BunkerWeb's public HTTPS endpoint, never Synapse's own listener: do not
put the internal port `8008` there. The server line already ends in `:443`, BunkerWeb's public
HTTPS port, so keep it when BunkerWeb answers on `443` (`matrix.example.com:443`). When BunkerWeb's
public HTTPS port is another one, replace `443` with it in the server line and add it to the
client line: `https://matrix.example.com:8448` and `matrix.example.com:8448`. You can also edit the
file before you upload it during the import. The `server_name` in your Synapse `homeserver.yaml`
must match the delegation you publish here.

## Install

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#install)
     for the web UI or plugin bundle method.
   > [!WARNING]
   > BunkerWeb 1.6.17's web UI answers `403` when you import this template, and when you create or
   > edit a service that uses it, if the web UI is itself protected by BunkerWeb's CRS (the setup
   > wizard default). The JSON in `well-known.conf` trips the CRS. Until a BunkerWeb release fixes
   > it, set the web UI service's Security mode to `detect` (Advanced mode, General) for the import
   > and for each save of this service, then set it back to `block`.

2. **Assign the template** to the site serving Synapse: choose it in the Templates menu of the
   service, in easy mode.
3. **Set the domain and TLS** in step 1: `SERVER_NAME` is the hostname you expose. For a name that is
   not public, turn off Let's Encrypt (if the toggle comes back on after you create the service,
   edit the service again). For your own certificate, turn on **Use custom certificate** in the
   same step and give the certificate and key.
4. **Update the upstream** in step 2: replace the example host in `REVERSE_PROXY_HOST` with the name
   or address where BunkerWeb reaches Synapse (for example `http://synapse:8008`). The default
   `mysynapse-server` exists nowhere and gives a `502`.
5. **Set the delegation** in the last step, as described above, so the well-known documents
   advertise your real Matrix hostname.
6. **Save the service.** Saving applies it, there is no separate reload. Verify login, federation
   and media upload through the proxy.

## Settings

### TURN / VoIP (calls)

The TURN server used for Matrix voice and video calls should be set up through
your firewall (direct port forwarding to the TURN/coturn service) rather than
routed through BunkerWeb. TURN/VoIP traffic proxied through the reverse proxy
can cause call interruptions and failed calls; keeping it outside BunkerWeb
gives the best performance and reliability. This template intentionally does not
proxy TURN.

### Only `/_matrix/` is proxied

The template proxies the Matrix client and federation API under `/_matrix/` and nothing else:
`/_synapse/client/` (single sign-on, password reset), `/_synapse/admin/` and `/health` answer a
BunkerWeb `404`. For SSO or password reset, add a second reverse proxy URL to the same service:
open the service in the **Advanced** tab, choose the **Reverse proxy** section and click **Add**.
Give the new rule the same host as the first one and the URL `/_synapse/client/`, then save the
service. Synapse then answers on that path (tested with a second rule: the path reached Synapse,
`/_synapse/admin/` stayed a BunkerWeb `404`). Leave `/_synapse/admin/` unproxied unless you restrict it to
administrators, because it is Synapse's admin API.

### Request rate and `429`

`LIMIT_REQ_RATE` is `20r/s` for each client address, with no burst allowance. Element Web opens
many parallel requests when it starts and when you switch rooms: with `15r/s` a start-up of an
account with ten rooms got `429` answers, with `20r/s` it got none. Raise the value when many
users share one address (an office behind NAT) or when remote homeservers federate with you, since
they arrive from few addresses. BunkerWeb answers `429` with an HTML page, not Matrix JSON
(`M_LIMIT_EXCEEDED`), so a client shows a generic error. A `429` does not count toward the bad
behavior ban: the template sets `BAD_BEHAVIOR_STATUS_CODES` to `405` only, so a shared address is
not banned for hitting the rate limit. If you add `429` to that setting, it is.

### Upload size

> [!TIP]
> `MAX_CLIENT_SIZE` is set to `50m` so media uploads succeed. Align this with the
> `max_upload_size` configured in your Synapse `homeserver.yaml`, and raise both
> if you expect larger files.

### ModSecurity exclusions

`configs/modsec-crs/synapse_false_positives.conf` removes three CRS false positives that Matrix
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

- `template.json` – Template definition with steps for TLS, reverse proxy, HTTP
  limits, whitelisting, and well-known delegation.
- `configs/modsec-crs/synapse_false_positives.conf` – ModSecurity CRS exclusions for Matrix clients.
- `configs/server-http/well-known.conf` – Serves `/.well-known/matrix/client`
  and `/.well-known/matrix/server` for delegation. **Replace the placeholder
  `example.com` with your own Matrix hostname** in the wizard's editor, or before you upload it.

## Validation

- Run `jq . template.json` to ensure the definition is valid JSON.
- Confirm `curl https://<server_name>/.well-known/matrix/client` and
  `/.well-known/matrix/server` return the expected JSON.
- Use the [Matrix Federation Tester](https://federationtester.matrix.org/) to
  confirm delegation and federation resolve correctly.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
