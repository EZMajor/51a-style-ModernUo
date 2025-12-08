# 51alpha Spell System Specification

## Document Metadata
- **Version**: 2.0.0
- **Last Updated**: 2025-12-08
- **Status**: Technical Specification (Ready for Implementation)

---

## Executive Summary

The 51alpha Spell System redesigns ModernUO's spell casting with:

- **Immediate Targeting**: Players select targets upon cast, then wait through delay
- **No Damage Interruption**: Taking damage does NOT interrupt spells
- **Costly Fizzles**: 50% mana consumed on fizzle (configurable)
- **Free Movement**: Players can move freely while casting
- **No Faster Casting**: FC attribute removed from all equipment
- **Enhanced Scrolls**: 43% mana reduction + 0.5s speed bonus

---

## 1. Casting Flow Comparison

### Current ModernUO Flow
```
Player Action → Cast()
    ├── CheckCast() validation
    ├── State = SpellState.Casting
    ├── CastTimer starts (delay)
    ├── [WAIT: Cast delay elapsed]
    └── CastTimer.OnTick()
        ├── State = SpellState.Sequencing
        ├── OnCast() → Target cursor appears
        └── [Player selects target]
            └── CheckSequence()
                ├── ConsumeReagents()
                ├── Mana check/consume
                ├── CheckFizzle()
                └── Execute OR Fizzle
```

### New 51alpha Flow
```
Player Action → Cast()
    ├── CheckCast() validation
    ├── State = SpellState.Casting
    ├── OnCast() → Target cursor appears IMMEDIATELY
    └── [Player selects target]
        └── CheckSequence()
            ├── ConsumeReagents()
            ├── Consume 100% mana (committed)
            ├── CastTimer starts (delay)
            ├── [WAIT: Cast delay elapsed]
            └── CheckFizzle()
                ├── SUCCESS: Execute spell
                └── FIZZLE: 50% mana refunded
```

### Key Differences

| Aspect | ModernUO | 51alpha |
|--------|----------|---------|
| Target selection | After cast delay | Immediately on cast |
| Mana consumption | After fizzle check | Before cast delay |
| Fizzle cost | 0% mana lost | 50% mana lost |
| Movement | Frozen during cast | Free movement |
| Damage interrupt | Yes (DisturbType.Hurt) | No |
| Equipment interrupt | Yes (DisturbType.EquipRequest) | No |

---

## 2. State Machine

### SpellState Enum (Unchanged)
```csharp
public enum SpellState
{
    None = 0,      // Not casting
    Casting = 1,   // Active cast (entire duration in 51alpha)
    Sequencing = 2 // UNUSED in 51alpha
}
```

### 51alpha State Usage
- **SpellState.None**: Default, not casting
- **SpellState.Casting**: From Cast() through execution/interruption
- **SpellState.Sequencing**: Not used (kept for compatibility)

---

## 3. Interruption Rules

### What CAN Interrupt Spells

| Action | DisturbType | Notes |
|--------|-------------|-------|
| Cast another spell | NewCast | Starting new spell cancels current |
| Toggle war mode | Custom | On/off war mode interrupts |
| Use bandages | UseRequest | Medical actions interrupt |
| Death | Kill | Caster dies |

### What CANNOT Interrupt Spells

| Action | ModernUO Behavior | 51alpha Behavior |
|--------|-------------------|------------------|
| Taking damage | Interrupts (Hurt) | **No interrupt** |
| Equipping items | Interrupts (EquipRequest) | **No interrupt** |
| Movement | Interrupts (frozen) | **No interrupt** |
| Using objects | Interrupts (UseRequest) | **No interrupt** (except bandages) |

### Implementation: OnCasterHurt()

