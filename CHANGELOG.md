# Changelog

All notable changes to kd-cli are recorded here. The project follows [Semantic Versioning](https://semver.org/).

## [Unreleased]

## [2.2.2] — 2026-04-18

### Fixed
- `kd /issue update ID --field "NAME="` now clears optional custom fields by sending `value: null` to YouTrack. Required fields reject empty values with a clear client-side error instead of the generic "invalid value" bundle message (KDCLI-81).

## [2.2.1] — 2026-04-18

### Added
- `kd --guide` / `-G` prints an embedded operating guide covering kd's product boundary, primer workflow, cost classes, and YouTrack gotchas — the doctrine complement to the syntax-focused `kd --reference`. Runs without a connection or working config, so it's safe in fresh or broken-config environments (KDCLI-94).

## [2.2.0] — 2026-04-18

Milestone 006 — tag discovery and taxonomy hygiene v1. Eight sequenced sessions that turn tags into a load-bearing cross-entity discovery primitive: `/issue` tag parity with `/article`, multi-tag intersection on `/tag issues`, the `/tag entities` cross-kind verb, `/tag list --prefix|--namespace` taxonomy introspection, and workspace-pinned tags on the primer consumed as an implicit OR scope.

### Added
- `/issue list --with-tags` opts in to a tag-aware field set; rows include a comma-joined `tags` column (cheap default path is unchanged). The flag is the explicit opt-in for tag-aware listings — `-f`/`--fields` stays a pure output filter (KDCLI-83, M006 S3).
- `/issue show` always renders a `tags` row (parity with `/article show`) and an `updated` row alongside `created` (timestamp normalisation handled automatically) (KDCLI-83, M006 S3).
- `/tag issues TAG --tag T2 [--tag T3] [--project P]` accepts repeatable `--tag` with AND semantics, mirroring `/tag articles`. Single-tag without `--project` keeps the fast `GET /api/tags/{id}/issues` endpoint; multi-tag (or any use of `--project`) composes a YouTrack query via `youtrack_query_literal()` and uses `/issues?query=...`. Same-field clauses are joined with the explicit `and` keyword because YouTrack DSL combines repeated same-field clauses as OR by default (KDCLI-88, M006 S4).
- `/tag entities TAG [--tag T2 ...] [--project P] [--kind issue|article|all] [--limit-issues N] [--limit-articles N] [--limit N]` — unified cross-kind tag discovery. Returns a flat `results` list plus a `meta` dict reporting per-kind completeness (`scanned`, `complete`, `truncated`, `count`, `limit_per_kind`, optional `error`). Default sort is kind asc (articles first), updated desc within kind. Single-kind failure yields exit 0 with `meta.<kind>.error` populated — structured consumers must read `meta` (exit 0 does not mean every kind succeeded). `--limit` is the symmetric per-kind default; `--limit-issues` and `--limit-articles` override per kind (specific over general) (KDCLI-89, M006 S5).
- `/tag list --prefix P` / `/tag list --namespace N` filter tags by name prefix via the server's substring `?query=` plus a client-side `startswith` post-filter. Pagination walks until `--limit` matches are collected or the source is exhausted; a hard cap at `10 × --limit` server pages emits a one-line stderr warning and exits 0 with whatever was found. `--namespace N` is a convenience shorthand that appends the trailing colon; the two flags are mutually exclusive. Output shape is unchanged — plain list of `Tag` rows (KDCLI-90, M006 S6).
- `/primer/tag list|add|remove` — workspace-pinned tag list in `.kd/primer.yaml`. `/primer show` renders a `tags:` block between notes and articles. `/tag entities --primer-scope` (mutually exclusive with positional `TAG` and `--tag`) uses the pinned list as an implicit OR scope; empty primer raises a `ValidationError` that names `/primer/tag add`. Every `/primer*` command accepts `--project` as a workspace assertion (KDCLI-84 folded in). Output envelope gains a structured `warnings: [{code, message, context}]` field; table and markdown output append `warning [<code>]: <message>` after rows, JSON and YAML pass the field through (KDCLI-92, M006 S7).

### Fixed
- `/issue show` and `/issue list --with-tags` now display tags. Tag state on issues was previously write-only from the CLI — operators had to reverse-lookup via `/tag issues TAGNAME` for each candidate (KDCLI-83, M006 S3).

### Documentation
- `docs/specs/001-youtrack-api-reference.md` gains alias equivalence (`tag: X` / `#X` / `tagged as:` and negation forms) and the full brace-escape matrix for tag predicates in the query DSL (M006 S8).
- New `docs/how-to/015-cross-entity-tag-discovery-how-to.md` — worked examples for `/tag entities`, `meta` interpretation, per-kind pagination, partial-failure handling, and `--primer-scope` (M006 S5 / S8).

### Internal
- `TagOperations.entities/issues/articles` gained a `match: Literal["all", "any"] = "all"` parameter. `"any"` joins multi-tag issue queries with the explicit `or` keyword and switches the article scan from `issubset` to `not isdisjoint`. Defaults preserve every existing caller's semantics — no user-visible API change (KDCLI-92, M006 S7).

## [2.1.7] — 2026-04-18

### Internal
- Config get/set/which surface now lives symmetrically in the SDK, not split across the CLI module. Pure refactor — no command output, flag, or error-message change (KDCLI-63).

## [2.1.6] — 2026-04-17

Milestone 005 — CLI friction audit, session S2. Two raise sites enriched so users land on the right file and the right fix.

### Fixed
- Missing-token errors now name the connection and URL, surface a copy-pasteable `token_cmd:` example, and cite the config file that declared `token_env:`. Previously the user saw only the env-var name — in a multi-connection setup, guesswork about which connection failed and which file to edit (KDCLI-72).
- Config merge-validation errors cite the contributing file for each offending locus. A bad local override now names the local file per key; a bad global-only entry names the global file. `kd /system/config set` leaf-coercion errors name the target config path. Origin data was already tracked via `ConnectionOrigin` for `config list`; this release threads it into the user-visible error string too (KDCLI-69).

### Changed
- The `.suggestion` attribute now lives on the `KdError` base instead of being declared per-subclass. `_print_error` renders `Hint:` for any `KdError` with a suggestion — `AuthError` and `ValidationError` now carry suggestions alongside `APIError` and `ConfigError`. Internal plumbing; no CLI surface change.

### Documentation
- Replace `vod` with `support` in help-text examples, code docstrings, current-behavior design docs, specs, how-tos, and test fixtures. Historical records (session docs, CHANGELOG, shipped milestone close-outs) keep their `vod` references intact so the audit trail stays honest (KDCLI-30).

## [2.1.5] — 2026-04-17

Milestone 005 — CLI friction audit, session S1. Four first-time-user surface fixes on the `--help` / `--output` axis.

### Fixed
- `--help`, `--reference`, and per-command help now render the `--output` choices list exactly once. The literal help text carries the list and the default (`Output format: table|json|yaml|md (default: table)`); four renderer sites that previously auto-appended `(table, json, yaml, md)` a second time stopped doing so (KDCLI-73).
- `kd` no longer flips the default output format to JSON when stdout is a pipe. `table` is the unconditional default across TTY, pipes, and subprocess contexts, per [ADR-006](../../docs/design/ADR-006-tty-detection-config-first.md). Consumers wanting JSON or YAML pass `-o json` / `-o yaml` explicitly (KDCLI-74).
- `kd /primer show` on an empty primer now prints a populate hint in `table` and `md` output, instead of rendering zero bytes. JSON and YAML continue to return the canonical empty container `{"notes": [], "articles": []}` (KDCLI-76).
- `kd /primer/note add`, `kd /primer/article add`, and their sibling `remove` / `update` commands accept `--project/-p` as a workspace assertion. A match against the active connection's `default_project` is a silent no-op; a mismatch raises a `ValidationError` naming both the asserted and resolved keys so an agent running from the wrong workspace fails loud instead of silently writing to the wrong primer (KDCLI-70).

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
