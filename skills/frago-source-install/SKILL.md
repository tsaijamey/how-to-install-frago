---
name: frago-source-install
description: |
  This skill should be used when a user wants to install or configure frago from source on their machine (macOS/Linux/Windows) using their existing Claude Code. It guides the agent to clone the repository, build the environment with uv, publish it as the system frago by running the server once, verify the deployed artifacts, offer to back the user's frago working directory up to a private GitHub repository via the gh CLI, and set up the credentials frago needs to actually do work — model profiles for sub-agents and per-recipe API keys — through the local web settings page, and finish by opening a welcome page that has the machine prove what changed. The user never types a frago command themselves. Trigger phrases: "install frago", "安装 frago", "配置 frago", "frago 源码安装", "set up frago", "frago installation", "deploy frago hooks", "从源码安装 frago", "配置 frago 模型", "frago profile", "配方没有 api key", "frago 备份", "frago-working-dir".
license: AGPL-3.0
---

## Overview
Install frago from source so the user's Claude Code (and opencode, if present) gains the frago runtime — hooks, knowledge index, CLI — then set up the two things without which most of frago sits idle: a backup of the working directory, and the credentials that let frago delegate work and call outside services. The agent performs every terminal step; the user's own hands are needed twice only — a browser login for GitHub, and typing API keys into a local web page, which is the one place a key should never be pasted into a chat.

Main path: clone → `uv sync` → run the server once from the checkout (this is also the packaging step) → verify four artifacts → offer GitHub backup → configure model profiles and recipe credentials in the web UI → decide about the resident server → open the welcome page → end-to-end check.

## Scope
Do NOT install the desktop (Tauri) client, Node.js, or Claude Code itself. Do NOT run `frago init` — it installs Claude Code and rewrites authentication, which is not what a user asking to install frago is asking for. Do NOT touch the user's existing Claude Code authentication. Hand-writing config files is a fallback for when the server will not start, not a normal step.

## Before you install: is this a server, and are you root?
Most installs are one person's own machine — this section does not apply, install and move on. It matters when the target is a server the user will reach over the network, or when the install is running as **root**. A fresh cloud VPS logs you in as root by default, so this happens constantly without anyone choosing it. Check once, early: `id -u` is `0`, or the user talks about a server, a domain, "部署", "公网", exposing a page. Two things then go wrong, and neither prints an error:

- **The frago server token is a credential for running commands as whatever account the server runs as.** Installed as root, that token — and anything that slips past the access gate — is root on the whole box, not a sandbox.
- **frago's sub-agents run as that same account, and Claude Code refuses to run its unattended permission-bypass (`--dangerously-skip-permissions`) as root.** So on a root install the agent side never starts: the web UI, recipes and knowledge index all work, but delegation (`frago agent`), the primary agent, and `frago remote` silently do nothing — the worker is spawned, Claude Code exits on the root refusal, and frago waits for a readiness signal that never comes. There is no line that says why. (This breakage is specific to how Claude Code guards root; it does not touch the token risk above, which stands regardless.)

So before installing on a server or as root, raise it with the user as a decision, not a step to push through — say what proceeding as root costs (a root-equivalent token, and an agent side that is silently dead), and that the fix is to run frago under a **dedicated non-root user**: create one (`useradd -m -s /bin/bash frago` or similar), and do the clone, the `uv` build, the resident server, and every later `frago` command as that user. The server may still need root for a privileged port or a systemd unit, but the account that runs `frago server` is the account every agent inherits, so that account must not be root. If the user understands the cost and still wants a root install — a throwaway box, no delegation needed — proceed, but state plainly what will not work.

This concerns Claude Code as the runtime driving the install. If the user runs the installing agent through opencode or codex instead, skip this section — those do not carry the root refusal, and this problem is out of scope for them.

## The one rule that breaks installs
frago refuses to run from its own source checkout. Every command except `server` exits with a refusal, and the server refuses to run out of the repository's virtual environment. This is deliberate: repository code paired with a system-installed server is a combination no user runs.

