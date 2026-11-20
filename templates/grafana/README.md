<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/grafana.svg" alt="Grafana logo" width="96" height="96" />
</p>

<h1 align="center">Grafana</h1>

<p align="center">Dashboard template with query and HTML-response WAF tuning</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#crs-exception-scope-and-cost">CRS exceptions</a> ·
  <a href="#validation">Validation</a>
</p>

Proxy Grafana at a dedicated hostname with Grafana Live, JSON API errors and CRS exceptions that let
SQL data sources work: queries in panels and Explore, and dashboard saves from the web UI. Review
[CRS exception scope and cost](#crs-exception-scope-and-cost) before importing: SQL text is not
checked in query fields, and the API dashboard save accepts some injection strings in every field.

## At a glance

| Fact        | Value                         |
| ----------- | ----------------------------- |
| Template id | `grafana`                     |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4 |
| Configs     | 2 (modsec-crs, modsec)        |
| Rule ids    | `9858000` to `9858999`        |
| Upstream    | `http://grafana:3000`         |

## Prerequisites

1. Run Grafana on a network reachable by BunkerWeb at `http://grafana:3000` or change the upstream.
   Set `GF_SERVER_ROOT_URL=https://grafana.example.com/`, including any public nonstandard port,
   and open Grafana only at that exact URL. If the address in the browser differs (http instead of
   https, another port or host), Grafana Live answers `403` on `/api/live/ws` and retries, and
   BunkerWeb's bad behavior ban locks your IP out of the whole service for 24 h (**Bans**, then
   **Unban**). Correct the URL, or list the browser's origin in Grafana's `allowed_origins` setting
   of the `[live]` section. With TLS terminated by BunkerWeb, keep Grafana's upstream protocol
   HTTP. See the official
   [proxy guide](https://grafana.com/tutorials/run-grafana-behind-a-proxy/) and
   [`root_url` reference](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/#root_url).

## Install

1. Follow the repository [installation guide](../../README.md#install). Raw import
   requires `template.json`, `configs/modsec-crs/grafana_false_positives.conf` and
   `configs/modsec/grafana_sql_fields.conf`; plugin bundles use the layout in that guide.
2. Select `grafana` for the service (or set `USE_TEMPLATE=grafana`), change `SERVER_NAME` and
   **all three** upstream settings: `REVERSE_PROXY_HOST`, `_1` and `_2`. Review the guided steps.
3. Reload and validate your dashboards. Automatic Let's Encrypt requires DNS and HTTP-01
   reachability. For another certificate source, set `AUTO_LETS_ENCRYPT=no` and configure
   `USE_CUSTOM_SSL` and its certificate settings; local tests can use `GENERATE_SELF_SIGNED_SSL`.
   The guided steps show none of these certificate settings: set them in the **Advanced** tab of
   the service, under **Custom SSL certificate** or **Self-signed certificate**.

> [!NOTE]
> This recipe serves `/`; subpath hosting needs separate configuration and validation. Keep Grafana's
> own authentication and dashboard/data-source permissions configured.

## Settings

- `REVERSE_PROXY_URL_1=/api/live/` and `REVERSE_PROXY_WS_1=yes` enable Grafana Live upgrades on
  that prefix only. The main proxy location keeps WebSocket support at its default `no`.
- `REVERSE_PROXY_INTERCEPT_ERRORS=no` preserves Grafana's JSON API errors, including its own
  `401` and `413` responses, instead of replacing their bodies with BunkerWeb HTML.
- `ALLOWED_METHODS=GET|POST|HEAD|OPTIONS|PUT|PATCH|DELETE` allows API updates/deletes and drops
  the default `QUERY`; append it if required.
- `USE_LIMIT_REQ=yes`, `LIMIT_REQ_URL=/`, `LIMIT_REQ_RATE=100r/s` support page/query bursts.
  This weakens throttling compared with the default `2r/s`, while retaining a finite limit.
- `LIMIT_CONN_MAX_HTTP2=400` (default `100`) lets **Connections**, **Add new connection** load. That
  page requests about 250 plugin logos at once over HTTP/2: with the default, about 140 of them get
  `429` and the bad behavior ban locks your IP out of the service for 24 h. The limit counts
  streams per client address, so it also bounds how many requests one client can hold open at the
  same time; lower it if the page is not needed.
- `MODSECURITY_REQ_BODY_NO_FILES_LIMIT=32m` raises non-file body inspection capacity **for the
  whole service**, increasing potential memory use. `/api/dashboards/` has a `64m` location
  body limit via `REVERSE_PROXY_MAX_CLIENT_SIZE_2`; other paths retain the service default
  `MAX_CLIENT_SIZE` of `10m`. The web UI saves dashboards through `/apis/dashboard.grafana.app/`, not
  through `/api/dashboards/`, so a UI save above `10m` gets `413` and only API saves reach `64m`.
  The 32 MiB non-file limit still applies on the dashboard location.
  Do not interpret `64m` as a usable dashboard size: historical Grafana 13.2.3 itself rejected
  dashboards over 16 MB. A 15 MiB save passed; behavior on other versions needs checking.
- Default bad-behavior protection remains enabled. Repeated API `400`, `401` or `404` replies
  can contribute to client bans; inspect logs during validation.

## CRS exception scope and cost

The exceptions live in two files at service scope: `configs/modsec-crs/grafana_false_positives.conf`
(rules `9858102`, `9858103`, `9858500`, `9858501`) and `configs/modsec/grafana_sql_fields.conf`
(six lines without a rule id).

- `configs/modsec/grafana_sql_fields.conf` skips CRS rules on one kind of field: the SQL text of a
  query. Grafana sends it in a JSON field named `rawSql`, in `POST /api/ds/query` (panels and
  Explore), in dashboard saves (the web UI saves through `/apis/dashboard.grafana.app/`, API
  clients through `/api/dashboards/db`), and under the same name in alert rules and library
  panels, which were not tested. The lines remove the SQL injection rules (tag `attack-sqli`, so
  all the `942xxx` rules) and `932130`, `932280`, `933135`, `933160`, `933180` from every JSON field
  whose name ends in `.rawSql`. Each one is needed by ordinary queries: the `942xxx` rules read
  `LIKE`, `UNION ALL`, `CAST ... IN (SELECT ...)`, `information_schema` and comments; `933180`
  reads a Grafana macro such as `$__timeFilter(time)`; `932130` and `933135` read a template
  variable in braces such as `${host}` and a comparison with a subquery; `933160` reads functions
  such as `substr` and `md5`; `932280` reads a JSON path such as `'{a,b}'` in PostgreSQL.
  The field is matched by name, on any path of the service and whatever the capitalisation, so a
  request to another Grafana path that carries a nested field called `rawSql` skips the same rules
  for that field. A `rawSql` key at the top of the JSON body, every other field, header and
  parameter, and every other rule (XSS, path traversal and file inclusion in a `rawSql` value)
  stay checked. Cost: SQL injection is not detected in these fields, so only users who may query
  your data sources should reach Grafana. Add a rule id to the file if a query of yours is
  blocked by another rule; the log names it.
- Rule `9858102` applies only to `POST /api/dashboards/db`, the API that provisioning tools use. It
  removes `932340`, `941100`, `941120`, `941160`, `941390`, `942100`, `942190` **for the entire
  request**, because dashboard panel/query indices vary. This body-wide SQLi exception also removes
  those rules' checks on query/header targets: it is not confined to SQL fields. SQL in a dashboard
  title was accepted in the historical test. Other rules remain active (a `<script>` title still
  triggered `941110`). Grafana must validate and authorize saved content; remove this exception if
  its cost is unsuitable. The web UI does not use this path since Grafana 13: a dashboard saved
  from the UI gets no body-wide exception, so SQL in its title, description or any other field
  except `rawSql` is blocked with `403`.
- Rule `9858103` applies only to `GET /explore`. Explore keeps the query in the `panes` parameter of
  its address, so reloading the page or opening a shared link sends the SQL text in the request
  line. The rule removes the same SQL injection family and the same five rules from that one
  parameter. Every other parameter, path and method (a `POST`, `/explore/`, another capitalisation)
  stays checked, and so do XSS and path traversal in `panes`. Cost: SQL injection is not detected
  in `panes`.
- Rule `9858500` removes response rule `953100` only for a `GET` whose response is `text/html`,
  whatever the `Accept` header says. Grafana serves one HTML shell on every front-end route and for
  unknown paths, and its boot data holds strings such as `sqlConnectionLimits` and the names of your
  data sources and dashboards, which `953100` reads as a PHP error. Without the exception a client
  that does not accept gzip gets the shell cut off, with curl exit 92 and a `500` in the access log.
  Grafana compresses the shell by default (`enable_gzip`) and ModSecurity cannot read a compressed
  body, so a browser reaches the rule only when gzip is off, while curl and most uptime monitors
  send `Accept: */*` and no `Accept-Encoding`. The request header is not part of the condition,
  because the client controls it. JSON and plain-text responses stay inspected whatever the client
  sends. A data source or dashboard named like a PHP error message can still block the JSON replies
  that list it (`/api/search`, `/api/datasources`): rename it.
- Rule `9858501` applies only to `GET /api/plugins`, the plugin list that Administration > Plugins
  and the data source pages load, with or without a query such as
  `?embedded=include-datasource&accesscontrol=true` or `?enabled=1&type=datasource`. It removes
  response rule `951220` (SQL Server error leak) for that path. The list is one long line, and
  `951220` reads the plugin id `stackdriver` and, further on, the description of the SQL Server
  plugin as one error message, so without the exception the reply is cut off and those pages show
  no plugins. Every other CRS check stays active on the path, and `951220` stays active on every
  other path: a dashboard or data source named like a SQL Server error message can still block the
  replies that list it, so rename it. The path with a trailing slash, which Grafana's pages do not
  use, receives no exception.

## Validation

On 2026-11-20 (Grafana 13.2.3, BunkerWeb 1.6.17 all-in-one, CRS 4.29.0, a PostgreSQL 16 data source
with real queries) the files of this template were imported through the web UI behind the CRS
(`POST /templates/create`: the JSON alone, each config alone and everything together, as Save sends
it: four requests, `200`, no rule fired), and the service was created in easy mode from the
template. In headless Chrome on that service:

- a new dashboard with a PostgreSQL panel whose query holds `$__timeGroupAlias(...)`,
  `$__timeFilter(...)` and `${__from:date}`, `LIKE '%h1%'`, `UNION ALL` and
  `CAST(...) IN (SELECT ...) ... HAVING` ran (`POST /api/ds/query` `200`, the graph drew rows)
  and saved (`201` on `POST /apis/dashboard.grafana.app/v2/namespaces/default/dashboards`, `200` on
  the `PUT` of the next save). Before the change the same actions answered `403` for every
  keystroke's query and for the save;
- Explore opened from an address that carries that query (`/explore?panes=...`) and ran it, where
  the address alone was `403` before rule `9858103`;
- **Connections**, **Add new connection** loaded 253 logos with no `429` and no ban; with the
  default `LIMIT_CONN_MAX_HTTP2=100` 139 of them were `429` and the IP was banned for 24 h.

A curl matrix of 52 requests (`rawSql` in `POST /api/ds/query` at index 0 and 10, in the web UI
and API dashboard saves, in `GET /explore`) matched the expected answer in every row, before the
change (the SQL rows `403`) and after it (`200`, `201`, `400` from Grafana itself for variables it
does not know). Its 20 controls stayed `403`: SQL injection in a title, a description, a query
alias, a search parameter and another field; XSS and path traversal inside a `rawSql` value and
inside `panes`; a `rawSql` key at the top of the body; a `POST` to `/explore`, `/explore/` and
`/EXPLORE`; and a parameter other than `panes` on `/explore`. Rows that pass by design: SQL
injection inside a `rawSql` or `panes` value, and a nested `rawSql` key sent to another Grafana
path. A 49-row response matrix (JSON and plain-text responses still blocked by `953100` and
`951220`, the plugin list, the HTML shell with and without gzip) and 17 rows for curl style
clients (`Accept: */*`, `application/json`, `text/plain`: complete `/login`, `/explore`, `404`
pages with rule `9858500` as it is now; cut off with curl exit 92 before) matched too, apart
from two rows whose only difference is the `Accept` header, which no longer matters.

On 2026-11-20 (Grafana 13.2.3, BunkerWeb 1.6.17 AIO, CRS 4.29.0) a 58-row curl matrix and a headless
Chrome smoke (login, home, dashboards, a dashboard with a panel, Explore, Administration > Plugins,
data sources, users, alerting, profile, preferences, dashboard save and delete, data source query)
exercised the HTML response exception and the plugin list exception: 58 of 58 rows as expected, no
`403` on any page. The smoke ran against a service created in easy mode from this template.
`/api/plugins` stays blocked without rule `9858501`, and a dashboard or data source named like a SQL
Server error message stays blocked with it.

Historical scripted tests: **2026-10-06**, **Grafana 13.2.3**, PostgreSQL **16.15** data source,
**BunkerWeb 1.6.15**, self-signed TLS. Final block-mode flows passed login/session/basic-auth API,
service account/token, PUT/PATCH/DELETE, PostgreSQL raw SQL queries, HTML/SQL/PromQL dashboards,
JSON error replies, dashboard saves of 200 KiB/1 MiB/15 MiB, Live WebSocket `101`, and a 30-asset
burst at `100r/s`. Without the HTML exception, a client that does not accept gzip gets the shell as
`200` with an incomplete body (curl exit 18), and Chrome did not finish loading `/login` with
Grafana's `enable_gzip` off.

Historical negative controls blocked other fields, paths, near paths and methods. SQLi in a
dashboard title passed on `/api/dashboards/db` deliberately, because of the broad exception above.
Not tested: alert rules and library panels, MySQL and SQL Server data sources, Loki/Prometheus data
sources, plugin installation, SSO, `USE_MODSECURITY_GLOBAL_CRS=yes` or uploads over 16 MB. PromQL
text in a saved dashboard did not prove a Prometheus data source. Revalidate the flows and
boundaries you use after import and upgrades.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
