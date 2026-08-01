# Palworld Mods — Planned & Ideas (Xidorian Studios)

Future mods and ideas not yet started. Once a mod moves into active development it
graduates to [`PROGRESS.md`](PROGRESS.md). Each mod ships as its OWN standalone repo
(keep mods modular — suite/bundling comes later, never fold features together prematurely).

_Legend: ⬜ not started · 💡 idea · 🅿️ parked_

---

## ⬜ Mod 3 — Discard keybind
Drop/discard the held item with a keypress.

## ⬜ Mod 4 — Automation QoL
A bundle of "keep things running" quality-of-life fixes so background actions don't stop
when the game normally interrupts them:

- **Auto-run persistence** — keep auto-run through menu / map / alt-tab, **taking damage**
  (player currently stops auto-running whenever hit — noticed in playtest), and through
  **crouch** (auto-run currently stops the moment you crouch; it should survive crouch-walk).
  Crouch matters because Predator & Stealth wants you crouched for stealth, so losing
  auto-run on crouch is real friction.
- **Crouch state persistence through actions** — rolling while crouched should leave you
  crouched, not stand you up. Actions shouldn't silently break the stealth stance (same
  spirit as auto-run-through-crouch above). Likely a `bIsCrouched` re-assert after the
  roll/dodge action completes.
- **Building & crafting persistence** — keep building and crafting running while in the map,
  inventory, and similar menus, not just alt-tab. The **Background Crafting** mod (Steam
  Workshop, already installed) only handles the alt-tab case; this extends that persistence
  to the other UI states so production doesn't pause when you open menus.

## 💡 Mod 5 — Sharper Predators
Make enemy pals' attacks harder to dodge = effectively "better aim". Fresh
investigation (aiming subsystem untouched). Candidate levers to probe:
- projectile behavior — speed / homing / spread (likely `DT_WazaDataTable` extra
  columns, or the projectile blueprints);
- AI target-leading / aim-error on the controller;
- a difficulty/accuracy multiplier in `PalGameSetting` if one exists.

Ship as its OWN mod (keep mods modular), not folded into Predator & Stealth.

## ⬜ Mod 6 — Always-on reticle  (QoL)
Show the aiming reticle/crosshair at all times, not only while aiming down sights —
currently you can't see where you're pointing unless you ADS. Options to probe: unhide
the game's native reticle widget (HUD/UMG) so it stays visible, or draw a lightweight
crosshair overlay ourselves. QoL, its own standalone mod.

## 💡 Mod 7 — Mod Options Framework (shared in-game settings menu)
A reusable **in-game options screen for mods** — ours *and* other modders'. Instead of every
mod shipping its own config file or keybind toggles, this is a standalone framework mod that
provides a settings UI any mod can register its options into (toggles, sliders, dropdowns,
keybinds), with values persisted and exposed back to the registering mod at runtime. Think of
it as the shared "settings panel" layer the whole suite (and the wider community) can build on.

Directly supersedes the [Config UX open question](#-config-ux--in-game-options-menu-items-open-question-cross-cutting)
below — this IS the "extend the options menu" investigation, promoted to its own mod. Scope
work still gated on that feasibility question:
- **Rendering path.** This build has UE4SS ImGui disabled (`GuiConsoleEnabled = 0`), so an ImGui
  overlay isn't available as-is. Options to probe: hook Palworld's native settings UMG and inject
  our own option rows, build a self-drawn UMG widget, or lean on an existing UI framework mod
  (PalSchema / UMG-adding community mods) rather than reinventing it.
- **Registration API.** A clean, documented way for another mod to declare its settings (name,
  type, default, range/choices, on-change callback) without touching this mod's internals —
  keep the coupling one-directional (mods depend on the framework, never the reverse).
- **Persistence.** Where values live (per-mod config files it manages, or a single shared store)
  and how a mod reads its current values each run.
- **Standalone + public.** Its own repo like every other mod; if it's genuinely reusable it's a
  strong candidate to document publicly for other modders. Modular first — do not fold any
  specific mod's settings into it; it only provides the shell.

## 💡 Difficulty Master Suite
A single "all my difficulty mods in one" bundle for players who want everything.
Build the individual mods standalone first; the suite just packages them. Do NOT
lump features together prematurely — modular first, suite later.

## 🔧 Config UX — in-game options-menu items?  (open question, cross-cutting)
**→ Promoted to [Mod 7 — Mod Options Framework](#-mod-7--mod-options-framework-shared-in-game-settings-menu).**
This section stays as the feasibility notes that mod inherits.

User wants mod settings exposed as **in-game option-menu items**, not config-file edits.
CONSTRAINT: this build has the UE4SS ImGui GUI disabled (`GuiConsoleEnabled = 0`), so an
ImGui overlay isn't available; hooking Palworld's native settings UMG is a big lift.
Pragmatic path today = editable config file (like `PreyList.txt`) and/or keybind toggles.
TODO: investigate whether the game's options menu can be extended (some UMG/PalSchema mods
add UI) before committing to config-file-only. Applies to every mod's settings.

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
- **Home location = obtainable (and there may be several).** `PalBaseCampModel` is hookable at runtime;
  base position → distance. The player can own **multiple bases**, so "home" = the **nearest** base to the
  death spot: enumerate all `PalBaseCampModel`s, pick min distance. That nearest-base distance is the input
  to the survival odds.
- **🟡 The blocker, refined: the navmesh is probably NOT the wall — the engine's teleport *choice* is.**
  Earlier framing said cross-map pathfinding "can't be trusted." Live observation revises this: **field
  Mammorests roam large starting-island territories on foot** — genuine long-range navmesh traversal, not
  teleport. So the mesh clearly carries a big creature across a large connected area. What actually blocks us
  is that **owned/summoned/worker pals run a different AI path that recalls/teleports** them (base workers
  out of range → teleport to Palbox; summoned pals auto-recall on death/fast-travel; Hungry Pal Rescuer
  teleports strays). Wild field bosses run a roam-in-territory AI that *does* path. So the real open question
  is: **can we force a directed long-range `MoveTo` on an *owned* pal (or push it onto the roam AI) instead of
  letting it hit the teleport path?** Two unknowns gate a real walk even if we can:
    - **Water / island boundaries** — those Mammorests roam *within* a landmass; a death spot and base split by
      ocean may be unpathable (roam AI likely never picks cross-water destinations).
    - **Distance ceiling** — roaming picks points within a territory radius; a single cross-map `MoveTo` may get
      culled or fail. Untested.
  **Until those are verified, still design the "journey" as an abstraction (timer + visible local milling),
  with a real walk as a stretch goal if the directed-`MoveTo`-on-owned-pal path pans out.**
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
2. Survival window: pals visibly mill about; odds from death→**nearest**-base distance (min over all
   `PalBaseCampModel`s) + pal level.
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
- **Directed long-range `MoveTo` on an owned pal** — can we issue one toward the nearest base and have the
  pal actually path there (the crux now that field Mammorests prove the navmesh carries long-range roam), or
  does the owned-pal AI always intercept with a teleport/recall? If it teleports, can we suppress that and/or
  push the pal onto the wild roam AI? Also test cross-water and cross-map distance limits.

Reusable engine levers from this pass are logged in `PALWORLD_MODDING_REFERENCE.md` §8.
