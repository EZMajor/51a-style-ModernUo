# Implementation Checklist

Actionable tasks with dependencies, files, and acceptance criteria.

## How to Use

- Check off tasks as completed: `[x]`
- Dependencies must complete before starting
- AI Prompt Hint helps when asking for implementation help

---

## Phase 1: Foundation

### 1.1 Project Setup

- [ ] **Create directory structure**
  - Files: `Sphere51a/*` directories per integration guide
  - Criteria: All folders exist, solution builds
  - AI Hint: "Create ModernUO project directory structure"

- [ ] **Add Npgsql NuGet**
  - Files: `UOContent.csproj`
  - Criteria: `dotnet restore` succeeds
  - AI Hint: "Add Npgsql package to .NET project"

- [ ] **Create Sphere51aCore**
  - Files: `Sphere51a/Core/Sphere51aCore.cs`
  - Criteria: Server logs "[51alpha] Initializing..."
  - AI Hint: "ModernUO initialization hook"

### 1.2 Database

- [ ] **Create database**
  - SQL: `CREATE DATABASE "51alpha"`
  - Criteria: Can connect via psql

- [ ] **Run schema**
  - Files: See `docs/architecture/database-schema.md`
  - Criteria: All 31 tables exist

- [ ] **Implement Database.cs**
  - Files: `Sphere51a/Core/Database.cs`
  - Criteria: Server logs "Database connected"
  - AI Hint: "Npgsql connection pooling async"

### 1.3 Configuration

- [ ] **Create config.json**
  - Files: `Data/51alpha/config.json`
  - Criteria: File exists with defaults

- [ ] **Implement Config.cs**
  - Files: `Sphere51a/Core/Config.cs`
  - Criteria: Config values load from file + database
  - AI Hint: "JSON config with database override"

- [ ] **Admin reload command**
  - Files: `Commands/AdminCommands.cs`
  - Criteria: `[s51a reload]` refreshes config

### 1.4 Telemetry

- [ ] **EventLogger**
  - Files: `Sphere51a/Telemetry/EventLogger.cs`
  - Criteria: Events appear in s51a_events table
  - AI Hint: "Async event logging PostgreSQL batching"

---

## Phase 2: Faction Core

### 2.1 Faction Structure

- [ ] **FactionId enum**
  - Files: `Sphere51a/Factions/FactionId.cs`
  - Criteria: Vampire=1, Daemon=2, Goblin=3

- [ ] **FactionManager**
  - Files: `Sphere51a/Factions/FactionManager.cs`
  - Criteria: Get/set guild factions works
  - AI Hint: "Guild faction management database persistence"

- [ ] **Guild extension**
  - Files: `Sphere51a/Factions/GuildExtension.cs`
  - Criteria: `guild.GetFaction()` works

### 2.2 Combat Status

- [ ] **CombatStatus manager**
  - Files: `Sphere51a/Factions/CombatStatus.cs`
  - Criteria: Toggle combatant/peaceful
  - Dependencies: FactionManager

- [ ] **24hr cooldown**
  - Criteria: Cannot toggle within 24 hours

- [ ] **Forced combatant**
  - Criteria: Attacking flags for 24 hours

### 2.3 UI

- [ ] **Faction status gump**
  - Files: `Sphere51a/Gumps/FactionStatusGump.cs`
  - Criteria: Shows faction, stats, controls

- [ ] **Faction commands**
  - Files: `Commands/FactionCommands.cs`
  - Criteria: `[faction join/leave/status]`

---

## Phase 3: Spell System

### 3.1 Configuration

- [ ] **Sphere51aConfig class**
  - Files: `Core/Sphere51aConfig.cs`
  - Criteria: All combat/spell settings accessible
  - Include: DamageInterrupts, EquipInterrupts, WarModeInterrupts, ScrollManaReduction, ScrollSpeedBonus

### 3.2 Movement

- [ ] **OnCasterMoving override**
  - Files: Modify `Spells/Base/Spell.cs`
  - Criteria: Can move during casting
  - AI Hint: "Override OnCasterMoving return true Sphere-style"

### 3.3 Interruption

- [ ] **OnCasterHurt - disable damage interrupt**
  - Files: Modify `Spells/Base/Spell.cs`
  - Criteria: Damage does NOT interrupt spells
  - AI Hint: "OnCasterHurt early return when 51alpha enabled"

- [ ] **OnCasterEquiping - disable equip interrupt**
  - Files: Modify `Spells/Base/Spell.cs`
  - Criteria: Equipping items does NOT interrupt
  - AI Hint: "OnCasterEquiping return true when 51alpha enabled"

- [ ] **War mode interrupt**
  - Files: Spell.cs or relevant hook
  - Criteria: Toggling war mode DOES interrupt spells

### 3.4 Target-First Flow

- [ ] **Modify Cast() for immediate targeting**
  - Files: Modify `Spells/Base/Spell.cs`
  - Criteria: OnCast() called immediately, cursor shows before delay
  - AI Hint: "Cast method calls OnCast immediately target-first Sphere"