```csharp
// ModernUO current implementation
public virtual void OnCasterHurt()
{
    if (!Caster.Player)
        return;
    
    if (IsCasting)
    {
        // Protection spell chance to not interrupt
        var protection = ProtectionSpell.Registry[Caster];
        bool disturb = true;
        
        if (protection is double chance && chance > Utility.RandomDouble() * 100.0)
            disturb = false;
        
        // Casting Focus can prevent
        int focus = SAAbsorptionAttributes.GetValue(Caster, SAAbsorptionAttribute.CastingFocus);
        if (focus > 0 && focus > Utility.Random(100))
            disturb = false;
        
        if (disturb)
            Disturb(DisturbType.Hurt, false, true);
    }
}

// 51alpha modification
public virtual void OnCasterHurt()
{
    // 51alpha: Damage does NOT interrupt spells
    if (Sphere51aConfig.Instance?.Enabled == true)
        return;
    
    // Original logic for non-51alpha mode...
}
```

### Implementation: OnCasterEquiping()

```csharp
// ModernUO current
public virtual bool OnCasterEquiping(Item item)
{
    if (IsCasting)
        Disturb(DisturbType.EquipRequest);
    return true;
}

// 51alpha modification
public virtual bool OnCasterEquiping(Item item)
{
    // 51alpha: Equipping does NOT interrupt spells
    if (Sphere51aConfig.Instance?.Enabled == true)
        return true;
    
    if (IsCasting)
        Disturb(DisturbType.EquipRequest);
    return true;
}
```

---

## 4. Resource Consumption (Option C)

### Timing
1. **Reagents**: Consumed in CheckSequence() after target selection
2. **Mana**: 100% consumed in CheckSequence() before cast delay
3. **Fizzle Refund**: 50% mana returned on fizzle

### Implementation

```csharp
public virtual bool CheckSequence()
{
    var mana = ScaleMana(GetMana());
    
    // Early failure checks
    if (Caster.Deleted || !Caster.Alive || Caster.Spell != this)
    {
        DoFizzle();
        return false;
    }
    
    // Scroll validation
    if (Scroll != null && !(Scroll is Runebook))
    {
        if (Scroll.Amount <= 0 || Scroll.Deleted || Scroll.RootParent != Caster)
        {
            DoFizzle();
            return false;
        }
    }
    
    // Reagent consumption - ALWAYS first
    if (!ConsumeReagents())
    {
        Caster.LocalOverheadMessage(MessageType.Regular, 0x22, 502630);
        return false;
    }
    
    // Mana validation
    if (Caster.Mana < mana)
    {
        Caster.LocalOverheadMessage(MessageType.Regular, 0x22, 502625);
        return false;
    }
    
    // === 51alpha: CONSUME MANA NOW (before delay) ===
    Caster.Mana -= mana;
    
    // Start cast delay timer
    var delay = GetCastDelay();
    new CastDelayTimer(this, delay, mana).Start();
    
    return true; // Validation passed, timer started
}

// New timer that handles fizzle at end
private class CastDelayTimer : Timer
{
    private Spell _spell;
    private int _manaConsumed;
    
    public CastDelayTimer(Spell spell, TimeSpan delay, int mana) : base(delay)
    {
        _spell = spell;
        _manaConsumed = mana;
    }
    
    protected override void OnTick()
    {
        if (_spell.State != SpellState.Casting)
            return; // Was interrupted
        
        if (_spell.CheckFizzle())
        {
            // SUCCESS - execute spell
            _spell.ExecuteSpell();
            
            // Scroll consumption
            if (_spell.Scroll is SpellScroll)
                _spell.Scroll.Consume();
        }
        else
        {
            // FIZZLE - refund 50% mana
            double refundRate = 1.0 - (Sphere51aConfig.Instance?.FizzleManaConsumptionRate ?? 0.5);
            int refund = (int)(_manaConsumed * refundRate);
            _spell.Caster.Mana += refund;
            
            _spell.DoFizzle();
        }
        
        _spell.FinishSequence();
    }
}
```

### Mana Consumption Examples

| Spell | Base Mana | On Success | On Fizzle (50%) |
|-------|-----------|------------|-----------------|
| Magic Arrow | 4 | -4 | -2 |
| Fireball | 9 | -9 | -4 (rounded) |
| Energy Bolt | 20 | -20 | -10 |
| Flamestrike | 40 | -40 | -20 |

