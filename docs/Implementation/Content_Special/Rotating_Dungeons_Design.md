# Rotating Dungeons Deep Dive

## Overview
Weekly rotating dungeon system, eventually through all dungeons; some require groups. PvM points boost faction PvM total and personal leaderboard if in faction. Faction town NPC discounts (5,10,15,20%) based on leaderboard tiers.

## Algorithms and Logic
Rotation weekly; loot tables for points (changeable online). Leaderboard tiers calculated by points/rank, 4 tiers for competition/fairness. Group requirements for harder dungeons.

## Edge Cases
Rotation skips, group formations fails, points doubles/tampering. Pre/post faction changes discounts.

## Implementation Details
Spawner updates on rotation, persistent point tables for live editing. Link to faction engine for town NPC pricing.

## Testing Plan
Rotation tests, point calculations, tier assignments, discount verifications.

## Design Details from User Spec
Weekly rotation through all dungeons eventually, group reqs for some. Faction PvM boosts + personal points. NPC discounts 5-20% in faction towns by leaderboard 4 tiers. Algorithm for tiers by position/count for fairness/competition. Monthly/3 month resets? Points system like loot tables, changeable online.

## Change Log
Filled with weekly rotation, PvM/faction ties, tiered discounts.


Dungeons -
higher loot chance / increase gold % for 1 dungeon every week.
Rotating selected dungeon that increases gold/chance of rare loot. 
Creating a PvP environment because it’s the best place for loot this week.

At the same time lower the gold/chance of rare loot at another dungeon and dont allow PK rotated weekly as well.

announced by town cryer
