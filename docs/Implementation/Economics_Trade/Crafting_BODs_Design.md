# Crafting BODs Deep Dive

## Document Metadata
- **Version**: v2.0.0
- **Last Updated**: 2025-01-03
- **Authors**: Sphere51a Development Team
- **Applicable ModernUO Version**: v24.0.0+
- **PR Ready**: No (Conceptual Design)

## Overview
Bulk Order Deed (BOD) system with immediate point crediting and authenticity verification via HMAC signatures to prevent crafted item tampering.

## Algorithms and Logic
Server-side validation with HMAC hash verification, immediate scoring on completion, and tamper-proof item tracking.

## BOD Authenticity Verification

csharppublic class CraftedItem : Item
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

## Edge Cases
Handles duplicate item submissions, expired BOD claims, rate-limited claim attempts, cross-dungeon material conflicts, and progressive unlock validation for new players.

## Implementation Details
Server-authentic crafting validation with item originality checks, weekly score batching system, faction point distribution, and integration with rotating dungeon loot mechanics.

## Testing Plan
Comprehensive item authenticity testing, weekly batch processing validation, exploit prevention testing, and dungeon loot table integration verification.

## Development Prerequisites

### ModernUO Base Requirements
- **Minimum Version**: ModernUO v24.0.0 or later (Assumes clean master branch)
- **Branch**: Use `feature/crafting_bods` for implementation
- **Dependencies**: Requires BOD framework, faction town system, rotating dungeons

### Required Framework Knowledge
- ModernUO BOD (Bulk Order Deed) system architecture
- Crafting skill system integration and validation
- Dungeon loot table management and randomization
- Faction system point distribution mechanics
- Server-side item validation and originality checks

### Pre-Implementation Checklist
- [ ] BOD framework integration points identified
- [ ] Dungeon loot tables and rarity mechanics understood
- [ ] Faction town scoring system mapped out
- [ ] Weekly processing and batch system designed
- [ ] Player progression limits and unlock mechanics configured

## Code Integration Guide

### Step 1: BOD Generation System (Medium Risk)
1. Implement weekly BOD generation from dungeon loot tables
2. Create skill-based difficulty escalation logic
3. Set up material rarity weighting system

### Step 2: Validation and Submission (High Risk)
1. Develop server-side crafting validation system
2. Implement originality checks for submitted items
3. Create claim processing and weekly score batching

### Step 3: Dungeon Integration (Medium Risk)
1. Connect BOD materials to rotating dungeon loot tables
2. Implement anti-PK vs standard vs bonus dungeon distinctions
3. Add faction score boost mechanics

### Step 4: Player Progression (Low Risk)
1. Implement 4 BOD per week limit with progressive unlocks
2. Add weekly reset and claim tracking
3. Create UI for BOD management and submission

### Step 5: Testing and Balancing (Low Risk)
1. Validate item submission and validation flows
2. Test weekly batch processing and faction scoring
3. Balance material rarity and difficulty progression

## Performance Benchmarks

### Expected Performance Impact
- **BOD Generation**: <50ms per weekly generation cycle
- **Item Validation**: <10ms per crafting submission
- **Claim Processing**: <25ms per weekly batch operation
- **Faction Scoring**: <100ms for point distribution across all towns
- **Memory Usage**: <5MB additional per active BOD system

### Monitoring Recommendations
- Track BOD completion rates and submission patterns
- Monitor validation failure rates (>5% may indicate exploits)
- Alert on weekly batch processing failures
- Track faction point distribution imbalances
- Monitor dungeon loot table material availability

## Maintenance Notes

### Future Enhancements
- Add rare material event mechanics
- Implement guild-wide BOD challenges
- Create custom BOD creation interface
- Add premium BOD marketplace

### Operational Considerations
- Regular dungeon loot table balancing
- BOD difficulty progression tuning
- Anti-exploit measure updates
- Player feedback processing and adjustments

### Rollback Procedures
1. Suspend BOD generation and submissions
2. Archive current weekly BOD data
3. Revert to previous validated state
4. Validate faction scores and player progression
5. Resume BOD system with clean weekly cycle

## Testing Procedures

### Unit Testing (Automated)
```csharp
[TestMethod]
public void BODValidation_ValidCraftingSubmission_AcceptsItem()
{
    // Arrange
    var bod = CreateWeeklyBOD();
    var submittedItem = CreateValidCraftingItem();

    // Act
    var result = bod.ValidateSubmission(submittedItem);

    // Assert
    Assert.IsTrue(result.IsValid);
    Assert.IsNotNull(result.FactionPoints);
}

[TestMethod]
public void BODGeneration_WeeklyCycle_CreatesProgressiveDifficulty()
{
    // Arrange
    var generator = new BODGenerator();
    var playerSkills = new Dictionary<SkillName, int> {
        { SkillName.Blacksmith, 100 }, { SkillName.Carpentry, 80 }
    };

    // Act
    var bod = generator.GenerateWeeklyBOD(playerSkills);

    // Assert
    Assert.IsNotNull(bod);
    Assert.IsTrue(bod.Difficulty > 0);
}
```

### Integration Testing (Live Server)
1. **BOD Submission Testing**: Verify complete craft-to-claim workflow
2. **Weekly Processing**: Test end-of-week batch processing
3. **Faction Scoring**: Validate point distribution to towns
4. **Dungeon Integration**: Confirm material drops from dungeons

### Load Testing
- Test BOD submissions under peak crafting activity
- Validate weekly processing with thousands of active BODs
- Test faction scoring with multiple competing towns
- Verify system performance with high-item submission volume

## Change Log

### v2.0.0 - 2025-01-03 (Updated to align with Architecture_Master v2.0.0)
- Changed to immediate point crediting on BOD completion.
- Added HMAC-based authenticity verification for crafted items.

### v1.0.0 - 2025-01-01 (Initial Professional Standard Update)
- Added **Document Metadata** section with versioning and compatibility info
- Added **Development Prerequisites** with framework knowledge requirements and pre-implementation checklist
- Added **Code Integration Guide** with step-by-step implementation instructions and risk levels
- Added **Performance Benchmarks** with expected impact and monitoring recommendations
- Added **Maintenance Notes** with enhancement plans and rollback procedures
- Added **Testing Procedures** with unit and integration testing guidelines
- Standardized structure to meet modern developer documentation standards
- Enhanced BOD system with comprehensive technical specifications and implementation details
