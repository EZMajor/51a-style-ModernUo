51alpha Master Architecture Document
Comprehensive Game Server Specification
Document Metadata

Version: v2.0.0
Last Updated: 2025-01-02
Authors: 51alpha Development Team
Platform: ModernUO v24.0.0+
Combat Style: Sphere51a (Sphere-style PvP)


Table of Contents

Executive Summary
Core Design Principles
Combat System
Spell System
Talisman System
Faction System (VvV)
Dungeon System
Economy Model
Housing & Relic System
New Player Experience
Crafting & BOD System
Glicko-2 Rating System
Social Systems
Tournament System
Infrastructure & Performance
Security
NPC Systems
Configuration Reference
Implementation Phases


1. Executive Summary
1.1 Project Vision
51alpha is a guild-based faction PvP Ultima Online server built on ModernUO with Sphere-style combat mechanics. The server emphasizes:

Skill-based PvP with free movement during casting
Guild-centric faction warfare with sigil capture events
Clear PvP/PvM separation - talisman bonuses disabled in PvP
Accessible progression - combat skills via quick quests, not grinding
Quarterly seasons with meaningful rewards

1.2 Key Design Decisions Summary
AreaDecisionRationaleCombatSphere-style (free movement)Skill-based, mobile gameplayFizzleResources consumedPunishes mistakes, rewards skillFactions3 factions, guild-basedNatural 2v1 dynamics, social playTalismansDisabled in PvP (5 min)Level playing field in PvPTown ControlTemporary (5 min)Rewards active playSeasonsQuarterly resetMeaningful progression + fresh startsNew Players2-week protection + quest skillsFast to competence
1.3 Target Metrics
MetricTargetNotesMax Concurrent Players5,000Realistic ceilingAverage Concurrent~1,000Expected steady stateTime to PvP Ready2-3 hoursVia starter questsMicrotick Precision50Hz (20ms)Combat timingServer Tick Budget<10ms @ 35% loadPerformance target

2. Core Design Principles
2.1 PvP/PvM Separation
Fundamental Rule: PvM advantages do not apply in PvP combat.
csharppublic static bool IsPvPContext(Mobile caster, Mobile target)
{
    // Direct player vs player
    if (caster is PlayerMobile && target is PlayerMobile)
        return true;
        
    // Player vs player-controlled creature
    if (target is BaseCreature bc && bc.ControlMaster is PlayerMobile)
        return true;
        
    // Player-controlled creature vs player
    if (caster is BaseCreature bc2 && bc2.ControlMaster is PlayerMobile 
        && target is PlayerMobile)
        return true;
        
    return false;
}
When PvP Context is Detected:

Talisman bonuses deactivate (5-minute timer)
Chivalry spell access removed (Sampire build)
Pet movement speed reduced 50%
Base damage only (no PvM multipliers)

2.2 Guild-Centric Design

Solo players cannot participate in factions
Guilds choose faction allegiance (7-day change cooldown)
All guild members inherit faction
Encourages social play and coordination

2.3 Skill-Based Gameplay

Fast skill acquisition via quests (not grinding)
Punishing fizzle mechanics (resource consumption)
Free movement during casting (Sphere-style)
Glicko-2 prevents farming low-skill players


3. Combat System
3.1 50Hz Microtick Engine
All combat calculations run at 50Hz (20ms intervals) for precise timing.
csharppublic class CombatTickEngine
{
    private const int TickRateHz = 50;
    private const int TickIntervalMs = 1000 / TickRateHz; // 20ms
    
    // All high-frequency systems unified at 50Hz
    // - Spell state machine
    // - Movement validation
    // - Combat resolution
    // - Resource gathering monitoring
}
3.2 Movement
Rule: Players can ALWAYS move unless paralyzed.

Movement during spell casting: Allowed
Movement during bandaging: Allowed
Movement restrictions: Only paralysis spell/effect

