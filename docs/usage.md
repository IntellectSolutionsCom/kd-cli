# kd Usage Guide

A scenario-driven guide to daily work with kd. Covers the workflows that replace the YouTrack web UI for terminal and LLM-based usage.

## Setup

### Install

```bash
pip install -e .
```

### Configure

kd uses named connections to manage YouTrack instances. Each connection carries a URL, auth source, and optional defaults. Scaffold the global config with:

```bash
kd init --global https://your-instance.youtrack.cloud
```

This creates `~/.config/kd/config.yaml` (and its parent directory) with the URL pre-filled. Edit it to customize:

```yaml
default_connection: main

connections:
  main:
    provider: youtrack
    url: https://your-instance.youtrack.cloud
    token_env: KD_TOKEN
    default_project: FB
```

Then export the token:

```bash
export KD_TOKEN=perm:your-api-token
```

For secret managers, use `token_cmd` instead:

```yaml
connections:
  main:
    url: https://your-instance.youtrack.cloud
    token_cmd: gopass show -o infra/services/youtrack/api-token
```

For project-specific config, initialize a local `.kd/config.yaml`:

```bash
cd your-project
kd init https://your-instance.youtrack.cloud
```

Select a different connection per-invocation with `-c`/`--connection`:

```bash
kd -c upstream /issue list
kd --connection corp /project list
```

`-c` wins over `KD_CONNECTION` (env var), which wins over the
`default_connection:` key in config.

### Multiple instances

Add more connections to switch between YouTrack instances:

```yaml
default_connection: corp

connections:
  corp:
    url: https://yt-onprem.example.com
    token_cmd: gopass show -o infra/services/corp-youtrack/api-token
    default_project: BCO
    compatibility:
      target: 2023.2
      mode: pinned

  upstream:
    url: https://client.youtrack.cloud
    token_env: YT_UPSTREAM_TOKEN
    default_project: CUST
    output: json
```

### Config sections

Beyond connections and flat keys, the config file supports three structured sections:

```yaml
# Per-command defaults — set options you always use
defaults:
  issue_list:
    limit: 100
    query: "for: me #Unresolved"
  issue_show:
    detail: rich

# Per-project overrides — field aliases and scoped defaults
projects:
  FB:
    field_aliases:
      state: Stage
      type: Kind
    defaults:
      issue_list:
        fields: id,summary,state,priority,assignee

# Command aliases — named shortcuts for full commands
aliases:
  my-bugs: "/issue list --query 'for: me Type: Bug #Unresolved'"
  standup: "/issue list --query 'for: me updated: Today'"
```

All three sections are optional. They work in both local and global config, with local overriding global. See [Configuration](configuration.md) for the full resolution chain, nested directory behavior, and collision rules.

### Verify

```bash
kd /system info
```

## Discover Commands

kd is organized into namespaces (`/issue`, `/article`, `/system/config`). Type a namespace without a command to see what's available:

```bash
kd /issue
```

```
kd /issue

Namespaces:
  attachment      Issue attachments
  comment         Issue comments
  link            Issue links

Commands:
  assign          Assign an issue to a user
  create          Create an issue
  list            List issues
  show            Show issue details
  ...
```

Type the command name explicitly to run it — bare namespace invocations always show discovery.

`--help` (`-h`) works at every level:

```bash
kd --help                # root — all namespaces, global + output options
kd /issue --help         # namespace — commands and sub-namespaces
kd /issue list --help    # command — usage, options, examples
```

Three top-level commands are dedicated to discovery:

```bash
kd /primer show              # pinned notes + references for the current repo
kd /system/config keys       # every configurable path with type + default
kd /release-notes            # what changed in the installed version
```

## Output Formats

All commands support `--output` (`-o`) to control formatting:

```bash
kd /issue list --project FB                  # default table
kd /issue list --project FB --output json    # JSON
kd /issue list --project FB --output yaml    # YAML
kd /issue list --project FB --output md      # Markdown table
```

