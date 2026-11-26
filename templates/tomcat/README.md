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
  <a href="#prepare-tomcat">Prepare Tomcat</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a> ·
  <a href="#files">Files</a> ·
  <a href="#validation">Validation</a>
</p>

Generate a servlet-friendly reverse proxy configuration for Apache Tomcat that
handles TLS automation, upstream routing, compression, caching, and HTTP method
allow lists tuned for typical Java web applications.

## At a glance

| Fact        | Value                                         |
| ----------- | --------------------------------------------- |
| Template id | `tomcat`                                      |
| Tested with | BunkerWeb 1.6.17, OWASP CRS 4, Tomcat 10.1.62 |
| Configs     | 1 (modsec-crs)                                |
| Rule ids    | `9873000` to `9873999`                        |
| Upstream    | `http://mytomcat:8080` (placeholder)          |

`mytomcat` is a placeholder: replace it with the name or address of your own Tomcat container.

## Prerequisites

- A Tomcat backend reachable from BunkerWeb (container hostname, service, or static IP). The
  template was tested with Tomcat 10.1.62 (`tomcat:10.1`).
- The BunkerWeb web UI. The steps below use easy mode.
- Two changes inside Tomcat, described in [Prepare Tomcat](#prepare-tomcat): make it trust the
  proxy headers, and, if you deploy WARs through the Manager, let the Manager answer your address.

## Prepare Tomcat

### Proxy headers

BunkerWeb sends `X-Forwarded-For`, `X-Forwarded-Proto` and `X-Forwarded-Host`. A stock Tomcat
ignores them, so it believes every request is plain HTTP on port 80 from BunkerWeb's own address.
Applications that build absolute URLs (OAuth callbacks, redirects, `Secure` logic) break, and the
client address in Tomcat's logs is the proxy. The `RemoteIpValve` fixes that. Add it inside the
`<Host>` element of `conf/server.xml`, before the `AccessLogValve`:

```xml
<Valve className="org.apache.catalina.valves.RemoteIpValve"
       internalProxies="172\.30\.73\.10" />
```

`internalProxies` is a regular expression for BunkerWeb's address as Tomcat sees it. Replace
`172\.30\.73\.10` with yours (each dot written `\.`):

- `docker inspect <bunkerweb container> --format '{{(index .NetworkSettings.Networks "<network>").IPAddress}}'`
  prints it. Docker can give a recreated container another address, so give BunkerWeb a fixed one
  in your Compose file, or update the valve after you recreate it.
- Or read the first column of Tomcat's access log (`logs/localhost_access_log.*.txt`) before you
  add the valve: it shows the address of the proxy.

Measured on Tomcat 10.1.62 with a page that prints the request (client address `172.30.73.1`):

| Tomcat setup                                              | Tomcat saw                                                         |
| --------------------------------------------------------- | ------------------------------------------------------------------ |
| Stock `server.xml`                                        | `scheme=http secure=false port=80 remote=172.30.73.10` (BunkerWeb) |
| Valve with `internalProxies` set to BunkerWeb's address   | `scheme=https secure=true port=443 remote=172.30.73.1` (client)    |
| Valve with an `internalProxies` that does not match       | the same as the stock setup: the valve does nothing                |

Why not leave `internalProxies` unset: the default trusts every private address range. A client on
a private network (a LAN, a VPN, the Docker bridge) is then also "internal" and can send its own
`X-Forwarded-For` to choose the address Tomcat sees (measured: a forged `6.6.6.6` became
`remote=6.6.6.6`). With only BunkerWeb's address in `internalProxies`, the same forged header was
ignored and Tomcat still saw the real client address.

Two details. Tomcat reports port `443` for an HTTPS request when no port header is present, and
BunkerWeb sends none: for a public HTTPS port other than 443, add `httpsServerPort="<port>"` to the
valve. To get the real client address in the access log, also set `requestAttributesEnabled="true"`
on the `AccessLogValve` placed after the `RemoteIpValve` (measured: the log showed `172.30.73.1`
instead of BunkerWeb's address).

### Manager access (WAR deploys)

Tomcat's Manager application answers loopback only. Behind BunkerWeb it sees the proxy's address
(or, with the valve above, your client address), so a request for `/manager/text/list` ends in
Tomcat's own `403`. BunkerWeb replaces Tomcat's error page with its own, so the page looks like a
WAF block, but BunkerWeb's log shows no ModSecurity rule and Tomcat's access log shows the `403`.

The restriction is what protects your server: anyone who can reach the Manager with a valid user
can deploy a WAR, which means running code. Open it to trusted addresses only.

1. Create a user for deploys in `conf/tomcat-users.xml` (`manager-script` is the role of the text
   interface; add `manager-gui` for a user of the Manager web page):

   ```xml
   <role rolename="manager-script"/>
   <user username="deployer" password="a-long-random-password" roles="manager-script"/>
   ```

2. Open `webapps/manager/META-INF/context.xml`. The official `tomcat` image ships the Manager in
   `webapps.dist/` and leaves `webapps/` empty: copy `webapps.dist/manager` to `webapps/manager`
   first. Tomcat 10.1 ships this valve, which allows loopback only:

   ```xml
   <Valve className="org.apache.catalina.valves.RemoteCIDRValve" allow="127.0.0.0/8,::1/128" />
   ```

   Add the address of the machine you deploy from. With the `RemoteIpValve` from the previous
   section, Tomcat sees your **client** address, so allow that:

   ```xml
   <Valve className="org.apache.catalina.valves.RemoteCIDRValve" allow="127.0.0.0/8,::1/128,203.0.113.7/32" />
   ```

   Older Tomcat releases use the regular expression form
   `<Valve className="org.apache.catalina.valves.RemoteAddrValve" allow="127\.\d+\.\d+\.\d+|::1|203\.0\.113\.7" />`,
   which also works.

   > [!WARNING]
   > Without the `RemoteIpValve`, Tomcat sees BunkerWeb's address for every request. Allowing that
   > address then opens the Manager to everyone who can reach the site: every request looks like it
   > comes from an allowed address (measured: `200`, also with a forged `X-Forwarded-For`). Add the
   > `RemoteIpValve` first, and allow your own address.

3. Restart Tomcat, then deploy through BunkerWeb:

   ```bash
   curl -u deployer:<password> -T app.war -H 'Content-Type: application/octet-stream' \
     'https://tomcat.example.com/manager/text/deploy?path=/app&update=true'
   ```

   The answer is `OK - Deployed application at context path [/app]`. The template's CRS exception
   (see [Settings](#settings)) is what lets this request through.

Large WARs: BunkerWeb's ModSecurity holds the request body in memory, so a big upload needs a lot of
RAM on the BunkerWeb container. A 60 MB WAR sent through `PUT /manager/text/deploy` to a BunkerWeb
container limited to 1.4 GB made it use all of that, and the kernel killed an nginx worker (the
client saw a dropped connection). The limit is memory, not `MAX_CLIENT_SIZE`. For a large WAR, copy it
to Tomcat's `webapps/` folder (Tomcat deploys it on its own), or copy it to a folder on the Tomcat
host and deploy it by path: `GET /manager/text/deploy?path=/app&war=file:/srv/wars/app.war`
(measured: `OK - Deployed`). Do not keep the file under `/tmp/`: the CRS blocks that path in the
query string (rule `930120`).

## Install

1. **Import the template**
   - Follow the repository's [installation guide](../../README.md#install) for the web UI or
     plugin bundle method.
2. **Create the service from the template**: in the web UI open **Services**, **Create new
   service**, and pick **Tomcat** in the **Templates** list above the steps.
3. **Review step 1 (front service)**: set `SERVER_NAME` to your public host name. For a name that
   is not public, turn **Automatic Let's Encrypt** off: BunkerWeb cannot get a certificate for it,
   and clients are redirected to HTTPS with a certificate that does not match. BunkerWeb 1.6.17
   ignores that switch when you create the service, so save, open the service again and turn it
   off a second time. For your own certificate, use the certificate fields of the same step.
4. **Point the reverse proxy** (step 2) at your Tomcat container. The upstream shown, `http://mytomcat:8080`,
   is a placeholder: replace `mytomcat` with the name or address of your Tomcat container and
   confirm DNS or networking between the containers.
5. **Tune HTTP settings** (step 4): adjust `ALLOWED_METHODS` for your application.
6. **Save** the service. Saving applies it, there is no separate reload step. Then check that you
   can reach your Tomcat application over HTTPS through the proxy.

## Settings

- `MAX_CLIENT_SIZE` (100m by default) rejects bigger bodies with `413`. Raise it for large
  uploads handled by your application. It is not the answer for big WARs: see
  [Large WARs](#manager-access-war-deploys) above.
- `REVERSE_PROXY_WS=yes` forwards WebSocket upgrades to the servlets. Set it to `no` if your
  applications do not use WebSockets.
- The Manager text interface deploys a WAR with `PUT /manager/text/deploy` and an
  `application/octet-stream` body, which CRS rule `920420` blocks. `modsec-crs/tomcat_false_positives.conf`
  removes that rule for this method and path only. A `PUT` of the same content type to another
  Manager command (`/manager/text/other`) still answers `403`. Keep the Manager restricted to
  trusted addresses, as described in [Manager access](#manager-access-war-deploys): that
  restriction is what protects WAR deploys. To turn the exception off, open the service, go to
  step 4 (**HTTP - Servlet tuning**), delete the rule in the `tomcat_false_positives` editor under
  the fields, and save (measured: the deploy then answers `403`, rule `920420`).
- `ALLOWED_METHODS` is `GET|POST|HEAD|PUT|DELETE|OPTIONS`. `PATCH` is not in the list: BunkerWeb
  answers `405` ("method PATCH is not allowed" in its log), so a REST application that uses `PATCH`
  needs it added. Keep `PUT`: the Manager deploy is a `PUT`, and narrowing the list to
  `GET|POST|HEAD` made it answer `405` (measured). Narrow the list only when you know the servlet
  endpoints accept fewer verbs.
- Leave `SERVE_FILES` off (step 3): the reverse proxy sends every path to Tomcat, which serves its
  own static files.
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
- Print the request in a page (scheme, `isSecure()`, port, `getRemoteAddr()`) to check the
  `RemoteIpValve`: through BunkerWeb it should say `https`, `true`, your client address.

---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
