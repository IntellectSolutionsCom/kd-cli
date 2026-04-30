# Changelog

All notable changes to kd-cli are recorded here. The project follows [Semantic Versioning](https://semver.org/).

## [Unreleased]

## [2.6.5] — 2026-04-30

This release closes M011 by adding empty-project bootstrap support to
`/issue create`. Together with M011 S1 + S2 (the Windows UTF-8 audit in
2.6.4), the milestone delivers a kd-only path from a fresh primer to
the first ticket on a brand-new project on Windows: file IO, stdin,
and field-resolution all work without falling back to the web UI.

### Fixed
- `/issue create` semantic bootstrap now succeeds on fresh
  (zero-issue, contributor-token) projects: when both schema-discovery
  paths return empty, kd synthesizes a best-effort payload for
  `--type`, `--priority`, and `--state` using the post-discovery
  `$type` mapping and lets YouTrack accept or reject. Pre-fix, kd
  bailed before the API call with `Error: No custom fields discovered
  for project '<id>'`, telling the operator to fall back to the web UI
  for the very first issue — a kd-only bootstrap workflow could not
  populate Type / Priority / State on a brand-new project.
  `--field NAME=VALUE` still raises with the improved hint (kd cannot
  guess server-side `$type` for arbitrary fields). The hint gains a
  third bullet pointing at the simplest workaround: `Or omit --type /
  --priority / --state to create a minimal first issue, then update it
  via `/issue update``. See KDCLI-A-22 for the empty-project bootstrap
  detail (KDCLI-171).

## [2.6.4] — 2026-04-30

This release closes the Windows UTF-8 audit started in 2.6.3. The
stream-encoding side (stdout/stderr) shipped earlier; 2.6.4 closes the
file-IO and stdin sides so primer notes, `--content @file`, and heredoc
or piped input round-trip non-cp1252 characters cleanly.

### Fixed
- File reads and writes on Windows are now explicit UTF-8 at every
  kd-owned text-IO call site (primer, config, history/diff snapshots,
  `--description @file` / `--content @file` content). The headline
  reproducer was `kd /primer show` raising `'charmap' codec can't
  decode byte 0x9d` whenever `.kd/primer.yaml` contained any non-cp1252
  byte (em-dash, arrow, right double quotation mark) — a single primer
  note authored on macOS/Linux was enough to read-block every
  downstream Windows session (KDCLI-170).
- Stdin on Windows is now forced to UTF-8 with strict error handling,
  so heredocs and pipes carrying non-ASCII content (em-dash, arrow,
  right double quotation mark) into `kd /primer/note update -`,
  `kd /article create --content @-`, `--description -`, or any other
  stdin-consuming command no longer get cp1252-mojibaked at the input
  layer before being persisted. Invalid UTF-8 bytes on stdin (and
  non-UTF-8 `@file` content) now surface as a clean
  `Error: Stdin input could not be decoded as UTF-8 ... / Hint: Ensure
  the upstream shell pipes UTF-8 bytes ...` (or
  `Hint: Save the file as UTF-8 ...`) instead of the generic "internal
  error: ... This is a bug." block (KDCLI-169).

## [2.6.3] — 2026-04-30

This release focuses on Windows shakedown: encoding crashes, primer
locking, mojibake in help text, the git-bash path-conversion gotcha,
and a contributor-token field-resolver bug surfaced along the way.

### Fixed
- `/project/field list` and the `--type` / `--priority` / `--state` /
  `--field` resolvers now work for tokens that lack project-administration
  rights (e.g. contributor / bot accounts). When both the per-issue and
  admin paths return no fields, kd raises a typed error pointing at
  token permissions instead of failing the resolver silently. Surfaced
  during the Windows shakedown but identity-class — the fix is
  platform-agnostic (KDCLI-164).
- Primer mutations (`/primer/note add`, `/primer/tag add`,
  `/primer/article add`, plus all update/remove paths) now succeed on
  Windows. Replaces the previous POSIX-only implementation that raised
  `ConfigError("File locking is not available on this platform (POSIX
  only).")` at the very first acquisition on Windows. Concurrent kd
  processes still serialize on the `.kd/primer.yaml.lock` file on both
  platforms (KDCLI-159).
- Output containing characters outside cp1252 (`→`, `—`, `≤`) no longer
  crashes with `UnicodeEncodeError` on Windows, and help/synopsis text
  on the default Windows console no longer renders mojibake (`?` / `�`)
  for en-dashes and similar. Both paths went through the same encoding
  layer (KDCLI-158, KDCLI-161, KDCLI-156).
- Unhandled exceptions in the CLI now render a one-line (encoding) or
  one-paragraph (generic) friendly diagnostic on stderr and exit `1`
  instead of dumping a raw Python traceback. The traceback is preserved
  under `--debug` / `-D` / `-DD` / `-DDD` so developer workflows are
  intact. Applies on every platform (KDCLI-162).
- Operating guide and user guide now document the git-bash MSYS
  path-conversion gotcha. `kd /primer show` from git-bash on Windows
  arrives as `kd C:/Program Files/Git/primer show` because MSYS
  rewrites the leading `/` before kd ever sees argv; recovery is the
  recommended `alias kd='MSYS_NO_PATHCONV=1 kd'` in `~/.bashrc`,
  `MSYS_NO_PATHCONV=1 kd …` per call, or the `kd //primer show`
  double-slash form (KDCLI-163).

### Added
- `kd /system info -o yaml` now exposes the active connection slug as
  a top-level `connection` key. Paste `kd /system info -o yaml` into
  a bug report and the slug used to reach the instance is captured
  without hand-typing — `-c <slug>` overrides flow through to the
  field as well. The previous payload named the URL but never the
  slug, so reports against multi-connection setups had to spell the
  slug out by hand (KDCLI-160).
- Git-bash MSYS path-conversion hint on Windows: when `kd <converted
  path>` reaches argv as `C:/.../<known kd namespace>`, kd prints a
  one-line stderr hint above the existing "Unknown command" failure
  pointing at the recovery recipes (KDCLI-163).

### Changed
- Added `filelock>=3.12` as a direct runtime dependency. Normal
  `kd-cli` end-user installs gain one new direct runtime package; the
  dev wheel set is unaffected because dev tooling already resolved
  `filelock` transitively. See
  `docs/design/ADR-012-cross-platform-locking.md` (KDCLI-159).
- Structured cache reports normalize file-path fields to POSIX form for
  cross-platform stability. `path:` and `manifest_path:` fields inside the
  YAML / JSON envelopes of `/cache refresh`, `/cache clear`, `/cache status`,
  and `/cache search` now use `/` separators on Windows the same as on macOS
  and Linux, so LLM consumers parsing the envelope see a platform-stable
  shape (KDCLI-168).

## [2.6.2] — 2026-04-28

### Added
- `--quiet` (long form only; no `-q` alias) suppresses the detail block on
  mutation commands that use the new write-output contract, leaving stdout as
  exactly the single-line ack `ok: <ID> <action> [key=value ...]`. Designed
  for shell loops that extract the new entity ID via
  `id=$(kd /issue create "..." --quiet | tail -1 | awk '{print $2}')`.
  `--quiet` is silently ignored on read commands and on structured modes
  (`-o yaml` / `-o json`).
- `docs/design/write-output-contract.md` documents the mutation envelope
  shape, action vocabulary, tail-line grammar, `--quiet` semantics, the
  multi-target `applied[]` shape, and the `/article push` dry-run-create
  carve-out.
- `docs/how-to/021-bulk-write-loops-how-to.md` shows the canonical
  `tail -1 | awk` extraction recipe for bulk migration scripts, plus
  recipes for the `/article push` slug capture and the
  `attachments[]`/`applied[]` shape on attachment-add commands.
- `attachments[]` plus `applied[]` shape on `/issue/attachment add` and
  `/article/attachment add`. The scalar `attachments` list produces the
  tail-line token (`attachments=file1.md,file2.md`) for shell loops; the
  structured `applied: [{name, size, id}, ...]` list carries per-file detail
  for machine consumers. The shape is consistent for both single-file and
  multi-file calls.

### Changed
- Every upstream-mutation command now returns a parseable single-line ack
  contract as the last line of stdout in text/md modes, except `/tag add`
  (reshaped with a multi-tag `applied[]` envelope in a separate session).
  The contracted set: `/issue` (`create`, `update`, `delete`, `assign`,
  `unassign`, `move`); `/issue/comment add`, `/issue/link add`,
  `/issue/attachment add`, `/issue/attachment delete`; `/article` (`create`,
  `update`, `delete`, `move`, `push`); `/article/comment add`,
  `/article/attachment add`, `/article/attachment delete`; `/tag`
  (`create`, `delete`, `rename`, `remove`); `/project` (`create`,
  `configure`, `delete`); `/user` (`create`, `delete`); `/query`
  (`create`, `update`, `delete`); `/time` (`log`, `update`, `delete`).
  Detail bodies that used to echo full entity state are replaced with a
  narrowed envelope (`{ok, action, id, ...}`) — run `kd /<entity> show
  ID` for the canonical post-write read. Update commands (`/issue update`,
  `/article update`, `/query update`, `/time update`) carry `changed:
  [...]` listing every key the call mutated.
- Action vocabulary expanded with `linked` (`/issue/link add`), `renamed`
  (`/tag rename`), `configured` (`/project configure`), and `noop` (the
  explicit "nothing happened" verb — today: `/tag create` for an
  already-visible tag).
- `/tag rename` envelope `id` is the **new name** (scripts that follow the
  rename refer to the tag by its new identity); the previous name is
  preserved in `old_name=`.
- `/project create` envelope `id` is the project **short_name**
  (operator-typed identifier); the internal numeric ID is dropped.
  `/user create` envelope `id` is the **login**; the internal numeric ID
  is dropped.
- `/article push` now maps SDK `PushResult.action` (`"create"` /
  `"update"`) to contract past-tense (`"created"` / `"updated"`) at the
  CLI boundary. Dry-run create has no upstream article yet — `id` is
  `null` in the envelope and the tail line is **not ID-extractable** via
  `tail -1 | awk '{print $2}'`. Drop `--dry-run` before relying on tail
  extraction. Production-create and production-update branches both
  carry the article ID as expected.
- Destructive deletes (`/tag delete`, `/project delete`, `/user delete`)
  retain their `--confirm` safety gate. The gate fires before any mutation
  envelope is rendered; the contract normalizes the success-path envelope
  only.
- Warning lines on `/primer/tag add` (and any other command attaching a
  `warnings` envelope) render to **stderr** in text/md modes so stdout stays
  parseable for `tail -1 | awk` recipes. Structured modes (`-o yaml`,
  `-o json`) leave `warnings` in the payload exactly as before — no
  double-emission to stderr.

## [2.6.1] — 2026-04-27

### Documentation
- `kd --sdlc` now opens with a reader-first mental model and vocabulary before
  the layer, cache, marker-bridged, and closeout rules. The packaged doctrine
  keeps the same tool contracts while making the SDLC easier for first-time
  human readers and LLM agents to explain.
- `kd --sdlc` now defines daybooks, surfaces the session starter checklist
  near the front, and clarifies the planning path from ideation through setup
  into the repeatable session loop.

## [2.6.0] — 2026-04-27

### Added
- `kd --sdlc` prints the portable SDLC doctrine from packaged resources;
  runs config-free and client-free, so an installed kd-cli can surface the
  method in any workspace without a tracker connection or repo checkout.
- `kd /cache refresh` can hydrate the local cache from primer scope, single
  issues, single articles, saved queries, milestone values, and epic children.
  Refresh writes disposable upstream-mirror files under
  `.kd/cache/connections/<slug>/`, records a per-scope `scope.yaml`, and returns
  structured hydrated/failed reports for partial failures.
- `kd /cache status` reports local cache health across all connection scopes,
  including per-kind counts, latest and oldest fetch timestamps, and stale-entry
  counts. `--scope NAME` focuses one connection; `--stale-after-days N`
  changes the stale threshold for that run.
- `kd /cache search TEXT` searches hydrated cache content locally, without a
  tracker token. Use `--scope`, `--kind article|issue|query`, and `--limit` to
  narrow the scan.
- `kd /cache clear` clears only the active connection scope by default and
  needs no live tracker token. `--all` wipes the whole `.kd/cache/` tree,
  including legacy flat cache directories.
- `kd /issue touched --git-range <A>..<B>` derives ticket IDs from local git
  commit subjects for offline reconciliation. It never reads the tracker and
  reports unmatched commits in structured output.
- `kd /article update` accepts `--append TEXT_OR_FILE` and
  `--prepend TEXT_OR_FILE` for one-call article body splices. Both forms
  preserve an existing `kd-meta` marker at content position 0, accept inline
  text, `@file`, `@-`, and `-`, and are mutually exclusive with `--content` and
  with each other.
- `kd /article push` uses cache sidecars for drift detection when they exist.
  Baseline priority is `cache > legacy_marker > none`; structured output
  reports `conflict_check: cache` when the cache drives the check. Malformed
  sidecars refuse before any upstream write, and successful pushes refresh the
  active article cache sidecar for the next run.

### Changed
- Cache files now live under per-connection scopes at
  `.kd/cache/connections/<slug>/` so workspaces that use multiple trackers do
  not collide on the same readable issue, article, or query names. New refreshes
  do not write the old flat `.kd/cache/articles|issues|queries` layout.
- `kd /article push` writes the local `kd-meta` marker only on first creation,
  first resolution, or stale-id repair. New markers contain only identity
  fields (`slug` and `upstream-id`); legacy `upstream-updated` lines are still
  accepted as a fallback drift baseline but are no longer emitted.
- `kd /article push` reports the update-path conflict policy through
  `conflict_check` in structured output. Identity-only files without a cache
  sidecar report `skipped_no_baseline` and proceed under the explicit trust-new
  policy; `--force` reports `skipped_force`.
- `kd /article pull` is now a passthrough read of the server article body. It
  no longer injects, refreshes, or canonicalizes marker fields.
- Upstream mutation commands are covered by a write-gating catalog test: they
  require a live tracker client before command execution begins, so cached
  content never substitutes for a mutation target.

### Documentation
- The article-push how-to, user guide, operating guide, and command reference
  now describe marker identity, cache-backed drift detection, cache health,
  cache search, offline reconciliation, write-gating, and article
  append/prepend behavior in current-tense operator guidance.
- The portable SDLC doctrine (`KDCLI-A-18`) is refined against the shipped
  M008 surface: legacy flat cache content is named neutrally rather than by
  internal milestone label; marker-bridged files are described as one source
  of truth in two forms; the two-commit pattern is recommended for sessions
  that mix code and docs and explicitly optional for docs-only or hygiene-only
  work; a new "What's project-specific?" section names the SDLC profile as
  the configurable-policy extension point.

## [2.5.2] — 2026-04-19

### Fixed
- `kd /article push` one-way migrations no longer force `git rm -f`. Before: push always rewrote the local markdown file with a `kd-meta` marker on success, which left the file "locally modified" and made `git rm` refuse without `-f` on migrations that delete the local copy after push. Now: pass `--no-marker-write` to skip the local rewrite. The upstream article is still created or updated, `--tag` values are still applied, and the outgoing server payload still carries a marker with the slug — so scan-by-slug resolves the article on any later push even without the local file. Default behavior is unchanged; the flag is opt-in (KDCLI-110).

### Added
- `kd /article push` structured output (`-o yaml` / `-o json`) now carries two sibling keys on every push: `marker_written` (`false` for dry-runs and `--no-marker-write` pushes, `true` on the default path after the local rewrite) and `parent` (readable id of the target article's upstream parent, or `null` for root articles). Scripts can assert local-file state and the article's place in the tree from the push response directly (KDCLI-110).

## [2.5.1] — 2026-04-19

### Fixed
- `kd /issue create --field NAME=VALUE` and `kd /issue update --field NAME=VALUE` now accept **string custom fields**. Before: `kd /issue update VIS-21 --field Milestone=M039` rejected with `Error: Unsupported field type 'SimpleProjectCustomField' for field 'Milestone'`, blocking workflows that track freeform per-issue metadata such as a rolling milestone string. `--field "NAME="` clears an optional string field just like it clears enum/user fields today; required string fields reject empty values with the same client-side error shipped in 2.2.2. Whitespace inside the value is preserved — `--field "Milestone=  M 039  "` sends `  M 039  ` verbatim. Only the field **name** is trimmed (KDCLI-106).

### Changed
- Unsupported custom-field errors now name both identifiers and the cause. Before: `Error: Unsupported field type 'X' for field 'Y'`. Now: `Error: Unsupported custom field type 'X' with subtype 'Y' for field 'Z'. kd does not know how to build a safe value payload for this YouTrack field family.` Applies to YouTrack text fields, multi-arity enum/user variants, and integer/float/date simple-field subtypes — paste the message into a bug report and the two type identifiers are already there (KDCLI-106).

## [2.5.0] — 2026-04-19

Dogfooding friction fixes across tag and article workflows, plus self-describing `/system info` output for bug reports. Focus of this release: the taxonomy and article-create loops now behave the way field use expects, and a paste of `kd /system info -o yaml` replaces hand-typed version footers.

### Added
- `kd /system info` now carries three additional fields alongside the existing connection identity: `kd_version` (matches `kd --version`), `python_version`, and `platform`. Paste `kd /system info -o yaml` into a bug report to tie an observed behavior to a specific build without hand-typing the version. The four existing keys (`url`, `user`, `name`, `email`) are unchanged; redact `url` / `email` before sharing externally (KDCLI-104).

### Changed
- `kd /tag create TAG_NAME` is now idempotent and returns a `{ok, action, tag}` envelope with `action` being either `create` or `exists`. Re-running create on an already-visible tag is safe and no longer surfaces `Tag.name-is-invalid`. **Breaking vs. the prior flat tag object** — downstream scripts and SDK consumers that parsed the top-level tag fields must now read from the nested `tag:` block. Python SDK: the `create(...)` method on the tag operations client returns a `TagCreateResult(action, tag)` object; unpack `.tag` to reach the tag fields (KDCLI-105).

### Fixed
- `kd /tag create TAG_NAME` against an already-existing tag no longer emits the raw upstream `Tag.name-is-invalid` error. When the POST still fails (genuinely unacceptable name, or a tag hidden by sharing/visibility that the caller cannot see), the error names both plausible causes without inventing a character grammar (KDCLI-105).
- `kd /tag entities` structured output (`-o yaml` / `-o json`, including `--primer-scope`) now populates the `tags` field on issue rows, bringing the verb to parity with `/issue show --rich` and `/issue list --with-tags`. Before: structured issue rows reported `tags: []` even when they matched via a pinned tag, forcing a second `/issue show` round-trip to see which tag caused the match. Table output is unchanged and still omits the tag column by design (KDCLI-100).
- `kd /article create --tag TAG` structured output now returns the post-tag article state: the response `tags` field matches a subsequent `kd /article show` on the returned id. Missing tags are rejected **before** the article is created, so a typo in `--tag` no longer leaves an untagged article behind; the error names `kd /tag create` for the missing tag, retrying `kd /article create` without the tag, or applying the tag after create via `kd /tag add ARTICLE-ID TAG`. Before: the CLI rendered `tags: ''` even when tags were attached upstream (KDCLI-101).

## [2.4.1] — 2026-04-19

### Fixed
- Concurrent `kd /primer/note`, `/primer/tag`, and `/primer/article` commands against the same `.kd/primer.yaml` no longer silently lose writes. Before: running three `kd /primer/note update` calls in parallel (common in agent-driven sessions) all reported `ok: true` but only one update survived. Now: each command runs inside an advisory file lock on a sidecar `.kd/primer.yaml.lock`, so concurrent kd processes queue rather than clobber each other and every successful call is reflected in the final file. Concurrent acquirers wait up to 30 s before erroring with `Another kd process is writing to …; retry in a moment`; the error includes the path of the sidecar lockfile for stale-holder recovery. POSIX only; first lock attempt on Windows raises a clear error pointing at sequential invocation (KDCLI-98).

## [2.4.0] — 2026-04-19

### Added
- `kd /primer init [TEMPLATE]` scaffolds `.kd/primer.yaml` from a packaged template so fresh workspaces don't start from a blank page. The default `hitl` template ships placeholder prose in three slots (scope guard, session workflow, canonical reference); the banner and example-shape comments are preserved verbatim. Refuses if a primer already exists; unknown template names reject with the available-templates list (KDCLI-97).
- `--confirm` flag on `/project delete`, `/tag delete`, and `/user delete` — these three deletes bypass YouTrack's Deleted Items and require a backup restore to undo, so kd now rejects them without `--confirm` and prints the literal retry command in the error. The flag is hidden from `kd --reference` but visible in per-command `--help` (KDCLI-96).

## [2.3.0] — 2026-04-18

### Added
- `/article push --create-tags` explicitly creates missing tags before pushing and attaching them, for scripted flows that own their tag namespace (KDCLI-85).

### Fixed
- `/article push --tag` missing-tag failures now print actionable recovery commands instead of a bare not-found / close-match message (KDCLI-85).

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
