# Daily Content System Specification
## 51alpha Daily Bounties, Faction Quest & Silver Vendor

### Document Metadata
- **Version**: v1.0.0
- **Last Updated**: 2025-01-02
- **Authors**: 51alpha Development Team
- **Applicable ModernUO Version**: v24.0.0+
- **Dependencies**: VvV_Integration.md, VvV_Siege_System.md
- **PR Ready**: No (Conceptual Design)

---

## 1. Executive Summary

The Daily Content System provides three interconnected systems for daily player engagement:

1. **Daily Bounties** - 3 universal tasks reset at server midnight
2. **Daily Faction Quest** - Mini-boss hunt at announced location
3. **Silver Vendor** - PvP currency shop for cosmetics and decorations

### Key Design Decisions

| System | Decision | Rationale |
|--------|----------|-----------|
| Bounties | Same tasks for all players | Community coordination, fair competition |
| Bounties | Server midnight reset | Predictable schedule |
| Bounties | Peaceful members eligible | Inclusive for all guild members |
| Faction Quest | Location announced | Encourages participation over exploration |
| Faction Quest | Chance-based relic drops | Maintains relic economy |
| Silver Vendor | Faction-colored cosmetics | Visual faction identity |

---

## 2. Daily Bounty System

### 2.1 Overview

Three daily tasks available to ALL faction members (Combatant and Peaceful). Same tasks for everyone, reset at server midnight.

### 2.2 Task Generation

```csharp
public static class DailyBountyManager
{
    // Current day's bounties (same for all players)
    private static DailyBountySet _todaysBounties;
    private static DateTime _lastResetDate;
    
    // Reset time: Server midnight (configurable timezone)
    public static readonly TimeSpan ResetTime = TimeSpan.Zero; // 00:00:00
    public static readonly string ServerTimezone = "Eastern Standard Time";
    
    public class DailyBountySet
    {
        public DateTime Date { get; set; }
        public BountyTask Task1_Monster { get; set; }
        public BountyTask Task2_Resource { get; set; }
        public BountyTask Task3_Activity { get; set; }
    }
    
    public class BountyTask
    {
        public BountyType Type { get; set; }
        public string TargetName { get; set; }
        public int RequiredCount { get; set; }
        public int FactionPointReward { get; set; }
        public int SilverReward { get; set; }
    }
    
    public enum BountyType
    {
        KillMonster,
        GatherResource,
        CraftItem,
        ExploreLocation
    }
}
```

### 2.3 Task Pools

Tasks are randomly selected daily from these pools:

#### Monster Kill Tasks
```csharp
public static readonly MonsterBounty[] MonsterPool = new[]
{
    // Easy (100-150 faction points, 25-50 silver)
    new MonsterBounty("Skeleton", 20, 100, 25),
    new MonsterBounty("Zombie", 20, 100, 25),
    new MonsterBounty("Orc", 15, 100, 30),
    new MonsterBounty("Lizardman", 15, 100, 30),
    new MonsterBounty("Ratman", 20, 100, 25),
    
    // Medium (150-200 faction points, 50-75 silver)
    new MonsterBounty("Ettin", 10, 150, 50),
    new MonsterBounty("Ogre", 10, 150, 50),
    new MonsterBounty("Troll", 10, 150, 50),
    new MonsterBounty("Gargoyle", 8, 175, 60),
    new MonsterBounty("Harpy", 12, 150, 50),
    
    // Hard (200-300 faction points, 75-100 silver)
    new MonsterBounty("Drake", 5, 200, 75),
    new MonsterBounty("Daemon", 3, 250, 100),
    new MonsterBounty("Lich", 3, 250, 100),
    new MonsterBounty("Elder Gazer", 4, 225, 85),
    new MonsterBounty("Balron", 2, 300, 100)
};
```

#### Resource Gathering Tasks
```csharp
public static readonly ResourceBounty[] ResourcePool = new[]
{
    // Mining
    new ResourceBounty("Iron Ore", 100, 100, 25),
    new ResourceBounty("Dull Copper Ore", 75, 125, 35),
    new ResourceBounty("Shadow Ore", 50, 150, 50),
    new ResourceBounty("Valorite Ore", 25, 200, 75),
    
    // Lumberjacking
    new ResourceBounty("Logs", 100, 100, 25),
    new ResourceBounty("Oak Logs", 75, 125, 35),
    new ResourceBounty("Yew Logs", 50, 150, 50),
    new ResourceBounty("Heartwood", 25, 200, 75),
    
    // Fishing
    new ResourceBounty("Fish", 50, 100, 25),
    new ResourceBounty("Big Fish", 10, 150, 50),
    
    // Skinning
    new ResourceBounty("Leather", 100, 100, 25),
    new ResourceBounty("Spined Leather", 50, 150, 50),
    new ResourceBounty("Horned Leather", 25, 175, 65),
    new ResourceBounty("Barbed Leather", 15, 200, 75)
};
```

