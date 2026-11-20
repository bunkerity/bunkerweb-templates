<p align="center">
  <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/apache-tomcat.svg" alt="Apache Tomcat logo" width="96" height="96" />
</p>

<h1 align="center">Tomcat</h1>

<p align="center">Reverse proxy template with servlet-friendly defaults</p>

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

Generate a servlet-friendly reverse proxy configuration for Apache Tomcat that
handles TLS automation, upstream routing, compression, caching, and HTTP method
allow lists tuned for typical Java web applications.

## At a glance

| Fact        | Value                         |
| ----------- | ----------------------------- |
| Template id | `tomcat`                      |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4 |
| Configs     | 1 (modsec-crs)                |
| Rule ids    | `9873000` to `9873999`        |
| Upstream    | `http://mytomcat:8080`        |

## Prerequisites

- A Tomcat backend reachable from BunkerWeb (container hostname, service, or
  static IP).
- Access to the BunkerWeb UI or the ability to set environment variables for
  template settings.
- Ensure Tomcat recognises the proxy headers (`proxyName`, `proxyPort`, and
  `scheme`) if you terminate TLS at BunkerWeb.

## Install

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#install)
     for the web UI or plugin bundle method.
2. **Assign the template** to the service that fronts Tomcat (`USE_TEMPLATE=tomcat`
   or pick it from the UI list).
3. **Review TLS options** so `SERVER_NAME` and certificate automation match your
   public hostname.
4. **Point the reverse proxy** at the Tomcat service (for example
   `http://tomcat:8080`) and confirm DNS or networking between containers.
5. **Tune HTTP settings**: adjust `ALLOWED_METHODS` for your application.
6. **Reload BunkerWeb** and verify you can reach your Tomcat application over
   HTTPS through the proxy.

## Settings

- Increase `MAX_CLIENT_SIZE` if your application handles large uploads (such as
  multi-gigabyte WAR deployments).
- `REVERSE_PROXY_WS=yes` forwards WebSocket upgrades to the servlets. Set it to `no` if your
  applications do not use WebSockets.
- The Manager text interface deploys a WAR with `PUT /manager/text/deploy` and an
  `application/octet-stream` body, which CRS rule `920420` blocks. `modsec-crs/tomcat_false_positives.conf`
  removes that rule for this method and path only. Keep Tomcat's `RemoteAddrValve` on the
  Manager application so that only trusted addresses can deploy.
- Narrow `ALLOWED_METHODS` when you know the servlet endpoints only accept a
  subset of verbs.
- Disable `USE_CLIENT_CACHE` for highly dynamic applications that must bypass
  browser caches.
- Keep `USE_GZIP=yes` for HTML, CSS, and JSON responses; disable it only if the
  upstream handles compression itself.

## Files

- `template.json` – Template definition that guides TLS, reverse proxy, and
  servlet tuning configuration.
- `configs/modsec-crs/tomcat_false_positives.conf` – ModSecurity CRS exclusion for the
  Manager WAR deployment.

## Validation

- Run `jq . template.json` to check for JSON syntax errors before importing.
- Exercise a login flow and any file upload endpoints through BunkerWeb to
  ensure body size limits and method restrictions align with your application.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