A handful of rendering behaviors are worth knowing about:

- **Empty lists render as `(no entries)` in table mode** so you can tell success-with-zero-results from silent failure. JSON/YAML/Markdown still emit `[]` for structured consumers.
- **Long content fields in detail views render as unindented blocks** under a `key:` label line, not padded by the label column. This keeps `/article show` and `/issue show` markdown bodies copy-paste clean regardless of length.
- **YAML output preserves non-ASCII characters literally** (em-dashes, smart quotes, Cyrillic, CJK). No `\uXXXX` escaping — useful for round-tripping typography and multi-script content.
- **Mutation commands return structured payloads** in JSON/YAML mode. For example, `kd /issue assign FB-42 alice --output json` emits `{"ok": true, "action": "assign", ...}` instead of prose. Works the same for delete, tag, comment, and every other mutating command. Scripts and LLM callers should rely on the `ok` field.
- **TTY detection is off by design.** Piping (`| cat`, `| jq`) does **not** auto-switch to JSON. Your configured `output:` default wins; pass `-o json` explicitly when you want structured output. This matches `gh`, `kubectl`, and `aws` conventions.

### Fields filter

`--fields` (`-f`) projects the output down to specific columns (table/md) or keys (json/yaml):

```bash
kd /issue list --project FB --fields id,summary,state
kd /project list --output json --fields key,name,leader
```

## Find My Work

List issues assigned to you, in a project, or matching a query:

```bash
# All issues in a project
kd /issue list --project FB

# My open issues
kd /issue list --query "for: me #Unresolved"

# Recently updated in a project
kd /issue list --project FB --query "updated: Today"

# Limit results
kd /issue list --project FB --limit 10

# Combine project and query
kd /issue list --project FB --query "State: {In Progress}"
```

## Read an Issue

Rich view is the default — shows description, comments, links, and attachments:

```bash
# Full view (default)
kd /issue show FB-42

# Header and custom fields only
kd /issue show FB-42 --brief

# Include only specific sections
kd /issue show FB-42 --brief --with comments

# Exclude noisy sections
kd /issue show FB-42 --without comments,attachments
```

Sections available for `--with` / `--without`: `description`, `comments`, `links`, `attachments`.

## Create an Issue

```bash
# Simple
kd /issue create "Fix login timeout" --project FB

# With type and priority
kd /issue create "Add dark mode" -p FB --type Feature --priority Major

# With description from a file
kd /issue create "Redesign auth flow" -p FB --description @design.md

# With inline description
kd /issue create "Quick fix" -p FB --description "One-liner fix for the null check"

# Create and attach files in one step
kd /issue create "Bug report" -p FB --attach screenshot.png
kd /issue create "Crash analysis" -p FB --attach log.txt --attach trace.json

# Set arbitrary custom fields
kd /issue create "Sprint task" -p FB --field "Sprint=Sprint 47" --field "Story Points=5"
```

The `--attach` and `--field` options are repeatable — use them multiple times for multiple values.

### Custom fields with `--field`

The `--field NAME=VALUE` option sets any project custom field by name. Field names are matched case-insensitively with fuzzy suggestions on typos:

```bash
# Exact name
kd /issue create "Task" -p FB --field "Priority=Critical"

# Case doesn't matter for values — auto-corrected
kd /issue create "Task" -p FB --state open          # sends "Open"

# Typo in field name → helpful error
kd /issue create "Task" -p FB --field "Priorty=Normal"
# Error: Unknown field 'Priorty'. Did you mean: Priority?

# Invalid value → shows allowed values
kd /issue create "Task" -p FB --state Bogus
# Error: Invalid value 'Bogus' for field 'State'. Allowed values: Submitted, Open, ...
```

User fields (like Assignee) accept logins directly — no need to use `/issue assign` separately:

```bash
kd /issue create "Task" -p FB --field "Assignee=admin"
kd /issue update FB-42 --field "Assignee=elena"
```