So there are exactly two forms:
- Inside the repository, only this: `uv run frago server start` (or `restart`). It bumps the patch version, builds a wheel, installs it as the system frago via `uv tool install --force`, and hands over. This is how source code becomes the installed product.
- Everywhere else, the plain `frago` command, which after the step above lives at `~/.local/bin/frago`.

Never suggest `<repo>/.venv/bin/frago <anything>`. It will be refused, and the refusal message reads like a failed install to a non-technical user.

## Steps

### 1. Prepare prerequisites
**To install frago at all, git and uv are the whole list.** Python need not be pre-installed — uv downloads a managed Python per `requires-python>=3.13`. All locked dependencies ship pre-built wheels, so no C/C++ toolchain is needed on any OS.
- macOS: ensure git via `xcode-select --install`; install uv via `curl -LsSf https://astral.sh/uv/install.sh | sh`.
- Linux: install git and curl via apt/dnf/pacman; uv same as macOS.
- Windows: `winget install Git.Git` (fallback: git-scm.com installer); uv via `powershell -c "irm https://astral.sh/uv/install.ps1 | iex"`.
- On POSIX, uv lands in `~/.local/bin`, which may not be on PATH in the current session — call it as `~/.local/bin/uv` or `source ~/.local/bin/env` first.

**Four more, each only for a particular capability.** frago installs and starts without any of them, which is exactly why they are worth naming here: what they break, breaks quietly — one capability at a time, long after the install was declared a success. Do not install them behind the user's back. Ask what they intend to use frago for, set up what that needs, and say plainly what stays unavailable.

| Needed for | What | Missing means |
|---|---|---|
| Delegating work (`frago agent`), the primary agent, `frago remote` | **tmux** | The worker never starts; `Error: tmux not found` is the whole message. Step 6, configuring model profiles, buys nothing without it |
| Browser automation (`frago browser`) | **Microsoft Edge**, or another Chromium-family browser | `frago browser check` lists every browser as not found and asks for Edge |
| Recording a tab, or the virtual desktop (`frago desktop`) | **ffmpeg** | The recording call fails outright; everything else keeps working. The virtual desktop also wants tmux and Edge |
| Running recipes **on Linux** | **bubblewrap** (`bwrap`) | Recipes are refused rather than run unconfined. macOS uses its own sandbox and needs nothing extra |

Where they come from: on macOS, tmux and ffmpeg via Homebrew, Edge from microsoft.com/edge as a normal app. On Linux, tmux, ffmpeg and bubblewrap from the distribution's package manager; Edge or Chromium from its repository. On Windows, Edge ships with the system and ffmpeg comes from winget, but tmux does not exist natively — delegation there needs WSL.

**Edge, not Chrome.** frago's default browser backend drives Edge's *real* profile through an extension, which is what makes the user's existing logins usable. Chrome Stable has silently ignored the extension-loading flag since v137, so a Chrome-only machine cannot take that path at all. Earlier versions of this skill said "Google Chrome is optional (browser automation only)" — wrong in both halves, and it pointed people down a road that does not go anywhere.

`~/.local/bin` must end up on the user's permanent PATH, not just this shell. frago's hooks invoke the bare command `frago`; if the shell that launches Claude Code cannot find it, hooks still fire but every knowledge injection comes back empty, and nothing announces why. Check the user's shell profile and add it if missing.

### 2. Clone and build the environment
`git clone https://github.com/tsaijamey/frago.git` to a stable path such as `~/frago`. In regions where github.com is unreachable, prefix with a mirror: `https://mirror.ghproxy.com/https://github.com/tsaijamey/frago.git` or `https://ghproxy.net/...`. Then run `uv sync` inside the repository (no `--all-extras --dev` needed). Keep the checkout — later steps read a file from it, and upgrades are a `git pull` plus one server restart.

