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

- `README.md`: Brief context about the service, prerequisites, and customization tips.
- `.env.example` or helper scripts: Only when they simplify recreating the service locally.
- Screenshot or diagram references when they improve understanding (keep assets lightweight).

### Distribution options

Use the repository's [installation guide](README.md#installing-templates) for both supported delivery methods:

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