Use `--type`, `--priority`, `--state` for the common fields, and `--field` for project-specific fields like Sprint, Story Points, or Subsystem.

### Inspect available fields

```bash
kd /project/field list FB
```

Shows all custom fields for a project with their types and allowed values.

## Update an Issue

```bash
# Change summary
kd /issue update FB-42 --summary "Updated title"

# Change state and priority
kd /issue update FB-42 --state "In Progress" --priority Critical

# Replace description from file
kd /issue update FB-42 --description @revised.md

# Replace description from stdin
cat revised.md | kd /issue update FB-42 --description @-

# Update arbitrary custom fields
kd /issue update FB-42 --field "Sprint=Sprint 48"
kd /issue update FB-42 --field "State=Fixed" --field "Priority=Minor"
```

## Assign and Reassign

kd resolves user references flexibly — by login, email, display name, or `me`:

```bash
# Assign to yourself
kd /issue assign FB-42 me

# Assign by login
kd /issue assign FB-42 alice

# Assign by email
kd /issue assign FB-42 alice@example.com

# Remove assignee
kd /issue unassign FB-42
```

If the reference is ambiguous, kd shows candidates with login, name, and email to help disambiguate.

### Discover assignable users

```bash
kd /issue assignees
kd /issue assignees --match alice
```

## Attachments

### Upload files

```bash
# Single file
kd /issue/attachment add FB-42 screenshot.png

# Multiple files
kd /issue/attachment add FB-42 screenshot.png error.log report.pdf
```

### List and inspect

```bash
# List all attachments on an issue
kd /issue/attachment list FB-42

# Show metadata for one attachment
kd /issue/attachment show FB-42 12-3
```

### Download

```bash
# Download to current directory
kd /issue/attachment download FB-42 12-3

# Download to a specific directory
kd /issue/attachment download FB-42 12-3 --to ./artifacts
```

### Delete

```bash
kd /issue/attachment delete FB-42 12-3
```

## Comments

```bash
# Add inline comment
kd /issue/comment add FB-42 "Verified on staging"

# Add comment from a file
kd /issue/comment add FB-42 @review-notes.md

# Pipe from stdin
echo "Automated test passed" | kd /issue/comment add FB-42 -

# List comments
kd /issue/comment list FB-42
```

## Text Input Conventions

Any command that accepts text supports three input modes:

| Syntax | Source |
|--------|--------|
| `"plain text"` | Literal string |
| `@path/to/file.md` | Read from file |
| `-` or `@-` | Read from stdin |

This applies to `--description`, comment text, and other text-accepting arguments.

## Move an Issue

Re-route an issue to a different project when triage reveals it belongs elsewhere:

```bash
kd /issue move FB-42 BACKEND
```

The issue is re-keyed automatically (e.g., `FB-42` becomes `BACKEND-15`).

## Issue Links

```bash
# List links
kd /issue/link list FB-42

# Create links
kd /issue/link add FB-42 subtask-of FB-1
kd /issue/link add FB-42 relates-to FB-43
kd /issue/link add FB-42 depends-on FB-10
```

Available link types: `subtask-of`, `parent-for`, `depends-on`, `required-for`, `duplicates`, `duplicated-by`, `relates-to`.

## Time Tracking

The `/time` namespace covers the full work-item lifecycle on issues: log (with explicit date), list (with date filters), update, and delete.

### Log work

```bash
# 30 minutes, dated today
kd /time log FB-42 30

# With type and description
kd /time log FB-42 60 --type Development --description "Auth refactor"

# Backfill for a past date (YYYY-MM-DD, strict)
kd /time log FB-42 90 --date 2025-12-31 --type Development --description "Feasibility study"
```

Work types: Development, Testing, Documentation, Investigation, Implementation. Type names are case-insensitive.

### List work items

```bash
# All entries on an issue
kd /time list FB-42

# Single-day filter
kd /time list FB-42 --date 2026-01-02

# Inclusive date range (e.g. weekly / monthly reports)
kd /time list FB-42 --from 2026-03-01 --to 2026-03-31
```

