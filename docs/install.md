# Install `kd` on macOS, Linux, and Windows

Install `kd` (Knowledge Dispatcher) from PyPI on a fresh machine. The PyPI
distribution is `kd-cli`; the installed command, the Python import name, and
the CLI entry point are all `kd`.

Requirements: Python 3.11 or newer. Everything else gets installed below.

## macOS

Homebrew is the path of least resistance for Python and pipx on macOS.

```bash
# 1. Homebrew (skip if already installed)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 2. Python + pipx
brew install python pipx
pipx ensurepath

# Close and reopen the terminal (or `source ~/.zshrc`) so PATH picks up pipx.

# 3. Install kd-cli
pipx install kd-cli
```

Verify:

```bash
kd --version
kd --help
```

## Linux

### Ubuntu / Debian

```bash
# 1. Python + pipx
sudo apt update
sudo apt install -y python3 python3-pip pipx
pipx ensurepath

# Close and reopen the terminal (or `source ~/.bashrc`) so PATH picks up pipx.

# 2. Install kd-cli
pipx install kd-cli
```

Ubuntu 24.04+ ships `pipx` in the archive. On older releases use
`python3 -m pip install --user pipx` followed by `python3 -m pipx ensurepath`.

### Fedora / RHEL / Rocky

```bash
sudo dnf install -y python3 pipx
pipx ensurepath
pipx install kd-cli
```

### Arch

```bash
sudo pacman -S --needed python python-pipx
pipx ensurepath
pipx install kd-cli
```

### Any Linux with `uv`

`uv` handles Python installation itself, so this works on any distro without
package-manager dependencies. Good for minimal containers or locked-down
systems.

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
uv tool install kd-cli
```

## Windows

PowerShell, from a fresh Windows 11 install.

```powershell
# 1. Python — install 3.12 or 3.13 from the Microsoft Store
#    or python.org. During the python.org installer, check
#    "Add python.exe to PATH".
python --version

# 2. pipx
python -m pip install --user pipx
python -m pipx ensurepath

# Close and reopen the terminal so PATH picks up pipx.

# 3. Install kd-cli
pipx install kd-cli
```

Verify:

```powershell
kd --version
kd --help
```

### Notes for Windows

- **Config path**: `kd` uses a Unix-style layout on all platforms. The global
  config lives at `C:\Users\<you>\.config\kd\config.yaml`, not under
  `%APPDATA%`. Create the directory if it does not exist yet.
- **Environment variables**: use `setx` to persist, then open a new terminal
  for the change to take effect.
- **Git Bash / PowerShell / cmd.exe**: all three work the same for running
  `kd` once installed.

## WSL (Windows Subsystem for Linux)

If you're using WSL, do the install **inside** WSL following the Linux steps
above, not by mixing Windows Python with WSL paths. The two environments have
separate PATHs; mixing them causes confusing "command not found" errors.

## Configure

Identical on every platform.

```bash
# Project-local config (in the current directory)
kd init

# Or set via environment (overrides workspace + global config)
export KD_TOKEN=your_token
export KD_CONNECTION=main
```

Config lookup order:

1. CLI flags (`-c`, `--project`, etc.)
2. Walk-up from `cwd` for `.kd/config.yaml`
3. Global `~/.config/kd/config.yaml` (Windows: `C:\Users\<you>\.config\kd\config.yaml`)

A minimal global config:

```yaml
default_connection: main

connections:
  main:
    provider: youtrack
    url: https://your-instance.youtrack.cloud
    token_env: KD_TOKEN
```

## Upgrade

```bash
pipx upgrade kd-cli
# or, if installed via uv
uv tool upgrade kd-cli
```

## Uninstall

```bash
pipx uninstall kd-cli
# or
uv tool uninstall kd-cli
```

User config (`~/.config/kd/`) and workspace configs (`.kd/`) are left in
place. Remove them manually if desired.

## Troubleshooting

### `kd: command not found` after install

The pipx bin directory is not on `PATH` yet. Run `pipx ensurepath` and open a
new terminal.

### Install fails with `Requires-Python >=3.11`

The active Python is older than 3.11. Install a newer Python and rerun pipx
against it: `pipx install --python python3.12 kd-cli`.

### `ModuleNotFoundError: No module named 'kd'`

Either an older install is shadowing the new one, or the venv got corrupted.
Reinstall from a clean state:

```bash
pipx uninstall kd-cli
pipx install kd-cli
```

### Behind a corporate proxy

Pipx uses the `HTTPS_PROXY` / `HTTP_PROXY` environment variables. Set them
before running pipx:

```bash
export HTTPS_PROXY=http://proxy.corp:8080
export HTTP_PROXY=http://proxy.corp:8080
pipx install kd-cli
```

## Related

- https://pypi.org/project/kd-cli/ — project page on PyPI
- https://pipx.pypa.io — pipx documentation
- https://docs.astral.sh/uv/ — uv documentation

---
Updated: 2026-04-14
