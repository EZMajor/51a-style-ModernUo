# VvV (Virtue vs Vice) Integration Specification
## 51alpha Faction Warfare System

### Document Metadata
- **Version**: v1.0.0
- **Last Updated**: 2025-01-02
- **Authors**: 51alpha Development Team
- **Applicable ModernUO Version**: v24.0.0+
- **PR Ready**: No (Conceptual Design)
- **Base System**: ModernUO VvV Engine (UOContent/Engines/VvV)

---

## 1. Executive Summary

The 51alpha faction system adapts ModernUO's existing Virtue vs Vice (VvV) engine to support a **3-faction guild-based territorial competition** with sigil capture events, Glicko-2 weighted point scoring, and quarterly seasonal rewards. This document specifies all customizations required to transform the base VvV system into the 51alpha faction experience.

### Key Design Decisions
| Decision | Choice | Rationale |
|----------|--------|-----------|
| Faction Count | 3 | Enables natural 2v1 underdog dynamics |
| Participation | Guild-based | Encourages group play and coordination |
| Town Control | Temporary (5 min) | Rewards active play, not camping |
| Point Weighting | Glicko-2 | Prevents farming low-skill players |
| Season Length | Quarterly (3 months) | Meaningful progression + fresh starts |

---

## 2. Faction Structure

### 2.1 The Three Factions

| Faction | Theme | Color | Primary Town |
|---------|-------|-------|--------------|
| **Vampire** | Blood, Night, Immortality | Red (0x21) | Britain |
| **Daemon** | Fire, Chaos, Power | Orange (0x30) | Trinsic |
| **Goblin** | Cunning, Trade, Resourcefulness | Green (0x3F) | Moonglow |

### 2.2 Related Documents

- **VvV_Siege_System.md**: Complete city siege warfare specification
  - 3-way free-for-all battles in Jhelom, Skara Brae, Yew, Trinsic
  - Persistent town control with NPC discounts (10-15%)
  - Trap and turret defense systems
  - Silver currency for PvP rewards
  - Peaceful participant option for non-PvP guild members

### 2.2 Guild Faction Membership

```csharp
public class FactionGuildManager
{
    // Cooldown between faction changes
    private static readonly TimeSpan FactionChangeCooldown = TimeSpan.FromDays(7);
    
    public static bool CanChangeFaction(Guild guild)
    {
        if (guild.LastFactionChange == DateTime.MinValue)
            return true; // Never changed before
            
        return DateTime.UtcNow - guild.LastFactionChange > FactionChangeCooldown;
    }
    
    public static FactionChangeResult ChangeFaction(Guild guild, Faction newFaction, PlayerMobile guildLeader)
    {
        // Validate guild leader
        if (guild.Leader != guildLeader)
            return FactionChangeResult.NotGuildLeader;
            
        // Check cooldown
        if (!CanChangeFaction(guild))
        {
            var remaining = FactionChangeCooldown - (DateTime.UtcNow - guild.LastFactionChange);
            return FactionChangeResult.CooldownActive(remaining);
        }
        
        // Process change
        var oldFaction = guild.Faction;
        guild.Faction = newFaction;
        guild.LastFactionChange = DateTime.UtcNow;
        
        // Notify all guild members
        foreach (var member in guild.Members)
        {
            member.SendMessage(0x35, $"Your guild has joined {newFaction.Name}!");
            UpdateFactionEquipment(member, oldFaction, newFaction);
        }
        
        // Log for analytics
        FactionAnalytics.LogFactionChange(guild, oldFaction, newFaction);
        
        return FactionChangeResult.Success;
    }
    
    // When player joins guild, they inherit faction
    public static void OnPlayerJoinGuild(PlayerMobile player, Guild guild)
    {
        if (guild.Faction != null)
        {
            player.SendMessage(0x35, $"You are now a member of {guild.Faction.Name}!");
        }
    }
    
    // When player leaves guild, they lose faction access
    public static void OnPlayerLeaveGuild(PlayerMobile player, Guild oldGuild)
    {
        if (oldGuild.Faction != null)
        {
            player.SendMessage(0x22, "You have left your faction. Join a guild to participate in faction warfare.");
            RemoveFactionEquipment(player);
        }
    }
}
```

