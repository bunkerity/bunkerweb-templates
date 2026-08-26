<p align="center">
	<img alt="BunkerWeb Templates logo" src="./assets/logo.png" width=350 />
</p>

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

> Ready-to-run BunkerWeb configurations for popular services, fully editable to fit your stack.

The **bunkerweb-templates** repository collects community-maintained templates that deliver a working [BunkerWeb](https://www.bunkerweb.io) configuration for well-known services such as WordPress, Nextcloud, and Matrix homeservers. Each template mirrors BunkerWeb’s template specification, giving you a safe baseline that you can extend or reshape for your environment.

All templates and documentation in this repository are released under the [GNU General Public License v3.0](LICENSE), in line with the main BunkerWeb project.

## Highlights
- **Service-first library:** Templates target specific apps (e.g. WordPress and Nextcloud) so you can jump straight to a functional setup.
- **Self-contained assets:** Every entry under `templates/<name>/` bundles the JSON definition and referenced configs.
- **Editable by design:** Settings, configs, and steps follow BunkerWeb’s template spec, making tweaks straightforward.
- **Built for iteration:** Contributor workflows, style guides, and review checklists keep the hub growing sustainably.
- **Open collaboration:** Discuss ideas, report issues, and ship updates alongside the BunkerWeb community.

## How Templates Work

BunkerWeb includes three predefined templates (`low`, `medium`, `high`) that bundle common security settings. The easy-mode UI and the `USE_TEMPLATE` setting can also consume **custom templates** defined as JSON. A custom template can declare:

- **Settings** – Multisite settings whose values should override defaults.
- **Configs** – Paths to reusable NGINX configuration snippets that will be merged into your service.
- **Steps** – Optional guided steps combining settings and configs so users can adjust the template interactively in the UI.

This repository packages those JSON definitions alongside their configuration snippets, giving you a safe baseline that you can drop into BunkerWeb and adapt to any deployment method.

## Table of Contents
- [Highlights](#highlights)
- [How Templates Work](#how-templates-work)
- [Table of Contents](#table-of-contents)
- [Installing Templates](#installing-templates)
- [Available Templates](#available-templates)
- [Template Anatomy](#template-anatomy)
- [Repository Layout](#repository-layout)
- [Contributing](#contributing)
- [Community](#community)
- [Maintainers](#maintainers)
- [License](#license)

## Installing Templates

The **web UI is recommended** for most users. Use the plugin method when you
already manage BunkerWeb plugins as files or need repeatable, automated
deployments.

First, download and extract the
[latest release archive](https://github.com/bunkerity/bunkerweb-templates/releases/latest),
or clone this repository. Choose the service directory you want to install and
read its README for prerequisites and service-specific settings. Below,
`<template-directory>` means that extracted service directory, such as
`wordpress/` in the release archive or `templates/wordpress/` in a repository
clone.

### Web UI (Recommended)

1. Sign in to the BunkerWeb web UI with an account that can edit the
   configuration.
2. Open **Templates**, select **Create new template**, and switch to **Raw**
   mode.
3. Select **Upload template JSON** and choose
   `<template-directory>/template.json`. You can instead paste the file's
   contents into the JSON editor. Confirm **Replace template** if prompted.
4. If the **Missing Custom Configs** dialog appears, the template needs its
   accompanying NGINX configuration files:
   - Select **Browse Files**, or drag files into the upload area.
   - Upload every listed file from `<template-directory>/configs/` and its
     subdirectories. Filenames must match the list; selecting the directories
     themselves is not required.
   - Wait until every item is marked **Uploaded**, then select **Done**.
5. Review the imported JSON and configs, then select **Save**. Templates without
   custom configs skip the previous dialog.
6. Add or edit a service in **Easy** mode, choose the new template, and review
   each guided step. Replace example domains, upstream URLs, and other defaults
   with values for your deployment before saving the service.

The web UI stores the imported template and config contents in BunkerWeb's
database. The local files are no longer needed after a successful import. For
non-UI configuration, apply the same template with `USE_TEMPLATE=<name>`, where
`<name>` is the template ID shown in the UI.

### Plugin (Managed Deployments)

Template directories from this repository are **not standalone plugins**. Add
the selected template to an existing external plugin, or create a plugin with a
`plugin.json` file by following the
[BunkerWeb plugin documentation](https://docs.bunkerweb.io/latest/plugins/#how-to-use-a-plugin).

BunkerWeb uses the JSON filename as the template ID, so the repository layout
must be rearranged inside the plugin:

1. Copy `<template-directory>/template.json` to
   `<plugin>/templates/<name>.json`.
2. If the template contains a `configs/` directory, copy it to
   `<plugin>/templates/<name>/configs/`. Preserve every config-type
   subdirectory, such as `modsec/` or `server-http/`.
3. Confirm the resulting plugin layout. WordPress, for example, must look like
   this:

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

4. Install or reload the plugin using the procedure for your BunkerWeb
   integration. The scheduler loads the template from the plugin.
5. Select the template when adding or editing a service in easy mode, or set
   `USE_TEMPLATE=<name>`. For the example above, use
   `USE_TEMPLATE=wordpress` because the file is named `wordpress.json`.

## Available Templates

| Template                                  | Summary                                                          | Directory                  |
| ----------------------------------------- | ---------------------------------------------------------------- | -------------------------- |
| [Directus](templates/directus/)           | Headless CMS template with REST and GraphQL API defaults         | `templates/directus/`      |
| [Drupal](templates/drupal/)               | Secure template with CMS-aware defaults                          | `templates/drupal/`        |
| [Jellyfin](templates/jellyfin/)           | Media streaming template with reverse proxy tuning               | `templates/jellyfin/`      |
| [Nextcloud](templates/nextcloud/)         | Secure template with WebDAV-aware defaults                       | `templates/nextcloud/`     |
| [NetBird](templates/netbird/)             | Self-hosted template with gRPC and websocket routing             | `templates/netbird/`       |
| [openGym](templates/opengym/)             | Passkey PWA template with JSON sync and WebAuthn tuning          | `templates/opengym/`       |
| [Pi-hole](templates/pi-hole/)             | Reverse proxy template with admin UI and API tuning              | `templates/pi-hole/`       |
| [Synapse](templates/synapse/)             | Matrix homeserver template with well-known delegation           | `templates/synapse/`       |
| [Tomcat](templates/tomcat/)               | Reverse proxy template with servlet-friendly defaults            | `templates/tomcat/`        |
| [Tuwunel](templates/tuwunel/)             | Matrix homeserver reverse proxy with WebSocket and delegation    | `templates/tuwunel/`       |
| [Xen Orchestra](templates/xen-orchestra/) | Reverse proxy template with WebSocket and rate limiting defaults | `templates/xen-orchestra/` |
| [WordPress](templates/wordpress/)         | Secure template with essential hardening defaults                | `templates/wordpress/`     |

```text
templates/
├── netbird/
│   ├── template.json
│   └── configs/
│       └── modsec/
│           └── netbird_false_positives.conf
└── wordpress/
    └── template.json
```

## Template Anatomy

Every repository template keeps its definition and supporting assets together:

- `template.json` describes the template `id`, user-facing `name`, and optional `settings`, `configs`, and `steps`.
- `configs/` contains any NGINX fragments referenced in the JSON file.
- Additional assets (e.g. helper scripts, documentation) can live beside these defaults if they assist users.

A minimal `template.json` might look like:

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
  "configs": [
    "modsec/wordpress_false_positives.conf"
  ],
  "steps": [
    {
      "title": "Domains and TLS",
      "subtitle": "Point the template at your WordPress hostname(s).",
      "settings": [
        "SERVER_NAME"
      ]
    },
    {
      "title": "Reverse proxy",
      "subtitle": "Tell BunkerWeb where your WordPress upstream lives.",
      "settings": [
        "REVERSE_PROXY_HOST",
        "MAX_CLIENT_SIZE",
        "MODSECURITY_CRS_PLUGINS"
      ],
      "configs": [
        "modsec/wordpress_false_positives.conf"
      ]
    }
  ]
}
```

When installed in a plugin, its files use this layout:

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

## Repository Layout

- `templates/` – Community-curated templates, one directory per service with its `template.json` and related configs.
- `TEMPLATE_GUIDE.md` – Canonical instructions for authors creating or updating a template.
- `CONTRIBUTING.md` – Contribution workflow, review expectations, and the submission checklist.

## Contributing

We welcome community contributions! Review the [CONTRIBUTING.md](CONTRIBUTING.md) guide for the full workflow. In short:

1. Fork the repository and create a feature branch.
2. Add or update your template following the guidelines.
3. Submit a pull request describing your changes and any testing performed.

Maintainers and community members will review submissions collaboratively to ensure templates stay reliable and helpful.

Before you push, install the repository’s `pre-commit` hooks (`pre-commit install`) and run `pre-commit run --all-files` so formatting, spell-check, and secret scans pass locally.

## Community

Questions or ideas? Join the conversation with fellow BunkerWeb maintainers and users:

- Discord: [discord.gg/YEdMKqztMZ](https://discord.gg/YEdMKqztMZ)
- GitHub Issues: Use the issue tracker to report bugs or request new templates.
- Wiki: Browse the [project wiki](https://github.com/bunkerity/bunkerweb-templates/wiki) for concise navigation and troubleshooting.
- Code of Conduct: Review our [community guidelines](CODE_OF_CONDUCT.md) before engaging.

## Maintainers

| Name           | GitHub                                             |
| -------------- | -------------------------------------------------- |
| Théophile Diot | [@TheophileDiot](https://github.com/TheophileDiot) |
| Alexandre      | [@YouKyi](https://github.com/YouKyi)               |

## License

This repository, including all templates and documentation, is licensed under the [GNU General Public License v3.0](LICENSE). By contributing, you agree that your contributions will be released under the same license.
