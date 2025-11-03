# Template Author Guide

Use this guide when creating or updating templates for **bunkerweb-templates**. Following these expectations keeps every template clear, consistent, and easy to maintain.

## Template Essentials

Each template lives under `templates/<template-name>/` and should contain:

- `template.json`: The BunkerWeb template definition that declares the template `id`, user-facing `name`, and any referenced settings, configs, or steps.
- `configs/`: Optional directory holding NGINX fragments referenced in `template.json`.
- Additional assets: Optional files such as helper scripts or a short `README.md` to provide service-specific notes.

Name templates descriptively (for example `wordpress`, `plex`, `nextcloud`, `base-hardening`). Because templates behave the same across Docker, Kubernetes, bare metal, and other integrations, keep everything directly under `templates/` instead of nesting by environment.

> ℹ️ For a deeper explanation of how BunkerWeb consumes templates (including the built-in `low`, `medium`, and `high` presets), see the [official templates concept documentation](https://docs.bunkerweb.io/latest/concepts/#templates).

### `template.json` essentials

Every `template.json` must follow the structure expected by BunkerWeb:

| Property   | Type   | Required | Description                                                                                          |
| ---------- | ------ | -------- | ---------------------------------------------------------------------------------------------------- |
| `id`       | string | Yes      | Stable identifier for the template; keep it unique and aligned with the directory name.             |
| `name`     | string | Yes      | User-facing label shown in the UI; keep it concise and descriptive.                                  |
| `settings` | object | No       | Key/value pairs of multisite settings that override defaults when the template is applied.           |
| `configs`  | array  | No       | Relative paths (inside the template folder) to NGINX configuration snippets that should be included. |
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

#### ID

- Use a lowercase, hyphenated slug that stays consistent between releases (for example `wordpress` or `nextcloud-hardening`).
- Keep the `id` synchronized with the directory name so users can map files to their template references easily.
- Treat the `id` as immutable once a template is published; consumers rely on it to detect updates.

#### Settings

- Only multisite settings are supported. Use the keys exactly as BunkerWeb expects them (e.g. `MULTISITE_DOMAIN`, `PROXY_CACHE_ENABLE`).
- Provide sensible defaults, knowing that users can still adjust them after importing the template.

#### Configs

- Point to files located within the template directory (for example `modsec/false_positives.conf`).
- Organize configuration snippets under subfolders such as `modsec/` or `addons/` if it adds clarity.

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

You can deliver templates in two ways:

1. **UI Raw import (recommended)** – From the BunkerWeb web UI, open the `Templates` page, click **Create new template**, switch to **Raw** mode, paste the contents of `template.json`, and add any referenced config files using the in-browser editor before saving. This is the fastest way to test or share a template without packaging a plugin.
2. **Plugin bundle** – Place the template directory inside a plugin’s `templates/` folder (as shown above). When the plugin is installed, the template becomes available automatically.

Regardless of the delivery method, keep the folder layout identical so users can switch between approaches effortlessly.

For deeper background on BunkerWeb concepts referenced in this guide, see the official docs:

- Templates overview: [docs.bunkerweb.io/concepts/templates](https://docs.bunkerweb.io/latest/concepts/#templates)
- Integrations and deployment guides: [docs.bunkerweb.io/integrations](https://docs.bunkerweb.io/latest/integrations)
- Advanced configuration examples: [docs.bunkerweb.io/advanced](https://docs.bunkerweb.io/latest/advanced)

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
- When updating an existing template, note the review date or version in the template README so users see it is maintained.

## Licensing Reminder

All templates and documentation must comply with the **GNU General Public License v3.0 (GPL-3.0)**. Only submit content you can license under GPL-3.0, and ensure any third-party snippets are compatible.

Questions or looking for feedback before you finalize a template? Open a draft pull request or start a discussion in the [Bunkerity Discord community](https://discord.gg/YEdMKqztMZ) (`#🧩・templates` channel recommended). We are happy to help!
