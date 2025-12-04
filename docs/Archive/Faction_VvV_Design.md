# Faction/VvV System Deep Dive

## Document Metadata
- **Version**: v1.0.0
- **Last Updated**: 2025-01-01
- **Authors**: Sphere51a Development Team
- **Applicable ModernUO Version**: v24.0.0+
- **PR Ready**: No (Conceptual Design)

## Overview
Comprehensive faction warfare system integrating guild allegiances, town control mechanics, PvP token rewards, tournament rankings with Glicko-2 ratings, PvM vanquish points, and crafting Bulk Order Deed contributions into a competitive ecosystem with seasonal rewards.

## Algorithms and Logic
Glicko-2 rating system for PvP matchmaking and leaderboards, stance-based peacefulness mechanics, siege win determination algorithms with tie-breaker rules, town control state management, and faction point aggregation across PvP/PvM/crafting activities.

## Edge Cases
Handles guild leadership transitions during faction assignments, mid-siege server crashes with state recovery, tie-breaking in equal-scoring factions, player stance conflicts, and simultaneous territory control attempts.

## Implementation Details
Single-writer FactionEngine per town region, EventHub with ring buffer event streaming, atomic TownControl state updates, CorrelationId tracking for all scoring events, and batched asynchronous database persistence.

## Testing Plan
Comprehensive siege cycle simulations, tournament bracket testing, multi-faction conflict scenarios, state recovery validation, tie-breaker algorithm verification, and scalability testing with concurrent faction activities.

## Development Prerequisites

### ModernUO Base Requirements
- **Minimum Version**: ModernUO v24.0.0 or later (Assumes clean master branch)
- **Branch**: Use `feature/faction_system` for implementation
- **Dependencies**: Requires PvP tournament system, Glicko-2 ratings, BOD crafting integration

### Required Framework Knowledge
- ModernUO guild and faction systems
- PvP rating and matchmaking algorithms
- Multi-region territory control mechanics
- Asynchronous event processing and state synchronization
- Large-scale player aggregation and scoring systems

### Pre-Implementation Checklist
- [ ] Guild system integration and faction assignment mechanics verified
- [ ] Tournament system with Glicko-2 ratings implemented
- [ ] Town region boundaries and control mechanisms defined
- [ ] PvP token and vanquish point systems operational
- [ ] BOD crafting integration with faction point allocation ready

## Code Integration Guide

### Step 1: Core Faction Infrastructure (High Risk)
1. Implement FactionEngine with single-writer pattern per town
2. Create TownControl state management system
3. Set up EventHub with ring buffer event streaming

### Step 2: Guild and Player Integration (Medium Risk)
1. Develop guild faction assignment with 7-day cooldown
2. Implement player stance selection (PvP/PvM/Crafter)
3. Add faction loyalty tracking and audit system

### Step 3: Scoring and Rewards System (High Risk)
1. Integrate Glicko-2 for PvP ratings and tournament brackets
2. Implement vanquish point tracking for PvM activities
3. Add BOD completion faction point allocation

### Step 4: Siege and Town Control (High Risk)
1. Create siege scheduling and lifecycle management
2. Implement territory control mechanics and state changes
3. Add faction perk assignment for controlling factions

### Step 5: UI and Player Experience (Medium Risk)
1. Develop faction status and registration Gumps
2. Create leaderboard interfaces and tournament tables
3. Implement town control visualization and announcements

## Performance Benchmarks

### Expected Performance Impact
- **Siege Processing**: <1 minute for complete siege cycle with 1000+ active participants
- **Point Scoring**: <10ms per PvP token/PvM vanquish/BOD completion
- **Glicko Updates**: <5 seconds for tournament result processing
- **Faction Queries**: <50ms for leaderboard and town status requests
- **Concurrent Sieges**: Support 4+ simultaneous town sieges with <15% performance impact

### Monitoring Recommendations
- Track siege participation rates and faction engagement metrics
- Monitor Glicko rating calculation performance
- Alert on scoring discrepancies (>0.1% error rate)
- Track player stance distribution and conflict rates
- Monitor database performance for point aggregation queries

