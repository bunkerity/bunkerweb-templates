# Changelog

All notable updates to the template collection live here. Keep the newest entries at the top so
manual releases can pull details straight from this file.

## Unreleased

- [@alfi4000] Add Directus template: reverse proxy with WebSocket support and buffering disabled for asset streaming, `PUT`/`PATCH`/`DELETE` in `ALLOWED_METHODS` for the REST API, `MAX_CLIENT_SIZE=1024m` for asset uploads, CORS settings for decoupled front ends, and upstream security headers preserved so the admin app keeps its own CSP.
- [@TheophileDiot] Harden the Directus template before merge: keep ModSecurity blocking instead of `DetectionOnly`, raise `MODSECURITY_REQ_BODY_NO_FILES_LIMIT` to 10 MB so JSON bodies are not rejected at 128 KB, drop `401`, `403` and `404` from `BAD_BEHAVIOR_STATUS_CODES` since Directus returns them on anonymous and permission-denied API calls, set `CORS_ALLOW_ORIGIN` explicitly, enable `USE_GZIP` so `GZIP_PROXIED` is not inert, and remove the internal-use `IS_DRAFT`, the unreachable `USE_CROWDSEC`, and `CLIENT_CACHE_ETAG` (a no-op behind `SERVE_FILES=no`).
- [@TheophileDiot] Add openGym template: reverse proxy to the bundled nginx front end, `PUT` in `ALLOWED_METHODS` for the `/api/data` sync path, `MODSECURITY_REQ_BODY_NO_FILES_LIMIT` raised in step with `MAX_CLIENT_SIZE` so ModSecurity does not reject the body at 128 KB, per-path rate limits on the passkey, pairing and coach endpoints, `401` dropped from `BAD_BEHAVIOR_STATUS_CODE` because `/api/me` returns it to every anonymous visitor, `COOKIE_FLAGS` kept free of `Domain` so the `__Host-` prefixed session cookie survives, and narrow CRS exclusions for the base64url WebAuthn ceremony bodies.
- [@TheophileDiot] Add Plumber CI/CD security scanning: `.github/workflows/plumber.yml` as a reusable workflow (weekly cron plus `workflow_call`), gated at `min-score: B` with `soft-fail: false`, wired as a `needs:` dependency on both release workflows, with a `plumber.yaml` policy that extends `plumber:default` and allowlists `softprops/action-gh-release`. Add the Plumber score badge to the README.
- [@TheophileDiot] Fix the Nextcloud template: move the CRS exclusion toggle (`900130`) to `configs/modsec-crs/` so it loads before the CRS rules, drop the redundant `900200` allowed-methods rule that BunkerWeb already generates from `ALLOWED_METHODS`, lower `MAX_CLIENT_SIZE` from `10G` to `512m` (it also drives ModSecurity's in-memory `SecRequestBodyLimit`), set `LIMIT_REQ_URL=/` with `LIMIT_REQ_RATE=15r/s` in place of BunkerWeb's implicit `2r/s` catch-all, and remove `401` from `BAD_BEHAVIOR_STATUS_CODES` since Nextcloud returns it on unauthenticated WebDAV requests.
- [@TheophileDiot] Expand the Nextcloud template README: reverse-proxy requirements (`trusted_proxies`, `overwriteprotocol`), upload sizing, rate-limit slots and matching semantics, CRS plugin pinning, CalDAV/CardDAV discovery checks, and validation commands.

## Templates release v0.5 - 2026-07-20

- [@TheophileDiot] Refresh repository and wiki documentation, centralize volatile guidance, and align contribution and plugin instructions.
- [@YouKyi] Fix ModSecurity rule syntax and clean up WordPress configuration.
- [@palmcoasty] Add Tuwunel (Matrix homeserver) template with reverse proxy, rate limiting, and .well-known delegation.
- [@palmcoasty] Add Synapse (Matrix homeserver) template with reverse proxy, well-known delegation, and upload limits.
- [@palmcoasty] Document that the WordPress REST API PUT/DELETE methods are disabled by default and how to enable them via ALLOWED_METHODS.
- [@palmcoasty] Add Pi-hole template with reverse proxy and ModSecurity tuning for the admin UI.
- [@TheophileDiot] Clarify template installation through the web UI and plugin bundles, and centralize setup guidance in the root README.
- [@TheophileDiot] Expand the Nextcloud template `ALLOWED_METHODS` with `SEARCH`, `MKCALENDAR`, `ACL`, and `PATCH` to cover WebDAV search, CalDAV, DAVACL, and chunked upload v2. Align `tx.allowed_methods` in `configs/modsec/nextcloud_false_positives.conf` with the template definition (also restores the missing `REPORT` verb).

## Templates release v0.4 - 2026-04-24

- [@TheophileDiot] Add Xen Orchestra template with reverse proxy, WebSocket, and rate limiting defaults.
- [@YouKyi] Edit the Nextcloud template to handle higher rate limits for the Nextcloud Mail app.
- [@Simonmiz] Add WebSocket reverse-proxy support to the Nextcloud template (`REVERSE_PROXY_WS`) and document the configuration in the template README.
- [@TheophileDiot] Add `.coderabbit.yaml` with automated pull request review guidelines tailored to template contributions.
- [@TheophileDiot] Add `CLAUDE.md` with project guidance and development commands for contributors using Claude Code.

## Templates release v0.3 - 2026-02-25

- [@TheophileDiot] Add an automated `dev` pre-release workflow that packages templates on every push to the `dev` branch and replaces any existing dev release.
- [@TheophileDiot] Add a NetBird template with reverse proxy, gRPC, and ModSecurity defaults.

## Templates release v0.2 - 2026-02-20

- [@TheophileDiot] Align Jellyfin reverse proxy config with official docs (ModSec, rate limit, CSP, Permissions-Policy) + fix issues with the CRS

## Templates release v0.1 - 2025-11-03

- [@TheophileDiot] Initial release bundling the existing templates.