- [ ] **Move delay to after target selection**
  - Files: Modify `Spells/Base/Spell.cs`
  - Criteria: CastTimer starts after CheckSequence target validation
  - AI Hint: "CheckSequence starts CastDelayTimer after mana consumed"

### 3.5 Mana Consumption

- [ ] **Option C in CheckSequence**
  - Files: Modify `Spells/Base/Spell.cs`
  - Criteria: 100% mana consumed before delay, 50% refund on fizzle
  - AI Hint: "CheckSequence consume mana before timer fizzle refund 50%"

### 3.6 Scroll Bonuses

- [ ] **ScaleMana scroll reduction**
  - Files: Modify `Spells/Base/Spell.cs`
  - Criteria: Scrolls cost 43% less mana (scalar /= 1.755)
  - AI Hint: "ScaleMana scroll SpellScroll divide by 1.755"

- [ ] **GetCastDelay scroll speed**
  - Files: Modify `Spells/Base/Spell.cs`
  - Criteria: Circle 3+ scrolls cast 0.5s faster
  - AI Hint: "GetCastDelay scroll SpellScroll subtract 0.5 circle 3 or higher"

### 3.7 Protection Spell

- [ ] **GetCastDelay modification**
  - Files: Modify `Spells/Base/Spell.cs`
  - Criteria: Protection has 0 FC penalty
  - AI Hint: "GetCastDelay configurable Protection FC penalty zero"

### 3.8 FC Removal

- [ ] **Remove from BaseRunicTool**
  - Files: `Items/Tools/BaseRunicTool.cs`
  - Criteria: No FC in random attributes

- [ ] **Remove from RandomItemGenerator**
  - Files: `Misc/RandomItemGenerator.cs`
  - Criteria: No FC in loot pools

- [ ] **Remove from LootPack**
  - Files: `Misc/LootPack.cs`
  - Criteria: No FC in loot tables

- [ ] **Remove from crafting**
  - Files: `Engines/Craft/Def*.cs`
  - Criteria: No FC bonuses

- [ ] **[RemoveAllFC] command**
  - Files: `Commands/AdminCommands.cs`
  - Criteria: Strips FC from all items

### 3.9 Testing

- [ ] Test free movement during cast
- [ ] Test damage does NOT interrupt
- [ ] Test equipping does NOT interrupt
- [ ] Test war mode DOES interrupt
- [ ] Test target cursor appears immediately
- [ ] Test delay starts after target selection
- [ ] Test fizzle 50% mana refund
- [ ] Test scroll mana reduction (43%)
- [ ] Test scroll speed bonus (0.5s for circle 3+)
- [ ] Test Protection no FC effect
- [ ] Test no FC on loot/craft

---

## Phase 4: Talisman System

### 4.1 Core Classes

- [ ] **TalismanState**
  - Files: `Progression/TalismanState.cs`
  - Criteria: Timer with pause/resume

- [ ] **BuildManager**
  - Files: `Progression/BuildManager.cs`
  - Criteria: PvP context detection

### 4.2 Talisman Items

- [ ] **BaseTalisman modifications**
  - Files: `Items/Talismans/BaseTalisman.cs`
  - Criteria: OnEquip/OnRemoved hooks

- [ ] **Four talisman types**
  - Criteria: Dexer, Tamer, Sampire, TH

### 4.3 Integration

- [ ] **Chivalry gating**
  - Files: `Spells/Chivalry/*.cs`
  - Criteria: Requires active Sampire talisman

- [ ] **Damage hooks**
  - Files: `Misc/AOS.cs` or similar
  - Criteria: Bonuses apply in PvM only

### 4.4 Crafting

- [ ] **Talisman recipes**
  - Files: Tinkering craft definitions
  - Criteria: Craftable with relics + skills

### 4.5 Testing

- [ ] Test PvM damage bonus
- [ ] Test PvP no bonus
- [ ] Test 5-min disable
- [ ] Test pause/resume

---

## Phase 5: Siege System

### 5.1 Town Control

- [ ] **TownControl manager**
  - Files: `Siege/TownControl.cs`
  - Criteria: Track city ownership

### 5.2 Battle Core

- [ ] **SiegeBattle class**
  - Files: `Siege/SiegeBattle.cs`
  - Criteria: State machine for battle

- [ ] **SiegeManager**
  - Files: `Siege/SiegeManager.cs`
  - Criteria: Trigger, manage, end sieges

### 5.3 Objectives

- [ ] **Sigil**
  - Files: `Siege/SiegeSigil.cs`
  - Criteria: 30s capture, 500 points

- [ ] **Altars**
  - Files: `Siege/SiegeAltar.cs`
  - Criteria: Proximity control, resurrection

- [ ] **Priests**
  - Files: `Mobiles/SiegePriest.cs`
  - Criteria: Faction resurrection

### 5.4 Defenses

- [ ] **Traps**
  - Files: `Items/Traps/*.cs`
  - Criteria: Alarm, Snare, Smoke, ManaDrain

- [ ] **Turrets**
  - Files: `Items/Turrets/*.cs`
  - Criteria: Arrow, Magic turrets

