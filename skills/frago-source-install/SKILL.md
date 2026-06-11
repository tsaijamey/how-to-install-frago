---
name: frago-source-install
description: |
  This skill should be used when a user asks to install frago from source on their machine (macOS/Linux/Windows) and configure it for Claude Code without running any frago commands for setup. It guides cloning the frago repository, building the environment with uv sync, then directly writing all required configuration files (~/.frago/config.json, ~/.frago/runtime.json, deploying the bundled frago-hook binary, and merging the hooks section into ~/.claude/settings.json) so the hook chain works on next Claude Code session. Trigger phrases: "install frago", "安装 frago", "配置 frago", "frago 源码安装", "set up frago from source", "frago source install", "frago manual setup", "把 frago 装到我电脑上".
license: AGPL-3.0
---

## Overview
Install frago from source and wire it into an existing Claude Code installation by writing configuration files directly. Do NOT run `frago init` or `frago server start` for setup — the server-side auto-configuration (hook deployment + launcher detection) is replaced here by hand-written files. Target users already have Claude Code working; do not install Node.js or Claude Code, do not touch their existing Claude Code authentication, and do not install the desktop (Tauri) client.

## Steps

### 1. Prepare prerequisites (git + uv only)
Python need not be pre-installed: uv auto-downloads a managed Python matching `requires-python>=3.13`. All locked dependencies ship prebuilt wheels except one pure-Python package, so no C/C++ toolchain is needed on any OS. Google Chrome is optional (browser automation only).
- macOS: git via `xcode-select --install` (or already present); uv via `curl -LsSf https://astral.sh/uv/install.sh | sh`.
- Linux: install `git curl` with apt/dnf/pacman; uv via the same curl script.
- Windows: `winget install Git.Git` (fallback: git-scm.com installer); uv via `powershell -c "irm https://astral.sh/uv/install.ps1 | iex"`. The uv installer updates the registry PATH, visible to new processes.
- POSIX PATH note: uv lands in `~/.local/bin`, which may not be on the current shell's PATH — call it as `~/.local/bin/uv` or `source ~/.local/bin/env` first.

### 2. Clone and build the environment
Clone `https://github.com/tsaijamey/frago.git` to a stable path (e.g. `~/frago`). If github.com is unreachable, use a mirror prefix: `https://mirror.ghproxy.com/https://github.com/tsaijamey/frago.git` or the `https://ghproxy.net/` prefix. In the repo directory run `uv sync` (no `--all-extras --dev` needed for normal use). The CLI entry point is `<repo>/.venv/bin/frago` (Windows: `<repo>/.venv/Scripts/frago.exe`).

### 3. Write ~/.frago/config.json (only if absent — never overwrite an existing one)
```json
{"schema_version": "1.0", "auth_method": "official", "init_completed": true}
```
`auth_method=official` keeps the user's existing Claude Code authentication untouched.

### 4. Write ~/.frago/runtime.json
```json
{"schema_version": "1.0", "launcher": {"command": ["<absolute repo path>/.venv/bin/frago"], "mode": "global", "detected_at": "<ISO timestamp>", "source": {}}}
```
This is the Rust hook binary's only way to find the CLI; if missing or empty the hook silently does nothing (by design). On Windows the path MUST use forward slashes (`C:/Users/...`) because the hook may launch via Git Bash where backslashes are eaten as escapes.

### 5. Deploy the hook binary
The binary ships in the repo at `src/frago/bin/<platform>/frago-hook` (platforms: darwin-arm64, darwin-x86_64, linux-x86_64, windows-x86_64 with `frago-hook.exe`; git preserves the executable bit). Run `mkdir -p ~/.claude/hooks/frago/`, copy the binary there, and `chmod 755` on POSIX. On unsupported platforms (e.g. linux-aarch64) skip hooks — the CLI still works.

### 6. Merge hooks into ~/.claude/settings.json (NEVER overwrite the whole file)
First run `~/.claude/hooks/frago/frago-hook --supported-events` to get the authoritative event list (a JSON array; currently SessionStart, UserPromptSubmit, PreToolUse). For each event, append to the existing `hooks` section:
```json
{"matcher": "", "hooks": [{"type": "command", "command": "<absolute path to ~/.claude/hooks/frago/frago-hook>", "timeout": 10}]}
```
Skip any event that already has an entry with the same command. Do NOT write hook-rules.json (builtin rules are compiled into the binary), `~/.frago/books/`, AGENTS.md, or `~/.frago/.gitignore` — none are needed; other `~/.frago` subdirectories are lazily created by the CLI.

### 7. Verify
1. `<repo>/.venv/bin/frago --version` prints a version.
2. `<repo>/.venv/bin/frago book` prints the knowledge index.
3. `~/.claude/hooks/frago/frago-hook --supported-events` prints the event JSON array.
4. Restart Claude Code; a new session should show the frago knowledge index injected at SessionStart, proving the runtime.json → frago book chain works.
On any failure, check: runtime.json launcher path is absolute and exists; settings.json hook command path is correct; the binary is executable.

## Optional appendix: third-party API endpoint
Only if the user lacks official Anthropic auth and wants a compatible endpoint (deepseek/aliyun/kimi/minimax/custom): merge into the `env` section of ~/.claude/settings.json these seven keys — ANTHROPIC_BASE_URL, ANTHROPIC_MODEL, ANTHROPIC_DEFAULT_SONNET_MODEL, ANTHROPIC_DEFAULT_HAIKU_MODEL, API_TIMEOUT_MS, CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC, ANTHROPIC_API_KEY — and ensure ~/.claude.json contains `{"hasCompletedOnboarding": true, "lastOnboardingVersion": "1.0.0", "isQualifiedForDataSharing": false}` to skip the login onboarding. Example: deepseek base_url is `https://api.deepseek.com/anthropic`.


---

## About

Generated by **frago** — An Agent OS that turns ad-hoc agent runs into reusable recipes.

Install: `uv tool install frago-cli`
Homepage: https://frago.ai · Docs: https://docs.frago.ai
