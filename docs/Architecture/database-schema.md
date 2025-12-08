# Database Schema

PostgreSQL 15+ schema for 51alpha custom systems. Core UO data (items, mobiles, skills) remains in ModernUO binary saves.

## Connection

```json
{
  "ConnectionStrings": {
    "51alpha": "Host=localhost;Database=51alpha;Username=51alpha_app;Password=<secure>;Pooling=true;MinPoolSize=5;MaxPoolSize=100"
  }
}
```

## Table Overview

| Category | Tables | Purpose |
|----------|--------|---------|
| Core | 4 | Factions, seasons, guilds, combat status |
| Currency | 5 | Faction points, silver, tournament coins + logs |
| Rating | 2 | Glicko-2 ratings and match history |
| Sieges | 4 | Town control, battles, participants, kills |
| Tournaments | 3 | Events, participants, matches |
| Daily Content | 6 | Bounties, faction quests, progress |
| NPE | 2 | Young players, quest progress |
| Statistics | 1 | Lifetime PvP stats |
| Housing | 4 | Static rentals, payments, co-tenants |
| Config | 3 | Runtime settings, siege cities |
| Events | 1 | Telemetry log |

**Total: 31 tables**

---

## Core Tables

```sql
-- Faction definitions (seed data)
CREATE TABLE s51a_factions (
    faction_id SMALLINT PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    color_hue INT NOT NULL,
    banner_item_id INT NOT NULL,
    robe_hue INT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

INSERT INTO s51a_factions VALUES
(1, 'Vampire', 0x21, 0x1627, 0x21),
(2, 'Daemon', 0x30, 0x1628, 0x30),
(3, 'Goblin', 0x3F, 0x1629, 0x3F);

-- Quarterly seasons
CREATE TABLE s51a_seasons (
    season_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    is_active BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Guild faction assignment
CREATE TABLE s51a_guild_factions (
    guild_serial BIGINT PRIMARY KEY,
    faction_id SMALLINT NOT NULL REFERENCES s51a_factions(faction_id),
    joined_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    change_cooldown_until TIMESTAMP WITH TIME ZONE
);

-- Player combatant/peaceful status
CREATE TABLE s51a_player_combat_status (
    player_serial BIGINT PRIMARY KEY,
    is_combatant BOOLEAN DEFAULT TRUE,
    status_changed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    cooldown_until TIMESTAMP WITH TIME ZONE,
    forced_combatant_until TIMESTAMP WITH TIME ZONE
);
```

---

## Currency & Points

```sql
-- Faction points (account-bound)
CREATE TABLE s51a_faction_points (
    account_name VARCHAR(100) NOT NULL,
    season_id INT NOT NULL REFERENCES s51a_seasons(season_id),
    faction_id SMALLINT NOT NULL REFERENCES s51a_factions(faction_id),
    points BIGINT DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    PRIMARY KEY (account_name, season_id)
);

-- Faction point audit log
CREATE TABLE s51a_faction_point_log (
    log_id BIGSERIAL PRIMARY KEY,
    account_name VARCHAR(100) NOT NULL,
    player_serial BIGINT NOT NULL,
    season_id INT NOT NULL,
    faction_id SMALLINT NOT NULL,
    points_delta INT NOT NULL,
    reason VARCHAR(200) NOT NULL,
    correlation_id VARCHAR(50),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Silver (character-bound)
CREATE TABLE s51a_silver_balances (
    player_serial BIGINT PRIMARY KEY,
    balance INT DEFAULT 0 CHECK (balance >= 0),
    lifetime_earned BIGINT DEFAULT 0,
    lifetime_spent BIGINT DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE s51a_silver_log (
    log_id BIGSERIAL PRIMARY KEY,
    player_serial BIGINT NOT NULL,
    silver_delta INT NOT NULL,
    reason VARCHAR(200) NOT NULL,
    correlation_id VARCHAR(50),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Tournament coins (character-bound)
CREATE TABLE s51a_tournament_coins (
    player_serial BIGINT PRIMARY KEY,
    balance INT DEFAULT 0 CHECK (balance >= 0),
    lifetime_earned BIGINT DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## Glicko-2 Rating

```sql
CREATE TABLE s51a_glicko_ratings (
    player_serial BIGINT PRIMARY KEY,
    rating DECIMAL(10,2) DEFAULT 1500.00,
    rating_deviation DECIMAL(10,2) DEFAULT 350.00,
    volatility DECIMAL(10,6) DEFAULT 0.060000,
    games_played INT DEFAULT 0,
    last_game_at TIMESTAMP WITH TIME ZONE,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    CONSTRAINT valid_rating CHECK (rating BETWEEN 100 AND 4000),
    CONSTRAINT valid_rd CHECK (rating_deviation BETWEEN 30 AND 500)
);