`--date` is mutually exclusive with `--from`/`--to`. `--from` and `--to` must be supplied together and `--from` must not be later than `--to` (catches the common typo before it silently returns an empty list). Dates render as `YYYY-MM-DD` in every output format.

### Update a work item

Find the work-item id via `/time list`, then edit any combination of fields:

```bash
# Fix a wrong date
kd /time update FB-42 199-88 --date 2026-01-02

# Bump duration and rewrite the note
kd /time update FB-42 199-88 --minutes 90 --description "Extended scope to JWT rotation"

# Change everything in one call
kd /time update FB-42 199-88 --minutes 45 --date 2026-01-03 --type Testing --description "re-verified"
```

At least one of `--minutes`, `--date`, `--type`, `--description` must be supplied.

### Delete a work item

```bash
kd /time delete FB-42 199-88
```

No confirmation prompt — pair with `/time list` for audit if needed.

The time-tracking workflow above covers the essentials; end-of-week backfills and totals-by-type are straightforward combinations of the commands shown.

## User Management

```bash
# List users
kd /user list

# Create a user (Hub API)
kd /user create testbot "Test Bot" testbot@example.com

# Delete a user by Hub ID
kd /user delete abc-123
```

## Project Management

```bash
# List projects (columns: key, name, leader, archived)
kd /project list

# Show project details
kd /project show FB

# Create a project (leader defaults to the current user)
kd /project create FB "Franco Bet"

# Enable time tracking
kd /project configure FB --time-tracking true

# Delete a project
kd /project delete FB
```

## Knowledge Base Articles

Articles are YouTrack's built-in documentation pages. They support Markdown content and a tree hierarchy (parent/child). Unlike issues, articles have no custom fields and no search query support — filtering is project-scoped only.

### List articles

When `default_project` is set in config, `--project` can be omitted — the config value is used as a fallback. This applies to `article list`, `article push`, and `tag articles` as well as `issue list`.

```bash
# All articles in a project
kd /article list --project KDCLI

# All articles across projects (or scoped to default_project if configured)
kd /article list

# Limit results
kd /article list --project KDCLI --limit 10

# Direct children of a parent article (alternative to /article children)
kd /article list --parent KDCLI-A-1
```

Default columns: `id`, `summary`, `project`, `reporter`, `parent`, `created`, `updated`, `children`. The `parent` column shows the readable id (e.g. `KDCLI-A-1`) of the parent article, or empty for root-level articles. When `--parent` is set, `--project` is ignored because child listing is already project-scoped.

### Read an article

```bash
kd /article show KDCLI-A-1
```

The full Markdown content is included in the output. Long content bodies render as an unindented block under a `content:` label line — copy-paste friendly.

### Pull raw content

`pull` writes only the article body to stdout, with no framing or metadata. Use it for scripting, backups, and migration round-trips:

```bash
# Print to stdout
kd /article pull KDCLI-A-1

# Redirect to a file
kd /article pull KDCLI-A-1 > setup.md

# Pipe into another tool
kd /article pull KDCLI-A-1 | pandoc -f markdown -t html > setup.html
```

`pull` is intentionally format-agnostic: the `--output` flag has no effect. It fails loudly if the article has no content. Contrast with `show`, which includes metadata (title, reporter, timestamps) alongside the body.

### Create an article

```bash
# With inline content
kd /article create "Setup Guide" --project KDCLI --content "# Getting Started"

# From a Markdown file
kd /article create "Migration Guide" -p KDCLI --content @migration.md

# From stdin
echo "# Draft" | kd /article create "Draft" -p KDCLI --content -

# Title only (no content)
kd /article create "Placeholder" -p KDCLI

# Create and attach to an existing parent in a single call
kd /article create "Sub Guide" -p KDCLI --parent KDCLI-A-1 --content @sub.md
```

Article IDs use the `{PROJECT}-A-{number}` format (e.g., `KDCLI-A-1`), distinct from issue IDs.

