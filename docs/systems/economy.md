# Economy

Crafting, BODs, housing, and gold sinks designed for healthy long-term economy.

## Currency Overview

| Currency | Binding | Source | Sink |
|----------|---------|--------|------|
| Gold | Character | PvM, trading | Vendors, repairs, housing |
| Silver | Character | Bounties, quests, VvV | Consumables, cosmetics |
| Faction Points | Account | Sieges, quests | Faction rewards |
| Tournament Coins | Character | Tournaments | Cosmetics |

## Crafting System

### Skill Progression

Traditional skill gain through practice. No shortcuts for trade skills.

| Skill | Products |
|-------|----------|
| Blacksmithy | Weapons, armor, tools |
| Tailoring | Cloth armor, bags |
| Tinkering | Jewelry, tools, talismans |
| Alchemy | Potions |
| Inscription | Scrolls, spellbooks |
| Carpentry | Furniture, containers |
| Bowcraft | Bows, arrows |
| Cooking | Food buffs |

### Talisman Crafting

Requires GM Tinkering + secondary skill.

| Talisman | Secondary Skill | Materials |
|----------|-----------------|-----------|
| Dexer | Blacksmithy | 10 Common, 3 Uncommon, 1 Rare relic |
| Tamer | Taming | 10 Common, 3 Uncommon, 1 Rare relic |
| Sampire | Chivalry | 10 Common, 3 Uncommon, 1 Rare relic |
| Treasure Hunter | Cartography | 10 Common, 3 Uncommon, 1 Rare relic |

---

## Bulk Order Deeds (BODs)

### Overview

ModernUO includes Publish 95 BOD system. 51alpha uses this with minor enhancements.

| Type | Profession |
|------|------------|
| Small BOD | Single item, 10-20 quantity |
| Large BOD | Collection of small BODs |

### Timing

| Event | Timing |
|-------|--------|
| BOD Request | Every 6 hours per profession |
| Points Credit | Immediate on turn-in |
| Cache Clear | 1 hour cooldown per NPC |

### Rewards

Points accumulated for:
- Higher-tier materials
- Special tools (runic)
- Rare crafting recipes
- Cosmetic items

---

## Housing

### Traditional Housing

ModernUO's standard housing system for player-placed homes.

| Feature | Status |
|---------|--------|
| Placement | Standard UO rules |
| Decay | 90 days without refresh |
| Security | Standard UO security |
| Taxes | ❌ Removed (design decision) |

### Static House Rentals

Pre-built structures in strategic locations for rent.

| Parameter | Value |
|-----------|-------|
| Weekly Rent | 10,000-50,000 gold |
| Max Lockdowns | 500 |
| Max Secures | 10 |
| Grace Period | 7 days after missed payment |

#### Rental Features

- Co-tenants: Up to 5 additional access
- Auto-renewal: Gold deducted from bank
- Eviction: Items moved to moving crate
- No decay: Admin-maintained structures

#### Rental Locations

Strategic positions near:
- Banks
- Dungeons
- Faction quest areas
- Trade hubs

---

## Gold Sinks

### Active Sinks

| Sink | Cost | Notes |
|------|------|-------|
| House Rent | 10-50k/week | Static rentals |
| Repairs | Variable | Armor/weapon durability |
| Vendor Fees | 1%/day | NPC vendor maintenance |
| Faction Defenses | Budget-limited | Traps, turrets |
| Gambling | Variable | Poker, duel betting |
| Cosmetics | 5-50k | Hair dye, name changes |

### Removed Sinks

| Sink | Reason |
|------|--------|
| House Taxes | Design decision - too punishing |
| Insurance | Complexity without gameplay value |

---

## Gambling

### Duel Pit Betting

- Location: Major cities
- Bet Range: 1,000 - 100,000 gold
- House Rake: 5%
- Both parties must agree to amount

### Texas Hold'em

- Tables in taverns
- Buy-in: 1,000 - 50,000 gold
- House Rake: 5% of pot
- 2-8 players per table

### Tournament Betting

- Bet on tournament participants
- Odds calculated from Glicko ratings
- House Rake: 5%
- Payout on match completion

---

## Trade Routes

### Material Bonuses (Rotating)

Each week, specific dungeons provide gathering bonuses:

| Dungeon | Bonus |
|---------|-------|
| Despise | +25% ore yield |
| Shame | +25% wood yield |
| Covetous | +25% leather yield |
| Deceit | +25% reagent drops |

Rotation announced via Town Cryer.

### AFK Resource Gathering

Monitored with anti-AFK checks:
- Random skill check prompts
- Movement pattern analysis
- Captcha after extended gathering

Violations: Warning → 24hr ban → Permanent ban

---

## Economy Monitoring

### Tracked Metrics

| Metric | Alert Threshold |
|--------|-----------------|
| Gold in circulation | +10% daily |
| Item duplication | Any duplicate serial |
| Trade volume | -50% from average |
| Inflation rate | +5% weekly |

### Admin Commands

```
[economy stats]     - Show economy dashboard
[economy audit]     - Run integrity check
[goldtrack <name>]  - Track player gold flow
```

---

## Weight Reduction Bags

Specialized containers for gatherers.

| Bag Type | Reduction | Capacity | Cost |
|----------|-----------|----------|------|
| Ore Bag | 50% | 400 stones | 25,000 gold |
| Log Bag | 50% | 400 stones | 25,000 gold |
| Leather Bag | 50% | 400 stones | 25,000 gold |
| Reagent Bag | 75% | 200 stones | 50,000 gold |

All bags:
- Blessed (cannot be stolen/looted)
- Single type only
- Purchased from Silver vendor

---

## Storage Shelves

Bank extension via furniture:

| Shelf | Slots | Cost |
|-------|-------|------|
| Small | 25 | 10,000 gold |
| Medium | 50 | 25,000 gold |
| Large | 100 | 50,000 gold |

Placed in player housing only. Contents accessible like bank box.

---

## Database Tables

| Table | Purpose |
|-------|---------|
| s51a_silver_balances | Silver currency |
| s51a_silver_log | Transaction audit |
| s51a_rental_buildings | Static house rentals |
| s51a_rental_payments | Rent history |
| s51a_rental_cotenants | Access control |
| s51a_rental_evictions | Eviction log |

---

## Configuration

```json
{
  "economy.house_rent_min": 10000,
  "economy.house_rent_max": 50000,
  "economy.gambling_rake": 0.05,
  "economy.vendor_fee_daily": 0.01,
  "economy.ore_bag_cost": 25000,
  "economy.ore_bag_reduction": 0.50,
  "economy.reagent_bag_cost": 50000,
  "economy.reagent_bag_reduction": 0.75
}
```