### 2.3 Solo Player Restriction

```csharp
// Solo players cannot participate in faction content
public static bool CanParticipateInFaction(PlayerMobile player)
{
    if (player.Guild == null)
    {
        player.SendMessage(0x22, "You must be in a guild to participate in faction warfare.");
        return false;
    }
    
    if (player.Guild.Faction == null)
    {
        player.SendMessage(0x22, "Your guild has not chosen a faction. Ask your guild leader to select one.");
        return false;
    }
    
    return true;
}
```

---

## 3. Sigil Battle System

### 3.1 Sigil Locations

Sigils spawn at preset locations across the world. These are the objectives for faction PvP.

| Location | Map | Coordinates | Terrain Type |
|----------|-----|-------------|--------------|
| Britain Crossroads | Felucca | 1432, 1698 | Urban |
| Trinsic South Gate | Felucca | 1836, 2780 | Urban |
| Moonglow Mage Tower | Felucca | 4445, 1128 | Arcane |
| Skara Brae Docks | Felucca | 642, 2236 | Coastal |
| Jhelom Arena | Felucca | 1333, 3784 | Arena |
| Yew Crypts | Felucca | 548, 1024 | Forest |
| Minoc Mines | Felucca | 2580, 510 | Mountain |
| Vesper Bridge | Felucca | 2892, 684 | Bridge |

*Minimum 8 sigil locations for variety. Add more as needed.*

### 3.2 Battle Frequency & Configuration

```csharp
public static class VvVConfig
{
    // Battle timing - adjustable at runtime via admin command
    public static TimeSpan TimeBetweenBattles { get; set; } = TimeSpan.FromMinutes(60);
    public static TimeSpan BattlePreparationTime { get; set; } = TimeSpan.FromMinutes(5);
    public static TimeSpan BattleDuration { get; set; } = TimeSpan.FromMinutes(20);
    public static TimeSpan TownControlDuration { get; set; } = TimeSpan.FromMinutes(5);
    
    // Minimum participants to start
    public static int MinimumParticipantsPerFaction { get; set; } = 3;
    public static int MinimumTotalParticipants { get; set; } = 6;
    
    // Points
    public static int PointsForSigilCapture { get; set; } = 500;
    public static int PointsPerKill { get; set; } = 100; // Base, before Glicko weighting
    public static int PointsPerSigilHoldMinute { get; set; } = 50;
    
    // Admin commands
    [Usage("[vvv config <setting> <value>")]
    public static void ConfigCommand(CommandEventArgs e)
    {
        // Runtime configuration adjustment
        // Example: [vvv config TimeBetweenBattles 30
    }
}
```

### 3.3 Battle Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SIGIL BATTLE LIFECYCLE                           │
└─────────────────────────────────────────────────────────────────────────┘

  SCHEDULED          PREPARATION         ACTIVE              RESOLUTION
  ─────────────────►─────────────────►─────────────────►─────────────────►
  
  │                 │                  │                  │
  │ Town Cryer      │ Sigil spawns     │ Capture/Contest  │ Winner declared
  │ announces       │ at location      │ Kill scoring     │ Town control (5m)
  │ next battle     │                  │ Point accumulate │ Points awarded
  │                 │ Players gather   │                  │ Quarterly totals
  │ -5 minutes      │ -1 minute        │ 20 minutes       │ +5 minutes
  │                 │                  │                  │
```

### 3.4 Sigil Capture Mechanics

```csharp
public class VvVSigil : Item
{
    private Faction _controllingFaction;
    private DateTime _captureStartTime;
    private PlayerMobile _capturingPlayer;
    
    // Time required to capture unclaimed sigil
    private static readonly TimeSpan CaptureTime = TimeSpan.FromSeconds(30);
    
    // Time required to steal sigil from another faction
    private static readonly TimeSpan ContestTime = TimeSpan.FromSeconds(45);
    
