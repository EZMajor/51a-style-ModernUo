# 51alpha Spell System Integration Plan: PvP/PvM Separation Architecture

## Document Metadata
- **Version**: v1.0.0
- **Last Updated**: 2025-01-01
- **Authors**: Sphere51a Development Team
- **Applicable ModernUO Version**: v24.0.0+
- **PR Ready**: No (Conceptual Design)

## Executive Summary

**Integration Difficulty: MEDIUM-HIGH (7/10)**

The core challenge is NOT integrating spells into ModernUO (your codebase IS already ModernUO-based). The real complexity is **implementing clean PvP/PvM separation** to prevent Talisman/Chivalry PvM bonuses from affecting PvP combat while keeping spells functionally identical.

###  Key Insight from Your Requirements

> "Spells are a main focus for PvP and we want to keep PvM and PvP separate. Most of the PvM abilities or functions are nullified when PvP happens. PvP is allowed everywhere in open world unless otherwise specified."

This means:
- Spell **casting mechanics** are identical (timing, interruption, mana cost)
- Base spell **damage/effects** work the same
- PvM **bonuses** (Talisman, Honor, Enemy of One, etc.) must be **completely disabled** in PvP
- PvM **progression systems** (talisman XP, build points) must **not trigger** in PvP

## The Real Problem: Context-Aware Spell System

Your architecture requires spells to behave differently based on **combat context**:

### Scenario 1: Player vs Monster (PvM)
```
Player casts Lightning on Dragon:
├─ Base spell damage: 23-27
├─ + Talisman "Dexer" bonus: +25% damage (PvM ONLY)
├─ + Honor perfection bonus: +50% (PvM ONLY)
├─ + Enemy of One: +100% vs reptiles (PvM ONLY)
├─ = Final damage: ~90-115
└─ Awards talisman XP: +5 build points
```

### Scenario 2: Player vs Player (PvP)
```
Player casts Lightning on Enemy Player:
├─ Base spell damage: 23-27
├─ ✗ Talisman bonus: DISABLED
├─ ✗ Honor perfection: DISABLED
├─ ✗ Enemy of One: DISABLED
├─ = Final damage: 23-27 (BASE ONLY)
└─ ✗ No talisman XP awarded
```

### Scenario 3: Transition State (Critical Edge Case)
```
Player attacks Dragon (PvM active):
├─ Talisman bonuses ACTIVE
├─ Enemy player attacks (PvP event triggered)
├─ → Talisman immediately DISABLED for 5 minutes
├─ → All active PvM bonuses removed
└─ → Subsequent spells use base damage only
```

## Architectural Challenges

### Challenge 1: How do spells know they're in PvP context?

**Current ModernUO**: Spells don't differentiate - `Spell.GetNewAosDamage()` applies all bonuses universally.

**Your requirement**: Need a **context check** that determines if the target is a player.

**Solution options**:
1. **Option A**: Check `target.Player` property in damage calculation
2. **Option B**: Add `IPvPContext` interface to track combat state
3. **Option C**: Use `BuildManager` to query if PvM bonuses are active

### Challenge 2: Where do we hook the PvP/PvM detection?

**Critical code paths that need modification**:

| Location | What Happens | What Needs To Change |
|----------|--------------|---------------------|
| `Spell.GetNewAosDamage()` | Calculates spell damage | Add PvP check, skip PvM bonuses if `target.Player == true` |
| `Spell.GetDamageScalar()` | Applies slayer/resist mods | Check talisman active state before applying |
| `BuildManager.OnDamageGiven()` | Awards talisman XP | Skip if `target.Player == true` |
| `HonorableExecution`/`EnemyOfOne` | Chivalry damage bonuses | Return 1.0x multiplier if `target.Player == true` |

### Challenge 3: How do we handle Chivalry abilities cleanly?

Your Combat_DeepDive.md shows Chivalry spells (Enemy of One, Consecrate, Honor) are core to PvM builds. These need to be **functionally disabled** in PvP without breaking the spell system.

**Problematic Chivalry spells**:
- **Enemy of One**: +100% damage vs creature type → Must return 1.0x vs players
- **Honor**: Perfection system grants up to +100% damage → Must not apply vs players
- **Consecrate Weapon**: Ignores armor → Needs separate PvP/PvM behavior

## Recommended Implementation Strategy

### Phase 1: Create Combat Context System

**Goal**: Provide spells with a reliable way to determine "am I in PvP or PvM?"

#### Approach A: Simple Target-Based (RECOMMENDED)

