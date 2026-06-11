---
name: frago-source-install
description: |
  This skill should be used when a user wants to install frago from source and configure it for Claude Code on macOS, Linux, or Windows. It clones the frago repository, builds the environment with uv, then writes every configuration file directly (~/.frago/config.json, ~/.frago/runtime.json, hook binary deployment to ~/.claude/hooks/frago/, and a hooks merge into ~/.claude/settings.json) — never running frago init or frago server. Trigger phrases: "install frago", "安装 frago", "配置 frago", "frago 源码安装", "set up frago from source", "frago source install", "frago manual setup", "在我电脑上装 frago".
license: AGPL-3.0
---

## Overview
Install frago from source on macOS, Linux, or Windows and configure it for Claude Code entirely by writing files directly. Never run `frago init` or `frago server start` — the automatic configuration those commands perform (hook deployment, settings registration, launcher detection) is replaced here by explicit file writes.

## Steps

### 1. Prerequisites
- Ensure `git` and `uv` are installed. macOS: `xcode-select --install` provides git; install uv via `curl -LsSf https://astral.sh/uv/install.sh | sh`. Linux: install git and curl via apt/dnf/pacman. Windows: `winget install Git.Git`; install uv via `powershell -c "irm https://astral.sh/uv/install.ps1 | iex"`.
- Do not pre-install Python: uv downloads a managed Python matching `requires-python >= 3.13` automatically.
- Google Chrome is optional (only needed for browser automation features).

### 2. Clone and build the environment
- `git clone https://github.com/tsaijamey/frago.git` into a stable path such as `~/frago`, then run `uv sync` inside the repo (no `--all-extras --dev` needed for normal use).
- The CLI entry point is `<repo>/.venv/bin/frago` (Windows: `<repo>/.venv/Scripts/frago.exe`).

### 3. Write `~/.frago/config.json` (only if absent — never overwrite an existing file)
```json
{"schema_version": "1.0", "auth_method": "official", "init_completed": true}
```
`auth_method=official` leaves the user's existing Claude Code authentication untouched.

### 4. Write `~/.frago/runtime.json`
```json
{"schema_version": "1.0", "launcher": {"command": ["<absolute repo path>/.venv/bin/frago"], "mode": "global", "detected_at": "<ISO timestamp>", "source": {}}}
```
This file is the only way the Rust hook binary locates the CLI; if it is missing or `launcher` is empty, the hook silently does nothing (intended failure mode). On Windows use forward slashes (`C:/Users/...`) — Claude Code launches hooks via Git Bash, and backslashes get consumed as escapes.

### 5. Deploy the hook binary
- Copy `<repo>/src/frago/bin/<platform>/frago-hook` to `~/.claude/hooks/frago/` (create the directory first). Platform directories: `darwin-arm64`, `darwin-x86_64`, `linux-x86_64`, `windows-x86_64` (Windows file is `frago-hook.exe`). On POSIX, `chmod 755` the copy.
- On unsupported platforms (e.g. linux-aarch64) skip hook setup entirely; the CLI still works.

### 6. Merge hooks into `~/.claude/settings.json` — merge, never overwrite the user's existing settings
- Run `~/.claude/hooks/frago/frago-hook --supported-events` to get the authoritative event list as a JSON array (currently SessionStart, UserPromptSubmit, PreToolUse). Prefer this dynamic output over a hardcoded list.
- For each event, append `{"matcher": "", "hooks": [{"type": "command", "command": "<absolute path to ~/.claude/hooks/frago/frago-hook>", "timeout": 10}]}` to the event's array — skip if an entry with the same command is already registered.

### 7. Verify
1. `<repo>/.venv/bin/frago --version` prints a version number.
2. `<repo>/.venv/bin/frago book` prints the knowledge index.
3. `~/.claude/hooks/frago/frago-hook --supported-events` prints a JSON array of three events.
4. Restart Claude Code; a new session should show the frago knowledge index injected at SessionStart, proving the full hook chain (runtime.json → frago book) works.
On any failure, check: the runtime.json launcher path is absolute and exists; the settings.json hook command path is correct; the binary has execute permission.

## Optional appendix: third-party API endpoints
Only when the user has no official Anthropic authentication and wants a compatible endpoint (deepseek/aliyun/kimi/minimax/custom): merge into the `env` section of `~/.claude/settings.json` these seven keys — ANTHROPIC_BASE_URL, ANTHROPIC_MODEL, ANTHROPIC_DEFAULT_SONNET_MODEL, ANTHROPIC_DEFAULT_HAIKU_MODEL, API_TIMEOUT_MS, CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC, ANTHROPIC_API_KEY (example: deepseek base_url is https://api.deepseek.com/anthropic). Also ensure `~/.claude.json` contains `{"hasCompletedOnboarding": true, "lastOnboardingVersion": "1.0.0", "isQualifiedForDataSharing": false}` to skip the official login onboarding.

## Do NOT
- Run any `frago init` / `frago server` command as part of setup.
- Write hook-rules.json, ~/.frago/books/, AGENTS.md, or ~/.frago/.gitignore — none are needed (builtin hook rules live inside the Rust binary; other ~/.frago subdirectories are lazily created by the CLI).
- Install the desktop (Tauri) client, Node.js, or Claude Code itself.
- Touch the user's existing Claude Code authentication.


---

## About

Generated by **frago** — An Agent OS that turns ad-hoc agent runs into reusable recipes.

Install: `uv tool install frago-cli`
Homepage: https://frago.ai · Docs: https://docs.frago.ai