    public override bool OnDragLift(Mobile from)
    {
        if (!(from is PlayerMobile pm))
            return false;
            
        if (!FactionGuildManager.CanParticipateInFaction(pm))
            return false;
            
        // Start capture attempt
        BeginCapture(pm);
        return false; // Don't actually pick up - just start capture
    }
    
    private void BeginCapture(PlayerMobile player)
    {
        var playerFaction = player.Guild?.Faction;
        
        if (playerFaction == _controllingFaction)
        {
            player.SendMessage("Your faction already controls this sigil!");
            return;
        }
        
        _capturingPlayer = player;
        _captureStartTime = DateTime.UtcNow;
        
        var captureTime = _controllingFaction == null ? CaptureTime : ContestTime;
        
        player.SendMessage($"Capturing sigil... Hold position for {captureTime.TotalSeconds} seconds!");
        
        // Visual effect
        Effects.SendLocationParticles(
            EffectItem.Create(Location, Map, EffectItem.DefaultDuration),
            0x376A, 9, 32, playerFaction.Color, 0, 5039, 0
        );
        
        // Start capture timer
        Timer.StartTimer(captureTime, () => CompleteCaptureAttempt(player, playerFaction));
    }
    
    private void CompleteCaptureAttempt(PlayerMobile player, Faction faction)
    {
        // Validate player is still in range and alive
        if (!player.Alive || player.GetDistanceToSqrt(this) > 3)
        {
            player.SendMessage(0x22, "Capture interrupted! You moved too far from the sigil.");
            _capturingPlayer = null;
            return;
        }
        
        // Success!
        var previousFaction = _controllingFaction;
        _controllingFaction = faction;
        _capturingPlayer = null;
        
        // Award capture points
        FactionPointManager.AwardPoints(player, VvVConfig.PointsForSigilCapture, "Sigil Capture");
        
        // Broadcast
        World.Broadcast(0x35, true, $"{faction.Name} has captured the {GetSigilName()} sigil!");
        
        // Town Cryer announcement
        TownCryerManager.AnnounceEvent(TownCryerCategory.FactionEvent, 
            $"{faction.Name} seizes control of {GetLocationName()}!");
    }
    
    // Interrupt capture if capturer takes damage
    public void OnCapturerDamaged(PlayerMobile attacker)
    {
        if (_capturingPlayer != null)
        {
            _capturingPlayer.SendMessage(0x22, "Capture interrupted by combat!");
            _capturingPlayer = null;
        }
    }
}
```

### 3.5 Town Control Benefits

When a faction captures a sigil, they gain temporary (5 minute) control of the associated town:

```csharp
public class TownControlManager
{
    private static Dictionary<Town, (Faction Faction, DateTime ExpiresAt)> _activeControl = new();
    
    public static void OnSigilCaptured(VvVSigil sigil, Faction faction)
    {
        var town = GetTownForSigil(sigil);
        
        _activeControl[town] = (faction, DateTime.UtcNow.Add(VvVConfig.TownControlDuration));
        
        // Apply town benefits
        ApplyTownBenefits(town, faction);
        
        // Schedule expiration
        Timer.StartTimer(VvVConfig.TownControlDuration, () => ExpireTownControl(town));
    }
    
    private static void ApplyTownBenefits(Town town, Faction faction)
    {
        // Faction vendors in this town offer discounts
        foreach (var vendor in town.Vendors)
        {
            vendor.SetFactionDiscount(faction, GetDiscountPercentage(faction));
        }
        
        // Visual indicator - faction banners appear
        SpawnFactionBanners(town, faction);
        
        // Broadcast
        World.Broadcast(0x35, true, 
            $"{faction.Name} controls {town.Name}! Faction members receive vendor discounts for 5 minutes!");
    }
    
    private static void ExpireTownControl(Town town)
    {
        if (_activeControl.TryGetValue(town, out var control))
        {
            // Remove benefits
            foreach (var vendor in town.Vendors)
            {
                vendor.ClearFactionDiscount();
            }
            
            // Remove banners
            RemoveFactionBanners(town);
            
            _activeControl.Remove(town);
            
            World.Broadcast(0x35, true, $"{town.Name} has returned to neutral control.");
        }
    }
    
