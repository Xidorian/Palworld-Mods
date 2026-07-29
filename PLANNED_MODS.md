# Palworld Mods — Planned & Ideas (Xidorian Studios)

Future mods and ideas not yet started. Once a mod moves into active development it
graduates to [`PROGRESS.md`](PROGRESS.md). Each mod ships as its OWN standalone repo
(keep mods modular — suite/bundling comes later, never fold features together prematurely).

_Legend: ⬜ not started · 💡 idea · 🅿️ parked_

---

## ⬜ Mod 3 — Discard keybind
Drop/discard the held item with a keypress.

## ⬜ Mod 4 — Auto-run persistence
Keep auto-run through menu / map / alt-tab.

Also keep **building and crafting** running while in the map, inventory, and similar
menus — not just alt-tab. The **Background Crafting** mod (Steam Workshop, already
installed) only handles the alt-tab case; this would extend that persistence to the
other UI states so production doesn't pause when you open menus.

**Crouch cancels auto-run** — auto-run stops the moment you crouch. It should survive
crouching too (crouch-walk while auto-running). Relevant because Predator & Stealth
wants you crouched for stealth, so losing auto-run on crouch is a real friction.

## 💡 Mod 5 — Sharper Predators
Make enemy pals' attacks harder to dodge = effectively "better aim". Fresh
investigation (aiming subsystem untouched). Candidate levers to probe:
- projectile behavior — speed / homing / spread (likely `DT_WazaDataTable` extra
  columns, or the projectile blueprints);
- AI target-leading / aim-error on the controller;
- a difficulty/accuracy multiplier in `PalGameSetting` if one exists.

Ship as its OWN mod (keep mods modular), not folded into Predator & Stealth.

## 💡 Difficulty Master Suite
A single "all my difficulty mods in one" bundle for players who want everything.
Build the individual mods standalone first; the suite just packages them. Do NOT
lump features together prematurely — modular first, suite later.

---

## 🅿️ Parked — "The Long Walk Home" (pals ejected on death, may not make it back)
Big standalone project. **Concept:** on player death, your party pals aren't lost to the inert
death-drop — they burst out as *living* pals where you fell and must make their way home; some
don't make it. Fits the "make it HARDER" goal (replaces free chest-retrieval with real stakes).

**Discovery — 2026-07-29 (desk research only, NOT yet verified in-game):**

Feasibility hinges on native levers, and most of the mechanic maps onto them cleanly. The one part
that genuinely can't be done is the literal walk.

- **No permadeath by default.** A defeated pal is *Incapacitated*, not dead — recovers via Palbox
  (10 min), Revival Potion, healing pal, or a base worker hauling it to a bed. So "death" here means
  intercepting the **player-death → drop-pals** flow, not a pal dying.
- **The "soul" drop = Death Penalty world setting.** `All` mode drops items + equipment + pals into
  a recoverable **Death Chest** marker. Separate flag `bPalLost=true` makes pal loss permanent.
- **Eject as a LIVING pal = solved lever.** Party pals ("otomo") live on `BP_OtomoPalHolderComponent`;
  `ActivateOtomo(int32 SlotID, FTransform StartTransform, bool& IsSuccess)` spawns the *actual* owned
  pal (identity/level/IVs preserved) — and `StartTransform` lets us choose the spawn spot (the death
  location). Community mods (Multi Pals Deploy) already drive this path.
- **Home location = obtainable.** `PalBaseCampModel` is hookable at runtime; base position → distance.
- **🔴 The blocker: no reliable long-distance navigation.** The engine itself doesn't walk pals home —
  it **teleports** them (base workers out of range → teleport to Palbox → run to task; summoned pals
  auto-recall/teleport on death/fast-travel). Two independent confirmations (this + Hungry Pal Rescuer
  teleporting strays) that cross-map pathfinding can't be trusted. **So the "journey" must be an
  abstraction (timer + visible milling), not a watch-them-hike simulation.**
- **🔴 No organic peril.** A wandering ex-pet has nothing to kill it (wild pals don't fight each other;
  Predator & Stealth aggros pals at the *player*, not at a loose pal). "May die" must be scripted odds.
- **Hardcore gives a SAFE death outcome — tap it.** `bHardcore` (perma player death) and `bPalLost`
  (perma pal death) are **separate** flags. Routing journey-failures through the native `bPalLost`
  removal path is save-safe (vs. hand-deleting a pal object = save-corruption risk) and shows the
  native warning. `bCharacterRecreateInHardcore` = allow new character after a hardcore player death.
- **Broker recovery = free economy layer.** Pal Merchants sell back lost pals (Black Marketeer = rare
  ones). Gives a penalty dial — though buyback is likely **species-level**, not your exact individual.

**Design that flies (native levers only):**
1. Player dies → `ActivateOtomo` each party pal as a living actor at the death `FTransform`.
2. Survival window: pals visibly mill about; odds from death→base distance (`PalBaseCampModel`) + pal level.
3. Outcome — **survive:** returned to Palbox via the engine's own out-of-range teleport-home.
   **fail:** removed via native `bPalLost` path. Penalty tiers: *soft* = lost but re-buyable at a
   Pal Merchant; *hard* = `bPalLost` permadeath, gone for good.
4. The literal cross-map walk is the only abstracted part — everything else is the game doing its thing.

**Open questions — need in-game reflection (deferred):**
- Are `bPalLost` / the pal-lost **removal routine** runtime-reachable (likely `PalGameSetting` or the
  world-options struct), and can we invoke it on **one specific pal** vs. flipping a global flag?
- **Hook timing:** does the game force-recall the party on death *before* we can `ActivateOtomo`?
  Likely need to hook the death/drop event and inject ejection at the right moment.
- Does `ActivateOtomo` behave with a **dead/respawning owner**, or assume a live nearby player?

Reusable engine levers from this pass are logged in `PALWORLD_MODDING_REFERENCE.md` §8.
