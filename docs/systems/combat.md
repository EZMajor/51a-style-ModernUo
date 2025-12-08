# Combat & Spells

Sphere-style combat with skill-based mechanics. Key differences from OSI: free movement during casting, partial mana on fizzle, no Faster Casting equipment.

## Casting Flow

```
ModernUO: Cast → Delay Timer → Target → Validation → Execute
51alpha:  Cast → Target (immediate) → Validation → Delay Timer → Execute
```

### Key Change: Target-First
In 51alpha, the target cursor appears **immediately** when you cast. Resources are consumed after you select a target, then the cast delay begins.

- **Cancel during targeting**: No cost (resources not yet consumed)
- **Interrupt after targeting**: 50% mana lost (resources committed)

### Spell States (ModernUO Actual)

```csharp
public enum SpellState
{
    None = 0,       // Not casting
    Casting = 1,    // During cast delay
    Sequencing = 2  // Waiting for target
}
```

Note: Design docs reference 6 states - use actual 3 states.

## Movement

**Rule**: Players can ALWAYS move unless paralyzed.

| Action | Movement |
|--------|----------|
| Casting spell | ✅ Allowed |
| Bandaging | ✅ Allowed |
| Paralyzed | ❌ Blocked |

Implementation:
```csharp
public virtual bool OnCasterMoving(Direction d)
{
    if (Sphere51aConfig.Instance?.Enabled == true)
        return true;  // Always allow
    
    // Original OSI logic below
    if (IsCasting && BlocksMovement)
        return false;
    return true;
}
```

## Spell Interruption

Actions that **CAN** interrupt (cause fizzle):
- Casting another spell
- Toggling War Mode (on/off)
- Applying bandage
- Death

Actions that **CANNOT** interrupt:
- **Taking damage** (51alpha change)
- **Equipping items** (51alpha change)
- Movement (Sphere-style)
- Using objects (except bandages)

### DisturbType (ModernUO Actual)

```csharp
public enum DisturbType
{
    Hurt,         // Took damage
    EquipRequest, // Tried to equip
    UseRequest,   // Tried to use object
    NewCast,      // Started new spell
    Kill          // Caster died
}
```

Note: No `DisturbType.Movement` - handled via `OnCasterMoving()`.

## Mana Consumption (Option C)

**Decision**: Partial mana consumed on fizzle.

| Outcome | Mana Consumed | Default Rate |
|---------|---------------|--------------|
| Success | Full | 100% |
| Fizzle | Partial | 50% (configurable) |

Reagents always consumed at cast start.

```csharp
// In CheckSequence()
if (CheckFizzle())
{
    // Success: full mana
    Caster.Mana -= mana;
    return true;
}
else
{
    // Fizzle: partial mana
    double rate = Config.FizzleManaConsumptionRate; // 0.5 default
    Caster.Mana -= (int)(mana * rate);
    DoFizzle();
    return false;
}
```

### Balance Recommendations

| Rate | Effect | Use Case |
|------|--------|----------|
| 0.00 | Free fizzles | Very casual |
| 0.25 | Light punishment | Casual PvM |
| 0.50 | **Balanced** | **Recommended** |
| 0.75 | Significant | Competitive PvP |
| 1.00 | Full punishment | Hardcore |

## Faster Casting

**Decision**: FC removed from all equipment.

**Rationale**:
- Fixed cast times enable skill-based play
- Removes gear dependency for casters
- Can be re-added after balance testing

### FC Removal Required

| Location | Action |
|----------|--------|
| BaseRunicTool.cs | Remove FC from random attributes |
| RandomItemGenerator.cs | Remove FC from loot pools |
| LootPack.cs | Remove FC from tables |
| Def*.cs (crafting) | Remove FC bonuses |
| Artifacts | Set FC = 0 |

Admin command: `[RemoveAllFC]` strips FC from all existing items.

## Protection Spell

**Decision**: No casting speed penalty.

Original: `fc -= 2` (makes casting slower)
Modified: `fc -= 0` (no effect)

```csharp
if (ProtectionSpell.Registry.ContainsKey(Caster))
    fc -= Config.ProtectionFCPenalty; // Default: 0
```

Keep other Protection effects:
- Physical resist bonus (+15 at GM)
- Magic resist skill bonus (+10 at GM)

## Scroll System

**Benefits**: Scrolls provide advantages over memory casting.

