# AGENTS.md

This file provides guidance to AI coding agents working in this repository.

## Project Overview

Community-curated collection of ready-to-import BunkerWeb security templates for popular services. The current catalog lives in `README.md`; templates bundle service-specific settings and optional NGINX configurations. GPL-3.0 licensed.

## Development Commands

```bash
pre-commit install                        # Register git hooks (one-time setup)
pre-commit run --all-files                # Run full validation suite (Prettier, Codespell, Gitleaks)
jq . templates/<name>/template.json       # Validate template JSON syntax
```

No build step or automated test harness exists. Validation is manual: `docker compose config`, `kubectl apply --dry-run=server -f`, or `nginx -t -c <config>` for service-specific checks.

## Repository Architecture

Shared contributor docs sit at the repository root (`CONTRIBUTING.md`, `TEMPLATE_GUIDE.md`). GitHub issue and PR templates are under `.github/ISSUE_TEMPLATE/` and `.github/PULL_REQUEST_TEMPLATE.md`. Tooling configuration stays close to the root, including `.pre-commit-config.yaml` and `.prettierignore`.

**Template structure** — each template follows this layout:

```
templates/<service-name>/
├── template.json                    # BunkerWeb template definition (required)
├── README.md                        # Service-specific docs
└── configs/                         # Optional NGINX/ModSecurity snippets
    ├── modsec-crs/                  # ModSecurity rules loaded before the CRS (most exclusions)
    │   └── <service>_false_positives.conf
    ├── modsec/                      # ModSecurity rules loaded after the CRS
    └── server-http/                 # NGINX directives for the service's server block
```

**template.json schema**:

- `id`: lowercase hyphenated identifier
- `name`: user-facing label
- `settings`: BunkerWeb multisite settings as key-value pairs (keys must match BunkerWeb setting names exactly)
- `configs`: array of paths relative to the template's `configs/` directory
- `steps`: ordered workflow steps with `title`, `subtitle`, `settings`, and `configs` for guided UI import

Use `templates/wordpress/` as the reference template for structure, tone, and step organization.

**CI/CD** — three GitHub Actions workflows:

- `dev-template-prerelease.yml`: workflow dispatch for dev pre-releases
- `manual-template-release.yml`: workflow dispatch for production releases
- `plumber.yml`: Plumber security scan of the workflows, weekly on `main` and as a required job of both release workflows

## Conventions

- Directory names: lowercase-kebab-case, ASCII only
- JSON: two-space indent, double-quoted keys, no trailing commas (Prettier-enforced)
- Commits: imperative mood ("Add Jellyfin template"), single concern per commit
- Changelog: all changes go under `## Unreleased` in `CHANGELOG.md` as `- [@github-handle] Summary`; releases track template updates from that list, so keep entries current
- Config file naming: `<service>_false_positives.conf` for ModSecurity exclusion rules
- All config paths in `template.json` are relative to the template's `configs/` directory; referenced files must exist in-repo
- NGINX snippets should include comments when relaxing security defaults

Markdown content should wrap commands in backticks and keep lines under roughly 100 characters. Prettier handles formatting for JSON, Markdown, and YAML via the pre-commit hook, so ensure the `prettier` CLI is available on your PATH.

## Testing Guidelines

For new templates, provide the commands you used to test imports or reverse proxies. Log output or screenshots can be attached in PR discussions when behavior is complex. Name config snippets to reflect their intent (e.g., `modsec/wordpress_false_positives.conf`) so reviewers understand coverage at a glance.

## Git Workflow

- Main branch: `main` (releases), development branch: `dev` (manual pre-releases)
- Feature branches: `feature/<name>`, bugfix branches: `bugfix/<name>`
- Follow `CONTRIBUTING.md` for the current PR target branch; include validation notes and linked issues
- Pre-commit suite must pass before every push and before opening PRs
- Pull requests must describe the service or docs touched, note validation steps, and link related issues. Before opening a PR, ensure `template.json` references only in-repo files and documentation explains how to import or deploy the template. Include screenshots when UI-facing docs change.

## External References

- [BunkerWeb templates documentation](https://docs.bunkerweb.io/latest/concepts/#templates)
- `TEMPLATE_GUIDE.md` for detailed template authoring specification
- `CONTRIBUTING.md` for full contribution workflow
