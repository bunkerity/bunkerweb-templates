# Plumber

CI/CD security scanning for the GitHub Actions workflows in this repository.
[Plumber](https://getplumber.io) statically analyses `.github/workflows/` for
supply-chain and pipeline misconfigurations: unpinned or unvetted actions,
`secrets: inherit`, missing `permissions:` blocks, dangerous triggers,
template injection, cache poisoning and branch protection gaps.

This repository ships no code, only template JSON and NGINX/ModSecurity
snippets, so the workflows that package and publish releases are the whole
attack surface Plumber has to cover.

## Files

| Path | Purpose |
|---|---|
| `.github/plumber/plumber.yaml` | Policy overlay on top of `plumber:default` |
| `.github/plumber/README.md` | This file |
| `.github/workflows/plumber.yml` | Reusable workflow invoked by both release workflows |

## Running it locally

```bash
brew install getplumber/tap/plumber        # or download the binary from the releases page
plumber analyze --config .github/plumber/plumber.yaml --score
plumber explain ISSUE-713                  # details for a given issue code
```

`plumber analyze` scans the local workflows and queries the GitHub API for
branch protection. That last control needs a token with `Administration: read`;
without it, protection findings are incomplete rather than wrong.

## Policy

`plumber.yaml` extends `plumber:default` and only adds an allowlist of the
third-party actions this repository already relies on — a single entry,
`softprops/action-gh-release`. It names an exact `owner/repo` rather than an
owner wildcard, so trust cannot spread to other or future repositories under
that account. It is pinned by commit SHA in both workflows, and Dependabot
bumps that pin weekly along with every other action.

## Gating

`dev-template-prerelease.yml` and `manual-template-release.yml` both call the
reusable workflow as a `needs:` dependency, so nothing is packaged or published
until the scan passes; the reusable workflow also runs on its own every Monday.
It is gated at `min-score: B` with `soft-fail: false`, so scores of C, D or E
fail the run. Results land in the Code Scanning tab.

Every input is set explicitly in `plumber.yml`, including those that match the
action's own defaults, so that an auditor reads the effective configuration
from the workflow alone and never has to diff it against `action.yml` at some
past tag. `verify-attestation: true` keeps the sigstore/SLSA provenance check
on the downloaded binary; `score-push: true` publishes the score used by the
hosted badge service — the README badge reads from
`https://score.getplumber.io/github.com/bunkerity/bunkerweb-templates.svg` and stays
`unknown` until the first run with `score-push` publishes a score.

Each run also uploads a `plumber-report` artifact holding the JSON report, the
PBOM, the CycloneDX SBOM and the raw SARIF (`upload-artifacts: true`). The
SARIF is redundant with Code Scanning; the PBOM and SBOM are kept as per-run
evidence of what the pipeline consumed.

Both callers declare `permissions: read-all` at the workflow level and widen to
`contents: write` only on the packaging job. A reusable workflow may only
maintain or reduce the caller's token scope, so the `plumber` job in each
caller grants exactly the three scopes `plumber.yml` needs: `contents: read`,
`security-events: write` and `id-token: write` for the OIDC identity behind
`score-push`.
