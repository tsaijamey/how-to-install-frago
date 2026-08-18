## What's in this release

v0.4.0 adds a decision the skill never made before installing: whether the target is a server or the install is running as root, and what that quietly costs.

## Added — a server/root check before the install begins

- **Installing frago as root, or on a server, is now a decision the agent raises with the user instead of walking past.** A fresh cloud VPS logs you in as root by default, so this happens constantly without anyone choosing it. Two consequences follow that neither the installer nor frago announces:
  - The frago server token is a credential for running commands as whatever account the server runs as — installed as root, that token is root on the whole box, not a sandbox.
  - Sub-agents run as that same account, and Claude Code refuses to run its unattended permission-bypass (`--dangerously-skip-permissions`) as root. So on a root install the web UI, recipes and knowledge index all work, but `frago agent`, the primary agent and `frago remote` silently do nothing — the worker is spawned, Claude Code exits on the root refusal, and frago waits for a readiness signal that never comes. No line says why.
- **The fix the skill now recommends:** run frago under a dedicated non-root user, and do the clone, the `uv` build, the resident server and every later `frago` command as that user — because the account that runs `frago server` is the account every agent inherits, so it must not be root. The server may still take root for a privileged port or a systemd unit.
- **Scoped to Claude Code as the driving runtime.** opencode and codex do not carry the root refusal, so the section is skipped for them.

## Known limitations

- Does not install the desktop (Tauri) client, Node.js, or Claude Code itself
- Hook binary unavailable on platforms without a shipped binary (e.g. linux-aarch64); the CLI still works there
- Resident server binds port 8093; port conflicts require the manual fallback
- `gh auth login` is interactive — the agent hands it to the user rather than answering its prompts
