# Seasonal Rotating Ore System Deep Dive

## Document Metadata
- **Version**: v1.1.0
- **Last Updated**: 2025-06-XX
- **Authors**: Sphere51a Development Team
- **Applicable ModernUO Version**: v24.0.0+
- **PR Ready**: No (Conceptual Design)
- **Dependencies**: Mining System, CraftResource System, Town Cryer, Timer Scheduling, Skill Jewelry System

---

## Overview

Dynamic seasonal ore rotation system that changes the available minable ore types every 3 months based on real-world seasons. Each season activates a unique set of 6 tiered ores (plus Iron which is always available), creating economic scarcity, seasonal cosmetic identity, and predictable rarity cycles for crafters and miners.

### Core Design Philosophy

| Principle | Implementation |
|-----------|----------------|
| Seasonal Identity | Each season has a distinct "chase metal" with vibrant, unique hue |
| Predictable Rarity | Players know when specific ores will return |
| Economic Scarcity | Out-of-season ores become tradeable legacy stock |
| PvP Balance | Ore tier affects AR only; weapon elemental bonuses are PvM-only |
| Difficulty in Acquisition | The challenge is obtaining the ore, not the crafting minigame |
| Classic UO Feel | Uses ModernUO dynamic tile seeds with % rarity |

---

## Skill System Integration

### Maximum Effective Skills

| System | Base Cap | With Jewelry | Notes |
|--------|----------|--------------|-------|
| Mining | 100.0 | 120.0 | Affects ore yield + smelting chance |
| Blacksmithing | 100.0 | 120.0 | Affects crafting success chance |

### Skill Jewelry System

**Overview**: Set-based jewelry system that provides skill bonuses enabling access to Tier 6 content. Requires a complete 4-piece set to achieve maximum +20 bonus.

---

#### Set Types

| Set Type | Skills Affected |
|----------|-----------------|
| **Resource Gathering Set** | Mining, Lumberjacking, Fishing, etc. |
| **Crafting Set** | Blacksmithing, Tailoring, Carpentry, Tinkering, Alchemy, etc. |

---

#### Set Pieces (4 Pieces per Set)

| Slot | Bonus | Notes |
|------|-------|-------|
| Necklace | +5 | — |
| Earrings | +5 | — |
| Ring | +5 | — |
| Bracelet | +5 | — |
| **Full Set** | **+20** | All 4 pieces required |

---

#### Decay Mechanics

| Property | Value |
|----------|-------|
| Timer Start | **First equip** (not on creation) |
| Duration | **7 days** from first equip |
| Decay Type | Real-time (continues while offline) |

```csharp
public abstract class BaseSkillJewelry : BaseJewelry
{
    private DateTime? _firstEquipTime;
    private static readonly TimeSpan DecayDuration = TimeSpan.FromDays(7);
    
    [CommandProperty(AccessLevel.GameMaster)]
    public DateTime? FirstEquipTime
    {
        get => _firstEquipTime;
        set => _firstEquipTime = value;
    }
    
    [CommandProperty(AccessLevel.GameMaster)]
    public TimeSpan? TimeRemaining
    {
        get
        {
            if (_firstEquipTime == null)
                return DecayDuration; // Full time until first equip
                
            var elapsed = DateTime.UtcNow - _firstEquipTime.Value;
            var remaining = DecayDuration - elapsed;
            return remaining > TimeSpan.Zero ? remaining : TimeSpan.Zero;
        }
    }
    
    [CommandProperty(AccessLevel.GameMaster)]
    public bool IsExpired => _firstEquipTime != null && TimeRemaining <= TimeSpan.Zero;
    
    public override bool CanEquip(Mobile from)
    {
        if (IsExpired)
        {
            from.SendMessage("This jewelry has expired and no longer functions.");
            return false;
        }
        return base.CanEquip(from);
    }
    
    public override void OnAdded(object parent)
    {
        base.OnAdded(parent);
        
        if (parent is Mobile mobile && _firstEquipTime == null)
        {
            // Start the 7-day timer on first equip
            _firstEquipTime = DateTime.UtcNow;
            mobile.SendMessage("This jewelry will function for 7 days from now.");
        }
    }
}
```

---

#### Crafting Requirements

| Property | Value |
|----------|-------|
| Crafting Skill | GM Tinkering (100.0) |
| Rare Component | Dungeon Relics |
| Relic Sources | Dungeon relics, mini-bosses, bosses |

**Suggested Relic Costs per Piece:**

| Piece | Relic Cost |
|-------|------------|
| Ring | 1 Uncommon Relic |
| Bracelet | 1 Uncommon Relic |
| Earrings | 1 Rare Relic |
| Necklace | 1 Rare Relic |
| **Full Set** | **2 Uncommon + 2 Rare Relics** |

---

#### Implementation Classes

```csharp
// === RESOURCE GATHERING SET ===
public class GatheringRing : BaseSkillJewelry
{
    public override SkillName[] AffectedSkills => new[] 
    { 
        SkillName.Mining, 
        SkillName.Lumberjacking, 
        SkillName.Fishing 
    };
    public override int SkillBonus => 5;
}

public class GatheringBracelet : BaseSkillJewelry { /* Same pattern */ }
public class GatheringEarrings : BaseSkillJewelry { /* Same pattern */ }
public class GatheringNecklace : BaseSkillJewelry { /* Same pattern */ }

// === CRAFTING SET ===
public class CraftingRing : BaseSkillJewelry
{
    public override SkillName[] AffectedSkills => new[] 
    { 
        SkillName.Blacksmith, 
        SkillName.Tailoring, 
        SkillName.Carpentry,
        SkillName.Tinkering,
        SkillName.Alchemy,
        SkillName.Inscribe,
        SkillName.Cooking,
        SkillName.Fletching
    };
    public override int SkillBonus => 5;
}

public class CraftingBracelet : BaseSkillJewelry { /* Same pattern */ }
public class CraftingEarrings : BaseSkillJewelry { /* Same pattern */ }
public class CraftingNecklace : BaseSkillJewelry { /* Same pattern */ }
```

---

#### Set Bonus Calculation