```csharp
// Add to Spell.cs or SpellHelper.cs
public static bool IsPvPContext(Mobile caster, Mobile target)
{
    // PvP if BOTH caster and target are players
    return caster != null && caster.Player &&
           target != null && target.Player;
}

// Usage in damage calculation:
public virtual int GetNewAosDamage(int bonus, int dice, int sides, Mobile target)
{
    var damage = Utility.Dice(dice, sides, bonus) * 100;

    // Apply base damage modifiers (always)
    damage = ApplyBaseDamage(damage);

    // Only apply PvM bonuses if NOT in PvP
    if (!IsPvPContext(Caster, target))
    {
        damage = ApplyTalismanBonus(damage, Caster);
        damage = ApplyHonorBonus(damage, Caster, target);
    }

    return damage / 100;
}
```

**Pros**:
- Simple, no new state management
- Works for all spell types
- Easy to understand and maintain

**Cons**:
- Does not handle "recently in PvP" state (5-minute talisman disable)
- Needs to be called in every damage calculation

#### Approach B: State-Based with Talisman Manager (COMPREHENSIVE)

```csharp
// Extend your BuildManager from PvM_Talismans_DeepDive.md
public static class BuildManager
{
    // Existing talisman progression tracking...

    /// <summary>
    /// Checks if player can use PvM bonuses right now
    /// </summary>
    public static bool CanUsePvMBonuses(PlayerMobile player)
    {
        var prog = GetOrCreateProgression(player);

        // Check if talisman is disabled due to recent PvP
        if (prog.DisabledUntil.HasValue &&
            prog.DisabledUntil.Value > DateTime.UtcNow)
        {
            return false;
        }

        // Check if talisman is equipped and active
        return prog.IsTalismanActive();
    }

    /// <summary>
    /// Called when player engages in PvP - disables talisman
    /// </summary>
    public static void OnPvPEngagement(PlayerMobile player)
    {
        var prog = GetOrCreateProgression(player);
        if (prog.ActiveTalismanId == null) return;

        var def = TalismanRegistry.Get(prog.ActiveTalismanId);
        prog.SetDisabledUntil(DateTime.UtcNow.AddMinutes(def.DisableOnPvPMinutes));

        player.SendMessage($"Your talisman bonuses are disabled for {def.DisableOnPvPMinutes} minutes due to PvP!");
    }
}
```

**Then in spells**:
```csharp
public virtual int GetNewAosDamage(int bonus, int dice, int sides, Mobile target)
{
    var damage = Utility.Dice(dice, sides, bonus) * 100;

    // Base damage (always applied)
    damage = ApplyBaseDamage(damage);

    // PvM bonuses (conditional)
    if (Caster is PlayerMobile pm && target != null)
    {
        bool isPvP = IsPvPContext(Caster, target);

        // Trigger PvP state tracking
        if (isPvP)
        {
            BuildManager.OnPvPEngagement(pm);
        }

        // Only apply bonuses if allowed
        if (!isPvP && BuildManager.CanUsePvMBonuses(pm))
        {
            damage = ApplyTalismanBonus(damage, pm);
            damage = ApplyHonorBonus(damage, pm, target);
        }
    }

    return damage / 100;
}
```

**Pros**:
- Handles "recently in PvP" state (5-minute disable)
- Integrates with existing talisman system
- Single source of truth for PvM bonus eligibility
- Player gets feedback when bonuses are disabled

**Cons**:
- More complex state management
- Requires talisman system to be implemented first

### Phase 2: Modify Core Spell Damage Calculation

**Files to modify**:

#### 1. `Spells/Base/Spell.cs` (Primary Integration Point)

**Current code** (lines 200-257):
```csharp
public virtual int GetNewAosDamage(int bonus, int dice, int sides, bool sdi = true, Mobile singleTarget = null)
{
    var damage = Utility.Dice(dice, sides, bonus) * 100;

    var inscribeSkill = GetInscribeFixed(Caster);
    var inscribeBonus = (inscribeSkill + 1000 * (inscribeSkill / 1000)) / 200;
    var damageBonus = inscribeBonus;

    var intBonus = Caster.Int / 10;
    damageBonus += intBonus;

    if (sdi)
    {
        var sdiBonus = AosAttributes.GetValue(Caster, AosAttribute.SpellDamage);
        if (playerVsPlayer && sdiBonus > 15)
            sdiBonus = 15;
        damageBonus += sdiBonus;
    }

    // ... transformation bonuses, eval skill scaling ...
}
```