| Benefit | Value | Notes |
|---------|-------|-------|
| Mana reduction | 43% | Scrolls cost ~57% of normal mana |
| Speed bonus | -0.5s | For Circle 3+ spells |
| No reagents | - | Standard UO behavior |

### Cast Time with Scrolls

| Circle | Memory | Scroll |
|--------|--------|--------|
| 1-2 | Standard | Standard |
| 3 | 1.5s | **1.0s** |
| 4 | 1.75s | **1.25s** |
| 5 | 2.0s | **1.5s** |
| 6 | 2.25s | **1.75s** |
| 7 | 2.5s | **2.0s** |
| 8 | 2.75s | **2.25s** |

See `specs/spell-system.md` for full implementation details.

## PvP/PvM Separation

**Core Rule**: PvM advantages do not apply in PvP.

```csharp
public static bool IsPvPContext(Mobile caster, Mobile target)
{
    if (caster is PlayerMobile && target is PlayerMobile)
        return true;
    if (target is BaseCreature bc && bc.ControlMaster is PlayerMobile)
        return true;
    if (caster is BaseCreature bc2 && bc2.ControlMaster is PlayerMobile 
        && target is PlayerMobile)
        return true;
    return false;
}
```

### PvP Context Effects

| System | PvP Effect |
|--------|------------|
| Talisman bonuses | Disabled (5 min timer) |
| Chivalry spells | Unavailable (Sampire) |
| Pet damage | Base only |
| Pet speed | 50% reduction |

## Damage Calculation

```csharp
public int GetSpellDamage(Mobile caster, Mobile target, int baseDamage)
{
    int damage = baseDamage;
    
    // PvM bonuses only outside PvP
    if (!IsPvPContext(caster, target))
    {
        if (caster is PlayerMobile pm)
        {
            damage = BuildManager.ApplyTalismanBonus(damage, pm);
            damage = ApplyEnemyOfOne(damage, pm, target);
        }
    }
    
    // Always apply
    damage = ApplyEvalIntBonus(damage, caster);
    damage = ApplyResistanceReduction(damage, target);
    
    return damage;
}
```

## Pet Combat

| Rule | Value |
|------|-------|
| Damage | Base only (no talisman) |
| PvP Speed | 50% reduction |
| Duration | 5 minutes after PvP |
| Dungeons | Allowed (mounts dismissed) |

## AoE and PvP Transition

If a player runs into an AoE targeted at monsters, both players enter PvP context:

```csharp
public void OnAoEDamage(Mobile caster, Mobile target, int damage)
{
    if (target is PlayerMobile && caster is PlayerMobile)
    {
        TalismanManager.TriggerPvPDisable(caster);
        TalismanManager.TriggerPvPDisable(target);
    }
}
```

## Configuration

```csharp
public class Sphere51aConfig
{
    public bool Enabled { get; set; } = true;
    public bool AllowMovementDuringCast { get; set; } = true;
    
    public double SuccessManaConsumptionRate { get; set; } = 1.0;
    public double FizzleManaConsumptionRate { get; set; } = 0.5;
    
    public int ProtectionFCPenalty { get; set; } = 0;
    
    public TimeSpan TalismanPvPDisableDuration { get; set; } = TimeSpan.FromMinutes(5);
}
```

## Testing Checklist

### Spell Casting
- [ ] Move freely during cast
- [ ] Damage does NOT interrupt (51alpha)
- [ ] Equipping does NOT interrupt (51alpha)
- [ ] War mode toggle DOES interrupt
- [ ] Skill fizzle, 50% mana consumed
- [ ] Success, full mana consumed
- [ ] Protection has no FC effect

### FC Removal
- [ ] Loot drops have no FC
- [ ] Crafted items have no FC
- [ ] `[RemoveAllFC]` works
- [ ] GetCastDelay returns base delay

### Scroll System
- [ ] Scrolls cost ~43% less mana
- [ ] Circle 3+ scrolls cast 0.5s faster
- [ ] Circle 1-2 scrolls have same speed
- [ ] Scroll fizzle costs 50% of reduced mana

### PvP Context
- [ ] Player vs Player triggers PvP
- [ ] Player vs Pet triggers PvP
- [ ] Pet vs Player triggers PvP
- [ ] Talisman disabled for 5 min

## Removed Features

| Feature | Reason |
|---------|--------|
| 50Hz Microtick Engine | ModernUO timer wheel sufficient |
| ML-Predictive Cancellation | Unnecessary complexity |
| Zero-Penalty Fizzle | Contradicts skill-based design |
| 50% Movement Threshold | Use 100% free movement |