`--parent` avoids the create-then-move two-step that migration scripts otherwise need. It accepts a readable id (`KDCLI-A-1`) and attaches the new article under that parent in one API call.

### Update an article

Updates are full replacements — there is no merge or diff. To edit an existing article, fetch the content, modify it locally, then push the updated version back:

```bash
# Change the title
kd /article update KDCLI-A-1 --summary "New Title"

# Replace content from file
kd /article update KDCLI-A-1 --content @revised.md

# Update both
kd /article update KDCLI-A-1 --summary "New Title" --content @revised.md
```

### Push (idempotent upsert from a file)

`push` is the file→YouTrack migration primitive. It takes a local Markdown file and makes sure there is exactly one YouTrack article that corresponds to it — creating on first run, updating on re-runs, refusing to clobber remote edits by default.

```bash
# First push — creates the article, rewrites the local file with a marker
kd /article push docs/infra/dns-setup.md --project KDCLI

# Re-push after editing the local file — updates the same article
kd /article push docs/infra/dns-setup.md --project KDCLI

# Preview without writing anywhere (exits 1 on would-conflict)
kd /article push docs/infra/dns-setup.md --project KDCLI --dry-run

# Override the remote-edit conflict check
kd /article push docs/infra/dns-setup.md --project KDCLI --force

# Attach a new article under a parent in a single call (first push only)
kd /article push sub-guide.md --project KDCLI --parent KDCLI-A-1
```

**How identity works.** On first push, `push` synthesizes an HTML comment marker at the top of the file with three keys — `slug`, `upstream-id`, `upstream-updated` — and rewrites the file on disk with the marker embedded. The same marker lives inside `article.content` on the YouTrack side. On re-runs, `push` reads the marker to find the right article. The slug survives title renames on either side; the upstream-id cache makes re-pushes a single API call.

```markdown
<!-- kd-meta
slug: infra/dns-setup
upstream-id: KDCLI-A-12
upstream-updated: 2026-04-05T13:34:22Z
-->
# DNS Configuration

Real article content starts here...
```

**The file is owned by push.** After the first successful push, the local file is no longer byte-identical to what you wrote — it has a marker at the top. Treat the file like `Cargo.lock`: commit the marker alongside your content changes. `push` rewrites the file on every successful run to refresh the marker's `upstream-updated` timestamp.

**Conflict safety.** If someone edits the article in the YouTrack web UI between two pushes, the next `push` refuses with a clear error showing both timestamps:

```
Error: Article KDCLI-A-12 was updated upstream at 2026-04-05T14:02:10Z after our last push at 2026-04-05T13:34:22Z. Pass --force to overwrite, or inspect the remote: kd /article pull KDCLI-A-12.
```

Resolve by pulling the remote, merging by hand, and re-pushing. Or pass `--force` if you know the remote changes are discardable.

**The article title** comes from the first `# H1` line in the body, unless you pass `--summary "Custom Title"`. The H1 stays in the body — it is not stripped.

**The slug** defaults to the file path relative to the current directory, minus the `.md` extension (`docs/infra/dns-setup.md` → `docs/infra/dns-setup`). Override with `--slug on-first-push` before the marker exists; after the marker exists, the slug is immutable unless you edit the marker directly.

**Scope caveats.** `push` is single-file only in M029. Directory tree push, two-phase delete (tombstones), and attachment push are reserved for later milestones. The marker format reserves keys for all three so older kd versions keep round-tripping files written by newer ones.

### Delete an article

```bash
kd /article delete KDCLI-A-1
```

### Browse child articles

```bash
kd /article children KDCLI-A-1
```

Lists direct children of an article (one level, not recursive).

### Tree view

Render the full article hierarchy with box-drawing characters:

```bash
# Default depth (5 levels)
kd /article tree KDCLI-A-1

# Limit depth
kd /article tree KDCLI-A-1 --depth 2
```

Output:

