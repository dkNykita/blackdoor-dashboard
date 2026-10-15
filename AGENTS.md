# Blackdoor Dashboard – instructions for AI agents

> For any agent (Claude, Gemini, Codex/GPT, Copilot…); `CLAUDE.md` only imports this file. Human-facing description: [README.md](README.md).

## What this is
Scripts for the game Blackdoor written in the game's own script language (`.bd`), in two generations:
- `shell/` – the original dashboard, kept as the working fallback. `shell/menu.bd` is the dashboard and launcher; scripts cannot launch other scripts, so it calls the action functions in `shell/utils/*.bd` directly by name, and a change of a name or signature in `shell/utils/` has to be mirrored in `shell/menu.bd`.
- `panel/` – the ground-up rewrite on Script Panels: entry point `panel/dash.bd`, modules `panel/dash/*.bd` (`store.bd` persistence, `safe.bd` session and trace safety). Milestones P1–P5 are done; P6 (GRAB + CLEANUP) and P7 (polish, port the remaining actions) are next.

## Notes
- The game runs the scripts from its saves and the repo mirrors them: `slot_0/mnt/Scripts/Active/menu.bd` and `utils/*.bd` → `shell/menu.bd` and `shell/utils/*.bd`; the dev save `slot_2/mnt/dash.bd` and `dash/*.bd` → `panel/dash.bd` and `panel/dash/*.bd` (`dash/data/` is runtime state, not mirrored). The owner copies changes between the game and the repo; the repo is also the backup, because some in-game machines can wipe a save.
- The game runs only on the owner's PC; changes cannot be executed in a cloud session, so describe how to verify them in the game.
- When in-game CPU or timing behaves oddly (e.g. `get_cpu()`), stay inside the game: script logic, the tick model, in-game diagnostics (`ps`, `jobs`, instrumentation, isolated test scripts). Do not blame the owner's real machine or OS; if it stays unexplained, say so and move to a practical mitigation.
- `shell/menu.bd` overloads the CPU at startup and was never fixed; the `panel/` rewrite replaces further tuning (it never calls `get_cpu()`). The plan, the per-phase results and the ruled-out directions are in the owner's notes (vault `Projects/Games/Blackdoor/`: `dashboard-stav.md`, `Internal/panel rewrite/`, `Internal/field_notes.txt`); do not retry ruled-out directions blindly.
- Non-core content (notes, experiments) is kept outside the repo; `external/` is ignored.
- Communicate in Czech unless the owner writes otherwise; be concise and summarise at the end.

## Git
- Never force-push, rewrite history, `git add -f`, `--no-verify` or `git clean`; changes to `main` go through a pull request.
- Hooks in `.githooks/` (enable with `git config core.hooksPath .githooks`) block secrets, files over 2 MB and force-pushes.
