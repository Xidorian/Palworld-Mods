# Xidorian Studios — Palworld Mods

UE4SS Lua mods for Palworld (single-player / client). Each mod folder drops into
`Palworld\Mods\NativeMods\UE4SS\Mods\` (see each mod's own README).

## Mods
| Mod | Status | What it does |
|-----|--------|--------------|
| **PunishingDeath** | ✅ Released | Lose current-level EXP progress on death; level never drops (exploit-proof). |
| **AIProbe** (Predator & Stealth) | 🔨 WIP | Most pals hunt you, small prey stay passive; crouch/line-of-sight/vision-cone stealth; level-scaled detection. Folder will be renamed `PredatorStealth` on release. |

See [PROGRESS.md](PROGRESS.md) for the live checklist.

## Repo layout
- `PunishingDeath/` — released mod (source, README, design notes, mod-page copy).
- `PunishingDeath-Steam/` — Steam Workshop package variant (no `enabled.txt`).
- `AIProbe/` — Predator & Stealth work-in-progress.
- `art/` — cover art and studio logo.
- `PalClassification.csv` — exported pal AI/classification data (reference archive).
- `PROGRESS.md` — progress checklist.

## Requirements
UE4SS (RE-UE4SS) for Palworld. PC (Steam) only — not Game Pass / console.
