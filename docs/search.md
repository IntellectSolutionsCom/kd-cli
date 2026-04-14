# How-To: Cross-Entity Search

`/search` finds things across YouTrack without knowing which namespace to look in first. One command, five entity types.

## Basic search

```bash
kd /search "router"
```

Returns a mixed table of issues, articles, projects, tags, and users that match "router".

## Filter by entity type

```bash
# Only issues and articles
kd /search "router" --types issue,article

# Only users
kd /search "alice" --types user

# Only tags
kd /search "sprint" --types tag
```

Valid types: `issue`, `user`, `tag`, `project`, `article`.

## Scope to a project

`--project` narrows issue queries and article scans to a single project:

```bash
kd /search "timeout" --project FB
```

For issues this prepends `project: FB` to the server-side query. For articles it uses the project-scoped listing endpoint. Users, tags, and projects are unaffected.

## Limit results

```bash
kd /search "router" --limit 5
```

Default is 50. Results are interleaved round-robin across types so you see a mix rather than 50 issues followed by everything else.

## Structured output

```bash
# JSON (stable contract for scripts and LLMs)
kd /search "router" -o json

# Specific columns
kd /search "router" -f type,id,title
```

Each result has four fields: `type`, `id`, `title`, `project`.

## How matching works

| Type | Mechanism | What matches |
|------|-----------|-------------|
| issue | Server-side YT query | Summary, description, custom fields (YouTrack query semantics) |
| user | Server-side YT query | Login, name, email |
| tag | Client-side substring | Tag name (case-insensitive) |
| project | Client-side substring | Project name or short name (case-insensitive) |
| article | Client-side substring | Article summary (case-insensitive) |

Issue and user search leverage YouTrack's built-in query engine. Tag, project, and article search fetch listings and filter client-side because the upstream API has no query parameter for these entity types.

## Combine with aliases

Define a search shortcut in config:

```yaml
aliases:
  find: "/search"
```

```bash
kd find "router" --types issue,article
```

## When to use /search vs /issue list

- Use `/search` when you don't know what type the result is, or want mixed results.
- Use `/issue list --query` when you need full YouTrack query language power (state filters, date ranges, custom fields).
- `/search` delegates issue search to the same API as `/issue list`, so issue results are identical for the same query string.

---
Updated: 2026-04-14