    // Discount based on faction standing (underdog bonus built in)
    private static double GetDiscountPercentage(Faction faction)
    {
        int standing = FactionStandingManager.GetCurrentStanding(faction);
        
        return standing switch
        {
            1 => 0.10, // 1st place: 10% discount
            2 => 0.12, // 2nd place: 12% discount (2% underdog bonus)
            3 => 0.15, // 3rd place: 15% discount (5% underdog bonus)
            _ => 0.10
        };
    }
}
```

---

## 4. Point Scoring System

### 4.1 Glicko-2 Weighted Kill Points

```csharp
public static class FactionPointManager
{
    public static void OnPlayerKill(PlayerMobile killer, PlayerMobile victim)
    {
        // Validate both are in factions
        if (killer.Guild?.Faction == null || victim.Guild?.Faction == null)
            return;
            
        // Must be different factions
        if (killer.Guild.Faction == victim.Guild.Faction)
            return;
            
        // Calculate Glicko-weighted points
        int basePoints = VvVConfig.PointsPerKill;
        double multiplier = CalculateGlickoMultiplier(killer, victim);
        int finalPoints = (int)(basePoints * multiplier);
        
        // Award points
        AwardPoints(killer, finalPoints, $"Kill: {victim.Name}");
        
        // Update Glicko ratings
        GlickoManager.RecordMatch(killer, victim, winner: killer);
    }
    
    private static double CalculateGlickoMultiplier(PlayerMobile killer, PlayerMobile victim)
    {
        double ratingDiff = killer.GlickoRating - victim.GlickoRating;
        
        // Scaled multiplier based on rating difference
        return ratingDiff switch
        {
            >= 400 => 0.25,   // Much higher rated killer
            >= 200 => 0.50,   // Higher rated
            >= 100 => 0.75,   // Slightly higher
            >= -100 => 1.00,  // Similar skill
            >= -200 => 1.25,  // Slightly lower (underdog)
            >= -400 => 1.50,  // Lower rated (underdog)
            _ => 2.00         // Much lower rated (major underdog)
        };
    }
    
    public static void AwardPoints(PlayerMobile player, int points, string reason)
    {
        var faction = player.Guild?.Faction;
        if (faction == null)
            return;
            
        // Apply underdog bonus based on faction standing
        int standing = FactionStandingManager.GetCurrentStanding(faction);
        double standingMultiplier = standing switch
        {
            1 => 1.00, // 1st place: no bonus
            2 => 1.02, // 2nd place: 2% bonus
            3 => 1.05, // 3rd place: 5% bonus
            _ => 1.00
        };
        
        int finalPoints = (int)(points * standingMultiplier);
        
        // Award to player
        player.FactionPoints += finalPoints;
        
        // Award to faction total
        faction.SeasonPoints += finalPoints;
        
        // Log
        player.SendMessage(0x35, $"+{finalPoints} faction points ({reason})");
        
        // Analytics
        FactionAnalytics.LogPointAward(player, faction, finalPoints, reason);
    }
}
```

### 4.2 Point Sources Summary

| Source | Base Points | Glicko Weighted | Underdog Bonus |
|--------|-------------|-----------------|----------------|
| Player Kill | 100 | Yes (0.25x - 2.0x) | Yes |
| Sigil Capture | 500 | No | Yes |
| Sigil Hold (per minute) | 50 | No | Yes |
| Daily Bounty Task | 50 | No | Yes |
| Daily Faction Quest | 150 | No | Yes |

### 4.3 Faction Standing Calculation

```csharp
public static class FactionStandingManager
{
    // Called after any point change to recalculate standings
    public static void RecalculateStandings()
    {
        var factions = FactionManager.GetAllFactions()
            .OrderByDescending(f => f.SeasonPoints)
            .ToList();
            
        for (int i = 0; i < factions.Count; i++)
        {
            factions[i].CurrentStanding = i + 1; // 1st, 2nd, 3rd
        }
    }
    
    public static int GetCurrentStanding(Faction faction)
    {
        return faction.CurrentStanding;
    }
    