3.3 Combat Interruption
Combat actions that cause spell fizzle:

Casting another spell while one is in flight
Toggling War Mode on/off
Applying a bandage
Losing Line of Sight when spell should land

Fizzle Consequence: Mana consumed, reagents/scroll consumed. No refunds.
3.4 Pets in Combat

Pets use base damage only (no talisman bonuses)
Pet speed reduced 50% during PvP context (5-minute duration)
Mounts auto-dismiss in dungeons, reappear on exit
Pets allowed in all dungeons


4. Spell System
4.1 Sphere-Style Casting Flow
OSI Style:    Cast → Freeze → Target → Resolution
Sphere Style: Target → Cast → Free Movement → Resolution (51alpha uses this)
4.2 Spell State Machine
csharppublic enum SpellState
{
    Idle,           // No spell active
    Targeting,      // Selecting target (pre-cast)
    Initiated,      // Cast command received
    Preparing,      // Spell delay countdown
    Casting,        // Final cast animation
    Completed       // Spell resolved
}
4.3 Spell Damage Calculation
csharppublic int GetSpellDamage(Mobile caster, Mobile target, int baseDamage)
{
    int damage = baseDamage;
    
    // PvM bonuses ONLY apply outside PvP context
    if (!IsPvPContext(caster, target))
    {
        if (caster is PlayerMobile pm)
        {
            // Talisman bonus
            damage = BuildManager.ApplyTalismanBonus(damage, pm);
            
            // Honor bonus
            damage = ApplyHonorBonus(damage, pm);
            
            // Enemy of One (Chivalry)
            damage = ApplyEnemyOfOne(damage, pm, target);
        }
    }
    
    // Base damage scaling always applies
    damage = ApplyEvalIntBonus(damage, caster);
    damage = ApplyResistanceReduction(damage, target);
    
    return damage;
}
4.4 Chivalry Access
Critical Rule: Chivalry spells require an active Sampire talisman.
csharppublic static bool CanCastChivalrySpell(PlayerMobile caster)
{
    var talisman = caster.FindItemOnLayer(Layer.Talisman) as BaseTalisman;
    
    // Must have Sampire talisman equipped
    if (talisman?.TalismanType != TalismanType.Sampire)
        return false;
        
    // Talisman must be active (not PvP disabled)
    if (!talisman.IsActive)
    {
        caster.SendMessage(0x22, "Your talisman is inactive. Chivalry spells are unavailable.");
        return false;
    }
    
    return true;
}
4.5 AoE and PvP Transition
Rule: If a player runs into an AoE targeted at monsters, BOTH players enter PvP context.
csharppublic void OnAoEDamage(Mobile caster, Mobile target, int damage)
{
    // If AoE hits a player (even unintentionally)
    if (target is PlayerMobile victim && caster is PlayerMobile attacker)
    {
        // Both enter PvP context
        TalismanManager.TriggerPvPDisable(attacker);
        TalismanManager.TriggerPvPDisable(victim);
    }
}

5. Talisman System
5.1 Talisman Types
TypePlaystyleKey BonusChivalry AccessDexerMelee DPS+% weapon damageNoTamerPet commands+% pet damageNoSampireChivalry/LeechLife leech, Chivalry spellsYesTreasure HunterExploration+% chest qualityNo
5.2 Acquisition

Farm relics from dungeons (see Relic Drop Rates)
Craft talisman using relics + crafting skill
Equip (only one at a time)

5.3 Timer Mechanics

Timer does NOT start until first equip
Talismans can be traded/sold before first equip
Once equipped, timer begins countdown
Timer PAUSES when unequipped (may adjust for balance later)

