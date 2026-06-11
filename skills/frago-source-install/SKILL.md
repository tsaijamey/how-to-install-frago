---
name: frago-source-install
description: |
  This skill should be used when the user wants to install frago from source on their machine (macOS/Linux/Windows) and configure it for Claude Code without running any frago commands. It guides Claude Code to clone the frago repository, build the environment with uv sync, then hand-write all configuration files (~/.frago/config.json, ~/.frago/runtime.json, hook binary deployment to ~/.claude/hooks/frago/, and merged hooks registration in ~/.claude/settings.json), replacing the frago init / frago server auto-configuration flow. Trigger phrases: "install frago", "安装 frago", "配置 frago", "frago 源码安装", "set up frago", "frago source install", "install frago from source", "deploy frago hooks", "把 frago 装到我的机器上".
license: AGPL-3.0
---

## Overview
Install frago from source and configure it for Claude Code by writing configuration files directly — never run `frago init` or `frago server start`. The product's auto-configuration (hook deployment + launcher detection at server startup) is intentionally replaced by hand-written files because productization is not yet complete. Prerequisites are only git and uv; Python need not be preinstalled (uv auto-downloads a managed Python per `requires-python>=3.13`), and no C/C++ toolchain is needed (62 of 63 locked dependencies ship prebuilt wheels). Google Chrome is optional, only for browser automation.

## Steps

### 1. Prepare the environment per OS
- macOS: ensure git via `xcode-select --install` (often already present); install uv with `curl -LsSf https://astral.sh/uv/install.sh | sh`.
- Linux (apt/dnf/pacman): install `git curl`; install uv with the same curl command; Chrome (optional) from dl.google.com as deb/rpm.
- Windows: `winget install Git.Git` — this satisfies two prerequisites at once: git itself AND Git Bash. Claude Code on Windows launches hooks through Git Bash (`/usr/bin/bash`), so Git for Windows is a hard requirement for the entire hook chain, not just for cloning. Install uv with `powershell -c "irm https://astral.sh/uv/install.ps1 | iex"`. If winget is unavailable, fall back to the installer from git-scm.com.
- PATH note (POSIX): uv installs to `~/.local/bin`, which may not be on PATH in the current shell. Use the absolute path `~/.local/bin/uv` for subsequent commands, or `source ~/.local/bin/env` first. On Windows the uv installer writes PATH to the registry; new processes see it.

### 2. Clone and build the environment
- `git clone https://github.com/tsaijamey/frago.git` into a stable path (e.g. `~/frago`).
- Network fallback for regions where github.com is unreachable: prefix with a mirror, e.g. `git clone https://mirror.ghproxy.com/https://github.com/tsaijamey/frago.git` or the `https://ghproxy.net/` prefix.
- Run `uv sync` in the repo directory (no `--all-extras --dev` needed for normal use).
- The CLI entry point is `<repo>/.venv/bin/frago` (Windows: `<repo>/.venv/Scripts/frago.exe`). Package name is `frago-cli`, entry point `frago = frago.cli.main:cli`.
- Hook binaries ship with the repo at `src/frago/bin/<platform>/frago-hook` (git-tracked with the executable bit). Platform dirs: `darwin-arm64`, `darwin-x86_64`, `linux-x86_64`, `windows-x86_64` (Windows file is `frago-hook.exe`). On unsupported platforms (e.g. linux-aarch64) the hook feature is unavailable but the CLI still works.

### 3. Write configuration files directly (replaces frago init / server auto-config)
Normally `frago server` deploys the hook binary, registers hooks via the binary's `--supported-events` output, and detects the launcher into `~/.frago/runtime.json`. Replicate these manually:

1. `~/.frago/config.json` — write ONLY if it does not exist; never overwrite an existing one:
   `{"schema_version": "1.0", "auth_method": "official", "init_completed": true}`
   `auth_method=official` means the user's existing Claude Code authentication is left untouched.
2. `~/.frago/runtime.json`:
   `{"schema_version": "1.0", "launcher": {"command": ["<absolute repo path>/.venv/bin/frago"], "mode": "global", "detected_at": "<ISO timestamp>", "source": {}}}`
   This file is the Rust hook binary's only way to invoke the knowledge index; if it is missing or `launcher` is empty, the hook silently does nothing (this is the designed failure mode). On Windows the path MUST use forward slashes (`C:/Users/...`) because hooks run through Git Bash, where backslashes are eaten as escapes.
3. Deploy the hook binary: `mkdir -p ~/.claude/hooks/frago/`, copy `<repo>/src/frago/bin/<platform>/frago-hook` there, and `chmod 755` on POSIX.
4. `~/.claude/settings.json` hooks section — MERGE into existing settings, never overwrite the whole file. For each of the three events `SessionStart`, `UserPromptSubmit`, `PreToolUse`, append:
   `{"matcher": "", "hooks": [{"type": "command", "command": "<absolute path to ~/.claude/hooks/frago/frago-hook>", "timeout": 10}]}`
   The authoritative event list comes from running `<binary> --supported-events` (outputs a JSON array) — run it before writing to stay correct across versions. Skip any event where an entry with the same command is already registered.
5. Do NOT write these files: `hook-rules.json` (builtin rules are compiled into the Rust binary; the user file is only an override layer), `~/.frago/books/` (system knowledge lives in the Python package; user domain dirs are lazily created), `AGENTS.md` and `~/.frago/.gitignore` (their writers were removed with the old sync stack). Other `~/.frago` subdirectories (`recipes/`, `projects/`, etc.) are lazily created by the CLI.

### 4. Optional appendix: third-party API endpoints
Only if the user lacks official Anthropic authentication and wants a compatible endpoint (deepseek/aliyun/kimi/minimax/custom): merge into the `env` section of `~/.claude/settings.json` the seven keys `ANTHROPIC_BASE_URL`, `ANTHROPIC_MODEL`, `ANTHROPIC_DEFAULT_SONNET_MODEL`, `ANTHROPIC_DEFAULT_HAIKU_MODEL`, `API_TIMEOUT_MS`, `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`, `ANTHROPIC_API_KEY`; and ensure `~/.claude.json` contains `{"hasCompletedOnboarding": true, "lastOnboardingVersion": "1.0.0", "isQualifiedForDataSharing": false}` to skip the official login onboarding. Example preset: deepseek `base_url=https://api.deepseek.com/anthropic`.

### 5. Verify
1. `<repo>/.venv/bin/frago --version` prints a version.
2. `<repo>/.venv/bin/frago book` prints the knowledge index.
3. `~/.claude/hooks/frago/frago-hook --supported-events` prints the three-event JSON array.
4. Restart Claude Code; a new session's SessionStart should inject the knowledge index, proving the full hook chain (runtime.json → CLI) works.
If any step fails, check: the launcher path in runtime.json is absolute and exists; the hook command path registered in settings.json is correct; the binary has execute permission.

## Scope boundaries
- Do NOT install the desktop client (Tauri app).
- Do NOT install Node.js or Claude Code itself (the target user already has them).
- Do NOT modify the user's existing Claude Code authentication.



---

## About

Generated by **frago** — An Agent OS that turns ad-hoc agent runs into reusable recipes.

Install: `uv tool install frago-cli`
Homepage: https://frago.ai · Docs: https://docs.frago.ai