```csharp
public static int GetEffectiveSkillBonus(Mobile from, SkillName skill)
{
    int totalBonus = 0;
    
    // Check each equipped jewelry slot
    var slots = new[] { from.FindItemOnLayer(Layer.Neck), 
                        from.FindItemOnLayer(Layer.Earrings),
                        from.FindItemOnLayer(Layer.Ring),
                        from.FindItemOnLayer(Layer.Bracelet) };
    
    foreach (var item in slots)
    {
        if (item is BaseSkillJewelry jewelry && 
            !jewelry.IsExpired && 
            jewelry.AffectedSkills.Contains(skill))
        {
            totalBonus += jewelry.SkillBonus; // +5 per piece
        }
    }
    
    // Cap at +20
    return Math.Min(totalBonus, 20);
}

public static double GetEffectiveSkill(Mobile from, SkillName skill)
{
    var baseSkill = from.Skills[skill].Value;
    var jewelryBonus = GetEffectiveSkillBonus(from, skill);
    
    // Cap at 120.0
    return Math.Min(baseSkill + jewelryBonus, 120.0);
}
```

---

## Complete Tier Difficulty Scaling

### Design Philosophy

> **"Difficulty is in the acquisition — not the crafting minigame"**

- Tier 6 ore rarity is the real gate
- Smelting creates pressure to gather slightly more ore than exact requirements
- Crafting feels impactful but not frustrating
- Skill jewelry becomes meaningful and desirable

---

### Tier 0 — Iron (Always Available)

**Smelting Difficulty**: Trivial

| Skill Level | Smelting Chance | Notes |
|-------------|-----------------|-------|
| 120.0 | 100% | Guaranteed |
| 110.0 | 100% | Guaranteed |
| 100.0 | 100% | Guaranteed |
| 90.0 | 100% | Guaranteed |
| 80.0 | 100% | Guaranteed |
| 65.0 | 100% | Guaranteed |
| 50.0 | 95% | Minimum practical skill |

**Crafting Difficulty**: Trivial

| Skill Level | Craft Chance | Notes |
|-------------|--------------|-------|
| 120.0 | 100% | Guaranteed |
| 100.0 | 100% | Guaranteed |
| 80.0 | 100% | Guaranteed |
| 65.0 | 98% | Base crafting level |

**Implementation**:
```csharp
SmeltDifficulty = 50.0,  // Base difficulty
CraftDifficulty = 50.0   // Base difficulty
```

---

### Tier 1 — Entry Level (OldCopper, ShadowIron, Agapite, DullCopper)

**Smelting Difficulty**: Very Easy

| Skill Level | Smelting Chance | Notes |
|-------------|-----------------|-------|
| 120.0 | 100% | Guaranteed |
| 110.0 | 100% | Guaranteed |
| 100.0 | 98% | Near guaranteed |
| 90.0 | 94% | Very reliable |
| 80.0 | 88% | Good success rate |
| 65.0 | 78% | Entry level with risk |

**Crafting Difficulty**: Very Easy

| Skill Level | Craft Chance | Notes |
|-------------|--------------|-------|
| 120.0 | 100% | Guaranteed |
| 110.0 | 100% | Guaranteed |
| 100.0 | 95% | Near guaranteed |
| 90.0 | 88% | Very reliable |
| 80.0 | 78% | Good success rate |

**Implementation**:
```csharp
SmeltDifficulty = 65.0,
CraftDifficulty = 60.0
```

**Weapon Elemental Bonuses**: None (Tier too low)

---

### Tier 2 — Slight Color (Silver, Copper, Verite, BlackDiamond)

**Smelting Difficulty**: Easy

| Skill Level | Smelting Chance | Notes |
|-------------|-----------------|-------|
| 120.0 | 100% | Guaranteed |
| 110.0 | 99% | Near guaranteed |
| 100.0 | 95% | Very reliable |
| 90.0 | 88% | Good success |
| 80.0 | 78% | Moderate risk |
| 70.0 | 65% | Higher risk |

**Crafting Difficulty**: Easy

| Skill Level | Craft Chance | Notes |
|-------------|--------------|-------|
| 120.0 | 99% | Near guaranteed |
| 110.0 | 96% | Very reliable |
| 100.0 | 90% | Reliable |
| 90.0 | 80% | Good success |
| 80.0 | 68% | Moderate risk |

**Implementation**:
```csharp
SmeltDifficulty = 75.0,
CraftDifficulty = 70.0
```

**Weapon Elemental Bonuses**: None (Tier too low)

---

### Tier 3 — Mid-Tier (Rose, Bronze, Valorite, BlackRock)

**Smelting Difficulty**: Moderate

| Skill Level | Smelting Chance | Notes |
|-------------|-----------------|-------|
| 120.0 | 99% | Near guaranteed |
| 110.0 | 96% | Very reliable |
| 100.0 | 88% | Reliable |
| 90.0 | 78% | Moderate risk |
| 80.0 | 62% | Significant risk |

**Crafting Difficulty**: Moderate

| Skill Level | Craft Chance | Notes |
|-------------|--------------|-------|
| 120.0 | 97% | Near guaranteed |
| 110.0 | 92% | Very reliable |
| 100.0 | 82% | Reliable |
| 90.0 | 68% | Moderate risk |
| 80.0 | 52% | Significant risk |

**Implementation**:
```csharp
SmeltDifficulty = 85.0,
CraftDifficulty = 80.0
```

**Weapon Elemental Bonuses**: Minor PvM bonuses begin

| Bonus Type | Value | PvP Effect |
|------------|-------|------------|
| WeaponFireDamage | +10 | None |
| WeaponColdDamage | +10 | None |
| WeaponEnergyDamage | +10 | None |

---

### Tier 4 — High-Tier (Gold, BloodRock, Mytheril, Oceanic)

**Smelting Difficulty**: Challenging

| Skill Level | Smelting Chance | Notes |
|-------------|-----------------|-------|
| 120.0 | 98% | Near guaranteed |
| 110.0 | 92% | Very reliable |
| 100.0 | 80% | Good success |
| 90.0 | 65% | Moderate risk |
| 80.0 | 48% | High risk |

**Crafting Difficulty**: Challenging

| Skill Level | Craft Chance | Notes |
|-------------|--------------|-------|
| 120.0 | 94% | Very reliable |
| 110.0 | 85% | Good success |
| 100.0 | 72% | Moderate success |
| 90.0 | 55% | Risky |
| 80.0 | 38% | Very risky |

**Implementation**:
```csharp
SmeltDifficulty = 95.0,
CraftDifficulty = 90.0
```

