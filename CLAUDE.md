# Palworld Mods — project instructions

**At the start of every session, read [`PALWORLD_MODDING_REFERENCE.md`](PALWORLD_MODDING_REFERENCE.md)
first.** It's the consolidated knowledge base of Palworld UE4SS internals — confirmed function
levers, the EXP/level chain, the pal AI/aggression/detection + hate systems, key data tables, and
the gotchas/lessons that cost real time to learn. Reading it avoids re-deriving everything.

Then skim:
- [`PROGRESS.md`](PROGRESS.md) — live checklist of what's done and what's next.
- [`PLANNED_MODS.md`](PLANNED_MODS.md) — future mods and ideas not yet started.
- [`README.md`](README.md) — repo layout and each mod's purpose.

## The hub is our knowledge base
This `Palworld-Mods` folder is the umbrella project and single source of truth: the bulk of our
data and findings (`PALWORLD_MODDING_REFERENCE.md`, `PalClassification.csv`, `PROGRESS.md`, art).
The mods are "sub" projects of it in the loosest sense only — **nothing in a mod literally points
at this hub.** No mod imports it, reads a path into it, or depends on it in any way; a shipped mod
is fully standalone. This folder is purely a reference database **we** (you + Claude) consult while
building — the connection lives in our workflow, not in the software:
- **Pull from past experience first.** Before deriving anything, check here — odds are we already
  confirmed the lever, table, or gotcha in a previous mod.
- **Add new discoveries back here.** Any internal we confirm or lesson we learn while working on a
  mod gets written up in the hub (usually `PALWORLD_MODDING_REFERENCE.md`), not left buried in one
  mod's code — so everything stays accessible from this one location and every mod benefits.

## Working style / preferences
- **Goal:** make the game HARDER. Reject any design that lets the player come out ahead
  (e.g. a death penalty that grants points). Prefer data-driven where it lowers runtime load.
- **Repos:** each mod is its OWN public GitHub repo — `Xidorian/PunishingDeath`,
  `Xidorian/PredatorStealth` — and stands completely alone (no dependency on the hub). They're
  "sub" projects of the `Palworld-Mods` umbrella only conceptually. THIS folder is the shared hub
  (`Xidorian/Palworld-Mods`): the knowledge base, PROGRESS, PalClassification.csv, and art only —
  no mod source lives here. Because each mod is a separate repo, on disk they sit as sibling folders
  next to the hub (`..\PunishingDeath\`, `..\PredatorStealth\`), not nested inside it.
- **Git:** I commit locally as we go after each meaningful change and tell the user; the user runs
  `git push` (in whichever repo the change is in).
- **Deploy loop:** edit `Scripts\main.lua` in that mod's repo folder → copy into
  `Palworld\Mods\NativeMods\UE4SS\Mods\<Mod>\Scripts\` → user focuses the game to auto-reload.
  Read `UE4SS.log` for our tagged `[..]` output. Ctrl+R hot-reload is broken here — use the
  file-watcher; turn it OFF and do a game restart when finalizing (clears orphaned scan loops).
- I read the UE4SS log for the user — they just press keys in-game and say "done".

## Changelog & release standard
- **Log every mod change.** Whenever we change a mod, record it in that mod's own
  `CHANGELOG.md` (in its repo folder) as part of the same change — not later. Add entries
  under an `## [Unreleased]` heading (create it if missing) using the existing style: bold
  section labels (`**Added**`, `**Changed**`, `**Fixed**`, `**Removed**`) with dashed bullets.
  A mod with no changelog yet (e.g. PredatorStealth) gets one created on its first logged change.
- **Gate packaging on the changelog.** Before building a release zip, review the diff since the
  last version and confirm `CHANGELOG.md` reflects **all** of it — nothing shipped is missing.
  Then rename `[Unreleased]` to the new version number and package.
- **Gate mod-site upload on the changelog.** Do not update the mod on mod sites (Nexus / mod.io /
  Steam) until the changelog is complete and versioned. The site's posted changes should match
  `CHANGELOG.md` for that version. Remember both zip variants ship (regular + `-Steam-`).
