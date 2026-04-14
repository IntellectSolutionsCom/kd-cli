# How-To: Project Primer

Store project-specific knowledge in `.kd/primer.yaml` so LLMs (and humans) can read it once and understand what matters in this project. Notes capture conventions and do's/don'ts; pinned articles point to key references.

## Show the primer

```bash
kd /primer show
```

Returns the full primer (notes + articles). If no primer exists yet, returns empty lists — not an error.

```bash
# Machine-readable for LLM consumption
kd /primer show --output json
kd /primer show --output yaml
```

## Add a note

```bash
kd /primer/note add "Sprint field is called 'Board' in this project"
kd /primer/note add "Never close issues tagged 'Evergreen'"
```

Notes are free-text strings — conventions, warnings, project-specific quirks. Each is appended and assigned a 1-based index.

Supports `@file` and `-` (stdin) for longer text:

```bash
kd /primer/note add @conventions.txt
echo "Important note" | kd /primer/note add -
```

## List notes

```bash
kd /primer/note list
```

Shows each note with its index.

## Update a note

Replace an existing note at a 1-based index without removing and re-adding:

```bash
kd /primer/note update 1 "Updated convention"
kd /primer/note update 2 @updated.txt
echo "Revised wording" | kd /primer/note update 3 -
```

Supports the same `@file` / `-` input modes as `add`. Use `/primer/note list` first to see current indices.

## Remove a note

```bash
# Remove note at index 2
kd /primer/note remove 2

# Remove all notes
kd /primer/note remove --all
```

Use `/primer/note list` first to see current indices. Indices shift after removal.

## Pin an article

```bash
kd /primer/article add FB-A-42

# With an optional summary
kd /primer/article add FB-A-42 --summary "Architecture Overview"
```

Pinned articles are YouTrack article references that matter for this project. The summary is optional descriptive text stored alongside the ID.

## List pinned articles

```bash
kd /primer/article list
```

## Unpin an article

```bash
kd /primer/article remove FB-A-42

# Remove all pinned articles
kd /primer/article remove --all
```

## Prerequisites

The primer lives in `.kd/primer.yaml`, which requires a `.kd/` directory. If you haven't initialized one:

```bash
kd init
```

Read-only commands (`show`, `list`) work without `.kd/` — they return empty results. Write commands (`add`, `remove`) require it.

## History and restore

Every primer mutation (add, remove, clear) automatically snapshots the current file to `.kd/history/` before writing. This means accidental deletions are always recoverable.

```bash
# List primer snapshots
kd /system history primer

# See what changed
kd /system diff primer 2026-04-09T14-20-00

# Restore a previous version
kd /system restore primer 2026-04-09T14-20-00
```

Restore snapshots the current file first, so restore itself is undoable. Retention defaults to 20 snapshots; configure via `history.retention` in config.

## Local only

Unlike config (which has both local `.kd/config.yaml` and global `~/.config/kd/config.yaml`), the primer is **project-local only**. There is no global primer and no merging across directories. Each project has its own primer or none at all — this is intentional, since primer content is project-specific by nature.

## How the file looks

The primer is a simple YAML file you can also edit directly:

```yaml
# .kd/primer.yaml
notes:
  - "Sprint field is called 'Board' in this project"
  - "Never close issues tagged 'Evergreen'"

articles:
  - id: FB-A-42
    summary: Architecture Overview
  - id: FB-A-7
```

## LLM workflow

An LLM agent working on a project uses two commands to orient itself:

```bash
# 1. Learn what kd can do
kd --reference

# 2. Learn what matters in this project
kd /primer show --output json
```

The primer appears in `--reference` output (as a registered command), so the LLM discovers it exists automatically.

## Troubleshooting

### "No .kd/ directory found"

You tried a write command (`add` or `remove`) without initializing.

```bash
kd init
```

### Duplicate article

Pinning an article that's already pinned raises an error. Use `/primer/article list` to check.

### Note index out of range

Note indices are 1-based and shift when notes are removed. Always check `/primer/note list` before removing.

## Related

- [Configuration](configuration.md) — config is tool settings; primer is project knowledge
- [Usage guide](usage.md) — the primer is embedded in the broader LLM workflow

---
Updated: 2026-04-14