## Maintenance Notes

### Future Enhancements
- Add cross-server faction alliances for larger conflicts
- Implement seasonal faction war themes and objectives
- Create guild-specific faction bonuses and abilities
- Add real-time faction war visualization on website

### Operational Considerations
- Regular Glicko rating system calibration and drift detection
- Siege scheduling optimization to maximize participation
- Point weighting adjustments based on faction activity metrics
- Regular faction turnover analysis to maintain engagement

### Rollback Procedures
1. Disable all faction-related scoring and siege mechanics
2. Create complete faction state snapshot for restoration
3. Revert town control states to previous known good state
4. Validate point totals and faction standings integrity
5. Restore faction system with corrected logic and balancer

## Testing Procedures

### Unit Testing (Automated)
```csharp
[TestMethod]
public void FactionSystem_SiegeScoring_CalculatesWinnersCorrectly()
{
    // Arrange
    var siege = CreateSiegeWithMultipleFactionScore();
    var expectedWinnerFaction = DetermineExpectedTieBreakerWinner();

    // Act
    var result = siege.ComputeWinner();

    // Assert
    Assert.AreEqual(expectedWinnerFaction, result.WinningFaction);
    Assert.IsTrue(result.PointsValidated);
}

[TestMethod]
public void FactionSystem_GuildFactionChange_EnforcesCooldownPeriod()
{
    // Arrange
    var guild = CreateTestGuild();
    var originalChangeTime = DateTime.UtcNow.AddDays(-6);
    guild.LastFactionChangeTime = originalChangeTime;

    // Act - attempt change within 7 days
    var canChange = guild.CanChangeFaction(DateTime.UtcNow);

    // Assert
    Assert.IsFalse(canChange, "Should enforce 7-day cooldown");
}
```

### Integration Testing (Live Server)
1. **Siege Cycles**: Test complete siege from scheduling to resolution
2. **Tournament Scoring**: Verify Glicko rating updates and leaderboard accuracy
3. **Cross-Game Interactions**: Test PvP, PvM, and crafting point integration
4. **Guild Management**: Validate faction assignment and stance inheritance

### Load Testing
- Test faction system with maximum concurrent active players
- Validate siege processing under heavy combat activity load
- Test tournament ranking calculations with high match volume
- Verify database performance with intensive point aggregation

## Change Log

### v1.0.0 - 2025-01-01 (Initial Professional Standard Update)
- Added **Document Metadata** section with versioning and compatibility info
- Added **Development Prerequisites** with framework knowledge requirements and pre-implementation checklist
- Added **Code Integration Guide** with step-by-step implementation instructions and risk levels
- Added **Performance Benchmarks** with expected impact and monitoring recommendations
- Added **Maintenance Notes** with enhancement plans and rollback procedures
- Added **Testing Procedures** with unit and integration testing guidelines
- Standardized structure to meet modern developer documentation standards
- Integrated comprehensive faction warfare ecosystem design
- Added Glicko-2 tournament rating system and siege mechanics specifications
</environment_details>


Large-Scale Faction System — Senior-Team Design (Sphere51a / ModernUO)

This is a complete, production-grade design for a Faction system that links Guilds → Factions → Towns (control) and aggregates PvP, PvM, Crafting into a single competitive ecosystem. It’s written to your Sphere51a constraints: struct-first models, pooling & zero-allocation in hot paths, async I/O via ValueTask, and strict governance gates.

At the top: recommendation about number of towns: start with 4. Rationale in the Decision section below. You can expand to 5 (or more) later; the system is designed to be configurable.

Executive summary (what this delivers)

Guilds opt one-time (changeable only every 7 days) to align to a Faction (e.g., Virtue/Upright / Vice/Dark / Neutral / Custom). Guild members inherit faction loyalty.

Faction influence is applied to Town Control (town hill / king-of-the-hill sieges) and to persistent city control states.

