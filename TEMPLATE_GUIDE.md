# Template Author Guide

Use this guide when creating or updating templates for **bunkerweb-templates**. Following these expectations keeps every template clear, consistent, and easy to maintain.

## Template Essentials

Each template lives under `templates/<template-name>/` and should contain:

- `template.json`: The BunkerWeb template definition that declares the template `id`, user-facing `name`, and any referenced settings, configs, or steps.
- `configs/`: Optional directory holding NGINX fragments referenced in `template.json`.
- Additional assets: Optional files such as helper scripts or a short `README.md` to provide service-specific notes.

Name templates descriptively (for example `wordpress`, `nextcloud`, `synapse`, `base-hardening`). Because templates behave the same across Docker, Kubernetes, bare metal, and other integrations, keep everything directly under `templates/` instead of nesting by environment.

> ℹ️ For a deeper explanation of how BunkerWeb consumes templates (including the built-in `low`, `medium`, and `high` presets), see the [official templates concept documentation](https://docs.bunkerweb.io/latest/concepts/#templates).

### `template.json` essentials

Every `template.json` must follow the structure expected by BunkerWeb:

| Property   | Type   | Required | Description                                                                                          |
| ---------- | ------ | -------- | ---------------------------------------------------------------------------------------------------- |
| `id`       | string | Yes      | Stable identifier for the template; keep it unique and aligned with the directory name.             |
| `name`     | string | Yes      | User-facing label shown in the UI; keep it concise and descriptive.                                  |
| `settings` | object | No       | Key/value pairs of multisite settings that override defaults when the template is applied.           |
| `configs`  | array  | No       | Paths relative to `configs/` for NGINX configuration snippets that should be included.                |
| `steps`    | array  | No       | Ordered steps that group related settings and configs for easy-mode guidance.                        |

Example:

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
    "modsec-crs/wordpress_false_positives.conf"
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
        "modsec-crs/wordpress_false_positives.conf"
      ]
    }
  ]
}
```

#### ID

- Use a lowercase, hyphenated slug that stays consistent between releases (for example `wordpress` or `nextcloud-hardening`).
- Keep the `id` synchronized with the directory name so users can map files to their template references easily.
- Treat the `id` as immutable once a template is published; consumers rely on it to detect updates.

#### Settings

- Only multisite settings are supported. Use the keys exactly as BunkerWeb expects them (e.g. `SERVER_NAME`, `USE_PROXY_CACHE`).
- Provide sensible defaults, knowing that users can still adjust them after importing the template.

#### Configs

- Point to files relative to the template's `configs/` directory (for example `modsec-crs/false_positives.conf`).
- Organize snippets under a supported BunkerWeb configuration type such as `modsec/`, `modsec-crs/`, or `server-http/`. Rules with `ctl:` exclusions belong in `modsec-crs/`.

#### Steps

- Steps are optional but recommended for multi-stage setups.
- Each step may reference settings and configs that appear elsewhere in the template. Doing so lets the easy-mode UI expose those items to the user for tweaking.
- Titles and subtitles should be concise instructions (for example “Adjust domains”, “Review hardening rules”).

### Template extras

While not required by BunkerWeb, consider adding:

- `README.md`: Brief context about the service, prerequisites, and customization tips, laid out as described in [README layout](#readme-layout).
- `.env.example` or helper scripts: Only when they simplify recreating the service locally.
- Screenshot or diagram references when they improve understanding (keep assets lightweight).

### README layout

Every template README follows one layout, so the pages read as a set on GitHub (light and dark
theme). [`templates/wordpress/README.md`](templates/wordpress/README.md) is the worked example:
copy its header and section names, then keep your own technical text. GitHub strips `style`,
`class` and scripts, so the design comes from layout tags GitHub keeps (`p`, `h1`, `img`,
`picture`, `source`, `a`, `br`, `sub`, `details`, tables), shields.io badges and GitHub alerts.
Do not use `div`, `span`, `style=` or emoji in headings.

**Sections, in this order.** Leave out a section the service does not need. Do not add empty ones;
only the header, At a glance and the footer are always present.

1. **Header**: the skeleton below (icon, name, one-line summary, badges, nav line).
2. **Intro**: the opening paragraph, with no "Overview" heading.
3. `## At a glance`: the table below.
4. `## Prerequisites`.
5. Service-side configuration (for example `## Configure Pocket ID for the proxy`). It comes before
   Install because the service is set up before the template is imported.
