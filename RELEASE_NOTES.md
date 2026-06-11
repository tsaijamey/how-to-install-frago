## What's in this release

Initial release of the source-install and manual-configuration workflow, verified layer by layer against the actual source code rather than assembled from docs.

## Features

- Per-OS prerequisite setup (macOS / Linux / Windows) using git and uv, with Python auto-provisioned by uv per the project's `requires-python` constraint
- Clone-and-build steps producing a ready CLI entry point inside the repo's `.venv`
- Hand-written configuration that replaces the init wizard and the server's auto-configuration: runtime launcher config, per-platform hook binary deployment with correct permissions, and merge-safe hook registration into `~/.claude/settings.json` (never overwriting existing user settings)
- Dynamic hook-event discovery via the binary's `--supported-events` output, so the registration stays correct as supported events evolve
- Windows-specific path guidance (forward slashes required because hooks launch through Git Bash)
- Optional appendix for routing Claude Code to a third-party Anthropic-compatible endpoint (deepseek, aliyun, kimi, minimax, custom) including the onboarding-skip fields
- A four-step verification checklist with failure-mode triage for each link of the hook chain

## Known limitations

- Hook functionality is unavailable on platforms without a bundled binary (e.g. linux-aarch64); the CLI itself still works
- The workflow assumes Claude Code is already installed; it configures hooks and endpoints but does not install Claude Code itself