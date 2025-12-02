# Swamp and Desert Event Deep Dive

## Overview
Regional event systems for swamp/desert areas, triggering quests or challenges.

## Algorithms and Logic
Event spawning, progression triggers, rewards distribution.

## Edge Cases
Multi-player participation, region boundaries.

## Implementation Details
Spawner coordination, UI notifications.

## Testing Plan
Event cycles, reward verification.

## Change Log
[To be filled]

# Swamp & Desert Event Systems: Developer Architecture Analysis

Based on the architecture leverage documentation, I'll analyze the planned swamp and desert event systems as Phase 8 region implementations.

## __System Architecture Overview__

### __Core ModernUO Foundation: Region System__

```csharp
// Base region architecture that Sphere51a extends
public abstract class BaseRegion
{
    public virtual void OnEnter(Mobile m) { }
    public virtual void OnExit(Mobile m) { }
    public virtual void OnSpeech(SpeechEventArgs e) { }
    public virtual bool OnBeginSpellCast(Mobile m, ISpell spell) => true;
    public virtual TimeSpan GetLogoutDelay(Mobile m) => TimeSpan.Zero;
    
    // Sphere51a leverages these extension points
    protected virtual void OnPlayerAdded(Mobile m) { }
    protected virtual void OnPlayerRemoved(Mobile m) { }
}
```

### __Sphere51a Extensions Pattern__

```csharp
// Standard extension pattern used across Sphere51a systems
[Conditional("SPHERE")]
public static class Sphere51aRegionExtensions
{
    // Hook into ModernUO region system without core modifications
    public static void InitializeRegionHooks()
    {
        // EventSink.RegionEnter += OnRegionEnter;
        // EventSink.RegionExit += OnRegionExit;
        // EventSink.PlayerMovement += OnMovement;
    }
}
```

---

## __1. SwampProgressRegion.cs - Shared Progress Bar System__

### __Design Intent__

Creates shared, persistent progress bars across multiple players in swamp regions. Encourages group cooperation and prevents AFK camping at resource nodes.

### __Technical Implementation__

#### __Region State Management__

```csharp
public class SwampProgressRegion : MondainRegion
{
    // Core state: Shared progress dictionary
    private static readonly ConcurrentDictionary<string, SwampProgressState> _progressStates = new();
    
    // Individual player contributions
    private static readonly ConcurrentDictionary<long, PlayerProgress> _playerContributions = new();
    
    public class SwampProgressState
    {
        public string RegionId { get; set; }
        public int CurrentProgress { get; set; } // 0-1000
        public int MaxProgress { get; set; } = 1000;
        public TimeSpan ResetTimer { get; set; }
        public bool IsActive => ResetTimer > TimeSpan.Zero;
    }
    
    public class PlayerProgress
    {
        public long PlayerSerial { get; set; }
        public int ContributionPoints { get; set; }
        public DateTime LastContribution { get; set; }
        public bool HasClaimedReward { get; set; }
    }
}
```

#### __Progress Mechanics__

```csharp
// Contribution system (called on player actions)
public void AddProgress(Mobile contributor, int points)
{
    var regionId = GetRegionKey(this);
    var player = contributor as PlayerMobile;
    if (player == null) return;
    
    // Get/create region state
    var state = _progressStates.GetOrAdd(regionId, 
        id => new SwampProgressState { RegionId = id });
    
    // Track player contribution
    var playerContrib = _playerContributions.GetOrAdd(player.Serial.Value,
        serial => new PlayerProgress { PlayerSerial = serial });
    
    // Add progress with diminishing returns for high contributors
    var effectivePoints = CalculateEffectivePoints(points, playerContrib.ContributionPoints);
    state.CurrentProgress += effectivePoints;
    playerContrib.ContributionPoints += effectivePoints;
    playerContrib.LastContribution = DateTime.UtcNow;
    
    // Check completion
    if (state.CurrentProgress >= state.MaxProgress)
    {
        CompleteRegionProgress(state, regionId);
    }
    
    // Update all players in region
    BroadcastProgressUpdate(state);
}
```

#### __Timer & Reset System__

