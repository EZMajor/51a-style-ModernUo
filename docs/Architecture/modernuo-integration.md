# ModernUO Integration

Hook points, file paths, and extension patterns for 51alpha on ModernUO v24.0.0+.

**Principle**: Extend, don't modify core files. Use events, inheritance, and extension methods to minimize update conflicts.

## Directory Structure

Create under `Projects/UOContent/`:

```
Sphere51a/
├── Core/
│   ├── Sphere51aCore.cs      # Main initialization
│   ├── Config.cs             # Configuration loader
│   ├── Database.cs           # PostgreSQL connection
│   └── Sphere51aTimers.cs    # Timer management
├── Combat/
│   ├── SphereCombatSystem.cs # Sphere-style mechanics
│   ├── FizzleMechanics.cs    # Mana consumption on fizzle
│   └── KillAttribution.cs    # PvP kill tracking
├── Factions/
│   ├── FactionManager.cs
│   ├── FactionId.cs
│   └── CombatStatus.cs
├── Siege/
│   ├── SiegeManager.cs
│   ├── SiegeBattle.cs
│   └── TownControl.cs
├── Tournament/
│   ├── TournamentManager.cs
│   └── TournamentMatch.cs
├── Daily/
│   ├── BountyManager.cs
│   └── FactionQuestManager.cs
├── Currency/
│   ├── FactionPointManager.cs
│   └── SilverManager.cs
├── Glicko/
│   ├── GlickoManager.cs
│   └── GlickoCalculator.cs
├── NPE/
│   ├── YoungPlayerManager.cs
│   └── StarterQuests.cs
├── Gumps/
├── Items/
├── Mobiles/
├── Commands/
└── Telemetry/
```

---

## Initialization

**Modify**: `Projects/UOContent/Initialization.cs`

Add at end of initialization:
```csharp
Sphere51a.Core.Sphere51aCore.Initialize();
```

**Create**: `Sphere51a/Core/Sphere51aCore.cs`

```csharp
namespace Sphere51a.Core;

public static class Sphere51aCore
{
    public static void Initialize()
    {
        Console.WriteLine("[51alpha] Initializing...");
        
        Database.Initialize();
        Config.Load();
        
        // Register events
        EventSink.PlayerDeath += Combat.KillAttribution.OnPlayerDeath;
        EventSink.CreatureDeath += Daily.BountyManager.OnCreatureDeath;
        EventSink.Login += OnPlayerLogin;
        
        // Initialize subsystems
        Factions.FactionManager.Initialize();
        Currency.FactionPointManager.Initialize();
        Glicko.GlickoManager.Initialize();
        Siege.SiegeManager.Initialize();
        Tournament.TournamentManager.Initialize();
        Daily.BountyManager.Initialize();
        NPE.YoungPlayerManager.Initialize();
        
        Sphere51aTimers.Initialize();
        
        Console.WriteLine("[51alpha] Ready.");
    }
    
    private static void OnPlayerLogin(LoginEventArgs e)
    {
        if (e.Mobile is PlayerMobile pm)
        {
            NPE.YoungPlayerManager.CheckYoungStatus(pm);
            Factions.FactionManager.OnPlayerLogin(pm);
        }
    }
}
```

---

## Spell System Hooks

**File**: `Projects/UOContent/Spells/Base/Spell.cs`

### Free Movement (Sphere-style)

```csharp
public virtual bool OnCasterMoving(Direction d)
{
    // 51alpha: Always allow movement during casting
    if (Sphere51a.Core.Config.GetBool("sphere.enabled", true))
        return true;
    
    // Original OSI logic
    if (IsCasting && BlocksMovement)
    {
        Caster.SendLocalizedMessage(500111);
        return false;
    }
    return true;
}
```

### Option C Mana Consumption

Modify `CheckSequence()`:

```csharp
public virtual bool CheckSequence()
{
    var mana = ScaleMana(GetMana());
    
    // Reagents consumed first (unchanged)
    if (!ConsumeReagents())
        return false;
    
    // Mana validation only
    if (Caster.Mana < mana)
    {
        Caster.LocalOverheadMessage(MessageType.Regular, 0x22, 502625);
        return false;
    }
    
    // === Option C: Mana at completion ===
    if (CheckFizzle())
    {
        // Success: 100% mana
        Caster.Mana -= mana;
        // ... scroll consumption, karma, etc
        return true;
    }
    else
    {
        // Fizzle: Partial mana (configurable)
        double rate = Sphere51a.Core.Config.GetDouble("combat.fizzle_mana_rate", 0.5);
        int fizzleMana = (int)(mana * rate);
        if (fizzleMana > 0)
            Caster.Mana -= fizzleMana;
        
        DoFizzle();
        return false;
    }
}
```

