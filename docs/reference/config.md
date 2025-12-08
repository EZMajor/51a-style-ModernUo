# Configuration Reference

All tunable parameters for 51alpha systems.

## Configuration Sources

1. **File**: `Data/51alpha/config.json` (default values)
2. **Database**: `s51a_config` table (overrides file)
3. **Code**: `Sphere51aConfig.cs` (compiled defaults)

Priority: Database > File > Code

---

## Combat Settings

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `sphere.enabled` | bool | true | Enable Sphere-style combat |
| `combat.allow_movement_during_cast` | bool | true | Free movement while casting |
| `combat.fizzle_mana_rate` | decimal | 0.5 | Mana consumed on fizzle (0-1) |
| `combat.success_mana_rate` | decimal | 1.0 | Mana consumed on success |
| `combat.protection_fc_penalty` | int | 0 | FC penalty from Protection spell |

### Spell Interruption

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `spell.damage_interrupts` | bool | false | Damage interrupts casting |
| `spell.equip_interrupts` | bool | false | Equipping interrupts casting |
| `spell.movement_interrupts` | bool | false | Movement interrupts casting |
| `spell.warmode_interrupts` | bool | true | War mode toggle interrupts |
| `spell.bandage_interrupts` | bool | true | Bandaging interrupts casting |

### Scroll System

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `scroll.mana_reduction` | decimal | 0.43 | Mana cost reduction (43%) |
| `scroll.speed_bonus_seconds` | decimal | 0.5 | Cast speed bonus |
| `scroll.speed_min_circle` | int | 3 | Minimum circle for speed bonus |

### Combat Balance

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `combat.pvp_damage_modifier` | decimal | 1.0 | PvP damage multiplier |
| `combat.pet_pvp_speed_reduction` | decimal | 0.5 | Pet speed in PvP (50%) |

---

## Talisman Settings

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `combat.talisman_pvp_disable_minutes` | int | 5 | Duration of PvP disable |
| `talisman.dexer_damage_bonus` | decimal | 0.15 | Dexer damage bonus |
| `talisman.tamer_damage_bonus` | decimal | 0.15 | Tamer pet damage bonus |
| `talisman.sampire_leech_bonus` | decimal | 0.10 | Sampire life leech bonus |
| `talisman.th_quality_bonus` | decimal | 0.15 | TH chest quality bonus |

---

## Faction Settings

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `faction.change_cooldown_days` | int | 7 | Days between faction changes |
| `faction.season_length_days` | int | 90 | Season duration |
| `faction.carryover_percent` | decimal | 0.10 | Points kept after season |

### Underdog Bonuses

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `faction.first_place_bonus` | decimal | 0.00 | 1st place point bonus |
| `faction.second_place_bonus` | decimal | 0.02 | 2nd place point bonus |
| `faction.third_place_bonus` | decimal | 0.05 | 3rd place point bonus |

---

## Siege Settings

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `siege.duration_minutes` | int | 30 | Battle duration |
| `siege.victory_score` | int | 10000 | Points to win |
| `siege.global_cooldown_minutes` | int | 30 | Time between any siege |
| `siege.city_cooldown_minutes` | int | 120 | Time between same city |
| `siege.min_total_players` | int | 10 | Minimum to trigger |
| `siege.min_players_per_faction` | int | 3 | Recommended per faction |

### Objectives

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `siege.sigil_enabled` | bool | true | Enable sigil mechanic |
| `siege.altars_enabled` | bool | true | Enable altars |
| `siege.sigil_capture_seconds` | int | 30 | Time to capture sigil |
| `siege.sigil_contest_seconds` | int | 45 | Contested capture time |

### Scoring

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `siege.points_per_kill` | int | 100 | Base kill points |
| `siege.points_sigil_capture` | int | 500 | Sigil capture points |
| `siege.points_altar_tick` | int | 25 | Points per altar per 30s |
| `siege.silver_per_kill` | int | 10 | Base kill silver |

### Defenses

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `siege.defense_budget_max` | int | 5000 | Max silver per faction |
| `siege.trap_limit` | int | 10 | Max traps per faction |
| `siege.turret_limit` | int | 3 | Max turrets per faction |

---

## Town Control

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `town.single_city_discount` | decimal | 0.10 | Discount for 1-3 cities |
| `town.all_cities_discount` | decimal | 0.15 | Discount for 4 cities |

---

## Tournament Settings

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `tournament.min_players` | int | 6 | Minimum to start |
| `tournament.registration_minutes` | int | 15 | Registration window |
| `tournament.match_duration_seconds` | int | 300 | Match time limit |
| `tournament.prep_seconds` | int | 30 | Pre-match prep time |
| `tournament.arena_count` | int | 8 | Number of arenas |