### 3. Publish and start: run the server once from the checkout
Run `uv run frago server start` from inside the repository and wait. Expect output naming the version it bumped to, the wheel it built, and the handover to the system frago. On startup the product code automatically:
- deploys the platform hook binary to `~/.frago/bin/frago-core` and deletes stale copies from older layouts;
- registers events in `~/.claude/settings.json` by merge-write, leaving unrelated hooks untouched;
- deploys the opencode bridge to `~/.config/opencode/plugin/` when opencode is installed.

Deployment finishes roughly twenty seconds into startup, not instantly. Checking too early shows a half-configured machine; wait for the server to answer before verifying.

### 4. Verify four artifacts
Run these as the plain `frago` command, from any directory that is not the checkout:
- `frago --version` prints the version that was just installed.
- `frago book` prints the knowledge index.
- `~/.frago/bin/frago-core` exists and is executable (`frago-core.exe` on Windows).
- The `hooks` section of `~/.claude/settings.json` registers, for SessionStart, UserPromptSubmit and PreToolUse, a command ending in `frago-core --engine` with `timeout: 10`.

The `--engine` flag is not optional. That binary holds two programs: without the flag it starts the agentic kernel instead of routing the hook event, and hooks go silently dead.

There is no `~/.frago/runtime.json` any more. If an older guide tells you to create or check one, ignore it — nothing reads that file.

If the user runs opencode, also confirm `~/.config/opencode/plugin/` contains `frago-hook.js` alongside three `.json` files.

### 5. Offer to back up the working directory to GitHub
Everything the user accumulates through frago — recipes they build, knowledge domains they fill, routing rules, run history — lives in `~/.frago`. It is not in the repository they just cloned and no upgrade recreates it. frago is designed to keep that directory as a git repository mirrored to a **private** GitHub repository named `frago-working-dir`, which is also what makes a second machine possible.

Raise this as a choice, not a step to push through: explain what is being backed up and where, and ask whether the user wants it. If they decline, say plainly that their recipes and knowledge then exist on this machine only, and move on — everything else works without it.

If they accept:

**a. Make sure the gh CLI is there and logged in.** Check `gh --version` and `gh auth status`. Install if missing — macOS `brew install gh`, Debian/Ubuntu `sudo apt install gh`, Fedora `sudo dnf install gh`, Windows `winget install GitHub.cli`.

`gh auth login` is interactive: it asks a few questions and opens a browser for a one-time code. The agent cannot answer those prompts on the user's behalf, so hand it over explicitly — in Claude Code the user can run it in place by typing `!gh auth login`, and the login is complete when `gh auth status` reports the account. Users who have no GitHub account create one at github.com first; the login flow will not do it for them.

**b. Put the ignore rules in place before anything is committed.** `~/.frago` holds `profiles.json`, `recipes.local.json` and `config.json`, all of which contain API keys in plain text. The package ships the authoritative ignore list; nothing deploys it automatically, so copy it from the checkout:

`cp <repo>/src/frago/resources/frago-home-gitignore.template ~/.frago/.gitignore`

**c. Confirm no secret is staged, before the first push.** Run `git -C ~/.frago init` (skip if already a repository), then `git -C ~/.frago status --short` and read the list. If `profiles.json`, `recipes.local.json`, `config.json` or any `.env` file appears, stop — the ignore file did not land. Never push past this check.

**d. Create the private repository and push.** `git -C ~/.frago add -A && git -C ~/.frago commit -m "frago working dir"`, then `gh repo create frago-working-dir --private --source ~/.frago --push`. Tell the user the repository is private and why that matters: it holds their work, their notes, and the shape of what they automate.

Later syncs are ordinary git — commit and push from `~/.frago`. On a second machine, clone that repository into `~/.frago` before installing frago there.

### 6. Configure model profiles (needed before frago can delegate work)
frago runs sub-agents for the user — research, monitoring, long jobs — and each sub-agent needs a model to run on. Those live in model profiles, and a fresh install has none. Without one, delegation falls back to whatever the user's own Claude Code is authenticated as, and `frago agent --use-profile <name>` has nothing to select.

