# How-To: Configuration

kd uses YAML config files with named connections and a hierarchical resolution chain. This guide covers the connection model, file layout, all config sections, and what happens when values collide.

## Config files

kd reads from two config files:

| File | Scope | Created by |
|------|-------|------------|
| `~/.config/kd/config.yaml` | Global -- applies everywhere | `kd init --global [URL]`, manual edit, or `kd /system/config set` |
| `.kd/config.yaml` | Local -- applies to this directory tree | `kd init [URL]` |

Both are optional. If neither exists, kd falls back to built-in defaults. At least one connection must be configured for commands that talk to YouTrack. `kd init --global` creates the parent directory as needed; `kd init` (local) additionally adds `.kd/` to `.gitignore` at the nearest git root.

## Connections

A connection is a named working context: URL, auth, defaults, and compatibility for one YouTrack instance. The `connections:` section is the primary config block:

```yaml
default_connection: corp

connections:
  corp:
    provider: youtrack
    url: https://yt-onprem.example.com
    token_cmd: gopass show -o infra/services/corp-youtrack/api-token
    default_project: BCO
    output: table
    limit: 50
    defaults:
      issue_list:
        query: "for: me #Unresolved"
    compatibility:
      target: 2023.2
      mode: pinned

  upstream:
    provider: youtrack
    url: https://client.youtrack.cloud
    token_env: YT_UPSTREAM_TOKEN
    default_project: CUST
    output: json
```

### Auth sources

Secrets never live in config files. Each connection resolves its token from:

1. **`token_cmd`** -- shell command that prints the token to stdout. First choice for secret managers:
   ```yaml
   token_cmd: gopass show -o infra/services/youtrack/api-token
   ```

2. **`token_env`** -- environment variable name. For CI pipelines and scripting:
   ```yaml
   token_env: KD_TOKEN
   ```

At least one of `token_cmd` or `token_env` is required. If both are set, `token_cmd` wins.

### Connection selection

`default_connection` names the active connection. If only one connection exists, it is used implicitly. With multiple connections and no `default_connection`, kd errors with a list of available names.

Per-invocation override: pass `-c <name>` (or `--connection <name>`) to target any configured connection without editing config. `KD_CONNECTION` env var works the same way. Priority: `-c` > `KD_CONNECTION` > `default_connection`.

```bash
kd -c upstream /issue list
KD_CONNECTION=corp kd /project list
```

Unknown connection names produce a clear error listing the configured connections.

### Overriding one key for a shared connection

Connections deep-merge across the global and local files, so a workspace-local `.kd/config.yaml` can pin a single key without redeclaring the full connection. Useful for per-project `default_project` on an otherwise-shared connection:

```yaml
# ~/.config/kd/config.yaml  (global — full connection)
default_connection: corp
connections:
  corp:
    url: https://yt.example.com
    token_cmd: gopass show -o infra/services/corp-youtrack/api-token
    default_project: GEN

# ./.kd/config.yaml  (local — one key)
connections:
  corp:
    default_project: BACKEND
```

In this directory, `kd /project list` resolves `url` and `token_cmd` from global and `default_project` from local. `kd /system/config list` attributes each key to its source file. Credential rotation stays a one-file edit in global.

The same merge rule applies inside `defaults:` and `compatibility:` — local inner keys override their global counterparts; keys not set locally are inherited.

### Compatibility

Each connection can pin a compatibility tier for version-aware behavior:

```yaml
compatibility:
  target: 2023.2    # or "auto" (default)
  mode: pinned
```

This lets connections targeting older YouTrack versions (e.g., 2023.2 on-prem) coexist with newer cloud instances.

## Global flat keys

The top level of the config file holds display settings that apply across all connections:

```yaml
output: table    # table, json, yaml, md
limit: 50
debug: false
```