    // Underdog multiplier for point gains
    public static double GetUnderdogMultiplier(Faction faction)
    {
        return faction.CurrentStanding switch
        {
            1 => 1.00,
            2 => 1.02,
            3 => 1.05,
            _ => 1.00
        };
    }
}
```

---

## 5. Daily Faction Content

### 5.1 Daily Bounties

Three tasks generated daily, same for all players:

```csharp
public static class DailyBountyManager
{
    private static List<DailyBounty> _todaysBounties = new();
    private static DateTime _lastGeneration = DateTime.MinValue;
    
    public static void GenerateDailyBounties()
    {
        _todaysBounties.Clear();
        
        // Bounty 1: Kill X monsters of type Y
        var monster = SelectRandomMonster(difficulty: Random(1, 5));
        int killCount = monster.Difficulty * 5; // Higher difficulty = more kills
        _todaysBounties.Add(new KillBounty(monster.Type, killCount, points: 50));
        
        // Bounty 2: Gather X resources
        var resource = SelectRandomResource();
        int gatherCount = Random(50, 200);
        _todaysBounties.Add(new GatherBounty(resource, gatherCount, points: 50));
        
        // Bounty 3: Another kill bounty (different monster)
        var monster2 = SelectRandomMonster(difficulty: Random(1, 5), exclude: monster);
        int killCount2 = monster2.Difficulty * 5;
        _todaysBounties.Add(new KillBounty(monster2.Type, killCount2, points: 50));
        
        _lastGeneration = DateTime.UtcNow.Date;
        
        // Town Cryer announces new bounties
        TownCryerManager.AnnounceEvent(TownCryerCategory.RotatingContent,
            "New faction bounties are available! Speak to the Faction Registrar for details.");
    }
    
    private static MonsterTemplate SelectRandomMonster(int difficulty, MonsterTemplate exclude = null)
    {
        var candidates = MonsterRegistry.GetByDifficultyRange(difficulty - 1, difficulty + 1)
            .Where(m => m != exclude)
            .ToList();
            
        return candidates[Utility.Random(candidates.Count)];
    }
    
    // Check daily at midnight
    public static void OnServerTick()
    {
        if (DateTime.UtcNow.Date > _lastGeneration)
        {
            GenerateDailyBounties();
        }
    }
}
```

### 5.2 Daily Faction Quest (Swamp/Desert Mini-Boss)

```csharp
public static class DailyFactionQuestManager
{
    // The 4 possible mini-boss locations
    private static readonly Point3D[] BossLocations = new[]
    {
        new Point3D(/*Swamp Location 1*/),
        new Point3D(/*Swamp Location 2*/),
        new Point3D(/*Desert Location 1*/),
        new Point3D(/*Desert Location 2*/),
    };
    
    private static int _todaysLocationIndex;
    private static bool _bossKilledToday;
    private static HashSet<Serial> _playersCompletedToday = new();
    
    public static void GenerateDailyQuest()
    {
        _todaysLocationIndex = Utility.Random(BossLocations.Length);
        _bossKilledToday = false;
        _playersCompletedToday.Clear();
        
        // Spawn the mini-boss
        SpawnDailyBoss();
        
        // Announce (but don't reveal location - players must search)
        TownCryerManager.AnnounceEvent(TownCryerCategory.FactionEvent,
            "A powerful creature has been sighted in the wilds. Faction warriors, seek it out!");
    }
    
    public static void OnBossKilled(BaseCreature boss, PlayerMobile killer)
    {
        if (!IsDailyBoss(boss))
            return;
            
        // All faction members who participated get credit
        var participants = boss.DamageEntries
            .Select(de => de.Damager)
            .OfType<PlayerMobile>()
            .Where(pm => pm.Guild?.Faction != null)
            .Where(pm => !_playersCompletedToday.Contains(pm.Serial))
            .ToList();
            
        foreach (var player in participants)
        {
            FactionPointManager.AwardPoints(player, 150, "Daily Faction Quest");
            _playersCompletedToday.Add(player.Serial);
            player.SendMessage(0x35, "You have completed today's Faction Quest!");
        }
        
        _bossKilledToday = true;
        
        // Respawn for others (can be killed multiple times, but each player only gets credit once)
        Timer.StartTimer(TimeSpan.FromMinutes(15), SpawnDailyBoss);
    }
}
```

### 5.3 Rare Faction Deco Drops

```csharp
public static class FactionDecoDrops
{
    // 0.01% chance on any monster kill
    private const double DropChance = 0.0001;
    
