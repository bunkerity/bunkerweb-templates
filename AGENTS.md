# Repository Guidelines

## Project Structure & Module Organization

Templates live under `templates/<template-name>/`, each with a `template.json` and any referenced assets inside `configs/` or sibling files such as `README.md`. Shared contributor docs sit at the repository root (`CONTRIBUTING.md`, `TEMPLATE_GUIDE.md`). GitHub issue and PR templates are under `.github/ISSUE_TEMPLATE/` and `.github/PULL_REQUEST_TEMPLATE.md`. Tooling configuration stays close to the root, including `.pre-commit-config.yaml` and `.prettierignore`.

## Build, Test, and Development Commands

- `pre-commit install`: Registers repository hooks so formatting, spell checks, and secret scans run automatically.
- `pre-commit run --all-files`: Executes the full hook suite (Prettier, Codespell, Gitleaks) across the tree; run before every push.
- `jq . templates/<name>/template.json`: Validates template JSON for syntax errors.
- Service-specific validation (pick what applies): `docker compose config`, `kubectl apply --server-dry-run -f`, or `nginx -t -c <config>` to confirm generated snippets.

## Coding Style & Naming Conventions

Use lowercase-kebab-case for directory names and stick to ASCII filenames. JSON files are indented with two spaces and double-quoted keys; avoid trailing commas. Markdown content should wrap commands in backticks and keep lines under roughly 100 characters. Prettier handles formatting for JSON, Markdown, and YAML via the pre-commit hook, so ensure the `prettier` CLI is available on your PATH.

## Testing Guidelines

There is no automated test harness; rely on targeted validation. For new templates, provide the commands you used to test imports or reverse proxies. Log output or screenshots can be attached in PR discussions when behavior is complex. Name config snippets to reflect their intent (e.g., `modsec/wordpress_false_positives.conf`) so reviewers understand coverage at a glance.

## Commit & Pull Request Guidelines

Write commits in the imperative mood (“Add Jellyfin template configs”) and keep histories focused on a single concern. Pull requests must describe the service or docs touched, note validation steps, and link related issues. Before opening a PR, ensure the pre-commit suite passes, `template.json` references only in-repo files, and documentation explains how to import or deploy the template. Include screenshots when UI-facing docs change.

Always record changes in `CHANGELOG.md` under the `## Unreleased` section using the bullet format `- [@github-handle] Summary of the change`. Releases track template updates from that list, so keep entries current and attribute the work clearly.
