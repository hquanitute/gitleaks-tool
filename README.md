# Gitleaks Configuration Guide

This repo contains configuration files for running [gitleaks](https://github.com/gitleaks/gitleaks) as a pre-commit hook to detect hardcoded secrets.

Key files:
- `gitleaks.toml` — gitleaks rules (based on upstream default, with custom allowlists)
- `.pre-commit-config.yaml` — pre-commit hook config for this repo

---

## Per-Repository Setup

1. Copy `.pre-commit-config.yaml` and `gitleaks.toml` to the repository root.
2. Install pre-commit hooks:
   ```bash
   pre-commit install
   ```
3. Update to latest hook versions:
   ```bash
   pre-commit autoupdate
   ```

### Scanning Commands

Scan all commits in the git history:
```bash
gitleaks git -v --log-opts="--all"
```

Scan a directory (non-git):
```bash
gitleaks dir -v
```

---

## Global Setup (WSL2 / NTFS)

> **WSL2 users with repos on `/mnt/c/`**: `pre-commit install` will fail with `PermissionError: [Errno 1] Operation not permitted` because the Windows NTFS filesystem does not support Python's atomic file replace operations. Use the global hook setup below instead.

The solution is to store hook scripts on the Linux native filesystem and point git to them globally via `core.hooksPath`.

### File Layout

```
~/.config/git/hooks/pre-commit        # global hook script
~/.config/git/pre-commit-config.yaml  # global default pre-commit config
~/.config/git/gitleaks.toml           # global gitleaks rules
```

### Setup Steps

1. Create the hooks directory on the Linux filesystem:
   ```bash
   mkdir -p ~/.config/git/hooks
   ```

2. Create `~/.config/git/hooks/pre-commit` with the following content and make it executable:
   ```bash
   #!/usr/bin/env bash
   # Global pre-commit hook
   # Uses repo's .pre-commit-config.yaml if present, otherwise falls back to global default

   INSTALL_PYTHON=/usr/bin/python3
   GLOBAL_CONFIG="$HOME/.config/git/pre-commit-config.yaml"

   if [ -f ".pre-commit-config.yaml" ]; then
       CONFIG=".pre-commit-config.yaml"
   else
       CONFIG="$GLOBAL_CONFIG"
   fi

   ARGS=(hook-impl --config="$CONFIG" --hook-type=pre-commit)

   HERE="$(cd "$(dirname "$0")" && pwd)"
   ARGS+=(--hook-dir "$HERE" -- "$@")

   if [ -x "$INSTALL_PYTHON" ]; then
       exec "$INSTALL_PYTHON" -mpre_commit "${ARGS[@]}"
   elif command -v pre-commit > /dev/null; then
       exec pre-commit "${ARGS[@]}"
   else
       echo '`pre-commit` not found.  Did you forget to activate your virtualenv?' 1>&2
       exit 1
   fi
   ```
   ```bash
   chmod +x ~/.config/git/hooks/pre-commit
   ```

3. Create `~/.config/git/pre-commit-config.yaml` (global default config):
   ```yaml
   repos:
     - repo: https://github.com/zricethezav/gitleaks
       rev: v8.30.1
       hooks:
         - id: gitleaks
           name: Detect hardcoded secrets
           description: Detect hardcoded secrets using Gitleaks
           entry: gitleaks git --pre-commit --redact --staged --verbose --config /home/siwan/.config/git/gitleaks.toml
           pass_filenames: false
   ```

4. Copy gitleaks rules to the Linux filesystem:
   ```bash
   cp /mnt/c/Users/siwan/work/gitleaks-tool/gitleaks.toml ~/.config/git/gitleaks.toml
   ```

5. Set the global hooks path:
   ```bash
   git config --global core.hooksPath ~/.config/git/hooks
   ```

The hook will now run on every commit across all repositories. If a repo has its own `.pre-commit-config.yaml`, that is used; otherwise the global config applies.

### Caveats

- **Do not run `pre-commit install`** while `core.hooksPath` is set — it will refuse with `Cowardly refusing to install hooks with core.hooksPath set`.
- **After `pre-commit autoupdate`**, manually sync the hook if it was regenerated in the repo:
  ```bash
  cp /mnt/c/Users/siwan/work/gitleaks-tool/.git/hooks/pre-commit ~/.config/git/hooks/pre-commit
  ```
- **After updating `gitleaks.toml`** in this repo, sync it to the global copy:
  ```bash
  cp /mnt/c/Users/siwan/work/gitleaks-tool/gitleaks.toml ~/.config/git/gitleaks.toml
  ```

---

## Testing Pre-commit Without Committing

```bash
pre-commit run --all-files

# or for a specific hook only:
pre-commit run gitleaks --all-files
```

---

## Special Cases

- **Unit test credentials**: Add allowlist entries in `gitleaks.toml` under `[[allowlists]]` to whitelist specific files or patterns.
- **Custom rules**: Add new `[[rules]]` entries in `gitleaks.toml` following the [gitleaks contributing guide](https://github.com/gitleaks/gitleaks/blob/master/CONTRIBUTING.md).