    private static readonly Type[] FactionClothingTypes = new[]
    {
        typeof(FactionCloak),
        typeof(FactionSash),
        typeof(FactionBandana),
        typeof(FactionRobe),
        typeof(FactionSurcoat),
    };
    
    public static void OnMonsterKilled(BaseCreature monster, PlayerMobile killer)
    {
        // Only faction members can receive drops
        if (killer.Guild?.Faction == null)
            return;
            
        if (Utility.RandomDouble() < DropChance)
        {
            var itemType = FactionClothingTypes[Utility.Random(FactionClothingTypes.Length)];
            var item = (BaseFactionClothing)Activator.CreateInstance(itemType);
            
            // Item is universal but only wearable by faction members
            item.FactionRestricted = true;
            item.Hue = GenerateRareHue();
            
            killer.AddToBackpack(item);
            killer.SendMessage(0x35, "You have found a rare faction artifact!");
            
            // Global announcement for epic drops
            World.Broadcast(0x35, true, 
                $"{killer.Name} has discovered a rare {item.Name}!");
        }
    }
}

public abstract class BaseFactionClothing : BaseClothing
{
    public bool FactionRestricted { get; set; } = true;
    
    public override bool CanEquip(Mobile from)
    {
        if (!base.CanEquip(from))
            return false;
            
        if (FactionRestricted && from is PlayerMobile pm)
        {
            if (pm.Guild?.Faction == null)
            {
                pm.SendMessage(0x22, "You must be in a faction to wear this item.");
                return false;
            }
        }
        
        return true;
    }
}
```

---

## 6. Quarterly Season System

### 6.1 Season Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      QUARTERLY SEASON LIFECYCLE                         │
└─────────────────────────────────────────────────────────────────────────┘

  WEEK 1-11              WEEK 12                 WEEK 13 (New Season)
  ────────────────────►────────────────────────►─────────────────────────►
  
  │                     │                        │
  │ Normal competition  │ Season finale events   │ Points reset
  │ Points accumulate   │ Double points weekend  │ Rewards distributed
  │ Standings update    │ Final battle           │ Glicko soft reset
  │                     │ Leaderboard freeze     │ New season begins
  │                     │                        │
```

### 6.2 Season Reset Logic

```csharp
public static class SeasonManager
{
    public static void ProcessSeasonReset()
    {
        // 1. Determine final standings
        var standings = FactionStandingManager.GetFinalStandings();
        
        // 2. Distribute rewards
        DistributeSeasonRewards(standings);
        
        // 3. Reset faction points
        foreach (var faction in FactionManager.GetAllFactions())
        {
            faction.SeasonPoints = 0;
        }
        
        // 4. Reset player faction points (but preserve Glicko)
        foreach (var player in World.Mobiles.OfType<PlayerMobile>())
        {
            player.FactionPoints = 0;
        }
        
        // 5. Soft reset Glicko ratings
        GlickoManager.ProcessSeasonReset();
        
        // 6. Log and announce
        World.Broadcast(0x35, true, 
            $"A new season begins! {standings[0].Name} claimed victory last season!");
            
        // 7. Create historical record
        SeasonHistory.RecordSeason(standings);
    }
    
    private static void DistributeSeasonRewards(List<Faction> standings)
    {
        // 1st Place Faction
        foreach (var guild in standings[0].Guilds)
        {
            foreach (var member in guild.Members)
            {
                member.AddToBackpack(new SeasonChampionToken());
                member.SendMessage(0x35, "Your faction has won the season! Claim your champion rewards!");
            }
        }
        
        // Top 10 individual players across all factions
        var topPlayers = World.Mobiles.OfType<PlayerMobile>()
            .Where(pm => pm.Guild?.Faction != null)
            .OrderByDescending(pm => pm.FactionPoints)
            .Take(10)
            .ToList();
            
        for (int i = 0; i < topPlayers.Count; i++)
        {
            var player = topPlayers[i];
            player.AddToBackpack(new TopPlayerReward(rank: i + 1));
            player.SendMessage(0x35, $"You placed #{i + 1} this season! Check your backpack for rewards!");
        }
    }
}
```