```
KDCLI-A-1: Setup Guide
├── KDCLI-A-2: Prerequisites
├── KDCLI-A-3: Configuration
│   ├── KDCLI-A-6: Database Setup
│   └── KDCLI-A-7: Auth Config
└── KDCLI-A-8: First Run
```

Each level requires an API call to fetch children, so deep trees may be slow.

### Move an article

Change an article's parent to reorganize the tree:

```bash
# Move under a new parent
kd /article move KDCLI-A-3 --parent KDCLI-A-1

# Move to project root (no parent)
kd /article move KDCLI-A-3 --parent none
```

### Article comments

```bash
# List comments
kd /article/comment list KDCLI-A-1

# Add inline comment
kd /article/comment add KDCLI-A-1 "Reviewed and approved"

# Add from file
kd /article/comment add KDCLI-A-1 @review.md
```

### Article attachments

```bash
# Upload
kd /article/attachment add KDCLI-A-1 diagram.png

# List
kd /article/attachment list KDCLI-A-1

# Show metadata
kd /article/attachment show KDCLI-A-1 att-1

# Download
kd /article/attachment download KDCLI-A-1 att-1
kd /article/attachment download KDCLI-A-1 att-1 --to ./docs

# Delete
kd /article/attachment delete KDCLI-A-1 att-1
```

## Tags

Tags are shared entities that span issues and articles. The `/tag` namespace manages tags and applies them to entities.

### List and inspect tags

```bash
# All tags visible to you
kd /tag list

# Tag details (owner, color, untag-on-resolve)
kd /tag show sprint-47
```

### Create, rename, delete

```bash
kd /tag create "sprint-47"
kd /tag rename sprint-47 sprint-48
kd /tag delete sprint-48
```

### Tag and untag entities

The `add` and `remove` commands auto-detect the entity type from the ID format — `FB-123` is an issue, `FB-A-1` is an article. No `--type` flag needed.

```bash
# Tag an issue
kd /tag add FB-123 sprint-47

# Tag an article
kd /tag add FB-A-1 reviewed

# Remove a tag
kd /tag remove FB-123 sprint-47
```

### Find tagged entities

```bash
# Issues by tag (server-side query via tags API)
kd /tag issues sprint-47

# Articles by tag (client-side scan — YouTrack has no server-side endpoint)
kd /tag articles "type:case"
kd /tag articles "type:case" --project CMCSA --limit 100

# Multi-tag intersection (AND semantics) — repeat --tag for additional filters
kd /tag articles "type:case" --tag "component:authservice" -p CMCSA

# Alternative for issues: use YouTrack query syntax
kd /issue list --query "#sprint-47"
```

## Saved Queries and Query Assist

The `/query` namespace manages YouTrack saved queries (saved searches) and wraps the search-autocomplete endpoint. Use it to name recurring search strings, run them on demand, and discover valid query tokens interactively.

### List saved queries

```bash
# Everything visible to you (system + personal + shared)
kd /query list

# Only queries you own
kd /query list --mine

# Raise the page size
kd /query list --limit 200
```

`--mine` hits `/api/users/me/savedQueries` under the hood — no need to know your own user ID.

### Create a saved query

```bash
kd /query create "My Bugs" "for: me Type: Bug #Unresolved"
kd /query create "Sprint 47" "Sprint: {Sprint 47} Assignee: me"
```

Two positional arguments: the name and the YT query string. New queries are owned by you; share them via the YouTrack web UI if the team needs access. The query is validated after saving — a warning is printed if the syntax is invalid.

Values containing colons, spaces, or braces must be wrapped in braces in YT query syntax:

```bash
# tag: type:case  → fails (YouTrack reads "type" as a field name)
# tag: {type:case} → correct
kd /query create "Cases" "project: CMCSA tag: {type:case}"
```

### Show, update, delete