#### Activity Tasks (Third Slot)
```csharp
public static readonly ActivityBounty[] ActivityPool = new[]
{
    // Crafting
    new ActivityBounty("Craft Bandages", 50, 100, 25),
    new ActivityBounty("Craft Potions", 20, 125, 35),
    new ActivityBounty("Craft Arrows", 100, 100, 25),
    new ActivityBounty("Craft Armor Pieces", 10, 150, 50),
    new ActivityBounty("Craft Weapons", 5, 175, 60),
    
    // Exploration
    new ActivityBounty("Visit Dungeon Entrance", 3, 100, 25),
    new ActivityBounty("Enter Champion Spawn Area", 1, 150, 50),
    
    // Social
    new ActivityBounty("Trade with Players", 3, 100, 25),
    new ActivityBounty("Resurrect Allied Player", 2, 125, 35)
};
```

### 2.4 Daily Generation Algorithm

```csharp
public static void GenerateDailyBounties()
{
    var today = GetServerDate();
    
    // Use date as seed for deterministic "random" (same for all players)
    var seed = today.Year * 10000 + today.Month * 100 + today.Day;
    var rng = new Random(seed);
    
    _todaysBounties = new DailyBountySet
    {
        Date = today,
        Task1_Monster = SelectRandom(MonsterPool, rng),
        Task2_Resource = SelectRandom(ResourcePool, rng),
        Task3_Activity = SelectRandom(ActivityPool, rng)
    };
    
    // Announce to players
    World.Broadcast(0x35, true, "Daily Bounties have been refreshed! Check with the Bounty Board.");
    
    // Log for debugging
    Console.WriteLine($"[Bounties] Generated for {today:yyyy-MM-dd}: " +
        $"{_todaysBounties.Task1_Monster.TargetName}, " +
        $"{_todaysBounties.Task2_Resource.TargetName}, " +
        $"{_todaysBounties.Task3_Activity.TargetName}");
}

private static DateTime GetServerDate()
{
    var tz = TimeZoneInfo.FindSystemTimeZoneById(ServerTimezone);
    return TimeZoneInfo.ConvertTimeFromUtc(DateTime.UtcNow, tz).Date;
}
```

### 2.5 Player Progress Tracking

```csharp
public class PlayerBountyProgress
{
    public Serial PlayerSerial { get; set; }
    public DateTime Date { get; set; }
    
    // Task progress
    public int Task1_Progress { get; set; }
    public bool Task1_Claimed { get; set; }
    
    public int Task2_Progress { get; set; }
    public bool Task2_Claimed { get; set; }
    
    public int Task3_Progress { get; set; }
    public bool Task3_Claimed { get; set; }
}

public static class BountyProgressTracker
{
    private static Dictionary<Serial, PlayerBountyProgress> _progress = new();
    
    public static PlayerBountyProgress GetProgress(PlayerMobile player)
    {
        var today = GetServerDate();
        
        if (!_progress.TryGetValue(player.Serial, out var progress) || progress.Date != today)
        {
            // New day or new player - reset progress
            progress = new PlayerBountyProgress
            {
                PlayerSerial = player.Serial,
                Date = today
            };
            _progress[player.Serial] = progress;
        }
        
        return progress;
    }
    
    public static void OnMonsterKilled(PlayerMobile player, BaseCreature creature)
    {
        var bounties = DailyBountyManager.GetTodaysBounties();
        if (bounties.Task1_Monster.TargetName != creature.GetType().Name)
            return;
            
        var progress = GetProgress(player);
        if (progress.Task1_Claimed)
            return;
            
        progress.Task1_Progress++;
        
        // Notify player
        int remaining = bounties.Task1_Monster.RequiredCount - progress.Task1_Progress;
        if (remaining > 0)
        {
            player.SendMessage(0x35, $"Bounty Progress: {bounties.Task1_Monster.TargetName} - {remaining} remaining");
        }
        else
        {
            player.SendMessage(0x35, $"Bounty Complete! Return to the Bounty Board to claim your reward.");
        }
    }
    
    // Similar methods for OnResourceGathered, OnItemCrafted, etc.
}
```

### 2.6 Bounty Board NPC/Item

