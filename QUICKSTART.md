# Developer Quickstart

Get 51alpha running in under an hour.

## Prerequisites

- [x] .NET 10 SDK installed
- [x] PostgreSQL 15+ running
- [x] ModernUO v24.0.0 cloned
- [x] IDE (VS Code, Rider, or Visual Studio)

## Step 1: Database (10 minutes)

```bash
# Create database
psql -U postgres
CREATE DATABASE "51alpha";
CREATE USER "51alpha_app" WITH PASSWORD 'dev_password_change_me';
GRANT ALL PRIVILEGES ON DATABASE "51alpha" TO "51alpha_app";
\q

# Run schema (use docs/architecture/database-schema.md)
psql -U 51alpha_app -d 51alpha -f schema.sql
```

## Step 2: ModernUO Setup (5 minutes)

```bash
# Clone ModernUO
git clone https://github.com/modernuo/ModernUO.git
cd ModernUO

# Add Npgsql package
cd Projects/UOContent
dotnet add package Npgsql --version 8.0.1
```

## Step 3: Create 51alpha Directory (5 minutes)

```bash
# In Projects/UOContent/
mkdir -p Sphere51a/{Core,Combat,Factions,Siege,Tournament,Daily,Currency,Glicko,NPE,Gumps,Items,Mobiles,Commands,Telemetry}
```

## Step 4: Core Files (15 minutes)

Create these three files from `docs/implementation/ai-prompts.md`:

### 4.1 Sphere51aConfig.cs
```csharp
// Projects/UOContent/Sphere51a/Core/Sphere51aConfig.cs
namespace Sphere51a.Core;

public class Sphere51aConfig
{
    public static Sphere51aConfig Instance { get; private set; }
    
    public bool Enabled { get; set; } = true;
    public double FizzleManaConsumptionRate { get; set; } = 0.5;
    public int ProtectionFCPenalty { get; set; } = 0;
    public TimeSpan TalismanPvPDisableDuration { get; set; } = TimeSpan.FromMinutes(5);
    
    public static void Initialize() => Instance = new Sphere51aConfig();
}
```

### 4.2 Database.cs
```csharp
// Projects/UOContent/Sphere51a/Core/Database.cs
namespace Sphere51a.Core;

using Npgsql;

public static class Database
{
    private static NpgsqlDataSource _dataSource;
    
    public static void Initialize()
    {
        var connStr = "Host=localhost;Database=51alpha;Username=51alpha_app;Password=dev_password_change_me";
        _dataSource = new NpgsqlDataSourceBuilder(connStr).Build();
        using var conn = _dataSource.OpenConnection();
        Console.WriteLine("[51alpha] Database connected");
    }
    
    public static NpgsqlConnection GetConnection() => _dataSource?.OpenConnection();
}
```

### 4.3 Sphere51aCore.cs
```csharp
// Projects/UOContent/Sphere51a/Core/Sphere51aCore.cs
namespace Sphere51a.Core;

public static class Sphere51aCore
{
    public static void Initialize()
    {
        Console.WriteLine("[51alpha] Initializing...");
        Database.Initialize();
        Sphere51aConfig.Initialize();
        Console.WriteLine("[51alpha] Ready.");
    }
}
```

## Step 5: Hook Initialization (2 minutes)

Edit `Projects/UOContent/Initialization.cs`, add at end:

```csharp
Sphere51a.Core.Sphere51aCore.Initialize();
```

## Step 6: Test (5 minutes)

```bash
cd ModernUO
dotnet build
dotnet run --project Projects/Server

# Should see:
# [51alpha] Initializing...
# [51alpha] Database connected
# [51alpha] Ready.
```

## You're Running!

Next steps:
1. Read `docs/implementation/phases.md` for full roadmap
2. Use `docs/implementation/checklist.md` to track progress
3. Copy prompts from `docs/implementation/ai-prompts.md` for AI assistance

## Quick Reference

| Task | Document |
|------|----------|
| Modify spell casting | `specs/spell-system.md` |
| Add talisman logic | `specs/talisman-system.md` |
| Database queries | `docs/architecture/database-schema.md` |
| Hook into ModernUO | `docs/architecture/modernuo-integration.md` |
| All config options | `docs/reference/config.md` |
| World coordinates | `docs/reference/locations.md` |
| Admin commands | `docs/reference/commands.md` |

## Common Issues

### Database connection fails
```
Check PostgreSQL is running: sudo systemctl status postgresql
Check credentials in Database.cs match your setup
```

### Npgsql not found
```
Run: dotnet add package Npgsql --version 8.0.1
Run: dotnet restore
```

### Build errors in Spell.cs
```
Check you're modifying the correct file: Projects/UOContent/Spells/Base/Spell.cs
Ensure Sphere51aConfig.Instance is not null (Initialize() called)
```

## File Locations Cheat Sheet

```
ModernUO/
├── Projects/
│   ├── Server/              # Core server (don't modify)
│   └── UOContent/
│       ├── Sphere51a/       # ALL 51alpha code here
│       ├── Spells/Base/     # Spell.cs modifications
│       ├── Items/Tools/     # BaseRunicTool.cs (FC removal)
│       ├── Misc/            # AOS.cs, LootPack.cs
│       └── Guilds/          # Guild.cs extensions
└── Data/
    └── 51alpha/
        └── config.json      # Runtime configuration
```
