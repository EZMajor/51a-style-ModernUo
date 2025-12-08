# System Overview

## Vision

51alpha is a guild-based faction PvP Ultima Online server emphasizing skill-based combat with Sphere-style mechanics. Players compete through three factions (Vampire, Daemon, Goblin) for territory control in quarterly seasons.

## Design Principles

### 1. PvP/PvM Separation
PvM advantages do not apply in PvP combat. When a player engages another player:
- Talisman bonuses deactivate (5-minute timer)
- Chivalry spell access removed (Sampire build)
- Base damage only (no PvM multipliers)

```csharp
public static bool IsPvPContext(Mobile caster, Mobile target)
{
    if (caster is PlayerMobile && target is PlayerMobile)
        return true;
    if (target is BaseCreature bc && bc.ControlMaster is PlayerMobile)
        return true;
    if (caster is BaseCreature bc2 && bc2.ControlMaster is PlayerMobile && target is PlayerMobile)
        return true;
    return false;
}
```

### 2. Guild-Centric Design
- Solo players cannot participate in factions
- Guilds choose faction allegiance (7-day change cooldown)
- All guild members inherit faction
- Peaceful member option available (non-combatant)

### 3. Skill-Based Gameplay
- Fast skill acquisition via quests (not grinding)
- Meaningful fizzle mechanics (partial resource consumption)
- Free movement during casting (Sphere-style)
- Glicko-2 rating prevents farming low-skill players

### 4. Seasonal Structure
- 90-day seasons with soft rating resets
- Seasonal rewards based on faction standing
- Fresh starts while preserving character progression

## Target Metrics

| Metric | Target | Notes |
|--------|--------|-------|
| Max Concurrent Players | 5,000 | Design ceiling |
| Expected Average | ~1,000 | Steady state |
| Time to PvP Ready | 2-3 hours | Via starter quests |
| Server Tick Budget | <10ms @ 35% load | Performance target |

## System Architecture

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Launcher  │────▶│   Website   │────▶│  Game Server│
│    (WPF)    │     │ (ASP.NET)   │     │ (ModernUO)  │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │
       └───────────────────┴───────────────────┘
                           │
                    ┌──────┴──────┐
                    │  PostgreSQL │
                    │    Redis    │
                    └─────────────┘
```

### Component Responsibilities

| Component | Responsibilities |
|-----------|-----------------|
| Launcher | Discord OAuth, client patching, news |
| Website | Account management, leaderboards, API |
| Game Server | All gameplay, state persistence |
| PostgreSQL | Primary data store, events, telemetry |
| Redis | Session cache, leaderboards (optional) |

## Resolved Design Decisions

| ID | Issue | Resolution |
|----|-------|------------|
| 001 | Talisman/Chivalry interaction | Chivalry requires active Sampire talisman |
| 002 | BOD timing | Points credited immediately on turn-in |
| 003 | Dungeon material access | Dungeons rotate bonuses, not access restriction |
| 004 | Mana on fizzle | 50% consumed (configurable via FizzleManaConsumptionRate) |
| 005 | Faster Casting | Removed from all equipment entirely |
| 006 | Movement during cast | 100% free (Sphere-style) |
| 007 | 50Hz Microtick Engine | Not required; use ModernUO timer wheel |
| 008 | ML Prediction | Removed entirely |
| 009 | Relic freshness timer | Removed (unnecessary complexity) |
| 010 | Talisman timer behavior | Elapsed time tracking with pause on unequip |
| 011 | Damage interruption | Damage does NOT interrupt spells |
| 012 | Equipment interruption | Equipping items does NOT interrupt spells |
| 013 | War mode interruption | Toggling war mode DOES interrupt spells |
| 014 | Targeting order | Target cursor appears IMMEDIATELY on cast |
| 015 | Scroll bonuses | 43% mana reduction + 0.5s speed (circle 3+) |

## Removed Features

| Feature | Reason |
|---------|--------|
| 50Hz Microtick Engine | ModernUO timer wheel sufficient |
| ML-Predictive Cancellation | Unnecessary complexity |
| Relic freshness timer | No gameplay benefit |
| Zero-penalty fizzle | Contradicts skill-based philosophy |
| House taxes | Design decision |

## Configuration Overview

All values tunable without code changes via `Data/51alpha/config.json` or database overrides.

```json
{
  "combat": {
    "fizzleManaRate": 0.5,
    "talismanPvPDisableMinutes": 5,
    "protectionFCPenalty": 0
  },
  "factions": {
    "changeCooldownDays": 7,
    "seasonLengthDays": 90
  },
  "newPlayer": {
    "youngDurationDays": 14
  }
}
```

See [Configuration Reference](../reference/config.md) for complete list.
