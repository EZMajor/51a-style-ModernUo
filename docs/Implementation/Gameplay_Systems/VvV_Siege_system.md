# VvV Siege System Specification
## 51alpha 3-Faction City Siege Warfare

### Document Metadata
- **Version**: v1.0.0
- **Last Updated**: 2025-01-02
- **Authors**: 51alpha Development Team
- **Applicable ModernUO Version**: v24.0.0+
- **Dependencies**: VvV_Integration.md, 51alpha_Master_Architecture.md
- **PR Ready**: No (Conceptual Design)

---

## 1. Executive Summary

The VvV Siege System provides 3-faction city warfare in designated siege cities. Sieges are triggered on-demand when sufficient faction players are online, lasting 30 minutes with objectives including sigil capture, altar control, and faction kills. Victorious factions gain persistent town control with NPC discounts until another faction captures the city.

### Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Faction Count | 3 (Vampire, Daemon, Goblin) | Creates dynamic 3-way battles |
| Battle Model | 3-way free-for-all | All factions fight simultaneously |
| Town Control | Persistent until captured | Strategic territory meta |
| Siege Trigger | On-demand (player count) | Ensures competitive battles |
| Siege Duration | 30 minutes | Balanced commitment time |
| Defense System | Silver-based budget | Rewards PvP participation |
| Spectators | Not allowed | Focused combat experience |

---

## 2. Siege Cities

### 2.1 Designated Cities

Only these 4 cities can be sieged (chosen for low traffic/strategic value):

| City | Map | Coordinates | Notes |
|------|-----|-------------|-------|
| **Jhelom** | Felucca | TBD | Island city, natural chokepoints |
| **Skara Brae** | Felucca | TBD | Coastal, multiple entry points |
| **Yew** | Felucca | TBD | Forest terrain, ambush potential |
| **Trinsic** | Felucca | TBD | Walled city, defensible |

### 2.2 Siege Region Definition

Each city has a defined siege region:

```csharp
public class SiegeCity
{
    public string Name { get; init; }
    public Map Map { get; init; }
    public Rectangle2D SiegeRegion { get; init; }
    public Point3D SigilSpawn { get; init; }
    public Point3D[] AltarLocations { get; init; }
    public Point3D[] PriestLocations { get; init; }
    public FactionId? ControllingFaction { get; set; }
    public DateTime? ControlledSince { get; set; }
}

public static class SiegeCities
{
    public static readonly SiegeCity Jhelom = new()
    {
        Name = "Jhelom",
        Map = Map.Felucca,
        SiegeRegion = new Rectangle2D(/* TBD */),
        SigilSpawn = new Point3D(/* TBD */),
        AltarLocations = new[] { /* TBD */ },
        PriestLocations = new[] { /* TBD */ }
    };
    
    // Similar for Skara Brae, Yew, Trinsic
}
```

---

## 3. Town Control System

### 3.1 Persistent Control

Town control persists **indefinitely** until another faction wins a siege in that city.

```csharp
public class TownControlState
{
    public SiegeCity City { get; set; }
    public FactionId ControllingFaction { get; set; }
    public DateTime ControlledSince { get; set; }
    public int SiegesDefended { get; set; } // Consecutive defenses
    
    // Persistence
    public void Serialize(GenericWriter writer)
    {
        writer.Write(City.Name);
        writer.Write((int)ControllingFaction);
        writer.Write(ControlledSince);
        writer.Write(SiegesDefended);
    }
}
```

### 3.2 Control Benefits

| Cities Controlled | NPC Discount | Additional Benefits |
|-------------------|--------------|---------------------|
| 1-3 cities | 10% | Faction banners displayed |
| All 4 cities | 15% | "Domination" title available |

### 3.3 Visual Indicators

When a faction controls a city:

```csharp
public static class TownControlVisuals
{
    public static void ApplyFactionControl(SiegeCity city, FactionId faction)
    {
        // Banner color based on faction
        int bannerHue = faction switch
        {
            FactionId.Vampire => 0x21,  // Red
            FactionId.Daemon => 0x30,   // Orange
            FactionId.Goblin => 0x3F,   // Green
            _ => 0
        };
        
        // Spawn/update faction banners at key locations
        foreach (var bannerLoc in city.BannerLocations)
        {
            var banner = new FactionBanner(faction, bannerHue);
            banner.MoveToWorld(bannerLoc, city.Map);
        }
        
        // Update guard NPCs to faction colors (optional)
        // Update town crier announcements
    }
}
```

### 3.4 NPC Discount Implementation

```csharp
public static class FactionVendorDiscount
{
    public static double GetDiscount(PlayerMobile buyer, BaseVendor vendor)
    {
        if (buyer.Guild?.Faction == null)
            return 0.0;
            
        var city = GetCityForVendor(vendor);
        if (city == null)
            return 0.0;
            
        var controlState = TownControlManager.GetControlState(city);
        if (controlState?.ControllingFaction != buyer.Guild.Faction)
            return 0.0;
            
        // Check for domination bonus
        int citiesControlled = TownControlManager.GetCitiesControlledBy(buyer.Guild.Faction);
        
        return citiesControlled >= 4 ? 0.15 : 0.10; // 15% if all 4, else 10%
    }
}
```

---

## 4. Siege Trigger System

### 4.1 On-Demand Activation

Sieges trigger automatically when sufficient faction players are online and in PvP-ready state.

```csharp
public static class SiegeTriggerManager
{
    // Minimum players per faction to trigger siege
    public const int MinPlayersPerFaction = 3;
    
    // Minimum total players across all factions
    public const int MinTotalPlayers = 10;
    
    // Cooldown between sieges (same city)
    public static readonly TimeSpan CityCooldown = TimeSpan.FromHours(2);
    
    // Cooldown between any sieges (global)
    public static readonly TimeSpan GlobalCooldown = TimeSpan.FromMinutes(30);
    
    private static DateTime _lastSiegeEnd = DateTime.MinValue;
    private static Dictionary<string, DateTime> _cityCooldowns = new();
    
    public static void CheckSiegeTrigger()
    {
        // Global cooldown check
        if (DateTime.UtcNow < _lastSiegeEnd + GlobalCooldown)
            return;
            
        // Count eligible players per faction
        var factionCounts = new Dictionary<FactionId, int>
        {
            { FactionId.Vampire, 0 },
            { FactionId.Daemon, 0 },
            { FactionId.Goblin, 0 }
        };
        
        foreach (var pm in World.Mobiles.Values.OfType<PlayerMobile>())
        {
            if (!IsEligibleForSiege(pm))
                continue;
                
            if (pm.Guild?.Faction is FactionId faction)
                factionCounts[faction]++;
        }
        
        // Check minimum requirements
        int factionsWithMinPlayers = factionCounts.Count(kvp => kvp.Value >= MinPlayersPerFaction);
        int totalPlayers = factionCounts.Values.Sum();
        
        if (factionsWithMinPlayers < 2 || totalPlayers < MinTotalPlayers)
            return;
            
        // Select city (round-robin or least recently sieged)
        var city = SelectSiegeCity();
        if (city == null)
            return;
            
        // Trigger siege
        StartSiege(city);
    }
    
    private static bool IsEligibleForSiege(PlayerMobile pm)
    {
        if (!pm.Alive || pm.NetState == null)
            return false;
            
        if (pm.Guild?.Faction == null)
            return false;
            
        // Must be Combatant status (not Peaceful)
        if (pm.FactionCombatStatus == GuildMemberCombatStatus.Peaceful)
            return false;
            
        // Must be in Felucca
        if (pm.Map != Map.Felucca)
            return false;
            
        return true;
    }
    
    private static SiegeCity SelectSiegeCity()
    {
        // Get cities off cooldown, sorted by least recently sieged
        return SiegeCities.All
            .Where(c => !_cityCooldowns.ContainsKey(c.Name) || 
                        DateTime.UtcNow > _cityCooldowns[c.Name] + CityCooldown)
            .OrderBy(c => c.LastSiegeTime ?? DateTime.MinValue)
            .FirstOrDefault();
    }
}
```

### 4.2 Siege Announcement

