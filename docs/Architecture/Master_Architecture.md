# 51alpha Master Architecture Document
## Comprehensive Game Server Specification

### Document Metadata
- **Version**: v2.0.0
- **Last Updated**: 2025-01-02
- **Authors**: 51alpha Development Team
- **Platform**: ModernUO v24.0.0+
- **Combat Style**: Sphere51a (Sphere-style PvP)

---

# Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Core Design Principles](#2-core-design-principles)
3. [Combat System](#3-combat-system)
4. [Spell System](#4-spell-system)
5. [Talisman System](#5-talisman-system)
6. [Faction System (VvV)](#6-faction-system-vvv)
7. [Dungeon System](#7-dungeon-system)
8. [Economy Model](#8-economy-model)
9. [Housing & Relic System](#9-housing--relic-system)
10. [New Player Experience](#10-new-player-experience)
11. [Crafting & BOD System](#11-crafting--bod-system)
12. [Glicko-2 Rating System](#12-glicko-2-rating-system)
13. [Social Systems](#13-social-systems)
14. [Tournament System](#14-tournament-system)
15. [Infrastructure & Performance](#15-infrastructure--performance)
16. [Security](#16-security)
17. [NPC Systems](#17-npc-systems)
18. [Configuration Reference](#18-configuration-reference)
19. [Implementation Phases](#19-implementation-phases)

---

# 1. Executive Summary

## 1.1 Project Vision

51alpha is a **guild-based faction PvP** Ultima Online server built on ModernUO with Sphere-style combat mechanics. The server emphasizes:

- **Skill-based PvP** with free movement during casting
- **Guild-centric faction warfare** with sigil capture events
- **Clear PvP/PvM separation** - talisman bonuses disabled in PvP
- **Accessible progression** - combat skills via quick quests, not grinding
- **Quarterly seasons** with meaningful rewards

## 1.2 Key Design Decisions Summary

| Area | Decision | Rationale |
|------|----------|-----------|
| Combat | Sphere-style (free movement) | Skill-based, mobile gameplay |
| Fizzle | Resources consumed | Punishes mistakes, rewards skill |
| Factions | 3 factions, guild-based | Natural 2v1 dynamics, social play |
| Talismans | Disabled in PvP (5 min) | Level playing field in PvP |
| Town Control | Temporary (5 min) | Rewards active play |
| Seasons | Quarterly reset | Meaningful progression + fresh starts |
| New Players | 2-week protection + quest skills | Fast to competence |

## 1.3 Target Metrics

| Metric | Target | Notes |
|--------|--------|-------|
| Max Concurrent Players | 5,000 | Realistic ceiling |
| Average Concurrent | ~1,000 | Expected steady state |
| Time to PvP Ready | 2-3 hours | Via starter quests |
| Microtick Precision | 50Hz (20ms) | Combat timing |
| Server Tick Budget | <10ms @ 35% load | Performance target |

---

# 2. Core Design Principles

## 2.1 PvP/PvM Separation

**Fundamental Rule**: PvM advantages do not apply in PvP combat.

```csharp
public static bool IsPvPContext(Mobile caster, Mobile target)
{
    // Direct player vs player
    if (caster is PlayerMobile && target is PlayerMobile)
        return true;
        
    // Player vs player-controlled creature
    if (target is BaseCreature bc && bc.ControlMaster is PlayerMobile)
        return true;
        
    // Player-controlled creature vs player
    if (caster is BaseCreature bc2 && bc2.ControlMaster is PlayerMobile 
        && target is PlayerMobile)
        return true;
        
    return false;
}
```

**When PvP Context is Detected:**
- Talisman bonuses deactivate (5-minute timer)
- Chivalry spell access removed (Sampire build)
- Pet movement speed reduced 50%
- Base damage only (no PvM multipliers)

## 2.2 Guild-Centric Design

- **Solo players cannot participate in factions**
- Guilds choose faction allegiance (7-day change cooldown)
- All guild members inherit faction
- Encourages social play and coordination

## 2.3 Skill-Based Gameplay

- Fast skill acquisition via quests (not grinding)
- Punishing fizzle mechanics (resource consumption)
- Free movement during casting (Sphere-style)
- Glicko-2 prevents farming low-skill players

---

# 3. Combat System

## 3.1 50Hz Microtick Engine

All combat calculations run at 50Hz (20ms intervals) for precise timing.

```csharp
public class CombatTickEngine
{
    private const int TickRateHz = 50;
    private const int TickIntervalMs = 1000 / TickRateHz; // 20ms
    
    // All high-frequency systems unified at 50Hz
    // - Spell state machine
    // - Movement validation
    // - Combat resolution
    // - Resource gathering monitoring
}
```

## 3.2 Movement

**Rule**: Players can ALWAYS move unless paralyzed.

- Movement during spell casting: **Allowed**
- Movement during bandaging: **Allowed**
- Movement restrictions: **Only paralysis spell/effect**

## 3.3 Combat Interruption

Combat actions that cause spell fizzle:
- Casting another spell while one is in flight
- Toggling War Mode on/off
- Applying a bandage
- Losing Line of Sight when spell should land

**Fizzle Consequence**: Mana consumed, reagents/scroll consumed. No refunds.

## 3.4 Pets in Combat

- Pets use **base damage only** (no talisman bonuses)
- Pet speed reduced 50% during PvP context (5-minute duration)
- Mounts auto-dismiss in dungeons, reappear on exit
- Pets allowed in all dungeons

---

# 4. Spell System

## 4.1 Sphere-Style Casting Flow (Revised)

```
Original OSI:    Cast → Freeze → Target → Resolution
51alpha Flow:     Player Action → Spell.Cast() → SpellTarget created → Immediate Target Selection
                                                                                                  ↓
                                                                                            Target Selected → Spell.Target()
                                                                                                  ↓
                                                                                        CheckSequence() → Validation (no consumption)
                                                                                                  ↓
                                                    [Validation Success: Start Casting & Consume] or [Fail/Target Disappears: Do Nothing]

                                                                                           ↓ (on Success)
                                                                                        SpellState.Casting (interruptible: ANY disturb = FIZZLE; movable)
                                                                                                  ↓
                                                                                        [Cast Delay]
                                                                                                  ↓
                                                                                CastTimer.OnTick() → CheckFizzle() → [Success: Execute] or [Fail: DoFizzle]
```

## 4.2 Spell State Machine

```csharp
public enum SpellState
{
    Idle,           // No spell active
    Targeting,      // Selecting target (pre-cast)
    Initiated,      // Cast command received
    Preparing,      // Spell delay countdown
    Casting,        // Final cast animation
    Completed       // Spell resolved
}
```

## 4.3 Spell Damage Calculation

```csharp
public int GetSpellDamage(Mobile caster, Mobile target, int baseDamage)
{
    int damage = baseDamage;
    
    // PvM bonuses ONLY apply outside PvP context
    if (!IsPvPContext(caster, target))
    {
        if (caster is PlayerMobile pm)
        {
            // Talisman bonus
            damage = BuildManager.ApplyTalismanBonus(damage, pm);
            
            // Honor bonus
            damage = ApplyHonorBonus(damage, pm);
            
            // Enemy of One (Chivalry)
            damage = ApplyEnemyOfOne(damage, pm, target);
        }
    }
    
    // Base damage scaling always applies
    damage = ApplyEvalIntBonus(damage, caster);
    damage = ApplyResistanceReduction(damage, target);
    
    return damage;
}
```

## 4.4 Chivalry Access

**Critical Rule**: Chivalry spells require an active Sampire talisman.

```csharp
public static bool CanCastChivalrySpell(PlayerMobile caster)
{
    var talisman = caster.FindItemOnLayer(Layer.Talisman) as BaseTalisman;
    
    // Must have Sampire talisman equipped
    if (talisman?.TalismanType != TalismanType.Sampire)
        return false;
        
    // Talisman must be active (not PvP disabled)
    if (!talisman.IsActive)
    {
        caster.SendMessage(0x22, "Your talisman is inactive. Chivalry spells are unavailable.");
        return false;
    }
    
    return true;
}
```

## 4.5 AoE and PvP Transition

**Rule**: If a player runs into an AoE targeted at monsters, BOTH players enter PvP context.

```csharp
public void OnAoEDamage(Mobile caster, Mobile target, int damage)
{
    // If AoE hits a player (even unintentionally)
    if (target is PlayerMobile victim && caster is PlayerMobile attacker)
    {
        // Both enter PvP context
        TalismanManager.TriggerPvPDisable(attacker);
        TalismanManager.TriggerPvPDisable(victim);
    }
}
```

---

# 5. Talisman System

## 5.1 Talisman Types

| Type | Playstyle | Key Bonus | Chivalry Access |
|------|-----------|-----------|-----------------|
| Dexer | Melee DPS | +% weapon damage | No |
| Tamer | Pet commands | +% pet damage | No |
| Sampire | Chivalry/Leech | Life leech, Chivalry spells | **Yes** |
| Treasure Hunter | Exploration | +% chest quality | No |

## 5.2 Acquisition

1. **Farm relics** from dungeons (see Relic Drop Rates)
2. **Craft talisman** using relics + crafting skill
3. **Equip** (only one at a time)

## 5.3 Timer Mechanics

- Timer does NOT start until first equip
- Talismans can be traded/sold before first equip
- Once equipped, timer begins countdown
- **Timer PAUSES when unequipped** (may adjust for balance later)

## 5.4 PvP Disable Mechanic

```csharp
public static class TalismanManager
{
    private static readonly TimeSpan PvPDisableDuration = TimeSpan.FromMinutes(5);
    
    public static void TriggerPvPDisable(PlayerMobile player)
    {
        var talisman = player.FindItemOnLayer(Layer.Talisman) as BaseTalisman;
        if (talisman == null)
            return;
            
        talisman.IsActive = false;
        talisman.ReactivationTime = DateTime.UtcNow + PvPDisableDuration;
        
        player.SendMessage(0x22, "Your talisman has been disabled for 5 minutes due to PvP combat!");
        
        // Also reduce pet speed
        if (player.AllFollowers != null)
        {
            foreach (var pet in player.AllFollowers.OfType<BaseCreature>())
            {
                pet.ApplyPvPSpeedDebuff(PvPDisableDuration);
            }
        }
    }
}
```

---

# 6. Faction System (VvV)

## 6.1 Structure

- **3 Factions**: Vampire, Daemon, Goblin
- **Guild-based membership** (solo players cannot participate)
- **7-day cooldown** on faction changes
- **Peaceful Participant Option**: Guild members can opt for peaceful status (no PvP outside sieges)

## 6.2 Siege System

See `VvV_Siege_System.md` for complete specification.

| Setting | Value |
|---------|-------|
| Battle Model | 3-way free-for-all |
| Siege Cities | Jhelom, Skara Brae, Yew, Trinsic |
| Siege Trigger | On-demand (minimum players) |
| Siege Duration | 30 minutes |
| Victory Score | 10,000 points |
| Town Control | Persistent until captured |
| NPC Discount | 10% (15% if all 4 cities) |

### Dual Currency System

| Currency | Source | Use | Tradeable |
|----------|--------|-----|-----------|
| **Faction Points** | All activities | Seasonal rankings | No |
| **Silver** | PvP kills, objectives | Traps, turrets, cosmetics | Yes |

### Peaceful Participant

Guild members can choose **Peaceful** status at guild stone:
- Cannot attack or be attacked by enemy factions outside siege zones
- Auto-flagged when entering active siege
- Still contributes via BODs, bounties, faction quests
- 24-hour cooldown on status changes

## 6.2 Sigil Battles

- **Multiple per day** (frequency configurable at runtime)
- **Capture mechanic**: Stand near sigil for 30-45 seconds
- **Interruption**: Taking damage cancels capture
- **Reward**: Faction points + temporary town control

## 6.3 Town Control

- **Duration**: 5 minutes after sigil capture
- **Benefit**: 10-15% NPC vendor discount for faction members
- **Visual**: Faction banners appear in town
- **Automatic expiry**: Returns to neutral after 5 minutes

## 6.4 Underdog Bonuses

| Standing | Point Bonus | Discount |
|----------|-------------|----------|
| 1st Place | 0% | 10% |
| 2nd Place | +2% | 12% |
| 3rd Place | +5% | 15% |

## 6.5 Daily Faction Content

### Daily Bounties (3 tasks, individual rewards)
- Kill X of [random monster type]
- Gather Y of [random resource]
- Kill Z of [different monster type]
- Same tasks for all players each day
- Reward: Faction points per task completed

### Daily Faction Quest
- Kill mini-boss at one of 4 swamp/desert locations
- **Location is announced** to players (not a search)
- Integrates with Swamp_and_Desert_Event.md content
- Reward: Bonus faction points
- Can be completed once per day per player

### Rare Faction Deco
- 0.01% drop chance from any monster
- Universal faction clothing items
- Only equippable by faction members (guild required)

## 6.6 Seasonal System

- **Season length**: Quarterly (3 months)
- **Monthly**: Leaderboard rankings (cosmetic)
- **Quarterly**: Full reset
  - Faction points reset to 0
  - Glicko ratings soft reset (compress 25% toward 1500)
  - Seasonal rewards distributed
  - Town control unaffected (temporary anyway)

---

# 7. Dungeon System

## 7.1 Dungeon Levels

| Level | Difficulty | Monster Tier | Relic Drops |
|-------|------------|--------------|-------------|
| 1 | Entry | Trivial-Easy | Common only |
| 2 | Easy | Easy-Moderate | Common, Uncommon |
| 3 | Medium | Moderate | All tiers |
| 4 | Hard | Hard | All tiers, better rates |
| 5 | Champion | Champion/Boss | Best rates |

## 7.2 Weekly Rotation

Two dungeons rotate each week:

1. **Bonus Dungeon**: 2x gold, 2x relic drop rates
2. **Safe Dungeon**: Reduced drops, but NO PK/stealing allowed

**Implementation**: Safe dungeon entrance teleports to Trammel copy (inherits ruleset).

All other dungeons: Normal rates, full PvP enabled.

## 7.3 Mount Behavior

- Mounts **auto-dismiss** on dungeon entry
- Mounts **auto-summon** on dungeon exit
- Pets (non-mount) allowed in dungeons

## 7.4 Relic Drop Rates

### Base Rates by Dungeon Level

| Level | Common | Uncommon | Rare | Epic |
|-------|--------|----------|------|------|
| 1 | 1.5% | 0.2% | 0% | 0% |
| 2 | 2.0% | 0.4% | 0.05% | 0% |
| 3 | 2.5% | 0.6% | 0.12% | 0.01% |
| 4 | 3.0% | 1.0% | 0.20% | 0.03% |
| 5 | 4.0% | 1.5% | 0.35% | 0.06% |

### Boss Multipliers

| Boss Type | Multiplier |
|-----------|------------|
| Mini-Boss | 2.0x |
| Dungeon Boss | 3.0x |
| World Boss | 5.0x |
| Event Boss | 4.0x |

### Rotation Bonuses (Bonus Dungeon of the Week)

| Tier | Bonus |
|------|-------|
| Common | 2.0x |
| Uncommon | 2.0x |
| Rare | 1.5x |
| Epic | 1.25x |

---

# 8. Economy Model

## 8.1 Gold Flow Principle

**Target**: Slight deflationary pressure (2-5% monthly gold drain)

## 8.2 Gold Sources (Faucets)

| Source | Gold/Hour | Notes |
|--------|-----------|-------|
| Monster Drops (L1-2) | 5,000-10,000 | Entry content |
| Monster Drops (L3-4) | 15,000-30,000 | Mid-game |
| Monster Drops (L5) | 40,000-80,000 | End-game |
| Treasure Chests | 20,000-100,000 | Per chest |
| BOD Rewards | 10,000-50,000 | Per completion |
| Starter Quest Completion | 5,000 | One-time |

## 8.3 Gold Sinks (Drains)

| Sink | Cost | Type |
|------|------|------|
| NPC Reagents | 5-15g each | Consumable |
| NPC Arrows/Bolts | 2-5g each | Consumable |
| Armor/Weapon Repair | 100-10,000g | Per death |
| House Placement | 50k-3M | One-time |
| Poker House Take | 5% of pot | Gambling |
| Duel Pit House Take | 2% of bet | Gambling |

## 8.4 Key Economic Decisions

| Item | Decision |
|------|----------|
| House Taxes | **None** |
| Poker rake destination | **Destroyed** (gold sink) |
| Duel rake destination | **Destroyed** (gold sink) |
| Relics tradeable | **Yes** |
| Talisman tradeable | **Yes** (before first equip) |

## 8.5 Transfer Systems (Not Faucets/Sinks)

- Player-to-player trading
- Duel winnings (minus 2% rake)
- Poker winnings (minus 5% rake)

---

# 9. Housing & Relic System

## 9.1 House Tiers

| Tier | Relic Requirements | Gold Cost | Time Estimate |
|------|-------------------|-----------|---------------|
| Small | 20 Common, 5 Uncommon | 50,000 | 1 week |
| Medium | 50 Common, 15 Uncommon, 2 Rare | 150,000 | 2-3 weeks |
| Large | 100 Common, 40 Uncommon, 8 Rare, 1 Epic | 400,000 | 1-2 months |
| Villa | 200 Common, 80 Uncommon, 20 Rare, 3 Epic | 800,000 | 2-3 months |
| Keep | 400 Common, 150 Uncommon, 50 Rare, 8 Epic | 1,500,000 | 4-6 months |
| Castle | 800 Common, 300 Uncommon, 100 Rare, 20 Epic | 3,000,000 | 8-12 months |

## 9.2 Relic Properties

- **Tradeable**: Yes
- **Stackable**: Yes (by tier)
- **Freshness timer**: **None** (removed)
- **Loss on death**: Yes (if in backpack in PvP zone)

---

# 10. New Player Experience

## 10.1 Young Player Protection

| Setting | Value |
|---------|-------|
| Duration | 2 weeks (calendar time) |
| PvP Protection | Cannot attack or be attacked |
| Theft Protection | Cannot be stolen from |
| Corpse Protection | Cannot be looted |
| Faction Access | Blocked until renounced |
| Early Termination | Player can renounce anytime |

## 10.2 Starter Quest System

**Location**: Quest Ferry at Britain Docks → Training Islands (Trammel)

| Quest | Island | Target | Kills | Skill Reward |
|-------|--------|--------|-------|--------------|
| 1 | Rabbit Island | Rabbits | 20 | 100 Swordsmanship |
| 2 | Skeleton Isle | Skeletons | 15 | 100 Tactics |
| 3 | Orc Camp | Orcs | 10 | 100 Mace Fighting |
| 4 | Lizardman Lair | Lizardmen | 10 | 100 Fencing |
| 5 | Imp Grotto | Imps | 5 | 90 Magery |

**Completion Bonus**: 5,000 gold

**Anti-Farming**: No gold drops on starter islands, cannot re-enter after completion.

## 10.3 Post-Quest State

After completing all quests, players have:
- 100 Swordsmanship
- 100 Tactics
- 100 Mace Fighting
- 100 Fencing
- 90 Magery

**Still need to train**:
- Magery (final 10 points)
- Resisting Spells
- Evaluating Intelligence
- Meditation
- Healing/Anatomy
- Any crafting skills

---

# 11. Crafting & BOD System

## 11.1 BOD Point Timing

**Decision**: Points credited **immediately** on BOD completion.

## 11.2 BOD Authenticity Verification

```csharp
public class CraftedItem : Item
{
    public Serial CrafterSerial { get; set; }
    public DateTime CraftedAt { get; set; }
    public string CraftingHash { get; set; } // HMAC signature
}

public bool ValidateBODSubmission(CraftedItem item)
{
    // Verify HMAC hash matches
    if (item.CraftingHash != GenerateHash(item))
        return false; // Tampered or spawned
        
    return true;
}
```

## 11.3 Talisman Crafting

**Materials Required**:
- Rare relics from dungeons
- Standard crafting materials
- Crafting skill requirements (TBD per talisman type)

---

# 12. Glicko-2 Rating System

## 12.1 Purpose

**NOT for matchmaking** - Anyone can fight anyone.

**Purpose**: Prevent high-skill players from farming points off new players by weighting kill rewards.

## 12.2 Kill Point Multipliers

| Rating Difference | Point Multiplier |
|-------------------|------------------|
| +400 or more (farming) | 0.25x |
| +200 to +399 | 0.50x |
| +100 to +199 | 0.75x |
| -99 to +99 (fair fight) | 1.00x |
| -199 to -100 | 1.25x |
| -399 to -200 | 1.50x |
| -400 or more (underdog) | 2.00x |

## 12.3 Adapted Parameters (Small Population)

| Parameter | Standard | 51alpha |
|-----------|----------|---------|
| Default Rating | 1500 | 1500 |
| Default RD | 350 | 350 |
| Rating Period | 1 month | 3 days |
| Tau | 0.5 | 0.75 |
| Min RD | 30 | 50 |

## 12.4 Seasonal Reset

- Compress rating 25% toward 1500: `new = 1500 + (old - 1500) * 0.75`
- Increase RD by 50

## 12.5 Future Use: Team Balancing

For future CTF/KotH modes, Glicko ratings will balance teams automatically.

---

# 13. Social Systems

## 13.1 Guilds

- Required for faction participation
- Guild leader chooses faction
- 7-day cooldown on faction changes
- Guild leader departure: Faction persists, new leader can change

## 13.2 Duel Pits

- **Location**: Specific arena locations only
- **Betting**: Gold wagers between combatants
- **House Take**: 2% of bet (destroyed as gold sink)
- **No rating integration**: Casual betting system
- **No matchmaking**: Challenge anyone

## 13.3 Texas Hold'em Poker

- **Location**: Specific taverns only
- **House Take**: 5% of pot (destroyed as gold sink)
- **Social feature**: No faction integration

---

# 14. Tournament System

## 14.1 Overview

Automated 1v1 single-elimination tournaments run on a fixed schedule with spectator betting and unique cosmetic rewards.

## 14.2 Schedule

| Day | Times (EST) | Target Region |
|-----|-------------|---------------|
| Wednesday | 2 PM, 8 PM | EU, NA |
| Saturday | 2 PM, 8 PM | EU, NA |
| Sunday | 2 PM, 8 PM | EU, NA |

**Total**: 6 tournaments per week

## 14.3 Registration

- Opens 15 minutes before tournament
- Closes at tournament start
- Free entry (no fee)
- Minimum 6 players (no maximum)
- Young players cannot participate

## 14.4 Match Rules

| Rule | Setting |
|------|---------|
| Selection | Random (no seeding) |
| Format | Single elimination |
| Duration | No time limit (fight to death) |
| Arenas | 8 parallel (expandable) |
| Byes | Round 1 only (early, not late) |
| Refresh | Players auto-healed between rounds |
| Disconnect | Character stays in-game |

## 14.5 Rewards

| Reward | Description |
|--------|-------------|
| **Trophy Statue** | House deco showing winner, date, participants |
| **100 Tournament Coins** | Cosmetic currency |
| **Title** | "Tournament Champion" for 1 week (toggleable) |

**Kill Points**: Faction points awarded for kills (Glicko weighted), but NO participation rewards.

## 14.6 Tournament Coins

Spent at Tournament NPC for cosmetic transformations:
- **100 coins**: Transform helmet → hat (keeps AR)
- **100 coins**: Transform helmet → mask (keeps AR)
- More options to be added

## 14.7 Spectator Betting

- Player-to-player betting only
- 5% house rake (gold sink)
- Can bet on any pending match
- Cannot bet on own matches
- Cannot bet once match starts

## 14.8 Glicko Integration

| Use | Enabled |
|-----|---------|
| Player selection | **No** (random) |
| Kill point weighting | **Yes** |
| Post-match rating update | **Yes** |
| Future team balancing | **Planned** |

---

# 15. Infrastructure & Performance

## 14.1 Performance Targets

| System | Target | Notes |
|--------|--------|-------|
| Microtick cycle | <10ms @ 35% load | 50Hz engine |
| Database connection | <5ms | Connection pooling |
| Player read ops | <10ms median | Hot path |
| Player write ops | <25ms | Batched |
| Concurrent connections | 5,000+ | PgBouncer |

## 14.2 Database Configuration

```yaml
# PgBouncer (connection pooler)
pool_mode: transaction
max_client_conn: 5000
default_pool_size: 50

# PostgreSQL
max_connections: 200
shared_buffers: 256MB
```

## 14.3 Redis Configuration

- Cache for leaderboards (5-second TTL)
- Graceful degradation on failure
- Background health monitoring

```csharp
public async Task<T> GetOrFallback<T>(string key, Func<Task<T>> fallback)
{
    if (!_redisAvailable)
        return await fallback();
        
    try { /* Redis get */ }
    catch (RedisConnectionException)
    {
        _redisAvailable = false;
        _ = Task.Run(MonitorRedisHealth);
        return await fallback();
    }
}
```

## 14.4 VvV Battle Processing Budget

- Target: 5ms per battle with 100 participants
- Priority 1: Sigil state (always process)
- Priority 2: Point calculations (defer if over budget)
- Priority 3: UI updates (skip if over budget)

---

# 16. Security

## 15.1 Authentication Flow

1. **Discord OAuth** → Launcher
2. **JWT Token** (1hr expiry) → Website/API
3. **Refresh Token** (30 day expiry)
4. **Game API Key** (permanent until revoked)

## 15.2 Anti-Exploit

- Crafting hash validation (HMAC signatures)
- Idempotent gold transfers (prevent double-spend)
- Server-side validation of all actions
- CorrelationId tracking for audit

## 15.3 Gold Transfer Safety

```csharp
public TransferResult TransferGold(Guid transferId, PlayerMobile from, PlayerMobile to, int amount)
{
    // Idempotency check
    if (_processedTransfers.Contains(transferId))
        return TransferResult.AlreadyProcessed;
        
    lock (GetLockObject(from, to))
    {
        // Atomic transfer
        if (from.BankBox.TotalGold < amount)
            return TransferResult.InsufficientFunds;
            
        from.BankBox.ConsumeTotal(typeof(Gold), amount);
        to.BankBox.DropItem(new Gold(amount));
        
        _processedTransfers.Add(transferId);
        return TransferResult.Success;
    }
}
```

---

# 17. NPC Systems

## 16.1 Town Cryer

- **Locations**: Major towns (Britain, Trinsic, Moonglow, Skara Brae, Jhelom)
- **Announcement Range**: 15 tiles
- **Per-Player Cooldown**: 10 minutes (anti-spam)
- **Categories**:
  - Rotating content (weekly dungeons)
  - Faction events (sigil battles)
  - Server events (admin-triggered)

## 16.2 Quest Ferry NPC

- **Location**: Britain Docks (Trammel)
- **Function**: Transport to starter islands
- **Access Control**: Only current quest island accessible

## 16.3 New Player Guide NPC

- **Location**: Britain Bank (Trammel)
- **Function**: Help menu, wiki link, young status management

---

# 18. Configuration Reference

## 17.1 Combat Configuration

```csharp
public static class CombatConfig
{
    public const int TickRateHz = 50;
    public const int TickIntervalMs = 20;
    public static TimeSpan TalismanPvPDisable = TimeSpan.FromMinutes(5);
    public const double PetPvPSpeedReduction = 0.50; // 50% slower
}
```

## 17.2 VvV Configuration

```csharp
public static class VvVConfig
{
    public static TimeSpan TimeBetweenBattles = TimeSpan.FromMinutes(60);
    public static TimeSpan BattleDuration = TimeSpan.FromMinutes(20);
    public static TimeSpan TownControlDuration = TimeSpan.FromMinutes(5);
    public static TimeSpan SigilCaptureTime = TimeSpan.FromSeconds(30);
    public static TimeSpan SigilContestTime = TimeSpan.FromSeconds(45);
    
    public static int PointsForSigilCapture = 500;
    public static int PointsPerKill = 100; // Before Glicko weighting
}
```

## 17.3 Faction Configuration

```csharp
public static class FactionConfig
{
    public static TimeSpan FactionChangeCooldown = TimeSpan.FromDays(7);
    public static TimeSpan SeasonLength = TimeSpan.FromDays(90); // Quarterly
    
    // Underdog bonuses
    public static double FirstPlaceBonus = 0.00;
    public static double SecondPlaceBonus = 0.02;
    public static double ThirdPlaceBonus = 0.05;
}
```

## 17.4 New Player Configuration

```csharp
public static class YoungPlayerConfig
{
    public static TimeSpan YoungDuration = TimeSpan.FromDays(14);
    public static bool CanBeAttackedByPlayers = false;
    public static bool CanAttackPlayers = false;
    public static bool CanJoinFaction = false;
}
```

## 18.5 Tournament Configuration

```csharp
public static class TournamentConfig
{
    // Timing
    public static TimeSpan RegistrationWindow = TimeSpan.FromMinutes(15);
    public static TimeSpan PostFinalWait = TimeSpan.FromSeconds(30);
    
    // Participation
    public static int MinimumPlayers = 6;
    public static int MaximumPlayers = int.MaxValue; // Uncapped
    
    // Arenas
    public static int ArenaCount = 8;
    
    // Rewards
    public static int WinnerCoinReward = 100;
    public static TimeSpan TitleDuration = TimeSpan.FromDays(7);
    
    // Betting
    public static double BettingHouseRake = 0.05; // 5%
    public static int MinimumBet = 1000;
    
    // Schedule (EST)
    public static TimeSpan[] TournamentTimes = { TimeSpan.FromHours(14), TimeSpan.FromHours(20) };
    public static DayOfWeek[] TournamentDays = { DayOfWeek.Wednesday, DayOfWeek.Saturday, DayOfWeek.Sunday };
}
```

---

# 19. Implementation Phases

## Phase 1: Core Combat (Weeks 1-4)
- [ ] 50Hz microtick engine
- [ ] Sphere-style spell FSM
- [ ] Fizzle mechanics (resource consumption)
- [ ] Free movement during casting
- [ ] PvP/PvM context detection

## Phase 2: Faction Foundation (Weeks 5-8)
- [ ] 3-faction structure
- [ ] Guild-faction binding
- [ ] VvV sigil capture (from ModernUO base)
- [ ] Town control (5 min temporary)
- [ ] Basic point scoring

## Phase 3: Talisman System (Weeks 9-12)
- [ ] Talisman types (Dexer, Tamer, Sampire, TH)
- [ ] PvP disable mechanic (5 min)
- [ ] Chivalry gating
- [ ] Relic drop system
- [ ] Talisman crafting

## Phase 4: New Player Experience (Weeks 13-16)
- [ ] Young player protection (2 weeks)
- [ ] Ferry quest system
- [ ] 5 training islands
- [ ] Skill reward system
- [ ] New Player Guide NPC

## Phase 5: Economy & Housing (Weeks 17-20)
- [ ] Relic-based house upgrades
- [ ] Gold sink implementation
- [ ] Duel pit betting
- [ ] Poker tables
- [ ] Economy monitoring

## Phase 6: Tournament System (Weeks 21-24)
- [ ] Tournament arena layout
- [ ] Registration system
- [ ] Bracket generation
- [ ] Match processing
- [ ] Trophy reward item
- [ ] Tournament Coin currency
- [ ] Cosmetic shop NPC
- [ ] Spectator betting
- [ ] Scheduling system

## Phase 7: Polish & Launch (Weeks 25-28)
- [ ] Town Cryer NPC
- [ ] Daily bounties
- [ ] Faction quests
- [ ] Wiki documentation
- [ ] Performance optimization
- [ ] Security audit
- [ ] Load testing

---

# Appendix A: Resolved Conflicts

| ID | Conflict | Resolution |
|----|----------|------------|
| 001 | Talisman/Chivalry | Chivalry requires active Sampire talisman |
| 002 | BOD timing | Points credited immediately |
| 003 | Dungeon/BOD materials | Dungeons rotate bonuses, not access |
| 004 | Tick rates | All systems unified at 50Hz |
| 005 | Relic tiers | Mapping defined in section 7.4 |
| 006 | Duel/Faction | Intentionally separate |
| 007 | Glicko/Season | 3-day rating period, quarterly soft reset |

# Appendix B: Removed Features

| Feature | Reason |
|---------|--------|
| ML spell prediction | Complexity, no training pipeline |
| Interrupt throttle | Natural fizzle mechanics sufficient |
| Relic freshness timer | Unnecessary complexity |
| House taxes | Design decision |
| Puzzle clue system | Deferred to future |

# Appendix C: Open Items (TBD)

| Item | Status |
|------|--------|
| Tournament Stadium layout/coordinates | Design needed |
| Tournament cosmetic options expansion | Future content |
| Additional starter quests | Testing will determine |
| Faction-specific visual themes | Art direction needed |

---

# Appendix D: Comprehensive Documentation Links

## Central Documentation

This appendix serves as a complete index of all Sphere51aFuture project documentation, including implementation guides, system designs, architecture specifications, and supporting materials. Each document is linked with relative paths for easy navigation within the project repository. Documents are organized by category matching the docs folder structure.

### Root Documentation

| Document | Description | Link |
|----------|-------------|------|
| Building Server Guide | Build and deployment instructions | [../building-server.md](../building-server.md) |
| Master Index | Main documentation index | [../index.md](../index.md) |
| Installation Guide | Setup and configuration procedures | [../installation.md](../installation.md) |
| README | Project overview and getting started | [../readme.md](../readme.md) |
| Starting Server | Quick start guide | [../starting-server.md](../starting-server.md) |

### Architecture Documentation

| Document | Description | Link |
|----------|-------------|------|
| Audit Report v2 | Architecture audit and compliance | [Audit_v2.md](Audit_v2.md) |

### Archive Documentation

| Document | Description | Link |
|----------|-------------|------|
| Faction VvV Design | Archived faction and vendor war designs | [../Archive/Faction_VvV_Design.md](../Archive/Faction_VvV_Design.md) |

### Development Documentation

| Document | Description | Link |
|----------|-------------|------|
| Development Workflow | Development processes and guidelines | [../Development/Development_Workflow.md](../Development/Development_Workflow.md) |
| Integration Map | System integration overview | [../Development/Integration_Map.md](../Development/Integration_Map.md) |
| Security Design | Security architecture and implementation | [../Development/Security_Design.md](../Development/Security_Design.md) |

### Implementation Core Documentation

| Document | Description | Link |
|----------|-------------|------|
| Implementation Index | Implementation guide waterfall | [../Implementation/Index.md](../Implementation/Index.md) |
| ModernUO Integration Guide | ModernUO framework integration | [../Implementation/ModernUO_Integration_Guide.md](../Implementation/ModernUO_Integration_Guide.md) |

#### Core Engine Implementation

| Document | Description | Link |
|----------|-------------|------|
| Combat System Design | Core combat mechanics implementation | [../Implementation/Core_Engine/Combat_System_Design.md](../Implementation/Core_Engine/Combat_System_Design.md) |
| Database Persistence Design | Data storage and retrieval systems | [../Implementation/Core_Engine/Database_Persistence_Design.md](../Implementation/Core_Engine/Database_Persistence_Design.md) |
| Spell System Integration Design | Spell framework implementation | [../Implementation/Core_Engine/SpellSystem_Integration_Design.md](../Implementation/Core_Engine/SpellSystem_Integration_Design.md) |

#### Content Special Implementation

| Document | Description | Link |
|----------|-------------|------|
| AFK Resource Gathering Design | Automated resource collection systems | [../Implementation/Content_Special/AFK_Resource_Gathering_Design.md](../Implementation/Content_Special/AFK_Resource_Gathering_Design.md) |
| Graveyard Lich King Event Design | Dynamic dungeon events | [../Implementation/Content_Special/Graveyard_Liche_King_Event_Design.md](../Implementation/Content_Special/Graveyard_Liche_King_Event_Design.md) |
| Rotating Dungeons Design | Dynamic dungeon rotation systems | [../Implementation/Content_Special/Rotating_Dungeons_Design.md](../Implementation/Content_Special/Rotating_Dungeons_Design.md) |
| Rotating Resource Gathering Design | Seasonal material availability | [../Implementation/Content_Special/Rotating_Resource_Gathering_Design.md](../Implementation/Content_Special/Rotating_Resource_Gathering_Design.md) |
| Swamp and Desert Event Design | Seasonal terraforming events | [../Implementation/Content_Special/Swamp_and_Desert_Event_Design.md](../Implementation/Content_Special/Swamp_and_Desert_Event_Design.md) |

#### Economics & Trade Implementation

| Document | Description | Link |
|----------|-------------|------|
| Crafting BOD Design | Bulk order system economics | [../Implementation/Economics_Trade/Crafting_BODs_Design.md](../Implementation/Economics_Trade/Crafting_BODs_Design.md) |
| Duel Pits Design | Gambling and tournament systems | [../Implementation/Economics_Trade/Duel_Pits_Design.md](../Implementation/Economics_Trade/Duel_Pits_Design.md) |
| Glicko Rating Design | Player ranking and matchmaking | [../Implementation/Economics_Trade/Glicko_Rating_Design.md](../Implementation/Economics_Trade/Glicko_Rating_Design.md) |
| Texas Hold'em Design | Poker game mechanics | [../Implementation/Economics_Trade/Texas_Holdem_Design.md](../Implementation/Economics_Trade/Texas_Holdem_Design.md) |

#### Gameplay Systems Implementation

| Document | Description | Link |
|----------|-------------|------|
| Daily Content System | Recurring player activities | [../Implementation/Gameplay_Systems/Daily_Content_System.md](../Implementation/Gameplay_Systems/Daily_Content_System.md) |
| Dungeon Rotation System Design | Weekly dungeon scheduling | [../Implementation/Gameplay_Systems/Dungeon_Rotation_System_Design.md](../Implementation/Gameplay_Systems/Dungeon_Rotation_System_Design.md) |
| House Crafting Design | Player housing upgrades | [../Implementation/Gameplay_Systems/House_Crafting_Design.md](../Implementation/Gameplay_Systems/House_Crafting_Design.md) |
| New Player Experience Design | Onboarding and progression | [../Implementation/Gameplay_Systems/New_Player_Experience_Design.md](../Implementation/Gameplay_Systems/New_Player_Experience_Design.md) |
| PvM Talismans Design | Monster combat enhancements | [../Implementation/Gameplay_Systems/PvM_Talismans_Design.md](../Implementation/Gameplay_Systems/PvM_Talismans_Design.md) |
| Seasonal Ore Quick Reference | Resource availability schedule | [../Implementation/Gameplay_Systems/Seasonal_Ore_quickrefV1.1.md](../Implementation/Gameplay_Systems/Seasonal_Ore_quickrefV1.1.md) |
| Seasonal Rotating Ore Design | Resource rotation mechanics | [../Implementation/Gameplay_Systems/Seasonal_Rotating_OreV1.1.md](../Implementation/Gameplay_Systems/Seasonal_Rotating_OreV1.1.md) |
| Tournament System | Competitive events framework | [../Implementation/Gameplay_Systems/Tournament_System.md](../Implementation/Gameplay_Systems/Tournament_System.md) |
| VvV Integration Design | Faction warfare mechanics | [../Implementation/Gameplay_Systems/VvV_Integration_Design.md](../Implementation/Gameplay_Systems/VvV_Integration_Design.md) |
| VvV Siege System | City control battles | [../Implementation/Gameplay_Systems/VvV_Siege_system.md](../Implementation/Gameplay_Systems/VvV_Siege_system.md) |

#### Interface & Operations Implementation

| Document | Description | Link |
|----------|-------------|------|
| GUMP UI Design | User interface components | [../Implementation/Interface_Operations/Gumps_UI_Design.md](../Implementation/Interface_Operations/Gumps_UI_Design.md) |
| Launcher Design | Client launcher system | [../Implementation/Interface_Operations/Launcher_Design.md](../Implementation/Interface_Operations/Launcher_Design.md) |
| Operations Infrastructure | Administrative systems | [../Implementation/Interface_Operations/Operations_Infrastructure.md](../Implementation/Interface_Operations/Operations_Infrastructure.md) |
| Storage Shelves Design | Item organization systems | [../Implementation/Interface_Operations/Storage_Shelves_Design.md](../Implementation/Interface_Operations/Storage_Shelves_Design.md) |
| Town Crier Design | News and announcement system | [../Implementation/Interface_Operations/Town_Cryer_Design.md](../Implementation/Interface_Operations/Town_Cryer_Design.md) |
| WebAPI Design | External API interfaces | [../Implementation/Interface_Operations/WebAPI_Design.md](../Implementation/Interface_Operations/WebAPI_Design.md) |
| Website Architecture Design | Web platform structure | [../Implementation/Interface_Operations/Website_Architecture_Design.md](../Implementation/Interface_Operations/Website_Architecture_Design.md) |
| Weight Reduction Bags Design | Inventory management | [../Implementation/Interface_Operations/Weight_Reduction_Bags_Design.md](../Implementation/Interface_Operations/Weight_Reduction_Bags_Design.md) |

### Scripting Guide Documentation

| Document | Description | Link |
|----------|-------------|------|
| Serialization Guide | Save/load system scripting | [../scripting-guide/serialization.md](../scripting-guide/serialization.md) |
| Timers Guide | Event scheduling systems | [../scripting-guide/timers.md](../scripting-guide/timers.md) |

---

# Document History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2024-12 | Initial architecture specification |
| 2.0.0 | 2025-01-02 | Comprehensive cohesion review, conflict resolution, professional specifications added |
| 2.1.0 | 2025-01-03 | Added comprehensive documentation links appendix with 40+ implementation and system documents |
