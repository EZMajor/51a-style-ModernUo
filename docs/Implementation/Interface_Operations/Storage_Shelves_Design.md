# Storage Shelves Deep Dive

## Overview
Virtual storage containers reducing UO engine overhead by compressing bulk items into data entries instead of individual object instances. Players secure in house/inn rooms for major QoL improvements in inventory management.

## Algorithms and Logic
Virtualization logic checks item compatibility (identical/stackable items without scripts/unique states). Stored as [ItemID, Quantity] in data tables; items instantiated only on withdrawal with auto-stacking to prevent clutter.

## Edge Cases
Items with unique properties (aspected gear, poisoned weapons, script-attached) excluded from virtualization. Container overflows managed by auto-stacking logic.

## Implementation Details
StorageShelf extends BaseContainer; Secured property required. Loadouts per-character for resupply automation; configurable settings for fallback rules (downgrade resources, substitute food types).

## Testing Plan
Virtualization integrity, loadout resupply accuracy, gump UI responsiveness.

## Benefits for UO Engine and Players

### Server-Side Benefits: Space and Memory Savings
UO engines struggle with tracking individual items as objects with serials, locations, attributes. Each has a full instance: ItemId, Amount, Hue, Owner, Durability, etc.

Storage shelves compress thousands of objects into virtual records: {"IronIngot": 12500} instead of 12,500 item objects. Reduces memory, CPU (no tracking/saving each), network (no massive container packets).

### Player-Side Benefits: Quality of Life and Functionality
Centralized storage: potions, reagents, resources, gear, scrolls organized by categories. Loadouts for auto-equip/resupply, reducing manual sorting.

### Core Mechanism: Virtual Item Storage
Items stored as data entries: no object instances until withdrawn. Prevents house overloads (10k items no longer catastrophic). Faster saves/loads.

### Prerequisites for Virtualization
Items must be stackable, identical, no unique metadata/script dependencies. Weapons/tools with durability/slayer mods cannot virtualize.

### Integration with Crafting: Resource Providers
Smart crafting checks shelves like backpacks: processes from virtual stocks without object creation/destruction. Enables craft-from-shelf without packet storms.

### Mass Crafting Without Lag
Vanilla UO scans backpacks for resources; shelves use dictionary lookups, avoiding N-item iterations. Only output items create packets.

### Withdrawal Stacking Logic
Large stacks created on withdrawal; auto-merge with existing backpack stacks. Minimizes new object churn.

### Packet Efficiency
Container open: virtual shelves send ~200-byte summaries vs. 240kB for 12k items.

### Advanced Features
Loadouts for resupply automation, fallback settings (resource downgrades, food substitutions), integration with crates/bulk commodities, per-character persistence.
