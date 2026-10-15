# Blackdoor Dashboard – instructions for AI agents

> For any agent (Claude, Gemini, Codex/GPT, Copilot…); `CLAUDE.md` only imports this file. Human-facing description: [README.md](README.md).

## What this is
Scripts for the game Blackdoor written in the game's own script language (`.bd`): `menu.bd` is the target dashboard and launcher, `utils/*.bd` hold the actions it calls. Scripts cannot launch other scripts, so `menu.bd` calls the action functions in `utils/` directly.

## Notes
- `menu.bd` calls the action functions in `utils/*.bd` by name, so a change of a name or signature in `utils/` has to be mirrored in `menu.bd`.
- Non-core content (notes, experiments) was moved out of the repo; `external/` is ignored.
- The game runs only on the owner's PC; changes cannot be executed in a cloud session, so describe how to verify them in the game.
- The game runs the scripts from the save: `slot_0/mnt/Scripts/Active/menu.bd` and `utils/*.bd` map to `menu.bd` and `utils/*.bd` here (the dev save `slot_2` maps to `panel/`). The owner copies changes between the game and the repo; the repo is also the backup, because some in-game machines can wipe a save.
- When in-game CPU or timing behaves oddly (e.g. `get_cpu()`), stay inside the game: script logic, the tick model, in-game diagnostics (`ps`, `jobs`, instrumentation, isolated test scripts). Do not blame the owner's real machine or OS; if it stays unexplained, say so and move to a practical mitigation.
- `menu.bd` overloads the CPU at startup and was never fixed; a rewrite on Script Panels (`panel/`, save `slot_2`) replaces further tuning, `shell/` stays as the fallback. Ruled-out directions and the leading theory are in the owner's notes (vault `Projects/Games/Blackdoor/dashboard-stav.md`); do not retry them blindly.
- Communicate in Czech unless the owner writes otherwise; be concise and summarise at the end.

## Git
- Never force-push, rewrite history, `git add -f`, `--no-verify` or `git clean`; changes to `main` go through a pull request.
- Hooks in `.githooks/` (enable with `git config core.hooksPath .githooks`) block secrets, files over 2 MB and force-pushes.
