# 51alpha Database Schema
## PostgreSQL Schema for Custom Systems

### Document Metadata
- **Version**: v1.0.0
- **Last Updated**: 2025-01-02
- **Database**: PostgreSQL 15+
- **Purpose**: All 51alpha custom system data (not core UO data)

---

## 1. Overview

This schema covers all 51alpha-specific data stored in PostgreSQL. Core UO data (items, mobiles, skills, houses) remains in ModernUO's binary save system.

### Schema Namespace
All tables use the `s51a_` prefix to avoid conflicts.

### Connection Configuration
```json
{
  "ConnectionStrings": {
    "51alpha": "Host=localhost;Database=51alpha;Username=51alpha_app;Password=<secure>;Pooling=true;MinPoolSize=5;MaxPoolSize=100"
  }
}
```

---

## 2. Core Tables

### 2.1 Faction Definition
```sql
-- Static faction data (rarely changes)
CREATE TABLE s51a_factions (
    faction_id SMALLINT PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    color_hue INT NOT NULL,
    banner_item_id INT NOT NULL,
    robe_hue INT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Seed data
INSERT INTO s51a_factions (faction_id, name, color_hue, banner_item_id, robe_hue) VALUES
(1, 'Vampire', 0x21, 0x1627, 0x21),   -- Red
(2, 'Daemon', 0x30, 0x1628, 0x30),    -- Orange  
(3, 'Goblin', 0x3F, 0x1629, 0x3F);    -- Green
```

### 2.2 Season Management
```sql
-- Quarterly seasons
CREATE TABLE s51a_seasons (
    season_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    is_active BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_seasons_active ON s51a_seasons(is_active) WHERE is_active = TRUE;

-- Example: Create seasons
-- INSERT INTO s51a_seasons (name, start_date, end_date, is_active) VALUES
-- ('Season 1 - Q1 2025', '2025-01-01', '2025-03-31', TRUE);
```

### 2.3 Guild Faction Assignment
```sql
-- Links ModernUO guilds to factions
CREATE TABLE s51a_guild_factions (
    guild_serial BIGINT PRIMARY KEY,  -- ModernUO Guild.Serial
    faction_id SMALLINT NOT NULL REFERENCES s51a_factions(faction_id),
    joined_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_changed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    change_cooldown_until TIMESTAMP WITH TIME ZONE,  -- 7-day cooldown
    
    CONSTRAINT valid_faction CHECK (faction_id BETWEEN 1 AND 3)
);

CREATE INDEX idx_guild_factions_faction ON s51a_guild_factions(faction_id);
```

### 2.4 Player Combat Status
```sql
-- Combatant vs Peaceful status per player
CREATE TABLE s51a_player_combat_status (
    player_serial BIGINT PRIMARY KEY,  -- ModernUO PlayerMobile.Serial
    is_combatant BOOLEAN DEFAULT TRUE,
    status_changed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    cooldown_until TIMESTAMP WITH TIME ZONE,  -- 24-hour cooldown
    forced_combatant_until TIMESTAMP WITH TIME ZONE  -- Auto-flag from attacking
);
```

---

## 3. Currency & Points

### 3.1 Faction Points (Account-Bound)
```sql
-- Faction points are account-bound, not character-bound
CREATE TABLE s51a_faction_points (
    account_name VARCHAR(100) NOT NULL,
    season_id INT NOT NULL REFERENCES s51a_seasons(season_id),
    faction_id SMALLINT NOT NULL REFERENCES s51a_factions(faction_id),
    points BIGINT DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    PRIMARY KEY (account_name, season_id)
);

CREATE INDEX idx_faction_points_season ON s51a_faction_points(season_id, faction_id);
CREATE INDEX idx_faction_points_ranking ON s51a_faction_points(season_id, points DESC);
```

