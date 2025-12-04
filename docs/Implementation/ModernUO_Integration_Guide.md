# ModernUO Integration Guide
## Exact File Paths & Hook Points for 51alpha

### Document Metadata
- **Version**: v1.0.0
- **Last Updated**: 2025-01-02
- **ModernUO Version**: v24.0.0+ (clean install)
- **Repository**: https://github.com/modernuo/ModernUO

---

## 1. Overview

This document specifies exactly WHERE to add 51alpha code in the ModernUO codebase. All paths are relative to the ModernUO root directory.

### Key Principle
**Extend, don't modify core files** wherever possible. Use events, inheritance, and partial classes to minimize conflicts with ModernUO updates.

---

## 2. Directory Structure

### 2.1 Create These New Directories

```
Projects/
└── UOContent/
    └── Sphere51a/                    # NEW - All 51alpha code here
        ├── Core/
        │   ├── Sphere51aCore.cs      # Main initialization
        │   ├── Config.cs             # Configuration loader
        │   └── Database.cs           # PostgreSQL connection
        ├── Factions/
        │   ├── FactionManager.cs
        │   ├── FactionId.cs
        │   ├── GuildFactionExtension.cs
        │   └── CombatStatus.cs
        ├── Siege/
        │   ├── SiegeManager.cs
        │   ├── SiegeBattle.cs
        │   ├── SiegeScoring.cs
        │   ├── TownControl.cs
        │   └── Objectives/
        │       ├── SiegeSigil.cs
        │       ├── SiegeAltar.cs
        │       └── SiegePriest.cs
        ├── Traps/
        │   ├── BaseSiegeTrap.cs
        │   ├── AlarmTrap.cs
        │   ├── SnareTrap.cs
        │   ├── SmokeTrap.cs
        │   └── ManaDrainTrap.cs
        ├── Turrets/
        │   ├── BaseSiegeTurret.cs
        │   ├── ArrowTurret.cs
        │   └── MagicTurret.cs
        ├── Tournament/
        │   ├── TournamentManager.cs
        │   ├── TournamentBracket.cs
        │   ├── TournamentMatch.cs
        │   └── TournamentArena.cs
        ├── Daily/
        │   ├── BountyManager.cs
        │   ├── BountyBoard.cs
        │   ├── FactionQuestManager.cs
        │   ├── FactionQuestBoss.cs
        │   └── SilverVendor.cs
        ├── Currency/
        │   ├── FactionPointManager.cs
        │   ├── SilverManager.cs
        │   └── TournamentCoinManager.cs
        ├── Glicko/
        │   ├── GlickoManager.cs
        │   ├── GlickoCalculator.cs
        │   └── GlickoRating.cs
        ├── NPE/
        │   ├── YoungPlayerManager.cs
        │   ├── StarterQuests.cs
        │   └── TrainingIsland.cs
        ├── Combat/
        │   ├── SphereCombatSystem.cs
        │   ├── FizzleMechanics.cs
        │   ├── TalismanDisable.cs
        │   └── KillAttribution.cs
        ├── Gumps/
        │   ├── SiegeStatusGump.cs
        │   ├── TournamentGump.cs
        │   ├── BountyBoardGump.cs
        │   ├── SilverVendorGump.cs
        │   ├── FactionStatusGump.cs
        │   └── GlickoLeaderboardGump.cs
        ├── Items/
        │   ├── Cosmetics/
        │   │   ├── FactionRobe.cs
        │   │   ├── FactionHairDye.cs
        │   │   └── FactionBeardDye.cs
        │   ├── Decorations/
        │   │   ├── FactionBanner.cs
        │   │   └── PvPStatsBoard.cs
        │   └── TrapKits/
        │       ├── AlarmTrapKit.cs
        │       ├── SnareTrapKit.cs
        │       ├── SmokeTrapKit.cs
        │       └── ManaDrainTrapKit.cs
        ├── Mobiles/
        │   ├── TournamentRegistrar.cs
        │   ├── SilverVendorNPC.cs
        │   ├── SiegePriestNPC.cs
        │   ├── BountyBoardNPC.cs
        │   └── TownCryer.cs
        ├── Commands/
        │   ├── SiegeCommands.cs
        │   ├── TournamentCommands.cs
        │   ├── FactionCommands.cs
        │   ├── BountyCommands.cs
        │   └── AdminCommands.cs
        ├── Locations/
        │   ├── SiegeLocations.cs
        │   ├── TournamentLocations.cs
        │   ├── FactionQuestLocations.cs
        │   └── NPELocations.cs
        └── Telemetry/
            ├── EventLogger.cs
            └── MetricsCollector.cs
```

