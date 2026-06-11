## What's in this release

First public release of the frago-source-install skill. It teaches Claude Code to install frago from source and configure it entirely by writing files — no `frago init`, no `frago server start`.

## Features

- Cross-platform prerequisite setup (macOS / Linux / Windows) using git and uv; Python is downloaded automatically by uv per the project's `requires-python >= 3.13`
- Repository clone and `uv sync` environment build, with the CLI entry point at `<repo>/.venv/bin/frago`
- Direct configuration writes: `~/.frago/config.json` (only when absent), `~/.frago/runtime.json` carrying the launcher path the Rust hook depends on, hook binary deployment to `~/.claude/hooks/frago/`, and a safe merge of hook registrations into `~/.claude/settings.json`
- Event list fetched dynamically from `frago-hook --supported-events`, so the skill stays correct as versions evolve
- Four-step verification covering the CLI, the knowledge index, the hook binary, and end-to-end SessionStart injection, with targeted troubleshooting pointers
- Optional appendix for Anthropic-compatible third-party API endpoints (deepseek, aliyun, kimi, minimax, custom)

## Known limitations

- Hook features are unavailable on platforms without a bundled binary (e.g. linux-aarch64); the CLI itself still works
- Does not install the desktop (Tauri) client, Node.js, or Claude Code itself
- Never modifies existing Claude Code authentication; third-party endpoint setup is opt-in only