5.4 PvP Disable Mechanic
csharppublic static class TalismanManager
{
    private static readonly TimeSpan PvPDisableDuration = TimeSpan.FromMinutes(5);
    
    public static void TriggerPvPDisable(PlayerMobile player)
    {
        var talisman = player.FindItemOnLayer(Layer.Talisman) as BaseTalisman;
        if (talisman == null)
            return;
            
        talisman.IsActive = false;
        talisman.ReactivationTime = DateTime.UtcNow + PvPDisableDuration;
        
        player.SendMessage(0x22, "Your talisman has been disabled for 5 minutes due to PvP combat!");
        
        // Also reduce pet speed
        if (player.AllFollowers != null)
        {
            foreach (var pet in player.AllFollowers.OfType<BaseCreature>())
            {
                pet.ApplyPvPSpeedDebuff(PvPDisableDuration);
            }
        }
    }
}

6. Faction System (VvV)
6.1 Structure

3 Factions: Vampire, Daemon, Goblin
Guild-based membership (solo players cannot participate)
7-day cooldown on faction changes
Peaceful Participant Option: Guild members can opt for peaceful status (no PvP outside sieges)

6.2 Siege System
See VvV_Siege_System.md for complete specification.
SettingValueBattle Model3-way free-for-allSiege CitiesJhelom, Skara Brae, Yew, TrinsicSiege TriggerOn-demand (minimum players)Siege Duration30 minutesVictory Score10,000 pointsTown ControlPersistent until capturedNPC Discount10% (15% if all 4 cities)
Dual Currency System
CurrencySourceUseTradeableFaction PointsAll activitiesSeasonal rankingsNoSilverPvP kills, objectivesTraps, turrets, cosmeticsYes
Peaceful Participant
Guild members can choose Peaceful status at guild stone:

Cannot attack or be attacked by enemy factions outside siege zones
Auto-flagged when entering active siege
Still contributes via BODs, bounties, faction quests
24-hour cooldown on status changes

6.2 Sigil Battles

Multiple per day (frequency configurable at runtime)
Capture mechanic: Stand near sigil for 30-45 seconds
Interruption: Taking damage cancels capture
Reward: Faction points + temporary town control

6.3 Town Control

Duration: 5 minutes after sigil capture
Benefit: 10-15% NPC vendor discount for faction members
Visual: Faction banners appear in town
Automatic expiry: Returns to neutral after 5 minutes

6.4 Underdog Bonuses
StandingPoint BonusDiscount1st Place0%10%2nd Place+2%12%3rd Place+5%15%
6.5 Daily Faction Content
Daily Bounties (3 tasks, individual rewards)

Kill X of [random monster type]
Gather Y of [random resource]
Kill Z of [different monster type]
Same tasks for all players each day
Reward: Faction points per task completed

Daily Faction Quest

Kill mini-boss at one of 4 swamp/desert locations
Location is announced to players (not a search)
Integrates with Swamp_and_Desert_Event.md content
Reward: Bonus faction points
Can be completed once per day per player

Rare Faction Deco

0.01% drop chance from any monster
Universal faction clothing items
Only equippable by faction members (guild required)

6.6 Seasonal System

Season length: Quarterly (3 months)
Monthly: Leaderboard rankings (cosmetic)
Quarterly: Full reset

Faction points reset to 0
Glicko ratings soft reset (compress 25% toward 1500)
Seasonal rewards distributed
Town control unaffected (temporary anyway)




7. Dungeon System
7.1 Dungeon Levels
LevelDifficultyMonster TierRelic Drops1EntryTrivial-EasyCommon only2EasyEasy-ModerateCommon, Uncommon3MediumModerateAll tiers4HardHardAll tiers, better rates5ChampionChampion/BossBest rates
7.2 Weekly Rotation
Two dungeons rotate each week:

Bonus Dungeon: 2x gold, 2x relic drop rates
Safe Dungeon: Reduced drops, but NO PK/stealing allowed

Implementation: Safe dungeon entrance teleports to Trammel copy (inherits ruleset).
All other dungeons: Normal rates, full PvP enabled.
7.3 Mount Behavior