**Weapon Elemental Bonuses**: Moderate PvM bonuses

| Bonus Type | Value | PvP Effect |
|------------|-------|------------|
| WeaponFireDamage | +20 | None |
| WeaponColdDamage | +20 | None |
| WeaponPoisonDamage | +15 | None |
| WeaponEnergyDamage | +20 | None |
| WeaponLuck | +40 | None |

---

### Tier 5 — Premium (Ice, Aqua, SandRock, DaemonSteel)

**Smelting Difficulty**: Hard

| Skill Level | Smelting Chance | Notes |
|-------------|-----------------|-------|
| 120.0 | 95% | Very reliable |
| 110.0 | 85% | Good success |
| 100.0 | 70% | Moderate success |
| 90.0 | 52% | Risky |
| 80.0 | 32% | Very risky |

**Crafting Difficulty**: Hard

| Skill Level | Craft Chance | Notes |
|-------------|--------------|-------|
| 120.0 | 88% | Good success |
| 110.0 | 76% | Moderate success |
| 100.0 | 60% | Risky |
| 90.0 | 42% | Very risky |
| 80.0 | 25% | Mostly failure |

**Implementation**:
```csharp
SmeltDifficulty = 105.0,
CraftDifficulty = 100.0
```

**Weapon Elemental Bonuses**: Strong PvM bonuses

| Bonus Type | Value | PvP Effect |
|------------|-------|------------|
| WeaponFireDamage | +30 | None |
| WeaponColdDamage | +35 | None |
| WeaponPoisonDamage | +25 | None |
| WeaponEnergyDamage | +30 | None |
| WeaponDurability | +30 | Applies |
| WeaponLowerReq | +90 | Applies |

---

### Tier 6 — Chase Metal (Amethyst, Fire, Dwarven, Reactive)

**Smelting Difficulty**: Very Hard

| Skill Level | Smelting Chance | Notes |
|-------------|-----------------|-------|
| 120.0 | 90% | Expected for high-end players |
| 110.0 | 73% | Competent crafter, some risk |
| 100.0 | 55% | Entry-level, high risk |
| 90.0 | 35% | Difficult, not recommended |
| 80.0 | 18% | Mostly failure |

**Crafting Difficulty**: Very Hard

| Skill Level | Craft Chance | Notes |
|-------------|--------------|-------|
| 120.0 | 80% | High success but not guaranteed |
| 110.0 | 62% | Reasonable but risky |
| 100.0 | 45% | Very difficult |
| 90.0 | 25% | Mostly failure |
| 80.0 | 12% | Near impossible |

**Implementation**:
```csharp
SmeltDifficulty = 115.0,
CraftDifficulty = 110.0
```

**Weapon Elemental Bonuses**: Maximum PvM bonuses

| Bonus Type | Value | PvP Effect |
|------------|-------|------------|
| WeaponFireDamage | +45 | None |
| WeaponColdDamage | +45 | None |
| WeaponPoisonDamage | +40 | None |
| WeaponEnergyDamage | +45 | None |
| WeaponDurability | +50 | Applies |
| WeaponLowerReq | +100 | Applies |

---

## Complete Difficulty Summary Table

### Smelting Success Rates

| Tier | @120 | @110 | @100 | @90 | @80 | SmeltDiff |
|------|------|------|------|-----|-----|-----------|
| 0 (Iron) | 100% | 100% | 100% | 100% | 100% | 50.0 |
| 1 | 100% | 100% | 98% | 94% | 88% | 65.0 |
| 2 | 100% | 99% | 95% | 88% | 78% | 75.0 |
| 3 | 99% | 96% | 88% | 78% | 62% | 85.0 |
| 4 | 98% | 92% | 80% | 65% | 48% | 95.0 |
| 5 | 95% | 85% | 70% | 52% | 32% | 105.0 |
| 6 | 90% | 73% | 55% | 35% | 18% | 115.0 |

### Crafting Success Rates

| Tier | @120 | @110 | @100 | @90 | @80 | CraftDiff |
|------|------|------|------|-----|-----|-----------|
| 0 (Iron) | 100% | 100% | 100% | 100% | 100% | 50.0 |
| 1 | 100% | 100% | 95% | 88% | 78% | 60.0 |
| 2 | 99% | 96% | 90% | 80% | 68% | 70.0 |
| 3 | 97% | 92% | 82% | 68% | 52% | 80.0 |
| 4 | 94% | 85% | 72% | 55% | 38% | 90.0 |
| 5 | 88% | 76% | 60% | 42% | 25% | 100.0 |
| 6 | 80% | 62% | 45% | 25% | 12% | 110.0 |

### Weapon Elemental Bonuses (PvM Only)

| Tier | Fire | Cold | Poison | Energy | Notes |
|------|------|------|--------|--------|-------|
| 0-2 | — | — | — | — | No bonuses |
| 3 | +10 | +10 | — | +10 | Minor |
| 4 | +20 | +20 | +15 | +20 | Moderate |
| 5 | +30 | +35 | +25 | +30 | Strong |
| 6 | +45 | +45 | +40 | +45 | Maximum |

---

## Algorithms and Logic

### Season Detection Algorithm

```csharp
public enum Season
{
    Spring = 0,  // March 1 - May 31
    Summer = 1,  // June 1 - August 31
    Fall = 2,    // September 1 - November 30
    Winter = 3   // December 1 - February 28/29
}

public static Season GetCurrentSeason()
{
    var now = DateTime.UtcNow;
    return now.Month switch
    {
        >= 3 and <= 5 => Season.Spring,
        >= 6 and <= 8 => Season.Summer,
        >= 9 and <= 11 => Season.Fall,
        _ => Season.Winter  // Dec, Jan, Feb
    };
}

public static DateTime GetNextSeasonStart()
{
    var now = DateTime.UtcNow;
    var year = now.Year;
    
    return GetCurrentSeason() switch
    {
        Season.Spring => new DateTime(year, 6, 1, 0, 0, 0, DateTimeKind.Utc),
        Season.Summer => new DateTime(year, 9, 1, 0, 0, 0, DateTimeKind.Utc),
        Season.Fall => new DateTime(year, 12, 1, 0, 0, 0, DateTimeKind.Utc),
        Season.Winter => new DateTime(year + (now.Month == 12 ? 1 : 0), 3, 1, 0, 0, 0, DateTimeKind.Utc),
        _ => throw new InvalidOperationException()
    };
}
```

