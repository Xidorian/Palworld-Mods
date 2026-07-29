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

### The HATE SYSTEM (aggro tables)  — DE-AGGRO SOLVED ✅
`ctrl:GetHateSystem()` → **`PalHate`**. Hate lives here, but the *active target* lives on the
controller (see below) — you must clear BOTH to de-aggro.
- PalHate functions: `ChangeHate`, `FindMostHateTarget`, `ForceHateUp_ForActiveAndAttackOtomoPal`,
  `DamageEvent`, `AttackSuccessEvent`, `SelfDeathEvent` (from reflection — **no read/clear getter**).
- PalHate properties: **`HateMap`** (a real `TMap<FWeakObjectPtr target, FStruct hateInfo>`),
  **`HateTimerHandle`**.

**✅ CONFIRMED DE-AGGRO (tested on Deer/Alpaca/Boar, no crash):** clear both containers on the
game thread —
```lua
ctrl:GetHateSystem().HateMap:Empty()   -- wipes hate (topTarget -> none, mapCount 1->0)
ctrl.TargetPlayers:Empty()             -- wipes the ACTIVE target (tp 1->0) -- THIS is what
                                       --   actually keeps them attacking; hate-only is NOT enough
```
The pal drops the aggro indicator and returns to wandering. Do it inside `ExecuteInGameThread`.
`TargetPlayers` is a plain `TArray` on the controller; UE4SS `TArray`/`TMap` only expose `:Empty()`
(no per-key/index remove), and both `:Empty()` calls are SAFE via UE4SS's typed-container path
(the old "TArray:Empty crash" was a different bad context, not this).

**Reading the HateMap (diagnostics):** `hm:ForEach(function(k,v) ... end)` — `k` is an
`FWeakObjectPtr` (use `k:Get()` to resolve the actor; `classNameOf(k)` fails on it), `v` is a
`UScriptStruct` (no clean field read from Lua — `type()` only says "UScriptStruct"). `hm:Find(key)`
/ `hm:Contains(key)` do NOT work with a raw player object (key type is weak-ptr → Set-pusher
mismatch; `Find` throws "Map key not found"). Return `true` from the `ForEach` cb to stop early.

**Dead ends (all tested, don't revisit):**
- **`ChangeHate(player, 0)` = no-op** (map/target unchanged). **`ChangeHate(player, -N)` = backfire**
  (registers the player as a target on passive pals). `ChangeHate` is an aggro lever, not de-aggro.
- **`SetActiveAI(false)` = FREEZE, not de-aggro** — pal stops attacking but keeps `tp=1`, keeps the
  hate map, keeps the aggro indicator, frozen in place even when hit. Reversible via `SetActiveAI(true)`.
  Not a substitute for the Empty+Empty recipe.
- Controller has NO purpose-built end-combat/clear-target function (full `PalAIController` dump has
  only aggro *adders*: `AddTargetNPC`, `AddTargetPlayer_ForEnemy`, `CopyTargetFromOtherAI`,
  `ForceHateUp…`). `K2_ClearFocus` clears look-focus only, not the target.
- **Native de-aggro is LEASH-based**, not stealth (`On Character Out Of Leash Range`); a boss chased
  64m with LOS broken ~20s never gave up. So "hide to escape" must be built on our Empty+Empty call.

`FindMostHateTarget` returns the top map entry even at ~0 hate, so it's a weak "is it aggro'd" signal;
`ctrl.TargetPlayers:GetArrayNum()` (`tp`) is the reliable "actively hunting" signal.

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
- Crack de-aggro (see §7) → unlocks de-aggro button, **hide-to-escape** timer, **prey-sneak**.
- Load measurement on a clean single instance.
- Tune ranges / cone / rear% / crouch% / level scaling.
- Finalize Predator & Stealth: rename `AIProbe` → `PredatorStealth`, strip dev keys/monitor,
  config header, auto-reload OFF, package + publish. Then commit + push.

---

## 7. Hate system deep-dive & de-aggro  (SOLVED — see the recipe in §3)

**DE-AGGRO IS SOLVED.** `HateMap:Empty()` + `TargetPlayers:Empty()` on the game thread (full recipe
and dead-ends in §3, "The HATE SYSTEM"). The rest of this section is the original investigation
notes + the hide-to-escape design that de-aggro now unlocks (NEXT: build it).



**How aggro works (our model):**
- Each wild pal's AI controller has a hate system: `ctrl:GetHateSystem()` → `PalHate`.
- **`HateMap`** = a map of `Actor -> hateValue`. The pal attacks whoever has the **highest** hate.
- **`FindMostHateTarget()`** = returns that highest-hate actor (its current target). Caveat: it
  returns the top entry even at ~0 hate, so it's a weak "is it actually aggro'd" signal alone.
- Hate is ADDED by: `DamageEvent` (you hit it — or it hits you), `AttackSuccessEvent`,
  `ForceHateUp_...`, our `ForceBattleStartToTarget`, and `ChangeHate`.
- **`HateTimerHandle`** = a timer, almost certainly hate decay / target re-evaluation. This is
  probably the real mechanism behind natural "lost interest."
- **`TargetPlayers`** (on the controller) is a SEPARATE list from `HateMap` (enemy-players list),
  not the same as who's being attacked.

**Confirmed dead end:** `ChangeHate(player, -N)` ADDS/registers the player (passive pals went
`none -> player`); it does not reduce hate. It's an aggro lever, not a de-aggro one.

**De-aggro test plan — do it CONTROLLED:** aggro exactly ONE pal, then step OUT of melee so
`DamageEvent` stops re-adding hate, and read values before/after each attempt:
1. **Read the actual number.** Add a diagnostic that reads the player's value in `HateMap`
   (figure out TMap access in UE4SS Lua — try `hs.HateMap:Find(player)`, or iterate the map).
   Knowing the real value is prerequisite to everything else.
2. **Try `ChangeHate(player, 0)`** (user's idea) — maybe 0 neutralizes where negative wrapped to a
   huge unsigned add.
3. **Change BOTH hate and the target** (user's idea) — a locked current-target may persist even at
   0 hate; look for a "set/clear target" on the controller or `PalHate` and clear it too.
4. **Shorten `HateTimerHandle`** (user's idea) — set/force the decay timer to ~1s so hate expires
   fast → natural de-aggro without fighting the map.
5. **Remove the player key from `HateMap`** directly (TMap remove) — last resort, container op is
   crash-risky, test carefully.

**Hide-to-escape design (the goal these unlock):**
- In the scan, for any pal whose top hate target is the player, track "time since it last had LOS
  to you" (`LineOfSightTo`).
- When you break LOS (duck behind a structure), start a **"searching" countdown (~3s)**.
- If LOS stays broken past the countdown → trigger de-aggro (whichever method above works).
- Result: real "break line of sight to lose them" stealth. Same de-aggro call also powers the
  manual F7 button and **prey-sneak** (suppress a fleeing prey's target while you're crouched/hidden).

**Also:** re-test F7 de-aggro in a controlled scenario once a working method exists (the earlier
test was chaotic — 12 pals, active melee, night). Current F7 = safe pause-only.