```bash
# Show (by name or ID; names are case-insensitive)
kd /query show "My Bugs"
kd /query show 11-24

# Partial update — pass --name, --query, or both
kd /query update "My Bugs" --name "My Bugs (archived)"
kd /query update 11-24 --query "for: me Type: Bug State: Open"
kd /query update 11-24 --name "Open Bugs" --query "for: me Type: Bug State: Open"

# Delete (no confirmation)
kd /query delete "My Bugs"
```

At least one of `--name` / `--query` must be supplied on `update`. Omitted fields are preserved upstream. Duplicate names fail loudly with the conflicting IDs — use the ID to disambiguate.

### Run a saved query

`run` resolves the name, fetches the query text, and pipes it through `/issue list`, so the output columns match the regular issue listing:

```bash
kd /query run "My Bugs"
kd /query run "Sprint 47" --limit 100
kd /query run 11-24 --output json
```

### Validate saved queries

Check whether saved queries have valid syntax by executing a dry-run search:

```bash
# Validate one query
kd /query validate "My Bugs"

# Validate all saved queries (reports ok/error per query)
kd /query validate
```

### Query Assist — autocomplete for query strings

Wraps `POST /api/search/assist`. Pass the current query string and optionally a caret position (defaults to the end of the input) to get back suggested completions:

```bash
# "What can follow 'project: '?"
kd /query assist "project: "

# Explicit caret (e.g. mid-string completion)
kd /query assist "project: ab" --caret 9

# "What can follow 'for: '?"
kd /query assist "for: "
```

Each suggestion surfaces `option`, `description`, optional `prefix` / `suffix` characters (e.g. `{` / `}` for project names with spaces), and the caret position the replacement should land at. Use it to discover valid tokens before baking a query into `/query create`, or pipe `--output json` into an editor-completion helper.

Combine `/query create` with `/alias` (see the Command Aliases section below) to pin a short CLI name to the saved query's numeric ID.

## Cross-Entity Search

`/search` is an aggregated search that fans out to multiple YouTrack APIs and returns a unified result set. It covers issues, users, tags, projects, and articles in a single call.

```bash
# Search across all entity types
kd /search "router"

# Restrict to specific types
kd /search "router" --types issue,article

# Search users
kd /search "alice" --types user

# Scope issues and articles to a project
kd /search "dns" --project FB

# Limit results
kd /search "router" --limit 10

# JSON output
kd /search "youtrack" -o json
```

Per-type behavior:

| Type | Mechanism |
|------|-----------|
| issue | Server-side query via `GET /api/issues?query=...` |
| user | Server-side query via `GET /api/users?query=...` |
| tag | Client-side case-insensitive substring match on tag name |
| project | Client-side substring match on project name or short name |
| article | Client-side substring match on article summary |

Results are interleaved round-robin across types so mixed searches surface variety rather than being dominated by one type. `--project` scopes issue queries (prepends `project: P`) and article scans (uses project-scoped listing).

This is a composed client-side operation, not a wrapper over a single upstream "global search" endpoint — the YouTrack public REST API does not expose one. See [Proposal 22](../proposals/22-search-proposal.md) for the design rationale.

## Project Primer

The primer is a local knowledge capsule stored in `.kd/primer.yaml`. It captures project-specific context — conventions, key article references, do's and don'ts — that an LLM (or human) reads once to understand what matters in this project.

### Show the primer

```bash
kd /primer show
kd /primer show --output json
```

Returns the full primer (notes + articles). If no primer exists yet, returns empty lists — not an error.

### Notes — conventions and do's/don'ts

```bash
# Add notes
kd /primer/note add "Sprint field is called 'Board' in this project"
kd /primer/note add "Never close issues tagged 'Evergreen'"

# From a file
kd /primer/note add @conventions.txt

# List (indexed)
kd /primer/note list

# Remove by index
kd /primer/note remove 2

# Remove all notes
kd /primer/note remove --all
```

### Pinned articles — key references

```bash
# Pin an article (optional summary)
kd /primer/article add FB-A-42 --summary "Architecture Overview"

# List
kd /primer/article list

# Unpin
kd /primer/article remove FB-A-42
```