```csharp
public class BountyBoard : Item
{
    [Constructable]
    public BountyBoard() : base(0x1E5E) // Bulletin board graphic
    {
        Name = "Daily Bounty Board";
        Movable = false;
    }
    
    public override void OnDoubleClick(Mobile from)
    {
        if (!(from is PlayerMobile pm))
            return;
            
        if (pm.Guild?.Faction == null)
        {
            pm.SendMessage(0x22, "You must be in a faction to view bounties.");
            return;
        }
        
        pm.SendGump(new BountyBoardGump(pm));
    }
}

public class BountyBoardGump : Gump
{
    private PlayerMobile _player;
    
    public BountyBoardGump(PlayerMobile player) : base(50, 50)
    {
        _player = player;
        var bounties = DailyBountyManager.GetTodaysBounties();
        var progress = BountyProgressTracker.GetProgress(player);
        
        AddBackground(0, 0, 400, 350, 9200);
        AddLabel(140, 20, 0x35, "Daily Bounties");
        AddLabel(20, 45, 0, $"Resets at midnight server time");
        
        int y = 80;
        
        // Task 1: Monster Kill
        AddBountyEntry(y, "Hunt", bounties.Task1_Monster, 
            progress.Task1_Progress, progress.Task1_Claimed, 1);
        y += 70;
        
        // Task 2: Resource Gathering
        AddBountyEntry(y, "Gather", bounties.Task2_Resource,
            progress.Task2_Progress, progress.Task2_Claimed, 2);
        y += 70;
        
        // Task 3: Activity
        AddBountyEntry(y, "Activity", bounties.Task3_Activity,
            progress.Task3_Progress, progress.Task3_Claimed, 3);
    }
    
    private void AddBountyEntry(int y, string category, BountyTask task, 
        int progress, bool claimed, int buttonId)
    {
        AddLabel(20, y, 0x35, $"{category}:");
        AddLabel(80, y, 0, task.TargetName);
        AddLabel(20, y + 20, 0, $"Progress: {progress}/{task.RequiredCount}");
        AddLabel(20, y + 40, 0, $"Reward: {task.FactionPointReward} FP, {task.SilverReward} Silver");
        
        if (claimed)
        {
            AddLabel(300, y + 20, 0x35, "CLAIMED");
        }
        else if (progress >= task.RequiredCount)
        {
            AddButton(300, y + 20, 4005, 4007, buttonId, GumpButtonType.Reply, 0);
            AddLabel(335, y + 20, 0x35, "Claim");
        }
        else
        {
            AddLabel(300, y + 20, 0x22, "Incomplete");
        }
    }
    
    public override void OnResponse(NetState sender, RelayInfo info)
    {
        var bounties = DailyBountyManager.GetTodaysBounties();
        var progress = BountyProgressTracker.GetProgress(_player);
        
        BountyTask task = null;
        bool canClaim = false;
        
        switch (info.ButtonID)
        {
            case 1:
                task = bounties.Task1_Monster;
                canClaim = progress.Task1_Progress >= task.RequiredCount && !progress.Task1_Claimed;
                if (canClaim) progress.Task1_Claimed = true;
                break;
            case 2:
                task = bounties.Task2_Resource;
                canClaim = progress.Task2_Progress >= task.RequiredCount && !progress.Task2_Claimed;
                if (canClaim) progress.Task2_Claimed = true;
                break;
            case 3:
                task = bounties.Task3_Activity;
                canClaim = progress.Task3_Progress >= task.RequiredCount && !progress.Task3_Claimed;
                if (canClaim) progress.Task3_Claimed = true;
                break;
        }
        
        if (canClaim && task != null)
        {
            // Award rewards
            FactionPointManager.AwardPoints(_player, task.FactionPointReward, $"Bounty: {task.TargetName}");
            SilverManager.AwardSilver(_player, task.SilverReward, $"Bounty: {task.TargetName}");
            
            _player.SendMessage(0x35, $"Bounty claimed! +{task.FactionPointReward} Faction Points, +{task.SilverReward} Silver");
            _player.PlaySound(0x5B5);
        }
        
        // Refresh gump
        _player.SendGump(new BountyBoardGump(_player));
    }
}
```

### 2.7 Bounty Board Locations

Place Bounty Boards in these locations:
- Britain (main city)
- Each siege city (Jhelom, Skara Brae, Yew, Trinsic)
- Near dungeon entrances

---

## 3. Daily Faction Quest

### 3.1 Overview

A daily mini-boss spawns at one of 4 swamp/desert locations. Location is **announced** to all faction players. Killing awards faction points; relic drops are **chance-based**.

### 3.2 Mini-Boss Locations