### Protection Spell FC Penalty

Modify `GetCastDelay()`:

```csharp
if (ProtectionSpell.Registry.ContainsKey(Caster))
{
    int penalty = Sphere51a.Core.Config.GetInt("combat.protection_fc_penalty", 0);
    fc -= penalty;  // Default 0 = no penalty
}
```

---

## Faster Casting Removal

Remove FC from loot generation:

**File**: `Projects/UOContent/Items/Tools/BaseRunicTool.cs`

Find `AosAttribute.CastSpeed` cases in attribute generation and remove or replace with `AosAttribute.RegenMana`.

**Files to modify**:
- `BaseRunicTool.cs` - Random attribute generation
- `Misc/RandomItemGenerator.cs` - Loot pools
- `Misc/LootPack.cs` - Loot tables
- `Engines/Craft/Def*.cs` - Crafting bonuses

**Admin cleanup command**:

```csharp
[Usage("[RemoveAllFC")]
[Description("Removes Faster Casting from all items")]
private static void RemoveAllFC_OnCommand(CommandEventArgs e)
{
    int count = 0;
    foreach (Item item in World.Items.Values)
    {
        if (item is BaseWeapon w && w.Attributes[AosAttribute.CastSpeed] != 0)
        {
            w.Attributes[AosAttribute.CastSpeed] = 0;
            count++;
        }
        else if (item is BaseArmor a && a.Attributes[AosAttribute.CastSpeed] != 0)
        {
            a.Attributes[AosAttribute.CastSpeed] = 0;
            count++;
        }
        else if (item is BaseJewel j && j.Attributes[AosAttribute.CastSpeed] != 0)
        {
            j.Attributes[AosAttribute.CastSpeed] = 0;
            count++;
        }
    }
    e.Mobile.SendMessage($"Removed FC from {count} items.");
}
```

---

## Combat Tracking

**File**: `Projects/UOContent/Mobiles/PlayerMobile.cs`

```csharp
public override void OnDeath(Container c)
{
    if (LastKiller is PlayerMobile killer)
    {
        Sphere51a.Combat.KillAttribution.ProcessKill(killer, this);
    }
    base.OnDeath(c);
}
```

**File**: `Projects/UOContent/Misc/AOS.cs`

Add damage tracking:

```csharp
public static int Damage(Mobile m, Mobile from, int damage, ...)
{
    // ... existing calculation ...
    
    // 51alpha: Track for attribution
    if (from is PlayerMobile && m is PlayerMobile)
    {
        Sphere51a.Combat.KillAttribution.RecordDamage(from, m, damage);
    }
    
    return damage;
}
```

---

## Guild Faction Extension

**Create**: `Sphere51a/Factions/GuildExtension.cs`

```csharp
namespace Sphere51a.Factions;

public static class GuildExtension
{
    public static FactionId? GetFaction(this Guild guild)
        => FactionManager.GetGuildFaction(guild);
    
    public static void SetFaction(this Guild guild, FactionId? faction)
        => FactionManager.SetGuildFaction(guild, faction);
    
    public static bool IsInFaction(this Guild guild)
        => GetFaction(guild).HasValue;
}
```

---

## Resource Gathering Hooks

**File**: `Projects/UOContent/Engines/Harvest/Mining.cs`

```csharp
public override void OnHarvest(Mobile from, Item tool, HarvestResource resource, ...)
{
    // ... existing logic ...
    
    if (from is PlayerMobile pm)
    {
        Sphere51a.Daily.BountyManager.OnResourceGathered(pm, resource.Types[0], amount);
    }
}
```

Same pattern for `Lumberjacking.cs` and `Fishing.cs`.

---

## Timer Management

**Create**: `Sphere51a/Core/Sphere51aTimers.cs`

