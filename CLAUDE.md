# Palworld Mods — project instructions

**At the start of every session, read [`PALWORLD_MODDING_REFERENCE.md`](PALWORLD_MODDING_REFERENCE.md)
first.** It's the consolidated knowledge base of Palworld UE4SS internals — confirmed function
levers, the EXP/level chain, the pal AI/aggression/detection + hate systems, key data tables, and
the gotchas/lessons that cost real time to learn. Reading it avoids re-deriving everything.

Then skim:
- [`PROGRESS.md`](PROGRESS.md) — live checklist of what's done and what's next.
- [`README.md`](README.md) — repo layout and each mod's purpose.

## Working style / preferences
- **Goal:** make the game HARDER. Reject any design that lets the player come out ahead
  (e.g. a death penalty that grants points). Prefer data-driven where it lowers runtime load.
- **Repos:** each mod is its OWN public GitHub repo — `Xidorian/PunishingDeath`,
  `Xidorian/PredatorStealth`. THIS folder is the shared hub (`Xidorian/Palworld-Mods`): the
  knowledge base, PROGRESS, PalClassification.csv, and art only — no mod source lives here.
  Mods are cloned as sibling folders (`..\PunishingDeath\`, `..\PredatorStealth\`).
- **Git:** I commit locally as we go after each meaningful change and tell the user; the user runs
  `git push` (in whichever repo the change is in).
- **Deploy loop:** edit `Scripts\main.lua` in that mod's repo folder → copy into
  `Palworld\Mods\NativeMods\UE4SS\Mods\<Mod>\Scripts\` → user focuses the game to auto-reload.
  Read `UE4SS.log` for our tagged `[..]` output. Ctrl+R hot-reload is broken here — use the
  file-watcher; turn it OFF and do a game restart when finalizing (clears orphaned scan loops).
- I read the UE4SS log for the user — they just press keys in-game and say "done".
