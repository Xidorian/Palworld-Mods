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

## 🅿️ Parked
- Pals released into the world on death, walk home, may die (big standalone project).
