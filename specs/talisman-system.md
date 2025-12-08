# Talisman System Technical Specification

PvP/PvM separation, timer mechanics, and Chivalry gating implementation.

## Core Design

### Principle

PvM advantages must not apply in PvP combat. Talismans provide significant PvM bonuses that would unbalance PvP if allowed.

### Talisman Types

| Type | Primary Bonus | Secondary | Chivalry |
|------|---------------|-----------|----------|
| Dexer | +15% weapon damage | +5% attack speed | ❌ |
| Tamer | +15% pet damage | +10% pet healing | ❌ |
| Sampire | +10% life leech | Chivalry access | ✅ |
| Treasure Hunter | +15% chest quality | +25% gold find | ❌ |

---

## PvP Context Detection

### IsPvPContext Algorithm

```csharp
public static bool IsPvPContext(Mobile attacker, Mobile defender)
{
    // Direct player vs player
    if (attacker is PlayerMobile && defender is PlayerMobile)
        return true;
    
    // Player attacking player's pet
    if (attacker is PlayerMobile && defender is BaseCreature bc)
    {
        if (bc.ControlMaster is PlayerMobile)
            return true;
    }
    
    // Player's pet attacking player
    if (attacker is BaseCreature bc2 && bc2.ControlMaster is PlayerMobile)
    {
        if (defender is PlayerMobile)
            return true;
    }
    
    // Player's pet attacking player's pet
    if (attacker is BaseCreature bc3 && bc3.ControlMaster is PlayerMobile pm1)
    {
        if (defender is BaseCreature bc4 && bc4.ControlMaster is PlayerMobile pm2)
        {
            if (pm1 != pm2)  // Different owners
                return true;
        }
    }
    
    return false;
}
```

### Context Triggers

| Trigger | Effect |
|---------|--------|
| Attack player | Both enter PvP context |
| Hit by player | Both enter PvP context |
| AoE hits player | Both enter PvP context |
| Pet attacks player | Owner enters PvP context |
| Pet hit by player | Owner enters PvP context |

---

## TalismanState Implementation

### State Machine

```
┌─────────────┐
│   Active    │ ← Default state, bonuses apply
└──────┬──────┘
       │ PvP engagement
       ▼
┌─────────────┐
│  Disabled   │ ← Timer running, no bonuses
└──────┬──────┘
       │ Unequip
       ▼
┌─────────────┐
│   Paused    │ ← Timer frozen
└──────┬──────┘
       │ Re-equip
       ▼
┌─────────────┐
│  Disabled   │ ← Timer resumes
└──────┬──────┘
       │ Timer expires
       ▼
┌─────────────┐
│   Active    │
└─────────────┘
```

### TalismanState Class