6. `## Install`: import steps and template-specific import notes. Link to the root
   [installation guide](README.md#install) instead of repeating it.
7. `## Settings`: settings, tradeoffs and customization tips. Template-specific parts are `###`
   subsections.
8. `## Files`.
9. `## Validation`: how it was validated and tested.
10. Troubleshooting or known-limit sections, with names that say what they hold.
11. **Footer**: a rule (`---`) and the `All templates` link.

**Header skeleton.** Replace the slug, name, summary and section links. `<icon base>` is the URL
in the Icons list below, up to and including `/svg`. Keep `width` and `height` on every image. The
header icon takes an `alt` that names the service (`alt="<Name> logo"`); an icon in a root README
table takes `alt=""`, because the service name sits right beside it, and `align="absmiddle"` (see
Root README below).

```html
<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="<icon base>/<slug>-light.svg">
    <img src="<icon base>/<slug>.svg" alt="<Name> logo" width="96" height="96" />
  </picture>
</p>

<h1 align="center"><Name></h1>

<p align="center">One-line summary, the same words as the root README row</p>

<p align="center">
  <img src="https://img.shields.io/badge/BunkerWeb-1.6.17-0b5577?labelColor=1d6f93&style=flat" alt="BunkerWeb 1.6.17" />
  <img src="https://img.shields.io/badge/OWASP_CRS-4-0b5577?labelColor=1d6f93&style=flat" alt="OWASP CRS 4" />
</p>

<p align="center">
  <a href="../../README.md#templates">All templates</a> ·
  <a href="#prerequisites">Prerequisites</a> ·
  <a href="#install">Install</a> ·
  <a href="#settings">Settings</a>
</p>
```

Footer:

```html
---

<p align="center">
  <a href="../../README.md#templates">All templates</a>
</p>
```

**At a glance.** A two-column table (`Fact`, `Value`). Every value comes from the README itself,
`template.json` or the Rule IDs table in this guide; never invent one.

| Row           | Value                                                                  | Source                                                                        |
| ------------- | ---------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `Template id` | the `id`, in code                                                      | `template.json`                                                               |
| `Tested with` | `BunkerWeb 1.6.17, OWASP CRS 4`, plus the service version if tested    | the note in the root README. List a service version only if your README says it was tested on BunkerWeb 1.6.17 with that version. A version tested on an older BunkerWeb stays out |
| `Requires`    | a minimum version, such as `BunkerWeb >= 1.6.9~rc1`; omit when none   | only a minimum your README states. A minimum never counts as tested           |
| `Configs`     | always present: count and config types, such as `2 (modsec-crs)`, or `0` when `template.json` lists no configs | the `configs` paths in `template.json` |
| `Rule ids`    | the template's block, such as `9878000` to `9878999`; omit without rules | the Rule IDs table below; only when the template ships ModSecurity rules     |
| `Upstream`    | the default `REVERSE_PROXY_HOST`, in code; omit when unset             | `template.json`                                                               |

Badges follow the same rule: `BunkerWeb`, `OWASP CRS`, and a `<Service> | <version>` badge only when
the README says the template was tested on BunkerWeb 1.6.17 with that service version. A service
version tested on an older BunkerWeb gets no badge. Every badge takes the same label colour,
`labelColor=1d6f93`, and the value colour is `0b5577` (BunkerWeb, OWASP CRS) or `248651` (the
service version). White text on each is at least 4.5:1 (`1d6f93` 5.6:1, `0b5577` 8.1:1, `248651`
4.56:1), and each colour keeps at least 2:1 against GitHub dark (`#0d1117`): `1d6f93` 3.4:1,
`0b5577` 2.3:1, `248651` 4.1:1. Do not pick other colours.

**Alerts.** A paragraph that already warns (it opens with "Security:", says "Do not" or "must", or
describes data loss or a ban) may become `> [!WARNING]` or `> [!IMPORTANT]`; a note becomes
`> [!NOTE]` or `> [!TIP]`. Keep the text word for word and use a few alerts per README, so each one
keeps its weight. An alert wraps at most three source lines of text (the `> [!TYPE]` line does not
count); a longer warning stays a plain paragraph, with its bold lead-in (such as `**Cost:**`) as the
signal. Wrap a long table in `<details>` only when it is evidence for a claim made above it, never
for steps the reader must follow.

**Icons.** Use the service's own logo from
[Dashboard Icons](https://github.com/homarr-labs/dashboard-icons) (Apache-2.0), hotlinked from its
`main` branch. Compare it with the product's own site or repository, since a same-named icon can
belong to another project.

- URL: `https://raw.githubusercontent.com/homarr-labs/dashboard-icons/main/svg/<slug>.svg`, always
  on the source's default branch (`main` for Dashboard Icons and Kaneo, `master` for DocuSeal),
  never a commit SHA or another branch. Icons follow upstream, so a redrawn icon shows
  up without a change here: raw.githubusercontent.com caches for 5 minutes and GitHub's image proxy
  follows it. The cost is that a renamed or deleted upstream file shows as a broken image until a
  README is fixed. The weekly `README icon check` workflow (`.github/workflows/readme-icons.yml`)
  fails when an icon URL stops answering HTTP 200; run it from the Actions tab before a release.
  Do not use the jsDelivr form of this URL: it answered `403` for some files of a large icon
  repository.
- No credit line: the icons are only linked, nothing is copied into this repository, so no licence
  notice is due. The root README License section keeps one sentence about trademarks. Do not copy
  SVG files into the repository: a copy makes the Apache-2.0 notice due.
- Variants: `<slug>.svg` has the brand colours and `<slug>-light.svg` is white (for dark
  backgrounds), when upstream has one. None of the services in this repository has a `-dark`
  variant in Dashboard Icons (checked 2026-11-19), so an icon that is hard to see on white has no
  replacement: ask in the pull request.
- Theme rule: use `<slug>.svg` in both themes. When it is hard to see on GitHub dark (`#0d1117`)
  and `<slug>-light.svg` exists, wrap it in `<picture>` with a dark `<source>` for
  `<slug>-light.svg`, as in the skeleton. Check the README in both themes.
- No entry for the service, or an entry that is another project's logo? Use the logo mark from
  the project's own repository on its default branch (Kaneo:
  `https://raw.githubusercontent.com/usekaneo/kaneo/main/apps/web/public/favicon.svg`; DocuSeal,
  whose Dashboard Icons entry is a cartoon seal face:
  `https://raw.githubusercontent.com/docusealco/docuseal/master/public/logo.svg`), the square mark
  and not a wordmark. When the project has no logo at all, use the mark of the
  protocol it serves, say so in `alt`, and mention it in the pull request.
- Sizes: 96 px in the header of a template README, 24 px in the root README table.

**Root README.** Add a row for your template to its category table, in alphabetical order. The
tables have two columns, `| Template | Summary |`. The first cell is the 24 px icon markup (an
`<img>`, or a `<picture>` when the icon needs a dark variant), one space, then the linked name:

```html
| <img src="<icon base>/<slug>.svg" align="absmiddle" alt="" width="24" height="24" /> [Name](templates/<name>/) | Same summary as the README subtitle |
```

`align="absmiddle"` centres the icon on the name: GitHub strips `style`, but keeps `align` on an
`<img>`, and without it the icon sits about 6 px above the middle of the text. In a `<picture>`
the `align` goes on the inner `<img>`. Nothing else goes in that cell. Add a new category only
when none of the existing ones fits.

### Distribution options

Use the repository's [installation guide](README.md#install) for both supported delivery methods:

1. **Web UI Raw import (recommended)** – Upload or paste `template.json`, upload every missing custom config when prompted, save, then select the template for a service.
2. **Plugin bundle** – Copy `template.json` to `<plugin>/templates/<name>.json` and copy `configs/` to `<plugin>/templates/<name>/configs/` before installing or reloading the plugin.

The repository and plugin layouts differ, so preserve the structure documented in the installation guide rather than copying the source directory unchanged.

For deeper background on BunkerWeb concepts referenced in this guide, see the official docs:

- Templates overview: [docs.bunkerweb.io/concepts/templates](https://docs.bunkerweb.io/latest/concepts/#templates)
- Integrations and deployment guides: [docs.bunkerweb.io/integrations](https://docs.bunkerweb.io/latest/integrations)
- Advanced configuration examples: [docs.bunkerweb.io/advanced](https://docs.bunkerweb.io/latest/advanced)

## Rule IDs

Every ModSecurity rule carries a unique numeric `id`, and each range of ids has an owner. Pick yours from
the template range; never reuse an id from a range that belongs to someone else (the OWASP Core Rule Set
owns 900,000 to 999,999, and a clash makes the config test fail with a duplicate id).

- **BunkerWeb** owns 9,800,000 to 9,899,999.
- **Community templates** use 9,850,000 to 9,899,999: 50 blocks of 1,000 ids, **one block per template**.
  A template uses only ids inside its own block.
- **A new template** takes the next free block from 9,880,000, in the order templates are added. Add its
  row to the table below in the same pull request.
- **Your own rules**, in examples and in your deployments, use 1 to 99,999: ModSecurity reserves that range
  for local use. Do not copy it into a template.

Layout of a block (the block of `wordpress`, 9,878,000, as the example):

| Ids in the block | Use                                                                                                                                                         |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `+000` to `+009` | Free.                                                                                                                                                       |
| `+010` upward    | Initialization `SecAction` rules (`setvar` of a CRS variable): 9,878,010, 9,878,011.                                                                        |
| `+100` to `+499` | Request rules, from `+100`. A group of related rules starts on a multiple of 10 and its siblings take the next numbers: 9,878,110, 9,878,111.               |
| `+500` to `+998` | Response rules (phase 3 or 4), from `+500`.                                                                                                                 |
| `+099`, `+999`   | Not used. Kept free for an optional kill switch.                                                                                                            |

The only id a template may keep from outside its block is a CRS setup id: `nextcloud` keeps 900130, the id
CRS gives to that setup rule.

| Block start | Template        |
| ----------- | --------------- |
| `9850000` | `atuin` |
| `9851000` | `authentik` |
| `9852000` | `beszel` |
| `9853000` | `directus` |
| `9854000` | `drupal` |
| `9855000` | `filestash` |
| `9856000` | `forgejo` |
| `9857000` | `garage` |
| `9858000` | `grafana` |
| `9859000` | `jellyfin` |
| `9860000` | `kaneo` |
| `9861000` | `n8n` |
| `9862000` | `netbird` |
| `9863000` | `nextcloud` |
| `9864000` | `ntfy` |
| `9865000` | `open-webui` |
| `9866000` | `opengym` |
| `9867000` | `outline` |
| `9868000` | `pi-hole` |
| `9869000` | `pocket-id` |
| `9870000` | `projectsend` |
| `9871000` | `searxng` |
| `9872000` | `synapse` |
| `9873000` | `tomcat` |
| `9874000` | `tuwunel` |
| `9875000` | `umami` |
| `9876000` | `uptime-kuma` |
| `9877000` | `vaultwarden` |
| `9878000` | `wordpress` |
| `9879000` | `xen-orchestra` |

### Rule form

Write the rules the way the CRS does (see the CRS `CONTRIBUTING.md`): the actions in the order `id`,
`phase`, the disruptive action, `t:`, `nolog`, `ctl:`, `setvar:`, `chain`, one action per line, four spaces
of indentation and chain links indented one level deeper than their starter. Give every rule an explicit
`phase`, an explicit operator and `t:none`, and a short comment above it. Prefer `@streq` and `@beginsWith`
over `@rx` for a literal path, and anchor the rest with `\A` and `\z`. Put runtime exclusions (`ctl:`) in
`modsec-crs/`, which loads before the CRS: a `ctl:` only reaches rules that have not run yet. Run
`crs-linter` over the file; the `ver` and `tag` findings do not apply to templates.

The comments of a config file are shown to the administrator in the web UI config editor once the
template is imported, and for most users they are the only explanation of the rules. Write them for that
reader: say what traffic the rule lets through and which feature of the app needs it, what it costs (the
CRS checks that are off for that traffic and any limit to change, such as an index range), and how to
adjust or remove it. Leave out how the rule was tested, and never put a comment inside a multi-line
directive (after a line that ends with `\`). Do not quote attack strings, function calls, HTML tags,
SQL keywords after a quote, or a semicolon followed by a word in a comment: the web UI runs the CRS
on the whole template during import, config text included, and one hit makes Save fail with a 403.

Templates target CRS 4, the default of BunkerWeb 1.6.17. The legacy short tags (`attack-rce`, `attack-lfi`,
`attack-xss`, `attack-sqli`, `attack-injection-php`) exist in CRS 3 and CRS 4, so a tag-based exclusion
(`ruleRemoveByTag`, `ruleRemoveTargetByTag`) that uses only them needs no note. The `OWASP_CRS/<family>`
tags (for example `OWASP_CRS/ATTACK-LFI`) are the CRS 4 ones: a template that uses them says "CRS 4
required" in its README. On BunkerWeb 1.6.17, `USE_MODSECURITY_GLOBAL_CRS=yes` skips the per-service
`modsec-crs` configs, so the exclusions of a template only take effect with the default (`no`).

## Formatting Guidelines

- **Pre-commit hooks**: Install `pre-commit`, run `pre-commit install`, and execute `pre-commit run --all-files` before pushing so the shared formatting (Prettier, Codespell, Gitleaks) is applied consistently.
- **Directory names**: Lowercase, words separated by hyphens.
- **JSON**: Two spaces for indentation, double quotes for keys and strings, trailing commas avoided.
- **NGINX snippets**: Keep indentation consistent; add short comments when behavior is non-obvious.
- **Markdown**: Wrap commands in backticks, keep lines under ~100 characters, use fenced code blocks for multi-line content.
- **Filenames**: Stick to ASCII characters and avoid spaces.

## Tips & Best Practices

- Document which tooling or platforms you used while testing (e.g. Docker Compose 2.17, Kubernetes 1.29) so users know what you validated, but avoid duplicating files unless truly required.
- Reference official documentation for service-specific configuration defaults so users understand why a value matters.
- Validate JSON before submitting (`jq . template.json`) and lint NGINX snippets locally where possible.
- Keep secrets and credentials out of the repository. Use placeholders and explain where real values belong.
- Prefer declarative configuration over shell scripts; if a script is required, explain why in the template README.
- Log every change in `CHANGELOG.md` by adding a bullet under `## Unreleased` using the format `- [@github-handle] What changed`. This keeps manual releases accurate and makes it easy to track ownership.

## Licensing Reminder

All templates and documentation must comply with the **GNU General Public License v3.0 (GPL-3.0)**. Only submit content you can license under GPL-3.0, and ensure any third-party snippets are compatible.

Questions or looking for feedback before you finalize a template? Open a draft pull request or start a discussion in the [Bunkerity Discord community](https://discord.gg/YEdMKqztMZ) (`#🧩・templates` channel recommended). We are happy to help!