```csharp
public static void AnnounceSiegeImminent(SiegeCity city)
{
    // 5-minute warning
    World.Broadcast(0x35, true,
        $"═══════════════════════════════════════════");
    World.Broadcast(0x35, true,
        $"  SIEGE WARNING: {city.Name} will be contested in 5 minutes!");
    World.Broadcast(0x35, true,
        $"  All faction combatants prepare for battle!");
    World.Broadcast(0x35, true,
        $"═══════════════════════════════════════════");
    
    // Town Cryer announcement
    TownCryerManager.AnnounceEvent(TownCryerCategory.FactionEvent,
        $"The city of {city.Name} will be under siege in 5 minutes! Faction warriors, to arms!");
}
```

---

## 5. Siege Battle System

### 5.1 Battle Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SIEGE BATTLE LIFECYCLE                                │
└─────────────────────────────────────────────────────────────────────────┘

  WARNING         SETUP           BATTLE              RESOLUTION
  ─────────────►──────────────►──────────────────►──────────────────
  
  │              │               │                    │
  │ 5 min warn   │ Spawn sigil   │ 30 min combat      │ Declare winner
  │ Announcement │ Spawn altars  │ Score tracking     │ Update control
  │              │ Spawn priests │ Objectives active  │ Award rewards
  │              │ Enable traps  │                    │ Cleanup
  │              │               │                    │
  │ T-5:00       │ T-0:00        │ T+0:00 to T+30:00  │ T+30:00
```

### 5.2 Battle State

```csharp
public class SiegeBattle
{
    public Guid BattleId { get; } = Guid.NewGuid();
    public SiegeCity City { get; set; }
    public DateTime StartTime { get; set; }
    public DateTime EndTime => StartTime + Duration;
    public TimeSpan Duration { get; } = TimeSpan.FromMinutes(30);
    public SiegeBattleState State { get; set; }
    
    // Scoring
    public Dictionary<FactionId, int> Scores { get; } = new()
    {
        { FactionId.Vampire, 0 },
        { FactionId.Daemon, 0 },
        { FactionId.Goblin, 0 }
    };
    
    // Active participants (for reward distribution)
    public Dictionary<FactionId, HashSet<Serial>> Participants { get; } = new();
    
    // Objectives
    public SiegeSigil Sigil { get; set; }
    public List<SiegeAltar> Altars { get; } = new();
    public List<SiegePriest> Priests { get; } = new();
    
    // Defenses
    public Dictionary<FactionId, List<SiegeTrap>> Traps { get; } = new();
    public Dictionary<FactionId, List<SiegeTurret>> Turrets { get; } = new();
    public Dictionary<FactionId, int> DefenseBudgetSpent { get; } = new();
    
    // Telemetry
    public long CorrelationId { get; set; }
}

public enum SiegeBattleState
{
    Warning,      // 5 min countdown
    Setup,        // Spawning objectives
    Active,       // Combat in progress
    Concluding,   // Final 60 seconds
    Complete      // Winner determined
}
```

### 5.3 Victory Conditions

```csharp
public static class SiegeVictoryCalculator
{
    public const int VictoryScore = 10000;
    
    public static FactionId? CheckVictory(SiegeBattle battle)
    {
        // Check for score victory
        foreach (var (faction, score) in battle.Scores)
        {
            if (score >= VictoryScore)
                return faction;
        }
        
        // Check for time expiration
        if (DateTime.UtcNow >= battle.EndTime)
        {
            return DetermineWinnerByScore(battle);
        }
        
        return null; // Battle continues
    }
    
    private static FactionId? DetermineWinnerByScore(SiegeBattle battle)
    {
        var sorted = battle.Scores
            .OrderByDescending(kvp => kvp.Value)
            .ToList();
            
        // Clear winner
        if (sorted[0].Value > sorted[1].Value)
            return sorted[0].Key;
            
        // Tie-breaker: most unique participants
        var topScore = sorted[0].Value;
        var tiedFactions = sorted
            .Where(kvp => kvp.Value == topScore)
            .Select(kvp => kvp.Key)
            .ToList();
            
        return tiedFactions
            .OrderByDescending(f => battle.Participants[f].Count)
            .First();
    }
}
```

---

## 6. Scoring System

### 6.1 Point Values

| Action | Base Points | Silver Reward | Notes |
|--------|-------------|---------------|-------|
| **Player Kill** | 100 | 50 | Glicko-weighted |
| **Kill Assist** | 25 | 10 | Within 30s of kill |
| **Sigil Capture** | 500 | 200 | Deliver to priest |
| **Sigil Steal** | 100 | 50 | Pick up enemy sigil |
| **Altar Capture** | 250 | 100 | 60s uncontested hold |
| **Altar Tick** | 25 | 10 | Per 30s while holding |
| **Trap Trigger** | 10 | 5 | When enemy triggers your trap |
| **Turret Kill** | 50 | 25 | If turret gets killing blow |

### 6.2 Glicko-Weighted Kill Scoring

```csharp
public static class SiegeKillScoring
{
    public static (int points, int silver) CalculateKillReward(
        PlayerMobile killer, 
        PlayerMobile victim,
        SiegeBattle battle)
    {
        int basePoints = 100;
        int baseSilver = 50;
        
        // Apply Glicko multiplier
        double glickoMultiplier = GlickoManager.CalculatePointMultiplier(killer, victim);
        
        int points = (int)(basePoints * glickoMultiplier);
        int silver = (int)(baseSilver * glickoMultiplier);
        
        // Apply uncontested penalty if applicable
        double contestPenalty = CalculateContestPenalty(battle);
        points = (int)(points * (1.0 - contestPenalty));
        silver = (int)(silver * (1.0 - contestPenalty));
        
        return (points, silver);
    }
}
```

### 6.3 Uncontested Penalty

```csharp
public static class UncontestedPenaltyCalculator
{
    private const int CheckIntervalSeconds = 60;
    private const int PenaltyThresholdMinutes = 10;
    
    // Sliding scale based on opponent count
    public static double CalculateContestPenalty(SiegeBattle battle)
    {
        var factionCounts = CountActivePlayers(battle);
        
        // Find dominant faction and opponent count
        var sorted = factionCounts.OrderByDescending(kvp => kvp.Value).ToList();
        int opponentCount = sorted.Skip(1).Sum(kvp => kvp.Value);
        
        // Check if penalty condition met (low opponents for 10+ min)
        if (!HasLowOpponentsForDuration(battle, opponentCount))
            return 0.0;
        
        return opponentCount switch
        {
            0 => 0.80,      // 80% penalty
            1 or 2 => 0.60, // 60% penalty
            >= 3 and <= 5 => 0.15, // 15% penalty
            _ => 0.0        // No penalty
        };
    }
    
    private static Dictionary<FactionId, int> CountActivePlayers(SiegeBattle battle)
    {
        var counts = new Dictionary<FactionId, int>
        {
            { FactionId.Vampire, 0 },
            { FactionId.Daemon, 0 },
            { FactionId.Goblin, 0 }
        };
        
        foreach (var pm in battle.City.SiegeRegion.GetMobiles<PlayerMobile>())
        {
            if (!IsActiveParticipant(pm, battle))
                continue;
                
            if (pm.Guild?.Faction is FactionId faction)
                counts[faction]++;
        }
        
        return counts;
    }
    
    private static bool IsActiveParticipant(PlayerMobile pm, SiegeBattle battle)
    {
        // Must be alive, in region, and have dealt/received damage in last 5 min
        if (!pm.Alive)
            return false;
            
        if (!battle.City.SiegeRegion.Contains(pm.Location))
            return false;
            
        var lastActivity = SiegeTelemetry.GetLastCombatActivity(pm, battle);
        return lastActivity != null && 
               DateTime.UtcNow - lastActivity.Value < TimeSpan.FromMinutes(5);
    }
}
```

---

## 7. Objectives

### 7.1 Sigil System

The sigil is the primary objective - capture and return to your faction's priest.

```csharp
public class SiegeSigil : Item
{
    public SiegeBattle Battle { get; set; }
    public FactionId? HeldByFaction { get; set; }
    public PlayerMobile Carrier { get; set; }
    public DateTime? PickedUpTime { get; set; }
    
