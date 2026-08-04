## What's in this release

v0.3.0 fixes an install path that no longer worked and extends the skill past "the software is on disk" to "the user can actually use it".

## Fixed — the previous version failed partway through

- **Main path was refused.** frago now rejects every command run from its own source checkout except `server`, and refuses to run the server out of the repository's virtual environment. The old instruction `<repo>/.venv/bin/frago server start` exits with a refusal that reads like a broken install. The skill now uses `uv run frago server start` from inside the repository — the packaging path, which builds a wheel, installs it as the system frago, and hands over.
- **Verification looked for the wrong things.** The hook binary is `~/.frago/bin/frago-core`, not `~/.claude/hooks/frago/frago-hook`, and its registered command must end in `--engine` — that binary holds two programs, and without the flag hooks silently start the wrong one.
- **`runtime.json` is gone.** Nothing reads it any more. Every instruction to create or verify it has been removed.

## Added — what was missing after a successful install

- **GitHub backup, offered as a choice.** Everything a user accumulates — recipes, knowledge domains, routing rules — lives in `~/.frago` and no upgrade recreates it. The skill now walks through installing and authenticating the gh CLI, placing the ignore rules *before* the first commit, checking that no key file is staged, and creating a private `frago-working-dir` repository.
- **Model profiles.** Sub-agents need a model to run on, and a fresh install has none. Configured through the local settings page, never by hand — the file holds plaintext keys.
- **Recipe credentials.** Recipes that call an outside service fail with `api_key missing` until their key is entered in the recipe's own credentials dialog. Keys are typed into the page, not pasted into the chat.
- **opencode users.** The bridge deploys automatically alongside the Claude Code hook; the skill now says what to verify.
- **PATH is load-bearing.** Hooks invoke the bare `frago` command. If `~/.local/bin` is missing from the permanent PATH, hooks still fire but every knowledge injection comes back empty and nothing announces why.
- **Troubleshooting by symptom**, written from what the user sees rather than what the system did — including the source-checkout refusal, empty injections, port 8093 conflicts, and gh login on a machine with no browser.

## Known limitations

- Does not install the desktop (Tauri) client, Node.js, or Claude Code itself
- Hook binary unavailable on platforms without a shipped binary (e.g. linux-aarch64); the CLI still works there
- Resident server binds port 8093; port conflicts require the manual fallback
- `gh auth login` is interactive — the agent hands it to the user rather than answering its prompts