```csharp
namespace Sphere51a.Progression;

public class TalismanState
{
    // When disable ends (null = not disabled)
    public DateTime? DisabledUntil { get; private set; }
    
    // Remaining time when paused
    private TimeSpan _remainingTime = TimeSpan.Zero;
    
    // Whether currently equipped
    public bool IsEquipped { get; private set; } = true;
    
    /// <summary>
    /// True if talisman bonuses should apply
    /// </summary>
    public bool IsActive
    {
        get
        {
            // If not disabled, active
            if (!DisabledUntil.HasValue && _remainingTime == TimeSpan.Zero)
                return true;
            
            // If disabled but timer expired, active
            if (DisabledUntil.HasValue && DateTime.UtcNow >= DisabledUntil.Value)
            {
                DisabledUntil = null;
                return true;
            }
            
            // If paused with remaining time, not active
            if (_remainingTime > TimeSpan.Zero)
                return false;
            
            // Disabled and timer not expired
            return false;
        }
    }
    
    /// <summary>
    /// Called when player engages in PvP
    /// </summary>
    public void TriggerPvPDisable()
    {
        var duration = Sphere51aConfig.Instance?.TalismanPvPDisableDuration 
            ?? TimeSpan.FromMinutes(5);
        
        // If already disabled, extend if new disable is longer
        if (DisabledUntil.HasValue)
        {
            var newEnd = DateTime.UtcNow.Add(duration);
            if (newEnd > DisabledUntil.Value)
                DisabledUntil = newEnd;
        }
        else if (_remainingTime > TimeSpan.Zero)
        {
            // Was paused, now extend
            var newRemaining = duration > _remainingTime ? duration : _remainingTime;
            DisabledUntil = DateTime.UtcNow.Add(newRemaining);
            _remainingTime = TimeSpan.Zero;
        }
        else
        {
            // Fresh disable
            DisabledUntil = DateTime.UtcNow.Add(duration);
        }
    }
    
    /// <summary>
    /// Called when talisman unequipped
    /// </summary>
    public void OnUnequip()
    {
        IsEquipped = false;
        
        // Pause timer if running
        if (DisabledUntil.HasValue && DisabledUntil.Value > DateTime.UtcNow)
        {
            _remainingTime = DisabledUntil.Value - DateTime.UtcNow;
            DisabledUntil = null;
        }
    }
    
    /// <summary>
    /// Called when talisman equipped
    /// </summary>
    public void OnEquip()
    {
        IsEquipped = true;
        
        // Resume timer if was paused
        if (_remainingTime > TimeSpan.Zero)
        {
            DisabledUntil = DateTime.UtcNow.Add(_remainingTime);
            _remainingTime = TimeSpan.Zero;
        }
    }
    
    /// <summary>
    /// Get remaining disable time (for display)
    /// </summary>
    public TimeSpan? GetRemainingTime()
    {
        if (_remainingTime > TimeSpan.Zero)
            return _remainingTime;
        
        if (DisabledUntil.HasValue)
        {
            var remaining = DisabledUntil.Value - DateTime.UtcNow;
            return remaining > TimeSpan.Zero ? remaining : null;
        }
        
        return null;
    }
    
    /// <summary>
    /// Serialize for persistence
    /// </summary>
    public void Serialize(GenericWriter writer)
    {
        writer.Write(0); // version
        writer.Write(DisabledUntil.HasValue);
        if (DisabledUntil.HasValue)
            writer.Write(DisabledUntil.Value);
        writer.Write(_remainingTime);
        writer.Write(IsEquipped);
    }
    
    /// <summary>
    /// Deserialize from persistence
    /// </summary>
    public void Deserialize(GenericReader reader)
    {
        int version = reader.ReadInt();
        
        bool hasDisabled = reader.ReadBool();
        if (hasDisabled)
            DisabledUntil = reader.ReadDateTime();
        
        _remainingTime = reader.ReadTimeSpan();
        IsEquipped = reader.ReadBool();
    }
}
```

---

## BuildManager Service

### Singleton Implementation