### Smelting Success Calculation

```csharp
public static double CalculateSmeltChance(Mobile from, OreDefinition ore)
{
    // Get effective skill (base + jewelry bonuses)
    var effectiveSkill = Math.Min(120.0, from.Skills.Mining.Value + GetJewelryBonus(from, SkillName.Mining));
    
    // Calculate success chance based on difficulty curve
    var difficulty = ore.SmeltDifficulty;
    var skillDelta = effectiveSkill - difficulty;
    
    // Base formula: 50% at difficulty, +5% per skill point above
    var chance = 50.0 + (skillDelta * 5.0);
    
    // Clamp to 0-100%
    return Math.Clamp(chance / 100.0, 0.0, 1.0);
}

public static bool AttemptSmelt(Mobile from, OreDefinition ore, int oreCount)
{
    var chance = CalculateSmeltChance(from, ore);
    
    if (Utility.RandomDouble() < chance)
    {
        // Success - create ingots
        var ingots = CreateIngots(ore.Resource, oreCount);
        from.AddToBackpack(ingots);
        from.SendLocalizedMessage(1044158); // You create some ingots.
        return true;
    }
    else
    {
        // Failure - lose half the ore
        var lost = oreCount / 2;
        from.SendMessage($"You fail to smelt the ore and lose {lost} pieces.");
        return false;
    }
}
```

### Crafting Success Calculation

```csharp
public static double CalculateCraftChance(Mobile from, CraftItem item, CraftResource resource)
{
    var ore = GetOreDefinition(resource);
    var effectiveSkill = Math.Min(120.0, from.Skills.Blacksmith.Value + GetJewelryBonus(from, SkillName.Blacksmith));
    
    // Base craft difficulty from item + ore tier bonus
    var baseDifficulty = item.MinSkill;
    var tierDifficulty = ore.CraftDifficulty;
    var totalDifficulty = Math.Max(baseDifficulty, tierDifficulty);
    
    var skillDelta = effectiveSkill - totalDifficulty;
    var chance = 50.0 + (skillDelta * 3.0); // Slightly steeper curve for crafting
    
    return Math.Clamp(chance / 100.0, 0.0, 1.0);
}
```

### PvM Weapon Bonus Application

```csharp
public class SeasonalWeaponBonusHandler
{
    public static int GetEffectiveElementalDamage(Mobile attacker, Mobile defender, DamageType type, BaseWeapon weapon)
    {
        var baseDamage = weapon.GetElementalDamage(type);
        
        // Check if this is PvP
        if (defender is PlayerMobile)
        {
            // PvP: No seasonal ore weapon bonuses apply
            return GetBaseElementalDamage(weapon, type);
        }
        
        // PvM: Full seasonal ore bonuses apply
        var oreBonus = GetSeasonalOreBonus(weapon.Resource, type);
        return baseDamage + oreBonus;
    }
    
    private static int GetSeasonalOreBonus(CraftResource resource, DamageType type)
    {
        var tier = GetOreTier(resource);
        
        return tier switch
        {
            <= 2 => 0,  // Tiers 0-2: No weapon bonuses
            3 => type switch
            {
                DamageType.Fire => 10,
                DamageType.Cold => 10,
                DamageType.Energy => 10,
                _ => 0
            },
            4 => type switch
            {
                DamageType.Fire => 20,
                DamageType.Cold => 20,
                DamageType.Poison => 15,
                DamageType.Energy => 20,
                _ => 0
            },
            5 => type switch
            {
                DamageType.Fire => 30,
                DamageType.Cold => 35,
                DamageType.Poison => 25,
                DamageType.Energy => 30,
                _ => 0
            },
            6 => type switch
            {
                DamageType.Fire => 45,
                DamageType.Cold => 45,
                DamageType.Poison => 40,
                DamageType.Energy => 45,
                _ => 0
            },
            _ => 0
        };
    }
}
```

### Ore Spawn Probability Algorithm

```csharp
public static CraftResource RollOreVein(int miningSkill, Season season)
{
    var activeOres = GetActiveOres(season);
    var totalWeight = activeOres.Sum(o => o.VeinChance);
    var roll = Utility.Random(totalWeight);
    
    var accumulated = 0;
    foreach (var ore in activeOres.OrderByDescending(o => o.VeinChance))
    {
        accumulated += ore.VeinChance;
        if (roll < accumulated && miningSkill >= ore.MinSkill)
            return ore.Resource;
    }
    
    return CraftResource.Iron; // Fallback
}
```

### Vein Chance Distribution

| Tier | Vein Chance | Spawn % | Cumulative |
|------|-------------|---------|------------|
| 0 (Iron) | 496 | 53.4% | 53.4% |
| 1 | 112 | 12.1% | 65.5% |
| 2 | 98 | 10.6% | 76.1% |
| 3 | 84 | 9.0% | 85.1% |
| 4 | 70 | 7.5% | 92.6% |
| 5 | 56 | 6.0% | 98.6% |
| 6 (Chase) | 14 | 1.5% | 100% |

---

## Complete Ore Definitions

### Tier Color Philosophy

| Tier Range | Color Style | Purpose |
|------------|-------------|---------|
| 0-2 | Muted earth tones | Entry-level, accessible |
| 3-4 | Mid saturation metallics | Progression milestone |
| 5 | Premium but tasteful | Rare, valuable |
| 6 | Vibrant, highly saturated | Seasonal chase cosmetic |

---

### 🌱 SPRING ORE SET (March - May)

#### Tier 0 — Iron (Always Active)
```csharp
new OreDefinition
{
    Resource = CraftResource.Iron,
    Tier = 0,
    Name = "Iron",
    Hue = 0x000,
    LocalizedNumber = 1053109,
    IngotType = typeof(IronIngot),
    OreType = typeof(IronOre),
    GraniteType = typeof(IronGranite),
    SmeltDifficulty = 50.0,
    CraftDifficulty = 50.0,
    MinSkill = 0.0,
    MaxSkill = 100.0,
    VeinChance = 496
    // No attribute bonuses
}
```

