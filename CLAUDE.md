# CLAUDE.md

## Project Overview

This repo contains configuration files for running [gitleaks](https://github.com/gitleaks/gitleaks) as a pre-commit hook to detect hardcoded secrets across git repositories.

Key files:
- `gitleaks.toml` — gitleaks rules (auto-generated from upstream default config, with custom allowlists)
- `gitleaks.txt` — supplementary notes or output
- `.pre-commit-config.yaml` — pre-commit hook config for this repo
- `README.md` — setup guide for per-repo and global usage

## Environment

- Repo lives on **Windows NTFS filesystem mounted via WSL2** (`/mnt/c/...`).
- This causes `PermissionError: [Errno 1] Operation not permitted` when pre-commit tries to write to `.git/hooks/` (NTFS doesn't support Python's atomic file replace).

## Pre-commit Hook Architecture

To work around the NTFS limitation, hooks are stored on the **Linux native filesystem** and git is configured globally to use them.

### Global setup (already configured)

```
~/.config/git/hooks/pre-commit        ← global hook script (Linux FS)
~/.config/git/pre-commit-config.yaml  ← global default pre-commit config
~/.config/git/gitleaks.toml           ← global gitleaks rules
```

Git global config:
```
git config --global core.hooksPath ~/.config/git/hooks
```

The hook script at `~/.config/git/hooks/pre-commit` selects the config dynamically:
- If the current repo has `.pre-commit-config.yaml` → uses it
- Otherwise → falls back to `~/.config/git/pre-commit-config.yaml`

### Important caveats

- **Never run `pre-commit install`** in a repo while `core.hooksPath` is set globally — it will refuse with `Cowardly refusing to install hooks with core.hooksPath set`.
- **After `pre-commit autoupdate`**, manually re-copy the hook if it was regenerated:
  ```bash
  cp /mnt/c/Users/siwan/work/gitleaks-tool/.git/hooks/pre-commit ~/.config/git/hooks/pre-commit
  ```
- **Never edit `.git/config` via `git config`** in repos on `/mnt/c/` — git creates a `.lock` file which also fails on NTFS. Use the Edit tool to modify `.git/config` directly instead.

## Testing Pre-commit Without Committing

```bash
pre-commit run --all-files
# or for a specific hook:
pre-commit run gitleaks --all-files
```

## Updating gitleaks Rules

When updating `gitleaks.toml` in this repo, also sync it to the global copy:
```bash
cp /mnt/c/Users/siwan/work/gitleaks-tool/gitleaks.toml ~/.config/git/gitleaks.toml
```