PvP scores use a Glicko-2 rating system for matchmaking & leaderboards; tournament/weekly events are supported.

PvM scores award points for vanquishes (configurable weight by mob/tag); tracked to faction leaderboards.

Crafters receive weekly BOD tasks with faction point values for completing Bulk Order Deeds; linked to faction totals.

Players choose a single role stance (PvP / PvM / Crafter) while allied to a faction; stances control PvP flagging & peacefulness.

Rewards: season prizes for top faction, top 3 players, and guild bonuses; economic sinks and non-tradeable rewards to limit inflation.

Governance: audits, anti-exploit, idempotency, zero LINQ in hot paths, allocation gates, and telemetry built in.

Decision: 4 towns vs 5 towns

Recommendation: START with 4 towns (expandable to 5).

Why:

4 = simpler balance (two-by-two skews, easier siege scheduling and rotation).

Easier to maintain symmetric mechanics (pairs of towns can rotate schedules).

Faster convergence for control (fewer regions to fight over), higher conflict density → better player engagement in early launches.

Odd number (5) prevents ties in simple majority models but creates more scheduling, test complexity, and tie edge cases for resource distribution.

Architecturally, the system is fully configurable — add 5th town after the first season when you have tuning telemetry.

If you must ship 5 immediately, design decisions below include tie-breaker rules and balancing notes.

Core concepts & terms

Faction — a persistent team (e.g., Azure League). Guilds opt-in to one faction. Players not in a faction or guild remain neutral.

Town — region with a control state (which faction controls). Each Town runs periodic Sieges (king-of-the-hill / objective).

Loyalty Points — the unit added to a Faction’s TownScore. Earned via PvP tokens, PvM vanquishes, and crafting BOD completions.

Stance — player selectable: PvP, PvM, Crafter. A stance confers peacefulness and flagging behavior while still contributing to faction.

Season — a time period (e.g., 6 weeks) culminating in rewards for top factions/players.

Tournament — scheduled competitive event (supports Glicko-2 rating updates).

Custodian — NPC or system that collects tokens, verifies, and credits faction/team.

High-level architecture

Sphere51a.Engines.FactionEngine : BaseEngine — single authoritative engine, single writer per town to avoid contention.

FactionEventHub — small struct events (no allocations), emits FactionEvent for telemetry & auditing.

Database tables in sphere51a schema: faction_guilds, faction_players, town_control, faction_leaderboard, tournaments, bods, faction_audit.

Hotpath design rules:

Hot paths (kill, vanquish, BOD claim) are allocation-free: small structs, ring buffers, pooled arrays.

Use Interlocked / single-writer patterns for local state; background tasks handle DB writes in batched async ValueTasks.