Mounts auto-dismiss on dungeon entry
Mounts auto-summon on dungeon exit
Pets (non-mount) allowed in dungeons

7.4 Relic Drop Rates
Base Rates by Dungeon Level
LevelCommonUncommonRareEpic11.5%0.2%0%0%22.0%0.4%0.05%0%32.5%0.6%0.12%0.01%43.0%1.0%0.20%0.03%54.0%1.5%0.35%0.06%
Boss Multipliers
Boss TypeMultiplierMini-Boss2.0xDungeon Boss3.0xWorld Boss5.0xEvent Boss4.0x
Rotation Bonuses (Bonus Dungeon of the Week)
TierBonusCommon2.0xUncommon2.0xRare1.5xEpic1.25x

8. Economy Model
8.1 Gold Flow Principle
Target: Slight deflationary pressure (2-5% monthly gold drain)
8.2 Gold Sources (Faucets)
SourceGold/HourNotesMonster Drops (L1-2)5,000-10,000Entry contentMonster Drops (L3-4)15,000-30,000Mid-gameMonster Drops (L5)40,000-80,000End-gameTreasure Chests20,000-100,000Per chestBOD Rewards10,000-50,000Per completionStarter Quest Completion5,000One-time
8.3 Gold Sinks (Drains)
SinkCostTypeNPC Reagents5-15g eachConsumableNPC Arrows/Bolts2-5g eachConsumableArmor/Weapon Repair100-10,000gPer deathHouse Placement50k-3MOne-timePoker House Take5% of potGamblingDuel Pit House Take2% of betGambling
8.4 Key Economic Decisions
ItemDecisionHouse TaxesNonePoker rake destinationDestroyed (gold sink)Duel rake destinationDestroyed (gold sink)Relics tradeableYesTalisman tradeableYes (before first equip)
8.5 Transfer Systems (Not Faucets/Sinks)

Player-to-player trading
Duel winnings (minus 2% rake)
Poker winnings (minus 5% rake)


9. Housing & Relic System
9.1 House Tiers
TierRelic RequirementsGold CostTime EstimateSmall20 Common, 5 Uncommon50,0001 weekMedium50 Common, 15 Uncommon, 2 Rare150,0002-3 weeksLarge100 Common, 40 Uncommon, 8 Rare, 1 Epic400,0001-2 monthsVilla200 Common, 80 Uncommon, 20 Rare, 3 Epic800,0002-3 monthsKeep400 Common, 150 Uncommon, 50 Rare, 8 Epic1,500,0004-6 monthsCastle800 Common, 300 Uncommon, 100 Rare, 20 Epic3,000,0008-12 months
9.2 Relic Properties

Tradeable: Yes
Stackable: Yes (by tier)
Freshness timer: None (removed)
Loss on death: Yes (if in backpack in PvP zone)


10. New Player Experience
10.1 Young Player Protection
SettingValueDuration2 weeks (calendar time)PvP ProtectionCannot attack or be attackedTheft ProtectionCannot be stolen fromCorpse ProtectionCannot be lootedFaction AccessBlocked until renouncedEarly TerminationPlayer can renounce anytime
10.2 Starter Quest System
Location: Quest Ferry at Britain Docks → Training Islands (Trammel)
QuestIslandTargetKillsSkill Reward1Rabbit IslandRabbits20100 Swordsmanship2Skeleton IsleSkeletons15100 Tactics3Orc CampOrcs10100 Mace Fighting4Lizardman LairLizardmen10100 Fencing5Imp GrottoImps590 Magery
Completion Bonus: 5,000 gold
Anti-Farming: No gold drops on starter islands, cannot re-enter after completion.
10.3 Post-Quest State
After completing all quests, players have:

100 Swordsmanship
100 Tactics
100 Mace Fighting
100 Fencing
90 Magery

Still need to train:

Magery (final 10 points)
Resisting Spells
Evaluating Intelligence
Meditation
Healing/Anatomy
Any crafting skills


