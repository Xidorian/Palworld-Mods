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

## 💡 Mod 7 — Mod Options Framework  (DECIDED 2026-08-01: ADOPT as consumers — gated on compat test)
Original idea: build a reusable **in-game options screen** any mod (ours + other modders')
registers its settings into, so mods stop shipping config files / keybind toggles.

**Discovery — 2026-08-01 (desk research): this already exists, and building our own is almost
certainly wasted effort.**

**[Mod Options Framework](https://www.nexusmods.com/palworld/mods/4408)** by Elvlin / "cloirecrom"
— GitHub **[Elvlin/Mod-Options-Framework](https://github.com/Elvlin/Mod-Options-Framework)**,
**MIT** (source + SDK), v0.1.10, actively maintained (updated 2026-08-01). It is exactly this mod,
already built and better-architected than a first pass would be:
- **Native UMG** — adds one "Mod Options" entry to Palworld's Esc menu with native-style per-mod
  pages. **So our `GuiConsoleEnabled=0` constraint is moot** — that only disabled *ImGui*; this
  never used ImGui. Kills the rendering-path blocker that gated the whole idea.
- **No per-frame update / actor scan / timer / settings poll** — matches our anti-stutter rule.
  Values publish on Apply via a generation counter; consumers `sync()` off an existing event.
- **Documented Lua SDK** ([DEVELOPER_API.md](https://github.com/Elvlin/Mod-Options-Framework/blob/main/DEVELOPER_API.md)):
  copy `PalModOptionsClient.lua` + `pmo_json.lua` from its DeveloperSDK into your mod's `Scripts/`,
  then `local options = require("PalModOptionsClient")` →
  `options.register_when_ready(schema, function(settings, err) … end)` →
  read with `options.get("key")` / `options.get()`. Seven control types (boolean, integer, number,
  enum, **keybind**, text, section), per-mod persistence at `PalModOptions\Scripts\config\<id>.ini`
  (JSON-encoded lines), apply modes `event` / `restart_mod` / `game_restart`, cross-field
  constraints, localization (17 locales), per-page theming, spelled-out multiplayer contract.
- **Required vs optional dependency** patterns supported — a mod can hard-require it, or use it
  when present and fall back to built-in defaults when absent (matches our `pcall`+defaults habit).

**🔴 The one real blocker — UE4SS distribution mismatch.** The framework requires **UE4SS
Experimental for Palworld** (dep id `UE4SSExperimentalPW`) and installs to
`Pal\Binaries\Win64\ue4ss\Mods\PalModOptions\`. **Our build runs the Steam-Workshop NativeMods
UE4SS** (`Palworld\Mods\NativeMods\UE4SS\Mods\…`) — a different distribution *and* mod path. So the
decision hinges entirely on: **does our current Steam-Workshop UE4SS run this framework, or does
adopting it mean migrating to UE4SS Experimental** (and re-homing PredatorStealth's runtime +
PalSchema onto it)? Needs in-game verification.

**DECISION (2026-08-01): ADOPT as consumers.** Reuse-first + API-vs-build + our own "native UMG
is a big lift" note all agree — don't reinvent a maintained, MIT, native, no-poll framework.
Build our own ONLY if the compat test below rules it out (incompatible distro we won't migrate to,
or it goes abandoned); the original build spec (registration API, persistence, standalone public
repo) stays parked here as that fallback.

**Next steps (ordered):**
1. **🔬 COMPAT TEST (blocker — user runs in-game, next action).** Determine whether Mod Options
   Framework loads under our **Steam-Workshop NativeMods UE4SS**, or forces a switch to **UE4SS
   Experimental**. Test shape: install the framework + a minimal test consumer (from its
   DeveloperSDK), launch, check whether **Esc → Mod Options** appears, and read `UE4SS.log` for its
   load lines. Outcomes → (a) loads as-is: proceed to step 2; (b) needs Experimental: weigh the
   cost of migrating our UE4SS distro (re-home PredatorStealth runtime + PalSchema) before
   committing; (c) won't work: fall back to build-our-own.
2. Integrate our mods as **OPTIONAL** consumers (require-if-present via `register_when_ready`, else
   fall back to existing `PreyList.txt` / keybind config — matches our `pcall`+defaults habit).
   First candidates: PredatorStealth's prey list + viewing distance, and the QoL toggles.
3. Log the confirmed integration recipe (SDK copy-in, schema shape, distro verdict) into
   `PALWORLD_MODDING_REFERENCE.md`.

Reusable levers from this pass → log in `PALWORLD_MODDING_REFERENCE.md` once the distro question
is settled in-game.

Supersedes the [Config UX open question](#-config-ux--in-game-options-menu-items-open-question-cross-cutting)
below (its feasibility notes are now answered by the above).

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
