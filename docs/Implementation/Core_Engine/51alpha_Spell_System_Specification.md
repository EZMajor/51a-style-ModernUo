# 51alpha Spell System Specification

## Document Metadata
- **Version**: v1.1.0
- **Last Updated**: 2025-12-08
- **Authors**: Sphere51a Development Team
- **Status**: Technical Specification (Ready for Implementation)

## Executive Summary

The 51alpha Spell System represents a comprehensive redesign of ModernUO's spell casting mechanics with the following key improvements:

- **Faster Targeting**: Players select targets immediately upon spell cast
- **Costly Interruptions**: Interrupting spells wastes resources already committed
- **Streamlined States**: Single `SpellState.Casting` replaces complex state machine
- **Enhanced Scrolls**: Scrolls provide mana efficiency and speed bonuses
- **Minimal NPC Impact**: Existing player-centric logic preserves monster behavior

## 1. New Spell Casting Flow

### Current Flow (Pre-51alpha)
```
Player Action → Cast() → Timer (castDelay) → OnCast() [Targeting]
                                     ↓
                            SpellState.Sequencing → CheckSequence() → Execute
                                     ↓
                            Resource Consumption
```

### New Flow (51alpha)
```
Player Action → Cast()
         ↓
Target Selection (immediate)
         ↓
CheckSequence() [Validation + Consumption]
         ↓
Timer (castDelay) → Execute
```

### Key Changes
- **Targeting moves before timing**: `OnCast()` called immediately in `Cast()`
- **Validation before commitment**: `CheckSequence()` handles all pre-validation
- **Resource consumption early**: Mana/regents consumed on target validation
- **Single state**: `SpellState.Casting` covers entire process
- **Costly interruptions**: No resource refund on interruption

## 2. State Machine Redesign

### State Definitions

#### SpellState.None
Default state for non-casting spells, identical to current behavior.

#### SpellState.Casting (Modified)
- **Purpose**: Single state covering entire casting process
- **Entry**: Set in `Cast()` before `OnCast()`
- **Duration**: From cast attempt to spell resolution OR interruption
- **Exit**: Set to `SpellState.None` on completion, timeout, or interruption

#### SpellState.Sequencing (REMOVED)
The `SpellState.Sequencing` state is completely eliminated from the system. All casting logic now occurs under `SpellState.Casting`.

### State Transition Flow
```mermaid
graph TD
    A[SpellState.None] --> B[Cast()]
    B --> C[SpellState.Casting]
    C --> D[OnCast() - Targeting]
    D --> E[CheckSequence() - Validation]
    E --> F[CastTimer.Start]
    F --> G[Execute Spell]
    G --> A
    E --> H[Interrupt] --> A
    F --> H
```

## 3. Timing Modifications

### CastDelay Calculation
- **Timing Point**: `GetCastDelay()` called AFTER target selection, not before
- **State Context**: Run while in `SpellState.Casting`
- **Animation**: Animations begin immediately on cast, not delayed
- **Movement**: Players can move while casting (no freezing)

### Player Movement During Casting

**Key Change**: Players are **NOT frozen** during spell casting in 51alpha.

#### Before (Classic UO/ModernUO):
- Player frozen (`Caster.Delta(MobileDelta.Flags)`) during entire cast delay
- Movement interrupts spell with fizzle
- "You are frozen and can not move" message

#### After (51alpha):
- Player **can freely move** while target window is open
- Movement **does not interrupt** spell casting
- Cast delay timing continues even while moving
- Only specific interruption sources can break spells

### Base Timing (Magery Spells)
```csharp
public override TimeSpan CastDelayBase =>
    TimeSpan.FromSeconds((3 + (int)Circle) * CastDelaySecondsPerTick); // 0.25s per tick
```

| Circle | Base Time | With FC 120 |
|--------|-----------|-------------|
| 1      | 1.0s      | 0.25s       |
| 2      | 1.25s     | 0.25s       |
| 3      | 1.5s      | 0.25s       |
| 4      | 1.75s     | 0.5s        |
| 5      | 2.0s      | 0.75s       |
| 6      | 2.25s     | 1.0s        |
| 7      | 2.5s      | 1.25s       |
| 8      | 2.75s     | 1.5s        |

## 4. Resource Consumption Timing

### Resource Flow Changes

#### Current System
- Mana check in `Cast()`
- Resource consumption in `CheckSequence()` after delay
- Interruption refunds all resources