### 5.5 UI & Commands

- [ ] **Siege gump**
  - Files: `Gumps/SiegeStatusGump.cs`

- [ ] **Admin commands**
  - Files: `Commands/SiegeCommands.cs`
  - Criteria: start, stop, status, toggles

### 5.6 Testing

- [ ] Test siege trigger
- [ ] Test all objectives
- [ ] Test victory conditions
- [ ] Test defenses

---

## Phase 6: Daily Content

### 6.1 Bounties

- [ ] **BountyManager**
  - Files: `Daily/BountyManager.cs`
  - Criteria: Generate/track bounties

- [ ] **Tracking hooks**
  - Files: EventSink handlers
  - Criteria: Monster/resource/craft credited

- [ ] **Bounty gump**
  - Files: `Gumps/BountyBoardGump.cs`

### 6.2 Faction Quest

- [ ] **FactionQuestManager**
  - Files: `Daily/FactionQuestManager.cs`
  - Criteria: Location selection, boss spawn

- [ ] **Quest boss**
  - Files: `Daily/FactionQuestBoss.cs`
  - Criteria: Scaling HP, rewards

### 6.3 Currency

- [ ] **SilverManager**
  - Files: `Currency/SilverManager.cs`

- [ ] **Silver vendor**
  - Files: `Mobiles/SilverVendor.cs`

### 6.4 Testing

- [ ] Test daily reset
- [ ] Test bounty completion
- [ ] Test boss mechanics
- [ ] Test silver spending

---

## Phase 7: Tournament System

### 7.1 Core

- [ ] **TournamentManager**
  - Files: `Tournament/TournamentManager.cs`
  - Criteria: Schedule, registration, flow

- [ ] **Bracket generation**
  - Files: `Tournament/TournamentBracket.cs`
  - Criteria: Powers of 2, byes

- [ ] **Match processing**
  - Files: `Tournament/TournamentMatch.cs`
  - Criteria: Win detection, advancement

### 7.2 Infrastructure

- [ ] **Arenas**
  - Files: `Tournament/TournamentArena.cs`
  - Criteria: 8 parallel arenas

- [ ] **Registrar NPC**
  - Files: `Mobiles/TournamentRegistrar.cs`

### 7.3 Rewards

- [ ] **Tournament coins**
  - Files: `Currency/TournamentCoinManager.cs`

- [ ] **Cosmetic vendor**
  - Files: `Mobiles/CosmeticVendor.cs`

### 7.4 Optional

- [ ] **Spectator betting**
  - Files: `Tournament/TournamentBetting.cs`

### 7.5 Testing

- [ ] Test full tournament cycle
- [ ] Test bracket edge cases
- [ ] Test rewards distribution

---

## Phase 8: NPE & Polish

### 8.1 Young Players

- [ ] **YoungPlayerManager**
  - Files: `NPE/YoungPlayerManager.cs`
  - Criteria: 14-day protection

- [ ] **Renounce system**
  - Criteria: Early opt-out

### 8.2 Training

- [ ] **Training islands**
  - Files: `NPE/TrainingIsland.cs`
  - Criteria: 5 themed islands

- [ ] **Starter quests**
  - Files: `NPE/StarterQuests.cs`
  - Criteria: GM skill rewards

- [ ] **Ferry NPC**
  - Files: `Mobiles/FerryNPC.cs`

### 8.3 NPCs

- [ ] **New Player Guide**
  - Files: `Mobiles/NewPlayerGuide.cs`

- [ ] **Town Cryer**
  - Files: `Mobiles/TownCryer.cs`

### 8.4 Glicko

- [ ] **GlickoManager**
  - Files: `Glicko/GlickoManager.cs`

- [ ] **GlickoCalculator**
  - Files: `Glicko/GlickoCalculator.cs`

- [ ] **Leaderboard gump**
  - Files: `Gumps/LeaderboardGump.cs`

### 8.5 Polish

- [ ] Load testing
- [ ] Security audit
- [ ] Bug fixes
- [ ] Documentation review

---

## Quick Reference

### Key Files to Modify (Core ModernUO)

| File | Modifications |
|------|---------------|
| `Initialization.cs` | Add 51alpha init call |
| `Spell.cs` | OnCasterMoving, CheckSequence, GetCastDelay |
| `AOS.cs` | Damage tracking hooks |
| `PlayerMobile.cs` | Death handling |
| `BaseRunicTool.cs` | Remove FC |
| `RandomItemGenerator.cs` | Remove FC |
| `LootPack.cs` | Remove FC |

### New Files to Create

All under `Projects/UOContent/Sphere51a/`:
- Core: 4 files
- Factions: 4 files
- Combat: 4 files
- Siege: 10+ files
- Tournament: 5 files
- Daily: 5 files
- Currency: 3 files
- Glicko: 3 files
- NPE: 4 files
- Gumps: 8+ files
- Items: 10+ files
- Mobiles: 8+ files
- Commands: 5 files
- Telemetry: 2 files

**Total: ~75 new files**
