# Changelog

All notable changes to kd-cli are recorded here. The project follows [Semantic Versioning](https://semver.org/).

## [Unreleased]

## [2.1.4] — 2026-04-15

### Fixed
- Connection `url` with a trailing slash no longer produces opaque `JSONDecodeError`. URLs are now normalized on load (`https://host/` → `https://host`), and the HTTP layer guards against non-JSON 2xx responses — a misconfigured URL that lands on YouTrack's HTML web UI now raises a clear `APIError` naming the URL and content type (KDCLI-67).

### Documentation
- `docs/how-to/012-install-how-to.md` now walks through obtaining a YouTrack API token (Profile → Account Security → New token) and clarifies that the `token_env` name is user-configurable. The configuration how-to cross-links to it (KDCLI-68).

## [2.1.3] — 2026-04-15

### Fixed
- `sync-changelog` CI job now correctly publishes `CHANGELOG.md` on its first run against `kd-cli`. The 2.1.2 release exposed a guard that ignored untracked files, so the initial sync was a no-op and the PyPI Changelog link 404'd until this release (KDCLI-65).

## [2.1.2] — 2026-04-15

### Changed
- PyPI `Changelog` sidebar link now resolves. `pyproject.toml` points at `CHANGELOG.md` in the root of the public `kd-cli` repo, and a new `sync-changelog` CI job (runs on `v*` tags after PyPI publish succeeds) keeps that mirror in lockstep with the canonical `kd/resources/CHANGELOG.md` shipped in the wheel (KDCLI-65).

## [2.1.1] — 2026-04-14

### Changed
- Scrubbed a real customer name from the `kd /project create` example so `kd /project create --help` and `kd --reference` now show a generic `Feedback Board` placeholder (KDCLI-30).

## [2.1.0] — 2026-04-14

Milestone 004 — config UX and release notes. Six sequenced sessions on the
`pipx install → kd init → edit config → hit an error → recover → find out
what's new` journey.

### Added
- `kd /release-notes [VERSION]` serves the bundled changelog; supports `--since X.Y.Z` and `--all` (KDCLI-57).
- `kd init --global [URL]` scaffolds `~/.config/kd/config.yaml` from the shared template, creating the parent directory if needed (KDCLI-56).
- `kd /system/config which <option_path>` traces the 10-layer resolution chain, showing the winning source and every shadowed layer. `config set` now emits an inline shadow warning when a higher-priority layer would override the written value (KDCLI-62).
- `kd /system/config keys` enumerates every configurable path with type, default, and description; `config set` and `config get` accept nested dotted paths (e.g. `connections.vod.default_project`) instead of the previous four-flat-key gate (KDCLI-59).

### Fixed
- Connection blocks now deep-merge per-key across the global and local tiers instead of whole-block replace. A local `connections.vod.default_project` override no longer strips the globally-defined `url` and token settings (KDCLI-60).
- Malformed configs render a clean `Error:`/`Hint:` surface instead of leaking raw Pydantic tracebacks. Broken YAML, missing connection URLs, and validation failures are all caught at the CLI boundary and rewritten with a path pointer and actionable suggestion (KDCLI-61).

## [2.0.6] — 2026-04-14

### Added
- PyPI project URLs (Homepage, Issues, Source) and the `OS Independent` classifier in the distribution metadata (KDCLI-30).

## [2.0.5] — 2026-04-14

### Changed
- README and CLI banner reframed around the "knowledge-network tool" narrative (KDCLI-30).

## [2.0.4] — 2026-04-14

### Added
- PyPI publishing pipeline and `KDCLI-A-15` how-to for releases (YTCTL-30).
- TestPyPI trigger on `rc-*` tags alongside the existing `v*` → PyPI flow (YTCTL-30).
- `types-PyYAML` and `types-tabulate` in the dev extras (YTCTL-30).

### Changed
- `__version__` is now looked up under the `kd-cli` distribution name, not the `kd` import name (YTCTL-30).
- Python floor lowered to 3.11; PyPI metadata prepared for the first kd-cli upload (YTCTL-30).

## [2.0.3] — 2026-04-13

Milestone 003 — `ytctl → kd` rename cutover.

### Added
- First release under the `kd-cli` distribution and `kd` import name, replacing `ytctl` end to end (YTCTL-48, YTCTL-49, YTCTL-50, YTCTL-51, YTCTL-52).

### Changed
- Package, module, distribution, entry point, workspace directory, config directory, environment variables, user-facing surface, and article marker all renamed. The parser treats the legacy `<!-- ytctl-meta -->` marker as a read-only synonym for transitional migration.
- Rename parity harness recorded S5 regression against the upstream `YTST` baseline (YTCTL-51).
- Sdist manifest and packaging metadata updated; `.ytctl/` removed from `.gitignore`; glossary and docs refreshed under the new name (YTCTL-52).