    // Respawn delay after capture
    public static readonly TimeSpan RespawnDelay = TimeSpan.FromMinutes(2);
    
    public SiegeSigil(SiegeBattle battle) : base(0x18A0) // Sigil graphic
    {
        Battle = battle;
        Name = "Siege Sigil";
        Hue = 0x501; // Gold
        Movable = false;
        Weight = 10.0;
    }
    
    public override void OnDoubleClick(Mobile from)
    {
        if (!(from is PlayerMobile pm))
            return;
            
        if (pm.Guild?.Faction == null)
        {
            pm.SendMessage(0x22, "You must be in a faction to carry the sigil.");
            return;
        }
        
        if (Carrier != null)
        {
            pm.SendMessage(0x22, "The sigil is already being carried.");
            return;
        }
        
        // Pick up sigil
        PickUpSigil(pm);
    }
    
    private void PickUpSigil(PlayerMobile pm)
    {
        Carrier = pm;
        HeldByFaction = pm.Guild.Faction;
        PickedUpTime = DateTime.UtcNow;
        
        // Move to carrier's backpack (visual only - special handling)
        MoveToWorld(pm.Location, pm.Map);
        
        // Announce
        SiegeBroadcast(Battle, $"{pm.Name} ({HeldByFaction}) has picked up the Sigil!");
        
        // Award steal points if taking from another faction's area
        var (points, silver) = (100, 50);
        Battle.Scores[HeldByFaction.Value] += points;
        AwardSilver(pm, silver, "Sigil Steal");
        
        // Log event
        SiegeTelemetry.LogEvent(Battle, SiegeEventType.SigilPickup, pm);
    }
    
    public void OnCarrierDeath()
    {
        if (Carrier == null)
            return;
            
        // Drop sigil at death location
        var dropLoc = Carrier.Location;
        Carrier = null;
        HeldByFaction = null;
        PickedUpTime = null;
        
        MoveToWorld(dropLoc, Battle.City.Map);
        
        SiegeBroadcast(Battle, "The Sigil has been dropped!");
    }
}
```

### 7.2 Priest NPCs

Priests are turn-in points for the sigil.

```csharp
public class SiegePriest : BaseCreature
{
    public SiegeBattle Battle { get; set; }
    public FactionId Faction { get; set; }
    
    public SiegePriest(SiegeBattle battle, FactionId faction) : base(AIType.AI_Vendor)
    {
        Battle = battle;
        Faction = faction;
        
        Name = $"{faction} Priest";
        Title = "the Siege Priest";
        
        // Faction-colored robe
        Hue = faction switch
        {
            FactionId.Vampire => 0x21,
            FactionId.Daemon => 0x30,
            FactionId.Goblin => 0x3F,
            _ => 0
        };
        
        // Invulnerable
        Blessed = true;
    }
    
    public override void OnDoubleClick(Mobile from)
    {
        if (!(from is PlayerMobile pm))
            return;
            
        // Check if player is carrying sigil
        var sigil = Battle.Sigil;
        if (sigil?.Carrier != pm)
        {
            Say("Bring me the Sigil to score for your faction!");
            return;
        }
        
        // Must be same faction
        if (pm.Guild?.Faction != Faction)
        {
            Say("You are not of my faction! Begone!");
            return;
        }
        
        // Capture sigil!
        CaptureSigil(pm, sigil);
    }
    
    private void CaptureSigil(PlayerMobile pm, SiegeSigil sigil)
    {
        // Award points
        var (points, silver) = (500, 200);
        Battle.Scores[Faction] += points;
        AwardSilver(pm, silver, "Sigil Capture");
        
        // Announce
        SiegeBroadcast(Battle, 
            $"═══ {pm.Name} has captured the Sigil for {Faction}! +{points} points! ═══");
        
        // Effects
        pm.FixedParticles(0x373A, 10, 15, 5018, EffectLayer.Head);
        pm.PlaySound(0x1F7);
        
        // Log
        SiegeTelemetry.LogEvent(Battle, SiegeEventType.SigilCapture, pm);
        
        // Respawn sigil after delay
        sigil.Delete();
        Timer.StartTimer(SiegeSigil.RespawnDelay, () =>
        {
            if (Battle.State == SiegeBattleState.Active)
            {
                var newSigil = new SiegeSigil(Battle);
                newSigil.MoveToWorld(Battle.City.SigilSpawn, Battle.City.Map);
                Battle.Sigil = newSigil;
                SiegeBroadcast(Battle, "The Sigil has respawned!");
            }
        });
    }
}
```

### 7.3 Altar System

Altars provide ongoing point generation for the controlling faction.

```csharp
public class SiegeAltar : Item
{
    public SiegeBattle Battle { get; set; }
    public int AltarNumber { get; set; }
    public FactionId? ControllingFaction { get; set; }
    public PlayerMobile Capturer { get; set; }
    public DateTime? CaptureStartTime { get; set; }
    
    // Capture timing
    public static readonly TimeSpan CaptureTime = TimeSpan.FromSeconds(60);
    public static readonly TimeSpan TickInterval = TimeSpan.FromSeconds(30);
    
    // Points
    public const int CapturePoints = 250;
    public const int CaptureSilver = 100;
    public const int TickPoints = 25;
    public const int TickSilver = 10;
    
    private Timer _tickTimer;
    
    public SiegeAltar(SiegeBattle battle, int number) : base(0x14F0) // Altar graphic
    {
        Battle = battle;
        AltarNumber = number;
        Name = $"Siege Altar {number}";
        Movable = false;
    }
    
    public override bool OnMoveOver(Mobile m)
    {
        if (!(m is PlayerMobile pm))
            return true;
            
        if (pm.Guild?.Faction == null)
            return true;
            
        var faction = pm.Guild.Faction.Value;
        
        // Already controlling
        if (ControllingFaction == faction)
            return true;
            
        // Start capture
        if (Capturer == null || Capturer.Guild?.Faction != faction)
        {
            StartCapture(pm);
        }
        
        return true;
    }
    
    private void StartCapture(PlayerMobile pm)
    {
        // Cancel existing capture
        if (Capturer != null)
        {
            Capturer.SendMessage(0x22, "Your altar capture has been interrupted!");
        }
        
        Capturer = pm;
        CaptureStartTime = DateTime.UtcNow;
        
        pm.SendMessage(0x35, $"Capturing Altar {AltarNumber}... Stay on the altar for {CaptureTime.TotalSeconds} seconds!");
        
        // Start capture check timer
        Timer.StartTimer(TimeSpan.FromSeconds(1), TimeSpan.FromSeconds(1), () =>
        {
            if (Battle.State != SiegeBattleState.Active)
                return;
                
            if (Capturer != pm || !pm.Alive || pm.Map != Map)
            {
                CancelCapture();
                return;
            }
            
            // Check if still on altar
            if (pm.Location.X < X - 1 || pm.Location.X > X + 1 ||
                pm.Location.Y < Y - 1 || pm.Location.Y > Y + 1)
            {
                CancelCapture();
                return;
            }
            
            // Check capture complete
            if (DateTime.UtcNow >= CaptureStartTime + CaptureTime)
            {
                CompleteCapture(pm);
            }
        });
    }
    
    private void CompleteCapture(PlayerMobile pm)
    {
        var faction = pm.Guild.Faction.Value;
        
        // Stop existing tick timer
        _tickTimer?.Stop();
        
        ControllingFaction = faction;
        Capturer = null;
        CaptureStartTime = null;
        
        // Update visual
        Hue = faction switch
        {
            FactionId.Vampire => 0x21,
            FactionId.Daemon => 0x30,
            FactionId.Goblin => 0x3F,
            _ => 0
        };
        
        // Award capture points
        Battle.Scores[faction] += CapturePoints;
        AwardSilver(pm, CaptureSilver, $"Altar {AltarNumber} Capture");
        
        SiegeBroadcast(Battle, $"{pm.Name} has captured Altar {AltarNumber} for {faction}!");
        
        // Start tick timer
        _tickTimer = Timer.StartTimer(TickInterval, TickInterval, () =>
        {
            if (Battle.State != SiegeBattleState.Active)
            {
                _tickTimer?.Stop();
                return;
            }
            
            if (ControllingFaction == faction)
            {
                Battle.Scores[faction] += TickPoints;
                // Silver distributed to faction members in region
                DistributeTickSilver(faction);
            }
        });
    }
    