---

## 3. Core ModernUO Integration Points

### 3.1 Initialization Hook

**File**: `Projects/UOContent/Initialization.cs`

Add to the initialization sequence:

```csharp
// In Initialization.cs, find the Configure() method or similar entry point
// Add at the end of initialization:

// === 51alpha Initialization ===
Sphere51a.Core.Sphere51aCore.Initialize();
```

**Create**: `Projects/UOContent/Sphere51a/Core/Sphere51aCore.cs`

```csharp
namespace Sphere51a.Core
{
    public static class Sphere51aCore
    {
        public static void Initialize()
        {
            Console.WriteLine("[51alpha] Initializing systems...");
            
            // Initialize in dependency order
            Database.Initialize();
            Config.Load();
            
            // Register event handlers
            RegisterEventHandlers();
            
            // Initialize subsystems
            Factions.FactionManager.Initialize();
            Currency.FactionPointManager.Initialize();
            Currency.SilverManager.Initialize();
            Glicko.GlickoManager.Initialize();
            Siege.SiegeManager.Initialize();
            Tournament.TournamentManager.Initialize();
            Daily.BountyManager.Initialize();
            Daily.FactionQuestManager.Initialize();
            NPE.YoungPlayerManager.Initialize();
            
            // Register commands
            Commands.SiegeCommands.Register();
            Commands.TournamentCommands.Register();
            Commands.FactionCommands.Register();
            Commands.BountyCommands.Register();
            Commands.AdminCommands.Register();
            
            Console.WriteLine("[51alpha] Initialization complete.");
        }
        
        private static void RegisterEventHandlers()
        {
            EventSink.PlayerDeath += Combat.KillAttribution.OnPlayerDeath;
            EventSink.CreatureDeath += Daily.BountyManager.OnCreatureDeath;
            EventSink.Login += OnPlayerLogin;
            EventSink.Logout += OnPlayerLogout;
            EventSink.WorldSave += OnWorldSave;
        }
        
        private static void OnPlayerLogin(LoginEventArgs e)
        {
            if (e.Mobile is PlayerMobile pm)
            {
                NPE.YoungPlayerManager.CheckYoungStatus(pm);
                Factions.FactionManager.OnPlayerLogin(pm);
            }
        }
        
        private static void OnPlayerLogout(LogoutEventArgs e)
        {
            if (e.Mobile is PlayerMobile pm)
            {
                Tournament.TournamentManager.OnPlayerLogout(pm);
                Siege.SiegeManager.OnPlayerLogout(pm);
            }
        }
        
        private static void OnWorldSave(WorldSaveEventArgs e)
        {
            // Flush any pending database operations
            Database.FlushPending();
        }
    }
}
```

---

### 3.2 Event Sink Extensions

**File**: `Projects/Server/Events/EventSink.cs`

ModernUO has many events. Key ones we need (verify these exist or add):

```csharp
// These should exist in EventSink.cs - verify names
public static event Action<PlayerDeathEventArgs> PlayerDeath;
public static event Action<CreatureDeathEventArgs> CreatureDeath;
public static event Action<LoginEventArgs> Login;
public static event Action<LogoutEventArgs> Logout;
public static event Action<WorldSaveEventArgs> WorldSave;
public static event Action<MovementEventArgs> Movement;

// We may need to ADD these if they don't exist:
public static event Action<ResourceGatherEventArgs> ResourceGather;
public static event Action<CraftItemEventArgs> CraftItem;
public static event Action<SpellCastEventArgs> SpellCast;
public static event Action<SpellFizzleEventArgs> SpellFizzle;
```

If events don't exist, add hooks in the appropriate locations (see section 3.3).

---

### 3.3 Combat System Hooks

**File**: `Projects/UOContent/Spells/Base/Spell.cs`