**Modified approach**:
```csharp
public virtual int GetNewAosDamage(int bonus, int dice, int sides, bool sdi = true, Mobile singleTarget = null)
{
    var damage = Utility.Dice(dice, sides, bonus) * 100;

    // Determine PvP context
    bool isPvP = singleTarget != null && IsPvPContext(Caster, singleTarget);

    // BASE BONUSES (always applied):
    var inscribeSkill = GetInscribeFixed(Caster);
    var inscribeBonus = (inscribeSkill + 1000 * (inscribeSkill / 1000)) / 200;
    var damageBonus = inscribeBonus;

    var intBonus = Caster.Int / 10;
    damageBonus += intBonus;

    if (sdi)
    {
        var sdiBonus = AosAttributes.GetValue(Caster, AosAttribute.SpellDamage);
        bool playerVsPlayer = Caster.Player && singleTarget?.Player == true;
        if (playerVsPlayer && sdiBonus > 15)
            sdiBonus = 15;
        damageBonus += sdiBonus;
    }

    // PvM-ONLY BONUSES (skip if PvP):
    if (!isPvP && Caster is PlayerMobile pm)
    {
        // Check if talisman bonuses are allowed
        if (BuildManager.CanUsePvMBonuses(pm))
        {
            // Apply talisman spell damage bonus
            damageBonus += GetTalismanSpellDamageBonus(pm);
        }
    }

    // Trigger PvP state tracking
    if (isPvP && Caster is PlayerMobile pvpPlayer)
    {
        BuildManager.OnPvPEngagement(pvpPlayer);
    }

    damage = AOS.Scale(damage, 100 + damageBonus);

    // ... rest of calculation
}

private int GetTalismanSpellDamageBonus(PlayerMobile pm)
{
    var prog = BuildManager.GetOrCreateProgression(pm);
    if (prog.ActiveTalismanId == null) return 0;

    var def = TalismanRegistry.Get(prog.ActiveTalismanId);
    // Return bonus based on talisman type (e.g., Dexer: +0, Mage: +15)
    return def.SpellDamageBonus;
}
```

#### 2. `Spells/Chivalry/EnemyOfOne.cs` - Nullify in PvP

**Current implementation** (needs research, but likely):
```csharp
public class EnemyOfOneSpell : PaladinSpell
{
    public static void ApplyDamageBonus(Mobile caster, Mobile target, ref int damage)
    {
        if (IsUnderEffects(caster) && IsValidTarget(target))
        {
            damage *= 2; // 100% bonus
        }
    }
}
```

**Modified**:
```csharp
public static void ApplyDamageBonus(Mobile caster, Mobile target, ref int damage)
{
    // PvP check: Enemy of One does NOT work vs players
    if (target != null && target.Player)
        return;

    if (IsUnderEffects(caster) && IsValidTarget(target))
    {
        damage *= 2;
    }
}
```

#### 3. `Spells/Bushido/HonorableExecution.cs` - Honor/Perfection System

**Honor perfection** grants up to +100% damage based on kill streak.

**Modification needed**:
```csharp
public static double GetDamageBonus(Mobile caster, Mobile target)
{
    // Perfection does not apply in PvP
    if (target != null && target.Player)
        return 1.0;

    // Existing perfection calculation for PvM
    return CalculatePerfectionBonus(caster);
}
```

### Phase 3: Talisman XP Integration Points

**Files to modify**:

#### `Systems/Sphere51a/BuildManager.cs`

**Existing code** (from PvM_Talismans_DeepDive.md lines 170-182):
```csharp
public static void OnDamageGiven(PlayerMobile attacker, Mobile target, int damage)
{
    // if target is NPC and talisman active -> maybe award credit on kill
    if (target.IsNpc && attacker != null) {
        var prog = GetOrCreateProgression(attacker);
        if (prog.IsTalismanActive() && damage >= prog.ActiveTalismanMinDamageThreshold()) {
            prog.MarkDamageOn(target, damage);
        }
    } else if (target.IsPlayer) {
        // attacker attacked a player -> disable talisman
        DisableTalismanForPvP(attacker);
    }
}
```

**This code is CORRECT** - it already implements the PvP/PvM separation you want!

**Integration point** - Hook this into spell damage:

In `Spells/Base/SpellHelper.cs` (or wherever spell damage is applied):
```csharp
public static void Damage(Spell spell, Mobile target, double damage, ...)
{
    // Existing damage application
    AOS.Damage(target, spell.Caster, (int)damage, ...);

    // NEW: Hook into talisman system
    if (spell.Caster is PlayerMobile pm && target != null)
    {
        BuildManager.OnDamageGiven(pm, target, (int)damage);
    }
}
```

