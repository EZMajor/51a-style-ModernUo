# Changelog

All notable changes to 51alpha documentation and design.

## [2.1.0] - 2025-12-08

### Spell System Clarifications
- **Damage Interruption**: Confirmed damage does NOT interrupt spells (major change from OSI)
- **Equipment Interruption**: Equipping items does NOT interrupt spells
- **War Mode**: Toggling war mode DOES interrupt spells
- **Bandages**: Using bandages DOES interrupt spells
- **Targeting**: Target cursor appears IMMEDIATELY on cast (before delay)
- **Scroll System**: Added 43% mana reduction + 0.5s speed bonus (circle 3+)

### Updated Files
- `specs/spell-system.md` - Complete rewrite v2.0.0 with verified flow
- `docs/systems/combat.md` - Corrected interruption rules, added scroll section
- `docs/reference/config.md` - Added interruption and scroll config keys

### Implementation Notes
- FC still being removed from all equipment
- 50% mana on fizzle confirmed (configurable)
- Target-first flow requires verification against ModernUO SpellState.Sequencing usage

---

## [2.0.0] - 2025-12-08

### Restructured
- Complete documentation reorganization into professional dev structure
- Consolidated 17 files (525KB) into 15 files (153KB)
- Created hierarchical organization: Architecture → Systems → Implementation → Reference

### Added
- `README.md` - Project overview with quick start
- `docs/architecture/overview.md` - Design principles and decisions
- `docs/architecture/database-schema.md` - Condensed PostgreSQL schema
- `docs/architecture/modernuo-integration.md` - Hooks and extension points
- `docs/systems/combat.md` - Sphere-style combat mechanics
- `docs/systems/factions.md` - VvV and siege warfare
- `docs/systems/progression.md` - Talismans and Glicko ratings
- `docs/systems/economy.md` - Crafting, BODs, housing
- `docs/systems/content.md` - Dungeons, bounties, NPE, tournaments
- `docs/implementation/phases.md` - 8-phase development plan
- `docs/implementation/checklist.md` - 70+ actionable tasks
- `docs/implementation/ai-prompts.md` - Copy-paste implementation prompts
- `docs/reference/locations.md` - World coordinates
- `docs/reference/config.md` - All configuration parameters
- `docs/reference/commands.md` - Admin command reference
- `specs/spell-system.md` - Verified ModernUO code analysis
- `specs/talisman-system.md` - PvP/PvM separation logic

### Verified (Against ModernUO GitHub)
- SpellState enum: 3 states (None, Casting, Sequencing), not 6
- DisturbType enum: 5 types, no Movement type
- CheckSequence() flow: Reagents first, mana after fizzle check
- GetCastDelay() FC calculation: Protection subtracts from FC
- Timer wheel: O(1) operations, event-driven

### Decisions Finalized
- **Option C Mana**: 50% consumed on fizzle (configurable)
- **Faster Casting**: Removed from all equipment
- **Protection Spell**: 0 FC penalty (configurable)
- **Movement**: 100% free during casting
- **50Hz Microtick**: NOT REQUIRED (ModernUO timer wheel sufficient)
- **Talisman Timer**: Elapsed time tracking with pause on unequip

### Removed
- 50Hz Microtick Engine (unnecessary complexity)
- ML-Predictive Cancellation (no benefit)
- Relic Freshness Timer (removed from design)
- Zero-Penalty Fizzle (contradicts skill-based design)

### Archived
- All original documentation moved to `_archive/`
- Preserved for reference during implementation

---

## [1.1.0] - 2025-12-05

### Added
- Deep spell system analysis with ModernUO verification
- 50Hz analysis document (concluded not needed)
- Option A/B/C mana consumption comparison
- FC removal implementation guide

### Changed
- Mana consumption from Option A to Option C
- Movement threshold from 50% to 100% free

---

## [1.0.0] - 2025-01

### Added
- Initial architecture documentation
- Database schema (31 tables)
- World locations (22KB)
- ModernUO integration guide
- Implementation checklist (70+ tasks)
- VvV siege system design
- Tournament system design
- Daily content system
- New player experience
- Static house rental system

### Identified Issues
- 6 design errors found during ModernUO verification
- Multiple conflicting specifications across documents
- Overcomplicated 50Hz engine proposal