For spell fizzle mechanics:

```csharp
// Find the method that handles spell interruption/fizzle
// Usually something like OnInterrupt() or Disturb()

public virtual void Disturb(DisturbType type)
{
    // ... existing code ...
    
    // === 51alpha Hook: Fizzle consumes resources ===
    if (type == DisturbType.Damage || type == DisturbType.Movement)
    {
        Sphere51a.Combat.FizzleMechanics.OnSpellFizzle(Caster, this);
    }
}
```

**File**: `Projects/UOContent/Misc/AOS.cs` (or damage calculation location)

For damage/kill tracking:

```csharp
// Find where player damage is calculated
// Add hook for damage tracking:

public static int Damage(Mobile m, Mobile from, int damage, ...)
{
    // ... existing damage calculation ...
    
    // === 51alpha Hook: Track damage for attribution ===
    if (from is PlayerMobile && m is PlayerMobile)
    {
        Sphere51a.Combat.KillAttribution.RecordDamage(from, m, damage);
    }
    
    return damage;
}
```

**File**: `Projects/UOContent/Mobiles/PlayerMobile.cs`

For death handling:

```csharp
public override void OnDeath(Container c)
{
    // === 51alpha Hook: Process PvP kill ===
    if (LastKiller is PlayerMobile killer)
    {
        Sphere51a.Combat.KillAttribution.ProcessKill(killer, this);
    }
    
    base.OnDeath(c);
}
```

---

### 3.4 Guild System Extension

**File**: `Projects/UOContent/Guilds/Guild.cs`

Add faction property to guilds:

```csharp
public class Guild : BaseGuild
{
    // ... existing properties ...
    
    // === 51alpha: Faction membership ===
    private FactionId? _faction;
    
    [CommandProperty(AccessLevel.GameMaster)]
    public FactionId? Faction
    {
        get => Sphere51a.Factions.FactionManager.GetGuildFaction(this);
        set => Sphere51a.Factions.FactionManager.SetGuildFaction(this, value);
    }
    
    // === 51alpha: Check if guild is in faction warfare ===
    public bool IsInFaction => Faction.HasValue;
}
```

Alternative approach using extension methods (no core modification):

**Create**: `Projects/UOContent/Sphere51a/Factions/GuildFactionExtension.cs`

```csharp
namespace Sphere51a.Factions
{
    public static class GuildFactionExtension
    {
        // Cache faction lookups
        private static readonly Dictionary<Serial, FactionId?> _guildFactions = new();
        
        public static FactionId? GetFaction(this Guild guild)
        {
            return FactionManager.GetGuildFaction(guild);
        }
        
        public static void SetFaction(this Guild guild, FactionId? faction)
        {
            FactionManager.SetGuildFaction(guild, faction);
        }
        
        public static bool IsInFaction(this Guild guild)
        {
            return GetFaction(guild).HasValue;
        }
    }
}
```

---

### 3.5 Resource Gathering Hooks

**File**: `Projects/UOContent/Engines/Harvest/Mining.cs`

```csharp
// In the harvest success method:
public override void OnHarvest(Mobile from, Item tool, HarvestResource resource, ...)
{
    // ... existing harvest logic ...
    
    // === 51alpha Hook: Track for bounties ===
    if (from is PlayerMobile pm)
    {
        Sphere51a.Daily.BountyManager.OnResourceGathered(pm, resource.Types[0], amount);
    }
}
```

**File**: `Projects/UOContent/Engines/Harvest/Lumberjacking.cs`
(Same pattern as Mining)

---

### 3.6 Crafting Hooks

**File**: `Projects/UOContent/Engines/Craft/DefBlacksmithy.cs` (and other craft defs)

```csharp
// In the craft success method, or in CraftItem.cs:
public override void PlayCraftEffect(Mobile from)
{
    // ... existing code ...
    
    // === 51alpha Hook: Track for bounties ===
    if (from is PlayerMobile pm)
    {
        Sphere51a.Daily.BountyManager.OnItemCrafted(pm, ItemType, Amount);
    }
}
```

Alternative - hook into `CraftSystem.cs`:

**File**: `Projects/UOContent/Engines/Craft/Core/CraftSystem.cs`