```csharp
public static class FactionQuestLocations
{
    public static readonly QuestLocation[] Locations = new[]
    {
        // Swamp locations
        new QuestLocation
        {
            Name = "Fens of the Dead",
            Region = "Swamp",
            SpawnPoint = new Point3D(/* TBD */),
            Map = Map.Felucca
        },
        new QuestLocation
        {
            Name = "Bog of Despair", 
            Region = "Swamp",
            SpawnPoint = new Point3D(/* TBD */),
            Map = Map.Felucca
        },
        
        // Desert locations
        new QuestLocation
        {
            Name = "Scorched Sands",
            Region = "Desert",
            SpawnPoint = new Point3D(/* TBD */),
            Map = Map.Felucca
        },
        new QuestLocation
        {
            Name = "Ruins of the Sun Temple",
            Region = "Desert", 
            SpawnPoint = new Point3D(/* TBD */),
            Map = Map.Felucca
        }
    };
}
```

### 3.3 Mini-Boss Definition

```csharp
public class FactionQuestBoss : BaseCreature
{
    private DateTime _spawnTime;
    private bool _hasBeenKilled;
    
    // Scaling based on player count nearby
    public int BaseHits { get; } = 5000;
    public int HitsPerPlayer { get; } = 500; // +500 HP per player in area
    public int MaxHits { get; } = 25000;
    
    [Constructable]
    public FactionQuestBoss() : base(AIType.AI_Mage, FightMode.Closest, 10, 1, 0.2, 0.4)
    {
        // Rotate between boss types based on location/day
        var bossType = GetTodaysBossType();
        
        switch (bossType)
        {
            case BossType.SwampLord:
                Name = "Swamp Lord";
                Body = 0x1C; // Swamp creature
                Hue = 0x851;
                BaseSoundID = 0x165;
                break;
                
            case BossType.DesertWraith:
                Name = "Desert Wraith";
                Body = 0x3CA; // Wraith-like
                Hue = 0x8A8;
                BaseSoundID = 0x482;
                break;
                
            case BossType.BogHorror:
                Name = "Bog Horror";
                Body = 0x9; // Large creature
                Hue = 0x85D;
                BaseSoundID = 0x16B;
                break;
                
            case BossType.SandElemental:
                Name = "Ancient Sand Elemental";
                Body = 0xD4; // Elemental
                Hue = 0x8B0;
                BaseSoundID = 0x11D;
                break;
        }
        
        _spawnTime = DateTime.UtcNow;
        ScaleToNearbyPlayers();
    }
    
    private void ScaleToNearbyPlayers()
    {
        int playerCount = GetPlayersInRange(20).Count();
        int scaledHits = Math.Min(BaseHits + (playerCount * HitsPerPlayer), MaxHits);
        
        SetHits(scaledHits);
        SetDamage(25, 40);
        
        SetResistance(ResistanceType.Physical, 50, 60);
        SetResistance(ResistanceType.Fire, 40, 50);
        SetResistance(ResistanceType.Cold, 40, 50);
        SetResistance(ResistanceType.Poison, 60, 70);
        SetResistance(ResistanceType.Energy, 40, 50);
        
        Fame = 15000;
        Karma = -15000;
    }
    
    public override void OnDeath(Container c)
    {
        _hasBeenKilled = true;
        
        // Award faction points to all participants
        var damagers = GetLootingRights();
        
        foreach (var damager in damagers)
        {
            if (damager.m_Mobile is PlayerMobile pm && pm.Guild?.Faction != null)
            {
                // Base reward for participation
                int factionPoints = 500;
                
                // Bonus for top damage
                if (damager.m_HasRight)
                    factionPoints += 250;
                
                FactionPointManager.AwardPoints(pm, factionPoints, "Daily Faction Quest");
                pm.SendMessage(0x35, $"Faction Quest Complete! +{factionPoints} Faction Points");
                
                // Mark quest as complete for this player today
                FactionQuestTracker.MarkComplete(pm);
            }
        }
        
        // Chance-based relic drops
        GenerateRelicLoot(c);
        
        base.OnDeath(c);
    }
    
    private void GenerateRelicLoot(Container c)
    {
        // Relic drop chances (per participant with looting rights)
        // These go into corpse for standard looting rules
        
        if (Utility.RandomDouble() < 0.15) // 15% common
            c.DropItem(new CommonRelic());
            
        if (Utility.RandomDouble() < 0.05) // 5% uncommon
            c.DropItem(new UncommonRelic());
            
        if (Utility.RandomDouble() < 0.01) // 1% rare
            c.DropItem(new RareRelic());
            
        // Always drop some gold
        c.DropItem(new Gold(Utility.RandomMinMax(2000, 5000)));
    }
    
    public override bool AutoDispel => true;
    public override bool BardImmune => true;
    public override bool Unprovokable => true;
    public override bool AreaPeaceImmune => true;
}
```

### 3.4 Quest Scheduler