#### Tier 1 — Old Copper
```csharp
new OreDefinition
{
    Resource = CraftResource.OldCopper,
    Tier = 1,
    EnumValue = 10,
    Name = "Old Copper",
    Hue = 0x0487,
    IngotType = typeof(OldCopperIngot),
    OreType = typeof(OldCopperOre),
    GraniteType = typeof(OldCopperGranite),
    SmeltDifficulty = 65.0,
    CraftDifficulty = 60.0,
    MinSkill = 15.0,
    MaxSkill = 25.0,
    VeinChance = 112,
    Attributes = new OreAttributes
    {
        ArmorPhysicalResist = 6,
        ArmorEnergyResist = 2,
        ArmorDurability = 50,
        ArmorLowerRequirements = 20,
        // No weapon elemental bonuses (Tier 1)
        WeaponDurability = 100,
        WeaponLowerRequirements = 50,
        RunicMinAttributes = 1,
        RunicMaxAttributes = 2,
        RunicMinIntensity = 10,
        RunicMaxIntensity = 35
    }
}
```

#### Tier 2 — Silver
```csharp
new OreDefinition
{
    Resource = CraftResource.Silver,
    Tier = 2,
    EnumValue = 11,
    Name = "Silver",
    Hue = 0x0497,
    IngotType = typeof(SilverIngot),
    OreType = typeof(SilverOre),
    GraniteType = typeof(SilverGranite),
    SmeltDifficulty = 75.0,
    CraftDifficulty = 70.0,
    MinSkill = 30.0,
    MaxSkill = 45.0,
    VeinChance = 98,
    Attributes = new OreAttributes
    {
        ArmorPhysicalResist = 7,
        ArmorFireResist = 3,
        ArmorColdResist = 3,
        ArmorPoisonResist = 3,
        ArmorEnergyResist = 4,
        ArmorDurability = 60,
        ArmorLuck = 30,
        ArmorLowerRequirements = 30,
        // No weapon elemental bonuses (Tier 2)
        WeaponDurability = 80,
        WeaponLowerRequirements = 60,
        RunicMinAttributes = 1,
        RunicMaxAttributes = 3,
        RunicMinIntensity = 20,
        RunicMaxIntensity = 45
    }
}
```

#### Tier 3 — Rose
```csharp
new OreDefinition
{
    Resource = CraftResource.Rose,
    Tier = 3,
    EnumValue = 12,
    Name = "Rose",
    Hue = 0x0B84,
    IngotType = typeof(RoseIngot),
    OreType = typeof(RoseOre),
    GraniteType = typeof(RoseGranite),
    SmeltDifficulty = 85.0,
    CraftDifficulty = 80.0,
    MinSkill = 45.0,
    MaxSkill = 65.0,
    VeinChance = 84,
    Attributes = new OreAttributes
    {
        ArmorPhysicalResist = 8,
        ArmorFireResist = 4,
        ArmorColdResist = 4,
        ArmorPoisonResist = 4,
        ArmorEnergyResist = 5,
        ArmorDurability = 70,
        ArmorLuck = 40,
        ArmorLowerRequirements = 40,
        // Minor weapon elemental bonuses (Tier 3) - PvM ONLY
        WeaponFireDamage = 10,
        WeaponColdDamage = 10,
        WeaponEnergyDamage = 10,
        WeaponDurability = 60,
        WeaponLowerRequirements = 70,
        RunicMinAttributes = 2,
        RunicMaxAttributes = 4,
        RunicMinIntensity = 30,
        RunicMaxIntensity = 55
    }
}
```

#### Tier 4 — Gold
```csharp
new OreDefinition
{
    Resource = CraftResource.Gold,
    Tier = 4,
    EnumValue = 13,
    Name = "Gold",
    Hue = 0x0494,
    LocalizedNumber = 1053104,
    IngotType = typeof(GoldIngot),
    OreType = typeof(GoldOre),
    GraniteType = typeof(GoldGranite),
    SmeltDifficulty = 95.0,
    CraftDifficulty = 90.0,
    MinSkill = 65.0,
    MaxSkill = 85.0,
    VeinChance = 70,
    Attributes = new OreAttributes
    {
        ArmorPhysicalResist = 9,
        ArmorFireResist = 5,
        ArmorEnergyResist = 7,
        ArmorLuck = 40,
        ArmorLowerRequirements = 50,
        // Moderate weapon elemental bonuses (Tier 4) - PvM ONLY
        WeaponFireDamage = 20,
        WeaponColdDamage = 20,
        WeaponPoisonDamage = 15,
        WeaponEnergyDamage = 20,
        WeaponLuck = 40,
        WeaponLowerRequirements = 80,
        RunicMinAttributes = 3,
        RunicMaxAttributes = 4,
        RunicMinIntensity = 35,
        RunicMaxIntensity = 75
    }
}
```

#### Tier 5 — Ice
```csharp
new OreDefinition
{
    Resource = CraftResource.Ice,
    Tier = 5,
    EnumValue = 14,
    Name = "Ice",
    Hue = 0x09D1,
    IngotType = typeof(IceIngot),
    OreType = typeof(IceOre),
    GraniteType = typeof(IceGranite),
    SmeltDifficulty = 105.0,
    CraftDifficulty = 100.0,
    MinSkill = 85.0,
    MaxSkill = 100.0,
    VeinChance = 56,
    Attributes = new OreAttributes
    {
        ArmorPhysicalResist = 10,
        ArmorFireResist = 7,
        ArmorColdResist = 8,
        ArmorPoisonResist = 7,
        ArmorEnergyResist = 8,
        ArmorDurability = 80,
        ArmorLuck = 50,
        ArmorLowerRequirements = 60,
        // Strong weapon elemental bonuses (Tier 5) - PvM ONLY
        WeaponFireDamage = 30,
        WeaponColdDamage = 35,
        WeaponPoisonDamage = 25,
        WeaponEnergyDamage = 30,
        WeaponDurability = 30,
        WeaponLowerRequirements = 90,
        RunicMinAttributes = 4,
        RunicMaxAttributes = 5,
        RunicMinIntensity = 45,
        RunicMaxIntensity = 85
    }
}
```