### Rewards

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `tournament.coin_participation` | int | 50 | Coins for entering |
| `tournament.coin_per_win` | int | 100 | Coins per match win |
| `tournament.coin_champion` | int | 500 | Champion bonus |
| `tournament.coin_runner_up` | int | 250 | Runner-up bonus |
| `tournament.title_duration_days` | int | 7 | Champion title duration |

### Betting

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `tournament.betting_enabled` | bool | true | Allow spectator betting |
| `tournament.betting_rake` | decimal | 0.05 | House take (5%) |
| `tournament.bet_minimum` | int | 1000 | Minimum bet |

---

## Daily Content

### Bounties

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `bounty.reset_hour_utc` | int | 5 | Daily reset hour (UTC) |
| `bounty.tasks_per_day` | int | 3 | Number of daily tasks |

### Faction Quest

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `factionquest.boss_base_hp` | int | 5000 | Boss base HP |
| `factionquest.boss_hp_per_player` | int | 500 | HP added per player |
| `factionquest.boss_max_hp` | int | 25000 | Maximum HP |
| `factionquest.respawn_hours` | int | 2 | Hours until respawn |
| `factionquest.reward_faction_points` | int | 500 | FP for completion |
| `factionquest.reward_bonus_top_damage` | int | 250 | Bonus for top damage |

### Relic Drops

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `factionquest.relic_common_chance` | decimal | 0.15 | Common drop rate |
| `factionquest.relic_uncommon_chance` | decimal | 0.05 | Uncommon drop rate |
| `factionquest.relic_rare_chance` | decimal | 0.01 | Rare drop rate |

---

## Glicko-2 Settings

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `glicko.default_rating` | decimal | 1500 | Starting rating |
| `glicko.default_rd` | decimal | 350 | Starting deviation |
| `glicko.default_volatility` | decimal | 0.06 | Starting volatility |
| `glicko.tau` | decimal | 0.5 | System constant |
| `glicko.rating_period_days` | int | 3 | Rating period length |
| `glicko.min_games_for_leaderboard` | int | 10 | Games to appear |

---

## New Player Experience

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `young.duration_days` | int | 14 | Protection duration |
| `young.can_attack_players` | bool | false | Can attack others |
| `young.can_be_attacked` | bool | false | Can be attacked |
| `young.can_join_faction` | bool | false | Can join factions |
| `young.can_enter_dungeon` | bool | false | Dungeon access |

---

## Economy

### Housing

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `economy.house_rent_min` | int | 10000 | Minimum weekly rent |
| `economy.house_rent_max` | int | 50000 | Maximum weekly rent |
| `economy.rent_grace_days` | int | 7 | Days before eviction |

### Gambling

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `economy.gambling_enabled` | bool | true | Enable gambling |
| `economy.gambling_rake` | decimal | 0.05 | House take (5%) |
| `economy.duel_bet_min` | int | 1000 | Minimum duel bet |
| `economy.duel_bet_max` | int | 100000 | Maximum duel bet |

### Vendors

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `economy.vendor_fee_daily` | decimal | 0.01 | Daily vendor fee (1%) |

### Special Items

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `economy.ore_bag_cost` | int | 25000 | Ore bag price |
| `economy.ore_bag_reduction` | decimal | 0.50 | Weight reduction |
| `economy.reagent_bag_cost` | int | 50000 | Reagent bag price |
| `economy.reagent_bag_reduction` | decimal | 0.75 | Weight reduction |

---

## Combat Status

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `combat_status.cooldown_hours` | int | 24 | Toggle cooldown |
| `combat_status.forced_duration_hours` | int | 24 | Forced flag duration |

---

## Database

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `database.connection_string` | string | (none) | PostgreSQL connection |
| `database.pool_min` | int | 5 | Min pool size |
| `database.pool_max` | int | 100 | Max pool size |

---

## Example config.json

```json
{
  "sphere.enabled": true,
  "database.connection_string": "Host=localhost;Database=51alpha;Username=51alpha_app;Password=secure;Pooling=true;MinPoolSize=5;MaxPoolSize=100",
  
  "combat.fizzle_mana_rate": 0.5,
  "combat.protection_fc_penalty": 0,
  "combat.talisman_pvp_disable_minutes": 5,
  
  "faction.change_cooldown_days": 7,
  "faction.season_length_days": 90,
  
  "siege.duration_minutes": 30,
  "siege.victory_score": 10000,
  "siege.min_total_players": 10,
  
  "tournament.min_players": 6,
  "tournament.match_duration_seconds": 300,
  
  "young.duration_days": 14,
  
  "glicko.default_rating": 1500,
  "glicko.default_rd": 350,
  
  "bounty.reset_hour_utc": 5
}
```

---

## Runtime Changes

Config values in database override file values and can be changed without restart:

```sql
-- Update a value
UPDATE s51a_config 
SET config_value = '600', updated_at = NOW() 
WHERE config_key = 'tournament.match_duration_seconds';

-- Reload in-game
[s51a reload config]
```