### Phase 4: Combat Integration Hooks

**Where spell damage gets applied** - need to identify these locations:

#### Typical ModernUO spell damage flow:
```
1. Player casts spell
2. Spell.OnCast() → Targeting
3. Spell.Target(Mobile m) method
4. Spell.CheckHSequence(m) → validates
5. Spell.GetNewAosDamage() → calculates damage
6. SpellHelper.Damage() → applies damage
7. Mobile.Damage() → final damage resolution
```

**Hook points needed**:

| Hook Point | Purpose | Where to Add |
|------------|---------|--------------|
| `GetNewAosDamage()` | Apply PvM bonuses conditionally | ✅ Already planned above |
| `SpellHelper.Damage()` | Track talisman damage contribution | Add `BuildManager.OnDamageGiven()` |
| `Mobile.OnDamageReceived()` | Disable talisman on PvP damage taken | Add `BuildManager.OnDamageReceived()` |
| `EnemyOfOne.ApplyBonus()` | Nullify vs players | Add PvP check |
| `Honor.GetPerfectionBonus()` | Nullify vs players | Add PvP check |

## Critical Edge Cases to Handle

### Edge Case 1: Player attacks monster while flagged PvP

**Scenario**: Player attacked another player 2 minutes ago (talisman disabled), now attacks a dragon.

**Expected behavior**: Talisman remains disabled for full 5 minutes, no PvM bonuses apply.

**Implementation**:
```csharp
public virtual int GetNewAosDamage(..., Mobile target)
{
    bool isPvP = IsPvPContext(Caster, target);

    // Check talisman state BEFORE applying bonuses
    if (!isPvP && Caster is PlayerMobile pm)
    {
        // This check handles the "recently in PvP" state
        if (BuildManager.CanUsePvMBonuses(pm))
        {
            // Apply bonuses
        }
        else
        {
            // No bonuses - player is still in PvP cooldown
        }
    }
}
```

### Edge Case 2: Player vs summoned creature (pet/summon)

**Scenario**: Player attacks another player's summoned daemon.

**Question**: Is this PvP or PvM?

**Options**:
- **Option A**: Pets/summons count as PvP (disable talisman)
- **Option B**: Pets/summons count as PvM (keep talisman active)

**Recommended**: Option A - treat as PvP to prevent cheese tactics.

```csharp
public static bool IsPvPContext(Mobile caster, Mobile target)
{
    if (caster == null || target == null) return false;

    // Direct player vs player
    if (caster.Player && target.Player) return true;

    // Player vs another player's pet/summon (treats as PvP)
    if (target is BaseCreature bc && bc.Controlled && bc.ControlMaster != null && bc.ControlMaster.Player)
        return true;

    return false;
}
```

### Edge Case 3: AoE spells hitting mixed targets

**Scenario**: Player casts Meteor Swarm, hits 2 monsters + 1 player.

**Expected behavior**:
- Damage to monsters: Full PvM bonuses (before PvP flag triggers)
- Damage to player: Base damage only, triggers talisman disable
- Subsequent spells: No PvM bonuses for 5 minutes

**Implementation consideration**:
- Call `BuildManager.OnPvPEngagement()` as soon as ANY player is hit
- Damage calculations happen per-target, so first two monsters get bonuses, player and subsequent targets don't

### Edge Case 4: Chivalry spells cast before PvP engagement

**Scenario**:
1. Player casts Enemy of One vs Dragons
2. Kills 5 dragons (perfection at +50%)
3. Attacks another player

**Expected behavior**:
- Enemy of One buff should remain active (it's a timed buff)
- BUT: Damage bonus should not apply vs players
- Perfection progress should not increase from player kills

**Implementation**:
```csharp
// In EnemyOfOne damage calculation
public static void ApplyDamageBonus(Mobile caster, Mobile target, ref int damage)
{
    if (!IsUnderEffects(caster)) return;

    // Buff is active, but check target type
    if (target != null && target.Player)
    {
        // Buff remains active, but no damage bonus vs players
        return;
    }

    // Only apply bonus vs creatures
    if (IsValidCreatureType(target))
    {
        damage *= 2;
    }
}
```

## Implementation Complexity Breakdown

| Phase | Complexity | Time Estimate | Risk Level |
|-------|-----------|---------------|------------|
| **Phase 1: Context System** | Low-Medium | 2-4 hours | Low |
| **Phase 2: Spell.cs Modifications** | Medium | 1-2 days | Medium |
| **Phase 3: Talisman Integration** | Low-Medium | 4-8 hours | Low |
| **Phase 4: Chivalry/Bushido Nullification** | Medium-High | 2-3 days | Medium |
| **Phase 5: Edge Case Testing** | High | 1 week | High |
| **Total** | 7/10 | 2-3 weeks | Medium |

## Architectural Questions Requiring Decisions

### Question 1: Pet/Summon Treatment

**How should player-controlled pets/summons be treated?**

A) **Attacking enemy pets = PvP** (disables talisman immediately)
- Pros: Prevents cheese, clear PvP boundary
- Cons: May feel unfair if pet attacks first