#### Tier 6 — Amethyst (Spring Chase Metal) 💎
```csharp
new OreDefinition
{
    Resource = CraftResource.Amethyst,
    Tier = 6,
    EnumValue = 15,
    Name = "Amethyst",
    Hue = 0x048B,
    IngotType = typeof(AmethystIngot),
    OreType = typeof(AmethystOre),
    GraniteType = typeof(AmethystGranite),
    SmeltDifficulty = 115.0,
    CraftDifficulty = 110.0,
    MinSkill = 100.0,
    MaxSkill = 105.0,
    VeinChance = 14,
    Attributes = new OreAttributes
    {
        ArmorPhysicalResist = 10,
        ArmorFireResist = 8,
        ArmorColdResist = 10,
        ArmorPoisonResist = 10,
        ArmorEnergyResist = 10,
        ArmorDurability = 100,
        ArmorLuck = 60,
        ArmorLowerRequirements = 70,
        // Maximum weapon elemental bonuses (Tier 6) - PvM ONLY
        WeaponFireDamage = 45,
        WeaponColdDamage = 45,
        WeaponPoisonDamage = 40,
        WeaponEnergyDamage = 45,
        WeaponDurability = 50,
        WeaponLowerRequirements = 100,
        RunicMinAttributes = 5,
        RunicMaxAttributes = 5,
        RunicMinIntensity = 50,
        RunicMaxIntensity = 100
    }
}
```

---

### ☀️ SUMMER ORE SET (June - August)

| Tier | Resource | Enum | Hue | Smelt | Craft | Mine Range | Granite |
|------|----------|------|-----|-------|-------|------------|---------|
| 0 | Iron | - | 0x000 | 50.0 | 50.0 | 0.0 | IronGranite |
| 1 | Shadow Iron | 20 | 0x0770 | 65.0 | 60.0 | 15.0-25.0 | ShadowIronGranite |
| 2 | Copper | 21 | 0x096D | 75.0 | 70.0 | 30.0-45.0 | CopperGranite |
| 3 | Bronze | 22 | 0x0972 | 85.0 | 80.0 | 45.0-65.0 | BronzeGranite |
| 4 | Blood Rock | 23 | 0x04C2 | 95.0 | 90.0 | 65.0-85.0 | BloodRockGranite |
| 5 | Aqua | 24 | 0x079B | 105.0 | 100.0 | 85.0-100.0 | AquaGranite |
| **6** | **Fire** 🔥 | **25** | **0x09D7** | **115.0** | **110.0** | **100.0-105.0** | **FireGranite** |

---

### 🍁 FALL ORE SET (September - November)

| Tier | Resource | Enum | Hue | Smelt | Craft | Mine Range | Granite |
|------|----------|------|-----|-------|-------|------------|---------|
| 0 | Iron | - | 0x000 | 50.0 | 50.0 | 0.0 | IronGranite |
| 1 | Agapite | 30 | 0x0979 | 65.0 | 60.0 | 15.0-25.0 | AgapiteGranite |
| 2 | Verite | 31 | 0x089F | 75.0 | 70.0 | 30.0-45.0 | VeriteGranite |
| 3 | Valorite | 32 | 0x08AB | 85.0 | 80.0 | 45.0-65.0 | ValoriteGranite |
| 4 | Mytheril | 33 | 0x052D | 95.0 | 90.0 | 65.0-85.0 | MytherilGranite |
| 5 | Sand Rock | 34 | 0x09F7 | 105.0 | 100.0 | 85.0-100.0 | SandRockGranite |
| **6** | **Dwarven** ⚒️ | **35** | **0x0794** | **115.0** | **110.0** | **100.0-105.0** | **DwarvenGranite** |

---

### ❄️ WINTER ORE SET (December - February)

| Tier | Resource | Enum | Hue | Smelt | Craft | Mine Range | Granite |
|------|----------|------|-----|-------|-------|------------|---------|
| 0 | Iron | - | 0x000 | 50.0 | 50.0 | 0.0 | IronGranite |
| 1 | Dull Copper | 40 | 0x0973 | 65.0 | 60.0 | 15.0-25.0 | DullCopperGranite |
| 2 | Black Diamond | 41 | 0x09E2 | 75.0 | 70.0 | 30.0-45.0 | BlackDiamondGranite |
| 3 | Black Rock | 42 | 0x047E | 85.0 | 80.0 | 45.0-65.0 | BlackRockGranite |
| 4 | Oceanic | 43 | 0x0B99 | 95.0 | 90.0 | 65.0-85.0 | OceanicGranite |
| 5 | Daemon Steel | 44 | 0x0493 | 105.0 | 100.0 | 85.0-100.0 | DaemonSteelGranite |
| **6** | **Reactive** ⚡ | **45** | **0x07A5** | **115.0** | **110.0** | **100.0-105.0** | **ReactiveGranite** |

---

## Granite System (Housing Integration)

### Purpose
All 25 granite types (Iron + 24 seasonal) are used exclusively for high-level housing construction and upgrades.

### Granite Item Template
```csharp
public abstract class BaseSeasonalGranite : Item
{
    public abstract CraftResource Resource { get; }
    public abstract int GraniteHue { get; }
    
    public BaseSeasonalGranite() : base(0x1779) // Granite item ID
    {
        Stackable = true;
        Hue = GraniteHue;
        Weight = 10.0;
    }
    
    public override void GetProperties(ObjectPropertyList list)
    {
        base.GetProperties(list);
        list.Add($"{GetOreDefinition(Resource).Name} Granite");
    }
}

// Example: Spring Tier 6
public class AmethystGranite : BaseSeasonalGranite
{
    public override CraftResource Resource => CraftResource.Amethyst;
    public override int GraniteHue => 0x048B;
    
    [Constructable]
    public AmethystGranite() : this(1) { }
    
    [Constructable]
    public AmethystGranite(int amount) : base()
    {
        Amount = amount;
    }
}
```

### Complete Granite List