    private void CancelCapture()
    {
        Capturer = null;
        CaptureStartTime = null;
    }
}
```

---

## 8. Peaceful Participant System

### 8.1 Combat Status

Guild members can choose their combat participation level.

```csharp
public enum GuildMemberCombatStatus
{
    Combatant,  // Full PvP - flagged against enemy factions everywhere
    Peaceful    // No PvP flags outside siege events - crafters/PvM
}

// In PlayerMobile.cs
public partial class PlayerMobile
{
    private GuildMemberCombatStatus _factionCombatStatus = GuildMemberCombatStatus.Combatant;
    
    [CommandProperty(AccessLevel.GameMaster)]
    public GuildMemberCombatStatus FactionCombatStatus
    {
        get => _factionCombatStatus;
        set
        {
            if (_factionCombatStatus != value)
            {
                _factionCombatStatus = value;
                _combatStatusChangedTime = DateTime.UtcNow;
                InvalidateProperties();
            }
        }
    }
    
    private DateTime _combatStatusChangedTime;
    
    // 24-hour cooldown on status changes
    public bool CanChangeCombatStatus => 
        DateTime.UtcNow >= _combatStatusChangedTime + TimeSpan.FromHours(24);
}
```

### 8.2 Peaceful Rules

```csharp
public static class PeacefulParticipantRules
{
    public static bool CanAttack(PlayerMobile attacker, PlayerMobile target)
    {
        // Both must be in factions
        if (attacker.Guild?.Faction == null || target.Guild?.Faction == null)
            return true; // Normal rules apply
            
        // Same faction - never attack
        if (attacker.Guild.Faction == target.Guild.Faction)
            return false;
            
        // Check if in active siege zone
        if (IsInActiveSiegeZone(attacker) && IsInActiveSiegeZone(target))
            return true; // Always can attack in siege
            
        // Outside siege - check peaceful status
        if (target.FactionCombatStatus == GuildMemberCombatStatus.Peaceful)
        {
            attacker.SendMessage(0x22, "That player is a peaceful faction member and cannot be attacked outside siege zones.");
            return false;
        }
        
        if (attacker.FactionCombatStatus == GuildMemberCombatStatus.Peaceful)
        {
            attacker.SendMessage(0x22, "You are a peaceful participant and cannot attack outside siege zones.");
            return false;
        }
        
        return true; // Both combatants, not in siege - allow faction warfare
    }
    
    public static void OnEnterSiegeZone(PlayerMobile pm, SiegeBattle battle)
    {
        if (pm.FactionCombatStatus == GuildMemberCombatStatus.Peaceful)
        {
            pm.SendMessage(0x35, "You have entered a siege zone. Your peaceful status is suspended for the duration.");
        }
        
        // Auto-flag for siege duration
        pm.SiegeParticipant = true;
    }
    
    public static void OnLeaveSiegeZone(PlayerMobile pm)
    {
        pm.SiegeParticipant = false;
        
        if (pm.FactionCombatStatus == GuildMemberCombatStatus.Peaceful)
        {
            pm.SendMessage(0x35, "You have left the siege zone. Your peaceful status is restored.");
        }
    }
}
```

### 8.3 Guild Stone Integration

```csharp
public class GuildCombatStatusGump : Gump
{
    private PlayerMobile _player;
    
    public GuildCombatStatusGump(PlayerMobile player) : base(50, 50)
    {
        _player = player;
        
        AddBackground(0, 0, 300, 200, 9200);
        AddLabel(80, 20, 0x35, "Faction Combat Status");
        
        AddLabel(20, 60, 0, $"Current Status: {player.FactionCombatStatus}");
        
        if (!player.CanChangeCombatStatus)
        {
            var remaining = player._combatStatusChangedTime + TimeSpan.FromHours(24) - DateTime.UtcNow;
            AddLabel(20, 80, 0x22, $"Cooldown: {remaining.Hours}h {remaining.Minutes}m remaining");
        }
        else
        {
            AddLabel(20, 100, 0, "Select Status:");
            
            // Combatant button
            AddButton(20, 130, 
                player.FactionCombatStatus == GuildMemberCombatStatus.Combatant ? 4006 : 4005,
                4007, 1, GumpButtonType.Reply, 0);
            AddLabel(55, 130, 0, "Combatant (Full PvP)");
            
            // Peaceful button
            AddButton(20, 155,
                player.FactionCombatStatus == GuildMemberCombatStatus.Peaceful ? 4006 : 4005,
                4007, 2, GumpButtonType.Reply, 0);
            AddLabel(55, 155, 0, "Peaceful (PvE/Crafting)");
        }
    }
    
    public override void OnResponse(NetState sender, RelayInfo info)
    {
        if (!_player.CanChangeCombatStatus)
        {
            _player.SendMessage(0x22, "You cannot change combat status yet. 24-hour cooldown active.");
            return;
        }
        
        switch (info.ButtonID)
        {
            case 1:
                _player.FactionCombatStatus = GuildMemberCombatStatus.Combatant;
                _player.SendMessage(0x35, "You are now a full Combatant. Enemy factions can attack you anywhere.");
                break;
                
            case 2:
                _player.FactionCombatStatus = GuildMemberCombatStatus.Peaceful;
                _player.SendMessage(0x35, "You are now a Peaceful participant. You can only be attacked in siege zones.");
                break;
        }
    }
}
```

---

## 9. Trap System

### 9.1 Trap Types (Sphere-Style Compatible)

```csharp
public abstract class SiegeTrap : Item
{
    public SiegeBattle Battle { get; set; }
    public FactionId OwnerFaction { get; set; }
    public Serial OwnerPlayer { get; set; }
    public int TriggerCount { get; set; }
    public int MaxTriggers { get; } = 5;
    public DateTime NextArmedTime { get; set; }
    public TimeSpan Cooldown { get; protected set; }
    
    public abstract void OnTrigger(PlayerMobile victim);
    
    public bool TryTrigger(PlayerMobile victim)
    {
        if (DateTime.UtcNow < NextArmedTime)
            return false;
            
        if (TriggerCount >= MaxTriggers)
            return false;
            
        // Don't trigger on friendly
        if (victim.Guild?.Faction == OwnerFaction)
            return false;
            
        // Detection skill check
        if (victim.Skills.DetectHidden.Value >= Utility.Random(100))
        {
            victim.SendMessage(0x35, "You detect and avoid a trap!");
            return false;
        }
        
        TriggerCount++;
        NextArmedTime = DateTime.UtcNow + Cooldown;
        
        OnTrigger(victim);
        
        // Award points to trap owner faction
        Battle.Scores[OwnerFaction] += 10;
        
        // Log
        SiegeTelemetry.LogEvent(Battle, SiegeEventType.TrapTriggered, victim, 
            $"Trap: {GetType().Name}, Owner: {OwnerFaction}");
        
        return true;
    }
}

// Alarm Trap - Reveals location
public class AlarmTrap : SiegeTrap
{
    public AlarmTrap() : base()
    {
        Name = "Alarm Trap";
        ItemID = 0x1BC4;
        Visible = false;
        Cooldown = TimeSpan.FromSeconds(30);
    }
    
    public override void OnTrigger(PlayerMobile victim)
    {
        // Sound alert
        Effects.PlaySound(Location, Map, 0x1F4);
        
        // Visual reveal
        victim.FixedParticles(0x376A, 9, 32, 5005, EffectLayer.Waist);
        
        // Broadcast to faction
        foreach (var pm in Battle.GetFactionPlayers(OwnerFaction))
        {
            pm.SendMessage(0x35, $"ALERT: Enemy detected at {Location}!");
        }
        
        victim.SendMessage(0x22, "You triggered an alarm trap! Your position is revealed!");
    }
}

// Snare Trap - 50% slow for 3 seconds
public class SnareTrap : SiegeTrap
{
    public SnareTrap() : base()
    {
        Name = "Snare Trap";
        ItemID = 0x1BC5;
        Visible = false;
        Cooldown = TimeSpan.FromSeconds(45);
    }
    
