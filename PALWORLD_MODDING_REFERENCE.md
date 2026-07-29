# Palworld UE4SS Modding Reference

Consolidated, hard-won knowledge from building these mods. **Read this at the start of
any session** so we don't re-derive things. Version drifts each game patch — verify a
name still exists before trusting it, but the patterns and lessons hold.

---

## 0. Setup, paths, workflow

- **Game:** `C:\Program Files (x86)\Steam\steamapps\common\Palworld`
- **UE4SS (non-standard, Steam Workshop build):** `...\Palworld\Mods\NativeMods\UE4SS\`
- **Lua mods live in:** `...\UE4SS\Mods\<ModName>\Scripts\main.lua`
- **Enable a mod:** add `ModName : 1` to `...\UE4SS\Mods\mods.txt` (this build uses mods.txt,
  not just `enabled.txt`). New mods / mods.txt edits need a game restart.
- **Log:** `...\UE4SS\UE4SS.log` (overwritten each launch). We tag our prints, e.g. `[PDST]`,
  and read the log to observe results. NOTE: `[PS]` collides with PalSchema — don't reuse it.
- **Dev source** lives in this repo (`C:\Users\Xidorian\PalworldMods\<Mod>\`). Deploy = copy
  `Scripts\main.lua` into the game Mods folder.

### Hot-reload
- `UE4SS-settings.ini`: `EnableAutoReloadingLuaMods = 1` = **file-watcher auto-reload** — WORKS,
  but only fires **while the game window is focused**. This is our iterate loop: edit source →
  copy to game folder → focus game → it reloads.
- `EnableHotReloadSystem` (Ctrl+R hotkey) is **broken in this build** (a mod even reports
  `hotReloadSupported=false`); Ctrl+R can kill other mods. Don't use it.
- **Turn both OFF when a mod is finalized** — the file-watcher is a continuous cost.
- **Orphaned loops:** each auto-reload leaves the previous `LoopAsync` running. They stack up →
  lag + multiplied behavior (e.g. multiple scans re-aggroing). A **full game restart clears
  them**. Do a restart before judging load or running experiments that rely on pausing.

---

## 1. UE4SS Lua API — patterns & gotchas

**Core functions:** `StaticFindObject(path)`, `FindFirstOf(shortClass)`, `FindAllOf(shortClass)`,
`NotifyOnNewObject(class, cb)`, `RegisterHook("/Script/...:Fn", cb)`, `RegisterKeyBind(Key.F8, cb)`
(also `ModifierKey.CONTROL`), `LoopAsync(ms, cb)` (return true to stop), `ExecuteInGameThread(cb)`,
`ExecuteWithDelay(ms, cb)`.

**Reflection / discovery:** `obj:GetClass()` → `:ForEachProperty(fn)` / `:ForEachFunction(fn)`,
walk `:GetSuperStruct()` for inherited members. Names via `x:GetFName():ToString()`.

**DataTables:** `dt:ForEachRow(function(rowName, rowData) ... end)`. Read cells as `rowData.Field`.

**GOTCHAS (each one cost us time):**
- Indexing a UObject with a **non-existent property returns a non-nil "TrivialObject"** (not nil).
  Guessing names is useless — enumerate instead.
- In `ForEachRow`, **`rowName` is a plain Lua string** → use `tostring(rowName)`, NOT `rowName:ToString()`
  (calling `:ToString()` on a string errors, silently zeroing your loop).
- **FName-typed cell values need `:ToString()`** → `rowData.AIResponse:ToString()`. `tostring()` on
  them gives `"fnameuserdata: 0x..."`, not the name.
- **Do state writes on the game thread** (`ExecuteInGameThread`). `LoopAsync` runs on an async thread.
- **Never do raw container ops like `TArray:Empty()`** — it caused an `EXCEPTION_ACCESS_VIOLATION`
  crash. Use the game's own functions instead (e.g. the hate system, below).
- A native access violation is **not catchable by `pcall`** — the only defense is to not make the
  bad call (revalidate objects, avoid unsafe ops).
- `FindFirstOf("PalPlayerCharacter")` can return a stale pawn if you're mounted; prefer it for the
  player but sanity-check. Player class is `BP_Player_Female_C`.

---

## 2. Player EXP / Level system  (Punishing Death)

- **Chain:** player (`PalPlayerCharacter`) → `.CharacterParameterComponent`
  (`PalCharacterParameterComponent`) → `.IndividualParameter` (`PalIndividualCharacterParameter`)
  → `.SaveParameter` (a `UScriptStruct`; `SaveParameterMirror` is a stale copy — don't use it).
- **`SaveParameter.Exp` / `.Level`** are plain ints — **readable AND writable** (writes persist &
  save; UI updates when you re-open the character sheet). Getters `ip:GetExp()` / `ip:GetLevel()`
  read the stored value; `GetLevel` does NOT re-derive from exp.
- **Level curve:** `/Game/Pal/DataTable/Exp/DT_PalExpTable.DT_PalExpTable`. `rowData.TotalEXP` =
  cumulative exp to reach that level (`NextEXP` = per-level). Anchor: L45 = 1,421,830.
- **Status points:** `ip:GetUnusedStatusPoint()`, `SaveParameter.UnusedStatusPoint`,
  `ip:DecrementUnusedStatusPoint()`.
- **Tech points:** `player.PlayerState.TechnologyData` (`PalTechnologyData`): `.TechnologyPoint`,
  `.bossTechnologyPoint`, `.UnlockedTechnologyNameArray`. (Resolution is sometimes flaky.)
- **KEY LESSON — do NOT de-level by writing `SaveParameter.Level`.** It bypasses the level-change
  event, which (1) re-grants status/tech points on the next natural level-up = a farm EXPLOIT, and
  (2) desyncs the HUD level from the character sheet. Clawing points back races the engine and is
  fragile. Correct design: change **EXP only**, keep Level fixed.
- **Death detection:** poll `comp:IsDead()` via `LoopAsync` with a `wasDead` latch + `seenAlive`
  guard; apply on the game thread. Do NOT hook `ClientRestart` (fires multiple times per death,
  and not on fast travel).

---

## 3. Pal AI / aggression / detection  (Predator & Stealth)

- **Wild pal:** `PalCharacter` subclass, class name `BP_<Species>_C`. **Controller** via
  `pal.Controller` = `BP_MonsterAIController_Wild_C` (base `PalAIController`). Filter wild pals by
  controller name containing `"Wild"` (so you never turn the player's own pals hostile).
- **No UE AIPerception** — `PerceptionComponent` is null on wild pals; detection is custom game
  logic. (`BlockDetectionParams` is pathfinding, not sight.)

### Confirmed runtime levers (tested)
- `ctrl:ForceBattleStartToTarget(player)` — makes a wild pal **attack** the player (works on any,
  even passive Chikipi). `AddTargetPlayer_ForEnemy(player)` only adds to `TargetPlayers` (0→1) but
  does NOT engage.
- `ctrl:LineOfSightTo(player, nil, false)` — real occlusion check (our see-through-walls fix).
- `player.bIsCrouched` — crouch state.
- `pal:GetActorForwardVector()` — facing (used for the vision-cone / rear-stealth check).
- `pal.CharacterParameterComponent:GetLevel()` — pal level (for level-disparity scaling).

### The data: `DT_PalMonsterParameter`
`/Game/Pal/DataTable/Character/DT_PalMonsterParameter.DT_PalMonsterParameter` — 90 columns/pal.
Exported to **`PalClassification.csv`** in this repo. Useful columns:
- **`Predator`** (bool) — game's own predator flag. **`Edible`**, **`Nocturnal`** (bool).
- **`Hp`** (int) — good size/threat proxy (small prey ~60–100, Mammorest 150+).
- **`AIResponse`** (FName) — `Escape`, `Escape_to_Battle`, `NotInterested`, `Friendly`, `Warlike`,
  `Kill_All`, `Boss`. Most pals default to Escape/NotInterested = why so few attack.
- `AISightResponse` (FName) — **all `None` / unused**.
- **`ViewingDistance`** (int) — **uniformly 25 for every pal** — the game's native detection is a
  flat, tiny constant with no per-pal or per-level variation. So we run our own detection instead.
- `ViewingAngle`, `HearingRate`, `Size`, `GenusCategory`, `Rarity`, `IsBoss`/`IsTowerBoss`/`IsRaidBoss`.

**Our classification:** prey = `Hp <= ~110` AND `AIResponse` not `Warlike`/`Kill_All`; everything
else is aggressive. Match a spawned pal to its row via species key = lower(className minus `BP_`/`_C`).

### Aggression presets (blueprints)
`/Game/Pal/Blueprint/Controller/AIResponsePreset/` — `escape`, `NotInterested`, `Friendly`,
`Warlike`, `Kill_All`, `Boss`, `VillageNPC`. Plus `AISightResponsePreset/` — `Citizen`, `Police`,
`Villain`. Pak mods (e.g. "Pals Attack Player", Nexus 2102) make everything hostile by overriding
these presets — static, all-or-nothing, **no stealth**. That's why we build our own at runtime.

### The HATE SYSTEM (aggro tables)  ← current investigation
`ctrl:GetHateSystem()` → **`PalHate`**. This is where aggro actually lives.
- Functions: **`ChangeHate`** (add/subtract hate on a target — the de-aggro lever),
  `FindMostHateTarget`, `ForceHateUp_ForActiveAndAttackOtomoPal`, `DamageEvent`,
  `AttackSuccessEvent`, `SelfDeathEvent`.
- Properties: **`HateMap`** (target → hate value), **`HateTimerHandle`** (hate has a timer → likely
  decays over time).
- **De-aggro plan:** `hs:ChangeHate(player, -big)` to wipe your hate. Safer than array-poking.
  NOTE: `TargetPlayers` (on the controller) and the `HateMap` are not the same list — a pal you
  provoked by attacking may hold hate in `HateMap` without being in `TargetPlayers`.
- **Native de-aggro is LEASH-based**, not stealth: functions like `On Character Out Of Leash Range`
  fire when the pal is dragged too far from its home/spawn. A boss chased to 64m with line-of-sight
  broken for ~20s never gave up. So "hide to escape" must be built on `ChangeHate`, not the leash.

---

## 4. Mods in this repo
- **PunishingDeath** (released): lose in-level EXP progress on death; level never changes.
- **AIProbe** = **Predator & Stealth** (WIP; folder renamed on release): predators hunt you, small
  prey stay passive; crouch + line-of-sight + vision-cone + level-scaled detection. Dev keys:
  **F7** de-aggro (WIP, via ChangeHate), **F8** enable/disable, **F9** nearest-pal info readout,
  **F10** dump controller/hate-system functions. `monitor` logs any pal targeting the player.

## 5. Existing mods we use instead of building
- **Background Crafting** (Steam Workshop) — keep crafting while tabbed out.
- **Less Map Shroud (2 squares)** — smaller map-reveal radius (`worldmapUIMaskClearSize`, default ~50).

## 6. Open threads
- Confirm `ChangeHate` signature/effect → build the safe de-aggro button, **hide-to-escape** timer,
  and **prey-sneak** suppression (all from one function).
- Load measurement on a clean single instance.
- Tune ranges / cone / rear% / crouch% / level scaling.