Profiles are created in the web settings page, never by hand: the file holds plaintext API keys and is written with owner-only permissions. Open `http://127.0.0.1:8093`, go to Settings, and use the Profiles panel. Each profile needs a display name, an endpoint type, an API key, and — for non-Anthropic endpoints — a base URL and the model names to use. Ask the user which provider they already pay for; do not recommend signing up for anything.

Tell the user plainly: paste the key into that page, not into the chat. Verify afterwards with `frago profile list`, which prints saved profiles with keys masked. That command is read-only by design — creating and editing happen only in the web UI.

### 7. Configure recipe credentials (only for recipes the user will actually run)
Recipes that call an outside service read their key from `~/.frago/recipes.local.json`. A recipe with no key fails immediately with a message naming what is missing — for example the built-in image-reading recipe reports `api_key missing` and stops.

Do not pre-fill keys for recipes the user has not asked for. When a recipe does fail this way, open `http://127.0.0.1:8093`, go to Recipes, open that recipe, and have the user fill its credentials dialog — the fields are generated from what the recipe itself declares, so only the keys it truly needs are asked for. Same rule as profiles: the key goes into the page, not the chat.

### 8. Decide about the resident server
The server provides the web UI on port 8093, scheduling, and task intake. Steps 6 and 7 need it running. Ask whether to keep it. If the user says no, run `frago server stop` — every on-disk artifact remains and the hook chain keeps working; only the web UI and scheduling go away, and the user will need `frago server start` again to change credentials later.

### 9. Hand over the welcome page
At this point everything works and the user has seen none of it. frago ships no recipes of its own, so fetch the one that introduces it from the community repository and run it — both commands, in this order, every install:

```bash
frago recipe install community:frago_welcome
frago recipe run frago_welcome
```

The first pulls the recipe from `tsaijamey/frago-recipe-community`; the second prepares the page and hands it to the user's **own default browser** (the recipe returns `open_url` and the runner opens it). Do not use `frago browser navigate` for this — that drives the agent's controlled browser, which the user is not looking at. Do not paste the address and ask them to open it either; running the recipe is what opens it.

Six screens, each with its own address (`#1`…`#6`, so any one of them can be reopened or sent to someone): what changed with the install, what they can ask for right now, a live demo, how that demo worked, and where to go next.

Two things on the page reach back into this machine, and both are worth knowing about before the user presses them:

- **The demo on screen four** — the user writes a rule of their own, presses a button, and it is stored in a knowledge domain here. The page then sends them back to this terminal to ask you about that rule.
- **The "not really" buttons on screen two** — each one hands its question to a real agent session on this machine and prints the answer. That needs a model to be reachable: with step 6 done it answers in a few seconds; without one it falls back to a written sample and says on screen that it is a sample. Either way nothing breaks.

When the user comes back and asks about the rule they stored, **answer it the way you would answer anything: look before you speak.** frago LightAgent routes the question to the domain it belongs in; if it does not, `frago my-rules find` reads it directly. Never tell them what their rule was from having watched them type it — that proves nothing. Read it back from the machine.

Say the page is open and let them drive it. Do not narrate the screens.

If either command fails (no network, repository unreachable), say so plainly and move on to the check below — the install itself is fine without the page.

### 10. Final end-to-end check
Have the user restart Claude Code. A new session should show frago's knowledge index injected at session start, which proves the whole chain: PATH finds `frago`, settings.json points at the binary, the binary routes the event, the CLI answers.

If step 9 ran, the rule the user stored is a stronger check than the banner: it survives the restart, and answering from it exercises the same chain end to end.