### 6.3 Glicko Soft Reset

```csharp
public static class GlickoManager
{
    public static void ProcessSeasonReset()
    {
        foreach (var player in World.Mobiles.OfType<PlayerMobile>())
        {
            // Compress rating toward 1500 by 25%
            // This preserves relative skill while allowing movement
            double currentRating = player.GlickoRating;
            double newRating = 1500 + (currentRating - 1500) * 0.75;
            
            player.GlickoRating = newRating;
            
            // Increase rating deviation (uncertainty) by 50
            player.GlickoRD = Math.Min(300, player.GlickoRD + 50);
        }
    }
}
```

---

## 7. Admin Commands

```csharp
public class VvVAdminCommands
{
    [Usage("[vvv start")]
    [Description("Immediately starts a sigil battle")]
    public static void StartBattle(CommandEventArgs e)
    {
        VvVBattleManager.ForceStartBattle();
        e.Mobile.SendMessage("Sigil battle started.");
    }
    
    [Usage("[vvv stop")]
    [Description("Immediately ends the current sigil battle")]
    public static void StopBattle(CommandEventArgs e)
    {
        VvVBattleManager.ForceEndBattle();
        e.Mobile.SendMessage("Sigil battle ended.");
    }
    
    [Usage("[vvv config <setting> <value>")]
    [Description("Adjust VvV configuration at runtime")]
    public static void Config(CommandEventArgs e)
    {
        // TimeBetweenBattles, BattleDuration, etc.
    }
    
    [Usage("[vvv season reset")]
    [Description("Force a season reset (use with caution)")]
    public static void SeasonReset(CommandEventArgs e)
    {
        SeasonManager.ProcessSeasonReset();
        e.Mobile.SendMessage("Season reset complete.");
    }
    
    [Usage("[vvv points <player> <amount>")]
    [Description("Award faction points to a player")]
    public static void AwardPoints(CommandEventArgs e)
    {
        // Admin point adjustment
    }
    
    [Usage("[vvv standing")]
    [Description("Display current faction standings")]
    public static void ShowStandings(CommandEventArgs e)
    {
        var standings = FactionStandingManager.GetCurrentStandings();
        foreach (var faction in standings)
        {
            e.Mobile.SendMessage($"{faction.CurrentStanding}. {faction.Name}: {faction.SeasonPoints} points");
        }
    }
}
```

---

## 8. Integration with Existing Systems

### 8.1 Talisman System

- Talismans are **disabled during VvV battles** (PvP context triggers disable)
- 5-minute disable timer starts when player damages/is damaged by enemy faction
- Chivalry access lost during disable period

### 8.2 Glicko Rating System

- VvV kills feed into Glicko ratings
- Ratings used for point weighting (prevent farming)
- Soft reset at season end

### 8.3 Town Cryer

- Announces battle scheduling (5 minutes before)
- Announces sigil captures
- Announces town control changes
- Announces new daily bounties

### 8.4 Economy

- Town control grants temporary vendor discounts (10-15%)
- No permanent economic advantages
- All gold sinks remain functional during battles

---

## 9. Performance Considerations

### 9.1 Battle Processing Budget

