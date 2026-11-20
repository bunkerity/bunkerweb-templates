<p align="center">
  <img alt="BunkerWeb Templates logo" src="./assets/logo.png" width="240" />
</p>

<h3 align="center">Ready-to-run BunkerWeb configurations for popular services,<br />fully editable to fit your stack.</h3>

<p align="center">
  <a href="#templates"><img src="https://img.shields.io/github/directory-file-count/bunkerity/bunkerweb-templates/templates?type=dir&label=templates&color=085577" alt="Number of templates" /></a>
  <a href="https://github.com/bunkerity/bunkerweb-templates/releases"><img src="https://img.shields.io/github/v/release/bunkerity/bunkerweb-templates?label=release&color=085577" alt="Latest release" /></a>
  <a href="https://github.com/bunkerity/bunkerweb-templates/releases"><img src="https://img.shields.io/github/downloads/bunkerity/bunkerweb-templates/total?label=downloads&color=085577" alt="Release downloads" /></a>
  <a href="https://github.com/bunkerity/bunkerweb"><img src="https://img.shields.io/badge/BunkerWeb-1.6.17-085577" alt="Tested with BunkerWeb 1.6.17" /></a>
  <a href="https://coreruleset.org/"><img src="https://img.shields.io/badge/OWASP%20CRS-4-085577" alt="OWASP CRS 4" /></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-GPLv3-blue.svg" alt="License: GPL v3" /></a>
</p>

<p align="center">
  <a href="https://github.com/bunkerity/bunkerweb-templates/stargazers"><img src="https://img.shields.io/github/stars/bunkerity/bunkerweb-templates?color=ffb400&logo=github" alt="GitHub stars" /></a>
  <a href="https://score.getplumber.io/github.com/bunkerity/bunkerweb-templates"><img src="https://score.getplumber.io/github.com/bunkerity/bunkerweb-templates.svg" alt="Plumber CI/CD security score" /></a>
  <a href="https://discord.gg/YEdMKqztMZ"><img src="https://img.shields.io/badge/community-Discord-5865F2?logo=discord&logoColor=white" alt="Join the Discord" /></a>
  <a href="CONTRIBUTING.md"><img src="https://img.shields.io/badge/PRs-welcome-brightgreen.svg" alt="PRs welcome" /></a>
</p>