### LLM workflow

An LLM agent uses two commands to orient itself in a project:

```bash
kd --reference              # learn what the tool can do
kd /primer show --output json   # learn what matters here
```

The primer is separate from config — config is tool settings (URLs, defaults), primer is project knowledge. The file is created lazily on first `add`; read commands work without it. Unlike config, the primer is **local only** — there is no global primer at `~/.config/kd/`. Each project has its own primer or none at all.

See [Primer](primer.md) for the full workflow and troubleshooting.

## Config and Primer History

kd automatically snapshots config and primer files before every write, so accidental changes are always recoverable.

### List snapshots

```bash
kd /system history              # both config and primer
kd /system history config       # config only
kd /system history primer       # primer only
```

Each entry shows the timestamp, file size, and lines changed relative to the current file.

### Compare snapshots

```bash
# Snapshot vs current file
kd /system diff config 2026-04-09T10-30-00

# Two snapshots against each other
kd /system diff config 2026-04-01T10-00-00 2026-04-09T10-30-00
```

### Restore a snapshot

```bash
kd /system restore config 2026-04-09T10-30-00
kd /system restore primer 2026-04-09T14-20-00
```

The current file is snapshotted before restoring, so restore is itself undoable. Snapshots are stored in `.kd/history/` with ISO 8601 timestamps. Retention defaults to 20 per file; configure via `history.retention` in config:

```yaml
history:
  retention: 50
```

## Command Aliases

Define named shortcuts in config for commands you run often. Add an `aliases:` section to `.kd/config.yaml` or `~/.config/kd/config.yaml`:

```yaml
aliases:
  my-bugs: "/issue list --query 'for: me Type: Bug #Unresolved'"
  standup: "/issue list --query 'for: me updated: Today'"
  fields: "/project field list FB"
```

Run an alias:

```bash
kd my-bugs
```

The explicit form through the alias namespace also works: `kd alias my-bugs`.

Override options by appending flags:

```bash
kd my-bugs --limit 5 --output json
```

Inspect defined aliases:

```bash
kd /alias list          # list all aliases
kd /alias show my-bugs  # show expansion for one alias
```

Aliases live in the `aliases:` section of your config. See [Configuration](configuration.md) for how they compose with connection-level defaults.

## Command Synonyms

A handful of built-in synonyms bridge vocabulary mismatches between HTTP/SDK conventions and CLI conventions. These work everywhere without any configuration:

| You type | Runs |
|----------|------|
| `kd /issue get FB-42` | `kd /issue show FB-42` |
| `kd /issue ls` | `kd /issue list` |
| `kd /article rm KDCLI-A-1` | `kd /article delete KDCLI-A-1` |

Synonyms resolve silently — they don't appear in `--help` or discovery output, and canonical names are always preferred. Where a canonical command already uses the synonym name (e.g., `/system/config get`), the canonical command wins.

User aliases (see above) take precedence over built-in synonyms.

## Diagnostics

When something is off, progressively raise the debug level with `-D`:

```bash
kd -D    /issue list --project FB      # info: resolved config sources, API call summaries
kd -DD   /issue list --project FB      # debug: full option resolution, per-request timings
kd -DDD  /issue list --project FB      # trace: HTTP bodies, field sets, correlation IDs
```

All diagnostic output goes to **stderr** so stdout stays clean for piping. Tokens are redacted (`perm-YWR***`) at every level. Use `-DDD` when filing a bug report or investigating an unexpected upstream response — trace mode logs failed response bodies so you can see exactly what YouTrack sent back.

`--debug` is the long form of `-D` (single level only). Stacking only works with the short form.

## Quick Reference

```bash
kd --help          # namespaces, global options, output options
kd /issue --help   # commands in a namespace
kd --reference     # full machine-readable command listing (LLM-friendly)
kd /system info    # verify URL, token, and connectivity
```

---
Updated: 2026-04-14 (kd-cli 2.1.0)

