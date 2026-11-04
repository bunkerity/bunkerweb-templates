<p align="center">
  <img alt="BunkerWeb Templates logo" src="./assets/logo.png" width="240" />
</p>

<h3 align="center">Ready-to-run BunkerWeb configurations for popular services,<br />fully editable to fit your stack.</h3>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-GPLv3-blue.svg" alt="License: GPL v3" /></a>
  <a href="https://github.com/bunkerity/bunkerweb-templates/stargazers"><img src="https://img.shields.io/github/stars/bunkerity/bunkerweb-templates?color=ffb400&logo=github" alt="GitHub stars" /></a>
  <a href="https://github.com/bunkerity/bunkerweb-templates/issues"><img src="https://img.shields.io/github/issues/bunkerity/bunkerweb-templates?label=issues" alt="Open issues" /></a>
  <a href="https://github.com/bunkerity/bunkerweb-templates/pulls"><img src="https://img.shields.io/github/issues-pr/bunkerity/bunkerweb-templates?label=PRs" alt="Open pull requests" /></a>
  <a href="https://github.com/bunkerity/bunkerweb-templates/commits"><img src="https://img.shields.io/github/last-commit/bunkerity/bunkerweb-templates?label=last%20update" alt="Last update" /></a>
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

| Template                                        | Summary                                                          |
| ----------------------------------------------- | ---------------------------------------------------------------- |
| [authentik](templates/authentik/)               | Identity provider template with flow and cookie-aware WAF tuning |
| [Directus](templates/directus/)                 | Headless CMS template with REST and GraphQL API defaults         |
| [DocuSeal](templates/docuseal/)                 | Document signing template with API-aware defaults                |
| [Drupal](templates/drupal/)                     | Secure template with CMS-aware defaults                          |
| [Filestash](templates/filestash/)               | File manager template with WebDAV-aware proxy defaults           |
| [Filestash Office](templates/filestash-office/) | Collabora (WOPI) host template with WebSocket proxy defaults     |
| [Forgejo](templates/forgejo/)                   | Git forge template with HTTPS, LFS and package API tuning        |
| [Grafana](templates/grafana/)                   | Dashboard template with query and HTML-response WAF tuning       |
| [Jellyfin](templates/jellyfin/)                 | Media streaming template with reverse proxy tuning               |
| [NetBird](templates/netbird/)                   | Self-hosted template with gRPC and websocket routing             |
| [Nextcloud](templates/nextcloud/)               | Secure template with WebDAV-aware defaults                       |
| [openGym](templates/opengym/)                   | Passkey PWA template with JSON sync and WebAuthn tuning          |
| [Pi-hole](templates/pi-hole/)                   | Reverse proxy template with admin UI and API tuning              |
| [ProjectSend](templates/projectsend/)           | File sharing template with chunked-upload-aware defaults         |
| [Seafile](templates/seafile/)                   | File sharing template with WebDAV-aware proxy defaults           |
| [Synapse](templates/synapse/)                   | Matrix homeserver template with well-known delegation            |
| [Tomcat](templates/tomcat/)                     | Reverse proxy template with servlet-friendly defaults            |
| [Tuwunel](templates/tuwunel/)                   | Matrix homeserver reverse proxy with WebSocket and delegation    |
| [Uptime Kuma](templates/uptime-kuma/)           | Monitoring template with Socket.IO polling and WebSockets        |
| [Vaultwarden](templates/vaultwarden/)           | Password vault template with encrypted-field WAF tuning          |
| [WordPress](templates/wordpress/)               | Secure template with essential hardening defaults                |
| [Xen Orchestra](templates/xen-orchestra/)       | Reverse proxy template with WebSocket and rate limiting defaults |

## Install

The **web UI is recommended** for most users. Use the plugin method when you already manage
BunkerWeb plugins as files or need repeatable, automated deployments.

First, download and extract the
[latest release archive](https://github.com/bunkerity/bunkerweb-templates/releases/latest), or
clone this repository. Choose the service directory you want to install and read its README for
prerequisites and service-specific settings. Below, `<template-directory>` means that extracted
service directory, such as `wordpress/` in the release archive or `templates/wordpress/` in a
repository clone.

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
   `modsec/` or `server-http/`.
3. Confirm the resulting plugin layout. WordPress, for example, must look like this:

   ```text
   plugins/
   └── example-plugin/
       ├── plugin.json
       └── templates/
           ├── wordpress.json
           └── wordpress/
               └── configs/
                   └── modsec/
                       └── wordpress_false_positives.conf
   ```

4. Install or reload the plugin using the procedure for your BunkerWeb integration. The scheduler
   loads the template from the plugin.
5. Select the template when adding or editing a service in easy mode, or set `USE_TEMPLATE=<name>`.
   For the example above, use `USE_TEMPLATE=wordpress` because the file is named `wordpress.json`.

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
  "configs": ["modsec/wordpress_false_positives.conf"],
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
      "configs": ["modsec/wordpress_false_positives.conf"]
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