11. Crafting & BOD System
11.1 BOD Point Timing
Decision: Points credited immediately on BOD completion.
11.2 BOD Authenticity Verification
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
11.3 Talisman Crafting
Materials Required:

Rare relics from dungeons
Standard crafting materials
Crafting skill requirements (TBD per talisman type)


12. Glicko-2 Rating System
12.1 Purpose
NOT for matchmaking - Anyone can fight anyone.
Purpose: Prevent high-skill players from farming points off new players by weighting kill rewards.
12.2 Kill Point Multipliers
Rating DifferencePoint Multiplier+400 or more (farming)0.25x+200 to +3990.50x+100 to +1990.75x-99 to +99 (fair fight)1.00x-199 to -1001.25x-399 to -2001.50x-400 or more (underdog)2.00x
12.3 Adapted Parameters (Small Population)
ParameterStandard51alphaDefault Rating15001500Default RD350350Rating Period1 month3 daysTau0.50.75Min RD3050
12.4 Seasonal Reset

Compress rating 25% toward 1500: new = 1500 + (old - 1500) * 0.75
Increase RD by 50

12.5 Future Use: Team Balancing
For future CTF/KotH modes, Glicko ratings will balance teams automatically.

13. Social Systems
13.1 Guilds

Required for faction participation
Guild leader chooses faction
7-day cooldown on faction changes
Guild leader departure: Faction persists, new leader can change

13.2 Duel Pits

Location: Specific arena locations only
Betting: Gold wagers between combatants
House Take: 2% of bet (destroyed as gold sink)
No rating integration: Casual betting system
No matchmaking: Challenge anyone

13.3 Texas Hold'em Poker

Location: Specific taverns only
House Take: 5% of pot (destroyed as gold sink)
Social feature: No faction integration


14. Tournament System
14.1 Overview
Automated 1v1 single-elimination tournaments run on a fixed schedule with spectator betting and unique cosmetic rewards.
14.2 Schedule
DayTimes (EST)Target RegionWednesday2 PM, 8 PMEU, NASaturday2 PM, 8 PMEU, NASunday2 PM, 8 PMEU, NA
Total: 6 tournaments per week
14.3 Registration

Opens 15 minutes before tournament
Closes at tournament start
Free entry (no fee)
Minimum 6 players (no maximum)
Young players cannot participate

14.4 Match Rules
RuleSettingSelectionRandom (no seeding)FormatSingle eliminationDurationNo time limit (fight to death)Arenas8 parallel (expandable)ByesRound 1 only (early, not late)RefreshPlayers auto-healed between roundsDisconnectCharacter stays in-game
14.5 Rewards
RewardDescriptionTrophy StatueHouse deco showing winner, date, participants100 Tournament CoinsCosmetic currencyTitle"Tournament Champion" for 1 week (toggleable)
Kill Points: Faction points awarded for kills (Glicko weighted), but NO participation rewards.
14.6 Tournament Coins
Spent at Tournament NPC for cosmetic transformations:

100 coins: Transform helmet → hat (keeps AR)
100 coins: Transform helmet → mask (keeps AR)
More options to be added

14.7 Spectator Betting

Player-to-player betting only
5% house rake (gold sink)
Can bet on any pending match
Cannot bet on own matches
Cannot bet once match starts

14.8 Glicko Integration
UseEnabledPlayer selectionNo (random)Kill point weightingYesPost-match rating updateYesFuture team balancingPlanned

15. Infrastructure & Performance
14.1 Performance Targets
SystemTargetNotesMicrotick cycle<10ms @ 35% load50Hz engineDatabase connection<5msConnection poolingPlayer read ops<10ms medianHot pathPlayer write ops<25msBatchedConcurrent connections5,000+PgBouncer
14.2 Database Configuration
yaml# PgBouncer (connection pooler)
pool_mode: transaction
max_client_conn: 5000
default_pool_size: 50

