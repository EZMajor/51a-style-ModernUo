# 51alpha

**Guild-Based Faction PvP Server for Ultima Online**

Built on ModernUO with Sphere-style combat mechanics.

> **New Developer?** Start with [QUICKSTART.md](QUICKSTART.md) - running in under an hour.

## Quick Start

```bash
# Prerequisites
- .NET 10 SDK
- PostgreSQL 15+
- ModernUO v24.0.0+

# Setup
1. Clone ModernUO repository
2. Apply 51alpha patches to Scripts/
3. Run database schema (docs/architecture/database-schema.md)
4. Configure Data/51alpha/config.json
5. dotnet run
```

## Project Status

| Component | Status | Notes |
|-----------|--------|-------|
| Architecture | ✅ Complete | All systems designed |
| Database Schema | ✅ Complete | 31 tables |
| ModernUO Integration | ✅ Verified | Hooks documented |
| Phase 1: Launcher | 🔲 Ready | 2 weeks estimated |
| Phase 2: Website | 🔲 Ready | 2 weeks estimated |
| Phase 3: Server Core | 🔲 Ready | 4 weeks estimated |

## Documentation

### Architecture
- [System Overview](docs/architecture/overview.md) - Core design, principles, metrics
- [Database Schema](docs/architecture/database-schema.md) - PostgreSQL tables and relationships
- [ModernUO Integration](docs/architecture/modernuo-integration.md) - Hooks, patches, extension points

### Game Systems
- [Combat & Spells](docs/systems/combat.md) - Sphere-style casting, damage, interrupts
- [Factions & VvV](docs/systems/factions.md) - Three-faction warfare, sieges, territory
- [Progression](docs/systems/progression.md) - Talismans, ratings, builds
- [Economy](docs/systems/economy.md) - Crafting, BODs, housing, gold sinks
- [Content](docs/systems/content.md) - Dungeons, daily content, events, NPE

### Implementation
- [Phase Guide](docs/implementation/phases.md) - 8 phases, 16 weeks
- [Task Checklist](docs/implementation/checklist.md) - 70+ actionable tasks
- [AI Prompts](docs/implementation/ai-prompts.md) - Ready-to-use implementation prompts

### Reference
- [World Locations](docs/reference/locations.md) - Coordinates, regions, spawns
- [Configuration](docs/reference/config.md) - All tunable parameters
- [Admin Commands](docs/reference/commands.md) - Server management

### Technical Specs
- [Spell System](specs/spell-system.md) - Verified ModernUO code flows
- [Talisman System](specs/talisman-system.md) - PvP/PvM separation logic

## Core Design

### Philosophy
- **Skill-based PvP**: Free movement during casting, meaningful fizzle punishment
- **Guild-centric**: Solo players cannot join factions; social play required
- **Clear separation**: PvM bonuses disabled in PvP (level playing field)
- **Accessible**: 2-3 hours to PvP-ready via quests, not grinding

### Key Decisions

| Area | Decision | Rationale |
|------|----------|-----------|
| Combat Style | Sphere-style (free movement) | Skill-based, mobile gameplay |
| Damage Interrupt | Disabled | Combat focuses on positioning, not spell immunity |
| Mana on Fizzle | 50% consumed (configurable) | Punishes mistakes, not devastating |
| Faster Casting | Removed entirely | Fixed timing, no gear dependency |
| Scrolls | 43% cheaper + 0.5s faster | Valuable consumables worth using |
| Targeting | Immediate (before delay) | Player aims without timer pressure |
| Factions | 3 guilds, 7-day switch cooldown | Natural 2v1 dynamics |
| Talismans | 5-min PvP disable | PvM rewards don't affect PvP |
| Seasons | Quarterly (90 days) | Fresh starts + meaningful progression |

## Technology Stack

| Layer | Technology |
|-------|------------|
| Game Server | ModernUO v24.0.0+ (.NET 10, C# 13) |
| Database | PostgreSQL 15+ |
| Cache | Redis (optional) |
| Website | ASP.NET Core + React |
| Launcher | WPF + Discord OAuth2 |
| Client | ClassicUO |

## Directory Structure

### Documentation
```
51alpha/
├── README.md              # This file
├── QUICKSTART.md          # Get running in 1 hour
├── CHANGELOG.md           # Version history
├── docs/
│   ├── architecture/      # System design
│   │   ├── overview.md
│   │   ├── database-schema.md
│   │   └── modernuo-integration.md
│   ├── systems/           # Game mechanics
│   │   ├── combat.md
│   │   ├── factions.md
│   │   ├── progression.md
│   │   ├── economy.md
│   │   └── content.md
│   ├── implementation/    # Development guides
│   │   ├── phases.md
│   │   ├── checklist.md
│   │   └── ai-prompts.md
│   └── reference/         # Lookup tables
│       ├── locations.md
│       ├── config.md
│       └── commands.md
├── specs/                 # Technical specifications
│   ├── spell-system.md
│   └── talisman-system.md
└── _archive/              # Legacy docs (reference only)
```

### Server Code (to create in ModernUO)
```
Scripts/Sphere51a/
├── Core/           # Initialization, config, database
├── Combat/         # Spell modifications, damage calculation
├── Factions/       # VvV, sieges, territory control
├── Progression/    # Talismans, ratings, builds
├── Economy/        # BODs, housing, gold sinks
├── Content/        # Dungeons, events, NPE
├── Telemetry/      # Logging, metrics
└── Commands/       # Admin commands
```

## Contributing

This is a solo project with AI-assisted development. Documentation is structured for AI consumption with explicit file paths, acceptance criteria, and implementation prompts.

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 2.1.0 | 2025-12-08 | Spell system clarifications, scroll bonuses |
| 2.0.0 | 2025-12-05 | Professional restructure, verified implementations |
| 1.0.0 | 2025-01 | Initial architecture documentation |