## Troubleshooting, by what the user sees
- "Refusing to run: this frago comes from the source checkout" — a command was run from inside the repository. Run it from elsewhere as the plain `frago`, or, if it is genuinely the server, use `uv run frago server start` from the repository.
- Sessions start with no frago knowledge — check PATH first (`frago --version` in a fresh terminal), then that the registered command in settings.json ends with `--engine`, then that `~/.frago/bin/frago-core` is executable.
- `frago: command not found` — `~/.local/bin` is not on the permanent PATH; fix the shell profile rather than using absolute paths.
- Port 8093 already in use — another frago server is already running; `frago server status` confirms it. Do not start a second one.
- A recipe stops with `api_key missing` — that is step 7, not a broken install.
- Delegation does nothing: `frago agent` returns without a worker ever starting — tmux is missing (step 1), or this is a root install (see the section above). Configuring more model profiles will not help either one.
- `frago browser check` shows every browser as not found — no Chromium-family browser is installed; Edge is the one to add, and Chrome alone will not do (step 1).
- A recording fails while everything else works — ffmpeg is missing (step 1).
- On Linux, recipes are refused before they run — bubblewrap is missing (step 1). This is a refusal on purpose, not a crash.
- `gh auth login` opens no browser (headless or remote machine) — choose the device-code path it offers and open the URL on any other device; the code is short-lived, so retry rather than reuse an expired one.
- `gh repo create` reports the name is taken — the user already has a `frago-working-dir`, probably from another machine. Clone that one into `~/.frago` instead of creating a second.

## Appendix: manual fallback (only when `uv run frago server start` fails outright)
Also usable as an acceptance checklist:
1. `~/.frago/config.json` (write only if absent, never overwrite): `{"schema_version": "1.0", "auth_method": "official", "init_completed": true}`. `auth_method=official` means the user's existing Claude Code auth is untouched.
2. `mkdir -p ~/.frago/bin/`, copy `<repo>/src/frago/bin/<platform>/frago-core` there (platform directories: darwin-arm64, darwin-x86_64, linux-x86_64, windows-x86_64; Windows filename `frago-core.exe`; on platforms without a binary, such as linux-aarch64, hooks are unavailable but the CLI works). POSIX: `chmod 755`.
3. Merge-write the `hooks` section of `~/.claude/settings.json`, never overwriting wholesale. Run `<binary> --supported-events` for the live event list, and append per event `{"matcher": "", "hooks": [{"type": "command", "command": "<absolute path to frago-core> --engine", "timeout": 10}]}`, skipping commands already registered. On Windows write the path with forward slashes — Claude Code launches hooks through Git Bash, which eats backslashes.
4. Do NOT hand-write: `hook-rules.json` (builtin rules are compiled into the binary), `~/.frago/books/` (knowledge ships inside the Python package), `profiles.json` or `recipes.local.json` (both hold plaintext keys and belong in the web UI), AGENTS.md. Other `~/.frago` subdirectories are created lazily. The one file worth placing by hand is `~/.frago/.gitignore`, and only from the template named in step 5.

## Appendix: third-party endpoint for Claude Code itself (optional)
Only when the user has no official Anthropic authentication, and only for their own Claude Code — this is separate from frago's model profiles in step 6, which serve sub-agents. Merge into the `env` section of `~/.claude/settings.json` the seven keys ANTHROPIC_BASE_URL, ANTHROPIC_MODEL, ANTHROPIC_DEFAULT_SONNET_MODEL, ANTHROPIC_DEFAULT_HAIKU_MODEL, API_TIMEOUT_MS, CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC, ANTHROPIC_API_KEY (deepseek example: base_url=https://api.deepseek.com/anthropic); and ensure `~/.claude.json` contains `{"hasCompletedOnboarding": true, "lastOnboardingVersion": "1.0.0", "isQualifiedForDataSharing": false}` to skip the official login onboarding.


---

## About

Generated by **frago** — An Agent OS that turns ad-hoc agent runs into reusable recipes.

Install: `uv tool install frago-cli`
Homepage: https://frago.ai · Docs: https://docs.frago.ai
