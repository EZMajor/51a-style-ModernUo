# Content

Dungeons, daily bounties, faction quests, tournaments, and new player experience.

## Dungeon System

### Available Dungeons

| Dungeon | Access | Relic Type |
|---------|--------|------------|
| Despise | Always | Combat |
| Shame | Always | Combat |
| Covetous | Always | Combat |
| Deceit | Always | Arcane |
| Destard | Always | Dragon |
| Hythloth | Always | Demonic |
| Wrong | Always | Undead |
| Fire | Always | Elemental |
| Ice | Always | Elemental |

### Rotating Bonuses

Weekly rotation of dungeon bonuses:

| Bonus Type | Effect |
|------------|--------|
| Relic Drop | +50% relic drop rate |
| Material | +25% ore/wood/leather yield |
| Gold | +25% gold drops |

Announced via Town Cryer at reset.

### Relic Drop Rates

| Relic Tier | Base Rate | Bonus Rate |
|------------|-----------|------------|
| Common | 5% | 7.5% |
| Uncommon | 1% | 1.5% |
| Rare | 0.1% | 0.15% |

Drop on boss kills and rare spawns.

### Dungeon Rules

| Rule | Details |
|------|---------|
| Mounts | Auto-dismissed on entry |
| Pets | Allowed |
| PvP | Full (Felucca rules) |
| Young Players | Cannot enter |

---

## Daily Bounties

### Overview

Three random tasks refresh daily at midnight EST (5 AM UTC).

### Bounty Types

| Type | Examples | Difficulty |
|------|----------|------------|
| Monster | Kill 20 Skeletons | Easy |
| Monster | Kill 5 Drakes | Hard |
| Resource | Mine 100 Iron Ore | Easy |
| Resource | Mine 25 Valorite Ore | Hard |
| Activity | Craft 50 Bandages | Easy |
| Activity | Craft 20 Potions | Medium |

### Rewards

| Difficulty | Faction Points | Silver |
|------------|----------------|--------|
| Easy | 100 | 25 |
| Medium | 150 | 50 |
| Hard | 200-250 | 75-100 |

### Tracking

Progress tracked automatically:
- Monster kills credited on death
- Resources credited on harvest
- Crafts credited on completion

Claim rewards at Bounty Board NPC.

---

## Faction Quest

### Overview

Daily boss spawn in rotating location. All factions compete for kill credit.

### Locations

| Region | Location |
|--------|----------|
| Swamp | Fens of the Dead |
| Swamp | Bog of Desolation |
| Desert | Scorched Sands |
| Desert | Sun Temple Ruins |

Rotation: Random selection daily.

### Boss Scaling

| Parameter | Value |
|-----------|-------|
| Base HP | 5,000 |
| HP per player | +500 |
| Maximum HP | 25,000 |
| Respawn | 2 hours after kill |

### Rewards

| Reward | Amount | Recipient |
|--------|--------|-----------|
| Faction Points | 500 | All participants |
| Bonus FP | 250 | Top damage dealer |
| Common Relic | 15% | Random participant |
| Uncommon Relic | 5% | Random participant |
| Rare Relic | 1% | Random participant |

Participation requirement: Minimum 1000 damage dealt.

---

## Tournament System

### Schedule

| Day | Time (EST) |
|-----|------------|
| Wednesday | 2 PM, 8 PM |
| Saturday | 2 PM, 8 PM |
| Sunday | 2 PM, 8 PM |

### Parameters

| Parameter | Value |
|-----------|-------|
| Registration | 15 minutes |
| Minimum Players | 6 |
| Match Duration | 5 minutes |
| Prep Time | 30 seconds |

### Format

- Single elimination bracket
- Random seeding (no Glicko weight)
- Powers of 2 brackets (byes for odd counts)
- Best of 1 per round

### Arenas

8 identical arenas for parallel matches:
- 20x20 tile fighting area
- Resurrection shrine outside
- Spectator balcony

### Rewards

