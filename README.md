# kd — Knowledge Dispatcher

Command-line and Python tool for accessing and managing information that lives in issue trackers and knowledge bases. Designed to be equally natural for humans and large language models.

- **Install**: `pipx install kd-cli` (or `pip install kd-cli`)
- **PyPI**: https://pypi.org/project/kd-cli/
- **Issues**: https://github.com/IntellectSolutionsCom/kd-cli/issues
- **Release notes**: `kd /release-notes` (in-tool) or the [Changelog on PyPI](https://pypi.org/project/kd-cli/)

Today kd speaks **YouTrack**. On the roadmap: **JIRA**, **GitHub Issues**, **GitLab**, and possibly **Obsidian**.

## Quick start

```bash
pipx install kd-cli

# Scaffold the global config (recommended on first install).
kd init --global https://your-instance.youtrack.cloud

# Or a project-local config in the current directory.
kd init

# Or set via environment.
export KD_TOKEN=your_token
export KD_CONNECTION=main

# Check connection, version, and what shipped in this release.
kd --version
kd /system info
kd /release-notes

# Work with issues, articles, time, tags, primer.
kd /issue list --project FB --query "#Unresolved"
kd /issue create "Fix router config" -p FB --type Task
kd /article pull FB-A-42 > setup.md
kd /time log FB-1 45 --type Development
kd /search "router" --types issue,article
kd /primer show
```

See [Install guide](docs/install.md) for full macOS / Linux / Windows instructions, and the [Usage guide](docs/usage.md) for scenario-driven walkthroughs of every namespace.

## The primer

The **primer** (`kd /primer`) is kd's centerpiece: a small, workspace-local file of pinned references and curated notes that every kd session can surface on demand.

It exists to solve a problem that has gotten worse as LLMs have become central to real work:

- **Visible memory.** Agentic tools accumulate hidden "memories" in opaque per-LLM buckets you cannot easily read, edit, or audit. The primer is a plain file you can open, version, and review.
- **LLM-portable.** One primer works for any model. Switch from Claude to GPT-5 to the next generation — the primer travels with the project, not with a vendor.
- **Shared by humans and LLMs.** A developer jotting "on this project the sprint field is called `Board`" in the primer is building the same artifact an LLM would build, in the same place, readable by both.
- **A routing layer, not a copy.** The primer can pin real knowledge-base articles (`kd /primer/article add FB-A-42`). It never duplicates their content — it points at them. The rest of kd extends this shape: pull what the current task needs from live systems, reason, push updates back.

## Filing an issue

When reporting a bug, please include:

- `kd --version`
- Your OS and Python version
- The exact command you ran and the observed output
- What you expected instead

For security-sensitive reports, email the maintainer rather than filing a public issue.

## License

MIT.