```csharp
public static class DailyFactionQuestManager
{
    private static QuestLocation _todaysLocation;
    private static FactionQuestBoss _currentBoss;
    private static DateTime _lastSpawnDate;
    
    // Boss respawn timer after being killed
    public static readonly TimeSpan RespawnDelay = TimeSpan.FromHours(2);
    private static DateTime? _nextRespawnTime;
    
    public static void Initialize()
    {
        // Check every minute for spawn/respawn
        Timer.StartTimer(TimeSpan.FromMinutes(1), TimeSpan.FromMinutes(1), CheckQuestSpawn);
    }
    
    private static void CheckQuestSpawn()
    {
        var today = GetServerDate();
        
        // New day - pick new location
        if (_lastSpawnDate != today)
        {
            _lastSpawnDate = today;
            SelectTodaysLocation();
            SpawnBoss();
            AnnounceFactionQuest();
        }
        // Check for respawn after kill
        else if (_currentBoss == null || _currentBoss.Deleted)
        {
            if (_nextRespawnTime == null)
            {
                _nextRespawnTime = DateTime.UtcNow + RespawnDelay;
                World.Broadcast(0x35, true, 
                    $"The {_currentBoss?.Name ?? "Faction Boss"} has been slain! It will return in {RespawnDelay.TotalHours} hours.");
            }
            else if (DateTime.UtcNow >= _nextRespawnTime)
            {
                SpawnBoss();
                _nextRespawnTime = null;
                World.Broadcast(0x35, true,
                    $"The Faction Boss has respawned at {_todaysLocation.Name}!");
            }
        }
    }
    
    private static void SelectTodaysLocation()
    {
        // Deterministic selection based on date
        var seed = GetServerDate().DayOfYear;
        var index = seed % FactionQuestLocations.Locations.Length;
        _todaysLocation = FactionQuestLocations.Locations[index];
    }
    
    private static void SpawnBoss()
    {
        _currentBoss = new FactionQuestBoss();
        _currentBoss.MoveToWorld(_todaysLocation.SpawnPoint, _todaysLocation.Map);
    }
    
    private static void AnnounceFactionQuest()
    {
        // World broadcast
        World.Broadcast(0x35, true,
            $"═══════════════════════════════════════════");
        World.Broadcast(0x35, true,
            $"  DAILY FACTION QUEST");
        World.Broadcast(0x35, true,
            $"  A powerful creature lurks at: {_todaysLocation.Name}");
        World.Broadcast(0x35, true,
            $"  Region: {_todaysLocation.Region}");
        World.Broadcast(0x35, true,
            $"═══════════════════════════════════════════");
        
        // Town Cryer
        TownCryerManager.AnnounceEvent(TownCryerCategory.FactionEvent,
            $"Faction Quest: Defeat the boss at {_todaysLocation.Name} ({_todaysLocation.Region})!");
    }
    
    public static QuestLocation GetTodaysLocation() => _todaysLocation;
    public static bool IsBossAlive() => _currentBoss != null && !_currentBoss.Deleted && _currentBoss.Alive;
}
```

### 3.5 Player Tracking

```csharp
public static class FactionQuestTracker
{
    private static Dictionary<Serial, DateTime> _completions = new();
    
    public static bool HasCompletedToday(PlayerMobile player)
    {
        if (!_completions.TryGetValue(player.Serial, out var date))
            return false;
            
        return date == GetServerDate();
    }
    
    public static void MarkComplete(PlayerMobile player)
    {
        _completions[player.Serial] = GetServerDate();
    }
    
    // Players can only get faction points once per day
    // But can fight boss multiple times for relic chances
}
```

---

## 4. Silver Vendor System

### 4.1 Overview

Silver earned from PvP sieges can be spent on faction-themed cosmetics and house decorations.

### 4.2 Vendor Items