# PostgreSQL
max_connections: 200
shared_buffers: 256MB
14.3 Redis Configuration

Cache for leaderboards (5-second TTL)
Graceful degradation on failure
Background health monitoring

csharppublic async Task<T> GetOrFallback<T>(string key, Func<Task<T>> fallback)
{
    if (!_redisAvailable)
        return await fallback();
        
    try { /* Redis get */ }
    catch (RedisConnectionException)
    {
        _redisAvailable = false;
        _ = Task.Run(MonitorRedisHealth);
        return await fallback();
    }
}
14.4 VvV Battle Processing Budget

Target: 5ms per battle with 100 participants
Priority 1: Sigil state (always process)
Priority 2: Point calculations (defer if over budget)
Priority 3: UI updates (skip if over budget)


16. Security
15.1 Authentication Flow

Discord OAuth → Launcher
JWT Token (1hr expiry) → Website/API
Refresh Token (30 day expiry)
Game API Key (permanent until revoked)

15.2 Anti-Exploit

Crafting hash validation (HMAC signatures)
Idempotent gold transfers (prevent double-spend)
Server-side validation of all actions
CorrelationId tracking for audit

15.3 Gold Transfer Safety
csharppublic TransferResult TransferGold(Guid transferId, PlayerMobile from, PlayerMobile to, int amount)
{
    // Idempotency check
    if (_processedTransfers.Contains(transferId))
        return TransferResult.AlreadyProcessed;
        
    lock (GetLockObject(from, to))
    {
        // Atomic transfer
        if (from.BankBox.TotalGold < amount)
            return TransferResult.InsufficientFunds;
            
        from.BankBox.ConsumeTotal(typeof(Gold), amount);
        to.BankBox.DropItem(new Gold(amount));
        
        _processedTransfers.Add(transferId);
        return TransferResult.Success;
    }
}

17. NPC Systems
16.1 Town Cryer

Locations: Major towns (Britain, Trinsic, Moonglow, Skara Brae, Jhelom)
Announcement Range: 15 tiles
Per-Player Cooldown: 10 minutes (anti-spam)
Categories:

Rotating content (weekly dungeons)
Faction events (sigil battles)
Server events (admin-triggered)



16.2 Quest Ferry NPC

Location: Britain Docks (Trammel)
Function: Transport to starter islands
Access Control: Only current quest island accessible

16.3 New Player Guide NPC

Location: Britain Bank (Trammel)
Function: Help menu, wiki link, young status management


18. Configuration Reference
17.1 Combat Configuration
csharppublic static class CombatConfig
{
    public const int TickRateHz = 50;
    public const int TickIntervalMs = 20;
    public static TimeSpan TalismanPvPDisable = TimeSpan.FromMinutes(5);
    public const double PetPvPSpeedReduction = 0.50; // 50% slower
}
17.2 VvV Configuration
csharppublic static class VvVConfig
{
    public static TimeSpan TimeBetweenBattles = TimeSpan.FromMinutes(60);
    public static TimeSpan BattleDuration = TimeSpan.FromMinutes(20);
    public static TimeSpan TownControlDuration = TimeSpan.FromMinutes(5);
    public static TimeSpan SigilCaptureTime = TimeSpan.FromSeconds(30);
    public static TimeSpan SigilContestTime = TimeSpan.FromSeconds(45);
    
    public static int PointsForSigilCapture = 500;
    public static int PointsPerKill = 100; // Before Glicko weighting
}
17.3 Faction Configuration
csharppublic static class FactionConfig
{
    public static TimeSpan FactionChangeCooldown = TimeSpan.FromDays(7);
    public static TimeSpan SeasonLength = TimeSpan.FromDays(90); // Quarterly
    
    // Underdog bonuses
    public static double FirstPlaceBonus = 0.00;
    public static double SecondPlaceBonus = 0.02;
    public static double ThirdPlaceBonus = 0.05;
}
17.4 New Player Configuration
csharppublic static class YoungPlayerConfig
{
    public static TimeSpan YoungDuration = TimeSpan.FromDays(14);
    public static bool CanBeAttackedByPlayers = false;
    public static bool CanAttackPlayers = false;
    public static bool CanJoinFaction = false;
}
18.5 Tournament Configuration
csharppublic static class TournamentConfig
{
    // Timing
    public static TimeSpan RegistrationWindow = TimeSpan.FromMinutes(15);
    public static TimeSpan PostFinalWait = TimeSpan.FromSeconds(30);
    
    // Participation
    public static int MinimumPlayers = 6;
    public static int MaximumPlayers = int.MaxValue; // Uncapped
    
    // Arenas
    public static int ArenaCount = 8;
    
    // Rewards
    public static int WinnerCoinReward = 100;
    public static TimeSpan TitleDuration = TimeSpan.FromDays(7);
    
    // Betting
    public static double BettingHouseRake = 0.05; // 5%
    public static int MinimumBet = 1000;
    
    // Schedule (EST)
    public static TimeSpan[] TournamentTimes = { TimeSpan.FromHours(14), TimeSpan.FromHours(20) };
    public static DayOfWeek[] TournamentDays = { DayOfWeek.Wednesday, DayOfWeek.Saturday, DayOfWeek.Sunday };
}

