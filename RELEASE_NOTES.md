## What's in this release

v0.5.0 fixes the thing every install ended on: it worked, and the user had seen none of it. The skill now finishes by putting a page in front of them and letting the machine prove itself.

## Added — the install ends with something to look at

- **New step 9: install and open `frago_welcome`.** frago ships no recipes of its own, so the skill fetches this one from the community repository (`frago recipe install community:frago_welcome`) and runs it. The page opens in the user's own default browser — six screens, each with its own address (`#1`…`#6`): what changed with the install, what they can ask for right now, a live demo, how that demo worked, and where to go next.
- **The demo is the point.** The user types a rule of their own on the page and presses a button; it is written into a knowledge domain on their machine. The page then sends them back to the terminal to ask the agent about that rule. Nothing in the conversation carries it — the agent has to read it off the machine to answer. That is the whole difference between frago and a capable chat window, shown rather than described.
- **The skill is told how to answer it.** Look before speaking, and read the rule back from `frago my-rules find` rather than from having watched the user type it. Repeating what was on screen proves nothing.
- **Step 10 (the end-to-end check) now leans on it.** The stored rule survives a Claude Code restart, so answering from it after the restart exercises the same chain the session-start banner only hints at.
- **The page can also answer for itself.** On the comparison screen, each row has a "not really" button that hands its question to a real agent session on this machine and prints the answer; with model profiles configured (step 6) it answers in seconds, and without one it falls back to a written sample and says on screen that it is a sample.
- **A failed install of the page is not a failed install of frago.** No network or an unreachable repository is reported plainly and the skill moves on.

## Fixed — the prerequisites section was both wrong and short

- **"Google Chrome is optional (browser automation only)" is gone.** It was wrong in both halves: the default browser backend drives **Edge's** real profile through an extension, and Chrome Stable has silently ignored the extension-loading flag since v137 — so a Chrome-only machine cannot use that path at all.
- **Four dependencies that were never mentioned now are**, each tied to the one capability it serves: **tmux** for delegation (`frago agent`, the primary agent, `frago remote`), **Edge** for browser automation, **ffmpeg** for recording a tab or the virtual desktop, and **bubblewrap** for running recipes on Linux. frago installs and starts without any of them, which is why their absence goes unnoticed until one capability quietly does nothing — the same shape as the root-install problem in v0.4.0.
- **tmux is the one that matters most.** Without it, step 6 (model profiles) buys nothing: the profile is configured, the worker still never starts, and the only message is `Error: tmux not found`.
- **The skill does not install any of them unattended.** It asks what the user intends to use, sets that up, and says plainly what stays unavailable. Troubleshooting now names each missing-dependency symptom.

## Known limitations

- Does not install the desktop (Tauri) client, Node.js, or Claude Code itself
- Hook binary unavailable on platforms without a shipped binary (e.g. linux-aarch64); the CLI still works there
- Resident server binds port 8093; port conflicts require the manual fallback
- `gh auth login` is interactive — the agent hands it to the user rather than answering its prompts
- The welcome page needs the resident server (step 8): stopping it also takes the page away
