# Palworld Mods — Progress (Xidorian Studios)

_Legend: ✅ done · 🔨 in progress · ⬜ not started_

---

## 🎮 Mod 1 — Punishing Death  ✅ SHIPPED
Lose current-level EXP progress on death; level never drops (exploit-proof).
- [x] Reverse-engineer the EXP/level system (SaveParameter.Exp/Level, DT_PalExpTable curve)
- [x] Death detection (IsDead poll)
- [x] Penalty logic (drain in-level progress)
- [x] Investigate & reject de-leveling (point-farm exploit) — see DESIGN_NOTES.md
- [x] Harden + lean build (test keys off, auto-reload off)
- [x] README + DESIGN_NOTES
- [x] Release zip + mod-page copy
- [x] Cover art + studio logo
- [x] Publish (Nexus / CurseForge / Steam Workshop)

## 🐾 Mod 2 — Predator & Stealth  🔨 IN PROGRESS  (current focus)
Most pals hunt you; small prey stay passive; stealth actually works.

**Discovery / foundation**
- [x] Map the pal AI system (no UE AIPerception — custom detection)
- [x] Confirm levers: `ForceBattleStartToTarget` (aggro), `LineOfSightTo` (occlusion), `bIsCrouched`
- [x] Study existing aggro mods (data-driven, all-or-nothing, no stealth)
- [x] Find `DT_PalMonsterParameter` (Predator flag, ViewingDistance, AIResponse, HP)
- [x] Export `PalClassification.csv` (reusable data archive)

**Core system**
- [x] Data-driven classification: aggressive-except-small-docile-prey (753 species) — WORKING
- [x] Force-aggro predators in range with line-of-sight
- [x] Wild-only filter (never turns your own pals hostile)
- [x] Crouch stealth (detection shrinks when crouched) — CONFIRMED in-game
- [x] Level-disparity scaling (higher-level pals notice from farther, capped)
- [x] Skip sleeping pals (sneak past sleepers)
- [x] Directional detection / vision cone (harder to notice from behind) — v6
- [x] Crash hardening (removed unsafe array op; aggro-per-scan cap; safe defaults)

**Remaining**
- [ ] Test line-of-sight hiding (break view behind rocks/walls)
- [ ] Safe de-aggro button (F7 is pause-only; need the proper end-battle function)
- [ ] Hide-to-escape timer (lose pursuers after N seconds out of sight) — needs the de-aggro fn
- [ ] Load measurement on a clean single instance (post-restart)
- [ ] Tuning pass (ranges, cone width, rear %, crouch %)
- [ ] Finalize (rename to PredatorStealth, strip test keys, config header, auto-reload off)
- [ ] Package + publish

## ⬜ Mod 3 — Discard keybind  (not started)
Drop/discard the held item with a keypress.

## ⬜ Mod 4 — Auto-run persistence  (not started)
Keep auto-run through menu / map / alt-tab.

---

## ✅ Solved by existing mods (no build needed)
- [x] Alt-tab while crafting → **Background Crafting** (Steam Workshop)
- [x] Less-forgiving fog of war → **Less Map Shroud (2 squares)** (installed)

## 🅿️ Parked
- Pals released into the world on death, walk home, may die (big standalone project)

## 🧰 Infrastructure / learnings
- [x] UE4SS setup + hot-reload dev workflow
- [x] `PalClassification.csv` data archive
- [x] Project memory files (setup paths, EXP + AI internals)
- [x] Crash debugging (access violation → unsafe TArray op)
- [ ] Put mods under version control on GitHub  ← starting now