```csharp
public static class SilverVendorInventory
{
    public static readonly VendorItem[] Items = new[]
    {
        // ═══ FACTION ROBES ═══
        new VendorItem
        {
            Name = "Faction Robe",
            Description = "A robe in your faction's color",
            Cost = 500,
            Category = "Cosmetics",
            ItemGenerator = (buyer) => new FactionRobe(buyer.Guild.Faction.Value)
        },
        
        // ═══ HAIR & BEARD DYES ═══
        new VendorItem
        {
            Name = "Faction Hair Dye",
            Description = "Dye your hair in faction colors",
            Cost = 250,
            Category = "Cosmetics",
            ItemGenerator = (buyer) => new FactionHairDye(buyer.Guild.Faction.Value)
        },
        new VendorItem
        {
            Name = "Faction Beard Dye",
            Description = "Dye your beard in faction colors",
            Cost = 250,
            Category = "Cosmetics",
            ItemGenerator = (buyer) => new FactionBeardDye(buyer.Guild.Faction.Value)
        },
        
        // ═══ HOUSE DECORATIONS ═══
        new VendorItem
        {
            Name = "Faction Banner (South)",
            Description = "A banner displaying your faction's colors",
            Cost = 1000,
            Category = "House Deco",
            ItemGenerator = (buyer) => new FactionBannerDeed(buyer.Guild.Faction.Value, Direction.South)
        },
        new VendorItem
        {
            Name = "Faction Banner (East)",
            Description = "A banner displaying your faction's colors",
            Cost = 1000,
            Category = "House Deco",
            ItemGenerator = (buyer) => new FactionBannerDeed(buyer.Guild.Faction.Value, Direction.East)
        },
        new VendorItem
        {
            Name = "PvP Statistics Board",
            Description = "Displays your PvP kill statistics",
            Cost = 2500,
            Category = "House Deco",
            ItemGenerator = (buyer) => new PvPStatsBoardDeed(buyer)
        },
        
        // ═══ SIEGE DEFENSES ═══
        // (Already defined in VvV_Siege_System.md, referenced here)
        new VendorItem
        {
            Name = "Alarm Trap Kit",
            Description = "Place during sieges to detect enemies",
            Cost = 100,
            Category = "Siege",
            ItemGenerator = (buyer) => new AlarmTrapKit()
        },
        new VendorItem
        {
            Name = "Snare Trap Kit",
            Description = "Place during sieges to slow enemies",
            Cost = 200,
            Category = "Siege",
            ItemGenerator = (buyer) => new SnareTrapKit()
        },
        new VendorItem
        {
            Name = "Smoke Trap Kit",
            Description = "Place during sieges to block line of sight",
            Cost = 300,
            Category = "Siege",
            ItemGenerator = (buyer) => new SmokeTrapKit()
        },
        new VendorItem
        {
            Name = "Mana Drain Trap Kit",
            Description = "Place during sieges to drain enemy mana",
            Cost = 250,
            Category = "Siege",
            ItemGenerator = (buyer) => new ManaDrainTrapKit()
        },
        new VendorItem
        {
            Name = "Arrow Turret Kit",
            Description = "Place during sieges for automated defense",
            Cost = 500,
            Category = "Siege",
            ItemGenerator = (buyer) => new ArrowTurretKit()
        },
        new VendorItem
        {
            Name = "Magic Turret Kit",
            Description = "Place during sieges for magical defense",
            Cost = 750,
            Category = "Siege",
            ItemGenerator = (buyer) => new MagicTurretKit()
        }
    };
}
```

### 4.3 Cosmetic Items

```csharp
// Faction Robe - colored to faction
public class FactionRobe : BaseOuterTorso
{
    private FactionId _faction;
    
    [Constructable]
    public FactionRobe(FactionId faction) : base(0x1F04) // Robe
    {
        _faction = faction;
        Name = $"{faction} Faction Robe";
        
        Hue = faction switch
        {
            FactionId.Vampire => 0x21,  // Red
            FactionId.Daemon => 0x30,   // Orange
            FactionId.Goblin => 0x3F,   // Green
            _ => 0
        };
        
        // Cannot be dyed
        Dyable = false;
        
        // Blessed (cannot be looted)
        LootType = LootType.Blessed;
    }
    
    public override bool CanEquip(Mobile m)
    {
        if (m is PlayerMobile pm)
        {
            if (pm.Guild?.Faction != _faction)
            {
                pm.SendMessage(0x22, "You must be a member of this faction to wear this robe.");
                return false;
            }
        }
        return base.CanEquip(m);
    }
}

// Faction Hair Dye
public class FactionHairDye : Item
{
    private FactionId _faction;
    
    [Constructable]
    public FactionHairDye(FactionId faction) : base(0xEFF)
    {
        _faction = faction;
        Name = $"{faction} Hair Dye";
        
        Hue = faction switch
        {
            FactionId.Vampire => 0x21,
            FactionId.Daemon => 0x30,
            FactionId.Goblin => 0x3F,
            _ => 0
        };
    }
    
    public override void OnDoubleClick(Mobile from)
    {
        if (!(from is PlayerMobile pm))
            return;
            
        if (pm.Guild?.Faction != _faction)
        {
            pm.SendMessage(0x22, "You must be a member of this faction to use this dye.");
            return;
        }
        
        pm.HairHue = Hue;
        pm.SendMessage(0x35, "Your hair has been dyed!");
        Delete();
    }
}

// Faction Beard Dye (similar to hair dye)
public class FactionBeardDye : Item
{
    private FactionId _faction;
    
    [Constructable]
    public FactionBeardDye(FactionId faction) : base(0xEFF)
    {
        _faction = faction;
        Name = $"{faction} Beard Dye";
        Hue = GetFactionHue(faction);
    }
    
    public override void OnDoubleClick(Mobile from)
    {
        if (!(from is PlayerMobile pm))
            return;
            
        if (pm.Guild?.Faction != _faction)
        {
            pm.SendMessage(0x22, "You must be a member of this faction to use this dye.");
            return;
        }
        
        pm.FacialHairHue = Hue;
        pm.SendMessage(0x35, "Your beard has been dyed!");
        Delete();
    }
}
```