```csharp
// Find CreateItem or similar method
public void CreateItem(Mobile from, Type itemType, ...)
{
    // ... existing creation logic ...
    
    // === 51alpha Hook ===
    Sphere51a.Daily.BountyManager.OnItemCrafted(from as PlayerMobile, itemType, 1);
}
```

---

### 3.7 Monster Death Hooks

**File**: `Projects/UOContent/Mobiles/BaseCreature.cs`

```csharp
public override void OnDeath(Container c)
{
    // ... existing death logic ...
    
    // === 51alpha Hook: Track for bounties ===
    var topDamager = GetTopDamager();
    if (topDamager is PlayerMobile pm)
    {
        Sphere51a.Daily.BountyManager.OnCreatureKilled(pm, this);
    }
    
    base.OnDeath(c);
}

// May need to add or find this method:
private Mobile GetTopDamager()
{
    // Return the mobile that did the most damage
    // ModernUO may have DamageStore or similar
    return null; // Implement based on existing damage tracking
}
```

---

### 3.8 Talisman System Hooks

**File**: `Projects/UOContent/Items/Equipment/Talismans/BaseTalisman.cs`

```csharp
public override void GetProperties(ObjectPropertyList list)
{
    base.GetProperties(list);
    
    // === 51alpha: Check if disabled in PvP ===
    if (Parent is Mobile m && Sphere51a.Combat.TalismanDisable.IsDisabled(m))
    {
        list.Add(1060847, "PvP Disabled"); // Or custom cliloc
    }
}

// Override bonus application methods:
public virtual int GetBonus(Mobile from, BonusType type)
{
    // === 51alpha: Return 0 if PvP disabled ===
    if (Sphere51a.Combat.TalismanDisable.IsDisabled(from))
        return 0;
        
    return base.GetBonus(from, type);
}
```

---

## 4. Command Registration

**File**: `Projects/UOContent/Sphere51a/Commands/SiegeCommands.cs`

```csharp
namespace Sphere51a.Commands
{
    public static class SiegeCommands
    {
        public static void Register()
        {
            CommandSystem.Register("siege", AccessLevel.GameMaster, Siege_OnCommand);
        }
        
        [Usage("siege <start|stop|status|sigil|altar|toggles> [args]")]
        [Description("Manages siege warfare system")]
        private static void Siege_OnCommand(CommandEventArgs e)
        {
            if (e.Arguments.Length == 0)
            {
                ShowUsage(e.Mobile);
                return;
            }
            
            switch (e.Arguments[0].ToLower())
            {
                case "start":
                    HandleStart(e);
                    break;
                case "stop":
                    HandleStop(e);
                    break;
                case "status":
                    HandleStatus(e);
                    break;
                case "sigil":
                    HandleSigil(e);
                    break;
                case "altar":
                    HandleAltar(e);
                    break;
                case "toggles":
                    HandleToggles(e);
                    break;
                default:
                    ShowUsage(e.Mobile);
                    break;
            }
        }
        
        // ... implement handlers ...
    }
}
```

---

## 5. Timer Integration

**File**: `Projects/UOContent/Sphere51a/Core/Sphere51aTimers.cs`

```csharp
namespace Sphere51a.Core
{
    public static class Sphere51aTimers
    {
        private static Timer _dailyResetTimer;
        private static Timer _tournamentCheckTimer;
        private static Timer _siegeTickTimer;
        private static Timer _questBossCheckTimer;
        
        public static void Initialize()
        {
            // Daily reset at midnight EST (5 AM UTC)
            var nextReset = CalculateNextReset();
            _dailyResetTimer = Timer.DelayCall(nextReset, TimeSpan.FromDays(1), OnDailyReset);
            
            // Tournament schedule check every minute
            _tournamentCheckTimer = Timer.DelayCall(TimeSpan.FromMinutes(1), TimeSpan.FromMinutes(1), 
                Tournament.TournamentManager.CheckSchedule);
            
            // Siege tick every second during active sieges
            _siegeTickTimer = Timer.DelayCall(TimeSpan.FromSeconds(1), TimeSpan.FromSeconds(1),
                Siege.SiegeManager.OnTick);
            
            // Quest boss check every minute
            _questBossCheckTimer = Timer.DelayCall(TimeSpan.FromMinutes(1), TimeSpan.FromMinutes(1),
                Daily.FactionQuestManager.CheckQuestSpawn);
        }
        
        private static TimeSpan CalculateNextReset()
        {
            var now = DateTime.UtcNow;
            var resetHour = Config.GetInt("bounty.reset_hour_utc", 5);
            var nextReset = now.Date.AddHours(resetHour);
            
            if (nextReset <= now)
                nextReset = nextReset.AddDays(1);
                
            return nextReset - now;
        }
        
        private static void OnDailyReset()
        {
            Console.WriteLine("[51alpha] Daily reset triggered");
            Daily.BountyManager.GenerateDailyBounties();
            Daily.FactionQuestManager.SelectDailyLocation();
        }
    }
}
```

