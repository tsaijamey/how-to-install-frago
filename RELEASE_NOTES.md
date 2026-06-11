## What's in this release
Initial release of frago-source-install: a complete, verified workflow for installing frago from source and configuring it for Claude Code by writing files directly, with no reliance on frago's own init/server auto-configuration.

## Features
- Cross-platform prerequisite setup (macOS / Linux / Windows) needing only git and uv — no pre-installed Python, no compiler toolchain
- Mirror-prefix clone fallback for regions where github.com is unreachable
- Direct-write configuration: ~/.frago/config.json (non-destructive), ~/.frago/runtime.json launcher wiring, hook binary deployment with correct permissions
- Safe merge of the hooks section into ~/.claude/settings.json, with the event list fetched dynamically from the binary's --supported-events output and duplicate-registration skipping
- Windows-specific path guidance (forward slashes for Git Bash hook execution)
- Four-step verification checklist covering CLI, knowledge index, hook binary, and the live SessionStart injection
- Optional appendix for third-party Anthropic-compatible API endpoints

## Known limitations
- Hook functionality is unavailable on platforms without a bundled binary (e.g. linux-aarch64); the CLI still works there
- Does not install the desktop (Tauri) client, Node.js, or Claude Code itself
- Does not modify existing Claude Code authentication