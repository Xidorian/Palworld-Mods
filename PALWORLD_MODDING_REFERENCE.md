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

**File I/O for player-editable config WORKS** (confirmed in this Steam-Workshop build): standard
Lua `io.open` is available, and `debug.getinfo(1,"S").source` returns this script's real path (with
an `@` prefix) so you can derive the mod dir and read a sibling file (e.g. `PreyList.txt` next to
`Scripts/`). Guard everything in `pcall` and fall back to built-in defaults. The UE4SS **ImGui GUI
is DISABLED** here (`GuiConsoleEnabled = 0`), so an in-game menu isn't viable — a config file is the
shippable way to let players customize.


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

### Ranged-vs-melee pal classification — NOT cleanly derivable (skip it)
Tried to auto-build a "ranged pals" list from game data to give ranged pals a bigger hide gate:
- **Skill tables:** `DT_WazaDataTable` (per-skill; key columns `WazaType`, `Category`, `MinRange`,
  `MaxRange`) and `DT_WazaMasterLevel` (`PalId`, `WazaID`, `Level` — which skills a pal learns at
  each level; **Level 1 = its default**). **Join key: `WazaData.WazaType == WazaMasterLevel.WazaID`**
  (no enum layer). `PalId` is an FName → use `:ToString()`. `PalId` lowercased == our species key.
- **Result: no clean ranged signal.** `Category==1` on 637/712 pals (it's "attack", not "shot").
  Level-1 `MaxRange` is 3000–5000 on 550+ pals **including melee-behaving ones** (Anubis 4000,
  Alpaca 4000, CuteFox 4000, KendoFrog 5000) — the data encodes skill *capability/AI-use-range*, not
  whether the pal keeps its distance. So "long-range-capable" is the norm (~78%), not a minority.
- **Also redundant:** hide-to-escape already requires NO line-of-sight, and a pal without LOS can't
  shoot you — so "can't shake it while it's firing" is enforced by the LOS check regardless of range.
- **Decision: one hide-distance gate for all pals** (no ranged/melee split, no RangedList.txt).

### Native-detection offload — INVESTIGATED & REJECTED (keep the runtime poll)  ⚠️ SUPERSEDED 2026-07-30 — see §3.9 (poll MEASURED at 80–150 ms = the stutter; offload reopened)
Tested whether enlarging the game's own sight could replace our per-scan acquire poll:
- **`ViewingDistance` in `DT_PalMonsterParameter` IS runtime-writable** (`rowData.ViewingDistance = 3000`
  on all 753 rows, readback confirmed; runtime-only, reverts on restart). So the data lever works.
- **But offloading detection to the engine is a dead end for THIS mod:** even if pals detect at range,
  **`AIResponse` governs the reaction** — most are `Escape`/`NotInterested`, so they'd flee/ignore, not
  attack. Making them hostile means overriding `AIResponse → Warlike` for everything = the static,
  all-or-nothing pak-mod approach, which **loses our live control** (per-pal range, level scaling,
  crouch cone, hide-to-escape, prey list). Our runtime poll is what buys that control and it's cheap
  (one `FindAllOf` + gated raycasts every 1.5s). **Decision: keep polling; tune `scan_ms`.**
- Side note: on `BP_FunnelCharacter_DreamDemon_C` (a special "funnel" pal), `PerceptionComponent` was
  **non-null** — contradicts the "perception null on wild pals" note above; may be funnel-pal-specific,
  don't over-trust either way. `BlockDetectionParams`/`bOverwriteBlockDetectionParams` also live on the ctrl.

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

### 3.9 UPDATE 2026-07-30 — poll cost MEASURED (it's the stutter); hooks viable; PalSchema pivot

Session driven by a mod-page report: hard traversal stutter on 3440×1440@120, gone when the mod is
disabled. What we learned (supersedes the "keep polling, it's cheap" call above):

- **The poll is NOT cheap — it IS the stutter.** `os.clock()` timing around `scan()` measured
  **63–150 ms per scan for only 12–27 pals** (~3–5 ms/pal), all synchronous on the game thread →
  dropped frames. Cost is dominated by per-pal `LineOfSightTo` raycasts + `GetLevel`/reflection,
  **not** `FindAllOf`. On a weak laptop it's brutal; a 60 Hz/1080p player may not feel it (~8 ms
  budget at 120 Hz is why the reporter did).
- **Stutter meter (reusable technique):** a 2nd `LoopAsync(50 ms)` sampler on the game thread; when a
  frame stalls the queued sample fires late and the wall-clock gap spikes. `os.clock()` is WALL-time
  on Windows. Report worst-gap + count of gaps >100 ms per ~5 s window = machine-independent hitch
  number. Measured worst 124–255 ms, several hitches / 5 s = real stutter.
- **Roster redesign did NOT fix it.** Swapped per-tick `FindAllOf` for a cached roster (seed once +
  `NotifyOnNewObject("/Script/Pal.PalCharacter")`, cache class/prey, resolve controller lazily).
  Scans stayed 80–140 ms → confirms per-pal raycast/reflection is the cost, not enumeration. Polling
  can't be made cheap enough.