### 4.4 House Decorations

```csharp
// Faction Banner Deed
public class FactionBannerDeed : Item
{
    private FactionId _faction;
    private Direction _direction;
    
    [Constructable]
    public FactionBannerDeed(FactionId faction, Direction dir) : base(0x14F0)
    {
        _faction = faction;
        _direction = dir;
        Name = $"{faction} Banner Deed ({dir})";
        Hue = GetFactionHue(faction);
    }
    
    public override void OnDoubleClick(Mobile from)
    {
        if (!(from is PlayerMobile pm))
            return;
            
        if (!BaseHouse.FindHouseAt(pm)?.IsOwner(pm) ?? true)
        {
            pm.SendMessage(0x22, "You must be in your house to place this.");
            return;
        }
        
        pm.SendMessage(0x35, "Target where you want to place the banner.");
        pm.Target = new BannerPlacementTarget(this, _faction, _direction);
    }
}

// PvP Statistics Board
public class PvPStatsBoardDeed : Item
{
    private Serial _ownerSerial;
    
    [Constructable]
    public PvPStatsBoardDeed(PlayerMobile owner) : base(0x14F0)
    {
        _ownerSerial = owner.Serial;
        Name = "PvP Statistics Board Deed";
    }
    
    public override void OnDoubleClick(Mobile from)
    {
        // Similar placement logic
    }
}

public class PvPStatsBoard : Item
{
    private Serial _ownerSerial;
    
    [Constructable]
    public PvPStatsBoard(Serial owner) : base(0x1E5E) // Board graphic
    {
        _ownerSerial = owner;
        Name = "PvP Statistics Board";
        Movable = false;
    }
    
    public override void OnDoubleClick(Mobile from)
    {
        var owner = World.FindMobile(_ownerSerial) as PlayerMobile;
        if (owner == null)
        {
            from.SendMessage(0x22, "The owner of this board no longer exists.");
            return;
        }
        
        from.SendGump(new PvPStatsBoardGump(owner));
    }
}

public class PvPStatsBoardGump : Gump
{
    public PvPStatsBoardGump(PlayerMobile player) : base(50, 50)
    {
        var stats = PvPStatisticsManager.GetStats(player);
        
        AddBackground(0, 0, 300, 300, 9200);
        AddLabel(100, 20, 0x35, "PvP Statistics");
        
        AddLabel(20, 60, 0, $"Player: {player.Name}");
        AddLabel(20, 80, 0, $"Faction: {player.Guild?.Faction?.ToString() ?? "None"}");
        
        AddLabel(20, 120, 0x35, "Combat Record:");
        AddLabel(20, 140, 0, $"Total Kills: {stats.TotalKills}");
        AddLabel(20, 160, 0, $"Total Deaths: {stats.TotalDeaths}");
        AddLabel(20, 180, 0, $"K/D Ratio: {stats.KDRatio:F2}");
        
        AddLabel(20, 220, 0x35, "Siege Record:");
        AddLabel(20, 240, 0, $"Sieges Won: {stats.SiegesWon}");
        AddLabel(20, 260, 0, $"Sigils Captured: {stats.SigilsCaptured}");
    }
}
```

### 4.5 Silver Vendor NPC