    public override void OnTrigger(PlayerMobile victim)
    {
        Effects.PlaySound(Location, Map, 0x1FF);
        victim.FixedParticles(0x37B9, 10, 30, 5052, EffectLayer.LeftFoot);
        
        // Apply 50% slow (does NOT prevent movement or casting)
        victim.SendMessage(0x22, "You are snared! Movement slowed for 3 seconds.");
        
        var originalDex = victim.Dex;
        victim.Dex = victim.Dex / 2;
        
        Timer.StartTimer(TimeSpan.FromSeconds(3), () =>
        {
            victim.Dex = originalDex;
            victim.SendMessage(0x35, "The snare has worn off.");
        });
    }
}

// Smoke Trap - Breaks LoS for 5 seconds
public class SmokeTrap : SiegeTrap
{
    public SmokeTrap() : base()
    {
        Name = "Smoke Trap";
        ItemID = 0x1BC6;
        Visible = false;
        Cooldown = TimeSpan.FromSeconds(60);
    }
    
    public override void OnTrigger(PlayerMobile victim)
    {
        Effects.PlaySound(Location, Map, 0x22F);
        
        // Create smoke cloud (blocks LoS)
        var smokeCloud = new SmokeCloudEffect(Location, Map, TimeSpan.FromSeconds(5));
        
        victim.SendMessage(0x22, "A smoke cloud erupts around you!");
        
        // Note: Smoke breaks LoS, causing spell fizzles for anyone casting through it
    }
}

// Mana Drain Trap - -20 mana
public class ManaDrainTrap : SiegeTrap
{
    public ManaDrainTrap() : base()
    {
        Name = "Mana Drain Trap";
        ItemID = 0x1BC7;
        Visible = false;
        Cooldown = TimeSpan.FromSeconds(30);
    }
    
    public override void OnTrigger(PlayerMobile victim)
    {
        Effects.PlaySound(Location, Map, 0x1F8);
        victim.FixedParticles(0x374A, 10, 15, 5013, EffectLayer.Waist);
        
        int manaDrain = 20;
        victim.Mana = Math.Max(0, victim.Mana - manaDrain);
        
        victim.SendMessage(0x22, $"A trap drains {manaDrain} of your mana!");
    }
}
```

### 9.2 Turret System

```csharp
public abstract class SiegeTurret : BaseCreature
{
    public SiegeBattle Battle { get; set; }
    public FactionId OwnerFaction { get; set; }
    public Serial OwnerPlayer { get; set; }
    public int MaxHits { get; protected set; }
    
    protected SiegeTurret() : base(AIType.AI_Archer, FightMode.Closest, 12, 1, 0.2, 0.4)
    {
        Blessed = false; // Can be destroyed
    }
    
    public override bool IsEnemy(Mobile m)
    {
        if (m is PlayerMobile pm && pm.Guild?.Faction != null)
        {
            return pm.Guild.Faction != OwnerFaction;
        }
        return false;
    }
    
    public override void OnDeath(Container c)
    {
        SiegeTelemetry.LogEvent(Battle, SiegeEventType.TurretDestroyed, null,
            $"Turret: {GetType().Name}, Owner: {OwnerFaction}");
            
        base.OnDeath(c);
    }
}

// Arrow Turret - 10-15 damage, 1 shot/3s
public class ArrowTurret : SiegeTurret
{
    public ArrowTurret() : base()
    {
        Name = "Arrow Turret";
        Body = 0x2F4; // Turret graphic
        MaxHits = 50;
        
        SetHits(50);
        SetDamage(10, 15);
        
        // 3 second attack delay
        ActiveSpeed = 3.0;
        PassiveSpeed = 3.0;
    }
    
    public override void OnGaveMeleeAttack(Mobile defender)
    {
        // Arrow visual
        MovingEffect(defender, 0xF42, 18, 1, false, false);
        Effects.PlaySound(Location, Map, 0x145);
    }
}

// Magic Turret - 15-20 damage, 1 shot/4s
public class MagicTurret : SiegeTurret
{
    public MagicTurret() : base()
    {
        Name = "Magic Turret";
        Body = 0x2F5;
        MaxHits = 75;
        
        SetHits(75);
        SetDamage(15, 20);
        
        ActiveSpeed = 4.0;
        PassiveSpeed = 4.0;
    }
    
    public override void OnGaveMeleeAttack(Mobile defender)
    {
        // Magic bolt visual
        MovingEffect(defender, 0x36E4, 18, 1, false, false);
        Effects.PlaySound(Location, Map, 0x1E3);
    }
}
```

### 9.3 Defense Budget System

```csharp
public static class DefenseBudgetManager
{
    public const int MaxBudgetPerFaction = 5000; // Silver
    
    public static readonly Dictionary<Type, int> TrapCosts = new()
    {
        { typeof(AlarmTrap), 100 },
        { typeof(SnareTrap), 200 },
        { typeof(SmokeTrap), 300 },
        { typeof(ManaDrainTrap), 250 }
    };
    
    public static readonly Dictionary<Type, int> TurretCosts = new()
    {
        { typeof(ArrowTurret), 500 },
        { typeof(MagicTurret), 750 }
    };
    
    public static readonly Dictionary<Type, int> PlacementLimits = new()
    {
        { typeof(AlarmTrap), 8 },
        { typeof(SnareTrap), 6 },
        { typeof(SmokeTrap), 4 },
        { typeof(ManaDrainTrap), 4 },
        { typeof(ArrowTurret), 4 },
        { typeof(MagicTurret), 2 }
    };
    
    public static PlacementResult TryPlaceDefense<T>(
        PlayerMobile player, 
        SiegeBattle battle, 
        Point3D location) where T : Item, new()
    {
        var faction = player.Guild?.Faction;
        if (faction == null)
            return PlacementResult.NotInFaction;
            
        // Check budget
        int cost = GetCost<T>();
        int spent = battle.DefenseBudgetSpent.GetValueOrDefault(faction.Value, 0);
        
        if (spent + cost > MaxBudgetPerFaction)
            return PlacementResult.BudgetExceeded;
            
        // Check placement limit
        int currentCount = GetPlacedCount<T>(battle, faction.Value);
        int limit = PlacementLimits.GetValueOrDefault(typeof(T), 0);
        
        if (currentCount >= limit)
            return PlacementResult.LimitReached;
            
        // Check silver
        if (!player.BankBox.ConsumeTotal(typeof(Silver), cost))
            return PlacementResult.InsufficientSilver;
            
        // Place defense
        var defense = new T();
        
        if (defense is SiegeTrap trap)
        {
            trap.Battle = battle;
            trap.OwnerFaction = faction.Value;
            trap.OwnerPlayer = player.Serial;
            battle.Traps[faction.Value].Add(trap);
        }
        else if (defense is SiegeTurret turret)
        {
            turret.Battle = battle;
            turret.OwnerFaction = faction.Value;
            turret.OwnerPlayer = player.Serial;
            battle.Turrets[faction.Value].Add(turret);
        }
        
        defense.MoveToWorld(location, battle.City.Map);
        
        // Update budget
        battle.DefenseBudgetSpent[faction.Value] = spent + cost;
        
        player.SendMessage(0x35, $"Defense placed! {cost} silver spent. Budget remaining: {MaxBudgetPerFaction - spent - cost}");
        
        return PlacementResult.Success;
    }
}
```

---

## 10. Reward Distribution

### 10.1 Victory Rewards

```csharp
public static class SiegeRewardDistributor
{
    public static void DistributeVictoryRewards(SiegeBattle battle, FactionId winner)
    {
        // Update town control
        TownControlManager.SetControl(battle.City, winner);
        
        // Get all participants
        var participants = battle.Participants[winner];
        
        foreach (var serial in participants)
        {
            var pm = World.FindMobile(serial) as PlayerMobile;
            if (pm == null)
                continue;
                
            // Bonus silver for winning
            int bonusSilver = 100;
            AwardSilver(pm, bonusSilver, "Siege Victory");
            
            // Bonus faction points
            int bonusPoints = 200;
            FactionPointManager.AwardPoints(pm, bonusPoints, "Siege Victory");
            
            // Placeholder reward item
            var voucher = new VvVPlaceholderReward
            {
                PlaceholderType = VvVPlaceholderReward.RewardType.Undetermined,
                SourceCorrelationId = battle.CorrelationId
            };
            pm.AddToBackpack(voucher);
            
            pm.SendMessage(0x35, "Congratulations! Your faction has won the siege!");
        }
        
        // Announce
        World.Broadcast(0x35, true,
            $"═══════════════════════════════════════════");
        World.Broadcast(0x35, true,
            $"  {winner} has captured {battle.City.Name}!");
        World.Broadcast(0x35, true,
            $"═══════════════════════════════════════════");
    }
}
```

### 10.2 Silver Award Helper

```csharp
public static class SilverManager
{
    public static void AwardSilver(PlayerMobile player, int amount, string reason)
    {
        var silver = new Silver(amount);
        
        if (player.BankBox != null)
        {
            player.BankBox.DropItem(silver);
            player.SendMessage(0x35, $"+{amount} silver ({reason})");
        }
        
        // Log for telemetry
        SiegeTelemetry.LogSilverAward(player, amount, reason);
    }
}

