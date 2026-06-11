---
name: frago-source-install
description: |
  This skill should be used when a user wants to install or configure frago from source on their machine (macOS/Linux/Windows) using their existing Claude Code. It guides the agent to clone the repository, build the environment with uv, run the frago server once so product code auto-deploys hooks and runtime config, then verify the artifacts — the user never types a frago command themselves. Trigger phrases: "install frago", "安装 frago", "配置 frago", "frago 源码安装", "set up frago", "frago installation", "deploy frago hooks", "从源码安装 frago".
license: AGPL-3.0
---

## Overview
Install and configure frago from source so the user's Claude Code gains the frago runtime (hooks, knowledge index, CLI). The agent performs every step; the user never types a frago command. Main path: clone → uv sync → run `frago server start` once so product code auto-completes all configuration → verify artifacts → ask whether to keep the resident server. Hand-writing config files is only a fallback when the server fails to start.

## Scope
Do NOT install the desktop (Tauri) client, Node.js, or Claude Code itself. Do NOT touch the user's existing Claude Code authentication. Third-party API endpoint setup is an optional appendix only.

## Steps

### 1. Prepare prerequisites (git + uv only)
Python need not be pre-installed — uv downloads a managed Python per `requires-python>=3.13`. All locked dependencies ship pre-built wheels except one pure-Python package, so no C/C++ toolchain is needed on any OS. Google Chrome is optional (browser automation only).
- macOS: ensure git via `xcode-select --install`; install uv via `curl -LsSf https://astral.sh/uv/install.sh | sh`.
- Linux: install git and curl via apt/dnf/pacman; uv same as macOS.
- Windows: `winget install Git.Git` (fallback: git-scm.com installer); uv via `powershell -c "irm https://astral.sh/uv/install.ps1 | iex"`.
- On POSIX, uv lands in `~/.local/bin`, which may not be on PATH in the current session — call it as `~/.local/bin/uv` or `source ~/.local/bin/env` first. Windows installer writes PATH to the registry; new processes see it.

### 2. Clone and build the environment
`git clone https://github.com/tsaijamey/frago.git` to a stable path such as `~/frago`. In regions where github.com is unreachable, prefix with a mirror: `https://mirror.ghproxy.com/https://github.com/tsaijamey/frago.git` or `https://ghproxy.net/...`. Then run `uv sync` inside the repository (no `--all-extras --dev` needed). The CLI entry point is `<repo>/.venv/bin/frago` (Windows: `.venv/Scripts/frago.exe`).

### 3. Main path: run the server once for automatic configuration
Execute `<absolute-repo-path>/.venv/bin/frago server start` and wait a few seconds. On startup the product code automatically: (a) deploys the platform hook binary to `~/.claude/hooks/frago/frago-hook`; (b) registers events from the binary's `--supported-events` output into `~/.claude/settings.json` via merge-write, leaving existing config untouched; (c) detects the launcher and writes `~/.frago/runtime.json`. This stays correct across frago versions, unlike hand-written snapshots.

### 4. Verify the three artifacts
- `~/.frago/runtime.json` exists with a non-empty `launcher.command`.
- `~/.claude/hooks/frago/frago-hook` (`.exe` on Windows) exists and is executable.
- The `hooks` section of `~/.claude/settings.json` registers frago-hook for SessionStart, UserPromptSubmit, and PreToolUse.
Also confirm `<repo>/.venv/bin/frago --version` and `frago book` produce output.

### 5. Ask about the resident server
The server provides the full agent-OS surface: Web UI at http://127.0.0.1:8093, scheduling, task intake (binds port 8093). Ask the user whether to keep it running. If not, run `frago server stop` — all on-disk configuration artifacts remain and the hook chain keeps working.

### 6. Final end-to-end check
Have the user restart Claude Code; a new session should inject the frago knowledge index at SessionStart, proving the runtime.json → hook → `frago book` chain works.

## Appendix A: Manual fallback (only if `server start` fails, e.g. port conflict)
Also usable as an acceptance checklist:
1. `~/.frago/config.json` (write only if absent, never overwrite): `{"schema_version": "1.0", "auth_method": "official", "init_completed": true}`. `auth_method=official` means the user's existing Claude Code auth is untouched.
2. `~/.frago/runtime.json`: `{"schema_version": "1.0", "launcher": {"command": ["<absolute-repo-path>/.venv/bin/frago"], "mode": "global", "detected_at": "<ISO timestamp>", "source": {}}}`. If missing or launcher empty, the hook silently does nothing (by design). Windows paths must use forward slashes (C:/Users/...).
3. `mkdir -p ~/.claude/hooks/frago/`, copy `<repo>/src/frago/bin/<platform>/frago-hook` there (platform dirs: darwin-arm64, darwin-x86_64, linux-x86_64, windows-x86_64; Windows filename frago-hook.exe; on platforms without a binary such as linux-aarch64 the hook is unavailable but the CLI works). POSIX: `chmod 755`.
4. Merge-write the `hooks` section of `~/.claude/settings.json` (never overwrite wholesale): run `<binary> --supported-events` to get the live event list (currently SessionStart/UserPromptSubmit/PreToolUse) and append per event `{"matcher": "", "hooks": [{"type": "command", "command": "<absolute path to ~/.claude/hooks/frago/frago-hook>", "timeout": 10}]}`, skipping commands already registered.
5. Do NOT hand-write: hook-rules.json (builtin rules are compiled into the binary), `~/.frago/books/` (system knowledge ships in the Python package), AGENTS.md/.gitignore; other `~/.frago` subdirectories are lazily created by the CLI.
Troubleshooting order: is the runtime.json launcher path absolute and existing → is the registered command path in settings.json correct → is the binary executable.

## Appendix B (optional): Third-party API endpoint
Only if the user has no official Anthropic auth: merge into the `env` section of `~/.claude/settings.json` the seven keys ANTHROPIC_BASE_URL, ANTHROPIC_MODEL, ANTHROPIC_DEFAULT_SONNET_MODEL, ANTHROPIC_DEFAULT_HAIKU_MODEL, API_TIMEOUT_MS, CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC, ANTHROPIC_API_KEY (deepseek example: base_url=https://api.deepseek.com/anthropic); and ensure `~/.claude.json` contains `{"hasCompletedOnboarding": true, "lastOnboardingVersion": "1.0.0", "isQualifiedForDataSharing": false}` to skip the official login onboarding.


---

## About

Generated by **frago** — An Agent OS that turns ad-hoc agent runs into reusable recipes.

Install: `uv tool install frago-cli`
Homepage: https://frago.ai · Docs: https://docs.frago.ai