```csharp
// Target: 5ms per battle tick with 100 participants
public class VvVBattleProcessor
{
    private const int TickBudgetMs = 5;
    
    public void ProcessTick(VvVBattle battle)
    {
        var sw = Stopwatch.StartNew();
        
        // Priority 1: Sigil state (always process)
        ProcessSigilState(battle);
        
        // Priority 2: Point calculations (defer if over budget)
        if (sw.ElapsedMilliseconds < TickBudgetMs * 0.5)
        {
            ProcessPendingPoints(battle);
        }
        
        // Priority 3: UI updates (skip if over budget)
        if (sw.ElapsedMilliseconds < TickBudgetMs * 0.8)
        {
            BroadcastUpdates(battle);
        }
        
        // Log if over budget
        if (sw.ElapsedMilliseconds > TickBudgetMs)
        {
            PerformanceLogger.Warn($"VvV tick exceeded budget: {sw.ElapsedMilliseconds}ms");
        }
    }
}
```

### 9.2 Point Batching

```csharp
// Batch point updates to reduce database writes
public class PointBatcher
{
    private ConcurrentQueue<PointUpdate> _pending = new();
    private Timer _flushTimer;
    
    public void QueuePointUpdate(PlayerMobile player, int points, string reason)
    {
        _pending.Enqueue(new PointUpdate(player.Serial, points, reason, DateTime.UtcNow));
    }
    
    // Flush every 5 seconds
    private void FlushUpdates()
    {
        var batch = new List<PointUpdate>();
        while (_pending.TryDequeue(out var update))
        {
            batch.Add(update);
        }
        
        if (batch.Count > 0)
        {
            DatabaseManager.BulkInsertPoints(batch);
        }
    }
}
```

---

## 10. Testing Procedures

### 10.1 Unit Tests

```csharp
[TestClass]
public class VvVTests
{
    [TestMethod]
    public void GlickoMultiplier_HigherRatedKiller_ReducedPoints()
    {
        var killer = CreatePlayer(rating: 1800);
        var victim = CreatePlayer(rating: 1200);
        
        double multiplier = FactionPointManager.CalculateGlickoMultiplier(killer, victim);
        
        Assert.AreEqual(0.25, multiplier); // 600 point difference = minimum multiplier
    }
    
    [TestMethod]
    public void UnderdogBonus_ThirdPlaceFaction_FivePercentBonus()
    {
        var faction = CreateFaction(standing: 3);
        
        double bonus = FactionStandingManager.GetUnderdogMultiplier(faction);
        
        Assert.AreEqual(1.05, bonus);
    }
    
    [TestMethod]
    public void SigilCapture_EnemyFaction_AwardsPoints()
    {
        var player = CreateFactionPlayer();
        var sigil = CreateEnemySigil();
        
        sigil.CompleteCaptureAttempt(player, player.Guild.Faction);
        
        Assert.AreEqual(500, player.FactionPoints);
    }
}
```

### 10.2 Integration Tests

1. **Full Battle Cycle**: Start → Capture → Control → Expiry
2. **Multi-Faction Combat**: 3-way battle scoring
3. **Season Reset**: Points, ratings, rewards distribution
4. **Daily Content**: Bounty generation, quest completion

---

## 11. File Modification List

| File | Change Type | Risk Level |
|------|-------------|------------|
| `UOContent/Engines/VvV/VvVBattle.cs` | Modify | Medium |
| `UOContent/Engines/VvV/VvVSigil.cs` | Modify | Medium |
| `Systems/Factions/FactionManager.cs` | New | Low |
| `Systems/Factions/FactionPointManager.cs` | New | Low |
| `Systems/Factions/FactionStandingManager.cs` | New | Low |
| `Systems/Factions/DailyBountyManager.cs` | New | Low |
| `Systems/Factions/SeasonManager.cs` | New | Low |
| `Mobiles/PlayerMobile.cs` | Modify | High |
| `Guilds/Guild.cs` | Modify | Medium |
| `Items/FactionClothing/` | New | Low |

---

## 12. Change Log

### v1.0.0 - 2025-01-02 (Initial Specification)
- Created comprehensive VvV integration specification
- Defined 3-faction guild-based system
- Implemented Glicko-2 weighted point scoring
- Added underdog bonus mechanics (2nd: 2%, 3rd: 5%)
- Specified daily bounties and faction quests
- Designed quarterly season system with soft Glicko reset
- Created rare faction clothing drop system
- Defined sigil capture and temporary town control mechanics