public class Silver : Item
{
    [Constructable]
    public Silver() : this(1) { }
    
    [Constructable]
    public Silver(int amount) : base(0xEF0) // Same as gold but different hue
    {
        Name = "Silver";
        Hue = 0x47F; // Silver color
        Stackable = true;
        Amount = amount;
    }
}
```

---

## 11. Kill Attribution

### 11.1 Last Human Attacker Rule

```csharp
public static class SiegeKillAttribution
{
    private static readonly TimeSpan AttributionWindow = TimeSpan.FromSeconds(10);
    
    // Track last human attackers per mobile
    private static Dictionary<Serial, (Serial attacker, DateTime time)> _lastAttackers = new();
    
    public static void OnDamageDealt(Mobile attacker, Mobile victim, int damage)
    {
        if (attacker is PlayerMobile pm && victim is Mobile)
        {
            _lastAttackers[victim.Serial] = (pm.Serial, DateTime.UtcNow);
        }
    }
    
    public static PlayerMobile GetKillCredit(Mobile victim, Mobile directKiller)
    {
        // If direct killer is player, they get credit
        if (directKiller is PlayerMobile pm)
            return pm;
            
        // If killed by trap/turret, check last human attacker
        if (_lastAttackers.TryGetValue(victim.Serial, out var record))
        {
            if (DateTime.UtcNow - record.time <= AttributionWindow)
            {
                return World.FindMobile(record.attacker) as PlayerMobile;
            }
        }
        
        // No human attacker - no credit
        return null;
    }
    
    public static void OnMobileDeath(Mobile victim, Mobile killer, SiegeBattle battle)
    {
        if (!(victim is PlayerMobile deadPlayer))
            return;
            
        var creditedKiller = GetKillCredit(victim, killer);
        
        if (creditedKiller == null)
        {
            // Trap/turret kill with no recent human attacker
            SiegeTelemetry.LogEvent(battle, SiegeEventType.UnattributedKill, deadPlayer);
            return;
        }
        
        // Award kill
        var killerFaction = creditedKiller.Guild?.Faction;
        if (killerFaction == null)
            return;
            
        var (points, silver) = SiegeKillScoring.CalculateKillReward(creditedKiller, deadPlayer, battle);
        
        battle.Scores[killerFaction.Value] += points;
        SilverManager.AwardSilver(creditedKiller, silver, $"Kill: {deadPlayer.Name}");
        
        // Update Glicko ratings
        GlickoManager.RecordMatch(creditedKiller, deadPlayer, creditedKiller);
        
        // Award faction points
        FactionPointManager.AwardPoints(creditedKiller, points, $"Siege Kill: {deadPlayer.Name}");
        
        // Track participation
        battle.Participants[killerFaction.Value].Add(creditedKiller.Serial);
        
        SiegeTelemetry.LogEvent(battle, SiegeEventType.PlayerKill, creditedKiller,
            $"Victim: {deadPlayer.Name}, Points: {points}, Silver: {silver}");
    }
}
```

---

## 12. Telemetry & Observability

### 12.1 Event Logging

```csharp
public enum SiegeEventType
{
    BattleStarted,
    BattleEnded,
    PlayerKill,
    UnattributedKill,
    SigilPickup,
    SigilCapture,
    SigilDropped,
    AltarCaptured,
    AltarTick,
    TrapPlaced,
    TrapTriggered,
    TurretPlaced,
    TurretDestroyed,
    DefenseBudgetSpent,
    UncontestedPenaltyApplied,
    PlayerJoined,
    PlayerLeft
}

public static class SiegeTelemetry
{
    public static void LogEvent(
        SiegeBattle battle, 
        SiegeEventType eventType, 
        PlayerMobile player,
        string details = null)
    {
        var entry = new SiegeEventLog
        {
            BattleId = battle.BattleId,
            CorrelationId = battle.CorrelationId,
            Timestamp = DateTime.UtcNow,
            EventType = eventType,
            PlayerSerial = player?.Serial ?? Serial.MinusOne,
            PlayerName = player?.Name,
            Faction = player?.Guild?.Faction,
            City = battle.City.Name,
            Details = details
        };
        
        // Write to database async
        _ = DatabaseManager.WriteSiegeEventAsync(entry);
        
        // Metrics
        Metrics.IncrementCounter($"siege_events_{eventType}");
    }
}
```

### 12.2 Key Metrics

```csharp
public static class SiegeMetrics
{
    // Counters
    public static void RecordKill(FactionId killer, FactionId victim)
        => Metrics.IncrementCounter("siege_kills_total", 
            ("killer_faction", killer.ToString()),
            ("victim_faction", victim.ToString()));
    
    public static void RecordSilverAwarded(int amount, string reason)
        => Metrics.RecordHistogram("siege_silver_awarded", amount,
            ("reason", reason));
    
    public static void RecordUncontestedPenalty(double penalty)
        => Metrics.RecordHistogram("siege_uncontested_penalty", penalty);
    
    public static void RecordBattleDuration(TimeSpan duration)
        => Metrics.RecordHistogram("siege_battle_duration_seconds", duration.TotalSeconds);
    
    public static void RecordParticipantCount(int count)
        => Metrics.RecordGauge("siege_participants", count);
}
```

---

## 13. Admin Commands

### 13.1 Mechanic Toggle System

GMs can enable/disable specific siege mechanics at runtime. Changes persist through server restarts.

```csharp
public static class SiegeMechanicToggles
{
    // Persisted toggle states
    private static bool _sigilEnabled = true;
    private static bool _altarsEnabled = true;
    
    [CommandProperty(AccessLevel.Administrator)]
    public static bool SigilMechanicEnabled
    {
        get => _sigilEnabled;
        set
        {
            _sigilEnabled = value;
            OnMechanicToggled("Sigil", value);
        }
    }
    
    [CommandProperty(AccessLevel.Administrator)]
    public static bool AltarMechanicEnabled
    {
        get => _altarsEnabled;
        set
        {
            _altarsEnabled = value;
            OnMechanicToggled("Altar", value);
        }
    }
    
    private static void OnMechanicToggled(string mechanic, bool enabled)
    {
        // Log the change
        Console.WriteLine($"[Siege] {mechanic} mechanic {(enabled ? "ENABLED" : "DISABLED")} by admin");
        
        // If there's an active siege, update it
        if (SiegeManager.CurrentBattle != null)
        {
            UpdateActiveSiege(mechanic, enabled);
        }
        
        // Persist to config
        SaveToggles();
    }
    
