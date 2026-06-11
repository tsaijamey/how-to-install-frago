## What's in this release
Initial release (v0.1.0) of the source-install skill: a complete, verified workflow for installing frago from a git clone and configuring it for Claude Code by writing configuration files directly, with no reliance on built-in setup commands.

## Features
- Per-OS environment preparation for macOS, Linux, and Windows, including the Git Bash requirement that makes hooks work on Windows
- Clone and build with uv (managed Python auto-download, no compiler toolchain needed) plus mirror fallbacks for restricted networks
- Hand-written configuration replacing the auto-setup flow: config.json, runtime.json launcher wiring, hook binary deployment, and safe merge-only edits to Claude Code settings.json
- Optional appendix for third-party Anthropic-compatible API endpoints
- Four-step verification covering CLI, knowledge index, hook binary, and the end-to-end SessionStart injection

## Known limitations
- Hook functionality requires a prebuilt binary for the platform (darwin-arm64, darwin-x86_64, linux-x86_64, windows-x86_64); other platforms get CLI-only
- Does not install the desktop client, Node.js, or Claude Code itself
- Does not touch existing Claude Code authentication