## What's in this release
Initial release of the frago source-install skill (v0.2 hybrid architecture): server-driven automatic configuration as the main path, with manual config writing demoted to a fallback appendix.

## Features
- Cross-platform prerequisite setup (macOS / Linux / Windows) needing only git and uv — Python is auto-managed by uv, no compiler toolchain required
- Mirror-prefix clone instructions for regions where github.com is unreachable
- One-shot `server start` auto-configuration: hook binary deployment, settings.json event registration via merge-write, runtime.json launcher detection
- Artifact verification checklist covering runtime.json, the hook binary, and settings.json registrations, plus an end-to-end SessionStart smoke test
- User choice on keeping the resident server; stopping it preserves all configuration
- Manual fallback appendix doubling as an acceptance checklist, with troubleshooting order
- Optional third-party API endpoint appendix for users without official Anthropic auth

## Known limitations
- Does not install the desktop (Tauri) client, Node.js, or Claude Code itself
- Hook binary unavailable on platforms without a shipped binary (e.g. linux-aarch64); the CLI still works there
- Resident server binds port 8093; port conflicts require the manual fallback