```csharp
namespace Sphere51a.Progression;

public static class BuildManager
{
    private static Dictionary<Serial, TalismanState> _states = new();
    
    public static void Initialize()
    {
        // Load states from database or save file
        LoadStates();
    }
    
    /// <summary>
    /// Check if player's talisman is active
    /// </summary>
    public static bool IsTalismanActive(PlayerMobile pm)
    {
        if (pm == null)
            return true;
        
        // No talisman equipped = no bonuses anyway
        var talisman = pm.FindItemOnLayer(Layer.Talisman) as BaseTalisman;
        if (talisman == null)
            return true;
        
        // Check state
        if (!_states.TryGetValue(pm.Serial, out var state))
            return true; // No state = never in PvP
        
        return state.IsActive;
    }
    
    /// <summary>
    /// Trigger PvP disable for a player
    /// </summary>
    public static void OnPvPEngagement(PlayerMobile pm)
    {
        if (pm == null)
            return;
        
        if (!_states.TryGetValue(pm.Serial, out var state))
        {
            state = new TalismanState();
            _states[pm.Serial] = state;
        }
        
        state.TriggerPvPDisable();
        
        // Notify player
        var remaining = state.GetRemainingTime();
        if (remaining.HasValue)
        {
            pm.SendMessage(0x22, $"PvP engaged! Talisman bonuses disabled for {remaining.Value.TotalMinutes:F1} minutes.");
        }
    }
    
    /// <summary>
    /// Called when talisman equipped
    /// </summary>
    public static void OnTalismanEquip(PlayerMobile pm)
    {
        if (_states.TryGetValue(pm.Serial, out var state))
        {
            state.OnEquip();
            
            if (!state.IsActive)
            {
                var remaining = state.GetRemainingTime();
                pm.SendMessage(0x22, $"Talisman timer resumed. {remaining?.TotalMinutes:F1} minutes until bonuses restore.");
            }
        }
    }
    
    /// <summary>
    /// Called when talisman unequipped
    /// </summary>
    public static void OnTalismanUnequip(PlayerMobile pm)
    {
        if (_states.TryGetValue(pm.Serial, out var state))
        {
            state.OnUnequip();
            
            if (!state.IsActive)
            {
                pm.SendMessage(0x35, "Talisman timer paused.");
            }
        }
    }
    
    /// <summary>
    /// Apply talisman damage bonus (PvM only)
    /// </summary>
    public static int ApplyTalismanBonus(int baseDamage, PlayerMobile pm, Mobile target)
    {
        // No bonus in PvP context
        if (IsPvPContext(pm, target))
            return baseDamage;
        
        // Check if talisman active
        if (!IsTalismanActive(pm))
            return baseDamage;
        
        var talisman = pm.FindItemOnLayer(Layer.Talisman) as BaseTalisman;
        if (talisman == null)
            return baseDamage;
        
        // Apply type-specific bonus
        double bonus = talisman.TalismanType switch
        {
            TalismanType.Dexer => 0.15,
            TalismanType.Tamer => 0.15,
            TalismanType.Sampire => 0.10,
            _ => 0.0
        };
        
        return (int)(baseDamage * (1.0 + bonus));
    }
    
    /// <summary>
    /// Check PvP context
    /// </summary>
    public static bool IsPvPContext(Mobile attacker, Mobile defender)
    {
        if (attacker is PlayerMobile && defender is PlayerMobile)
            return true;
        
        if (defender is BaseCreature bc && bc.ControlMaster is PlayerMobile)
            return true;
        
        if (attacker is BaseCreature bc2 && bc2.ControlMaster is PlayerMobile 
            && defender is PlayerMobile)
            return true;
        
        return false;
    }
    
    private static void LoadStates()
    {
        // Load from database
        // Implementation depends on persistence strategy
    }
    
    public static void SaveStates()
    {
        // Save to database
        // Called on world save
    }
}
```

---

## Chivalry Gating

### Implementation

```csharp
// In Spells/Chivalry/ChivalrySpell.cs (or base class)

public abstract class ChivalrySpell : Spell
{
    public override bool CheckCast()
    {
        if (!CanCastChivalry(Caster))
            return false;
        
        return base.CheckCast();
    }
    
    public static bool CanCastChivalry(Mobile caster)
    {
        if (caster is not PlayerMobile pm)
            return true; // NPCs can cast
        
        // Must have Sampire talisman equipped
        var talisman = pm.FindItemOnLayer(Layer.Talisman) as BaseTalisman;
        
        if (talisman == null || talisman.TalismanType != TalismanType.Sampire)
        {
            pm.SendMessage(0x22, "You must have a Sampire talisman equipped to cast Chivalry spells.");
            return false;
        }
        
        // Talisman must be active (not PvP disabled)
        if (!BuildManager.IsTalismanActive(pm))
        {
            pm.SendMessage(0x22, "Your talisman is inactive. Chivalry spells are unavailable.");
            return false;
        }
        
        return true;
    }
}
```

### Affected Spells

All Chivalry spells require active Sampire talisman:
- Cleanse by Fire
- Close Wounds
- Consecrate Weapon
- Dispel Evil
- Divine Fury
- Enemy of One
- Holy Light
- Noble Sacrifice
- Remove Curse
- Sacred Journey

---

## BaseTalisman Modifications

### OnEquip Hook

```csharp
// In Items/Equipment/Talismans/BaseTalisman.cs

public override bool OnEquip(Mobile from)
{
    if (!base.OnEquip(from))
        return false;
    
    if (from is PlayerMobile pm)
    {
        BuildManager.OnTalismanEquip(pm);
    }
    
    return true;
}
```

### OnRemoved Hook

```csharp
public override void OnRemoved(object parent)
{
    base.OnRemoved(parent);
    
    if (parent is PlayerMobile pm)
    {
        BuildManager.OnTalismanUnequip(pm);
    }
}
```