### 3.2 Faction Point Transactions (Audit Log)
```sql
-- All point changes logged for auditing
CREATE TABLE s51a_faction_point_log (
    log_id BIGSERIAL PRIMARY KEY,
    account_name VARCHAR(100) NOT NULL,
    player_serial BIGINT NOT NULL,
    season_id INT NOT NULL,
    faction_id SMALLINT NOT NULL,
    points_delta INT NOT NULL,  -- Can be negative
    reason VARCHAR(200) NOT NULL,
    correlation_id VARCHAR(50),  -- Links to siege/tournament/bounty
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_fp_log_account ON s51a_faction_point_log(account_name, created_at DESC);
CREATE INDEX idx_fp_log_correlation ON s51a_faction_point_log(correlation_id) WHERE correlation_id IS NOT NULL;

-- Partition by month for performance (optional)
-- CREATE TABLE s51a_faction_point_log_2025_01 PARTITION OF s51a_faction_point_log
--     FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

### 3.3 Silver Currency (Character-Bound)
```sql
-- Silver is character-bound (tradeable in-game but tracked per character)
CREATE TABLE s51a_silver_balances (
    player_serial BIGINT PRIMARY KEY,
    balance INT DEFAULT 0 CHECK (balance >= 0),
    lifetime_earned BIGINT DEFAULT 0,
    lifetime_spent BIGINT DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### 3.4 Silver Transactions (Audit Log)
```sql
CREATE TABLE s51a_silver_log (
    log_id BIGSERIAL PRIMARY KEY,
    player_serial BIGINT NOT NULL,
    silver_delta INT NOT NULL,
    reason VARCHAR(200) NOT NULL,
    correlation_id VARCHAR(50),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_silver_log_player ON s51a_silver_log(player_serial, created_at DESC);
```

### 3.5 Tournament Coins (Character-Bound)
```sql
CREATE TABLE s51a_tournament_coins (
    player_serial BIGINT PRIMARY KEY,
    balance INT DEFAULT 0 CHECK (balance >= 0),
    lifetime_earned BIGINT DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## 4. Glicko-2 Rating System

### 4.1 Player Ratings
```sql
CREATE TABLE s51a_glicko_ratings (
    player_serial BIGINT PRIMARY KEY,
    rating DECIMAL(10,2) DEFAULT 1500.00,
    rating_deviation DECIMAL(10,2) DEFAULT 350.00,
    volatility DECIMAL(10,6) DEFAULT 0.060000,
    games_played INT DEFAULT 0,
    last_game_at TIMESTAMP WITH TIME ZONE,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Constraints for valid Glicko values
    CONSTRAINT valid_rating CHECK (rating BETWEEN 100 AND 4000),
    CONSTRAINT valid_rd CHECK (rating_deviation BETWEEN 30 AND 500),
    CONSTRAINT valid_volatility CHECK (volatility BETWEEN 0.01 AND 0.15)
);

CREATE INDEX idx_glicko_leaderboard ON s51a_glicko_ratings(rating DESC) WHERE games_played >= 10;
```

### 4.2 Match History
```sql
CREATE TABLE s51a_glicko_matches (
    match_id BIGSERIAL PRIMARY KEY,
    winner_serial BIGINT NOT NULL,
    loser_serial BIGINT NOT NULL,
    match_type VARCHAR(20) NOT NULL,  -- 'siege_kill', 'tournament', 'duel'
    
    -- Ratings at time of match (for historical accuracy)
    winner_rating_before DECIMAL(10,2) NOT NULL,
    winner_rating_after DECIMAL(10,2) NOT NULL,
    loser_rating_before DECIMAL(10,2) NOT NULL,
    loser_rating_after DECIMAL(10,2) NOT NULL,
    
    correlation_id VARCHAR(50),  -- Links to siege/tournament
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_glicko_matches_winner ON s51a_glicko_matches(winner_serial, created_at DESC);
CREATE INDEX idx_glicko_matches_loser ON s51a_glicko_matches(loser_serial, created_at DESC);
```

---

## 5. Town Control & Sieges

### 5.1 Town Control State
```sql
CREATE TABLE s51a_town_control (
    city_name VARCHAR(50) PRIMARY KEY,
    controlling_faction SMALLINT REFERENCES s51a_factions(faction_id),
    controlled_since TIMESTAMP WITH TIME ZONE,
    sieges_defended INT DEFAULT 0,
    total_sieges INT DEFAULT 0,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Seed siege cities
INSERT INTO s51a_town_control (city_name) VALUES
('Jhelom'), ('Skara Brae'), ('Yew'), ('Trinsic');
```

### 5.2 Siege Battle History
```sql
CREATE TABLE s51a_siege_battles (
    battle_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    city_name VARCHAR(50) NOT NULL,
    started_at TIMESTAMP WITH TIME ZONE NOT NULL,
    ended_at TIMESTAMP WITH TIME ZONE,
    
    -- Final scores
    vampire_score INT DEFAULT 0,
    daemon_score INT DEFAULT 0,
    goblin_score INT DEFAULT 0,
    
    -- Winner
    winner_faction SMALLINT REFERENCES s51a_factions(faction_id),
    victory_type VARCHAR(20),  -- 'score_limit', 'time_expiration', 'forfeit'
    
    -- Participants
    vampire_participants INT DEFAULT 0,
    daemon_participants INT DEFAULT 0,
    goblin_participants INT DEFAULT 0,
    
    -- Mechanic states at battle time
    sigil_enabled BOOLEAN DEFAULT TRUE,
    altars_enabled BOOLEAN DEFAULT TRUE,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_siege_city ON s51a_siege_battles(city_name, started_at DESC);
CREATE INDEX idx_siege_date ON s51a_siege_battles(started_at DESC);
```

### 5.3 Siege Participation
```sql
CREATE TABLE s51a_siege_participants (
    battle_id UUID NOT NULL REFERENCES s51a_siege_battles(battle_id),
    player_serial BIGINT NOT NULL,
    faction_id SMALLINT NOT NULL,
    
    -- Stats
    kills INT DEFAULT 0,
    deaths INT DEFAULT 0,
    assists INT DEFAULT 0,
    sigils_captured INT DEFAULT 0,
    sigils_stolen INT DEFAULT 0,
    altars_captured INT DEFAULT 0,
    traps_placed INT DEFAULT 0,
    trap_triggers INT DEFAULT 0,
    
    -- Rewards earned
    faction_points_earned INT DEFAULT 0,
    silver_earned INT DEFAULT 0,
    
    joined_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    PRIMARY KEY (battle_id, player_serial)
);

CREATE INDEX idx_siege_participants_player ON s51a_siege_participants(player_serial, joined_at DESC);
```

### 5.4 Siege Kill Log
```sql
CREATE TABLE s51a_siege_kills (
    kill_id BIGSERIAL PRIMARY KEY,
    battle_id UUID NOT NULL REFERENCES s51a_siege_battles(battle_id),
    killer_serial BIGINT NOT NULL,
    killer_faction SMALLINT NOT NULL,
    victim_serial BIGINT NOT NULL,
    victim_faction SMALLINT NOT NULL,
    
    -- Points awarded (after Glicko weighting)
    points_awarded INT NOT NULL,
    silver_awarded INT NOT NULL,
    glicko_multiplier DECIMAL(4,2) DEFAULT 1.00,
    
    -- Attribution
    kill_source VARCHAR(20) NOT NULL,  -- 'player', 'trap', 'turret'
    attributed_to_serial BIGINT,  -- If trap/turret, who gets credit
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_siege_kills_battle ON s51a_siege_kills(battle_id);
CREATE INDEX idx_siege_kills_killer ON s51a_siege_kills(killer_serial, created_at DESC);
```

---

## 6. Tournament System

### 6.1 Tournaments
```sql
CREATE TABLE s51a_tournaments (
    tournament_id SERIAL PRIMARY KEY,
    started_at TIMESTAMP WITH TIME ZONE NOT NULL,
    ended_at TIMESTAMP WITH TIME ZONE,
    status VARCHAR(20) DEFAULT 'registration',  -- 'registration', 'in_progress', 'complete', 'cancelled'
    
    -- Results
    winner_serial BIGINT,
    runner_up_serial BIGINT,
    participant_count INT DEFAULT 0,
    total_rounds INT DEFAULT 0,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_tournaments_status ON s51a_tournaments(status, started_at DESC);
```

### 6.2 Tournament Participants
```sql
CREATE TABLE s51a_tournament_participants (
    tournament_id INT NOT NULL REFERENCES s51a_tournaments(tournament_id),
    player_serial BIGINT NOT NULL,
    
    -- Bracket position
    seed_position INT,
    final_position INT,  -- 1 = winner, 2 = runner-up, etc.
    
    -- Stats
    wins INT DEFAULT 0,
    losses INT DEFAULT 0,
    rounds_survived INT DEFAULT 0,
    
    -- Rewards
    coins_earned INT DEFAULT 0,
    
    registered_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    eliminated_at TIMESTAMP WITH TIME ZONE,
    
    PRIMARY KEY (tournament_id, player_serial)
);

CREATE INDEX idx_tournament_participants_player ON s51a_tournament_participants(player_serial);
```

### 6.3 Tournament Matches
```sql
CREATE TABLE s51a_tournament_matches (
    match_id SERIAL PRIMARY KEY,
    tournament_id INT NOT NULL REFERENCES s51a_tournaments(tournament_id),
    round_number INT NOT NULL,
    match_number INT NOT NULL,
    arena_id INT NOT NULL,
    
    -- Combatants
    player1_serial BIGINT NOT NULL,
    player2_serial BIGINT NOT NULL,
    
    -- Result
    winner_serial BIGINT,
    loser_serial BIGINT,
    win_reason VARCHAR(20),  -- 'kill', 'timeout', 'forfeit', 'disconnect'
    
    -- Timing
    scheduled_at TIMESTAMP WITH TIME ZONE,
    started_at TIMESTAMP WITH TIME ZONE,
    ended_at TIMESTAMP WITH TIME ZONE,
    duration_seconds INT,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_tournament_matches_tournament ON s51a_tournament_matches(tournament_id, round_number);
```

---

## 7. Daily Content

### 7.1 Daily Bounty Definitions
```sql
-- Defines possible bounty tasks (admin-editable)
CREATE TABLE s51a_bounty_definitions (
    bounty_id SERIAL PRIMARY KEY,
    bounty_type VARCHAR(20) NOT NULL,  -- 'monster', 'resource', 'activity'
    target_name VARCHAR(100) NOT NULL,
    target_type VARCHAR(100),  -- Class name for code matching
    required_count INT NOT NULL,
    faction_point_reward INT NOT NULL,
    silver_reward INT NOT NULL,
    difficulty VARCHAR(10) DEFAULT 'medium',  -- 'easy', 'medium', 'hard'
    is_active BOOLEAN DEFAULT TRUE,
    
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Seed bounty definitions
INSERT INTO s51a_bounty_definitions (bounty_type, target_name, target_type, required_count, faction_point_reward, silver_reward, difficulty) VALUES
-- Monster kills
('monster', 'Skeleton', 'Skeleton', 20, 100, 25, 'easy'),
('monster', 'Zombie', 'Zombie', 20, 100, 25, 'easy'),
('monster', 'Orc', 'Orc', 15, 100, 30, 'easy'),
('monster', 'Lizardman', 'Lizardman', 15, 100, 30, 'easy'),
('monster', 'Ettin', 'Ettin', 10, 150, 50, 'medium'),
('monster', 'Ogre', 'Ogre', 10, 150, 50, 'medium'),
('monster', 'Troll', 'Troll', 10, 150, 50, 'medium'),
('monster', 'Drake', 'Drake', 5, 200, 75, 'hard'),
('monster', 'Daemon', 'Daemon', 3, 250, 100, 'hard'),
('monster', 'Lich', 'Lich', 3, 250, 100, 'hard'),
-- Resources
('resource', 'Iron Ore', 'IronOre', 100, 100, 25, 'easy'),
('resource', 'Dull Copper Ore', 'DullCopperOre', 75, 125, 35, 'medium'),
('resource', 'Shadow Ore', 'ShadowOre', 50, 150, 50, 'medium'),
('resource', 'Valorite Ore', 'ValoriteOre', 25, 200, 75, 'hard'),
('resource', 'Logs', 'Log', 100, 100, 25, 'easy'),
('resource', 'Oak Logs', 'OakLog', 75, 125, 35, 'medium'),
('resource', 'Leather', 'Leather', 100, 100, 25, 'easy'),
('resource', 'Spined Leather', 'SpinedLeather', 50, 150, 50, 'medium'),
-- Activities
('activity', 'Craft Bandages', 'Bandage', 50, 100, 25, 'easy'),
('activity', 'Craft Potions', 'BasePotion', 20, 125, 35, 'medium'),
('activity', 'Craft Arrows', 'Arrow', 100, 100, 25, 'easy');
```

### 7.2 Daily Bounty Schedule
```sql
-- Which bounties are assigned each day
CREATE TABLE s51a_daily_bounties (
    bounty_date DATE PRIMARY KEY,
    task1_bounty_id INT REFERENCES s51a_bounty_definitions(bounty_id),
    task2_bounty_id INT REFERENCES s51a_bounty_definitions(bounty_id),
    task3_bounty_id INT REFERENCES s51a_bounty_definitions(bounty_id),
    generated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_daily_bounties_date ON s51a_daily_bounties(bounty_date DESC);
```

### 7.3 Player Bounty Progress
```sql
CREATE TABLE s51a_player_bounty_progress (
    player_serial BIGINT NOT NULL,
    bounty_date DATE NOT NULL,
    
    task1_progress INT DEFAULT 0,
    task1_claimed BOOLEAN DEFAULT FALSE,
    task1_claimed_at TIMESTAMP WITH TIME ZONE,
    
    task2_progress INT DEFAULT 0,
    task2_claimed BOOLEAN DEFAULT FALSE,
    task2_claimed_at TIMESTAMP WITH TIME ZONE,
    
    task3_progress INT DEFAULT 0,
    task3_claimed BOOLEAN DEFAULT FALSE,
    task3_claimed_at TIMESTAMP WITH TIME ZONE,
    
    PRIMARY KEY (player_serial, bounty_date)
);
```

### 7.4 Daily Faction Quest
```sql
-- Quest location schedule
CREATE TABLE s51a_faction_quest_schedule (
    quest_date DATE PRIMARY KEY,
    location_id INT NOT NULL,
    location_name VARCHAR(100) NOT NULL,
    boss_type VARCHAR(50) NOT NULL,
    spawn_x INT NOT NULL,
    spawn_y INT NOT NULL,
    spawn_z INT NOT NULL,
    map_id INT DEFAULT 0,  -- 0 = Felucca
    generated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- Quest completion tracking
CREATE TABLE s51a_faction_quest_completions (
    player_serial BIGINT NOT NULL,
    quest_date DATE NOT NULL,
    faction_points_earned INT NOT NULL,
    completed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    PRIMARY KEY (player_serial, quest_date)
);

-- Quest locations definition
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

-- Seed locations
INSERT INTO s51a_faction_quest_locations (name, region_type, spawn_x, spawn_y, spawn_z) VALUES
('Fens of the Dead', 'swamp', 5765, 3190, 0),
('Bog of Desolation', 'swamp', 1475, 3680, 0),
('Scorched Sands', 'desert', 1955, 2680, 0),
('Sun Temple Ruins', 'desert', 2085, 2300, 0);
```

---

## 8. New Player Experience

### 8.1 Young Player Status
```sql
CREATE TABLE s51a_young_players (
    player_serial BIGINT PRIMARY KEY,
    account_name VARCHAR(100) NOT NULL,
    started_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    expires_at TIMESTAMP WITH TIME ZONE NOT NULL,  -- started_at + 14 days
    renounced BOOLEAN DEFAULT FALSE,
    renounced_at TIMESTAMP WITH TIME ZONE,
    
    -- Tracking
    quests_completed INT DEFAULT 0,
    total_play_time_minutes INT DEFAULT 0
);

CREATE INDEX idx_young_players_expires ON s51a_young_players(expires_at) WHERE renounced = FALSE;
```

### 8.2 Starter Quest Progress
```sql
CREATE TABLE s51a_starter_quest_progress (
    player_serial BIGINT NOT NULL,
    quest_id VARCHAR(50) NOT NULL,
    
    status VARCHAR(20) DEFAULT 'not_started',  -- 'not_started', 'in_progress', 'complete'
    progress_data JSONB,  -- Quest-specific progress
    started_at TIMESTAMP WITH TIME ZONE,
    completed_at TIMESTAMP WITH TIME ZONE,
    
    PRIMARY KEY (player_serial, quest_id)
);
```

---

## 9. PvP Statistics

### 9.1 Lifetime Stats
```sql
CREATE TABLE s51a_pvp_stats (
    player_serial BIGINT PRIMARY KEY,
    
    -- Combat
    total_kills INT DEFAULT 0,
    total_deaths INT DEFAULT 0,
    total_assists INT DEFAULT 0,
    kill_streak_best INT DEFAULT 0,
    
    -- Sieges
    sieges_participated INT DEFAULT 0,
    sieges_won INT DEFAULT 0,
    sigils_captured INT DEFAULT 0,
    altars_captured INT DEFAULT 0,
    
    -- Tournaments
    tournaments_entered INT DEFAULT 0,
    tournaments_won INT DEFAULT 0,
    tournament_matches_won INT DEFAULT 0,
    tournament_matches_lost INT DEFAULT 0,
    
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

---

## 10. Configuration Tables

### 10.1 System Configuration
```sql
-- Runtime-configurable values (no restart needed)
CREATE TABLE s51a_config (
    config_key VARCHAR(100) PRIMARY KEY,
    config_value TEXT NOT NULL,
    value_type VARCHAR(20) DEFAULT 'string',  -- 'string', 'int', 'decimal', 'bool', 'json'
    description TEXT,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_by VARCHAR(100)
);

-- Seed configuration
INSERT INTO s51a_config (config_key, config_value, value_type, description) VALUES
-- Siege settings
('siege.duration_minutes', '30', 'int', 'Siege battle duration'),
('siege.global_cooldown_minutes', '30', 'int', 'Cooldown between any sieges'),
('siege.city_cooldown_minutes', '120', 'int', 'Cooldown per city'),
('siege.min_players_per_faction', '3', 'int', 'Minimum players per faction to trigger'),
('siege.min_total_players', '10', 'int', 'Minimum total players to trigger'),
('siege.victory_score', '10000', 'int', 'Score needed to win'),
('siege.sigil_enabled', 'true', 'bool', 'Is sigil mechanic active'),
('siege.altars_enabled', 'true', 'bool', 'Is altar mechanic active'),
('siege.defense_budget_max', '5000', 'int', 'Max silver per faction for defenses'),

-- Town control
('town.single_city_discount', '0.10', 'decimal', 'NPC discount for 1-3 cities'),
('town.all_cities_discount', '0.15', 'decimal', 'NPC discount for all 4 cities'),

-- Tournament settings
('tournament.min_players', '6', 'int', 'Minimum players to start'),
('tournament.match_duration_seconds', '300', 'int', 'Match time limit'),
('tournament.prep_time_seconds', '30', 'int', 'Prep time before match'),
('tournament.coin_reward_participation', '50', 'int', 'Coins for participating'),
('tournament.coin_reward_win', '100', 'int', 'Coins per match win'),
('tournament.coin_reward_champion', '500', 'int', 'Bonus coins for winning'),

-- Bounty settings
('bounty.reset_hour_utc', '5', 'int', 'Hour in UTC for daily reset (5 = midnight EST)'),

-- Faction quest settings
('factionquest.boss_base_hp', '5000', 'int', 'Boss base hit points'),
('factionquest.boss_hp_per_player', '500', 'int', 'Additional HP per nearby player'),
('factionquest.boss_max_hp', '25000', 'int', 'Boss maximum hit points'),
('factionquest.respawn_hours', '2', 'int', 'Hours until boss respawns'),
('factionquest.reward_faction_points', '500', 'int', 'FP reward for quest'),
('factionquest.reward_bonus_top_damage', '250', 'int', 'Bonus FP for top damage'),

-- Relic drop chances
('factionquest.relic_common_chance', '0.15', 'decimal', 'Common relic drop chance'),
('factionquest.relic_uncommon_chance', '0.05', 'decimal', 'Uncommon relic drop chance'),
('factionquest.relic_rare_chance', '0.01', 'decimal', 'Rare relic drop chance'),

-- Glicko settings
('glicko.default_rating', '1500', 'decimal', 'Starting rating'),
('glicko.default_rd', '350', 'decimal', 'Starting rating deviation'),
('glicko.default_volatility', '0.06', 'decimal', 'Starting volatility'),
('glicko.tau', '0.5', 'decimal', 'System constant (tau)'),

-- Young player settings
('young.duration_days', '14', 'int', 'Duration of young player protection'),

-- Combat status
('combat_status.cooldown_hours', '24', 'int', 'Cooldown for combat status change'),

-- Faction change
('faction.change_cooldown_days', '7', 'int', 'Days between faction changes');
```

### 10.2 Siege City Configuration
```sql
CREATE TABLE s51a_siege_cities (
    city_name VARCHAR(50) PRIMARY KEY,
    is_active BOOLEAN DEFAULT TRUE,
    
    -- Region bounds
    region_x1 INT NOT NULL,
    region_y1 INT NOT NULL,
    region_x2 INT NOT NULL,
    region_y2 INT NOT NULL,
    map_id INT DEFAULT 0,
    
    -- Spawn points (stored as JSON arrays)
    sigil_spawn JSONB NOT NULL,  -- {x, y, z}
    altar_spawns JSONB NOT NULL,  -- [{x, y, z}, ...]
    priest_spawns JSONB NOT NULL,  -- [{faction_id, x, y, z}, ...]
    banner_locations JSONB NOT NULL  -- [{x, y, z}, ...]
);
```

---

## 11. Telemetry & Analytics

### 11.1 Event Log
```sql
CREATE TABLE s51a_events (
    event_id BIGSERIAL PRIMARY KEY,
    event_type VARCHAR(50) NOT NULL,
    event_data JSONB NOT NULL,
    correlation_id VARCHAR(50),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_events_type ON s51a_events(event_type, created_at DESC);
CREATE INDEX idx_events_correlation ON s51a_events(correlation_id) WHERE correlation_id IS NOT NULL;
CREATE INDEX idx_events_date ON s51a_events(created_at DESC);

-- Partition by month for performance
-- (implement partitioning for production)
```

### 11.2 Daily Aggregates
```sql
CREATE TABLE s51a_daily_stats (
    stat_date DATE NOT NULL,
    stat_name VARCHAR(100) NOT NULL,
    stat_value BIGINT NOT NULL,
    
    PRIMARY KEY (stat_date, stat_name)
);

-- Example stats tracked:
-- 'active_players', 'sieges_started', 'sieges_completed', 'tournament_participants',
-- 'faction_points_awarded', 'silver_awarded', 'bounties_completed', etc.
```

---

## 12. Database Functions

### 12.1 Award Faction Points
```sql
CREATE OR REPLACE FUNCTION s51a_award_faction_points(
    p_account_name VARCHAR(100),
    p_player_serial BIGINT,
    p_season_id INT,
    p_faction_id SMALLINT,
    p_points INT,
    p_reason VARCHAR(200),
    p_correlation_id VARCHAR(50) DEFAULT NULL
) RETURNS VOID AS $$
BEGIN
    -- Upsert points
    INSERT INTO s51a_faction_points (account_name, season_id, faction_id, points, updated_at)
    VALUES (p_account_name, p_season_id, p_faction_id, p_points, NOW())
    ON CONFLICT (account_name, season_id) 
    DO UPDATE SET 
        points = s51a_faction_points.points + p_points,
        updated_at = NOW();
    
    -- Log transaction
    INSERT INTO s51a_faction_point_log 
        (account_name, player_serial, season_id, faction_id, points_delta, reason, correlation_id)
    VALUES 
        (p_account_name, p_player_serial, p_season_id, p_faction_id, p_points, p_reason, p_correlation_id);
END;
$$ LANGUAGE plpgsql;
```

### 12.2 Award Silver
```sql
CREATE OR REPLACE FUNCTION s51a_award_silver(
    p_player_serial BIGINT,
    p_amount INT,
    p_reason VARCHAR(200),
    p_correlation_id VARCHAR(50) DEFAULT NULL
) RETURNS VOID AS $$
BEGIN
    -- Upsert balance
    INSERT INTO s51a_silver_balances (player_serial, balance, lifetime_earned, updated_at)
    VALUES (p_player_serial, p_amount, p_amount, NOW())
    ON CONFLICT (player_serial) 
    DO UPDATE SET 
        balance = s51a_silver_balances.balance + p_amount,
        lifetime_earned = s51a_silver_balances.lifetime_earned + p_amount,
        updated_at = NOW();
    
    -- Log transaction
    INSERT INTO s51a_silver_log (player_serial, silver_delta, reason, correlation_id)
    VALUES (p_player_serial, p_amount, p_reason, p_correlation_id);
END;
$$ LANGUAGE plpgsql;
```

### 12.3 Spend Silver
```sql
CREATE OR REPLACE FUNCTION s51a_spend_silver(
    p_player_serial BIGINT,
    p_amount INT,
    p_reason VARCHAR(200)
) RETURNS BOOLEAN AS $$
DECLARE
    v_current_balance INT;
BEGIN
    -- Get current balance with lock
    SELECT balance INTO v_current_balance
    FROM s51a_silver_balances
    WHERE player_serial = p_player_serial
    FOR UPDATE;
    
    IF v_current_balance IS NULL OR v_current_balance < p_amount THEN
        RETURN FALSE;
    END IF;
    
    -- Deduct
    UPDATE s51a_silver_balances
    SET balance = balance - p_amount,
        lifetime_spent = lifetime_spent + p_amount,
        updated_at = NOW()
    WHERE player_serial = p_player_serial;
    
    -- Log
    INSERT INTO s51a_silver_log (player_serial, silver_delta, reason)
    VALUES (p_player_serial, -p_amount, p_reason);
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;
```

### 12.4 Get Config Value
```sql
CREATE OR REPLACE FUNCTION s51a_get_config_int(p_key VARCHAR(100), p_default INT DEFAULT 0)
RETURNS INT AS $$
DECLARE
    v_value TEXT;
BEGIN
    SELECT config_value INTO v_value FROM s51a_config WHERE config_key = p_key;
    IF v_value IS NULL THEN
        RETURN p_default;
    END IF;
    RETURN v_value::INT;
EXCEPTION WHEN OTHERS THEN
    RETURN p_default;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION s51a_get_config_bool(p_key VARCHAR(100), p_default BOOLEAN DEFAULT FALSE)
RETURNS BOOLEAN AS $$
DECLARE
    v_value TEXT;
BEGIN
    SELECT config_value INTO v_value FROM s51a_config WHERE config_key = p_key;
    IF v_value IS NULL THEN
        RETURN p_default;
    END IF;
    RETURN v_value::BOOLEAN;
EXCEPTION WHEN OTHERS THEN
    RETURN p_default;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION s51a_get_config_decimal(p_key VARCHAR(100), p_default DECIMAL DEFAULT 0)
RETURNS DECIMAL AS $$
DECLARE
    v_value TEXT;
BEGIN
    SELECT config_value INTO v_value FROM s51a_config WHERE config_key = p_key;
    IF v_value IS NULL THEN
        RETURN p_default;
    END IF;
    RETURN v_value::DECIMAL;
EXCEPTION WHEN OTHERS THEN
    RETURN p_default;
END;
$$ LANGUAGE plpgsql;
```

---

## 13. Indexes Summary

All indexes are defined inline with table creation. Key indexes for performance:

| Table | Index | Purpose |
|-------|-------|---------|
| s51a_faction_points | season_id, points DESC | Leaderboard queries |
| s51a_glicko_ratings | rating DESC WHERE games >= 10 | Ranked leaderboard |
| s51a_siege_battles | city, started_at DESC | Recent sieges by city |
| s51a_events | type, created_at DESC | Event queries |
| s51a_events | correlation_id | Trace related events |

---

## 14. Maintenance Procedures

### 14.1 Archive Old Data
```sql
-- Archive events older than 90 days
CREATE OR REPLACE PROCEDURE s51a_archive_old_events()
AS $$
BEGIN
    -- Move to archive table (create if needed)
    INSERT INTO s51a_events_archive
    SELECT * FROM s51a_events
    WHERE created_at < NOW() - INTERVAL '90 days';
    
    -- Delete archived
    DELETE FROM s51a_events
    WHERE created_at < NOW() - INTERVAL '90 days';
END;
$$ LANGUAGE plpgsql;
```

### 14.2 Generate Daily Bounties
```sql
CREATE OR REPLACE PROCEDURE s51a_generate_daily_bounties(p_date DATE)
AS $$
DECLARE
    v_seed INT;
    v_monster_id INT;
    v_resource_id INT;
    v_activity_id INT;
BEGIN
    -- Check if already generated
    IF EXISTS (SELECT 1 FROM s51a_daily_bounties WHERE bounty_date = p_date) THEN
        RETURN;
    END IF;
    
    -- Use date as seed for deterministic random
    v_seed := EXTRACT(DOY FROM p_date)::INT + EXTRACT(YEAR FROM p_date)::INT * 1000;
    
    -- Select one of each type
    SELECT bounty_id INTO v_monster_id FROM s51a_bounty_definitions 
    WHERE bounty_type = 'monster' AND is_active = TRUE
    ORDER BY (bounty_id * v_seed) % 1000 LIMIT 1;
    
    SELECT bounty_id INTO v_resource_id FROM s51a_bounty_definitions 
    WHERE bounty_type = 'resource' AND is_active = TRUE
    ORDER BY (bounty_id * v_seed) % 1000 LIMIT 1;
    
    SELECT bounty_id INTO v_activity_id FROM s51a_bounty_definitions 
    WHERE bounty_type = 'activity' AND is_active = TRUE
    ORDER BY (bounty_id * v_seed) % 1000 LIMIT 1;
    
    INSERT INTO s51a_daily_bounties (bounty_date, task1_bounty_id, task2_bounty_id, task3_bounty_id)
    VALUES (p_date, v_monster_id, v_resource_id, v_activity_id);
END;
$$ LANGUAGE plpgsql;
```

---

## 15. Initial Setup Script

```sql
-- Run this script to initialize the database

-- Create database (run as superuser)
-- CREATE DATABASE "51alpha" OWNER "51alpha_app";

-- Connect to 51alpha database, then run:

-- Execute all CREATE TABLE statements above
-- Execute all CREATE INDEX statements above
-- Execute all CREATE FUNCTION statements above
-- Execute all INSERT seed data statements above

-- Verify setup
SELECT 'Tables' as type, COUNT(*) as count FROM information_schema.tables WHERE table_name LIKE 's51a_%'
UNION ALL
SELECT 'Functions', COUNT(*) FROM information_schema.routines WHERE routine_name LIKE 's51a_%'
UNION ALL
SELECT 'Indexes', COUNT(*) FROM pg_indexes WHERE indexname LIKE 'idx_%';
```

---

## Change Log

### v1.0.0 - 2025-01-02 (Initial Schema)
- Complete schema for all 51alpha systems
- Faction, currency, rating tables
- Siege and tournament history
- Daily content tracking
- Configuration tables with seed data
- Helper functions for common operations