# AI Implementation Prompts

Copy-paste prompts for AI-assisted development. Each prompt is self-contained with context.

---

## Spell System

### Free Movement During Casting

```
Modify ModernUO Spell.cs OnCasterMoving() method to implement Sphere-style free movement:

Current behavior: Returns false during casting, blocking movement
Required behavior: Always return true to allow movement during casting

Add a config check at the start:
if (Sphere51aConfig.Instance?.Enabled == true)
    return true;

Keep original logic as fallback when Sphere mode disabled.

File: Projects/UOContent/Spells/Base/Spell.cs
```

### Disable Damage Interruption

```
Modify ModernUO Spell.cs OnCasterHurt() method to prevent damage from interrupting spells:

Current behavior: Calls Disturb(DisturbType.Hurt) when player takes damage while casting
Required behavior: In 51alpha mode, damage should NOT interrupt spells

Add early return at start of method:
public virtual void OnCasterHurt()
{
    // 51alpha: Damage does NOT interrupt spells
    if (Sphere51aConfig.Instance?.Enabled == true)
        return;
    
    // Original logic below...
}

Keep all original logic after the early return for non-51alpha compatibility.

File: Projects/UOContent/Spells/Base/Spell.cs
```

### Disable Equipment Interruption

```
Modify ModernUO Spell.cs OnCasterEquiping() method to prevent equipping from interrupting spells:

Current behavior: Calls Disturb(DisturbType.EquipRequest) when player equips item while casting
Required behavior: In 51alpha mode, equipping should NOT interrupt spells

Modify method:
public virtual bool OnCasterEquiping(Item item)
{
    // 51alpha: Equipping does NOT interrupt spells
    if (Sphere51aConfig.Instance?.Enabled == true)
        return true;
    
    if (IsCasting)
        Disturb(DisturbType.EquipRequest);
    return true;
}

File: Projects/UOContent/Spells/Base/Spell.cs
```

### Target-First Flow (Immediate Targeting)

```
Modify ModernUO Spell.cs Cast() method to show target cursor IMMEDIATELY:

Current flow: Cast() → CastTimer → OnCast() (target cursor after delay)
Required flow: Cast() → OnCast() (immediate targeting) → CheckSequence() → Delay → Execute

In 51alpha mode, call OnCast() immediately in Cast() instead of waiting for timer:

public virtual bool Cast()
{
    if (!CheckCast())
        return false;
    
    State = SpellState.Casting;
    Caster.Spell = this;
    
    // 51alpha: Immediate targeting - show cursor now
    if (Sphere51aConfig.Instance?.Enabled == true)
    {
        OnCast(); // Target cursor appears immediately
        return true;
    }
    
    // Original: start delay timer, OnCast called after
    // ... original code ...
}

This requires CheckSequence() to start the delay timer AFTER target selection.

File: Projects/UOContent/Spells/Base/Spell.cs
```

### Option C Mana Consumption (Pre-Delay with Fizzle Refund)

```
Modify ModernUO Spell.cs CheckSequence() method for 51alpha mana flow:

51alpha flow:
1. Consume reagents (unchanged)
2. Validate mana (unchanged)
3. CONSUME FULL MANA NOW (before delay)
4. Start CastDelayTimer
5. On timer completion: CheckFizzle
6. If fizzle: REFUND 50% mana

Implementation:
// After validation passes, in 51alpha mode:
if (Sphere51aConfig.Instance?.Enabled == true)
{
    Caster.Mana -= mana; // Consume 100% now
    
    var delay = GetCastDelay();
    new CastDelayTimer(this, delay, mana).Start();
    return true;
}

// CastDelayTimer.OnTick():
if (_spell.CheckFizzle())
{
    _spell.ExecuteSpell();
    // Scroll consumption here
}
else
{
    // FIZZLE: Refund 50% mana
    double refundRate = 1.0 - Sphere51aConfig.Instance.FizzleManaConsumptionRate; // 0.5
    int refund = (int)(_manaConsumed * refundRate);
    _spell.Caster.Mana += refund;
    _spell.DoFizzle();
}

File: Projects/UOContent/Spells/Base/Spell.cs
```

### Scroll Mana Reduction

```
Modify ModernUO Spell.cs ScaleMana() method to apply scroll mana reduction:

Scrolls should cost 43% less mana (divide by 1.755).

Add after existing LMC calculations:
// 51alpha: Scroll mana reduction (43% cheaper)
if (Sphere51aConfig.Instance?.Enabled == true && Scroll is SpellScroll)
{
    scalar /= 1.755; // ~43% reduction
}

Example: Fireball base 9 mana → scroll costs ~5 mana

File: Projects/UOContent/Spells/Base/Spell.cs
```

