# Xidorian Studios — Palworld Mods

Shared knowledge base and studio assets for my Palworld UE4SS Lua mods. **Each mod
now lives in its own repository:**

| Mod | Repo | What it does |
|-----|------|--------------|
| **Punishing Death** | [PunishingDeath](https://github.com/Xidorian/PunishingDeath) | Die and lose ~10% of your total EXP — drops a level and strips that level's technologies + tech/stat points. Exploit-proof. |
| **Predators & Stealth** | [PredatorStealth](https://github.com/Xidorian/PredatorStealth) | Most wild Pals hunt you on sight; a curated prey list stays passive. Line-of-sight detection, crouch stealth, level-scaled awareness, hide-to-escape. |

## This repo (the hub)
Shared dev material — not shipped with any mod:
- `PALWORLD_MODDING_REFERENCE.md` — consolidated UE4SS internals knowledge base (function levers, EXP/level chain, pal AI + hate system, data tables, hard-won gotchas).
- `PROGRESS.md` — progress checklist across all mods.
- `PalClassification.csv` — exported pal AI / classification data (reference archive).
- `art/` — cover art and studio logo.
- `CLAUDE.md` — working notes / session instructions.

## Local layout
```
PalworldMods\        this hub (docs + art)
PunishingDeath\      clone of the PunishingDeath repo
PredatorStealth\     clone of the PredatorStealth repo
```

## Requirements
UE4SS (RE-UE4SS) for Palworld. PC (Steam) only — not Game Pass / console.
