# Weight Reduction Bags Deep Dive

## Overview
Special pouches/bags reducing the effective weight of items placed inside; wear system (10/10 quality), death degradation leading to disappearance; rare mounts provide 5% global reduction.

### Tiered Weight Reduction Bags
- **Tier 1 (SpinedLeather)**: 5% effective weight reduction, crafted using SpinedLeather (Tailoring skill 65+ required).
- **Tier 2 (HornedLeather)**: 10% effective weight reduction, crafted using HornedLeather (Tailoring skill 80+ required).
- **Tier 3 (BarbedLeather)**: 20% effective weight reduction, crafted using BarbedLeather (Tailoring skill 100+ required).

## Algorithms and Logic
Weight reduction applies multiplicatively to child item weights when calculating container total weight. Formula: reduced_total = base_weight + sum((child_weight * (1 - reduction_percentage)) for child in items). Tiers ensure escalating benefits while requiring higher skill/resource investments:
- Lightweight Pouch/Tier 1: 5% weight reduction.
- Reinforced Pouch/Tier 2: 10% weight reduction.
- Master Pouch/Tier 3: 20% weight reduction.

## Edge Cases
Pouch wear-out prevents farming exploits; global mount bonus stacks with pouches but capped.

## Implementation Details
Pouch items primarily crafted via Tailoring skill using specific leather resources (SpinedLeather for Tier 1, etc.), requiring appropriate skill levels. Wear tracked on player death events; bonuses applied via container TotalWeight override to reduce item weights. Classes like SpinedLeatherWeightReductionBackpack inherit from Backpack with overridden TotalWeight method.

## Testing Plan
Weight reductions on stacking items, ladder wear simulation, mount bonus integration.

## Change Log
- Added leather-type linkages for tiered weight reduction bags (Spined, Horned, Barbed Leathers).
- Updated tiers to 5%, 10%, 20% with corresponding Tailoring skill requirements (65, 80, 100).
- Detailed implementation via crafts and TotalWeight overrides.
- Confirmed reduction applies to all items inside the bag/container.
