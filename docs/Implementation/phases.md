# Implementation Phases

8 phases over 16 weeks. Solo developer with AI assistance.

## Phase Overview

| Phase | Focus | Weeks | Hours |
|-------|-------|-------|-------|
| 1 | Foundation | 1-2 | 20-30 |
| 2 | Faction Core | 2-3 | 25-35 |
| 3 | Spell System | 3-4 | 20-30 |
| 4 | Talisman System | 4-5 | 20-25 |
| 5 | Siege System | 5-7 | 30-40 |
| 6 | Daily Content | 7-9 | 25-30 |
| 7 | Tournament System | 9-11 | 25-35 |
| 8 | NPE & Polish | 11-16 | 35-50 |

**Total**: 200-275 hours

---

## Phase 1: Foundation (Weeks 1-2)

### Objectives
- Project structure in ModernUO
- Database connection
- Configuration system
- Logging framework

### Tasks

| Task | Files | Hours |
|------|-------|-------|
| Create directory structure | Sphere51a/* | 1 |
| Add Npgsql NuGet | UOContent.csproj | 0.5 |
| Implement Sphere51aCore | Core/Sphere51aCore.cs | 2 |
| Implement Database | Core/Database.cs | 3 |
| Implement Config | Core/Config.cs | 2 |
| Create PostgreSQL schema | SQL scripts | 2 |
| Test initialization | - | 2 |
| Implement EventLogger | Telemetry/EventLogger.cs | 2 |

### Deliverables
- Server starts with "[51alpha] Ready"
- Database connection verified
- Config loads from file

---

## Phase 2: Faction Core (Weeks 2-3)

### Objectives
- Three-faction structure
- Guild-faction binding
- Combat status system

### Tasks

| Task | Files | Hours |
|------|-------|-------|
| Create FactionId enum | Factions/FactionId.cs | 1 |
| Implement FactionManager | Factions/FactionManager.cs | 4 |
| Guild extension methods | Factions/GuildExtension.cs | 2 |
| Combat status system | Factions/CombatStatus.cs | 3 |
| Faction change cooldown | FactionManager.cs | 2 |
| Admin commands | Commands/FactionCommands.cs | 2 |
| Faction gump | Gumps/FactionStatusGump.cs | 3 |
| Database queries | FactionManager.cs | 3 |

### Deliverables
- Guilds can join/leave factions
- 7-day cooldown enforced
- Combatant/Peaceful toggle works

---

## Phase 3: Spell System (Weeks 3-4)

### Objectives
- Sphere-style movement (free during cast)
- No damage interruption (51alpha rule)
- Target-first flow (immediate targeting)
- Option C mana consumption (50% on fizzle)
- FC removal
- Scroll bonuses (43% mana, 0.5s speed)

### Tasks

| Task | Files | Hours |
|------|-------|-------|
| Create Sphere51aConfig | Core/Sphere51aConfig.cs | 2 |
| Modify OnCasterMoving | Spells/Base/Spell.cs | 1 |
| Modify OnCasterHurt | Spell.cs (no interrupt) | 1 |
| Modify OnCasterEquiping | Spell.cs (no interrupt) | 1 |
| Implement target-first flow | Spell.cs Cast() | 4 |
| Implement Option C mana | Spell.cs CheckSequence | 4 |
| Implement scroll mana reduction | Spell.cs ScaleMana | 2 |
| Implement scroll speed bonus | Spell.cs GetCastDelay | 2 |
| Protection FC change | Spell.cs GetCastDelay | 1 |
| FC removal (loot) | BaseRunicTool.cs, etc. | 4 |
| [RemoveAllFC] command | Commands/AdminCommands.cs | 2 |
| Test spell casting | - | 4 |
| PvP context detection | SpellHelper.cs | 2 |

### Deliverables
- Free movement during casting
- Damage does NOT interrupt
- Equipping does NOT interrupt
- War mode toggle DOES interrupt
- Target cursor appears immediately on cast
- Scrolls 43% cheaper mana
- Circle 3+ scrolls cast 0.5s faster
- No FC on any equipment
- Protection has no FC penalty
- 50% mana consumed on fizzle

---

## Phase 4: Talisman System (Weeks 4-5)

### Objectives
- Four talisman types
- PvP disable mechanic
- Chivalry gating

### Tasks

| Task | Files | Hours |
|------|-------|-------|
| TalismanState class | Progression/TalismanState.cs | 3 |
| BuildManager service | Progression/BuildManager.cs | 4 |
| BaseTalisman modifications | Items/Talismans/BaseTalisman.cs | 3 |
| Chivalry check | ChivalrySpell.cs | 2 |
| Damage calculation hooks | AOS.cs | 3 |
| Talisman crafting | Tinkering additions | 3 |
| Test talisman disable | - | 2 |

### Deliverables
- Talismans craftable
- 5-min PvP disable works
- Timer pauses on unequip
- Chivalry requires Sampire talisman

---

## Phase 5: Siege System (Weeks 5-7)

### Objectives
- Town control
- Siege battles
- Scoring and objectives
- Defenses

### Tasks

| Task | Files | Hours |
|------|-------|-------|
| TownControl manager | Siege/TownControl.cs | 4 |
| SiegeBattle class | Siege/SiegeBattle.cs | 6 |
| SiegeManager | Siege/SiegeManager.cs | 5 |
| Sigil objective | Siege/SiegeSigil.cs | 4 |
| Altar objectives | Siege/SiegeAltar.cs | 3 |
| Priest NPCs | Mobiles/SiegePriest.cs | 2 |
| Trap items | Items/Traps/*.cs | 4 |
| Turret items | Items/Turrets/*.cs | 3 |
| Siege gump | Gumps/SiegeStatusGump.cs | 3 |
| Admin commands | Commands/SiegeCommands.cs | 2 |
| Test full siege | - | 4 |

### Deliverables
- Sieges trigger automatically
- All objectives functional
- Defenses purchasable
- Victory conditions work

---

## Phase 6: Daily Content (Weeks 7-9)

### Objectives
- Bounty system
- Faction quests
- Silver currency

### Tasks

| Task | Files | Hours |
|------|-------|-------|
| BountyManager | Daily/BountyManager.cs | 4 |
| Bounty tracking hooks | Monster/Resource death | 3 |
| BountyBoard gump | Gumps/BountyBoardGump.cs | 3 |
| FactionQuestManager | Daily/FactionQuestManager.cs | 4 |
| Quest boss spawning | Daily/FactionQuestBoss.cs | 4 |
| SilverManager | Currency/SilverManager.cs | 3 |
| Silver vendor | Mobiles/SilverVendor.cs | 3 |
| Daily reset timer | Core/Sphere51aTimers.cs | 2 |
| Test daily cycle | - | 2 |

### Deliverables
- Three daily bounties
- Faction quest boss spawns
- Silver currency works
- Rewards distributed correctly

---

## Phase 7: Tournament System (Weeks 9-11)

### Objectives
- Registration system
- Bracket generation
- Match processing
- Rewards

### Tasks

| Task | Files | Hours |
|------|-------|-------|
| TournamentManager | Tournament/TournamentManager.cs | 5 |
| Bracket generation | Tournament/TournamentBracket.cs | 4 |
| Match processing | Tournament/TournamentMatch.cs | 4 |
| Arena setup | Tournament/TournamentArena.cs | 3 |
| Registrar NPC | Mobiles/TournamentRegistrar.cs | 2 |
| Tournament gump | Gumps/TournamentGump.cs | 3 |
| Tournament coins | Currency/TournamentCoinManager.cs | 2 |
| Cosmetic vendor | Mobiles/CosmeticVendor.cs | 2 |
| Spectator betting | Tournament/TournamentBetting.cs | 3 |
| Test full tournament | - | 4 |

### Deliverables
- Tournaments run on schedule
- Brackets work correctly
- Rewards distributed
- Betting functional

---

## Phase 8: NPE & Polish (Weeks 11-16)

### Objectives
- Young player protection
- Training islands
- Town Cryer
- Final testing

### Tasks

| Task | Files | Hours |
|------|-------|-------|
| YoungPlayerManager | NPE/YoungPlayerManager.cs | 4 |
| Training island content | NPE/TrainingIsland.cs | 8 |
| Starter quests | NPE/StarterQuests.cs | 6 |
| Ferry NPC | Mobiles/FerryNPC.cs | 2 |
| New Player Guide NPC | Mobiles/NewPlayerGuide.cs | 2 |
| Town Cryer NPC | Mobiles/TownCryer.cs | 3 |
| Glicko integration | Glicko/*.cs | 6 |
| Leaderboard gump | Gumps/LeaderboardGump.cs | 3 |
| Load testing | - | 4 |
| Security audit | - | 4 |
| Bug fixes | - | 8 |

### Deliverables
- NPE complete
- All systems integrated
- Performance verified
- Security reviewed

---

## Dependencies

```
Phase 1: Foundation
    ↓
Phase 2: Faction Core
    ↓
Phase 3: Spell System ←── Can parallelize
    ↓
Phase 4: Talisman System
    ↓
Phase 5: Siege System
    ↓
Phase 6: Daily Content
    ↓
Phase 7: Tournament System
    ↓
Phase 8: NPE & Polish
```

## Risk Factors

| Risk | Mitigation |
|------|------------|
| ModernUO API changes | Pin to specific version |
| Database performance | Index optimization, caching |
| Siege complexity | Admin toggles for objectives |
| Balance issues | Configurable values |

## Success Criteria

| Metric | Target |
|--------|--------|
| Server starts | No errors |
| All phases complete | < 300 hours |
| Test coverage | Core flows verified |
| Performance | < 10ms tick at 35% load |