    private static void UpdateActiveSiege(string mechanic, bool enabled)
    {
        var battle = SiegeManager.CurrentBattle;
        
        switch (mechanic)
        {
            case "Sigil":
                if (!enabled)
                {
                    // Despawn sigil and priests
                    battle.Sigil?.Delete();
                    battle.Sigil = null;
                    
                    foreach (var priest in battle.Priests)
                        priest.Delete();
                    battle.Priests.Clear();
                    
                    SiegeBroadcast(battle, "The Sigil objective has been disabled by the Game Masters.");
                }
                else
                {
                    // Spawn sigil and priests
                    SpawnSigilAndPriests(battle);
                    SiegeBroadcast(battle, "The Sigil objective has been enabled!");
                }
                break;
                
            case "Altar":
                if (!enabled)
                {
                    // Despawn all altars
                    foreach (var altar in battle.Altars)
                        altar.Delete();
                    battle.Altars.Clear();
                    
                    SiegeBroadcast(battle, "Altar objectives have been disabled by the Game Masters.");
                }
                else
                {
                    // Spawn altars
                    SpawnAltars(battle);
                    SiegeBroadcast(battle, "Altar objectives have been enabled!");
                }
                break;
        }
    }
    
    // Persistence
    private static readonly string ToggleFilePath = "Data/SiegeToggles.json";
    
    public static void LoadToggles()
    {
        if (File.Exists(ToggleFilePath))
        {
            var json = File.ReadAllText(ToggleFilePath);
            var data = JsonSerializer.Deserialize<SiegeToggleData>(json);
            _sigilEnabled = data.SigilEnabled;
            _altarsEnabled = data.AltarsEnabled;
        }
    }
    
    public static void SaveToggles()
    {
        var data = new SiegeToggleData
        {
            SigilEnabled = _sigilEnabled,
            AltarsEnabled = _altarsEnabled
        };
        var json = JsonSerializer.Serialize(data, new JsonSerializerOptions { WriteIndented = true });
        File.WriteAllText(ToggleFilePath, json);
    }
    
    private class SiegeToggleData
    {
        public bool SigilEnabled { get; set; } = true;
        public bool AltarsEnabled { get; set; } = true;
    }
}
```

### 13.2 Toggle Admin Commands

```csharp
public static class SiegeToggleCommands
{
    [Usage("[siege sigil <on|off>")]
    [Description("Enable or disable the Sigil mechanic (includes Priests)")]
    public static void ToggleSigil(CommandEventArgs e)
    {
        if (string.IsNullOrEmpty(e.ArgString))
        {
            // Show current status
            e.Mobile.SendMessage(0x35, $"Sigil Mechanic: {(SiegeMechanicToggles.SigilMechanicEnabled ? "ENABLED" : "DISABLED")}");
            e.Mobile.SendMessage(0x35, "Usage: [siege sigil <on|off>");
            return;
        }
        
        string arg = e.ArgString.Trim().ToLower();
        
        switch (arg)
        {
            case "on":
            case "true":
            case "enable":
            case "1":
                SiegeMechanicToggles.SigilMechanicEnabled = true;
                e.Mobile.SendMessage(0x35, "Sigil mechanic ENABLED. Priests will spawn with sigil.");
                World.Broadcast(0x35, true, "[GM] Sigil objectives have been enabled for sieges.");
                break;
                
            case "off":
            case "false":
            case "disable":
            case "0":
                SiegeMechanicToggles.SigilMechanicEnabled = false;
                e.Mobile.SendMessage(0x22, "Sigil mechanic DISABLED. No sigil or priests will spawn.");
                World.Broadcast(0x22, true, "[GM] Sigil objectives have been disabled for sieges.");
                break;
                
            default:
                e.Mobile.SendMessage(0x22, "Invalid argument. Use: [siege sigil <on|off>");
                break;
        }
    }
    
    [Usage("[siege altar <on|off>")]
    [Description("Enable or disable the Altar mechanic")]
    public static void ToggleAltar(CommandEventArgs e)
    {
        if (string.IsNullOrEmpty(e.ArgString))
        {
            // Show current status
            e.Mobile.SendMessage(0x35, $"Altar Mechanic: {(SiegeMechanicToggles.AltarMechanicEnabled ? "ENABLED" : "DISABLED")}");
            e.Mobile.SendMessage(0x35, "Usage: [siege altar <on|off>");
            return;
        }
        
        string arg = e.ArgString.Trim().ToLower();
        
        switch (arg)
        {
            case "on":
            case "true":
            case "enable":
            case "1":
                SiegeMechanicToggles.AltarMechanicEnabled = true;
                e.Mobile.SendMessage(0x35, "Altar mechanic ENABLED. Altars will spawn in sieges.");
                World.Broadcast(0x35, true, "[GM] Altar objectives have been enabled for sieges.");
                break;
                
            case "off":
            case "false":
            case "disable":
            case "0":
                SiegeMechanicToggles.AltarMechanicEnabled = false;
                e.Mobile.SendMessage(0x22, "Altar mechanic DISABLED. No altars will spawn.");
                World.Broadcast(0x22, true, "[GM] Altar objectives have been disabled for sieges.");
                break;
                
            default:
                e.Mobile.SendMessage(0x22, "Invalid argument. Use: [siege altar <on|off>");
                break;
        }
    }
    
    [Usage("[siege toggles")]
    [Description("Show status of all siege mechanic toggles")]
    public static void ShowToggles(CommandEventArgs e)
    {
        e.Mobile.SendMessage(0x35, "═══ Siege Mechanic Toggles ═══");
        e.Mobile.SendMessage(0x35, $"  Sigil (+ Priests): {(SiegeMechanicToggles.SigilMechanicEnabled ? "ENABLED" : "DISABLED")}");
        e.Mobile.SendMessage(0x35, $"  Altars:            {(SiegeMechanicToggles.AltarMechanicEnabled ? "ENABLED" : "DISABLED")}");
        e.Mobile.SendMessage(0x35, "═══════════════════════════════");
        e.Mobile.SendMessage(0x35, "Use [siege sigil <on|off> or [siege altar <on|off> to change.");
    }
}
```

### 13.3 Battle Spawn Integration

The siege battle system respects toggle states when spawning objectives:

```csharp
public static class SiegeBattleSpawner
{
    public static void SpawnObjectives(SiegeBattle battle)
    {
        // Always spawn based on current toggle state
        
        if (SiegeMechanicToggles.SigilMechanicEnabled)
        {
            SpawnSigilAndPriests(battle);
        }
        else
        {
            Console.WriteLine($"[Siege] Sigil mechanic disabled - skipping sigil/priest spawn");
        }
        
        if (SiegeMechanicToggles.AltarMechanicEnabled)
        {
            SpawnAltars(battle);
        }
        else
        {
            Console.WriteLine($"[Siege] Altar mechanic disabled - skipping altar spawn");
        }
    }
    
    private static void SpawnSigilAndPriests(SiegeBattle battle)
    {
        // Spawn sigil at designated location
        var sigil = new SiegeSigil(battle);
        sigil.MoveToWorld(battle.City.SigilSpawn, battle.City.Map);
        battle.Sigil = sigil;
        
        // Spawn one priest per faction
        foreach (FactionId faction in Enum.GetValues<FactionId>())
        {
            var priestLoc = battle.City.PriestLocations[(int)faction];
            var priest = new SiegePriest(battle, faction);
            priest.MoveToWorld(priestLoc, battle.City.Map);
            battle.Priests.Add(priest);
        }
        
        SiegeBroadcast(battle, "The Sigil has spawned! Capture it and return to your Priest!");
    }
    
