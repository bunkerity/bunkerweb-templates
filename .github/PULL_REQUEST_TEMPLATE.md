# Summary

- <!-- Give a concise overview of the change -->

# Testing

- [ ] `jq . templates/<template-name>/template.json`
- [ ] Validated template or docs using the listed commands
- [ ] Other (add details below)

<details>
<summary>Validation details</summary>

- <!-- List commands, logs, screenshots, or notes -->

</details>

# Checklist

- [ ] I installed the pre-commit hooks and ran `pre-commit run --all-files`.
- [ ] I described the service or scenario this change targets.
- [ ] `template.json` references only files shipped in the same template directory.
- [ ] Template docs cover service-specific setup, link to the root installation guide and follow the [README layout](https://github.com/bunkerity/bunkerweb-templates/blob/dev/TEMPLATE_GUIDE.md#readme-layout).
- [ ] I updated configs, screenshots, or notes impacted by this change.
- [ ] I added an attributed entry under `## Unreleased` in `CHANGELOG.md`.
- [ ] I linked related issues or discussions and added context for reviewers.
