<p align="center">
	<img alt="BunkerWeb Templates logo" src="./assets/logo.png" width=350 />
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-GPLv3-blue.svg" alt="License: GPL v3" /></a>
  <a href="https://github.com/bunkerity/bunkerweb-templates/stargazers"><img src="https://img.shields.io/github/stars/bunkerity/bunkerweb-templates?color=ffb400&logo=github" alt="GitHub stars" /></a>
  <a href="https://github.com/bunkerity/bunkerweb-templates/issues"><img src="https://img.shields.io/github/issues/bunkerity/bunkerweb-templates?label=issues" alt="Open issues" /></a>
  <a href="https://github.com/bunkerity/bunkerweb-templates/pulls"><img src="https://img.shields.io/github/issues-pr/bunkerity/bunkerweb-templates?label=PRs" alt="Open pull requests" /></a>
  <a href="https://github.com/bunkerity/bunkerweb-templates/commits"><img src="https://img.shields.io/github/last-commit/bunkerity/bunkerweb-templates?label=last%20update" alt="Last update" /></a>
  <a href="https://discord.gg/YEdMKqztMZ"><img src="https://img.shields.io/badge/community-Discord-5865F2?logo=discord&logoColor=white" alt="Join the Discord" /></a>
  <a href="CONTRIBUTING.md"><img src="https://img.shields.io/badge/PRs-welcome-brightgreen.svg" alt="PRs welcome" /></a>
</p>

> Ready-to-run BunkerWeb configurations for popular services, fully editable to fit your stack.

The **bunkerweb-templates** repository collects community-maintained templates that deliver a working [BunkerWeb](https://www.bunkerweb.io) configuration for well-known services such as WordPress, Plex, Nextcloud, or self-hosted dashboards. Each template mirrors BunkerWeb’s template specification, giving you a safe baseline that you can extend or reshape for your environment.

All templates and documentation in this repository are released under the [GNU General Public License v3.0](LICENSE), in line with the main BunkerWeb project.

## Highlights
- **Service-first library:** Templates target specific apps (e.g. WordPress, Plex) so you can jump straight to a functional setup.
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
- [Getting Started](#getting-started)
- [Available Templates](#available-templates)
- [Template Anatomy](#template-anatomy)
- [Repository Layout](#repository-layout)
- [Contributing](#contributing)
- [Community](#community)
- [Maintainers](#maintainers)
- [License](#license)

## Getting Started

1. **Browse templates:** Explore `templates/`; each directory targets a specific service or use case and includes its required assets.
2. **Review the JSON definition:** Inspect `<template>/template.json` to understand the settings, configs, and optional steps it provides.
3. **Import the template:** Either drop the directory into a plugin’s `templates/` folder or upload the JSON bundle from the BunkerWeb web UI on the `Templates` page, then reference it with the `USE_TEMPLATE` setting or the easy-mode workflow.
4. **Adjust to taste:** Edit the JSON and accompanying configuration snippets so the template reflects your infrastructure.
5. **Share feedback:** Open an issue or pull request if you find improvements, new services, or fixes worth sharing.

## Available Templates

| Template                          | Summary                                               | Directory              |
| --------------------------------- | ----------------------------------------------------- | ---------------------- |
| [Drupal](templates/drupal/)       | Secure template with CMS-aware defaults               | `templates/drupal/`    |
| [Jellyfin](templates/jellyfin/)   | Media streaming template with reverse proxy tuning    | `templates/jellyfin/`  |
| [Nextcloud](templates/nextcloud/) | Secure template with WebDAV-aware defaults            | `templates/nextcloud/` |
| [NetBird](templates/netbird/)     | Self-hosted template with gRPC and websocket routing  | `templates/netbird/`   |
| [Tomcat](templates/tomcat/)       | Reverse proxy template with servlet-friendly defaults | `templates/tomcat/`    |
| [WordPress](templates/wordpress/) | Secure template with essential hardening defaults     | `templates/wordpress/` |
| [Tuwunel](templates/tuwunel/) | Secure template with essential hardening defaults for the Tuwunel matrix server     | `templates/tuwunel/` |

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

Every template directory mirrors the structure that BunkerWeb expects when loading custom templates from a plugin:

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

And its directory can be embedded in a plugin like:

```text
plugins/
└── example-plugin/
    ├── plugin.json
    └── templates/
        └── wordpress/
            ├── template.json
            └── configs/
                └── modsec/
                    └── wordpress_false_positives.conf
```

Alternatively, you can zip or upload the directory contents directly from the BunkerWeb web UI (`Templates` page) to make it available without packaging a plugin.

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
- Code of Conduct: Review our [community guidelines](CODE_OF_CONDUCT.md) before engaging.

## Maintainers

| Name           | GitHub                                             |
| -------------- | -------------------------------------------------- |
| Théophile Diot | [@TheophileDiot](https://github.com/TheophileDiot) |
| Alexandre      | [@YouKyi](https://github.com/YouKyi)               |

## License

This repository, including all templates and documentation, is licensed under the [GNU General Public License v3.0](LICENSE). By contributing, you agree that your contributions will be released under the same license.
