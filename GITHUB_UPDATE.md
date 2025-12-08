# GitHub Repository Update Instructions

## Status: ✅ READY FOR UPLOAD

Last updated: 2025-12-08
Version: 2.1.0

All documentation is complete and verified. Spell system clarifications finalized.

## Overview

This guide will help you replace the old flat documentation structure with the new professional dev structure.

### Recent Changes (v2.1.0)

| File | Update |
|------|--------|
| `specs/spell-system.md` | Complete rewrite - damage/equip don't interrupt, target-first, scroll bonuses |
| `docs/systems/combat.md` | Fixed interruption rules, added scroll section |
| `docs/reference/config.md` | Added spell interruption and scroll config keys |
| `docs/architecture/overview.md` | Added decisions 011-015 for spell system |
| `docs/implementation/phases.md` | Updated Phase 3 with expanded spell tasks |
| `docs/implementation/checklist.md` | Expanded Phase 3 checklist (11 new tasks) |
| `docs/implementation/ai-prompts.md` | Added 6 new spell system implementation prompts |
| `README.md` | Updated key decisions table |
| `CHANGELOG.md` | Added v2.1.0 entry |

## Current State (Your Repo)

Your repository likely has files like:
```
51a-style-ModernUo/
├── 51alpha_integration_map.md
├── AFK_Resource_Gathering.md
├── Combat.md
├── Crafting_BODs.md
├── Database_Persistence.md
├── Development_Workflow.md
├── DuelPits.md
├── Dungeons.md
├── FactionVvV.md
├── Glicko_Rating.md
├── ... (many more .md files)
```

## New Structure (This Package)

```
51alpha/
├── README.md
├── QUICKSTART.md
├── CHANGELOG.md
├── .gitignore
├── docs/
│   ├── architecture/
│   ├── systems/
│   ├── implementation/
│   └── reference/
└── specs/
```

---

## Update Steps

### Option A: Clean Replace (Recommended)

```bash
# 1. Clone your repo
git clone https://github.com/EZMajor/51a-style-ModernUo.git
cd 51a-style-ModernUo

# 2. Create a backup branch
git checkout -b backup-old-docs
git push origin backup-old-docs
git checkout main

# 3. Remove all old .md files from root (keep any code folders)
rm -f *.md

# 4. Extract the new documentation (from downloaded zip)
# Copy all contents from the 51alpha/ folder to your repo root
cp -r /path/to/downloaded/51alpha/* .

# 5. Verify structure
ls -la
# Should show: README.md, QUICKSTART.md, CHANGELOG.md, docs/, specs/, .gitignore

# 6. Commit and push
git add -A
git commit -m "Restructure documentation - professional dev structure

- Consolidated 17 files into organized hierarchy
- Added QUICKSTART.md for fast onboarding
- Created docs/ with architecture, systems, implementation, reference
- Created specs/ with verified ModernUO code analysis
- Reduced documentation size by 70%
- All design decisions preserved and verified"

git push origin main
```

### Option B: Manual Update

1. **Download** the 51alpha folder from Claude
2. **Delete** these files from your repo root:
   - All `*.md` files except any you want to keep
3. **Copy** these folders/files to your repo root:
   - `README.md`
   - `QUICKSTART.md`
   - `CHANGELOG.md`
   - `.gitignore`
   - `docs/` (entire folder)
   - `specs/` (entire folder)
4. **Commit** with descriptive message

---

## Files to DELETE from Old Structure

These old files are now consolidated into the new structure:

| Old File | Now In |
|----------|--------|
| `51alpha_integration_map.md` | `docs/architecture/modernuo-integration.md` |
| `Combat.md` | `docs/systems/combat.md` + `specs/spell-system.md` |
| `SpellSystem_Integration.md` | `specs/spell-system.md` |
| `PvM_Talismans.md` | `docs/systems/progression.md` + `specs/talisman-system.md` |
| `FactionVvV.md` | `docs/systems/factions.md` |
| `Glicko_Rating.md` | `docs/systems/progression.md` |
| `Database_Persistence.md` | `docs/architecture/database-schema.md` |
| `Development_Workflow.md` | `docs/implementation/phases.md` |
| `Dungeons.md` | `docs/systems/content.md` |
| `DuelPits.md` | `docs/systems/content.md` |
| `Crafting_BODs.md` | `docs/systems/economy.md` |
| `House_Crafting.md` | `docs/systems/economy.md` |
| `Gumps_UI.md` | `docs/reference/commands.md` |
| `Operations_Infrastructure.md` | `docs/architecture/overview.md` |
| `Security.md` | `docs/architecture/overview.md` |
| `WebAPI.md` | `docs/architecture/overview.md` |
| `Website_Architecture.md` | Deferred (not core to server) |
| `Launcher.md` | Deferred (not core to server) |