    private static void SpawnAltars(SiegeBattle battle)
    {
        for (int i = 0; i < battle.City.AltarLocations.Length; i++)
        {
            var altar = new SiegeAltar(battle, i + 1);
            altar.MoveToWorld(battle.City.AltarLocations[i], battle.City.Map);
            battle.Altars.Add(altar);
        }
        
        SiegeBroadcast(battle, $"{battle.Altars.Count} Altars are now active! Capture and hold them for points!");
    }
}
```

### 13.4 Command Reference

| Command | Description | Example |
|---------|-------------|---------|
| `[siege sigil` | Show sigil mechanic status | `[siege sigil` |
| `[siege sigil on` | Enable sigil + priests | `[siege sigil on` |
| `[siege sigil off` | Disable sigil + priests | `[siege sigil off` |
| `[siege altar` | Show altar mechanic status | `[siege altar` |
| `[siege altar on` | Enable altars | `[siege altar on` |
| `[siege altar off` | Disable altars | `[siege altar off` |
| `[siege toggles` | Show all toggle states | `[siege toggles` |

### 13.5 Other Admin Commands

```csharp
public static class SiegeAdminCommands
{
    [Usage("[siege start <city>")]
    [Description("Force start a siege in specified city")]
    public static void ForceStart(CommandEventArgs e)
    {
        string cityName = e.ArgString.Trim();
        var city = SiegeCities.All.FirstOrDefault(c => 
            c.Name.Equals(cityName, StringComparison.OrdinalIgnoreCase));
            
        if (city == null)
        {
            e.Mobile.SendMessage(0x22, $"Unknown city. Valid: {string.Join(", ", SiegeCities.All.Select(c => c.Name))}");
            return;
        }
        
        SiegeManager.StartSiege(city);
        e.Mobile.SendMessage(0x35, $"Siege started in {city.Name}");
    }
    
    [Usage("[siege stop")]
    [Description("Force stop current siege")]
    public static void ForceStop(CommandEventArgs e)
    {
        SiegeManager.EndCurrentSiege(SiegeEndReason.AdminForced);
        e.Mobile.SendMessage(0x35, "Siege stopped.");
    }
    
    [Usage("[siege setcontrol <city> <faction>")]
    [Description("Set town control")]
    public static void SetControl(CommandEventArgs e)
    {
        // Parse args and set control
    }
    
    [Usage("[siege status")]
    [Description("Show siege status")]
    public static void Status(CommandEventArgs e)
    {
        if (SiegeManager.CurrentBattle == null)
        {
            e.Mobile.SendMessage(0x35, "No active siege.");
        }
        else
        {
            var b = SiegeManager.CurrentBattle;
            e.Mobile.SendMessage(0x35, $"City: {b.City.Name}");
            e.Mobile.SendMessage(0x35, $"Time Remaining: {(b.EndTime - DateTime.UtcNow).TotalMinutes:F1} min");
            foreach (var (faction, score) in b.Scores)
            {
                e.Mobile.SendMessage(0x35, $"  {faction}: {score} points");
            }
        }
    }
    
    [Usage("[siege givesilver <player> <amount>")]
    [Description("Give silver to player")]
    public static void GiveSilver(CommandEventArgs e)
    {
        // Admin silver grant
    }
}
```

---

## 14. Configuration Reference

```csharp
public static class SiegeConfig
{
    // Mechanic Toggles (runtime adjustable via admin commands)
    public static bool SigilMechanicEnabled = true;  // [siege sigil on/off
    public static bool AltarMechanicEnabled = true;  // [siege altar on/off
    
    // Timing
    public static TimeSpan WarningDuration = TimeSpan.FromMinutes(5);
    public static TimeSpan BattleDuration = TimeSpan.FromMinutes(30);
    public static TimeSpan GlobalCooldown = TimeSpan.FromMinutes(30);
    public static TimeSpan CityCooldown = TimeSpan.FromHours(2);
    
    // Trigger requirements
    public static int MinPlayersPerFaction = 3;
    public static int MinTotalPlayers = 10;
    
    // Victory
    public static int VictoryScore = 10000;
    
    // Scoring
    public static int KillBasePoints = 100;
    public static int KillBaseSilver = 50;
    public static int AssistPoints = 25;
    public static int AssistSilver = 10;
    public static int SigilCapturePoints = 500;
    public static int SigilCaptureSilver = 200;
    public static int SigilStealPoints = 100;
    public static int SigilStealSilver = 50;
    public static int AltarCapturePoints = 250;
    public static int AltarCaptureSilver = 100;
    public static int AltarTickPoints = 25;
    public static int AltarTickSilver = 10;
    public static int TrapTriggerPoints = 10;
    public static int TrapTriggerSilver = 5;
    
    // Uncontested penalties
    public static double UncontestedPenalty0 = 0.80;  // 0 opponents
    public static double UncontestedPenalty1_2 = 0.60; // 1-2 opponents
    public static double UncontestedPenalty3_5 = 0.15; // 3-5 opponents
    public static int UncontestedThresholdMinutes = 10;
    
    // Defense budget
    public static int MaxDefenseBudget = 5000;
    
    // Town control
    public static double SingleCityDiscount = 0.10;
    public static double AllCitiesDiscount = 0.15;
    
    // Combat status
    public static TimeSpan CombatStatusCooldown = TimeSpan.FromHours(24);
    
    // Kill attribution
    public static TimeSpan KillAttributionWindow = TimeSpan.FromSeconds(10);
    
    // Sigil
    public static TimeSpan SigilRespawnDelay = TimeSpan.FromMinutes(2);
    
    // Altar
    public static TimeSpan AltarCaptureTime = TimeSpan.FromSeconds(60);
    public static TimeSpan AltarTickInterval = TimeSpan.FromSeconds(30);
}
```

---

## 15. Testing Checklist

### Trigger System
- [ ] Siege triggers when minimum players met
- [ ] Siege does not trigger during cooldown
- [ ] City selection avoids recently sieged cities
- [ ] 5-minute warning broadcasts correctly

### Mechanic Toggles
- [ ] `[siege sigil on/off` works correctly
- [ ] `[siege altar on/off` works correctly
- [ ] `[siege toggles` displays current state
- [ ] Toggle state persists through server restart
- [ ] Disabling during active siege despawns objectives
- [ ] Enabling during active siege spawns objectives
- [ ] New sieges respect toggle state
- [ ] Priests despawn with sigil when disabled

### Battle Flow
- [ ] Sigil spawns correctly
- [ ] Altars spawn at designated locations
- [ ] Priests spawn per faction
- [ ] Score tracking accurate
- [ ] Victory at 10,000 points works
- [ ] Time expiration determines winner correctly
- [ ] Tie-breaker uses participant count

### Objectives
- [ ] Sigil pickup awards points
- [ ] Sigil capture at priest works
- [ ] Sigil drops on carrier death
- [ ] Altar capture requires 60s
- [ ] Altar tick awards periodic points
- [ ] Altar capture interrupted by leaving

### Peaceful Participant
- [ ] Can set status at guild stone
- [ ] 24-hour cooldown enforced
- [ ] Peaceful players not attackable outside siege
- [ ] Peaceful players flagged in siege zone
- [ ] Peaceful status restored on leaving siege

### Traps & Turrets
- [ ] All 4 trap types function correctly
- [ ] Both turret types function correctly
- [ ] Defense budget limits enforced
- [ ] Placement limits enforced
- [ ] Detection skill avoids traps
- [ ] Turrets targetable and destroyable
- [ ] Trap/turret kills attributed to last human attacker

### Town Control
- [ ] Winner gains control after siege
- [ ] Banners update to faction colors
- [ ] 10% discount applies to controlling faction
- [ ] 15% discount applies when controlling all 4 cities
- [ ] Control persists through server restart

### Rewards
- [ ] Silver awarded correctly
- [ ] Faction points awarded correctly
- [ ] Placeholder vouchers given
- [ ] Glicko ratings updated
- [ ] Uncontested penalty applied correctly

### Persistence
- [ ] Battle state survives server restart
- [ ] Town control persists
- [ ] Trap/turret placements restore correctly

---

## 16. Integration Points

| System | Integration |
|--------|-------------|
| **Faction System** | Uses FactionId enum (Vampire, Daemon, Goblin) |
| **Guild System** | Guild.Faction property determines eligibility |
| **Glicko-2** | Kill scoring weighted by rating difference |
| **Faction Points** | Kill/objective points add to quarterly total |
| **Town Cryer** | Announces siege warnings, results |
| **Economy** | Silver currency + gold sinks |

---

## 17. Change Log

### v1.0.0 - 2025-01-02 (Initial Specification)
- Created comprehensive VvV Siege System specification
- 3-faction free-for-all model
- 4 siege cities: Jhelom, Skara Brae, Yew, Trinsic
- Persistent town control with NPC discounts
- On-demand siege triggering
- 30-minute battle duration
- Peaceful participant system for non-PvP guild members
- Dual currency: Faction Points + Silver
- Sphere-compatible trap system (Alarm, Snare, Smoke, Mana Drain)
- Turret system (Arrow, Magic)
- Defense budget system (5,000 silver max)
- Kill attribution with 10-second human attacker window
- Sliding uncontested penalty (80/60/15/0%)
- Complete scoring and reward system