### Scroll Cast Speed Bonus

```
Modify ModernUO Spell.cs GetCastDelay() to apply scroll speed bonus:

Scrolls of Circle 3+ should cast 0.5 seconds faster.

Add in GetCastDelay():
// 51alpha: Scroll speed bonus for circle 3+
if (Sphere51aConfig.Instance?.Enabled == true && 
    Scroll is SpellScroll && 
    Circle >= SpellCircle.Third)
{
    return TimeSpan.FromSeconds(
        Math.Max(0.25, baseDelay.TotalSeconds - 0.5)
    );
}

Circle 1-2 scrolls: no speed bonus
Circle 3+ scrolls: 0.5s faster (minimum 0.25s)

File: Projects/UOContent/Spells/Base/Spell.cs
```

### Protection Spell FC Change

```
Modify ModernUO Spell.cs GetCastDelay() method to make Protection spell FC penalty configurable:

Find: if (ProtectionSpell.Registry.ContainsKey(Caster)) fc -= 2;

Replace with:
if (ProtectionSpell.Registry.ContainsKey(Caster))
    fc -= Sphere51aConfig.Instance?.ProtectionFCPenalty ?? 0;

Also create/update Sphere51aConfig.cs with:
public int ProtectionFCPenalty { get; set; } = 0;

File: Projects/UOContent/Spells/Base/Spell.cs
```

### FC Removal from BaseRunicTool

```
Modify ModernUO BaseRunicTool.cs to remove Faster Casting (AosAttribute.CastSpeed) from random attribute generation.

Search for all occurrences of: AosAttribute.CastSpeed
Either remove the case entirely, or replace with: AosAttribute.RegenMana

This affects:
- ApplyAttributesTo methods
- Random attribute selection switch statements

Keep all other attributes intact.

File: Projects/UOContent/Items/Tools/BaseRunicTool.cs
```

---

## Configuration

### Sphere51aConfig Singleton

```
Create a configuration singleton for 51alpha combat settings:

namespace Sphere51a.Core;

public class Sphere51aConfig
{
    public static Sphere51aConfig Instance { get; private set; }
    
    // Core toggles
    public bool Enabled { get; set; } = true;
    public bool AllowMovementDuringCast { get; set; } = true;
    
    // Spell interruption
    public bool DamageInterrupts { get; set; } = false;
    public bool EquipInterrupts { get; set; } = false;
    public bool MovementInterrupts { get; set; } = false;
    public bool WarModeInterrupts { get; set; } = true;
    public bool BandageInterrupts { get; set; } = true;
    
    // Mana consumption
    public double SuccessManaConsumptionRate { get; set; } = 1.0;
    public double FizzleManaConsumptionRate { get; set; } = 0.5;
    
    // Scroll bonuses
    public double ScrollManaReduction { get; set; } = 0.43;
    public double ScrollSpeedBonus { get; set; } = 0.5;
    public int ScrollSpeedMinCircle { get; set; } = 3;
    
    // Protection spell
    public int ProtectionFCPenalty { get; set; } = 0;
    
    // Talisman
    public TimeSpan TalismanPvPDisableDuration { get; set; } = TimeSpan.FromMinutes(5);
    
    public static void Initialize()
    {
        Instance = new Sphere51aConfig();
        // Load from config file/database
    }
}

File: Projects/UOContent/Sphere51a/Core/Sphere51aConfig.cs
```

---

## Database

### PostgreSQL Connection

```
Create a database connection class using Npgsql for ModernUO:

namespace Sphere51a.Core;

using Npgsql;

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
    
    public static async Task<NpgsqlConnection> GetConnectionAsync() 
        => await _dataSource.OpenConnectionAsync();
    
    public static async Task ExecuteAsync(string sql, params NpgsqlParameter[] parameters)
    {
        await using var conn = await GetConnectionAsync();
        await using var cmd = new NpgsqlCommand(sql, conn);
        cmd.Parameters.AddRange(parameters);
        await cmd.ExecuteNonQueryAsync();
    }
    
    public static async Task<T> ScalarAsync<T>(string sql, params NpgsqlParameter[] parameters)
    {
        await using var conn = await GetConnectionAsync();
        await using var cmd = new NpgsqlCommand(sql, conn);
        cmd.Parameters.AddRange(parameters);
        var result = await cmd.ExecuteScalarAsync();
        return (T)Convert.ChangeType(result, typeof(T));
    }
}

Required NuGet: Npgsql 8.0.1
File: Projects/UOContent/Sphere51a/Core/Database.cs
```