| Season | Tier | Granite Type | Hue |
|--------|------|--------------|-----|
| All | 0 | IronGranite | 0x000 |
| Spring | 1 | OldCopperGranite | 0x0487 |
| Spring | 2 | SilverGranite | 0x0497 |
| Spring | 3 | RoseGranite | 0x0B84 |
| Spring | 4 | GoldGranite | 0x0494 |
| Spring | 5 | IceGranite | 0x09D1 |
| Spring | 6 | AmethystGranite | 0x048B |
| Summer | 1 | ShadowIronGranite | 0x0770 |
| Summer | 2 | CopperGranite | 0x096D |
| Summer | 3 | BronzeGranite | 0x0972 |
| Summer | 4 | BloodRockGranite | 0x04C2 |
| Summer | 5 | AquaGranite | 0x079B |
| Summer | 6 | FireGranite | 0x09D7 |
| Fall | 1 | AgapiteGranite | 0x0979 |
| Fall | 2 | VeriteGranite | 0x089F |
| Fall | 3 | ValoriteGranite | 0x08AB |
| Fall | 4 | MytherilGranite | 0x052D |
| Fall | 5 | SandRockGranite | 0x09F7 |
| Fall | 6 | DwarvenGranite | 0x0794 |
| Winter | 1 | DullCopperGranite | 0x0973 |
| Winter | 2 | BlackDiamondGranite | 0x09E2 |
| Winter | 3 | BlackRockGranite | 0x047E |
| Winter | 4 | OceanicGranite | 0x0B99 |
| Winter | 5 | DaemonSteelGranite | 0x0493 |
| Winter | 6 | ReactiveGranite | 0x07A5 |

---

## Edge Cases

### Legacy Ore Persistence
```csharp
// Ores mined in previous seasons remain in:
// - Player banks ✓
// - Player houses ✓
// - Vendor inventories ✓
// - Crafted gear ✓
// - Cannot be mined until season returns

public bool CanMineOre(CraftResource resource)
{
    var season = GetCurrentSeason();
    var activeOres = GetActiveOres(season);
    return activeOres.Any(o => o.Resource == resource) || resource == CraftResource.Iron;
}
```

### Season Transition Handling
```csharp
// At season boundary (midnight UTC on transition day):
// 1. Update ActiveOres list
// 2. Start 7-day Town Cryer broadcast period
// 3. Log transition for analytics

private bool _isTransitionWeek = false;
private DateTime _transitionEndDate;

public void OnSeasonTransition(Season oldSeason, Season newSeason)
{
    _activeOres = LoadSeasonOres(newSeason);
    
    // Enable Town Cryer broadcasts for 7 days
    _isTransitionWeek = true;
    _transitionEndDate = DateTime.UtcNow.AddDays(7);
    
    TownCryerManager.StartSeasonalBroadcasts(oldSeason, newSeason, _activeOres, TimeSpan.FromDays(7));
    
    AnalyticsManager.LogSeasonTransition(oldSeason, newSeason);
}
```

### Town Cryer Broadcast Rules
- Broadcasts occur during the **first week** after season change only
- No login notification for players who were offline
- Town Cryers at banks announce the seasonal change
- Clickable for detailed ore list gump

### Runic Tool Rotation
```csharp
// Runic tools tied to seasonal ore set
// Existing runic tools remain usable
// New runic creation/loot disabled for non-active ores

public bool CanCreateRunicTool(CraftResource resource)
{
    return CanMineOre(resource); // Same logic as mining
}
```

### BOD Integration
```csharp
// BODs use current season materials only
public static CraftResource GetBODMaterial(BulkOrderType type, int tier)
{
    var season = SeasonManager.Instance.CurrentSeason;
    var activeOres = SeasonManager.Instance.ActiveOres;
    
    // Clamp tier to available ores
    tier = Math.Clamp(tier, 0, 6);
    
    return activeOres[tier].Resource;
}
```

---

## Implementation Details

### File Structure
```
/Server/Engines/Craft/
├── SeasonalOre/
│   ├── SeasonManager.cs           # Season detection & rotation
│   ├── OreDefinitions.cs          # All 25 ore type definitions (Iron + 24 seasonal)
│   ├── SeasonalOreConfig.cs       # Configuration & constants
│   ├── SeasonTransitionHandler.cs # Transition events
│   ├── DifficultyCalculator.cs    # Smelt/Craft success formulas
│   └── PvMWeaponBonusHandler.cs   # PvM-only elemental bonuses
├── DefBlacksmithy.cs              # (Modify) Add seasonal ore support
└── Core/CraftResource.cs          # (Modify) Add new enum values

/Server/Engines/Harvest/
├── Mining.cs                      # (Modify) Hook seasonal ore spawning
└── HarvestResource.cs             # (Modify) Support seasonal resources

/Server/Items/Resources/
├── Ingots/
│   ├── SeasonalIngots.cs          # All 25 ingot item types
│   └── [Per-ore ingot classes]
├── Ore/
│   ├── SeasonalOres.cs            # All 25 ore item types
│   └── [Per-ore classes]
└── Granite/
    ├── SeasonalGranites.cs        # All 25 granite item types
    └── [Per-ore granite classes]

/Server/Items/Equipment/
└── Jewelry/
    ├── BaseSkillJewelry.cs        # Base class with decay logic
    ├── GatheringRing.cs           # +5 Mining/Lumber/Fishing
    ├── GatheringBracelet.cs
    ├── GatheringEarrings.cs
    ├── GatheringNecklace.cs
    ├── CraftingRing.cs            # +5 All crafting skills
    ├── CraftingBracelet.cs
    ├── CraftingEarrings.cs
    └── CraftingNecklace.cs
```

### CraftResource Enum Extension

```csharp
public enum CraftResource
{
    None = 0,
    Iron = 1,
    
    // === SPRING ORE SET (10-15) ===
    OldCopper = 10,
    Silver = 11,
    Rose = 12,
    Gold = 13,
    Ice = 14,
    Amethyst = 15,
    
    // === SUMMER ORE SET (20-25) ===
    ShadowIron = 20,
    Copper = 21,
    Bronze = 22,
    BloodRock = 23,
    Aqua = 24,
    Fire = 25,
    
    // === FALL ORE SET (30-35) ===
    Agapite = 30,
    Verite = 31,
    Valorite = 32,
    Mytheril = 33,
    SandRock = 34,
    Dwarven = 35,
    
    // === WINTER ORE SET (40-45) ===
    DullCopper = 40,
    BlackDiamond = 41,
    BlackRock = 42,
    Oceanic = 43,
    DaemonSteel = 44,
    Reactive = 45,
    
    // Standard resources (unchanged)
    RegularLeather = 101,
    // ... etc
}
```

---

## Testing Plan

### Unit Testing (Automated)