---

## Damage Calculation Integration

### Hook Point

```csharp
// In Misc/AOS.cs or damage calculation method

public static int GetNewAosDamage(Mobile attacker, Mobile defender, int baseDamage, ...)
{
    int damage = baseDamage;
    
    // ... existing damage calculations ...
    
    // 51alpha: Apply talisman bonus (PvM only)
    if (attacker is PlayerMobile pm)
    {
        damage = BuildManager.ApplyTalismanBonus(damage, pm, defender);
    }
    
    // ... rest of damage calculations ...
    
    return damage;
}
```

### PvP Engagement Hook

```csharp
// When damage is dealt between players
public static void OnDamageDealt(Mobile attacker, Mobile defender, int damage)
{
    if (BuildManager.IsPvPContext(attacker, defender))
    {
        // Trigger disable for both
        if (attacker is PlayerMobile pm1)
            BuildManager.OnPvPEngagement(pm1);
        
        if (defender is PlayerMobile pm2)
            BuildManager.OnPvPEngagement(pm2);
        
        // Also trigger for pet owners
        if (attacker is BaseCreature bc1 && bc1.ControlMaster is PlayerMobile owner1)
            BuildManager.OnPvPEngagement(owner1);
        
        if (defender is BaseCreature bc2 && bc2.ControlMaster is PlayerMobile owner2)
            BuildManager.OnPvPEngagement(owner2);
    }
}
```

---

## Testing Matrix

### Timer Behavior

| Test | Initial State | Action | Expected |
|------|---------------|--------|----------|
| Fresh PvP | Active | Attack player | 5 min disable |
| Re-engage | 3 min left | Attack player | 5 min (extended) |
| Unequip | 3 min left | Unequip | Timer paused |
| Re-equip | 3 min paused | Equip | Timer resumes |
| Expire | 0 min left | Wait | Bonuses restore |

### Bonus Application

| Test | Context | Talisman State | Expected |
|------|---------|----------------|----------|
| PvM hit | Player → Monster | Active | +15% damage |
| PvM hit | Player → Monster | Disabled | Base damage |
| PvP hit | Player → Player | Active | Base damage |
| PvP hit | Player → Player | Disabled | Base damage |
| Pet PvM | Pet → Monster | Owner Active | +15% damage |
| Pet PvP | Pet → Player | Any | Base damage |

### Chivalry Gating

| Test | Talisman | State | Expected |
|------|----------|-------|----------|
| Cast | None | - | Blocked + message |
| Cast | Dexer | Active | Blocked + message |
| Cast | Sampire | Active | Allowed |
| Cast | Sampire | Disabled | Blocked + message |

---

## Configuration

```csharp
// Sphere51aConfig.cs additions

public TimeSpan TalismanPvPDisableDuration { get; set; } = TimeSpan.FromMinutes(5);
public double DexerDamageBonus { get; set; } = 0.15;
public double TamerDamageBonus { get; set; } = 0.15;
public double SampireLeechBonus { get; set; } = 0.10;
public double THQualityBonus { get; set; } = 0.15;
```

### Config Keys

| Key | Type | Default |
|-----|------|---------|
| `combat.talisman_pvp_disable_minutes` | int | 5 |
| `talisman.dexer_damage_bonus` | decimal | 0.15 |
| `talisman.tamer_damage_bonus` | decimal | 0.15 |
| `talisman.sampire_leech_bonus` | decimal | 0.10 |
| `talisman.th_quality_bonus` | decimal | 0.15 |

---

## Design Decision: Elapsed Time Tracking

### Decision

Timer tracks **elapsed time**, not real-world time. Timer pauses when talisman unequipped.

### Rationale

1. **Fairness**: Player shouldn't lose disable time while talisman not equipped
2. **Exploitation Prevention**: Can't bypass by unequipping during PvP
3. **Intuitive**: Timer represents "time served" not "time passed"
4. **Logout Handling**: Timer pauses on logout automatically (no talisman equipped)

### Removed Consideration

"Relic freshness timer" was removed from design as unnecessary complexity. Talismans have no decay or freshness mechanic.