### Files Safe to Delete (Stub/Placeholder)

These were placeholders with minimal content:
- `AFK_Resource_Gathering.md`
- `Graveyard_Liche_King_Event.md`
- `Rotating_Dungeons.md`
- `Storage_Shelves.md`
- `Texas_Holdem.md`
- `Town_Cryer.md`
- `Weight_Reduction_Bags.md`

Content preserved in `docs/systems/economy.md` and `docs/systems/content.md`.

---

## Verification

After updating, your repo should have:

```
51a-style-ModernUo/
├── .gitignore
├── README.md
├── QUICKSTART.md
├── CHANGELOG.md
├── docs/
│   ├── architecture/
│   │   ├── overview.md
│   │   ├── database-schema.md
│   │   └── modernuo-integration.md
│   ├── systems/
│   │   ├── combat.md
│   │   ├── factions.md
│   │   ├── progression.md
│   │   ├── economy.md
│   │   └── content.md
│   ├── implementation/
│   │   ├── phases.md
│   │   ├── checklist.md
│   │   └── ai-prompts.md
│   └── reference/
│       ├── locations.md
│       ├── config.md
│       └── commands.md
└── specs/
    ├── spell-system.md
    └── talisman-system.md
```

**Total: 18 files** (down from 25+ scattered files)

---

## After Update

### Step 1: Update GitHub Repository
Follow Option A or B above to update your GitHub repo.

### Step 2: Update Claude Project Knowledge

**Remove these 27 files from your Claude Project:**
```
51alpha_integration_map.md      (37K) → now in modernuo-integration.md
AFK_Resource_Gathering.md       (1K)  → now in economy.md
Combat.md                       (34K) → now in combat.md + spell-system.md
Crafting_BODs.md               (7K)  → now in economy.md
Database_Persistence.md        (7K)  → now in database-schema.md
Development_Workflow.md        (16K) → now in phases.md
DuelPits.md                    (7.5K)→ now in content.md
Dungeons.md                    (8.5K)→ now in content.md
FactionVvV.md                  (26K) → now in factions.md
Glicko_Rating.md               (20K) → now in progression.md
Graveyard_Liche_King_Event.md  (1K)  → now in content.md
Gumps_UI.md                    (7K)  → now in commands.md
House_Crafting.md              (27K) → now in economy.md
Launcher.md                    (88K) → DEFER (not core server)
Operations_Infrastructure.md   (9.5K)→ now in overview.md
PvM_Talismans.md               (33K) → now in progression.md + talisman-system.md
Rotating_Dungeons.md           (2K)  → now in content.md
Rotating_Resource_Gathering.md (7.5K)→ now in economy.md
Security.md                    (18K) → now in overview.md
SpellSystem_Integration.md     (34K) → now in spell-system.md
Storage_Shelves.md             (3K)  → now in economy.md
Swamp_and_Desert_Event.md      (17K) → now in content.md
Texas_Holdem.md                (1K)  → now in economy.md
Town_Cryer.md                  (1K)  → now in content.md
WebAPI.md                      (8.5K)→ DEFER (not core server)
Website_Architecture.md        (49K) → DEFER (not core server)
Weight_Reduction_Bags.md       (2.5K)→ now in economy.md
```

**Total old: 474KB → New: 160KB (66% reduction)**

**Add these 18 new files to Claude Project:**
```
README.md
QUICKSTART.md
CHANGELOG.md
docs/architecture/overview.md
docs/architecture/database-schema.md
docs/architecture/modernuo-integration.md
docs/systems/combat.md
docs/systems/factions.md
docs/systems/progression.md
docs/systems/economy.md
docs/systems/content.md
docs/implementation/phases.md
docs/implementation/checklist.md
docs/implementation/ai-prompts.md
docs/reference/locations.md
docs/reference/config.md
docs/reference/commands.md
specs/spell-system.md
specs/talisman-system.md
```

### Step 3: Verify
Ask Claude: "What's the mana consumption on spell fizzle?"
Should answer: "50% (configurable via FizzleManaConsumptionRate)" referencing `specs/spell-system.md`

### Deferred Documents

These are NOT included in the new structure (not core to game server):
- `Launcher.md` (88KB) - WPF launcher, separate project
- `Website_Architecture.md` (49KB) - ASP.NET website, separate project
- `WebAPI.md` (8.5KB) - API design, separate project

Add these back when ready to work on those components.