- **Wild-AI decision hooks ARE hookable and fire.** `BP_MonsterAIController_Wild_C` (Blueprint) →
  `ForceEscaleStartForOutside` (Pocketpair's "Escale" = the FLEE decision), `ForceBattleStartForOutside`
  (FIGHT), and native `PalAIController:AddTargetPlayer_ForEnemy` (marks player). Full path e.g.
  `/Game/Pal/Blueprint/Controller/Monster/BP_MonsterAIController_Wild.BP_MonsterAIController_Wild_C:ForceEscaleStartForOutside`.
  All registered + fired. **Census matched the data exactly:** `NightFox` (`Escape_to_Battle`) → FLEE;
  `NegativeKoala`/`CloverFairy` (`Warlike`) → FIGHT/ADD_TARGET. So the hook to convert flee→fight is
  `ForceEscaleStartForOutside`.
- **Hooks + streaming = crash risk.** During a fast-travel got `EXCEPTION_ACCESS_VIOLATION` (near-null
  deref) — a hook/roster callback touching an object mid-construction/destruction; `pcall` can't catch
  it (native fault, see §1). This install already had 5 crash dumps predating any hooks and fast-travel
  is a known Palworld crasher, so not conclusively ours — but keep per-object reflection OFF the
  streaming path.
- **PalSchema CAN patch this (v0.6.1, installed).** Edits DataTable rows AND Blueprint defaults via
  conflict-free JSON. Patches live in `…/UE4SS/Mods/PalSchema/mods/<ModName>/raw/<file>.json` (subfolder
  `raw` = DataTable edits; also `blueprints`, `pals`, …; mod-folder name = display name, no per-mod
  config). `PalSchema/config/config.json` has `enableDebugLogging`. Format:
  ```json
  { "DT_PalMonsterParameter": { "Sheepball": { "AIResponse": "Warlike" } } }
  ```
  Only targeted rows/fields change. Row keys = exact `Pal` column in `PalClassification.csv`. This is a
  **static** patch that PERSISTS across restart (unlike the runtime `rowData.X=Y` writes above, which revert).
- **Reload reality (amends §0):** live install had `EnableAutoReloadingLuaMods = 0` (not the `1` §0
  assumes) so focus-reload did nothing, AND **Ctrl+R hot-reload tore every mod down to vanilla and did
  NOT restart them** — the **C++** `PalSchema` mod can't hot-reload, breaking the chain. **Only a full
  game restart reliably loads changes here.** Restore `EnableAutoReloadingLuaMods=1` for the focus loop,
  or accept full restarts.

**DIRECTION — HYBRID, not pure-data.** Pure `AIResponse→Warlike` for all non-prey kills the stutter
(native AI + native detection, zero poll) and even closes the "neutral pals ignore you" gap — BUT loses
the STEALTH half (crouch cone, hide-to-escape, level scaling) that the poll provided and that names the
mod. Reconciliation: **data patch for AGGRESSION** (no acquire poll → no stutter) **+ a thin runtime
layer for STEALTH over only the pals actively hunting you** (`tp>0` = a handful → cheap), using the
SOLVED de-aggro (`HateMap:Empty()` + `TargetPlayers:Empty()`) for crouch-suppress + hide-to-escape.
PoC in progress: PalSchema flip `Sheepball→Warlike` to confirm the static patch path applies here.

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

---

## 8. Pals, bases, death/permadeath levers  (desk research — VERIFY in-game before trusting)

Gathered while scoping "The Long Walk Home" (see `PLANNED_MODS.md`). These are from community
docs/mods, **not yet confirmed by our own reflection** — names/signatures drift per patch.

- **Summon/deploy a party pal at a chosen spot:** party pals ("otomo") are stored on
  `BP_OtomoPalHolderComponent`; **`ActivateOtomo(int32 SlotID, FTransform StartTransform, bool& IsSuccess)`**
  spawns the *actual owned* pal (identity/level/IVs preserved). `StartTransform` = where it appears.
  Driven by community "Multi Pals Deploy" mods.
- **Base location:** `PalBaseCampModel` is hookable at runtime (base-range mods scale its `AreaRange`);
  read base position from it for distance checks.
- **The engine returns pals home by TELEPORT, not navigation.** Base workers out of range → teleport
  to Palbox → run to task; summoned pals auto-recall/teleport on death/fast-travel. **Lesson: reliable
  long-distance pal pathfinding to an arbitrary point does not exist — don't design on it.**
- **Death Penalty world setting:** `All` drops items+equipment+pals into a recoverable **Death Chest**.
- **Permadeath = two SEPARATE flags:** `bHardcore` (perma **player** death), `bPalLost` (perma **pal**
  death — 0 HP = gone, no revive, native warning). `bCharacterRecreateInHardcore` = new char after
  hardcore player death. **Prefer routing any "pal is gone" outcome through the native `bPalLost`
  path over hand-deleting a pal object** (deletion is the save-corruption risk class). TODO: confirm
  these are runtime-reachable (likely `PalGameSetting` / world-options struct) and whether the removal
  can target a single pal.
- **Broker recovery:** Pal Merchants sell back lost pals (Black Marketeer = rare ones) — but likely
  **species-level** re-buy, not your exact individual.