```csharp
namespace Sphere51a.Core;

public static class Sphere51aTimers
{
    public static void Initialize()
    {
        // Daily reset at 5 AM UTC (midnight EST)
        var nextReset = CalculateNextReset();
        Timer.DelayCall(nextReset, TimeSpan.FromDays(1), OnDailyReset);
        
        // Siege tick every second
        Timer.DelayCall(TimeSpan.FromSeconds(1), TimeSpan.FromSeconds(1),
            Siege.SiegeManager.OnTick);
        
        // Tournament check every minute
        Timer.DelayCall(TimeSpan.FromMinutes(1), TimeSpan.FromMinutes(1),
            Tournament.TournamentManager.CheckSchedule);
    }
    
    private static TimeSpan CalculateNextReset()
    {
        var now = DateTime.UtcNow;
        var resetHour = 5; // 5 AM UTC
        var nextReset = now.Date.AddHours(resetHour);
        if (nextReset <= now) nextReset = nextReset.AddDays(1);
        return nextReset - now;
    }
    
    private static void OnDailyReset()
    {
        Console.WriteLine("[51alpha] Daily reset");
        Daily.BountyManager.GenerateDailyBounties();
        Daily.FactionQuestManager.SelectDailyLocation();
    }
}
```

---

## Database Connection

**Add NuGet**: `Projects/UOContent/UOContent.csproj`

```xml
<PackageReference Include="Npgsql" Version="8.0.1" />
```

**Create**: `Sphere51a/Core/Database.cs`

```csharp
using Npgsql;

namespace Sphere51a.Core;

public static class Database
{
    private static NpgsqlDataSource _dataSource;
    
    public static void Initialize()
    {
        var connStr = Config.GetString("database.connection_string");
        if (string.IsNullOrEmpty(connStr))
        {
            Console.WriteLine("[51alpha] WARNING: No database configured");
            return;
        }
        
        _dataSource = new NpgsqlDataSourceBuilder(connStr).Build();
        
        using var conn = _dataSource.OpenConnection();
        Console.WriteLine("[51alpha] Database connected");
    }
    
    public static NpgsqlConnection GetConnection() => _dataSource?.OpenConnection();
    
    public static async Task ExecuteAsync(string sql, params NpgsqlParameter[] parameters)
    {
        await using var conn = await _dataSource.OpenConnectionAsync();
        await using var cmd = new NpgsqlCommand(sql, conn);
        cmd.Parameters.AddRange(parameters);
        await cmd.ExecuteNonQueryAsync();
    }
}
```

---

## Key EventSink Events

ModernUO events to hook:

| Event | Usage |
|-------|-------|
| `PlayerDeath` | Kill attribution, Glicko updates |
| `CreatureDeath` | Bounty tracking |
| `Login` | Young status, faction loading |
| `Logout` | Tournament/siege cleanup |
| `WorldSave` | Flush pending DB operations |
| `Movement` | PvP context detection |

Events that may need creation:
- `ResourceGather` - Bounty resource tracking
- `CraftItem` - Bounty craft tracking
- `SpellCast` - Talisman disable trigger

---

## Configuration File

**Create**: `Data/51alpha/config.json`

```json
{
  "sphere.enabled": true,
  "database.connection_string": "Host=localhost;Database=51alpha;Username=51alpha_app;Password=secure",
  
  "combat.fizzle_mana_rate": 0.5,
  "combat.protection_fc_penalty": 0,
  "combat.talisman_pvp_disable_minutes": 5,
  
  "faction.change_cooldown_days": 7,
  "young.duration_days": 14,
  
  "siege.duration_minutes": 30,
  "siege.victory_score": 10000,
  
  "tournament.min_players": 6,
  "tournament.match_duration_seconds": 300
}
```

---

## Verified Code References

These ModernUO code flows have been verified against GitHub:

| File | Key Method | Notes |
|------|------------|-------|
| `Spell.cs` | `CheckSequence()` | Reagents consumed first, mana after fizzle check |
| `Spell.cs` | `GetCastDelay()` | FC from `AosAttributes.GetValue()`, Protection subtracts |
| `Spell.cs` | `OnCasterMoving()` | Return true to allow, false to block |
| `SpellState.cs` | - | 3 states: None, Casting, Sequencing |
| `DisturbType` | - | 5 types: Hurt, EquipRequest, UseRequest, NewCast, Kill |