B) **Attacking enemy pets = PvM** (keeps talisman active)
- Pros: Feels more natural, pets are "monsters"
- Cons: Easy to exploit - attack pets to avoid PvP flag

**Recommendation**: Option A - treat pets as PvP for talisman purposes.

### Question 2: Chivalry Spell Behavior

**Should Chivalry spells like Enemy of One be completely disabled in PvP, or just not provide bonuses?**

A) **Keep buff active, zero bonus vs players**
- Pros: Doesn't break the buff system
- Cons: May confuse players ("why is my EoO not working?")

B) **Remove buff entirely when engaging PvP**
- Pros: Clear feedback to player
- Cons: More complex state management

**Recommendation**: Option A - simpler implementation, less state management.

### Question 3: Talisman Disable Timer

**When player engages in PvP, how long should talisman bonuses be disabled?**

A) **5 minutes** - Shorter cooldown, less punishing
B) **10 minutes** - Longer cooldown, stronger PvP/PvM separation
C) **Until death or manual reset** - Permanent until player resets

**Recommendation**: Start with 5 minutes, tune based on player behavior.

### Question 4: Spell Interruption in PvP vs PvM

**Should interruption rules differ in PvP vs PvM?**

A) **Same rules for both** - Consistency
B) **PvP allows more interruptions** - Higher skill ceiling
C) **PvM stricter interruption** - Prevents cheese

**Recommendation**: Same rules for both to avoid confusion.

### Question 5: Talisman System Implementation Priority

**Do we need to implement the full talisman system before integrating spells?**

A) **Yes - implement talismans first**
- Pros: Can test PvP/PvM separation thoroughly
- Cons: Delays spell testing

B) **No - use stub methods**
- Pros: Can test spell mechanics independently
- Cons: May miss integration issues

**Recommendation**: Implement stub methods first (`BuildManager.CanUsePvMBonuses()` always returns false), then add full talisman system.

## Recommended Implementation Order

### Sprint 1: Foundation (Week 1)
1. Add `IsPvPContext()` helper method to Spell.cs or SpellHelper.cs
2. Add stub `BuildManager` with `CanUsePvMBonuses()` returning false
3. Test that spells still work with no bonuses

### Sprint 2: Core Integration (Week 2)
4. Modify `Spell.GetNewAosDamage()` to check PvP context
5. Add talisman bonus calculation hooks (stubbed for now)
6. Hook `BuildManager.OnDamageGiven()` into `SpellHelper.Damage()`
7. Test spell damage in PvP (should be base only)

### Sprint 3: Chivalry/Bushido (Week 2-3)
8. Modify `EnemyOfOne` to nullify vs players
9. Modify `Honor/Perfection` to nullify vs players
10. Modify `Consecrate Weapon` (research needed)
11. Test all Chivalry/Bushido abilities in PvP

### Sprint 4: Talisman System (Week 3-4)
12. Implement full `BuildManager` from PvM_Talismans_DeepDive.md
13. Add talisman items and progression tracking
14. Test talisman bonuses in PvM
15. Test talisman disable on PvP engagement

### Sprint 5: Edge Cases & Polish (Week 4-5)
16. Handle pet/summon targeting
17. Handle AoE spells with mixed targets
18. Handle pre-cast buffs (EoO, Honor) in PvP
19. Add player feedback messages
20. Comprehensive testing

## Risk Mitigation Strategies

### Risk 1: Breaking existing ModernUO spells

**Mitigation**:
- Make all PvM bonus code **additive only** - never modify base calculations
- Use feature flags to toggle PvP/PvM separation on/off
- Test with standard ModernUO scripts first

### Risk 2: Performance impact of context checks