---

## Talisman System

### TalismanState with Pause/Resume

```
Create a talisman state tracker that supports pause/resume for PvP disable timer:

namespace Sphere51a.Progression;

public class TalismanState
{
    public DateTime? DisabledUntil { get; private set; }
    private TimeSpan _remainingTime;
    
    public bool IsActive => !DisabledUntil.HasValue || DateTime.UtcNow >= DisabledUntil;
    
    public void TriggerPvPDisable()
    {
        var duration = Sphere51aConfig.Instance.TalismanPvPDisableDuration;
        DisabledUntil = DateTime.UtcNow.Add(duration);
        _remainingTime = TimeSpan.Zero;
    }
    
    public void OnUnequip()
    {
        if (DisabledUntil.HasValue && DisabledUntil > DateTime.UtcNow)
        {
            _remainingTime = DisabledUntil.Value - DateTime.UtcNow;
            DisabledUntil = null; // Pause
        }
    }
    
    public void OnEquip()
    {
        if (_remainingTime > TimeSpan.Zero)
        {
            DisabledUntil = DateTime.UtcNow.Add(_remainingTime);
            _remainingTime = TimeSpan.Zero;
        }
    }
    
    public TimeSpan? GetRemainingTime()
    {
        if (!DisabledUntil.HasValue)
            return _remainingTime > TimeSpan.Zero ? _remainingTime : null;
        
        var remaining = DisabledUntil.Value - DateTime.UtcNow;
        return remaining > TimeSpan.Zero ? remaining : null;
    }
}

File: Projects/UOContent/Sphere51a/Progression/TalismanState.cs
```

### BuildManager Service

```
Create the central build manager for talisman state and PvP context:

namespace Sphere51a.Progression;

public static class BuildManager
{
    private static Dictionary<Serial, TalismanState> _states = new();
    
    public static bool IsTalismanActive(PlayerMobile pm)
    {
        if (!_states.TryGetValue(pm.Serial, out var state))
            return true;
        return state.IsActive;
    }
    
    public static void OnPvPEngagement(PlayerMobile pm)
    {
        if (!_states.TryGetValue(pm.Serial, out var state))
        {
            state = new TalismanState();
            _states[pm.Serial] = state;
        }
        state.TriggerPvPDisable();
    }
    
    public static void OnTalismanEquip(PlayerMobile pm)
    {
        if (_states.TryGetValue(pm.Serial, out var state))
            state.OnEquip();
    }
    
    public static void OnTalismanUnequip(PlayerMobile pm)
    {
        if (_states.TryGetValue(pm.Serial, out var state))
            state.OnUnequip();
    }
    
    public static int ApplyTalismanBonus(int damage, PlayerMobile pm)
    {
        if (!IsTalismanActive(pm))
            return damage;
        
        var talisman = pm.FindItemOnLayer(Layer.Talisman) as BaseTalisman;
        if (talisman == null)
            return damage;
        
        return (int)(damage * (1.0 + talisman.DamageBonus));
    }
    
    public static bool IsPvPContext(Mobile attacker, Mobile defender)
    {
        if (attacker is PlayerMobile && defender is PlayerMobile)
            return true;
        if (defender is BaseCreature bc && bc.ControlMaster is PlayerMobile)
            return true;
        if (attacker is BaseCreature bc2 && bc2.ControlMaster is PlayerMobile 
            && defender is PlayerMobile)
            return true;
        return false;
    }
}

File: Projects/UOContent/Sphere51a/Progression/BuildManager.cs
```

---

## Admin Commands

### RemoveAllFC Command