```csharp
public class SilverVendor : BaseVendor
{
    [Constructable]
    public SilverVendor() : base("the Silver Merchant")
    {
        Title = "Silver Merchant";
    }
    
    public override void OnDoubleClick(Mobile from)
    {
        if (!(from is PlayerMobile pm))
            return;
            
        if (pm.Guild?.Faction == null)
        {
            Say("I only deal with faction members. Join a faction first!");
            return;
        }
        
        pm.SendGump(new SilverVendorGump(pm));
    }
    
    public override void InitSBInfo() { } // No standard inventory
}

public class SilverVendorGump : Gump
{
    private PlayerMobile _player;
    
    public SilverVendorGump(PlayerMobile player) : base(50, 50)
    {
        _player = player;
        int silverBalance = player.BankBox?.GetAmount(typeof(Silver)) ?? 0;
        
        AddBackground(0, 0, 450, 500, 9200);
        AddLabel(160, 20, 0x35, "Silver Merchant");
        AddLabel(20, 45, 0, $"Your Silver: {silverBalance}");
        
        int y = 80;
        int buttonId = 1;
        string currentCategory = "";
        
        foreach (var item in SilverVendorInventory.Items)
        {
            // Category header
            if (item.Category != currentCategory)
            {
                currentCategory = item.Category;
                AddLabel(20, y, 0x35, $"═══ {currentCategory} ═══");
                y += 25;
            }
            
            // Item entry
            AddButton(20, y, 4005, 4007, buttonId, GumpButtonType.Reply, 0);
            AddLabel(55, y, silverBalance >= item.Cost ? 0 : 0x22, item.Name);
            AddLabel(250, y, 0, $"{item.Cost} silver");
            AddLabel(55, y + 15, 0x3B2, item.Description);
            
            y += 40;
            buttonId++;
        }
    }
    
    public override void OnResponse(NetState sender, RelayInfo info)
    {
        if (info.ButtonID < 1)
            return;
            
        int index = info.ButtonID - 1;
        if (index >= SilverVendorInventory.Items.Length)
            return;
            
        var item = SilverVendorInventory.Items[index];
        
        // Check silver
        if (!_player.BankBox.ConsumeTotal(typeof(Silver), item.Cost))
        {
            _player.SendMessage(0x22, $"You need {item.Cost} silver to purchase this.");
            _player.SendGump(new SilverVendorGump(_player));
            return;
        }
        
        // Generate and give item
        var purchasedItem = item.ItemGenerator(_player);
        _player.AddToBackpack(purchasedItem);
        
        _player.SendMessage(0x35, $"You purchased {item.Name} for {item.Cost} silver!");
        _player.PlaySound(0x5B5);
        
        // Refresh
        _player.SendGump(new SilverVendorGump(_player));
    }
}
```

### 4.6 Vendor Locations

Place Silver Vendors at:
- Each siege city (Jhelom, Skara Brae, Yew, Trinsic)
- Britain (faction headquarters area)

---

## 5. Configuration Reference

```csharp
public static class DailyContentConfig
{
    // ═══ BOUNTIES ═══
    public static TimeSpan BountyResetTime = TimeSpan.Zero; // Midnight
    public static string ServerTimezone = "Eastern Standard Time";
    
    // ═══ FACTION QUEST ═══
    public static TimeSpan BossRespawnDelay = TimeSpan.FromHours(2);
    public static int BossBaseHits = 5000;
    public static int BossHitsPerPlayer = 500;
    public static int BossMaxHits = 25000;
    public static int QuestFactionPointReward = 500;
    public static int QuestBonusForTopDamage = 250;
    
    // Relic drop chances
    public static double CommonRelicChance = 0.15;
    public static double UncommonRelicChance = 0.05;
    public static double RareRelicChance = 0.01;
    
    // ═══ SILVER VENDOR ═══
    public static int FactionRobeCost = 500;
    public static int HairDyeCost = 250;
    public static int BeardDyeCost = 250;
    public static int BannerCost = 1000;
    public static int PvPBoardCost = 2500;
}
```

---

## 6. Testing Checklist

### Daily Bounties
- [ ] Same tasks generated for all players on same day
- [ ] Tasks reset at server midnight
- [ ] Progress tracks correctly per task type
- [ ] Rewards (FP + Silver) awarded on claim
- [ ] Cannot claim twice
- [ ] Progress resets on new day
- [ ] Peaceful faction members can complete bounties

### Daily Faction Quest
- [ ] Location announced on new day
- [ ] Boss spawns at correct location
- [ ] Boss scales with nearby player count
- [ ] All participants get faction points
- [ ] Top damage gets bonus points
- [ ] Relic drops are chance-based
- [ ] Boss respawns after 2 hours
- [ ] Players can only get FP once per day

### Silver Vendor
- [ ] Only faction members can purchase
- [ ] Silver deducted correctly
- [ ] Faction robes match faction color
- [ ] Hair/beard dyes apply correct color
- [ ] Banners placeable in houses
- [ ] PvP board shows correct stats
- [ ] Trap/turret kits work correctly

---

## 7. Integration Points

| System | Integration |
|--------|-------------|
| **Faction Points** | Bounties and quest award points |
| **Silver Currency** | Bounties award silver, vendor spends silver |
| **VvV Siege** | Trap/turret kits purchased here |
| **Town Cryer** | Announces daily quest location |
| **Housing** | Banner and PvP board decorations |
| **PvP Statistics** | Board displays kill/death stats |

---

## 8. Change Log

### v1.0.0 - 2025-01-02 (Initial Specification)
- Daily Bounty System with 3 universal tasks
- Server midnight reset
- Daily Faction Quest with mini-boss
- 4 swamp/desert locations
- 2-hour boss respawn
- Chance-based relic drops
- Silver Vendor with faction cosmetics
- Faction Robes, Hair Dye, Beard Dye
- Faction Banners, PvP Statistics Board
- Trap/turret kit purchasing