**Mitigation**:
- `IsPvPContext()` is a simple property check (no allocations)
- BuildManager uses readonly structs (your architecture requirement)
- Add telemetry to measure performance impact

### Risk 3: Edge cases breaking talisman system

**Mitigation**:
- Implement comprehensive logging for PvP/PvM transitions
- Add admin commands to inspect player talisman state
- Create unit tests for all edge cases

## Files Requiring Modification

### Core Spell System
1. ✏️ `Spells/Base/Spell.cs` - Add PvP context checks, talisman bonuses
2. ✏️ `Spells/Base/SpellHelper.cs` - Add `IsPvPContext()`, hook talisman tracking
3. ✏️ `Spells/Base/MagerySpell.cs` - May need override for PvM bonuses

### Chivalry System
4. ✏️ `Spells/Chivalry/EnemyOfOne.cs` - Nullify vs players
5. ✏️ `Spells/Chivalry/ConsecrateWeapon.cs` - Research needed
6. ✏️ `Spells/Chivalry/HolyLight.cs` - Research if needs changes

### Bushido System
7. ✏️ `Spells/Bushido/HonorableExecution.cs` - Nullify perfection vs players
8. ✏️ `Spells/Bushido/Confidence.cs` - Research if needs changes

### Talisman System (New)
9. 🆕 `Systems/Sphere51a/BuildManager.cs` - From PvM_Talismans_DeepDive.md
10. 🆕 `Systems/Sphere51a/TalismanDefinition.cs` - Talisman data structures
11. 🆕 `Systems/Sphere51a/TalismanItem.cs` - Equippable talisman items
12. 🆕 `Systems/Sphere51a/BuildProgression.cs` - Player progression tracking

### Configuration
13. 🆕 `Systems/Sphere51a/Sphere51aConfig.cs` - From Combat_DeepDive.md
14. 🆕 `Data/Talismans.json` - Talisman definitions

## Success Criteria

### ✅ Integration is successful when:

1. **Spells work identically in PvP** - No PvM bonuses leak through
2. **Talisman bonuses apply in PvM** - Full bonuses vs monsters
3. **Talisman disables on PvP** - Immediate disable with 5min cooldown
4. **Chivalry nullified in PvP** - EoO, Honor, Consecrate don't affect players
5. **Edge cases handled** - Pets, AoE, pre-cast buffs all work correctly
6. **No performance regression** - Context checks don't slow down combat
7. **Player feedback clear** - Messages explain when/why bonuses are disabled

## Final Assessment

**Difficulty**: 7/10 (Medium-High)
- NOT difficult because of ModernUO incompatibility
- Difficult because of **architectural complexity** of PvP/PvM separation
- Requires careful design to avoid edge cases

**Time Estimate**: 2-3 weeks for complete implementation + testing

**Biggest Risk**: Edge cases where PvM bonuses leak into PvP, or where transitions between PvM→PvP don't properly disable systems.

**Success depends on**: Clear architectural decisions (questions above) and comprehensive testing of edge cases.

## Deep Analysis Results (Post-Codebase Analysis)

### Creature Types and Spell Casting Architecture
From `BaseCreature.cs` analysis:

**Player-Owned Creatures:**
- **Pets**: `Controlled = true`, `ControlMaster = PlayerMobile`, `SummonMaster = null`
- **Summons**: `Summoned = true`, `SummonMaster = PlayerMobile`, `Controlled = false`

**Spell System Architecture:**
- All creatures (players, monsters, pets) use the same `Spell.cs` `GetNewAosDamage()` for spell damage calculation.
- PvM bonuses (talisman, Chivalry) are **PlayerMobile-specific** - monsters/pets don't have `TalismanProgression` or `EnemyOfOneType`.
- When a pet casts a spell: `Caster` is `BaseCreature{Controlled: true}`, so `Caster.Player == false`, no PvM bonuses apply.
- Monsters casting spells (via `MageAI`) use the same code but get base damage + slayer/resist mods only.

**Ability System (Separate from Spells):**
- `BaseCreature` has `MonsterAbility[]` system for special attacks.
- Abilities are NOT spells - PvP/PvM separation for abilities would require separate implementation.
- Current document scope focuses on spells only.