```
Create an admin command to remove Faster Casting from all items in the world:

namespace Sphere51a.Commands;

public static class AdminCommands
{
    public static void Register()
    {
        CommandSystem.Register("RemoveAllFC", AccessLevel.Administrator, RemoveAllFC_OnCommand);
    }
    
    [Usage("[RemoveAllFC")]
    [Description("Removes Faster Casting attribute from all items")]
    private static void RemoveAllFC_OnCommand(CommandEventArgs e)
    {
        int count = 0;
        
        foreach (Item item in World.Items.Values)
        {
            if (item is BaseWeapon weapon && weapon.Attributes[AosAttribute.CastSpeed] != 0)
            {
                weapon.Attributes[AosAttribute.CastSpeed] = 0;
                count++;
            }
            else if (item is BaseArmor armor && armor.Attributes[AosAttribute.CastSpeed] != 0)
            {
                armor.Attributes[AosAttribute.CastSpeed] = 0;
                count++;
            }
            else if (item is BaseJewel jewel && jewel.Attributes[AosAttribute.CastSpeed] != 0)
            {
                jewel.Attributes[AosAttribute.CastSpeed] = 0;
                count++;
            }
            else if (item is BaseClothing clothing && clothing.Attributes[AosAttribute.CastSpeed] != 0)
            {
                clothing.Attributes[AosAttribute.CastSpeed] = 0;
                count++;
            }
            else if (item is Spellbook book && book.Attributes[AosAttribute.CastSpeed] != 0)
            {
                book.Attributes[AosAttribute.CastSpeed] = 0;
                count++;
            }
        }
        
        e.Mobile.SendMessage($"Removed Faster Casting from {count} items.");
        Console.WriteLine($"[51alpha] FC removed from {count} items by {e.Mobile.Name}");
    }
}

File: Projects/UOContent/Sphere51a/Commands/AdminCommands.cs
```

---

## Event Hooks

### Kill Attribution

```
Create kill attribution tracking for PvP kills:

namespace Sphere51a.Combat;

public static class KillAttribution
{
    private static Dictionary<Serial, DamageTracker> _trackers = new();
    
    public static void RecordDamage(Mobile attacker, Mobile victim, int damage)
    {
        if (!_trackers.TryGetValue(victim.Serial, out var tracker))
        {
            tracker = new DamageTracker();
            _trackers[victim.Serial] = tracker;
        }
        tracker.AddDamage(attacker.Serial, damage);
    }
    
    public static void OnPlayerDeath(PlayerDeathEventArgs e)
    {
        if (e.Mobile is PlayerMobile victim && victim.LastKiller is PlayerMobile killer)
        {
            ProcessKill(killer, victim);
        }
    }
    
    public static void ProcessKill(PlayerMobile killer, PlayerMobile victim)
    {
        // Trigger PvP disable for both
        BuildManager.OnPvPEngagement(killer);
        BuildManager.OnPvPEngagement(victim);
        
        // Award points based on context
        if (SiegeManager.IsInActiveSiege(killer))
        {
            SiegeManager.ProcessKill(killer, victim);
        }
        
        // Update Glicko ratings
        GlickoManager.RecordMatch(killer, victim);
        
        // Clear damage tracker
        _trackers.Remove(victim.Serial);
    }
    
    private class DamageTracker
    {
        public Dictionary<Serial, int> DamageByPlayer { get; } = new();
        public DateTime FirstDamage { get; private set; }
        
        public void AddDamage(Serial attacker, int damage)
        {
            if (DamageByPlayer.Count == 0)
                FirstDamage = DateTime.UtcNow;
            
            if (!DamageByPlayer.TryGetValue(attacker, out var total))
                total = 0;
            DamageByPlayer[attacker] = total + damage;
        }
    }
}

File: Projects/UOContent/Sphere51a/Combat/KillAttribution.cs
```

---

## Initialization

### Sphere51aCore Main Entry

```
Create the main initialization entry point:

namespace Sphere51a.Core;

public static class Sphere51aCore
{
    public static void Initialize()
    {
        Console.WriteLine("[51alpha] Initializing systems...");
        
        // Core systems (order matters)
        Database.Initialize();
        Config.Load();
        Sphere51aConfig.Initialize();
        
        // Register event handlers
        EventSink.PlayerDeath += Combat.KillAttribution.OnPlayerDeath;
        EventSink.CreatureDeath += Daily.BountyManager.OnCreatureDeath;
        EventSink.Login += OnPlayerLogin;
        EventSink.Logout += OnPlayerLogout;
        
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
        
        // Start timers
        Sphere51aTimers.Initialize();
        
        // Register commands
        Commands.AdminCommands.Register();
        Commands.SiegeCommands.Register();
        Commands.FactionCommands.Register();
        
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
    
    private static void OnPlayerLogout(LogoutEventArgs e)
    {
        if (e.Mobile is PlayerMobile pm)
        {
            Tournament.TournamentManager.OnPlayerLogout(pm);
            Siege.SiegeManager.OnPlayerLogout(pm);
        }
    }
}

Then add to ModernUO Initialization.cs:
Sphere51a.Core.Sphere51aCore.Initialize();

File: Projects/UOContent/Sphere51a/Core/Sphere51aCore.cs
```
