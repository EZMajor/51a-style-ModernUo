# Factions & VvV

Three-faction guild warfare with territory control, sieges, and seasonal competition.

## Faction Overview

| ID | Faction | Color | Theme |
|----|---------|-------|-------|
| 1 | Vampire | Red (0x21) | Dark, undead aesthetic |
| 2 | Daemon | Orange (0x30) | Fiery, demonic |
| 3 | Goblin | Green (0x3F) | Savage, tribal |

## Guild-Based Membership

**Core Rule**: Only guilds can join factions. Solo players cannot participate.

| Requirement | Details |
|-------------|---------|
| Membership | Must be in a guild |
| Assignment | Guild leader chooses faction |
| Inheritance | All members inherit faction |
| Cooldown | 7 days between faction changes |

### Member Types

| Type | Can Attack | Can Be Attacked | Earns Points |
|------|------------|-----------------|--------------|
| Combatant | ✅ | ✅ | ✅ |
| Peaceful | ❌ | ❌ | ❌ |

Peaceful members can:
- Craft/supply faction
- Attend events as spectators
- Change to Combatant (24hr cooldown)

Forced Combatant: Attacking another player auto-flags for 24 hours.

## Currency System

### Faction Points (Account-Bound)
- Earned from: Sieges, quests, bounties, tournaments
- Spent on: Faction rewards, seasonal rankings
- Reset: Soft reset each season (keep 10%)

### Silver (Character-Bound)
- Earned from: Bounties, quests, VvV kills
- Spent on: Consumables, cosmetics, gambling
- Tradeable: Yes (in-game trading)

## Territory Control

### Siege Cities

| City | Region |
|------|--------|
| Jhelom | Felucca |
| Skara Brae | Felucca |
| Yew | Felucca |
| Trinsic | Felucca |

### Control Benefits

| Cities Controlled | NPC Discount |
|-------------------|--------------|
| 1-3 | 10% |
| 4 (All) | 15% |

Benefits apply at faction NPCs in controlled cities.

---

## Siege System

### Battle Parameters

| Parameter | Value |
|-----------|-------|
| Duration | 30 minutes |
| Victory Score | 10,000 points |
| Min Players | 10 total (3+ per faction recommended) |
| Global Cooldown | 30 minutes between any siege |
| City Cooldown | 2 hours per city |

### Triggers

Sieges start when:
1. Minimum player threshold met in siege city
2. City cooldown expired
3. Global cooldown expired
4. At least 2 factions represented

### Scoring

| Action | Points | Silver |
|--------|--------|--------|
| Kill (base) | 100 | 10 |
| Kill (Glicko weighted) | 50-200 | 5-20 |
| Sigil Capture | 500 | 50 |
| Altar Control (per 30s) | 25 | - |
| Trap Kill | 50 | 5 |
| Turret Kill | 75 | 7 |

Glicko weight: `0.5 + (victim_rating / caster_rating) * 0.5`, capped 0.5-2.0

### Victory Conditions

1. **Score Limit**: First to 10,000 points
2. **Time Expiration**: Highest score at 30 minutes
3. **Forfeit**: All enemy factions leave

### Objectives

#### Sigil
- Single capture point in city center
- 30 seconds to capture (uncontested)
- 45 seconds if contested (multiple factions)
- 500 points on capture

#### Altars (3 per city)
- Control by proximity (most faction members nearby)
- 25 points per 30 seconds of control
- Enable resurrection at controlled altars

#### Priests
- One per faction at spawn area
- Resurrect dead faction members
- Cannot be killed (non-targetable)

### Defenses

Each faction has a defense budget (5,000 silver default) per siege:

| Defense | Cost | Effect |
|---------|------|--------|
| Alarm Trap | 100 | Reveals hidden enemies |
| Snare Trap | 200 | Roots target 3s |
| Smoke Trap | 300 | AoE blind 5s |
| Mana Drain Trap | 400 | Drains 50 mana |
| Arrow Turret | 500 | Auto-attacks enemies |
| Magic Turret | 750 | AoE damage |

Traps: 10 max per faction, single-use
Turrets: 3 max per faction, 500 HP each

### Admin Controls

```
[siege start <city>]     - Force start siege
[siege stop]             - End active siege
[siege status]           - Show siege state
[siege toggles]          - Enable/disable sigil/altars
```

---

## Seasonal Competition

### Season Structure

| Duration | 90 days (quarterly) |
|----------|---------------------|
| Q1 | Jan 1 - Mar 31 |
| Q2 | Apr 1 - Jun 30 |
| Q3 | Jul 1 - Sep 30 |
| Q4 | Oct 1 - Dec 31 |

### Season End

1. Faction rankings finalized
2. Rewards distributed
3. 10% points carry over
4. Glicko ratings soft reset (RD increases)

### Underdog System

Trailing factions receive point bonuses:

| Position | Bonus |
|----------|-------|
| 1st | 0% |
| 2nd | +2% |
| 3rd | +5% |

Calculated weekly based on total faction points.

---

## Glicko-2 Integration

PvP kills update Glicko ratings:

```csharp
public void ProcessKill(PlayerMobile killer, PlayerMobile victim)
{
    var killerRating = GlickoManager.GetRating(killer);
    var victimRating = GlickoManager.GetRating(victim);
    
    // Calculate point multiplier
    double weight = 0.5 + (victimRating.Rating / killerRating.Rating) * 0.5;
    weight = Math.Clamp(weight, 0.5, 2.0);
    
    int points = (int)(100 * weight);
    
    // Update ratings
    GlickoManager.RecordMatch(killer, victim);
    
    // Award points
    FactionPointManager.Award(killer, points, "Siege Kill");
}
```

### Rating Periods

- Rating period: 3 days
- Matches accumulate during period
- Batch update at period end
- Prevents farming by increasing volatility

---

## Gumps

### Faction Status (`[factionstatus]`)
- Current faction and standing
- Personal stats (kills, deaths, points)
- Guild faction info

### Siege Status (auto-opens during siege)
- Faction scores
- Objective states
- Time remaining
- Personal stats this siege

### Leaderboard
- Top 100 by Faction Points
- Top 100 by Glicko Rating
- Faction totals

---

## Database Tables

| Table | Purpose |
|-------|---------|
| s51a_factions | Faction definitions |
| s51a_seasons | Season tracking |
| s51a_guild_factions | Guild → faction mapping |
| s51a_player_combat_status | Combatant/peaceful |
| s51a_faction_points | Point balances |
| s51a_town_control | City ownership |
| s51a_siege_battles | Battle history |
| s51a_siege_participants | Per-player stats |
| s51a_siege_kills | Kill log |

---

## Configuration

```json
{
  "faction.change_cooldown_days": 7,
  "siege.duration_minutes": 30,
  "siege.victory_score": 10000,
  "siege.global_cooldown_minutes": 30,
  "siege.city_cooldown_minutes": 120,
  "siege.min_total_players": 10,
  "siege.defense_budget_max": 5000,
  "siege.sigil_enabled": true,
  "siege.altars_enabled": true,
  "town.single_city_discount": 0.10,
  "town.all_cities_discount": 0.15
}
```