CREATE TABLE s51a_glicko_matches (
    match_id BIGSERIAL PRIMARY KEY,
    winner_serial BIGINT NOT NULL,
    loser_serial BIGINT NOT NULL,
    match_type VARCHAR(20) NOT NULL,  -- 'siege_kill', 'tournament', 'duel'
    winner_rating_before DECIMAL(10,2) NOT NULL,
    winner_rating_after DECIMAL(10,2) NOT NULL,
    loser_rating_before DECIMAL(10,2) NOT NULL,
    loser_rating_after DECIMAL(10,2) NOT NULL,
    correlation_id VARCHAR(50),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## Town Control & Sieges

```sql
CREATE TABLE s51a_town_control (
    city_name VARCHAR(50) PRIMARY KEY,
    controlling_faction SMALLINT REFERENCES s51a_factions(faction_id),
    controlled_since TIMESTAMP WITH TIME ZONE,
    sieges_defended INT DEFAULT 0,
    total_sieges INT DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

INSERT INTO s51a_town_control (city_name) VALUES
('Jhelom'), ('Skara Brae'), ('Yew'), ('Trinsic');

CREATE TABLE s51a_siege_battles (
    battle_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    city_name VARCHAR(50) NOT NULL,
    started_at TIMESTAMP WITH TIME ZONE NOT NULL,
    ended_at TIMESTAMP WITH TIME ZONE,
    vampire_score INT DEFAULT 0,
    daemon_score INT DEFAULT 0,
    goblin_score INT DEFAULT 0,
    winner_faction SMALLINT REFERENCES s51a_factions(faction_id),
    victory_type VARCHAR(20),
    vampire_participants INT DEFAULT 0,
    daemon_participants INT DEFAULT 0,
    goblin_participants INT DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE s51a_siege_participants (
    battle_id UUID NOT NULL REFERENCES s51a_siege_battles(battle_id),
    player_serial BIGINT NOT NULL,
    faction_id SMALLINT NOT NULL,
    kills INT DEFAULT 0,
    deaths INT DEFAULT 0,
    assists INT DEFAULT 0,
    sigils_captured INT DEFAULT 0,
    altars_captured INT DEFAULT 0,
    faction_points_earned INT DEFAULT 0,
    silver_earned INT DEFAULT 0,
    joined_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    PRIMARY KEY (battle_id, player_serial)
);

CREATE TABLE s51a_siege_kills (
    kill_id BIGSERIAL PRIMARY KEY,
    battle_id UUID NOT NULL REFERENCES s51a_siege_battles(battle_id),
    killer_serial BIGINT NOT NULL,
    killer_faction SMALLINT NOT NULL,
    victim_serial BIGINT NOT NULL,
    victim_faction SMALLINT NOT NULL,
    points_awarded INT NOT NULL,
    silver_awarded INT NOT NULL,
    glicko_multiplier DECIMAL(4,2) DEFAULT 1.00,
    kill_source VARCHAR(20) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## Tournaments

```sql
CREATE TABLE s51a_tournaments (
    tournament_id SERIAL PRIMARY KEY,
    started_at TIMESTAMP WITH TIME ZONE NOT NULL,
    ended_at TIMESTAMP WITH TIME ZONE,
    status VARCHAR(20) DEFAULT 'registration',
    winner_serial BIGINT,
    runner_up_serial BIGINT,
    participant_count INT DEFAULT 0,
    total_rounds INT DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE s51a_tournament_participants (
    tournament_id INT NOT NULL REFERENCES s51a_tournaments(tournament_id),
    player_serial BIGINT NOT NULL,
    seed_position INT,
    final_position INT,
    wins INT DEFAULT 0,
    losses INT DEFAULT 0,
    coins_earned INT DEFAULT 0,
    registered_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    eliminated_at TIMESTAMP WITH TIME ZONE,
    PRIMARY KEY (tournament_id, player_serial)
);

CREATE TABLE s51a_tournament_matches (
    match_id SERIAL PRIMARY KEY,
    tournament_id INT NOT NULL REFERENCES s51a_tournaments(tournament_id),
    round_number INT NOT NULL,
    match_number INT NOT NULL,
    arena_id INT NOT NULL,
    player1_serial BIGINT NOT NULL,
    player2_serial BIGINT NOT NULL,
    winner_serial BIGINT,
    win_reason VARCHAR(20),
    started_at TIMESTAMP WITH TIME ZONE,
    ended_at TIMESTAMP WITH TIME ZONE,
    duration_seconds INT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## Daily Content

```sql
-- Bounty task definitions
CREATE TABLE s51a_bounty_definitions (
    bounty_id SERIAL PRIMARY KEY,
    bounty_type VARCHAR(20) NOT NULL,  -- 'monster', 'resource', 'activity'
    target_name VARCHAR(100) NOT NULL,
    target_type VARCHAR(100),
    required_count INT NOT NULL,
    faction_point_reward INT NOT NULL,
    silver_reward INT NOT NULL,
    difficulty VARCHAR(10) DEFAULT 'medium',
    is_active BOOLEAN DEFAULT TRUE
);

-- Daily bounty schedule (3 tasks per day)
CREATE TABLE s51a_daily_bounties (
    bounty_date DATE PRIMARY KEY,
    task1_bounty_id INT REFERENCES s51a_bounty_definitions(bounty_id),
    task2_bounty_id INT REFERENCES s51a_bounty_definitions(bounty_id),
    task3_bounty_id INT REFERENCES s51a_bounty_definitions(bounty_id),
    generated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Player bounty progress
CREATE TABLE s51a_player_bounty_progress (
    player_serial BIGINT NOT NULL,
    bounty_date DATE NOT NULL,
    task1_progress INT DEFAULT 0,
    task1_claimed BOOLEAN DEFAULT FALSE,
    task2_progress INT DEFAULT 0,
    task2_claimed BOOLEAN DEFAULT FALSE,
    task3_progress INT DEFAULT 0,
    task3_claimed BOOLEAN DEFAULT FALSE,
    PRIMARY KEY (player_serial, bounty_date)
);

-- Faction quest locations
CREATE TABLE s51a_faction_quest_locations (
    location_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    region_type VARCHAR(20) NOT NULL,  -- 'swamp', 'desert'
    spawn_x INT NOT NULL,
    spawn_y INT NOT NULL,
    spawn_z INT NOT NULL,
    map_id INT DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE
);

-- Daily faction quest schedule
CREATE TABLE s51a_faction_quest_schedule (
    quest_date DATE PRIMARY KEY,
    location_id INT NOT NULL,
    location_name VARCHAR(100) NOT NULL,
    boss_type VARCHAR(50) NOT NULL,
    spawn_x INT NOT NULL,
    spawn_y INT NOT NULL,
    spawn_z INT NOT NULL,
    map_id INT DEFAULT 0
);

-- Quest completions
CREATE TABLE s51a_faction_quest_completions (
    player_serial BIGINT NOT NULL,
    quest_date DATE NOT NULL,
    faction_points_earned INT NOT NULL,
    completed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    PRIMARY KEY (player_serial, quest_date)
);
```

---

## New Player Experience

```sql
CREATE TABLE s51a_young_players (
    player_serial BIGINT PRIMARY KEY,
    account_name VARCHAR(100) NOT NULL,
    started_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
    renounced BOOLEAN DEFAULT FALSE,
    quests_completed INT DEFAULT 0,
    total_play_time_minutes INT DEFAULT 0
);

CREATE TABLE s51a_starter_quest_progress (
    player_serial BIGINT NOT NULL,
    quest_id VARCHAR(50) NOT NULL,
    status VARCHAR(20) DEFAULT 'not_started',
    progress_data JSONB,
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    PRIMARY KEY (player_serial, quest_id)
);
```

---

## Statistics

```sql
CREATE TABLE s51a_pvp_stats (
    player_serial BIGINT PRIMARY KEY,
    total_kills INT DEFAULT 0,
    total_deaths INT DEFAULT 0,
    total_assists INT DEFAULT 0,
    kill_streak_best INT DEFAULT 0,
    sieges_participated INT DEFAULT 0,
    sieges_won INT DEFAULT 0,
    sigils_captured INT DEFAULT 0,
    tournaments_entered INT DEFAULT 0,
    tournaments_won INT DEFAULT 0,
    tournament_matches_won INT DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## Static House Rentals

```sql
CREATE TABLE s51a_rental_buildings (
    building_id SERIAL PRIMARY KEY,
    house_serial BIGINT NOT NULL UNIQUE,
    building_name VARCHAR(100) NOT NULL,
    location_x INT NOT NULL,
    location_y INT NOT NULL,
    location_z INT NOT NULL,
    map_id INT DEFAULT 0,
    weekly_rent_price INT NOT NULL DEFAULT 10000,
    max_lockdowns INT NOT NULL DEFAULT 500,
    max_secures INT NOT NULL DEFAULT 10,
    is_enabled BOOLEAN DEFAULT TRUE,
    renter_serial BIGINT,
    renter_account VARCHAR(100),
    rental_expires TIMESTAMP WITH TIME ZONE,
    status VARCHAR(20) DEFAULT 'Available',
    total_rent_collected BIGINT DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE s51a_rental_payments (
    payment_id BIGSERIAL PRIMARY KEY,
    building_id INT NOT NULL REFERENCES s51a_rental_buildings(building_id),
    renter_serial BIGINT NOT NULL,
    amount INT NOT NULL,
    payment_type VARCHAR(20) NOT NULL,
    paid_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE s51a_rental_cotenants (
    building_id INT NOT NULL REFERENCES s51a_rental_buildings(building_id),
    cotenant_serial BIGINT NOT NULL,
    added_by_serial BIGINT NOT NULL,
    added_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    PRIMARY KEY (building_id, cotenant_serial)
);

CREATE TABLE s51a_rental_evictions (
    eviction_id BIGSERIAL PRIMARY KEY,
    building_id INT NOT NULL,
    renter_serial BIGINT NOT NULL,
    reason VARCHAR(50) NOT NULL,
    items_cleared INT DEFAULT 0,
    evicted_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## Configuration

```sql
-- Runtime config (no restart needed)
CREATE TABLE s51a_config (
    config_key VARCHAR(100) PRIMARY KEY,
    config_value TEXT NOT NULL,
    value_type VARCHAR(20) DEFAULT 'string',
    description TEXT,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Siege city definitions
CREATE TABLE s51a_siege_cities (
    city_name VARCHAR(50) PRIMARY KEY,
    is_active BOOLEAN DEFAULT TRUE,
    region_x1 INT NOT NULL,
    region_y1 INT NOT NULL,
    region_x2 INT NOT NULL,
    region_y2 INT NOT NULL,
    map_id INT DEFAULT 0,
    sigil_spawn JSONB NOT NULL,
    altar_spawns JSONB NOT NULL,
    priest_spawns JSONB NOT NULL
);

-- Telemetry events
CREATE TABLE s51a_events (
    event_id BIGSERIAL PRIMARY KEY,
    event_type VARCHAR(50) NOT NULL,
    player_serial BIGINT,
    event_data JSONB,
    correlation_id VARCHAR(50),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_events_type ON s51a_events(event_type, created_at DESC);
CREATE INDEX idx_events_player ON s51a_events(player_serial, created_at DESC);
```

---

## Key Config Values (Seed Data)

```sql
INSERT INTO s51a_config (config_key, config_value, value_type, description) VALUES
-- Sieges
('siege.duration_minutes', '30', 'int', 'Battle duration'),
('siege.victory_score', '10000', 'int', 'Score to win'),
('siege.min_total_players', '10', 'int', 'Min players to trigger'),

-- Tournaments  
('tournament.min_players', '6', 'int', 'Min to start'),
('tournament.match_duration_seconds', '300', 'int', 'Match time limit'),
('tournament.coin_reward_champion', '500', 'int', 'Winner bonus'),

-- Faction quests
('factionquest.boss_base_hp', '5000', 'int', 'Base HP'),
('factionquest.boss_hp_per_player', '500', 'int', 'HP per player'),
('factionquest.reward_faction_points', '500', 'int', 'FP reward'),

-- Glicko
('glicko.default_rating', '1500', 'decimal', 'Starting rating'),
('glicko.default_rd', '350', 'decimal', 'Starting RD'),

-- Young players
('young.duration_days', '14', 'int', 'Protection duration'),

-- Faction changes
('faction.change_cooldown_days', '7', 'int', 'Days between changes');
```

---

## Index Summary

Key performance indexes included in table definitions above. Additional indexes for reporting:

```sql
CREATE INDEX idx_faction_points_ranking ON s51a_faction_points(season_id, points DESC);
CREATE INDEX idx_glicko_leaderboard ON s51a_glicko_ratings(rating DESC) WHERE games_played >= 10;
CREATE INDEX idx_siege_city ON s51a_siege_battles(city_name, started_at DESC);
CREATE INDEX idx_rental_status ON s51a_rental_buildings(status);
```