```csharp
[TestClass]
public class SeasonalOreDifficultyTests
{
    [TestMethod]
    [DataRow(120.0, 0.90, 0.80)]  // Tier 6 at max skill
    [DataRow(110.0, 0.73, 0.62)]
    [DataRow(100.0, 0.55, 0.45)]
    [DataRow(90.0, 0.35, 0.25)]
    public void Tier6_DifficultyMatchesSpec(double skill, double expectedSmelt, double expectedCraft)
    {
        var ore = GetOreDefinition(CraftResource.Amethyst);
        
        var smeltChance = CalculateSmeltChance(skill, ore);
        var craftChance = CalculateCraftChance(skill, ore);
        
        Assert.AreEqual(expectedSmelt, smeltChance, 0.02);
        Assert.AreEqual(expectedCraft, craftChance, 0.02);
    }
    
    [TestMethod]
    public void WeaponBonuses_OnlyApplyInPvM()
    {
        var weapon = CreateWeaponWithResource(CraftResource.Amethyst);
        var player = CreateTestPlayer();
        var monster = CreateTestMonster();
        var otherPlayer = CreateTestPlayer();
        
        // PvM: Bonuses apply
        var pvmDamage = GetEffectiveElementalDamage(player, monster, DamageType.Fire, weapon);
        Assert.AreEqual(45, pvmDamage); // Full Tier 6 bonus
        
        // PvP: No bonuses
        var pvpDamage = GetEffectiveElementalDamage(player, otherPlayer, DamageType.Fire, weapon);
        Assert.AreEqual(0, pvpDamage); // Base only, no ore bonus
    }
    
    [TestMethod]
    public void AllGraniteTypes_Exist()
    {
        // Should have 25 granite types (1 Iron + 24 seasonal)
        var graniteTypes = GetAllGraniteTypes();
        Assert.AreEqual(25, graniteTypes.Count);
    }
}
```

### Integration Testing (Live Server)

1. **Difficulty Verification**: Craft 100 items at each tier/skill level, verify success rates
2. **PvM Bonus Testing**: Test elemental damage against monsters vs players
3. **Skill Jewelry**: Verify +20 cap applies correctly
4. **BOD Generation**: Confirm BODs only use current season materials
5. **Granite Drops**: Verify all 25 granite types drop from mining

---

## Performance Benchmarks

| Operation | Target | Notes |
|-----------|--------|-------|
| Season Detection | <1ms | Cached value |
| Ore Roll | <5ms | Weighted random |
| Smelt Calculation | <1ms | Simple formula |
| Craft Calculation | <2ms | Includes item lookup |
| PvM Bonus Check | <1ms | Per-damage calculation |

---

## Admin Commands

```csharp
[Command("SetSeason")]
[Usage("[Season]")]
[Description("Force set the current season (testing only)")]
public static void SetSeason_OnCommand(CommandEventArgs e) { }

[Command("OreInfo")]
[Description("Display current active ores and difficulty values")]
public static void OreInfo_OnCommand(CommandEventArgs e) { }

[Command("TestSmelt")]
[Usage("[Resource] [Skill]")]
[Description("Calculate smelt success chance")]
public static void TestSmelt_OnCommand(CommandEventArgs e) { }

[Command("TestCraft")]
[Usage("[Resource] [Skill]")]
[Description("Calculate craft success chance")]
public static void TestCraft_OnCommand(CommandEventArgs e) { }
```

---

## Development Prerequisites

### ModernUO Base Requirements
- **Minimum Version**: ModernUO v24.0.0 or later
- **Branch**: Use `feature/seasonal_ores` for implementation
- **Dependencies**: Mining system, CraftResource framework, Timer scheduling, Town Cryer

### Pre-Implementation Checklist
- [ ] CraftResource enum extended with all 25 ore types
- [ ] Ingot, Ore, and Granite item classes created (25 each)
- [ ] Mining.cs hook points identified
- [ ] DefBlacksmithy.cs modification plan approved
- [ ] Difficulty calculator formulas implemented
- [ ] PvM-only weapon bonus handler created
- [ ] Skill jewelry system designed
- [ ] Town Cryer integration for 7-day broadcast
- [ ] BOD system updated for seasonal materials
- [ ] Admin command suite implemented

---

## Code Integration Guide

### Step 1: Core Infrastructure (Medium Risk)
1. Create SeasonManager singleton with season detection
2. Implement OreDefinition data classes with difficulty values
3. Set up timer-based season transition

### Step 2: Enum & Item Types (Low Risk)
1. Extend CraftResource enum with 25 values
2. Create ingot item classes (25 types)
3. Create ore item classes (25 types)
4. Create granite item classes (25 types)

### Step 3: Difficulty System (Medium Risk)
1. Implement DifficultyCalculator with smelt/craft formulas
2. Hook into DefBlacksmithy crafting pipeline
3. Add smelting hooks in Mining.cs

### Step 4: PvM Weapon Bonuses (High Risk)
1. Create PvMWeaponBonusHandler
2. Hook into damage calculation pipeline
3. Add defender-type check (PvP vs PvM)
4. Apply tier-based elemental bonuses

### Step 5: Skill Jewelry (Medium Risk)
1. Create base SkillJewelry class with decay timer
2. Implement MiningJewelry and BlacksmithJewelry
3. Add relic crafting requirements
4. Cap effective skill at 120.0

### Step 6: Integration (Low Risk)
1. Update BOD system for seasonal materials
2. Integrate Town Cryer 7-day broadcasts
3. Create admin commands for testing
4. Add analytics logging

---

## Change Log

### v1.1.1 - Jewelry System Clarification
- Changed jewelry to **set-based system** (4 pieces required for +20)
- Two set types: Resource Gathering and Crafting
- Set pieces: Necklace, Earrings, Ring, Bracelet (+5 each)
- Decay timer starts on **first equip** (not creation)
- Duration changed to **7 days** from first equip
- Updated file structure for 8 jewelry item classes

### v1.1.0 - Clarification Update
- Added complete tier-by-tier difficulty scaling (Tiers 0-6)
- Added PvM-only weapon elemental bonuses with tier scaling
- Added skill jewelry system specification (+20 cap, decay, relic crafting)
- Added all 25 unique granite types for housing integration
- Updated Town Cryer to broadcast during first week only
- Removed Gargoyle Pickaxe references (not on this server)
- Added BOD integration with seasonal materials
- Confirmed mining-only rotation (no tailoring/leather)
- Added SmeltDifficulty and CraftDifficulty to all ore definitions

### v1.0.0 - Initial Design
- Complete seasonal ore rotation specification
- All 25 ore types defined with attributes
- Season detection and transition logic
- Mining and crafting integration patterns