```csharp
// Automatic reset mechanism
private static readonly Timer _progressResetTimer = Timer.DelayCall(
    TimeSpan.FromMinutes(30), // Reset period
    TimeSpan.FromMinutes(30),
    () => ResetExpiredProgress());

private static void ResetExpiredProgress()
{
    var now = DateTime.UtcNow;
    
    foreach (var kvp in _progressStates)
    {
        var state = kvp.Value;
        if (now - state.LastProgressUpdate > TimeSpan.FromHours(2))
        {
            // Reset progress for inactive regions
            state.CurrentProgress = 0;
            state.ResetTimer = TimeSpan.Zero;
            BroadcastRegionReset(kvp.Key);
        }
    }
}
```

#### __Reward Distribution__

```csharp
private void CompleteRegionProgress(SwampProgressState state, string regionId)
{
    var regionPlayers = GetPlayersInRegion(regionId);
    
    foreach (var player in regionPlayers)
    {
        var contribution = _playerContributions[player.Serial.Value];
        var rewardMultiplier = CalculateRewardMultiplier(contribution.ContributionPoints);
        
        // Distribute rewards (items, XP, etc.)
        GivePlayerReward(player, contribution, rewardMultiplier);
    }
    
    // Reset contributions for next cycle
    ResetPlayerContributions(regionId);
}
```

---

## __2. DesertGateSystem.cs - Combination Lock Access Control__

### __Design Intent__

Creates puzzle-based access control using desert terrain features. Players must solve environmental combination locks to access restricted areas.

### __Technical Implementation__

#### __Gate State Management__

```csharp
public class DesertGateSystem
{
    private static readonly ConcurrentDictionary<string, GateCombination> _activeGates = new();
    
    public class GateCombination
    {
        public string GateId { get; set; }
        public int[] RequiredSequence { get; set; } = new int[4]; // 4-digit combination
        public int[] PlayerSequence { get; set; } = new int[4];   // Current player progress
        public TimeSpan TimeLimit { get; set; } = TimeSpan.FromMinutes(10);
        public DateTime StartedAt { get; set; }
        public bool IsSolved => PlayerSequence.SequenceEqual(RequiredSequence);
    }
}
```

#### __Gate Interaction Logic__

```csharp
// Called when player interacts with desert gate elements
public void OnGateInteraction(Mobile player, DesertGateElement element, int sequencePosition)
{
    var gateId = GetGateKey(element.Location);
    var gate = _activeGates.GetOrAdd(gateId, id => GenerateNewCombination(id));
    
    // Check if gate is expired
    if (DateTime.UtcNow - gate.StartedAt > gate.TimeLimit)
    {
        ResetGate(gateId);
        gate = _activeGates[gateId];
    }
    
    // Process player input
    gate.PlayerSequence[sequencePosition] = element.ElementType;
    
    // Check for completion
    if (gate.IsSolved)
    {
        GrantGateAccess(player, gateId);
        BroadcastGateSolved(gateId);
    }
    else if (IsSequenceIncorrect(gate))
    {
        // Punish failed attempts
        ApplyFailurePenalty(player, gateId);
        ResetGate(gateId);
    }
}
```

#### __Combination Generation & Hints__

```csharp
private GateCombination GenerateNewCombination(string gateId)
{
    var combo = new GateCombination
    {
        GateId = gateId,
        StartedAt = DateTime.UtcNow,
        RequiredSequence = new int[4]
    };
    
    // Generate solvable combination with environmental clues
    var random = new Random();
    
    // Desert elements: Oasis=0, Sandstorm=1, Cactus=2, Ruins=3
    for (int i = 0; i < 4; i++)
    {
        // Weighted generation favoring environmental elements
        combo.RequiredSequence[i] = random.Next(4);
    }
    
    // Place hints in environment
    PlaceEnvironmentalHints(combo.RequiredSequence, gateId);
    
    return combo;
}
```

#### __Environmental Hint System__

```csharp
private void PlaceEnvironmentalHints(int[] sequence, string gateId)
{
    var gateRegion = GetGateRegion(gateId);
    
    for (int i = 0; i < sequence.Length; i++)
    {
        var hintLocation = CalculateHintLocation(gateRegion, i);
        
        // Spawn hint objects based on sequence
        switch (sequence[i])
        {
            case 0: // Oasis hint
                new OasisHint().MoveToWorld(hintLocation);
                break;
            case 1: // Sandstorm hint
                new SandstormHint().MoveToWorld(hintLocation);
                break;
            case 2: // Cactus hint
                new CactusHint().MoveToWorld(hintLocation);
                break;
            case 3: // Ruins hint
                new RuinsHint().MoveToWorld(hintLocation);
                break;
        }
    }
}
```