---

## 6. Database Connection

**File**: `Projects/UOContent/Sphere51a/Core/Database.cs`

```csharp
using Npgsql;

namespace Sphere51a.Core
{
    public static class Database
    {
        private static string _connectionString;
        private static NpgsqlDataSource _dataSource;
        
        public static void Initialize()
        {
            _connectionString = Config.GetString("database.connection_string");
            
            if (string.IsNullOrEmpty(_connectionString))
            {
                Console.WriteLine("[51alpha] WARNING: No database connection string configured");
                return;
            }
            
            var builder = new NpgsqlDataSourceBuilder(_connectionString);
            _dataSource = builder.Build();
            
            // Test connection
            using var conn = _dataSource.OpenConnection();
            Console.WriteLine("[51alpha] Database connected successfully");
        }
        
        public static NpgsqlConnection GetConnection()
        {
            return _dataSource?.OpenConnection();
        }
        
        public static async Task<NpgsqlConnection> GetConnectionAsync()
        {
            return await _dataSource?.OpenConnectionAsync();
        }
        
        public static void FlushPending()
        {
            // Flush any batched operations
        }
        
        // Convenience methods
        public static async Task ExecuteAsync(string sql, params NpgsqlParameter[] parameters)
        {
            await using var conn = await GetConnectionAsync();
            await using var cmd = new NpgsqlCommand(sql, conn);
            cmd.Parameters.AddRange(parameters);
            await cmd.ExecuteNonQueryAsync();
        }
        
        public static async Task<T> ExecuteScalarAsync<T>(string sql, params NpgsqlParameter[] parameters)
        {
            await using var conn = await GetConnectionAsync();
            await using var cmd = new NpgsqlCommand(sql, conn);
            cmd.Parameters.AddRange(parameters);
            var result = await cmd.ExecuteScalarAsync();
            return (T)Convert.ChangeType(result, typeof(T));
        }
    }
}
```

**NuGet Package Required**: Add to `Projects/UOContent/UOContent.csproj`:

```xml
<ItemGroup>
    <PackageReference Include="Npgsql" Version="8.0.1" />
</ItemGroup>
```

---

## 7. Configuration Loading

**File**: `Projects/UOContent/Sphere51a/Core/Config.cs`

```csharp
using System.Text.Json;

namespace Sphere51a.Core
{
    public static class Config
    {
        private static Dictionary<string, string> _values = new();
        private static readonly string ConfigPath = "Data/51alpha/config.json";
        
        public static void Load()
        {
            // Load from file
            if (File.Exists(ConfigPath))
            {
                var json = File.ReadAllText(ConfigPath);
                _values = JsonSerializer.Deserialize<Dictionary<string, string>>(json) ?? new();
                Console.WriteLine($"[51alpha] Loaded {_values.Count} config values from file");
            }
            
            // Override with database values
            LoadFromDatabase();
        }
        
        private static void LoadFromDatabase()
        {
            try
            {
                using var conn = Database.GetConnection();
                if (conn == null) return;
                
                using var cmd = new NpgsqlCommand(
                    "SELECT config_key, config_value FROM s51a_config", conn);
                using var reader = cmd.ExecuteReader();
                
                int count = 0;
                while (reader.Read())
                {
                    _values[reader.GetString(0)] = reader.GetString(1);
                    count++;
                }
                
                Console.WriteLine($"[51alpha] Loaded {count} config values from database");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"[51alpha] Warning: Could not load config from database: {ex.Message}");
            }
        }
        
        // Reload from database (for live updates)
        public static void Reload()
        {
            LoadFromDatabase();
        }
        
        public static string GetString(string key, string defaultValue = "")
        {
            return _values.TryGetValue(key, out var value) ? value : defaultValue;
        }
        
        public static int GetInt(string key, int defaultValue = 0)
        {
            if (_values.TryGetValue(key, out var value) && int.TryParse(value, out var result))
                return result;
            return defaultValue;
        }
        
        public static decimal GetDecimal(string key, decimal defaultValue = 0)
        {
            if (_values.TryGetValue(key, out var value) && decimal.TryParse(value, out var result))
                return result;
            return defaultValue;
        }
        
        public static bool GetBool(string key, bool defaultValue = false)
        {
            if (_values.TryGetValue(key, out var value) && bool.TryParse(value, out var result))
                return result;
            return defaultValue;
        }
    }
}
```