Data models (struct-first, simplified C# pseudotypes)
// small, immutable where possible
public readonly struct FactionId { public readonly ushort Value; }
public readonly struct TownId   { public readonly ushort Value; }

// player loyalty record - lightweight
public readonly struct FactionPlayer
{
    public readonly int PlayerSerial;     // Mobile.Serial
    public readonly FactionId Faction;
    public readonly ushort GuildSerial;   // 0 = none
    public readonly sbyte Stance;         // enum: 0=Neutral,1=PvP,2=PvM,3=Crafter
    public readonly DateTime LoyaltySinceUtc;
    public readonly DateTime LastChangeAllowedUtc; // enforce 7 day cooldown
}

// live town control snapshot
public readonly struct TownControl
{
    public readonly TownId Town;
    public readonly FactionId ControllingFaction;
    public readonly int ControlStrength;  // 0..100000, configurable scale
    public readonly DateTime NextSiegeUtc;
    public readonly byte Version;
}

// event: small & pooled in engine streams
public readonly struct FactionEvent
{
    public readonly long CorrelationId;
    public readonly int ActorSerial;
    public readonly TownId Town;
    public readonly FactionEventType Type;
    public readonly int DeltaPoints;
    public readonly DateTime TimestampUtc;
}


VvvEngine pattern applies — FactionEngine owns TownControl for single-writer updates.

Database schema (notes + sample fields)

faction_guilds:

guild_serial PK

faction_id

last_changed_utc

faction_players:

player_serial PK

faction_id

stance (tinyint)

joined_utc

last_change_allowed_utc

indexes: faction_id, guild_serial

town_control:

town_id PK

controlling_faction

control_strength

next_siege_utc

last_change_utc

version int

faction_points_log (append-only audit):

entry_id PK

correlation_id

actor_serial

town_id

points

source_type (pvp/pvm/bod/tournament)

details_json

created_utc

glicko_player:

player_serial

rating_mu (double)

rating_phi (double volatility)

last_update_utc

bods_master & bods_claims for craft weekly tasks.

Partitioning: faction_points_log partitioned by season_id and date for scale.

Scoring & how points flow (sources & weights)

PvP

Honorable kills spawn Tokens. TokenClaim → credits FactionPoints to killer’s faction (or guild’s faction).

PvP token weight configurable (e.g., 10 points).

Tournament wins and match placements add bonus points (Glicko integration for rating; finish placement -> points).

PvM

Vanquish Points: primary tracker is MonsterKill events where last-hit mechanics or damage contribution ring buffer determines attribution.

Each NPC type has BasePvMPoints, multiplied by configurable TownMultiplier (if the kill is within a contested town region).

Boss / event kills have higher weights and audit.

Crafting (BOD)

Weekly BODs assigned per crafter (or guild) with point values. Completion triggers BodClaim, awarded once validated (check items, timestamps).

BOD completions are batched weekly for ledgering and rank reward assignment.

Scoring formula example:

TownContribution = PvP_Tokens * PvP_Weight
                 + PvM_Vanquishes * PvM_Weight
                 + BOD_Completions * BOD_Weight
FactionTownScore = sum(TownContribution) over last SiegeWindow


Points decay/aging:

Points count towards the current siege/season only. A longer-term loyalty value for ranks persists but has decay to avoid stale leaders.

Player & Guild actions / rules

Guild → Faction binding

A guild leader may choose a faction for the guild. This decision applies to existing guild members and restricts change frequency to once every 7 days.

Members can individually leave a guild and choose faction freely (but guild membership will override in faction tally if guild is aligned).

Individual Stance

When a player is aligned to a faction (via guild or individually), they select one stance:

PvP — flagged for PvP in faction zones and eligible for PvP rewards.

PvM — peaceful in PvP clashes; safe for crafting/harvesting; still contributes PvM points.

Crafter — peaceful and allowed to do BOD activities without flagging; contributes crafting points.

Stance can be changed but has cooldown (configurable e.g., 1 hour) to avoid abuse mid-siege.

Player switching constraints

To prevent flip-flopping: faction change (guild or player) has a 7-day cooldown. Stance changes have short cooldowns.

PvP: Glicko-2 rating + tournaments
Why Glicko-2

Glicko-2 provides a rating (mu), rating deviation (phi), and volatility. Gives faster adaptation and confidence estimates. Good for matchmaking and leaderboards.

Integration plan

Each rated PvP match (tournament or ranked match) logs results to glicko_player entries.

Glicko updates are expensive but not hot: compute in a background task after match completion and persist.

API (concept)
ValueTask RecordRankedMatchAsync(MatchResult result, CancellationToken ct);
// MatchResult includes players, outcomes (win/loss/draw), timestamps, CorrelationId.

ValueTask<GlickoSnapshot> ComputeGlickoUpdateAsync(IEnumerable<MatchResult> matches);

Offline computation model

Match ends → emit FactionEvent with CorrelationId → enqueue to GlickoProcessor (background, batched).

GlickoProcessor processes updates, writes DB, publishes new rating events.

Tournament scheduling

Two weekend tournaments per week: configurable schedule. Each tournament has:

bracket type (single elimination, swiss)

seed (Glicko mu or open)

entry cap

Integration with tournament UI & sign-up gumps.

PvM: Vanquish leaderboard mechanics

Vanquish attribution

Use a compact DamageRingBuffer on each NPC (fixed length) to record recent attackers and damage.

The last damage rule or highest contribution in window determines vanquisher for point awarding.

Event submissions

Boss encounters add heavy points and broadcast to faction gumps.

Scaling & anti-farm

Implement soft diminishing returns for repeated farming of the same NPC by same player within a short window.

Auto-detect smurfing / multi-account farming with IP / device checks (soft alerting first).

Crafting: Weekly BOD + point linkage

Weekly BOD generation

Each crafter (or guild) receives a BOD from bods_master. BOD has pointsValue, deadlineUtc, requiredItems.

BODs vary difficulty; guild crafters may receive collaborative BODs that require multiple submissions (guild BODs).

Claiming & validation

When player submits BOD items to NPC, FactionEngine validates items, removes items, records bods_claim.

Weighting

BOD points are weighted per the crafting value and are aggregated weekly. Top crafters contribute to their faction’s weekly crafting score.

Town control & Siege mechanics (KOTH)

Siege scheduling

Towns run sieges at configurable cadence (hourly/daily/weekly). Admin UI to set schedule and exceptions.

Allow tournament-linked sieges (special times) for high rewards.

Score aggregation

Siege window collects points from PvP, PvM, Crafting for that town.

At siege end, compare faction totals, compute winner, and change TownControl.

Tie-breakers

If tie: use (1) number of unique active participants, (2) Glicko team aggregate, or (3) last-token spawns as deterministic tie break.

Region Flags

Winning faction receives region perks: resurrection stone, minor buffs, special vendor, resource nodes.

Perks are designed to be non-gamebreaking, mostly cosmetic or small economic advantages.

Rewards & economy design

Faction rewards

Season rewards for top faction: monument in town, faction-wide visual buff (cosmetic), exclusive vendor, a shared faction vault (limited withdraw).

Individual rewards

Top 3 players: unique title, cosmetic mount or vanity item, a season chest with bounded rewards (not fully tradable).

Guild rewards

Bonus craft tables, shared guild store inventory slots, tax discount for guild members in town vendor.

Economic safety

Most valuable rewards are non-transferable or vanity to avoid pay-to-win economies.

Resource nodes have daily caps and sinks (vanity purchases) to remove currency.

UI / Commands / Gumps (player & admin)
Player commands & gumps

[faction join <faction>] (guild leader use to set guild faction)

[faction status] — shows player + guild allegiance, stance, top players, upcoming sieges.

Faction Gump:

Tabs: Overview, Towns (control map), Leaderboards (PvP/PvM/Crafter), BODs, Tournaments, My Guild

Real-time status tickers, CorrelationId displayed for major events (e.g., "Town captured — CorrelationId: 0x1234").

Admin commands & tools

[faction settown <town> <faction>]

[faction snapshot save|load <name>]

[faction debug replay <correlationId>] — replay event for triage

Admin UI: dashboards for current scores, dispute resolution tools (revert last N events), audit viewer.

Anti-exploit & governance controls

Idempotency & tokens

All point award actions attach a CorrelationId and are idempotent. DB unique key on (source_type, source_id, correlation_id).

Detection & soft enforcement

Soft detection for macros/multi-account farming; gather metrics (claims_per_minute, rapid_stance_switches) and build automated score flags.

Validation server-side

All client actions for scoring must be validated server-side against world state (position, items, timestamps).

Audit trail

faction_points_log append-only for rollback & audit; admin rollback command reverts via inverse ledger entry.

CI & code governance

Roslyn analyzer to prevent LINQ in hot namespaces.

Allocation p95 thresholds in CI for kill handling and BOD claim paths.

Telemetry & metrics (essential)

faction_points_total{town,faction,source}

faction_active_players{town,faction}

siege_duration_seconds

glicko_update_latency_seconds

bods_assigned_weekly, bods_completed_weekly

claims_conflict_total, duplication_attempts_total

player_stance_changes_total (for abuse detection)

Design dashboards: Town control heatmap, top leaderboards, sieges per week chart, BOD completion rates, anti-exploit flags.

Testing matrix (high level)

Unit: stance cooldown, guild→faction change enforcement, BOD validation rules, Glicko update math (deterministic).

Integration: full siege cycle with multiple simulated players, BOD batch processing, tournament brackets.

Load: simulate X concurrent players (configurable). Hotpath metrics: kills/sec, claims/sec, allocations.

Chaos: restart mid-siege; verify idempotency & no double credit.

Anti-exploit: simulate scripted clients to ensure detection thresholds trigger.

Rollout plan (concrete phases)

Phase 0 — Foundations (2 weeks)

Implement FactionEngine skeleton, DB schema, config JSON for towns (start with 4).

Implement player & guild faction join rules (7 day cooldown enforcement).

UI: minimal faction status gump and town map.

Phase 1 — PvP token & siege core (3 weeks)

Implement honest token spawn & custodian claim system with idempotency.

Basic siege scheduler & town control changes (safe swap window).

Telemetry for tokens & sieges.

Phase 2 — PvM & BOD linking (3 weeks)

Implement vanquish scoreboard and damage attribution.

Weekly BOD generation & claim flow; link BOD completions to faction points.

Phase 3 — Tournament & Glicko (2–3 weeks)

Create ranked match queue, tournament brackets, and Glicko processor.

Add rating UI and leaderboards.

Phase 4 — Rewards & balancing (2 weeks)

Implement seasonal reward distribution, top players/guilds.

Add cosmetic vendors & small economic perks.

Phase 5 — QA & Hardening (2–3 weeks)

Load & chaos tests, anti-exploit tuning, CI gates, monitor metrics, community opt-in beta.

Phase 6 — Expand towns (optional)

Add 5th town/configurable expansion after observing balance & population.

Example config (JSON)
{
  "Towns": [
    { "Id": 1, "Name": "Trinsic", "Region": "TrinsicRegion", "SiegeIntervalMinutes": 60 },
    { "Id": 2, "Name": "Jhelom", "Region": "JhelomRegion", "SiegeIntervalMinutes": 60 },
    { "Id": 3, "Name": "Moonglow", "Region": "MoonglowRegion", "SiegeIntervalMinutes": 60 },
    { "Id": 4, "Name": "Skara Brae", "Region": "SkaraRegion", "SiegeIntervalMinutes": 60 }
  ],
  "PointsWeights": { "PvP": 10, "PvM": 1, "BOD": 5 },
  "FactionChangeCooldownDays": 7,
  "StanceChangeCooldownMinutes": 60
}

Anti-abuse quick wins (low effort, high ROI)

CorrelationId on every point award — immediate traceability.

TokenClaimState atomic flag via Interlocked.CompareExchange to prevent duplicate claims.

Publish opt-in normalized PvP arena to show fairness commitment.

Add glicko_processor as background batched task — avoid heavy sync computations in hot paths.

Governance & acceptance criteria

All hotpath code (kill handling, claim processing) must show zero LINQ, allocations per kill < X bytes (CI gate).

kill_processing_p95 < 10ms (target; tune per shard).

no duplication incidents in chaos tests (restart/replay).

Audit logs for all town control changes with CorrelationId and rollback capability.

Player experience KPIs: new player conversion to faction > 5%, weekly active participants in sieges > N (tunable).

Edge cases & open questions (for product)

Should guild leadership changes be allowed to reassign guild faction within 7 days? (we enforce guild leader action; log & require confirmation)

How aggressive should anti-macro enforcement be? (start soft: log + manual flagging)

Are season lengths configurable? (recommend 4–8 weeks)

Is PvP normalized mode always on, or opt-in? (recommend opt-in; default is linked to faction PvP)

Final notes (why this will scale)

Single-writer engine per town reduces concurrency complexity and race conditions.

Configurable weights let you tune PvP/PvM/Craft balance without code changes.

Struct-first design, pooling, and async background tasks protect performance at scale.

Built-in telemetry + correlation ids let ops react quickly and tune the economy & balance.