---

## __3. RotatingBonusSystem.cs - Weekly Rotation Mechanics__

### __Design Intent__

Provides dynamic region bonuses that change weekly (especially "Tuesday cycles") to keep regions fresh and encourage exploration patterns.

### __Technical Implementation__

#### __Bonus Rotation Engine__

```csharp
public class RotatingBonusSystem
{
    public enum BonusType
    {
        SkillGain = 0,
        LootQuality = 1,
        MonsterDifficulty = 2,
        ResourceRespawn = 3,
        ExperienceMultiplier = 4
    }
    
    private static readonly ConcurrentDictionary<string, WeeklyBonus> _regionBonuses = new();
    
    public class WeeklyBonus
    {
        public string RegionId { get; set; }
        public BonusType CurrentBonus { get; set; }
        public float BonusMultiplier { get; set; } // 0.5-3.0
        public DateTime WeekEnd { get; set; } // Sunday reset
        public int WeekNumber { get; set; }
    }
}
```

#### __Weekly Rotation Logic__

```csharp
// Called weekly (Sunday) by scheduled task
public static void RotateWeeklyBonuses()
{
    var currentWeek = GetCurrentWeekNumber();
    
    foreach (var regionId in GetAllTrackedRegions())
    {
        var bonus = _regionBonuses.GetOrAdd(regionId, 
            id => new WeeklyBonus { RegionId = id });
        
        // Rotate to next bonus type
        bonus.CurrentBonus = (BonusType)(((int)bonus.CurrentBonus + 1) % Enum.GetValues<BonusType>().Length);
        bonus.WeekNumber = currentWeek;
        bonus.WeekEnd = GetWeekEnd(currentWeek);
        bonus.BonusMultiplier = GenerateBonusMultiplier(bonus.CurrentBonus);
        
        // Apply any Tuesday special rules
        if (IsTuesdayBonusWeek(currentWeek))
        {
            bonus.BonusMultiplier *= 2.0f; // Tuesday double bonus
        }
        
        BroadcastBonusChange(regionId, bonus);
    }
    
    SaveBonusConfiguration();
}
```

#### __Tuesday Special Cycle__

```csharp
private static bool IsTuesdayBonusWeek(int weekNumber)
{
    // Special Tuesday rotation cycle
    // Every 4th week, or weeks ending on Tuesday
    return (weekNumber % 4) == 0 || 
           GetWeekEnd(weekNumber).DayOfWeek == DayOfWeek.Tuesday;
}

private static float GenerateBonusMultiplier(BonusType bonusType)
{
    var random = new Random();
    
    return bonusType switch
    {
        BonusType.SkillGain => 0.8f + (float)random.NextDouble() * 0.7f,      // 0.8-1.5x
        BonusType.LootQuality => 1.2f + (float)random.NextDouble() * 1.0f,    // 1.2-2.2x  
        BonusType.MonsterDifficulty => 0.5f + (float)random.NextDouble() * 1.0f, // 0.5-1.5x
        BonusType.ResourceRespawn => 2.0f + (float)random.NextDouble() * 2.0f,  // 2.0-4.0x
        BonusType.ExperienceMultiplier => 1.1f + (float)random.NextDouble() * 0.9f, // 1.1-2.0x
        _ => 1.0f
    };
}
```

#### __Bonus Application Hooks__

```csharp
// Integrate with existing region system
public static void ApplyRegionBonus(Mobile player, string regionId)
{
    if (!_regionBonuses.TryGetValue(regionId, out var bonus))
        return;
        
    switch (bonus.CurrentBonus)
    {
        case BonusType.SkillGain:
            ApplySkillGainBonus(player, bonus.BonusMultiplier);
            break;
            
        case BonusType.LootQuality:
            ApplyLootQualityBonus(player, bonus.BonusMultiplier);
            break;
            
        case BonusType.MonsterDifficulty:
            ApplyMonsterDifficultyBonus(player, bonus.BonusMultiplier);
            break;
            
        case BonusType.ResourceRespawn:
            ApplyResourceRespawnBonus(regionId, bonus.BonusMultiplier);
            break;
            
        case BonusType.ExperienceMultiplier:
            ApplyExperienceBonus(player, bonus.BonusMultiplier);
            break;
    }
}
```

