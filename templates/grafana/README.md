# Grafana Template

Proxy Grafana at a dedicated hostname with Grafana Live, JSON API errors and SQL/dashboard CRS
exceptions from a historically tested recipe. Review the dashboard exception before importing:
it accepts some SQLi strings throughout dashboard-save requests.

## Prerequisites and import

1. Run Grafana on a network reachable by BunkerWeb at `http://grafana:3000` or change the upstream.
   Set `GF_SERVER_ROOT_URL=https://grafana.example.com/`, including any public nonstandard port.
   With TLS terminated by BunkerWeb, keep Grafana's upstream protocol HTTP. See the official
   [proxy guide](https://grafana.com/tutorials/run-grafana-behind-a-proxy/) and
   [`root_url` reference](https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/#root_url).
2. Follow the repository [installation guide](../../README.md#installing-templates). Raw import
   requires `template.json` and `configs/modsec/grafana_false_positives.conf`; plugin bundles use
   the layout in that guide.
3. Select `grafana` for the service (or set `USE_TEMPLATE=grafana`), change `SERVER_NAME` and
   **all three** upstream settings: `REVERSE_PROXY_HOST`, `_1` and `_2`. Review the guided steps.
4. Reload and validate your dashboards. Automatic Let's Encrypt requires DNS and HTTP-01
   reachability. For another certificate source, set `AUTO_LETS_ENCRYPT=no` and configure
   `USE_CUSTOM_SSL` and its certificate settings; local tests can use `GENERATE_SELF_SIGNED_SSL`.

This recipe serves `/`; subpath hosting needs separate configuration and validation. Keep Grafana's
own authentication and dashboard/data-source permissions configured.

## Routing and limits

- `REVERSE_PROXY_URL_1=/api/live/` and `REVERSE_PROXY_WS_1=yes` enable Grafana Live upgrades on
  that prefix only. The main proxy location keeps WebSocket support at its default `no`.
- `REVERSE_PROXY_INTERCEPT_ERRORS=no` preserves Grafana's JSON API errors, including its own
  `401` and `413` responses, instead of replacing their bodies with BunkerWeb HTML.
- `ALLOWED_METHODS=GET|POST|HEAD|OPTIONS|PUT|PATCH|DELETE` allows API updates/deletes and drops
  the default `QUERY`; append it if required.
- `USE_LIMIT_REQ=yes`, `LIMIT_REQ_URL=/`, `LIMIT_REQ_RATE=100r/s` support page/query bursts.
  This weakens throttling compared with the default `2r/s`, while retaining a finite limit.
- `MODSECURITY_REQ_BODY_NO_FILES_LIMIT=32m` raises non-file body inspection capacity **for the
  whole service**, increasing potential memory use. `/api/dashboards/` has a `64m` location
  body limit via `REVERSE_PROXY_MAX_CLIENT_SIZE_2`; other paths retain the service default
  `MAX_CLIENT_SIZE` of `10m`. The 32 MiB non-file limit still applies on the dashboard location.
  Do not interpret `64m` as a usable dashboard size: historical Grafana 13.2.3 itself rejected
  dashboards over 16 MB. A 15 MiB save passed; behavior on other versions needs checking.
- Default bad-behavior protection remains enabled. Repeated API `400`, `401` or `404` replies
  can contribute to client bans; inspect logs during validation.

## CRS exception scope and cost

All rules live in `configs/modsec/grafana_false_positives.conf`, at service scope:

- Rule `77400` removes response rule `953100` for **any GET** whose `Accept` contains the
  literal `text/html`. This prevents plugin names such as `SQLConnect` from truncating HTML
  pages. It also removes that PHP-leak check on every matching GET response; there is no path
  restriction. A JSON-accepting page request receives no exception.
- Rule `77401` applies only to `POST /api/ds/query`. Rules `942100`, `942151`, `942160`,
  `942190`, `942280`, `942320`, `942350` skip only the literal
  `ARGS:json.queries.array_0.rawSql` through `ARGS:json.queries.array_9.rawSql` targets.
  Index **10 and above remains inspected**, as do other fields and endpoints. Split larger query
  batches or review additional literal targets; a regex target did not work in the source tests.
  Query parameters with those exact target names also match on an allowed POST.
- Rule `77402` applies only to `POST /api/dashboards/db`. It removes `932340`, `941100`,
  `941120`, `941160`, `941390`, `942100`, `942190` **for the entire request**, because dashboard
  panel/query indices vary. This body-wide SQLi exception also removes those rules' checks on
  query/header targets: it is not confined to SQL fields. SQL in a dashboard title was accepted
  in the historical test. Other rules remain active (a `<script>` title still triggered `941110`).
  Grafana must validate and authorize saved content; remove this exception if its cost is unsuitable.

## Validation and historical evidence

On 2026-11-03, the packaged template was imported through a local test plugin into BunkerWeb
`:tests` (1.6.18-rc2, libmodsecurity 3.0.17, CRS 4.30.0). Applied defaults matched the JSON and
nginx/ModSecurity loading passed. Echo-backend SQL-query and dashboard-save probes passed,
while another field or path stayed blocked. Literal index coverage was checked statically.
The HTML response exception, Grafana itself and browser behavior were not freshly exercised.

Historical scripted tests: **2026-10-06**, **Grafana 13.2.3**, PostgreSQL **16.15** data source,
**BunkerWeb 1.6.15**, self-signed TLS. Final block-mode flows passed login/session/basic-auth API,
service account/token, PUT/PATCH/DELETE, PostgreSQL raw SQL queries, HTML/SQL/PromQL dashboards,
JSON error replies, dashboard saves of 200 KiB/1 MiB/15 MiB, Live WebSocket `101`, and a 30-asset
burst at `100r/s`. Without the HTML exception, a page returned `200` but was truncated near 40 KB.

Historical negative controls blocked other fields, paths, near paths, methods and query index 10.
SQLi in a dashboard title passed deliberately because of the broad exception above. Not tested:
alerting, Loki/Prometheus data sources, plugin installation, SSO, a real browser or uploads over
16 MB. PromQL text in a saved dashboard did not prove a Prometheus data source. Revalidate the
flows and boundaries you use after import and upgrades.