---

## 5. Movement During Casting

### 51alpha Behavior
Players can **freely move** while casting. No freezing, no interruption.

### Implementation: OnCasterMoving()

```csharp
public virtual bool OnCasterMoving(Direction d)
{
    // 51alpha: Free movement during cast
    if (Sphere51aConfig.Instance?.Enabled == true)
        return true;
    
    // Original OSI logic
    if (IsCasting && BlocksMovement)
    {
        Caster.SendLocalizedMessage(500111); // You are frozen
        return false;
    }
    return true;
}
```

---

## 6. Faster Casting Removal

### Design Decision
FC attribute is **completely removed** from all equipment in 51alpha.

### Files to Modify

| File | Change |
|------|--------|
| `Items/Tools/BaseRunicTool.cs` | Remove AosAttribute.CastSpeed from generation |
| `Misc/RandomItemGenerator.cs` | Remove FC from attribute pools |
| `Misc/LootPack.cs` | Remove FC from loot tables |
| `Engines/Craft/DefTailoring.cs` | Remove FC from crafting |
| `Engines/Craft/DefTinkering.cs` | Remove FC from crafting |
| `Engines/Craft/DefBlacksmithy.cs` | Remove FC from crafting |
| `Engines/Craft/DefInscription.cs` | Remove FC from crafting |

### GetCastDelay() Modification

```csharp
public virtual TimeSpan GetCastDelay()
{
    // 51alpha: No FC from equipment
    if (Sphere51aConfig.Instance?.Enabled == true)
    {
        // Base delay only, no FC reduction
        return Utility.Max(CastDelayBase, CastDelayMinimum);
    }
    
    // Original ModernUO logic with FC...
}
```

### Cleanup Command

```csharp
[Usage("[RemoveAllFC")]
[Description("Removes Faster Casting from all items in world")]
private static void RemoveAllFC_OnCommand(CommandEventArgs e)
{
    int count = 0;
    
    foreach (Item item in World.Items.Values)
    {
        AosAttributes attrs = null;
        
        if (item is BaseWeapon w) attrs = w.Attributes;
        else if (item is BaseArmor a) attrs = a.Attributes;
        else if (item is BaseJewel j) attrs = j.Attributes;
        else if (item is BaseClothing c) attrs = c.Attributes;
        else if (item is Spellbook s) attrs = s.Attributes;
        
        if (attrs != null && attrs[AosAttribute.CastSpeed] != 0)
        {
            attrs[AosAttribute.CastSpeed] = 0;
            count++;
        }
    }
    
    e.Mobile.SendMessage($"Removed Faster Casting from {count} items.");
}
```

---

## 7. Scroll System

### Benefits
Scrolls provide two advantages over memory casting:

1. **Mana Efficiency**: 43% reduction in mana cost
2. **Speed Bonus**: 0.5 second faster for circle 3+ spells

### Mana Scaling

```csharp
public virtual int ScaleMana(int mana)
{
    double scalar = 1.0;
    
    // Existing LMC calculations...
    
    // 51alpha: Scroll mana reduction (43% cheaper)
    if (Sphere51aConfig.Instance?.Enabled == true && Scroll is SpellScroll)
    {
        scalar /= 1.755; // ~43% reduction
    }
    
    return (int)(mana * scalar);
}
```

### Mana Cost Comparison

| Spell | Memory | Scroll | Savings |
|-------|--------|--------|---------|
| Heal (4) | 4 | 2 | 50% |
| Lightning (11) | 11 | 6 | 45% |
| Energy Bolt (20) | 20 | 11 | 45% |
| Flamestrike (40) | 40 | 23 | 42% |

### Cast Speed Bonus

```csharp
public override TimeSpan GetCastDelay()
{
    var baseDelay = base.GetCastDelay();
    
    // 51alpha: Scroll speed bonus for circle 3+
    if (Sphere51aConfig.Instance?.Enabled == true && 
        Scroll is SpellScroll && 
        Circle >= SpellCircle.Third)
    {
        return TimeSpan.FromSeconds(
            Math.Max(0.25, baseDelay.TotalSeconds - 0.5)
        );
    }
    
    return baseDelay;
}
```