19. Implementation Phases
Phase 1: Core Combat (Weeks 1-4)

 50Hz microtick engine
 Sphere-style spell FSM
 Fizzle mechanics (resource consumption)
 Free movement during casting
 PvP/PvM context detection

Phase 2: Faction Foundation (Weeks 5-8)

 3-faction structure
 Guild-faction binding
 VvV sigil capture (from ModernUO base)
 Town control (5 min temporary)
 Basic point scoring

Phase 3: Talisman System (Weeks 9-12)

 Talisman types (Dexer, Tamer, Sampire, TH)
 PvP disable mechanic (5 min)
 Chivalry gating
 Relic drop system
 Talisman crafting

Phase 4: New Player Experience (Weeks 13-16)

 Young player protection (2 weeks)
 Ferry quest system
 5 training islands
 Skill reward system
 New Player Guide NPC

Phase 5: Economy & Housing (Weeks 17-20)

 Relic-based house upgrades
 Gold sink implementation
 Duel pit betting
 Poker tables
 Economy monitoring

Phase 6: Tournament System (Weeks 21-24)

 Tournament arena layout
 Registration system
 Bracket generation
 Match processing
 Trophy reward item
 Tournament Coin currency
 Cosmetic shop NPC
 Spectator betting
 Scheduling system

Phase 7: Polish & Launch (Weeks 25-28)

 Town Cryer NPC
 Daily bounties
 Faction quests
 Wiki documentation
 Performance optimization
 Security audit
 Load testing


Appendix A: Resolved Conflicts
IDConflictResolution001Talisman/ChivalryChivalry requires active Sampire talisman002BOD timingPoints credited immediately003Dungeon/BOD materialsDungeons rotate bonuses, not access004Tick ratesAll systems unified at 50Hz005Relic tiersMapping defined in section 7.4006Duel/FactionIntentionally separate007Glicko/Season3-day rating period, quarterly soft reset
Appendix B: Removed Features
FeatureReasonML spell predictionComplexity, no training pipelineInterrupt throttleNatural fizzle mechanics sufficientRelic freshness timerUnnecessary complexityHouse taxesDesign decisionPuzzle clue systemDeferred to future
Appendix C: Open Items (TBD)
ItemStatusTournament Stadium layout/coordinatesDesign neededTournament cosmetic options expansionFuture contentAdditional starter questsTesting will determineFaction-specific visual themesArt direction needed

Document History
VersionDateChanges1.0.02024-12Initial architecture specification2.0.02025-01-02Comprehensive cohesion review, conflict resolution, professional specifications added