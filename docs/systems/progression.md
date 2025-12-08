# Progression

Talisman builds, Glicko-2 ratings, and character advancement.

## Talisman System

### Talisman Types

| Type | Playstyle | Key Bonus | Chivalry |
|------|-----------|-----------|----------|
| Dexer | Melee DPS | +15% weapon damage | ❌ |
| Tamer | Pet commands | +15% pet damage | ❌ |
| Sampire | Chivalry/Leech | Life leech, Chivalry access | ✅ |
| Treasure Hunter | Exploration | +15% chest quality | ❌ |

**Core Rule**: Chivalry spells require active Sampire talisman.

### Acquisition

1. **Farm relics** from dungeons (see Content > Dungeons)
2. **Craft talisman** at Tinkering station (GM skill required)
3. **Equip** in talisman slot (one at a time)

### Crafting Recipe

| Talisman | Common | Uncommon | Rare | Skill |
|----------|--------|----------|------|-------|
| Dexer | 10 | 3 | 1 | GM Blacksmithy |
| Tamer | 10 | 3 | 1 | GM Taming |
| Sampire | 10 | 3 | 1 | GM Chivalry |
| Treasure Hunter | 10 | 3 | 1 | GM Cartography |

All require GM Tinkering + secondary skill.

### PvP Disable Mechanic

**Rule**: Talisman bonuses disabled for 5 minutes after PvP engagement.

```csharp
public class TalismanState
{
    public DateTime? DisabledUntil { get; private set; }
    public TimeSpan ElapsedDisable { get; private set; }
    
    public void TriggerPvPDisable()
    {
        DisabledUntil = DateTime.UtcNow.Add(Config.TalismanPvPDisableDuration);
    }
    
    public void OnUnequip()
    {
        // Pause timer
        if (DisabledUntil.HasValue)
            ElapsedDisable = DisabledUntil.Value - DateTime.UtcNow;
    }
    
    public void OnEquip()
    {
        // Resume timer
        if (ElapsedDisable > TimeSpan.Zero)
            DisabledUntil = DateTime.UtcNow.Add(ElapsedDisable);
    }
    
    public bool IsActive => !DisabledUntil.HasValue || DateTime.UtcNow >= DisabledUntil;
}
```

### Timer Behavior

| Action | Timer Effect |
|--------|--------------|
| Enter PvP | Start 5-min countdown |
| Unequip talisman | Pause countdown |
| Re-equip talisman | Resume countdown |
| Timer expires | Bonuses restore |

**No** expiration while logged out (timer pauses).

### Chivalry Gating

```csharp
public static bool CanCastChivalrySpell(PlayerMobile caster)
{
    var talisman = caster.FindItemOnLayer(Layer.Talisman) as BaseTalisman;
    
    if (talisman?.TalismanType != TalismanType.Sampire)
        return false;
    
    if (!talisman.IsActive)
    {
        caster.SendMessage(0x22, "Your talisman is inactive.");
        return false;
    }
    
    return true;
}
```

---

## Glicko-2 Rating System

### Overview

Glicko-2 provides accurate skill measurement with confidence intervals.

| Parameter | Default | Range |
|-----------|---------|-------|
| Rating (μ) | 1500 | 100-4000 |
| Deviation (RD) | 350 | 30-500 |
| Volatility (σ) | 0.06 | 0.01-0.15 |
| Tau (τ) | 0.5 | System constant |

### Rating Meaning

| Rating | Skill Level |
|--------|-------------|
| < 1200 | Beginner |
| 1200-1400 | Below Average |
| 1400-1600 | Average |
| 1600-1800 | Above Average |
| 1800-2000 | Expert |
| > 2000 | Elite |

### Rating Period

- Duration: 3 days
- Matches accumulate during period
- Batch calculation at period end
- Prevents rapid manipulation

### Match Types

| Type | Weight | Notes |
|------|--------|-------|
| Siege Kill | 1.0 | Standard weight |
| Tournament | 1.5 | Higher stakes |
| Duel | 0.5 | Optional, lower stakes |

### Calculation Flow

```csharp
public void ProcessRatingPeriod()
{
    foreach (var player in PlayersWithMatches)
    {
        var matches = GetMatchesThisPeriod(player);
        if (matches.Count == 0)
        {
            // Increase RD due to inactivity
            player.RD = Math.Min(player.RD * 1.05, 350);
            continue;
        }
        
        // Glicko-2 algorithm
        var newRating = CalculateNewRating(player, matches);
        player.Rating = newRating.Rating;
        player.RD = newRating.RD;
        player.Volatility = newRating.Volatility;
    }
}
```

### Leaderboard Requirements

| Requirement | Value |
|-------------|-------|
| Minimum games | 10 |
| Maximum RD | 100 |
| Displayed | Top 100 |

Players with RD > 100 show as "Provisional".

### Season Reset

At season end:
- Rating preserved
- RD increased to 200 (increases uncertainty)
- Volatility reset to 0.06

---

## Build Manager

Central service managing talisman state and PvP context.

```csharp
public static class BuildManager
{
    private static Dictionary<Serial, TalismanState> _states = new();
    
    public static bool IsTalismanActive(PlayerMobile pm)
    {
        if (!_states.TryGetValue(pm.Serial, out var state))
            return true; // No state = never in PvP
        
        return state.IsActive;
    }
    
    public static void OnPvPEngagement(PlayerMobile pm)
    {
        if (!_states.TryGetValue(pm.Serial, out var state))
        {
            state = new TalismanState();
            _states[pm.Serial] = state;
        }
        
        state.TriggerPvPDisable();
    }
    
    public static int ApplyTalismanBonus(int damage, PlayerMobile pm)
    {
        if (!IsTalismanActive(pm))
            return damage;
        
        var talisman = pm.FindItemOnLayer(Layer.Talisman) as BaseTalisman;
        if (talisman == null)
            return damage;
        
        return (int)(damage * talisman.DamageBonus);
    }
}
```

---

## Skill Acquisition

### Design Philosophy

Skills acquired quickly via quests, not grinding. PvP success depends on player skill, not time investment.

### Starter Quests

New players complete 5 island quests for combat skills:

| Quest | Skill Rewards |
|-------|---------------|
| Combat Basics | GM Swords, Tactics |
| Magic Fundamentals | GM Magery, Eval Int |
| Healing Arts | GM Healing, Anatomy |
| Stealth & Cunning | GM Hiding, Stealth |
| Advanced Combat | GM Resist, Meditation |

Total time: 2-3 hours for PvP-ready character.

### Trade Skills

Trade skills require traditional grinding:
- Crafting (Blacksmithy, Tailoring, etc.)
- Gathering (Mining, Lumberjacking, etc.)
- Support (Tinkering, Inscription, etc.)

Rationale: Trade skills provide economy engagement, not combat advantage.

---

## Database Tables

| Table | Purpose |
|-------|---------|
| s51a_glicko_ratings | Player ratings |
| s51a_glicko_matches | Match history |
| s51a_pvp_stats | Lifetime statistics |
| s51a_starter_quest_progress | NPE tracking |

---

## Configuration

```json
{
  "combat.talisman_pvp_disable_minutes": 5,
  
  "glicko.default_rating": 1500,
  "glicko.default_rd": 350,
  "glicko.default_volatility": 0.06,
  "glicko.tau": 0.5,
  "glicko.rating_period_days": 3,
  
  "talisman.dexer_damage_bonus": 0.15,
  "talisman.tamer_damage_bonus": 0.15,
  "talisman.sampire_leech_bonus": 0.10,
  "talisman.th_quality_bonus": 0.15
}
```