### Cast Time Comparison

| Circle | Memory Base | Scroll Speed |
|--------|-------------|--------------|
| 1st | 1.0s | 1.0s |
| 2nd | 1.25s | 1.25s |
| 3rd | 1.5s | **1.0s** |
| 4th | 1.75s | **1.25s** |
| 5th | 2.0s | **1.5s** |
| 6th | 2.25s | **1.75s** |
| 7th | 2.5s | **2.0s** |
| 8th | 2.75s | **2.25s** |

---

## 8. Target-First Implementation

### Core Change
Target cursor appears **immediately** on cast, before delay starts.

### Modified Cast() Method

```csharp
public virtual bool Cast()
{
    // Pre-cast validations
    if (!CheckCast())
        return false;
    
    // Set state
    State = SpellState.Casting;
    Caster.Spell = this;
    
    // 51alpha: Immediate targeting
    if (Sphere51aConfig.Instance?.Enabled == true)
    {
        // Show target cursor NOW
        OnCast();
        
        // Target timeout (30 seconds)
        if (Caster.Player && Caster.Target != null)
        {
            Caster.Target.BeginTimeout(Caster, TimeSpan.FromSeconds(30));
        }
        
        return true;
    }
    
    // Original: Start delay timer, targeting comes after
    // ... original ModernUO logic ...
}
```

### Flow Visualization

```
51alpha:
[Cast] → [Target Cursor] → [Player Aims] → [Select] → [Validation] → [Delay] → [Execute]
         ↑                                    ↑
         Immediate                            Resources consumed
         
ModernUO:
[Cast] → [Delay Timer] → [OnCast] → [Target Cursor] → [Select] → [Validation] → [Execute]
                                                                   ↑
                                                                   Resources consumed
```

### Why This Matters
- Player can take time aiming without wasting the cast
- Resources only consumed after target confirmed
- Interruption during aiming = no cost
- Interruption after validation = 50% mana lost

---

## 9. Configuration

### Sphere51aConfig.cs

```csharp
namespace Sphere51a.Core;

public class Sphere51aConfig
{
    public static Sphere51aConfig Instance { get; private set; }
    
    // Master toggle
    public bool Enabled { get; set; } = true;
    
    // Movement
    public bool AllowMovementDuringCast { get; set; } = true;
    
    // Mana consumption
    public double FizzleManaConsumptionRate { get; set; } = 0.5; // 50% lost on fizzle
    
    // Scroll bonuses
    public double ScrollManaReduction { get; set; } = 0.43; // 43% cheaper
    public double ScrollSpeedBonus { get; set; } = 0.5; // 0.5s faster
    public int ScrollSpeedMinCircle { get; set; } = 3; // Circle 3+
    
    // Interruption
    public bool DamageInterrupts { get; set; } = false;
    public bool EquipInterrupts { get; set; } = false;
    public bool MovementInterrupts { get; set; } = false;
    public bool WarModeInterrupts { get; set; } = true;
    public bool BandageInterrupts { get; set; } = true;
    
    public static void Initialize()
    {
        Instance = new Sphere51aConfig();
        // Load from config file if exists
    }
}
```

### Config Keys

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `spell.enabled` | bool | true | Enable 51alpha spell system |
| `spell.fizzle_mana_rate` | double | 0.5 | Mana consumed on fizzle |
| `spell.scroll_mana_reduction` | double | 0.43 | Scroll mana discount |
| `spell.scroll_speed_bonus` | double | 0.5 | Scroll cast speed bonus |
| `spell.damage_interrupts` | bool | false | Damage interrupts casting |
| `spell.equip_interrupts` | bool | false | Equipping interrupts |
| `spell.movement_interrupts` | bool | false | Movement interrupts |
| `spell.warmode_interrupts` | bool | true | War mode toggle interrupts |

---