---

## 8. Region System Integration

**File**: `Projects/UOContent/Sphere51a/Siege/SiegeRegion.cs`

```csharp
namespace Sphere51a.Siege
{
    public class SiegeRegion : BaseRegion
    {
        public string CityName { get; }
        public SiegeBattle ActiveBattle { get; set; }
        
        public SiegeRegion(string cityName, Map map, Rectangle2D area) 
            : base($"Siege_{cityName}", map, DefaultPriority, area)
        {
            CityName = cityName;
        }
        
        public override void OnEnter(Mobile m)
        {
            base.OnEnter(m);
            
            if (m is PlayerMobile pm && ActiveBattle != null)
            {
                // Auto-flag peaceful players entering siege
                var status = Factions.CombatStatus.GetStatus(pm);
                if (!status.IsCombatant)
                {
                    pm.SendMessage(0x22, "You have entered a siege zone and are now flagged for combat!");
                }
                
                SiegeManager.OnPlayerEnterSiege(pm, this);
            }
        }
        
        public override void OnExit(Mobile m)
        {
            base.OnExit(m);
            
            if (m is PlayerMobile pm && ActiveBattle != null)
            {
                SiegeManager.OnPlayerExitSiege(pm, this);
            }
        }
        
        public override bool OnMoveInto(Mobile m, Direction d, Point3D newLocation, Point3D oldLocation)
        {
            // Could restrict non-faction members during siege
            return base.OnMoveInto(m, d, newLocation, oldLocation);
        }
    }
}
```

---

## 9. File List Summary

### Core Files to CREATE (in Sphere51a folder):

| Path | Purpose |
|------|---------|
| `Core/Sphere51aCore.cs` | Main initialization |
| `Core/Database.cs` | PostgreSQL connection |
| `Core/Config.cs` | Configuration loader |
| `Core/Sphere51aTimers.cs` | Scheduled tasks |

### Core ModernUO Files to MODIFY:

| Path | Modification |
|------|--------------|
| `Initialization.cs` | Add Sphere51aCore.Initialize() call |
| `Mobiles/PlayerMobile.cs` | Add OnDeath hook |
| `Mobiles/BaseCreature.cs` | Add OnDeath hook for bounties |
| `Guilds/Guild.cs` | Add Faction property (or use extension) |
| `Spells/Base/Spell.cs` | Add fizzle hook |
| `Engines/Harvest/Mining.cs` | Add resource gather hook |
| `Engines/Harvest/Lumberjacking.cs` | Add resource gather hook |
| `Engines/Craft/Core/CraftSystem.cs` | Add crafting hook |

### NuGet Packages to ADD:

```xml
<PackageReference Include="Npgsql" Version="8.0.1" />
```

---

## 10. Build Verification

After adding all files:

```bash
cd Projects/UOContent
dotnet build
```

Expected: No errors, warnings reviewed.

Test initialization:

```bash
dotnet run
# Should see:
# [51alpha] Initializing systems...
# [51alpha] Database connected successfully
# [51alpha] Loaded X config values from database
# [51alpha] Initialization complete.
```

---

## Change Log

### v1.0.0 - 2025-01-02 (Initial Integration Guide)
- Complete directory structure
- All hook points identified
- Database connection setup
- Configuration system
- Timer integration
- Region system integration
- Build verification steps