#### New System
- **Mana check**: Moved to `CheckSequence()` (post-targeting)
- **Reagent consumption**: In `CheckSequence()` (post-validation)
- **Resource commitment**: Everything consumed before delay starts
- **Interruption impact**: Resources wasted on interruption

### Mana Scaling Logic
```csharp
public virtual int ScaleMana(int mana)
{
    double scalar = 1.0;

    // Existing LMC, Mind Rot corrections...

    // NEW: Scroll mana reduction (43% cheaper)
    if (Scroll is SpellScroll)
        scalar /= 1.755;

    return (int)(mana * scalar);
}
```

### Skill Requirements
- **Validation Point**: Skill checks remain in `CheckSequence()`
- **Interruption Risk**: Failed skill check wastes consumed resources
- **Timing**: Occurs during target validation phase

## 5. Interruption System Changes

### Current Behavior
```csharp
if (State == SpellState.Sequencing)
    Target.Cancel(); // Simple cancellation, no cost
```

### New Behavior (Costly Interruptions)
```csharp
if (State == SpellState.Casting)
{
    State = SpellState.None;
    // NO resource refund - resources already consumed
    DoDisturbFizzle(); // Visual feedback
    // Spell fails with resource loss
}
```

### Interruption Types & Rules

#### What CAN Interrupt Spells:
- **Casting Another Spell**: Attempting to cast a second spell while first is active
- **War Mode Toggle**: Changing from/to war mode during spell casting
- **Using Bandages**: Medical actions interrupt spell casting
- **COMBAT**: Entering combat situations

#### What CANNOT Interrupt Spells:
- **Movement**: Players can freely move while casting (no freezing)
- **Equipping Items**: Changing equipment does not break casting
- **Regular Damage**: Non-lethal damage does not interrupt (requires protection check)
- **Other Object Usage**: Using non-interruptive objects is allowed

#### Interruption Results:
- **Before Resource Consumption**: Targeting phase interruptions = no resource cost
- **After Validation Success**: Post-CheckSequence interruptions = costly fizzle (resources wasted)
- **During Execution Phase**: Resources already committed = wasted resources

### Interruption Timing
- **Early Phase**: Before mana consumption (targeting) = no cost
- **Post-Validation**: After CheckSequence() success = resources wasted
- **During Execution**: Resources already consumed = costly fizzle

## 6. Target Selection Changes

### Timing Shift
- **Current**: Targeting created during timer callback (SpellState.Sequencing)
- **New**: Targeting created immediately in `Cast()` (SpellState.Casting)

### State Implications
- **Movement Freedom**: Player can move freely during targeting phase (no freezing)
- **Target Timeout**: Applies from target creation through resolution (30 seconds)
- **Interruption Window**: Longer interruption window (includes targeting phase)

### Target Management
```csharp
public bool Cast()
{
    // ... validations ...

    State = SpellState.Casting;
    OnCast(); // Creates targeting immediately

    if (caster.Player && caster.Target != null)
    {
        caster.Target.BeginTimeout(caster, 30000); // 30-second timeout
    }

    return true;
}
```

## 7. Scroll System Modifications

### Mana Costs: 43% Reduction
Scroll spells consume significantly less mana than memory casting:

| Spell | Memory Mana | Scroll Mana | Reduction |
|--------|-------------|-------------|----------|
| Heal   | 4-6         | ~2-3        | -43%     |
| Poison | 8-12        | ~5-7        | -43%     |
| Wall of Stone | 30-45    | ~17-26     | -43%     |

### Cast Timing Benefits
Scrolls gain 0.5 second casting speed reduction for circles 3+:

| Circle | Memory Base | Scroll Speed |
|--------|-------------|--------------|
| 1,2    | Standard    | Standard     |
| 3      | 1.5s        | **1.0s**     |
| 4      | 1.75s       | **1.25s**    |
| 5      | 2.0s        | **1.5s**     |
| 8      | 2.75s       | **2.25s**    |

### Implementation Across Schools

#### Magery Scrolls
```csharp
public override TimeSpan GetCastDelay()
{
    var baseDelay = base.GetCastDelay();

    if (Scroll is SpellScroll && Circle >= SpellCircle.Third)
    {
        return TimeSpan.FromSeconds(
            Math.Max(0.25, baseDelay.TotalSeconds - 0.5)
        );
    }

    return baseDelay;
}
```

#### Necromancy/Chivalry Scrolls
Same logic applied using circle equivalents for non-circle spell schools.

