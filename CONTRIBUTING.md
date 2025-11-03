# Contributing to bunkerweb-templates

Thanks for helping grow the BunkerWeb template ecosystem! This repository thrives when the community shares knowledge, fixes bugs, and keeps templates up to date for new scenarios. The information below covers the end-to-end process for contributing.

## Code of Conduct

We follow the tone and values of the BunkerWeb community: be respectful, be inclusive, and collaborate openly. Please review our [Code of Conduct](CODE_OF_CONDUCT.md) before participating. If you ever feel unsure, reach out before you submit and we will figure it out together.

## Contribution Workflow

1. **Fork** this repository and clone your fork locally.
2. **Create a branch** named after your change, e.g. `feature/wordpress-template`.
3. **Install tooling and Git hooks** so your changes match the repository style:
   - Install [`pre-commit`](https://pre-commit.com/) (`pipx install pre-commit` or `pip install pre-commit` inside a virtualenv).
   - Install the hooks: `pre-commit install`.
   - Ensure the `prettier` CLI is available on your PATH (for example `npm install --global prettier` or via your package manager) so the formatting hook can run.
4. **Implement your update** (add a template, adjust docs, etc.) following the guidelines in this document and in [TEMPLATE_GUIDE.md](TEMPLATE_GUIDE.md).
5. **Test and lint** your contribution. Run `pre-commit run --all-files` and address any issues it reports. For templates, also validate configuration files where possible (e.g. `docker compose config`, `kubectl apply --server-dry-run`, schema validation).
6. **Update the changelog**. Add a line under the `## Unreleased` section in `CHANGELOG.md` summarizing your change using the format `- [@github-handle] Short description of the change`.
7. **Commit** with clear messages that describe the change and reference any related issues.
8. **Open a Pull Request** against `main`:
   - Summarize the motivation and changes.
   - Highlight testing steps and expected outcomes.
   - Link issues or discussions that provide additional context.
9. **Participate in the review**. Address comments, push revisions, and keep the conversation open and collaborative.
10. Once approved, a maintainer will merge your PR.

## Template Submission Guidelines

To keep the templates repository tidy and easy to navigate, please align with these conventions:

- **Naming**: Use lowercase-kebab-case for directories (e.g. `wordpress`, `plex`, `nextcloud-hardening`). Prefer descriptive names that reflect the service or goal.
- **Folder structure**: Place template assets inside `templates/<template-name>/`. Avoid adding extra environment subdirectories; each template stands alone. At minimum, include a `template.json` that follows the BunkerWeb schema.
- **Template file**: `template.json` must define a unique template `id`, a user-facing `name`, and any `settings`, `configs`, or `steps` it relies on. Paths inside `configs` should resolve relative to the template directory.
- **Documentation**: Add a `README.md` or short `NOTES.md` when additional guidance helps users customize the template. Mention whether the template is meant for UI import via Raw mode (recommended), plugin distribution, or both.
- **Examples**: When relevant, include sample configuration fragments under `configs/` and keep them referenced inside `template.json`.

Refer to [TEMPLATE_GUIDE.md](TEMPLATE_GUIDE.md) for detailed expectations and examples.

## Style & Quality Expectations

- **Consistency**: Follow the formatting rules in the template guide. Keep indentation, file naming, and JSON structure consistent.
- **Clarity**: Comment configuration blocks when the intent is not obvious. Avoid secrets or sensitive data.
- **Testing**: Share how you validated the template. Provide commands, logs, or screenshots if it helps reviewers.
- **Review readiness**: Ensure new templates are self-contained, with clear instructions and metadata, so reviewers can try them quickly.

## Pull Request Checklist

Before submitting, double-check the following:

- [ ] I installed the pre-commit hooks and ran `pre-commit run --all-files`.
- [ ] My branch builds or validates locally (where applicable).
- [ ] The template follows the documented folder structure and naming conventions.
- [ ] `template.json` is valid and references only files included in the template directory.
- [ ] Template README (if provided) explains prerequisites, setup steps, and how to import the template (UI Raw import, recommended, vs. plugin bundle).
- [ ] I updated or added tests, examples, or documentation as needed.
- [ ] I added or updated an entry in `CHANGELOG.md` under `## Unreleased` using the format `- [@github-handle] Summary of the change`.
- [ ] I confirmed that all files include the necessary license headers if required.
- [ ] I linked related issues or discussions in the PR description.

## Licensing

By submitting a contribution, you agree that it will be distributed under the **GNU General Public License v3.0 (GPL-3.0)**. Please only contribute content that you are authorized to share under this license.

## Need Help?

If you have questions about the workflow, template expectations, or licensing:

- Open an issue for general questions or suggestions.
- Reach out on the [Bunkerity Discord server](https://discord.gg/YEdMKqztMZ) (`#🧩・templates` channel recommended).
- Draft your PR early and mark it as a work in progress to get feedback.

Thank you for contributing and helping others adopt BunkerWeb faster!