| Placement | Tournament Coins |
|-----------|------------------|
| Participation | 50 |
| Per Win | 100 |
| Champion | 500 bonus |
| Runner-up | 250 bonus |

### Tournament Coins

Spent at Cosmetic Vendor:
- Unique dyes
- Special titles
- Trophy furniture
- Emotes

### Champion Perks

| Perk | Duration |
|------|----------|
| Title: "Champion" | 7 days |
| Trophy Item | Permanent |
| Leaderboard entry | Permanent |

---

## New Player Experience

### Young Player Protection

| Protection | Details |
|------------|---------|
| Duration | 14 days |
| Cannot attack | Other players |
| Cannot be attacked | By players |
| Cannot join | Factions |
| Dungeon access | ❌ Blocked |

### Renouncing Young Status

Players can renounce early via:
- New Player Guide NPC
- Command: `[renounceyoung]`

Cannot be undone.

### Starter Islands

5 themed training islands accessed via Ferry NPC:

| Island | Skills Taught |
|--------|---------------|
| Combat Basics | Swords, Tactics |
| Magic Fundamentals | Magery, Eval Int |
| Healing Arts | Healing, Anatomy |
| Stealth & Cunning | Hiding, Stealth |
| Advanced Combat | Resist, Meditation |

### Quest Structure

Each island:
1. NPC introduction
2. Training dummies/targets
3. Skill challenges
4. Completion reward (GM skill)

Total time: 2-3 hours for all islands.

### New Player Guide NPC

Location: Britain Bank (Trammel)

Services:
- Help menu with commands
- Wiki link
- Renounce young status
- Teleport to training islands

---

## Town Cryer

### Locations

- Britain (central)
- Trinsic (south)
- Moonglow (mage quarter)
- Skara Brae (west)
- Jhelom (arena)

### Announcement Types

| Type | Trigger |
|------|---------|
| Dungeon Bonus | Weekly rotation |
| Siege Starting | Battle begins |
| Tournament | Registration opens |
| Faction Quest | Boss spawns |
| Server Event | Admin triggered |

### Parameters

| Parameter | Value |
|-----------|-------|
| Range | 15 tiles |
| Cooldown | 10 minutes per player |
| Overlap | Prevented (one at a time) |

---

## Events

### Graveyard Liche King

Seasonal event (Halloween):
- Location: Britain Graveyard
- Boss: Liche King with unique mechanics
- Rewards: Exclusive Halloween items

### Swamp & Desert Event

World boss rotation:
- Alternates between swamp/desert
- Scaling difficulty
- Faction quest integration

---

## Database Tables

| Table | Purpose |
|-------|---------|
| s51a_bounty_definitions | Bounty task pool |
| s51a_daily_bounties | Daily selection |
| s51a_player_bounty_progress | Player tracking |
| s51a_faction_quest_locations | Quest spawn points |
| s51a_faction_quest_schedule | Daily location |
| s51a_faction_quest_completions | Completion log |
| s51a_tournaments | Tournament events |
| s51a_tournament_participants | Registrations |
| s51a_tournament_matches | Match results |
| s51a_young_players | NPE tracking |
| s51a_starter_quest_progress | Quest progress |

---

## Configuration

```json
{
  "bounty.reset_hour_utc": 5,
  "bounty.tasks_per_day": 3,
  
  "factionquest.boss_base_hp": 5000,
  "factionquest.boss_hp_per_player": 500,
  "factionquest.boss_max_hp": 25000,
  "factionquest.respawn_hours": 2,
  "factionquest.reward_faction_points": 500,
  "factionquest.relic_common_chance": 0.15,
  "factionquest.relic_uncommon_chance": 0.05,
  "factionquest.relic_rare_chance": 0.01,
  
  "tournament.min_players": 6,
  "tournament.registration_minutes": 15,
  "tournament.match_duration_seconds": 300,
  "tournament.prep_seconds": 30,
  "tournament.coin_participation": 50,
  "tournament.coin_per_win": 100,
  "tournament.coin_champion": 500,
  
  "young.duration_days": 14
}
```