---

## __4. Region Integration Pattern__

### __ModernUO Region Extension__

```csharp
// Swamp region extension
public class SwampProgressExtension : RegionExtension
{
    public override void OnEnter(Mobile m)
    {
        base.OnEnter(m);
        
        if (m is PlayerMobile pm)
        {
            SwampProgressRegion.RegisterPlayer(pm, GetRegionKey(this));
        }
    }
    
    public override void OnExit(Mobile m)
    {
        base.OnExit(m);
        
        if (m is PlayerMobile pm)  
        {
            SwampProgressRegion.UnregisterPlayer(pm);
        }
    }
}

// Desert region extension
public class DesertGateExtension : RegionExtension
{
    public override void OnDoubleClick(Mobile m, object clicked)
    {
        if (clicked is DesertGateElement element)
        {
            DesertGateSystem.OnGateInteraction(m, element, element.SequencePosition);
        }
    }
}
```

### __Memory Optimization__

```csharp
// Pool-based state management (zero-LINQ hot paths)
private static readonly ObjectPool<SwampProgressState> _progressPool = ObjectPool.Create(
    new DefaultPooledObjectPolicy<SwampProgressState>());

private static readonly ObjectPool<GateCombination> _gateCombinationPool = ObjectPool.Create(
    new DefaultPooledObjectPolicy<GateCombination>());

// Atomic persistence with background saves
private static readonly ConcurrentQueue<PersistenceOperation> _saveQueue = new();
```

---

## __Performance Considerations__

### __Scalability Metrics__

- __Memory per region__: ~2-5KB (progress states, player lists)
- __Database operations__: Batched saves every 30 seconds
- __Network updates__: Targeted broadcasts to region players only
- __Timer overhead__: One global timer per system

### __Concurrent Access Patterns__

```csharp
// Read-heavy with occasional writes
private static readonly ReaderWriterLockSlim _regionLock = new();

// Hot path: No contention reads
public void GetProgress(string regionId)
{
    using (_regionLock.ReadLock())
    {
        return _progressStates.TryGetValue(regionId, out var state) ? state : null;
    }
}

// Write path: Minimal contention
public void UpdateProgress(string regionId, SwampProgressState newState)
{
    using (_regionLock.WriteLock())
    {
        _progressStates[regionId] = newState;
        _saveQueue.Enqueue(new PersistenceOperation(regionId, newState));
    }
}
```

---

## __Development Integration Guide__

### __For New Region Features__

1. __Extend BaseRegion__: Add custom override methods
2. __Register Events__: Hook into RegionSink for global events
3. __State Management__: Use concurrent collections with pooling
4. __Timer Integration__: Leverage existing RegionTicker infrastructure
5. __Persistence__: Implement atomic saves with error recovery

### __Testing Strategy__

```csharp
[Test]
public void SwampProgress_SharedProgressAccumulates()
{
    var region = CreateTestRegion();
    var players = CreateTestPlayers(3);
    
    // Each player contributes
    foreach (var player in players)
    {
        region.AddProgress(player, 100);
    }
    
    Assert.AreEqual(300, region.GetTotalProgress());
    Assert.AreEqual(100, region.GetPlayerContribution(players[0]));
}

[Test]  
public void DesertGate_CombinationLocksWork()
{
    var gate = CreateTestGate();
    var player = CreateTestPlayer();
    
    // Player solves combination
    gate.Interact(player, 0, ElementType.Cactus);
    gate.Interact(player, 1, ElementType.Ruins);
    
    Assert.IsFalse(gate.IsSolved);
    
    gate.Interact(player, 2, ElementType.Oasis);
    gate.Interact(player, 3, ElementType.Sandstorm);
    
    Assert.IsTrue(gate.IsSolved);
}
```

These systems provide dynamic, player-driven events that enhance world exploration while maintaining low server overhead through efficient state management and event-driven architecture.