### Balance Philosophy
Scrolls provide:
- **Economic Advantage**: Lower upfront mana cost
- **Convenience**: No reagent requirements
- **Speed**: Faster casting for complex spells
- **Risk**: Still subject to interruption (only through war mode toggle, casting another spell before it completes, using a bandage) (equipping does not interrupt)
- **Trade-off**: Single-use, expensive to obtain

## 8. NPC and Monster Effects

### Minimal Behavioral Changes
- **Existing Player Gating**: UI/mantra/effects bypass NPCs via `Caster.Player` checks
- **Targeting Logic**: NPCs use direct assignment vs. client cursors
- **Resource Logic**: NPCs skip reagent costs (already handled)
- **State Dependencies**: NPC AI uses same spell mechanics

### Performance Benefits
- **Faster AI Response**: Immediate targeting enables quicker decisions
- **Simplified Logic**: Single state management reduces complexity

### Compatibility Notes
- **Memory Spells**: NPCs continue using normal mana/scaling
- **Scroll Usage**: NPCs can use scrolls (no behavioral difference)
- **Interruption**: Players or NPCs do not have hurt-based interruptions

## 9. Code Architecture Changes

### Modified Files

#### Spell.cs - Core Changes
- `Cast()`: Immediate `OnCast()` call, state management
- `CheckSequence()`: Early resource consumption
- `ScaleMana()`: Scroll cost reduction
- `Disturb()`: Costly interruption logic
- `GetCastDelay()`: Post-target timing calculation

#### MagerySpell.cs - Timing
- `CastDelayBase`: Circle-based timing formula
- `GetCastDelay()`: Scroll speed bonuses

#### NecromancerSpell.cs & PaladinSpell.cs - Extension
- Same scroll logic applied for non-circle spell schools

### New Methods/Virtuals
- `AdjustCastDelayForScrolls()`: Allows per-spell scroll modifications
- Scroll mana calculations integrated into `ScaleMana()`

## 10. Testing Requirements

### Functional Testing
- [ ] Spell casting flow: Cast → Target → Validation → Execution
- [ ] Resource consumption timing: Post-validation consumption
- [ ] Interruption behavior: Costly fizzle without refund
- [ ] State management: Single SpellState.Casting throughout
- [ ] Timing accuracy: Target-first delay calculation

### Balance Testing
- [ ] Mana costs: 43% reduction for scrolls
- [ ] Cast speeds: -0.5s for scroll circles 3+
- [ ] Interruption frequency: No resource refunds
- [ ] NPC behavior: Unchanged spell mechanics

### Integration Testing
- [ ] PvP/PvM mechanics (from SpellSystem_Integration_Design.md)
- [ ] Faster Casting compatibility
- [ ] Multi-spell queuing
- [ ] Edge cases: Connection loss, system interrupts

## 11. Balance Implications

### Player Experience
- **Faster Combat**: Immediate targeting reduces decision paralysis
- **Risk/Reward**: Interrupting becomes much more punishing
- **Economic Choice**: Scrolls vs. memory becomes refined
- **Skill Dependency**: FC remains crucial, but targeting speed helps

### System Performance
- **State Management**: Simplified single-state system
- **Memory Usage**: Unchanged resource allocation
- **Server Load**: Minimal impact from timing shifts
- **Interrupt Handling**: Streamlined fizzle system

## 12. Future Considerations

### Potential Extensions
- **Scroll Variants**: Different scroll qualities (novice/master)
- **Interrupt Mitigation**: Defensive spells against interruption
- **Targeting Improvements**: Enhanced client targeting UI
- **Performance Monitoring**: Cast timing analytics

### Balance Tuning Points
- **Mana Reduction**: Scroll efficiency adjustable via `/1.755` modifier
- **Speed Bonuses**: Circle thresholds and timing reductions modifiable
- **Interruption Severity**: Resource waste levels adjustable
- **FC Compatibility**: Scroll bonus stacking rules tweakable

## 13. Document Integration

### Related Documents
- **Combat_System_Design.md**: Core spell flow diagrams
- **SpellSystem_Integration_Design.md**: PvP/PvM integration
- **Master_Architecture.md**: High-level system overview

### Implementation Status
- **Design Phase**: ✅ Complete
- **Code Implementation**: Ready for Act Mode
- **Testing Phase**: Awaiting implementation
- **Deployment**: Following successful testing

---

**This specification captures the complete technical requirements for the 51alpha spell system redesign, including all modifications to flow, timing, resource consumption, interruption behavior, and enhanced scroll mechanics.**