Community-maintained templates for [BunkerWeb](https://www.bunkerweb.io). Each one gives a
well-known service, such as WordPress, Nextcloud or a Matrix homeserver, a working configuration
that you can extend for your environment.

## Quick start

1. Pick a template below and read its README for prerequisites.
2. In the BunkerWeb web UI, open **Templates**, select **Create new template**, switch to **Raw**
   mode and select **Upload template JSON**.
3. Select **Save**, then choose the template in **Easy** mode when you edit a service.

Every step, the upload dialog and the plugin method are in [Install](#install).

## Templates

[Identity and security](#identity-and-security) · [Files and storage](#files-and-storage) ·
[Productivity and documents](#productivity-and-documents) ·
[Websites and CMS](#websites-and-cms) ·
[Developer tools and automation](#developer-tools-and-automation) ·
[Monitoring and analytics](#monitoring-and-analytics) · [Communication](#communication) ·
[Network and infrastructure](#network-and-infrastructure) ·
[Media, search and personal apps](#media-search-and-personal-apps)

### Identity and security

| Template | Summary |
| --- | --- |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/authentik.svg" align="absmiddle" alt="" width="24" height="24" /> [authentik](templates/authentik/) | Identity provider template with flow and cookie-aware WAF tuning |
| <picture><source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/pocket-id-light.svg"><img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/pocket-id.svg" align="absmiddle" alt="" width="24" height="24" /></picture> [Pocket ID](templates/pocket-id/) | OIDC provider template with passkey and callback WAF tuning |
| <picture><source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/vaultwarden-light.svg"><img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/vaultwarden.svg" align="absmiddle" alt="" width="24" height="24" /></picture> [Vaultwarden](templates/vaultwarden/) | Password vault template with encrypted-field WAF tuning |

### Files and storage

| Template | Summary |
| --- | --- |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/filestash.svg" align="absmiddle" alt="" width="24" height="24" /> [Filestash](templates/filestash/) | File manager template with WebDAV-aware proxy defaults |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/filestash.svg" align="absmiddle" alt="" width="24" height="24" /> [Filestash Office](templates/filestash-office/) | Collabora (WOPI) host template with WebSocket proxy defaults |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/garage.svg" align="absmiddle" alt="" width="24" height="24" /> [Garage](templates/garage/) | S3 object storage template with signed-request WAF tuning |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/nextcloud.svg" align="absmiddle" alt="" width="24" height="24" /> [Nextcloud](templates/nextcloud/) | Secure template with WebDAV-aware defaults |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/projectsend.svg" align="absmiddle" alt="" width="24" height="24" /> [ProjectSend](templates/projectsend/) | File sharing template with chunked-upload-aware defaults |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/seafile.svg" align="absmiddle" alt="" width="24" height="24" /> [Seafile](templates/seafile/) | File sharing template with WebDAV-aware proxy defaults |

### Productivity and documents

| Template | Summary |
| --- | --- |
| <img src="https://raw.githubusercontent.com/docusealco/docuseal/master/public/logo.svg" align="absmiddle" alt="" width="24" height="24" /> [DocuSeal](templates/docuseal/) | Document signing template with API-aware defaults |
| <img src="https://raw.githubusercontent.com/usekaneo/kaneo/main/apps/web/public/favicon.svg" align="absmiddle" alt="" width="24" height="24" /> [Kaneo](templates/kaneo/) | Task board template with WebSocket and task-text WAF tuning |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/outline.svg" align="absmiddle" alt="" width="24" height="24" /> [Outline](templates/outline/) | Wiki template with realtime collaboration and MCP support |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/stirling-pdf.svg" align="absmiddle" alt="" width="24" height="24" /> [Stirling-PDF](templates/stirling-pdf/) | PDF toolbox template with big uploads, long jobs and tuned WAF rules |

### Websites and CMS

| Template | Summary |
| --- | --- |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/directus.svg" align="absmiddle" alt="" width="24" height="24" /> [Directus](templates/directus/) | Headless CMS template with REST and GraphQL API defaults |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/drupal.svg" align="absmiddle" alt="" width="24" height="24" /> [Drupal](templates/drupal/) | Secure template with CMS-aware defaults |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/apache-tomcat.svg" align="absmiddle" alt="" width="24" height="24" /> [Tomcat](templates/tomcat/) | Reverse proxy template with servlet-friendly defaults |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/wordpress.svg" align="absmiddle" alt="" width="24" height="24" /> [WordPress](templates/wordpress/) | Secure template with essential hardening defaults |

### Developer tools and automation

| Template | Summary |
| --- | --- |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/atuin.svg" align="absmiddle" alt="" width="24" height="24" /> [Atuin sync server](templates/atuin/) | Shell history sync server template with API client tuning |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/forgejo.svg" align="absmiddle" alt="" width="24" height="24" /> [Forgejo](templates/forgejo/) | Git forge template with HTTPS, LFS and package API tuning |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/n8n.svg" align="absmiddle" alt="" width="24" height="24" /> [n8n](templates/n8n/) | Workflow automation template with editor and webhook tuning |

### Monitoring and analytics

| Template | Summary |
| --- | --- |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/beszel.svg" align="absmiddle" alt="" width="24" height="24" /> [Beszel](templates/beszel/) | Monitoring hub template with WebSocket agent connections |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/grafana.svg" align="absmiddle" alt="" width="24" height="24" /> [Grafana](templates/grafana/) | Dashboard template with query and HTML-response WAF tuning |
| <picture><source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/umami-light.svg"><img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/umami.svg" align="absmiddle" alt="" width="24" height="24" /></picture> [Umami](templates/umami/) | Web analytics template with tracker and dashboard tuning |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/uptime-kuma.svg" align="absmiddle" alt="" width="24" height="24" /> [Uptime Kuma](templates/uptime-kuma/) | Monitoring template with Socket.IO polling and WebSockets |

### Communication

| Template | Summary |
| --- | --- |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/ntfy.svg" align="absmiddle" alt="" width="24" height="24" /> [ntfy](templates/ntfy/) | Push notification template with publish and subscribe tuning |
| <picture><source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/synapse-light.svg"><img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/synapse.svg" align="absmiddle" alt="" width="24" height="24" /></picture> [Synapse](templates/synapse/) | Matrix homeserver template with well-known delegation |
| <picture><source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/matrix-light.svg"><img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/matrix.svg" align="absmiddle" alt="" width="24" height="24" /></picture> [Tuwunel](templates/tuwunel/) | Matrix homeserver reverse proxy with WebSocket and delegation |

### Network and infrastructure

| Template | Summary |
| --- | --- |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/netbird.svg" align="absmiddle" alt="" width="24" height="24" /> [NetBird](templates/netbird/) | Self-hosted template with gRPC and websocket routing |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/pi-hole.svg" align="absmiddle" alt="" width="24" height="24" /> [Pi-hole](templates/pi-hole/) | Reverse proxy template with admin UI and API tuning |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/xen-orchestra.svg" align="absmiddle" alt="" width="24" height="24" /> [Xen Orchestra](templates/xen-orchestra/) | Reverse proxy template with WebSocket and rate limiting defaults |

### Media, search and personal apps

| Template | Summary |
| --- | --- |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/jellyfin.svg" align="absmiddle" alt="" width="24" height="24" /> [Jellyfin](templates/jellyfin/) | Media streaming template with reverse proxy tuning |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/opengym.svg" align="absmiddle" alt="" width="24" height="24" /> [openGym](templates/opengym/) | Passkey PWA template with JSON sync and WebAuthn tuning |
| <picture><source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/open-webui-light.svg"><img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/open-webui.svg" align="absmiddle" alt="" width="24" height="24" /></picture> [Open WebUI](templates/open-webui/) | AI chat template with WebSocket and prompt-aware WAF tuning |
| <img src="https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/searxng.svg" align="absmiddle" alt="" width="24" height="24" /> [SearXNG](templates/searxng/) | Metasearch template with query-aware WAF tuning |

## Install

The **web UI is recommended** for most users. Use the plugin method when you already manage
BunkerWeb plugins as files or need repeatable, automated deployments.

First, download and extract the
[latest release archive](https://github.com/bunkerity/bunkerweb-templates/releases/latest), or
clone this repository. Choose the service directory you want to install and read its README for
prerequisites and service-specific settings. Below, `<template-directory>` means that extracted
service directory, such as `wordpress/` in the release archive or `templates/wordpress/` in a
repository clone.

> [!NOTE]
> The templates are written and tested on OWASP CRS 4, the default of BunkerWeb 1.6.17. Their
> ModSecurity exclusions only take effect with `USE_MODSECURITY_GLOBAL_CRS=no`, which is the
> default: with `yes`, BunkerWeb 1.6.17 skips the per-service `modsec-crs` configs.

<details open>
<summary><strong>Web UI (recommended)</strong></summary>

<br />

1. Sign in to the BunkerWeb web UI with an account that can edit the configuration.
2. Open **Templates**, select **Create new template**, and switch to **Raw** mode.
3. Select **Upload template JSON** and choose `<template-directory>/template.json`. You can instead
   paste the file's contents into the JSON editor. Confirm **Replace template** if prompted.
4. If the **Missing Custom Configs** dialog appears, the template needs its accompanying NGINX
   configuration files:
   - Select **Browse Files**, or drag files into the upload area.
   - Upload every listed file from `<template-directory>/configs/` and its subdirectories.
     Filenames must match the list; selecting the directories themselves is not required.
   - Wait until every item is marked **Uploaded**, then select **Done**.
5. Review the imported JSON and configs, then select **Save**. Templates without custom configs
   skip the previous dialog.
6. Add or edit a service in **Easy** mode, choose the new template, and review each guided step.
   Replace example domains, upstream URLs, and other defaults with values for your deployment
   before saving the service.

The web UI stores the imported template and config contents in BunkerWeb's database. The local
files are no longer needed after a successful import. For non-UI configuration, apply the same
template with `USE_TEMPLATE=<name>`, where `<name>` is the template ID shown in the UI.

</details>

<details>
<summary><strong>Plugin (managed deployments)</strong></summary>

<br />

> [!WARNING]
> Template directories from this repository are **not standalone plugins**. Add the selected
> template to an existing external plugin, or create a plugin with a `plugin.json` file by
> following the [BunkerWeb plugin documentation](https://docs.bunkerweb.io/latest/plugins/#how-to-use-a-plugin).

BunkerWeb uses the JSON filename as the template ID, so the repository layout must be rearranged
inside the plugin:

1. Copy `<template-directory>/template.json` to `<plugin>/templates/<name>.json`.
2. If the template contains a `configs/` directory, copy it to
   `<plugin>/templates/<name>/configs/`. Preserve every config-type subdirectory, such as
   `modsec-crs/` or `server-http/`.
3. Confirm the resulting plugin layout. WordPress, for example, must look like this:

   ```text
   plugins/
   └── example-plugin/
       ├── plugin.json
       └── templates/
           ├── wordpress.json
           └── wordpress/
               └── configs/
                   └── modsec-crs/
                       ├── wordpress_crs_tuning.conf
                       └── wordpress_false_positives.conf
   ```

4. Install or reload the plugin using the procedure for your BunkerWeb integration. The scheduler
   loads the template from the plugin.
5. Select the template when adding or editing a service in easy mode, or set `USE_TEMPLATE=<name>`.
   For the example above, use `USE_TEMPLATE=wordpress` because the file is named `wordpress.json`.

</details>

<details>
<summary><strong>Updating authentik, forgejo, grafana, netbird, opengym, vaultwarden or WordPress</strong></summary>

<br />

These seven templates moved their exclusion file from `configs/modsec/` to `configs/modsec-crs/`
and gave its rules new ids. Update them so that only the new file stays active; an old file can
be wider than the new one (the old WordPress rule matched every path that starts with
`/wp-admin/admin-ajax.php`, the new one matches that path only). Tested with WordPress on
`bunkerity/bunkerweb-all-in-one:1.6.17`, in a browser for the web UI; the other six templates make
the same change. Other integrations were not tested.

- **Web UI:** uploading the new files over the installed template is not enough. The editor keeps
  the old `modsec` config next to the new ones, and **Done** and **Save** both stop with
  `Configs ... are not assigned to any step`. Remove the old config first:
  1. Open the installed template under **Templates**. Do not create it again, BunkerWeb answers
     that the template already exists. In **Easy** mode, find the old config under
     **Custom configs**: type `MODSEC`, name `<template>_false_positives` (for example
     `wordpress_false_positives`). Select its red trash button (**Remove**).
  2. Switch to **Raw**, select **Upload template JSON**, choose the new `template.json` and
     confirm **Replace template**.
  3. In the **Missing Custom Configs** dialog, upload every listed file from the new
     `configs/modsec-crs/` directory, then select **Done**. The editor reports
     `All missing configs have been uploaded!`.
  4. Select **Save**. The old file is gone after the next reload (about 5 seconds, no restart).
     Do not try to delete it on the **Configs** page: BunkerWeb refuses to delete template configs
     there.
- **Plugin:** replace the plugin directory with the new files and restart the BunkerWeb container.
  Do not remove the plugin first: BunkerWeb then keeps the rendered files as custom configs of
  the service, which stay active and replace the same-named file of the new template, and the
  service loses the template settings until the plugin is back.

**Check the result (web UI and plugin).** Replace `<container>` with the BunkerWeb container,
`<service>` with the service's server name, and `<name>` and `<id>` with a row of the table below.
Right before you select **Save** or restart the container, note the time (UTC, with the `Z`; GNU
`date` on Linux):

```sh
S=$(date -u +%Y-%m-%dT%H:%M:%S.%NZ)
```

Then run this. It looks every 2 seconds, for up to 60 seconds. After the last
`Generating new custom configs` line since `S`, it waits for the push of `/etc/nginx` and then
`reload successful`, and only then reads the rendered file. It prints `new rules loaded` if the file
holds the new rules. It stops with `apply failed` if `failing over` or `was not applied` follows
that line:

```sh
for i in $(seq 30); do
  r=$(docker logs --since "$S" <container> 2>&1 \
    | grep -E 'Generating new custom configs|Sending /etc/nginx|reload successful|failing over|was not applied' \
    | awk '/Generating/{s=1;f=0} /failing over|was not applied/{f=1} /Sending/&&s==1{s=2} /reload successful/&&s==2{s=3} END{if(f)print "failed";else if(s==3)print "ok"}')
  [ "$r" = failed ] && { echo "apply failed, see docker logs <container>"; break; }
  [ "$r" = ok ] && docker exec <container> grep -q 'id:<id>' /data/configs/modsec-crs/<service>/<name>.conf 2>/dev/null \
    && { echo "new rules loaded"; break; }
  sleep 2
done
```

In testing the check ended 6 to 9 seconds after `docker restart` and about 5 seconds after
**Save**, and the first request after it was served normally. For the seven
`<template>_false_positives` rows, also run this once. It prints a line when the old file is gone:

```sh
docker exec <container> sh -c \
  'test ! -e /data/configs/modsec/<service>/<name>.conf && echo "old file gone"'
```

| Template                 | `<name>`                      | `<id>`    |
| ------------------------ | ----------------------------- | --------- |
| authentik                | `authentik_false_positives`   | `9851100` |
| forgejo                  | `forgejo_false_positives`     | `9856103` |
| forgejo (second file)    | `forgejo_protocol`            | `9856100` |
| grafana                  | `grafana_false_positives`     | `9858103` |
| netbird                  | `netbird_false_positives`     | `9862100` |
| opengym                  | `opengym_false_positives`     | `9866100` |
| vaultwarden              | `vaultwarden_false_positives` | `9877100` |
| WordPress                | `wordpress_false_positives`   | `9878101` |
| WordPress (second file)  | `wordpress_crs_tuning`        | `9878010` |

The two second-file rows are files that kept their name and got new ids; they have no old file in
`modsec/`, so skip the second command for them.

If the first command prints `apply failed`, BunkerWeb did not apply the new configuration.
Depending on where it failed, it keeps the previous one and retries, or tries to go back to its
last working configuration. What happened and why are in `docker logs <container>` (for an nginx
error, an `[emerg]` line names the file). Fix the cause, then select **Save** or restart again.

If the first command prints nothing after 60 seconds:

- with the plugin, look at the plugin directory. Only if it holds the old files again (the old
  plugin copy came back over the new one in 5 of 8 test restarts made right after a first start),
  put the new files back and restart. If it already holds the new files, do not restart again;
- otherwise the file may be stuck as a stored custom config. This happens after the plugin was
  removed once: BunkerWeb 1.6.17 then keeps the rendered files as custom configs with method
  `manual`. They replace the same-named file of the new template, so the old, wider exclusion stays
  active and the new file is not loaded. The **Configs** page cannot remove them: it answers
  `Custom config <service>/modsec_crs/<name> is not a UI/API custom config and will not be
  deleted.` BunkerWeb 1.6.17 has no safe supported way to remove them, so this README gives no
  manual steps: removing them by hand can delete other custom configs. A timeout can also mean a
  missing file or a failed `docker exec`. This read-only command prints the stored row, if there
  is one (all-in-one image, SQLite; other integrations were not tested):

  ```sh
  docker exec <container> sqlite3 -readonly /var/lib/bunkerweb/db.sqlite3 \
    "select service_id,type,name,method from bw_custom_configs where service_id='<service>' and type='modsec_crs' and name='<name>' and method='manual'"
  ```

  A line `<service>|modsec_crs|<name>|manual` means a stored `manual` config replaces the
  template's file. It is stale if its file does not hold the new rules; this prints `0` then:

  ```sh
  docker exec <container> grep -c 'id:<id>' /data/configs/modsec-crs/<service>/<name>.conf
  ```

</details>

## More

<details>
<summary><strong>How templates work</strong></summary>

<br />

BunkerWeb includes three predefined templates (`low`, `medium`, `high`) that bundle common
security settings. The easy-mode UI and the `USE_TEMPLATE` setting can also consume **custom
templates** defined as JSON. A custom template can declare:

- **Settings**: multisite settings whose values override defaults.
- **Configs**: paths to reusable NGINX configuration snippets that get merged into your service.
- **Steps**: optional guided steps that combine settings and configs, so users can adjust the
  template interactively in the UI.

This repository packages those JSON definitions alongside their configuration snippets, so you
can drop them into BunkerWeb for any deployment method. The
[BunkerWeb documentation](https://docs.bunkerweb.io/latest/concepts/#templates) covers the concept.

</details>

<details>
<summary><strong>Template anatomy</strong></summary>

<br />

Every template keeps its definition and supporting assets together:

```text
templates/<service-name>/
├── template.json      # required: id, name, settings, configs, steps
├── README.md          # prerequisites and service-specific notes
└── configs/           # optional NGINX and ModSecurity snippets
    ├── modsec-crs/    # rules loaded before the CRS (most exclusions)
    ├── modsec/        # rules loaded after the CRS
    └── server-http/   # directives for the service's server block
```

- `template.json` describes the template `id`, user-facing `name`, and optional `settings`,
  `configs`, and `steps`.
- `configs/` contains any NGINX fragments referenced in the JSON file.
- Additional assets, such as helper scripts or documentation, can live beside these defaults if they
  help users.

A minimal `template.json`:

```json
{
  "id": "wordpress",
  "name": "WordPress",
  "settings": {
    "SERVER_NAME": "www.example.com",
    "REVERSE_PROXY_HOST": "http://mywp",
    "MAX_CLIENT_SIZE": "50m",
    "MODSECURITY_CRS_PLUGINS": "wordpress-rule-exclusions"
  },
  "configs": ["modsec-crs/wordpress_false_positives.conf"],
  "steps": [
    {
      "title": "Domains and TLS",
      "subtitle": "Point the template at your WordPress hostname(s).",
      "settings": ["SERVER_NAME"]
    },
    {
      "title": "Reverse proxy",
      "subtitle": "Tell BunkerWeb where your WordPress upstream lives.",
      "settings": [
        "REVERSE_PROXY_HOST",
        "MAX_CLIENT_SIZE",
        "MODSECURITY_CRS_PLUGINS"
      ],
      "configs": ["modsec-crs/wordpress_false_positives.conf"]
    }
  ]
}
```

[`TEMPLATE_GUIDE.md`](TEMPLATE_GUIDE.md) holds the full authoring rules.

</details>

<details>
<summary><strong>Contributing</strong></summary>

<br />

Review [CONTRIBUTING.md](CONTRIBUTING.md) for the full workflow. In short:

1. Fork the repository and create a feature branch.
2. Add or update your template following the guidelines.
3. Submit a pull request describing your changes and any testing performed.

Before you push, install the repository's pre-commit hooks with `pre-commit install` and run
`pre-commit run --all-files`, so formatting, spell-check, and secret scans pass locally.

</details>

<details>
<summary><strong>Community and maintainers</strong></summary>

<br />

- Discord: [discord.gg/YEdMKqztMZ](https://discord.gg/YEdMKqztMZ)
- GitHub Issues: report bugs or request new templates in the
  [issue tracker](https://github.com/bunkerity/bunkerweb-templates/issues).
- Wiki: browse the [project wiki](https://github.com/bunkerity/bunkerweb-templates/wiki) for
  navigation and troubleshooting.
- Code of Conduct: read the [community guidelines](CODE_OF_CONDUCT.md) before engaging.

| Name           | GitHub                                             |
| -------------- | -------------------------------------------------- |
| Théophile Diot | [@TheophileDiot](https://github.com/TheophileDiot) |
| Alexandre      | [@YouKyi](https://github.com/YouKyi)               |

</details>

## License

[GNU General Public License v3.0](LICENSE), in line with the main BunkerWeb project. The license
covers this repository, including all templates and documentation. By contributing, you agree that
your contributions will be released under the same license.

Product names and logos are trademarks of their owners.