These are fallbacks -- connection-level `output` and `limit` override them when set. Set any path (flat or nested) via CLI — see [Inspect and modify config](#inspect-and-modify-config) below.

## Structured sections

Four optional sections provide deeper customization.

### `defaults:` -- per-command defaults

Set options that apply every time you run a specific command:

```yaml
defaults:
  issue_list:
    limit: 100
    query: "for: me #Unresolved"
  issue_show:
    detail: rich
  article_list:
    limit: 20
```

The key is the command key: namespace path with `/` replaced by `_`, plus the command name. Examples: `issue_list`, `issue_comment_add`, `system_config_list`.

### `projects:` -- per-project overrides

Override field names and command defaults for a specific project:

```yaml
projects:
  FB:
    field_aliases:
      state: Stage
      type: Kind
    defaults:
      issue_list:
        fields: id,summary,state,priority,assignee
        limit: 200
  BACKEND:
    field_aliases:
      state: Status
```

`field_aliases` map semantic CLI names (`state`, `type`, `priority`) to the actual YouTrack field names used in that project.

Project defaults follow the same `command_key: {option: value}` pattern as top-level defaults, but scoped to commands that target that project.

### `aliases:` -- command shortcuts

Named shortcuts that expand to full commands:

```yaml
aliases:
  my-bugs: "/issue list --query 'for: me Type: Bug #Unresolved'"
  standup: "/issue list --query 'for: me updated: Today'"
  fields: "/project field list FB"
```

See the `aliases:` section in the usage guide for the full alias workflow.

### `history:` -- snapshot retention

Controls how many copy-on-write snapshots to keep for config and primer files:

```yaml
history:
  retention: 20    # default; keep the last 20 snapshots per file
```

Before every config or primer write, kd automatically snapshots the current file to `.kd/history/`. Use `kd /system history`, `kd /system restore`, and `kd /system diff` to browse, restore, and compare snapshots.

## Resolution order

### Flat keys (default_connection, output, limit, debug)

First match wins, per key:

| Priority | Source | Example |
|----------|--------|---------|
| 1 | CLI flag | `--output json` |
| 2 | Local config | `.kd/config.yaml` |
| 3 | Global config | `~/.config/kd/config.yaml` |
| 4 | Built-in default | `output: table`, `limit: 50` |

### Command options (limit, output, query, fields, etc.)

When a command runs, its options are resolved through a 10-level chain. First match wins:

| Priority | Source | Example |
|----------|--------|---------|
| 1 | CLI flag | `--limit 10` |
| 2 | Environment variable | (N/A for command options) |
| 3 | Project command default | `projects.FB.defaults.issue_list.limit` |
| 4 | Local command default | `defaults.issue_list.limit` in local config |
| 5 | Local flat key | `limit: 50` in local config |
| 6 | Connection command default | `connections.corp.defaults.issue_list.limit` |
| 7 | Connection flat key | `limit: 50` on the active connection |
| 8 | Global command default | `defaults.issue_list.limit` in global config |
| 9 | Global flat key | `limit: 50` in global config |
| 10 | Built-in default | hardcoded in the option declaration |

Levels 5, 7, and 9 (flat key fallback) only apply to `limit` and `output`.

### What this means in practice

If your connection says `limit: 100` and your local `defaults.issue_list.limit` is `200`, then `kd /issue list` uses 200 (level 4 beats level 7). But `kd /issue list --limit 10` uses 10 (level 1 always wins).

Local config overrides connection defaults. This lets a project-local `.kd/config.yaml` customize behavior for that directory without changing the connection definition.

## Local config and directory walk

`kd init` creates `.kd/config.yaml` in the current directory and adds `.kd/` to `.gitignore`. To scaffold the global file instead, run `kd init --global [URL]` — the parent directory is created as needed.

When kd runs, it walks up from the current working directory to the filesystem root, looking for the first `.kd/config.yaml` it finds. This means:

```
~/code/myproject/.kd/config.yaml    <-- local config for myproject
~/code/myproject/src/                  <-- uses myproject's config
~/code/myproject/src/deep/nested/      <-- still uses myproject's config
~/code/other/.kd/config.yaml        <-- separate local config
```

### Nested local configs

If you have configs at multiple levels, the **closest** one wins:

```
~/code/.kd/config.yaml              <-- parent config
~/code/myproject/.kd/config.yaml    <-- child config (wins for myproject/)
```

There is no merging between local configs -- only one local file is active at a time.

### No local config

If no `.kd/config.yaml` is found anywhere up the tree, kd uses global config only.

## Project determination

Several config features (project defaults, field aliases) are scoped to a project. kd determines the target project in this order:

1. Explicit `--project` flag
2. `default_project` from the active connection
3. Inferred from the first issue-ID-shaped argument (e.g., `FB-123` implies project `FB`)

If no project can be determined, project-scoped config is skipped entirely.

## Inspect and modify config

### Discover what's settable

`kd /system/config keys` is the single source of truth for every configurable
path. It lists each path with its type, default, and description — derived
from the schema, so it never drifts from the code.

```bash
kd /system/config keys
kd /system/config keys -o json   # structured output for LLM consumers
```

Variadic segments render as placeholders (`<name>` for connections, `<key>`
for projects, `<command_key>`/`<option>` for command defaults, `<alias>` for
field aliases).

### Set a value by path

`kd /system/config set PATH VALUE` accepts any path enumerated by
`config keys`. Values are coerced through the schema; the whole config is
revalidated after the write, and failures roll back to the prior file.

```bash
# Flat key
kd /system/config set default_connection corp

# Nested connection override (adds one line to local config, no defaults dumped)
kd /system/config set connections.corp.default_project BACKEND --local

# Nested history setting (coerced to int)
kd /system/config set history.retention 50 --global
```

Unknown paths surface a clean error pointing at `config keys`.

### Inspect active values

See what's resolved and where each value comes from:

```bash
# Full config with sources
kd /system/config list

# Single value by path (flat or nested) with source + origin
kd /system/config get default_connection
kd /system/config get connections.corp.default_project

# Which config files are active
kd /system/config which
```

Example output of `config list`:

```
key                              value                       source    origin
-------------------------------  --------------------------  --------  ---------------------------------
default_connection               corp                        global    ~/.config/kd/config.yaml
output                           table                       local     /code/proj/.kd/config.yaml
limit                            50                          default   default
debug                            false                       default   default
connections.corp.url             https://yt.example.com
connections.corp.default_project BCO
defaults.issue_list.limit        200                         local
projects.FB.field_aliases.state  Stage                       local
aliases.my-bugs                  /issue list --query '...'   local
```

## Debug resolution

### Trace a single command option

`kd /system/config which <cmd>.<opt>` walks the 10-layer command-option chain and shows every layer's value plus which is winning, shadowed, not set, or n/a:

```bash
kd /system/config which primer_show.output
kd /system/config which issue_list.limit -o json     # parseable for LLMs
kd /system/config which issue_list.limit --connection upstream
kd /system/config which primer_show.output --project BACKEND
```

The trace is the user-facing answer to "did setting this path actually take effect?" — particularly when a value at a higher-tier layer (e.g. `connections.<name>.output`, layer 7) shadows a more-specific value at a lower tier (e.g. `defaults.<cmd>.output`, layer 8). `--project` / `--connection` let you explore resolution under a different context without switching `-c` on every command.

`config which` with no argument still prints the active local and global config file paths; with a flat key (`default_connection`, `output`, `limit`, `debug`) it returns the winning value plus its origin.

### Shadow warning on `config set`

When `config set` writes a value that is shadowed by a higher-priority layer, kd emits a non-fatal warning to stderr naming the winning layer and suggesting a path that would actually override it:

```
$ kd /system/config set defaults.primer_show.output yaml --global
…
Warning: wrote primer_show.output = 'yaml' at layer 8 (global.defaults.primer_show),
but layer 7 (connection.flat) = 'table' shadows it.
Run 'kd /system/config which primer_show.output' for the full chain.
Hint: to override at a higher-priority layer, set:
  kd /system/config set connections.corp.defaults.primer_show.output 'yaml'
```

The warning is structured under the `warning` key in JSON/YAML output. Pass `--no-shadow-warning` to suppress for scripts.

### Verbose trace

For deeper inspection, use `-D` to see how every key is resolved:

```bash
kd -D /issue list --project FB
```

At `-DD` (debug level), you'll see per-key resolution decisions. At `-DDD` (trace), the full walk through every source for every key.

## Complete example

A config file using all sections:

```yaml
default_connection: corp

connections:
  corp:
    provider: youtrack
    url: https://yt.example.com
    token_cmd: gopass show -o infra/services/youtrack/api-token
    default_project: FB
    output: table
    defaults:
      issue_list:
        query: "for: me #Unresolved"
    compatibility:
      target: 2023.2
      mode: pinned

# Global display fallbacks
output: table
limit: 50

# Per-command defaults (apply to all connections)
defaults:
  issue_show:
    detail: rich

# Per-project overrides
projects:
  FB:
    field_aliases:
      state: Stage
      type: Kind
    defaults:
      issue_list:
        fields: id,summary,state,priority,assignee

# Command aliases
aliases:
  my-bugs: "/issue list --query 'for: me Type: Bug #Unresolved'"
  standup: "/issue list --query 'for: me updated: Today'"
```

---
Updated: 2026-04-14