### How Code Changes Affect NPCs vs Players
**Spell System (Spells):**
- Changes to `Spell.cs` `GetNewAosDamage()` only affect PvM bonuses if checks like `if (Caster is PlayerMobile pm && !IsPvPContext(Caster, target))` are added.
- Monsters/pets casting spells: No PvM bonuses (they're not `PlayerMobile`), so unchanged behavior.
- Monster-on-player spells: Damage scales apply (slayer/reduce), but no caster PvM bonuses. Correct.
- Pet-on-monster spells: Base damage only, no talisman/honor bonuses. Intuitive.

**Ability System:**
- Monster abilities are separate (not spells) - changes to `Spell.cs` don't affect them.
- PvP/PvM separation for abilities could be added separately if needed.

### Finalized Architectural Decisions

Based on deep analysis of `Spell.cs`, `BaseCreature.cs`, and system interactions:

1. **Pet/Summon Treatment**: Attack by/on player pet/summon = PvP, disable talisman. Pet speed debuff (50% reduction) for PvP engagement.

2. **Chivalry Spell Behavior**: **SOLVED by Talisman Disable** - Chivalry spells require active talisman. PvP disable prevents Chivalry access entirely.

3. **Talisman Disable Timer**: 5 minutes post-PvP to allow PvM recovery.

4. **Spell Interruption Rules**: Same rules for PvP/PvM.

5. **Pet Spell Casting**: Player-controlled pets get base damage only (Caster != PlayerMobile).

6. **AoE Mixed Targets**: Any hit on player-controlled target triggers PvP disable.

7. **Summoning**: No talisman disable (summons allowed in PvP).

8. **Monster Abilities**: Out of scope (abilities ≠ spells).

## Cross-System Dependencies

### Talisman System Integration
- **Chivalry Control**: Chivalry spell access requires active talisman. PvP deactivation immediately disables all Chivalry casting.
- **PvP State Management**: Talisman tracks PvP engagement via BuildManager.OnDamageGiven/Received.
- **Timer Coordination**: 5-minute disable timer syncs with pet speed debuff duration.

### Combat System Integration
- **PvP Context Detection**: Spell damage bonuses check CombatTimerManager for active PvP contexts.
- **Pet Speed Modifications**: BaseCreature movement speed reduced 50% during PvP state tracking.
- **Interrupt Hooks**: Combat system provides PvP state for spell bonus eligibility decisions.

### Creature Behavior Notes
- **Pet Casting**: Pets use Spell.cs but bypass PvM bonus checks (no PlayerMobile progression).
- **Monster Casting**: AI spells get base calculations only, no talisman/Chivalry bonuses.
- **Summons**: Treated same as pets for PvP contexts (control source determines PvP triggering).

## Simplified Implementation Plan (4 Sprints)

**Sprint 1: Core Infrastructure**
- Add `IsPvPContext()` and `PlayerControlled()` helpers to SpellHelper.cs
- Stub `BuildManager` with talisman state tracking and 5-minute PvP disable timer

**Sprint 2: PvP/PvM Spell Separation**
- Modify `Spell.GetNewAosDamage()`: `if (Caster.Player && !IsPvPContext() && BuildManager.CanUsePvMBonuses(pm)) { apply bonuses }`
- Hook `BuildManager.OnDamageGiven()` for XP tracking and PvP detection

**Sprint 3: Pet PvP Mechanics**
- Add 50% speed reduction in `BaseCreature` during PvP (CombatTimerManager integration)
- 5-minute debuff duration matching talisman timer

**Sprint 4: Integration Testing**
- Verify Chivalry spells fail when talisman deactivated
- Test pet speed debuffs and PvP state resets
- Comprehensive PvP/PvM balance testing

## Success Criteria
- Spells damage identically in PvP (base only), full bonuses in PvM
- Talisman disables on PvP hit, enables after 5 minutes PvP-free → Chivalry automatically re-enabled
- Pets 50% slower when attacking players, full speed elsewhere
- No impact on monster/NPC spell casting behavior

## Development Prerequisites

### ModernUO Base Requirements
- **Minimum Version**: ModernUO v24.0.0 or later (Assumes clean master branch)
- **Branch**: Use `feature/spell_pvp_pvm_sep` for implementation
- **Dependencies**: Requires PvM Talisman system (BuildManager) to be implemented first

### Required Framework Knowledge
- ModernUO Spell system architecture (`Spells/Base/Spell.cs`, `SpellHelper.cs`)
- Combat damage flow and hooks
- PlayerMobile state persistence
- C# async patterns for timer management

### Pre-Implementation Checklist
- [ ] PvM Talisman system implemented and tested
- [ ] Database migration scripts prepared for talisman progression tables
- [ ] Feature flag added to disable PvP/PvM separation for testing
- [ ] Combat damage hooks verified in `CombatSystem.cs` or equivalent

## Code Integration Guide

### Step 1: Context Helpers (Low Risk)
Add `IsPvPContext()` method to `Spells/Base/SpellHelper.cs`:
```csharp
public static bool IsPvPContext(Mobile caster, Mobile target)
{
    if (caster == null || target == null) return false;
    // Direct PvP
    if (caster.Player && target.Player) return true;
    // Player vs player-controlled creature = PvP
    if (target is BaseCreature bc && bc.Controlled && bc.ControlMaster?.Player == true) return true;
    return false;
}
```

### Step 2: Spell Damage Modifications (Medium Risk)
Modify `Spell.cs.GetNewAosDamage()`:
1. Add PvP context check early in method
2. Skip talisman/honor bonuses when `isPvPContext == true`
3. Add `BuildManager.OnPvPEngagement()` call for PvP state tracking

### Step 3: Talisman XP Hooks (Low Risk)
Hook into spell damage application in `SpellHelper.Damage()`:
```csharp
if (spell.Caster is PlayerMobile pm && target != null)
{
    BuildManager.OnDamageGiven(pm, target, (int)damage);
}
```

### Step 4: Chivalry Nullification (Medium Risk)
Modify `Spells/Chivalry/EnemyOfOne.cs` and `Spells/Bushido/HonorableExecution.cs` per examples above.

### Step 5: Testing Integration
1. Unit tests for `IsPvPContext()` method
2. Integration tests for PvP spell damage vs base damage
3. Endurance tests for talisman disable timers

## Performance Benchmarks

### Expected Performance Impact
- `IsPvPContext()`: < 0.1ms (simple property checks)
- `BuildManager.CanUsePvMBonuses()`: 0.5-2ms (registry lookup + state check)
- Total spell damage calculation: ~5% increase due to conditional logic

### Monitoring Recommendations
- Add telemetry counter for PvP context checks
- Log talisman disable events for balance data
- Profile combat heavy scenarios (20+ players fighting)

## Maintenance Notes

### Future Enhancements
- Consider caching PvP state to reduce lookups in high-population scenarios
- Add analytics for PvP/PvM engagement ratios
- Evaluate expanding PvP context to include guild wars

### Database Considerations
- Additional fields may be needed in player persistence for talisman state
- Consider compressed storage for large talisman registry

### Rollback Procedures
1. Comment out PvM bonus application blocks
2. Disable `BuildManager` hooks
3. Restore original Chivalry bonus methods

## Testing Procedures

### Unit Testing (Automated)
```csharp
[TestMethod]
public void IsPvPContext_DirectPlayerVsPlayer_ReturnsTrue()
{
    var caster = new PlayerMobile();
    var target = new PlayerMobile();
    Assert.IsTrue(SpellHelper.IsPvPContext(caster, target));
}

[TestMethod]
public void IsPvPContext_PlayerVsMonster_ReturnsFalse()
{
    var caster = new PlayerMobile();
    var target = new BaseCreature();
    Assert.IsFalse(SpellHelper.IsPvPContext(caster, target));
}
```

### Integration Testing (Live Server)
1. **Spell Damage Verification**: Cast spells vs monsters and players, log damage values
2. **Talisman Disable Test**: Engage PvP, verify bonuses disabled for 5 minutes
3. **Chivalry Nullification**: Cast EoO in PvP, verify no bonuses vs players
4. **Edge Case Testing**: AoE spells hitting mixed targets, pet attacks, pre-cast buffs

### Load Testing
- Simulate 50 players casting spells simultaneously
- Verify no performance degradation under combat load

## Change Log

### v1.0.0 - 2025-01-01 (Initial Professional Standard Update)
- Added **Document Metadata** section with versioning and compatibility info
- Added **Development Prerequisites** with framework knowledge requirements and pre-implementation checklist
- Added **Code Integration Guide** with step-by-step implementation instructions
- Added **Performance Benchmarks** with expected impact and monitoring recommendations
- Added **Maintenance Notes** with enhancement plans and rollback procedures
- Added **Testing Procedures** with unit and integration testing guidelines
- Standardized **Success Criteria** section with measurable outcomes
- Enhanced **Risk Mitigation** with specific telemetry and logging strategies
- Improved **File Modification List** with risk levels and implementation order
- Added cross-references to related system documents (PvM Talismans, Combat DeepDive)

### v0.3.0 - 2024-XX-XX (Original Architecture Analysis)
- Initial architectural planning and requirements analysis
- PvP/PvM context decision framework established
- Implementation strategy baselines defined
- Edge case identification completed