## 10. Testing Matrix

### Targeting Flow

| Test | Action | Expected |
|------|--------|----------|
| Immediate cursor | Cast fireball | Target cursor appears instantly |
| Target timeout | Wait 30s without selecting | Spell cancelled, no cost |
| Target then delay | Select target | Cast delay begins after selection |
| Cancel during aim | Press Escape | Spell cancelled, no cost |

### Interruption

| Test | Action | Expected |
|------|--------|----------|
| Take damage | Get hit while casting | **No interrupt** |
| Equip item | Equip weapon mid-cast | **No interrupt** |
| Move | Walk while casting | **No interrupt** |
| Toggle war mode | Press Tab | **Interrupt, 50% mana** |
| Use bandage | Apply bandage | **Interrupt, 50% mana** |
| Cast new spell | Start second spell | **Interrupt, 50% mana** |

### Mana Consumption

| Test | Setup | Expected |
|------|-------|----------|
| Success | High skill, cast spell | 100% mana consumed |
| Fizzle | Low skill, cast spell | 50% mana consumed |
| Scroll success | Use scroll | 57% of normal mana |
| Scroll fizzle | Use scroll, fizzle | ~28% of normal mana |

### Scroll Speed

| Test | Setup | Expected |
|------|-------|----------|
| Circle 1 scroll | Cast Magic Arrow from scroll | Same speed as memory |
| Circle 3 scroll | Cast Fireball from scroll | 0.5s faster |
| Circle 8 scroll | Cast Energy Vortex from scroll | 0.5s faster |

---

## 11. Implementation Priority

### Phase 1: Core Changes (Week 1)
1. Modify `Cast()` for immediate targeting
2. Modify `OnCasterHurt()` to not interrupt
3. Modify `OnCasterEquiping()` to not interrupt
4. Modify `OnCasterMoving()` for free movement

### Phase 2: Resource Flow (Week 1-2)
1. Modify `CheckSequence()` for early mana consumption
2. Implement `CastDelayTimer` with fizzle refund
3. Add `ScaleMana()` scroll reduction

### Phase 3: Scroll System (Week 2)
1. Implement `GetCastDelay()` scroll speed bonus
2. Test all circles for correct timing

### Phase 4: FC Removal (Week 2-3)
1. Modify loot generation
2. Modify crafting systems
3. Create `[RemoveAllFC]` command
4. Test world item cleanup

### Phase 5: Testing (Week 3)
1. Full testing matrix execution
2. Balance verification
3. Edge case testing

---

## 12. Verification Against ModernUO

### Files to Review Before Implementation

| File | Purpose |
|------|---------|
| `Spells/Base/Spell.cs` | Core spell class |
| `Spells/Base/SpellHelper.cs` | Helper methods |
| `Spells/Magery/MagerySpell.cs` | Magery base class |
| `Server/SpellState.cs` | State enum |
| `Server/Mobile.cs` | OnCasterHurt hooks |

### Key Methods to Verify

1. **Cast()** - Entry point, timer creation
2. **CheckSequence()** - Resource consumption
3. **CheckFizzle()** - Skill check
4. **OnCasterHurt()** - Damage interrupt hook
5. **OnCasterMoving()** - Movement interrupt hook
6. **GetCastDelay()** - Timing calculation
7. **ScaleMana()** - Mana cost calculation

### Warning: State Machine Coupling
The targeting system in ModernUO may have dependencies on `SpellState.Sequencing`. Review all uses of this state before removing it from the flow.

---

## 13. Balance Notes

### Subject to Tuning
- Fizzle mana rate (currently 50%)
- Scroll mana reduction (currently 43%)
- Scroll speed bonus (currently 0.5s)
- Minimum circle for scroll speed (currently 3)

### Design Intent
- Scrolls are valuable consumables worth using
- Fizzles are punishing but not devastating
- Combat focuses on positioning, not spell immunity
- High skill matters (lower fizzle rate)

---

**This specification represents the complete technical requirements for the 51alpha spell system. All values are configurable for balance testing.**
