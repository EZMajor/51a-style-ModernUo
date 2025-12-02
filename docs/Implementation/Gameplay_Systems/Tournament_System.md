# Tournament System Specification
## 51alpha Automated 1v1 Tournament System

### Document Metadata
- **Version**: v1.0.0
- **Last Updated**: 2025-01-02
- **Authors**: 51alpha Development Team
- **Applicable ModernUO Version**: v24.0.0+
- **PR Ready**: No (Conceptual Design)

---

## 1. Executive Summary

The 51alpha Tournament System provides automated, scheduled 1v1 single-elimination tournaments with spectator betting, trophy rewards, and a unique cosmetic currency system. Tournaments run at set times targeting both European and North American players.

### Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Selection Method | Random | Fair entry, exciting upsets |
| Fight Duration | No limit (death) | Authentic PvP resolution |
| Entry Fee | Free | Encourage participation |
| Minimum Players | 6 | Meaningful bracket |
| Maximum Players | Uncapped | Scale with population |
| Parallel Arenas | 8 (expandable) | Fast tournament progression |
| Prize | Trophy statue + 100 Tournament Coins | Collectible + cosmetic currency |

---

## 2. Tournament Schedule

### 2.1 Weekly Schedule

| Day | Time (EST) | Target Region |
|-----|------------|---------------|
| Wednesday | 2:00 PM | Europe |
| Wednesday | 8:00 PM | North America |
| Saturday | 2:00 PM | Europe |
| Saturday | 8:00 PM | North America |
| Sunday | 2:00 PM | Europe |
| Sunday | 8:00 PM | North America |

**Total**: 6 tournaments per week

### 2.2 Tournament Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TOURNAMENT TIMELINE                                   │
└─────────────────────────────────────────────────────────────────────────┘

  T-15 min           T-0               ROUNDS              AWARDS
  ───────────────►──────────────────►──────────────────►──────────────────
  
  │                 │                  │                   │
  │ Registration    │ Registration     │ Fights proceed    │ Trophy given
  │ opens           │ closes           │ in parallel       │ Coins awarded
  │                 │ Bracket generated│ Auto-refresh      │ Title granted
  │ Town Cryer      │ Players teleport │ between rounds    │ Broadcast
  │ announces       │ to waiting area  │                   │
  │                 │                  │                   │
```

---

## 3. Registration System

### 3.1 Tournament NPC

**Location**: Britain Tournament Stadium (coordinates TBD)

```csharp
public class TournamentRegistrar : BaseVendor
{
    public override void OnDoubleClick(Mobile from)
    {
        if (!(from is PlayerMobile pm))
            return;
            
        if (!TournamentManager.IsRegistrationOpen)
        {
            var nextTournament = TournamentManager.GetNextTournamentTime();
            Say($"Registration is closed. Next tournament: {nextTournament:dddd} at {nextTournament:h:mm tt} EST");
            return;
        }
        
        pm.SendGump(new TournamentRegistrationGump(pm));
    }
}

public class TournamentRegistrationGump : Gump
{
    public TournamentRegistrationGump(PlayerMobile player) : base(50, 50)
    {
        AddBackground(0, 0, 400, 350, 9200);
        AddLabel(120, 20, 0x35, "Tournament Registration");
        
        // Tournament info
        AddLabel(20, 60, 0, $"Next Tournament: {TournamentManager.CurrentTournamentTime:h:mm tt} EST");
        AddLabel(20, 80, 0, $"Registered Players: {TournamentManager.RegisteredCount}");
        AddLabel(20, 100, 0, $"Minimum Required: 6 players");
        
        // Rules
        AddLabel(20, 140, 0x35, "Tournament Rules:");
        AddLabel(20, 160, 0, "• 1v1 Single Elimination");
        AddLabel(20, 180, 0, "• Random matchups (no seeding)");
        AddLabel(20, 200, 0, "• Fight until death");
        AddLabel(20, 220, 0, "• Free entry");
        AddLabel(20, 240, 0, "• Winner receives Trophy + 100 Coins");
        
        // Registration status
        if (TournamentManager.IsPlayerRegistered(player))
        {
            AddLabel(20, 280, 0x35, "You are registered!");
            AddButton(150, 310, 4017, 4019, 2, GumpButtonType.Reply, 0);
            AddLabel(185, 310, 0x22, "Withdraw");
        }
        else
        {
            AddButton(150, 310, 4005, 4007, 1, GumpButtonType.Reply, 0);
            AddLabel(185, 310, 0x35, "Register");
        }
    }
    
    public override void OnResponse(NetState sender, RelayInfo info)
    {
        var pm = sender.Mobile as PlayerMobile;
        
        switch (info.ButtonID)
        {
            case 1: // Register
                TournamentManager.RegisterPlayer(pm);
                pm.SendMessage(0x35, "You have registered for the tournament!");
                break;
                
            case 2: // Withdraw
                TournamentManager.UnregisterPlayer(pm);
                pm.SendMessage(0x22, "You have withdrawn from the tournament.");
                break;
        }
    }
}
```

### 3.2 Registration Rules

```csharp
public static class TournamentRegistration
{
    public static readonly TimeSpan RegistrationWindow = TimeSpan.FromMinutes(15);
    
    public static RegistrationResult RegisterPlayer(PlayerMobile player)
    {
        // Check if registration is open
        if (!IsRegistrationOpen)
            return RegistrationResult.RegistrationClosed;
            
        // Check if already registered
        if (_registeredPlayers.Contains(player.Serial))
            return RegistrationResult.AlreadyRegistered;
            
        // Check if player is alive
        if (!player.Alive)
            return RegistrationResult.MustBeAlive;
            
        // Check if player is in combat
        if (player.Combatant != null)
            return RegistrationResult.InCombat;
            
        // Check Young player status - they can't do PvP
        if (player.Young)
            return RegistrationResult.YoungPlayerBlocked;
            
        // Register
        _registeredPlayers.Add(player.Serial);
        
        return RegistrationResult.Success;
    }
    
    public static void OpenRegistration(DateTime tournamentTime)
    {
        _registeredPlayers.Clear();
        _registrationOpenTime = DateTime.UtcNow;
        _tournamentTime = tournamentTime;
        
        // Town Cryer announcement
        TownCryerManager.AnnounceEvent(TownCryerCategory.ServerEvent,
            "Tournament registration is now open! Speak to the Tournament Registrar in Britain to sign up!");
            
        // World broadcast
        World.Broadcast(0x35, true, 
            "Tournament registration is now OPEN! Visit Britain Stadium to register. Tournament begins in 15 minutes!");
    }
}
```

---

## 4. Bracket Generation

### 4.1 Bye System (Early Round Byes)

Byes are distributed in Round 1 to reach the next power of 2:

```csharp
public class TournamentBracket
{
    private List<TournamentMatch> _matches = new();
    private int _totalRounds;
    
    public static TournamentBracket Generate(List<PlayerMobile> participants)
    {
        var bracket = new TournamentBracket();
        
        // Shuffle randomly - NO seeding
        var shuffled = participants
            .OrderBy(_ => Utility.Random(int.MaxValue))
            .ToList();
        
        int playerCount = shuffled.Count;
        int bracketSize = NextPowerOf2(playerCount);
        int byesNeeded = bracketSize - playerCount;
        
        bracket._totalRounds = (int)Math.Log2(bracketSize);
        
        // Create Round 1 matches
        // Byes are distributed to make bracket fair
        var round1Slots = new List<TournamentSlot>();
        
        // Distribute byes randomly among Round 1 positions
        var byePositions = new HashSet<int>();
        while (byePositions.Count < byesNeeded)
        {
            byePositions.Add(Utility.Random(bracketSize));
        }
        
        int playerIndex = 0;
        for (int i = 0; i < bracketSize; i++)
        {
            if (byePositions.Contains(i))
            {
                round1Slots.Add(TournamentSlot.Bye);
            }
            else
            {
                round1Slots.Add(new TournamentSlot(shuffled[playerIndex++]));
            }
        }
        
        // Generate Round 1 matches
        for (int i = 0; i < bracketSize; i += 2)
        {
            var match = new TournamentMatch
            {
                Round = 1,
                MatchNumber = i / 2,
                Slot1 = round1Slots[i],
                Slot2 = round1Slots[i + 1]
            };
            
            // Auto-advance if bye
            if (match.Slot1.IsBye)
            {
                match.Winner = match.Slot2;
                match.IsComplete = true;
            }
            else if (match.Slot2.IsBye)
            {
                match.Winner = match.Slot1;
                match.IsComplete = true;
            }
            
            bracket._matches.Add(match);
        }
        
        // Generate placeholder matches for future rounds
        int matchesInRound = bracketSize / 4;
        for (int round = 2; round <= bracket._totalRounds; round++)
        {
            for (int m = 0; m < matchesInRound; m++)
            {
                bracket._matches.Add(new TournamentMatch
                {
                    Round = round,
                    MatchNumber = m,
                    Slot1 = TournamentSlot.Pending,
                    Slot2 = TournamentSlot.Pending
                });
            }
            matchesInRound /= 2;
        }
        
        return bracket;
    }
    
    private static int NextPowerOf2(int n)
    {
        if (n < 2) return 2;
        int power = 1;
        while (power < n) power *= 2;
        return power;
    }
}
```

### 4.2 Example Bracket (13 Players)

```
Round 1 (16 slots, 3 byes):

Match 1: Player A vs Player B
Match 2: Player C vs BYE → Player C advances
Match 3: Player D vs Player E
Match 4: Player F vs Player G
Match 5: BYE vs Player H → Player H advances
Match 6: Player I vs Player J
Match 7: Player K vs Player L
Match 8: Player M vs BYE → Player M advances

Round 2 (8 players):
Match 9: Winner(1) vs Player C
Match 10: Winner(3) vs Winner(4)
Match 11: Player H vs Winner(6)
Match 12: Winner(7) vs Player M

Round 3 (4 players):
Match 13: Winner(9) vs Winner(10)
Match 14: Winner(11) vs Winner(12)

Finals:
Match 15: Winner(13) vs Winner(14)
```

---

## 5. Arena System

### 5.1 Arena Configuration

```csharp
public class TournamentArena
{
    public int ArenaNumber { get; set; }
    public Point3D FightLocation1 { get; set; }  // Player 1 spawn
    public Point3D FightLocation2 { get; set; }  // Player 2 spawn
    public Rectangle2D ArenaBounds { get; set; } // Fight area boundary
    public bool IsOccupied { get; set; }
    public TournamentMatch CurrentMatch { get; set; }
    
    // 8 arenas initially, expandable
    public static readonly int InitialArenaCount = 8;
}

public class TournamentArenaManager
{
    private static List<TournamentArena> _arenas = new();
    
    public static void InitializeArenas()
    {
        // Create 8 arenas in the Tournament Stadium
        // Layout: 2 rows of 4 arenas
        for (int i = 0; i < TournamentArena.InitialArenaCount; i++)
        {
            _arenas.Add(new TournamentArena
            {
                ArenaNumber = i + 1,
                FightLocation1 = CalculateArenaSpawn1(i),
                FightLocation2 = CalculateArenaSpawn2(i),
                ArenaBounds = CalculateArenaBounds(i),
                IsOccupied = false
            });
        }
    }
    
    public static TournamentArena GetAvailableArena()
    {
        return _arenas.FirstOrDefault(a => !a.IsOccupied);
    }
    
    public static void ReleaseArena(TournamentArena arena)
    {
        arena.IsOccupied = false;
        arena.CurrentMatch = null;
    }
}
```

### 5.2 Stadium Layout

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TOURNAMENT STADIUM LAYOUT                             │
└─────────────────────────────────────────────────────────────────────────┘

                         SPECTATOR SEATING (NORTH)
    ┌─────────────────────────────────────────────────────────────────┐
    │                                                                  │
    │   ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐               │
    │   │Arena 1 │  │Arena 2 │  │Arena 3 │  │Arena 4 │               │
    │   │        │  │        │  │        │  │        │               │
    │   └────────┘  └────────┘  └────────┘  └────────┘               │
S   │                                                                  │   S
P   │   ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐               │   P
E   │   │Arena 5 │  │Arena 6 │  │Arena 7 │  │Arena 8 │               │   E
C   │   │        │  │        │  │        │  │        │               │   C
T   │   └────────┘  └────────┘  └────────┘  └────────┘               │   T
A   │                                                                  │   A
T   │                      WAITING AREA                                │   T
O   │                  ┌─────────────────────┐                        │   O
R   │                  │  Registered Players │                        │   R
S   │                  │  Wait Here Between  │                        │   S
    │                  │      Rounds         │                        │
(W) │                  └─────────────────────┘                        │ (E)
    │                                                                  │
    │   ┌──────────────┐              ┌──────────────┐                │
    │   │ Tournament   │              │   Betting    │                │
    │   │  Registrar   │              │    NPC       │                │
    │   └──────────────┘              └──────────────┘                │
    │                                                                  │
    │                        ENTRANCE (SOUTH)                          │
    └─────────────────────────────────────────────────────────────────┘
```

### 5.3 Arena Regions

```csharp
public class TournamentArenaRegion : BaseRegion
{
    private readonly int _arenaNumber;
    
    public override bool OnBeginSpellCast(Mobile m, ISpell s)
    {
        // Allow all spells in tournament
        return true;
    }
    
    public override bool AllowHousing(Mobile from, Point3D p) => false;
    
    public override void OnEnter(Mobile m)
    {
        if (m is PlayerMobile pm)
        {
            // Check if player belongs in this arena
            var match = TournamentManager.GetCurrentMatch(_arenaNumber);
            if (match != null && !match.IsParticipant(pm) && !IsSpectatorArea(pm.Location))
            {
                // Bounce out non-participants
                pm.SendMessage(0x22, "You are not a participant in this match!");
                Timer.StartTimer(TimeSpan.FromMilliseconds(500), () =>
                {
                    pm.MoveToWorld(WaitingAreaLocation, Map.Felucca);
                });
            }
        }
    }
    
    public override bool OnDeath(Mobile m)
    {
        if (m is PlayerMobile pm)
        {
            // Handle tournament death
            var match = TournamentManager.GetCurrentMatch(_arenaNumber);
            if (match != null && match.IsParticipant(pm))
            {
                TournamentManager.OnFighterDeath(match, pm);
            }
        }
        
        return true; // Allow death
    }
}
```

---

## 6. Match Processing

### 6.1 Match Lifecycle

```csharp
public class TournamentMatch
{
    public int Round { get; set; }
    public int MatchNumber { get; set; }
    public TournamentSlot Slot1 { get; set; }
    public TournamentSlot Slot2 { get; set; }
    public TournamentSlot Winner { get; set; }
    public bool IsComplete { get; set; }
    public bool InProgress { get; set; }
    public TournamentArena Arena { get; set; }
    public DateTime StartTime { get; set; }
    
    public bool IsParticipant(PlayerMobile player)
    {
        return Slot1.Player?.Serial == player.Serial || 
               Slot2.Player?.Serial == player.Serial;
    }
}

public class TournamentMatchProcessor
{
    public static void StartMatch(TournamentMatch match, TournamentArena arena)
    {
        match.Arena = arena;
        match.InProgress = true;
        match.StartTime = DateTime.UtcNow;
        arena.IsOccupied = true;
        arena.CurrentMatch = match;
        
        var player1 = match.Slot1.Player;
        var player2 = match.Slot2.Player;
        
        // Refresh both players (full heal, cure, remove debuffs)
        RefreshPlayer(player1);
        RefreshPlayer(player2);
        
        // Teleport to arena positions
        player1.MoveToWorld(arena.FightLocation1, Map.Felucca);
        player2.MoveToWorld(arena.FightLocation2, Map.Felucca);
        
        // Brief countdown
        BroadcastToArena(arena, "Match starting in 5 seconds...");
        
        Timer.StartTimer(TimeSpan.FromSeconds(5), () =>
        {
            BroadcastToArena(arena, "FIGHT!");
            
            // Enable combat (players were frozen during countdown)
            player1.Frozen = false;
            player2.Frozen = false;
        });
        
        // Announce to spectators
        BroadcastToSpectators($"Arena {arena.ArenaNumber}: {player1.Name} vs {player2.Name}");
    }
    
    private static void RefreshPlayer(PlayerMobile player)
    {
        // Full heal
        player.Hits = player.HitsMax;
        player.Mana = player.ManaMax;
        player.Stam = player.StamMax;
        
        // Cure poison
        player.CurePoison(player);
        
        // Remove debuffs
        player.Paralyzed = false;
        player.Frozen = true; // Freeze until countdown ends
        
        // Clear combat state
        player.Combatant = null;
        player.Warmode = false;
        
        // Remove active spells (optional - discuss)
        // RemoveActiveSpells(player);
    }
    
    public static void OnFighterDeath(TournamentMatch match, PlayerMobile loser)
    {
        var winner = match.Slot1.Player?.Serial == loser.Serial 
            ? match.Slot2 
            : match.Slot1;
            
        match.Winner = winner;
        match.IsComplete = true;
        match.InProgress = false;
        
        // Announce result
        BroadcastToArena(match.Arena, $"{winner.Player.Name} wins!");
        World.Broadcast(0x35, true, 
            $"Tournament: {winner.Player.Name} defeats {loser.Name} in Round {match.Round}!");
        
        // Resurrect loser
        Timer.StartTimer(TimeSpan.FromSeconds(2), () =>
        {
            loser.Resurrect();
            RefreshPlayer(loser);
            
            // Send loser to spectator area
            loser.MoveToWorld(SpectatorAreaLocation, Map.Felucca);
            loser.SendMessage(0x22, "You have been eliminated. You may watch from the spectator area.");
        });
        
        // Award faction points for kill (Glicko weighted)
        AwardKillPoints(winner.Player, loser);
        
        // Move winner to waiting area
        Timer.StartTimer(TimeSpan.FromSeconds(3), () =>
        {
            RefreshPlayer(winner.Player);
            winner.Player.MoveToWorld(WaitingAreaLocation, Map.Felucca);
            winner.Player.SendMessage(0x35, "Victory! Wait here for your next match.");
        });
        
        // Release arena
        TournamentArenaManager.ReleaseArena(match.Arena);
        
        // Advance bracket
        TournamentManager.AdvanceBracket(match);
    }
    
    private static void AwardKillPoints(PlayerMobile winner, PlayerMobile loser)
    {
        // Use Glicko weighting for faction points
        if (winner.Guild?.Faction != null)
        {
            int basePoints = 100;
            double multiplier = FactionPointManager.CalculateGlickoMultiplier(winner, loser);
            int points = (int)(basePoints * multiplier);
            
            FactionPointManager.AwardPoints(winner, points, $"Tournament Kill: {loser.Name}");
        }
        
        // Update Glicko ratings
        GlickoManager.RecordMatch(winner, loser, winner);
    }
}
```

### 6.2 Disconnect Handling

```csharp
public class TournamentDisconnectHandler
{
    // Character stays in game - opponent can finish them or wait
    
    public static void OnPlayerDisconnect(PlayerMobile player)
    {
        var match = TournamentManager.GetActiveMatchForPlayer(player);
        if (match == null)
            return;
            
        var opponent = match.GetOpponent(player);
        
        // Notify opponent
        opponent.SendMessage(0x22, $"{player.Name} has disconnected. You may finish them or wait for reconnection.");
        
        // Character remains in arena - no special handling
        // Combat continues normally
        // If opponent kills disconnected player, they win
        // If player reconnects, they continue fighting
    }
    
    public static void OnPlayerReconnect(PlayerMobile player)
    {
        var match = TournamentManager.GetActiveMatchForPlayer(player);
        if (match == null)
            return;
            
        // Check if still in arena
        if (!match.Arena.ArenaBounds.Contains(player.Location))
        {
            // They were killed while disconnected
            return;
        }
        
        player.SendMessage(0x35, "Welcome back! Your match is still in progress.");
    }
}
```

---

## 7. Round Management

### 7.1 Round Processing

```csharp
public class TournamentRoundManager
{
    public static async Task ProcessRound(TournamentBracket bracket, int roundNumber)
    {
        var roundMatches = bracket.GetMatchesForRound(roundNumber)
            .Where(m => !m.IsComplete)
            .ToList();
            
        // Announce round start
        World.Broadcast(0x35, true, 
            $"Tournament Round {roundNumber} beginning! {roundMatches.Count} matches to fight.");
        
        // Process matches in parallel using available arenas
        while (roundMatches.Any(m => !m.IsComplete))
        {
            var pendingMatches = roundMatches.Where(m => !m.IsComplete && !m.InProgress);
            
            foreach (var match in pendingMatches)
            {
                var arena = TournamentArenaManager.GetAvailableArena();
                if (arena == null)
                    break; // All arenas occupied, wait
                    
                // Start this match
                TournamentMatchProcessor.StartMatch(match, arena);
            }
            
            // Check every second for completed matches
            await Task.Delay(1000);
        }
        
        // Round complete
        World.Broadcast(0x35, true, $"Round {roundNumber} complete!");
        
        // Brief pause before next round (players auto-refreshed when teleported to waiting area)
        if (roundNumber < bracket.TotalRounds)
        {
            World.Broadcast(0x35, true, "Next round begins in 30 seconds...");
            await Task.Delay(TimeSpan.FromSeconds(30));
        }
    }
}
```

### 7.2 Tournament Flow

```csharp
public class TournamentManager
{
    private static TournamentBracket _currentBracket;
    private static TournamentState _state = TournamentState.Idle;
    
    public static async Task RunTournament()
    {
        // Validate minimum players
        if (_registeredPlayers.Count < 6)
        {
            World.Broadcast(0x22, true, 
                "Tournament cancelled - not enough players (minimum 6 required).");
            _state = TournamentState.Idle;
            return;
        }
        
        _state = TournamentState.Running;
        
        // Generate bracket
        var participants = GetRegisteredPlayers();
        _currentBracket = TournamentBracket.Generate(participants);
        
        // Announce bracket
        World.Broadcast(0x35, true, 
            $"Tournament starting with {participants.Count} players! {_currentBracket.TotalRounds} rounds.");
        
        // Teleport all participants to waiting area
        foreach (var player in participants)
        {
            TournamentMatchProcessor.RefreshPlayer(player);
            player.MoveToWorld(WaitingAreaLocation, Map.Felucca);
            player.SendMessage(0x35, "Welcome to the tournament! Wait here for your matches.");
        }
        
        // Process each round
        for (int round = 1; round <= _currentBracket.TotalRounds; round++)
        {
            await TournamentRoundManager.ProcessRound(_currentBracket, round);
        }
        
        // Tournament complete - award winner
        var winner = _currentBracket.GetWinner();
        await AwardWinner(winner);
        
        _state = TournamentState.Idle;
    }
    
    public static void AdvanceBracket(TournamentMatch completedMatch)
    {
        // Find the next round match this winner advances to
        int nextRound = completedMatch.Round + 1;
        if (nextRound > _currentBracket.TotalRounds)
            return; // This was the finals
            
        int nextMatchNumber = completedMatch.MatchNumber / 2;
        var nextMatch = _currentBracket.GetMatch(nextRound, nextMatchNumber);
        
        // Fill in the winner
        if (completedMatch.MatchNumber % 2 == 0)
            nextMatch.Slot1 = completedMatch.Winner;
        else
            nextMatch.Slot2 = completedMatch.Winner;
    }
}
```

---

## 8. Reward System

### 8.1 Winner Awards

```csharp
public class TournamentRewards
{
    public static async Task AwardWinner(PlayerMobile winner)
    {
        // Dramatic pause
        await Task.Delay(TimeSpan.FromSeconds(2));
        
        // World announcement
        World.Broadcast(0x35, true, 
            $"═══════════════════════════════════════════");
        World.Broadcast(0x35, true, 
            $"  TOURNAMENT CHAMPION: {winner.Name}!");
        World.Broadcast(0x35, true, 
            $"═══════════════════════════════════════════");
        
        // Teleport winner to center stage
        winner.MoveToWorld(CenterStageLocation, Map.Felucca);
        
        // Visual celebration
        winner.FixedParticles(0x373A, 10, 15, 5018, EffectLayer.Head);
        winner.PlaySound(0x5B5);
        
        // Award 1: Trophy Statue
        var trophy = new TournamentTrophy(winner.Name, DateTime.UtcNow, _registeredPlayers.Count);
        winner.AddToBackpack(trophy);
        winner.SendMessage(0x35, "You received a Tournament Trophy!");
        
        // Award 2: Tournament Coins
        var coins = new TournamentCoin(100);
        winner.AddToBackpack(coins);
        winner.SendMessage(0x35, "You received 100 Tournament Coins!");
        
        // Award 3: Temporary Title
        winner.TournamentTitle = "Tournament Champion";
        winner.TournamentTitleExpiry = DateTime.UtcNow.AddDays(7);
        winner.TournamentTitleEnabled = true;
        winner.SendMessage(0x35, "You have been granted the title 'Tournament Champion' for 1 week!");
        winner.SendMessage(0x35, "Use [toggletitle to show/hide your title.");
    }
}
```

### 8.2 Tournament Trophy

```csharp
public class TournamentTrophy : Item
{
    private string _winnerName;
    private DateTime _winDate;
    private int _participantCount;
    
    [Constructable]
    public TournamentTrophy(string winnerName, DateTime winDate, int participants) : base(0x12CB) // Trophy item ID
    {
        _winnerName = winnerName;
        _winDate = winDate;
        _participantCount = participants;
        
        Name = "Tournament Champion Trophy";
        Hue = 0x501; // Gold hue
        Weight = 5.0;
    }
    
    public override void OnDoubleClick(Mobile from)
    {
        // Show scroll-style gump with tournament info
        from.SendGump(new TournamentTrophyGump(_winnerName, _winDate, _participantCount));
    }
    
    public override void GetProperties(ObjectPropertyList list)
    {
        base.GetProperties(list);
        list.Add($"Champion: {_winnerName}");
        list.Add($"Date: {_winDate:MMMM dd, yyyy}");
        list.Add($"Participants: {_participantCount}");
    }
}

public class TournamentTrophyGump : Gump
{
    public TournamentTrophyGump(string winner, DateTime date, int participants) : base(50, 50)
    {
        // Scroll-style background
        AddBackground(0, 0, 300, 250, 9380); // Scroll background
        
        AddHtml(30, 30, 240, 20, "<center><b>Tournament Victory</b></center>", false, false);
        
        AddLabel(30, 70, 0x35, "Champion:");
        AddLabel(120, 70, 0, winner);
        
        AddLabel(30, 100, 0x35, "Date:");
        AddLabel(120, 100, 0, date.ToString("MMMM dd, yyyy"));
        
        AddLabel(30, 130, 0x35, "Participants:");
        AddLabel(120, 130, 0, participants.ToString());
        
        AddLabel(30, 170, 0x35, "Rounds Fought:");
        AddLabel(120, 170, 0, ((int)Math.Ceiling(Math.Log2(participants))).ToString());
        
        AddHtml(30, 200, 240, 20, "<center><i>Glory to the Champion!</i></center>", false, false);
    }
}
```

### 8.3 Tournament Coins & Cosmetic Shop

```csharp
public class TournamentCoin : Item
{
    private int _amount;
    
    [Constructable]
    public TournamentCoin(int amount) : base(0xEF0) // Coin graphic
    {
        _amount = amount;
        Name = "Tournament Coins";
        Hue = 0x501; // Gold
        Stackable = true;
        Amount = amount;
    }
}

public class TournamentCosmeticVendor : BaseVendor
{
    public override void OnDoubleClick(Mobile from)
    {
        if (!(from is PlayerMobile pm))
            return;
            
        pm.SendGump(new TournamentShopGump(pm));
    }
}

public class TournamentShopGump : Gump
{
    public TournamentShopGump(PlayerMobile player) : base(50, 50)
    {
        int coins = player.Backpack?.GetAmount(typeof(TournamentCoin)) ?? 0;
        
        AddBackground(0, 0, 400, 400, 9200);
        AddLabel(130, 20, 0x35, "Tournament Cosmetic Shop");
        
        AddLabel(20, 50, 0, $"Your Tournament Coins: {coins}");
        
        // Cosmetic options
        AddLabel(20, 90, 0x35, "Available Cosmetics:");
        
        // Helmet → Hat conversion
        int y = 120;
        AddButton(20, y, 4005, 4007, 1, GumpButtonType.Reply, 0);
        AddLabel(55, y, 0, "Helmet → Hat (100 coins)");
        AddLabel(55, y + 20, 0x3B2, "Transform any helmet into a stylish hat");
        
        y += 50;
        AddButton(20, y, 4005, 4007, 2, GumpButtonType.Reply, 0);
        AddLabel(55, y, 0, "Helmet → Mask (100 coins)");
        AddLabel(55, y + 20, 0x3B2, "Transform any helmet into a mysterious mask");
        
        // Future cosmetics
        y += 50;
        AddLabel(20, y, 0x22, "More cosmetics coming soon...");
    }
    
    public override void OnResponse(NetState sender, RelayInfo info)
    {
        var pm = sender.Mobile as PlayerMobile;
        
        switch (info.ButtonID)
        {
            case 1: // Helmet → Hat
            case 2: // Helmet → Mask
                pm.SendMessage(0x35, "Target the helmet you wish to transform.");
                pm.Target = new CosmeticTransformTarget(info.ButtonID == 1 ? CosmeticType.Hat : CosmeticType.Mask);
                break;
        }
    }
}

public class CosmeticTransformTarget : Target
{
    private CosmeticType _targetType;
    
    public CosmeticTransformTarget(CosmeticType type) : base(12, false, TargetFlags.None)
    {
        _targetType = type;
    }
    
    protected override void OnTarget(Mobile from, object targeted)
    {
        if (!(from is PlayerMobile pm))
            return;
            
        if (!(targeted is BaseArmor helmet) || helmet.Layer != Layer.Helm)
        {
            pm.SendMessage(0x22, "You must target a helmet.");
            return;
        }
        
        // Check coins
        int coinCost = 100;
        if (!pm.Backpack.ConsumeTotal(typeof(TournamentCoin), coinCost))
        {
            pm.SendMessage(0x22, $"You need {coinCost} Tournament Coins.");
            return;
        }
        
        // Apply cosmetic transformation
        // This keeps AR but changes visual appearance
        ApplyCosmeticOverride(helmet, _targetType);
        
        pm.SendMessage(0x35, $"Your helmet has been transformed into a {_targetType}!");
        pm.PlaySound(0x5B5);
    }
    
    private void ApplyCosmeticOverride(BaseArmor helmet, CosmeticType type)
    {
        // Store original ItemID and apply cosmetic override
        helmet.CosmeticItemID = type switch
        {
            CosmeticType.Hat => GetRandomHatItemID(),
            CosmeticType.Mask => GetRandomMaskItemID(),
            _ => helmet.ItemID
        };
        
        helmet.HasCosmeticOverride = true;
        helmet.InvalidateProperties();
    }
}
```

### 8.4 Title System

```csharp
// Add to PlayerMobile.cs
public partial class PlayerMobile
{
    private string _tournamentTitle;
    private DateTime _tournamentTitleExpiry;
    private bool _tournamentTitleEnabled = true;
    
    [CommandProperty(AccessLevel.GameMaster)]
    public string TournamentTitle
    {
        get => _tournamentTitle;
        set { _tournamentTitle = value; InvalidateProperties(); }
    }
    
    [CommandProperty(AccessLevel.GameMaster)]
    public DateTime TournamentTitleExpiry
    {
        get => _tournamentTitleExpiry;
        set => _tournamentTitleExpiry = value;
    }
    
    [CommandProperty(AccessLevel.GameMaster)]
    public bool TournamentTitleEnabled
    {
        get => _tournamentTitleEnabled;
        set { _tournamentTitleEnabled = value; InvalidateProperties(); }
    }
    
    public bool HasActiveTournamentTitle => 
        !string.IsNullOrEmpty(TournamentTitle) && 
        DateTime.UtcNow < TournamentTitleExpiry &&
        TournamentTitleEnabled;
}

// Title toggle command
public class ToggleTitleCommand
{
    [Usage("[toggletitle")]
    [Description("Toggle your tournament title on/off")]
    public static void ToggleTitle(CommandEventArgs e)
    {
        if (e.Mobile is PlayerMobile pm)
        {
            if (string.IsNullOrEmpty(pm.TournamentTitle))
            {
                pm.SendMessage(0x22, "You don't have a tournament title.");
                return;
            }
            
            pm.TournamentTitleEnabled = !pm.TournamentTitleEnabled;
            pm.SendMessage(0x35, 
                pm.TournamentTitleEnabled 
                    ? "Your tournament title is now visible." 
                    : "Your tournament title is now hidden.");
        }
    }
}

// In name display (GetNameProperties or similar)
public override void GetProperties(ObjectPropertyList list)
{
    base.GetProperties(list);
    
    if (HasActiveTournamentTitle)
    {
        list.Add(1060658, $"Title\t{TournamentTitle}"); // Shows as "Title: Tournament Champion"
    }
}
```

---

## 9. Spectator Betting System

### 9.1 Betting NPC

```csharp
public class TournamentBettingNPC : BaseVendor
{
    public override void OnDoubleClick(Mobile from)
    {
        if (!(from is PlayerMobile pm))
            return;
            
        if (TournamentManager.State != TournamentState.Running)
        {
            Say("There is no tournament in progress. Come back during a tournament to place bets!");
            return;
        }
        
        pm.SendGump(new TournamentBettingGump(pm));
    }
}

public class TournamentBettingGump : Gump
{
    public TournamentBettingGump(PlayerMobile player) : base(50, 50)
    {
        var activeMatches = TournamentManager.GetPendingMatches();
        
        AddBackground(0, 0, 500, 400, 9200);
        AddLabel(180, 20, 0x35, "Tournament Betting");
        
        AddLabel(20, 50, 0, "Active & Upcoming Matches:");
        
        int y = 80;
        int buttonId = 1;
        
        foreach (var match in activeMatches.Take(10))
        {
            var player1Name = match.Slot1.Player?.Name ?? "TBD";
            var player2Name = match.Slot2.Player?.Name ?? "TBD";
            
            AddLabel(20, y, 0, $"Round {match.Round}: {player1Name} vs {player2Name}");
            
            if (!match.InProgress && !match.IsComplete && match.Slot1.Player != null && match.Slot2.Player != null)
            {
                // Can bet on this match
                AddButton(380, y, 4005, 4007, buttonId, GumpButtonType.Reply, 0);
                AddLabel(415, y, 0, "Bet");
            }
            else if (match.InProgress)
            {
                AddLabel(380, y, 0x22, "In Progress");
            }
            
            y += 25;
            buttonId++;
        }
        
        // Show current bets
        var playerBets = TournamentBettingManager.GetPlayerBets(player);
        if (playerBets.Any())
        {
            y += 20;
            AddLabel(20, y, 0x35, "Your Active Bets:");
            y += 25;
            
            foreach (var bet in playerBets)
            {
                AddLabel(20, y, 0, $"{bet.Amount}g on {bet.TargetPlayer.Name}");
                y += 20;
            }
        }
    }
}
```

### 9.2 Betting Logic

```csharp
public static class TournamentBettingManager
{
    private static Dictionary<TournamentMatch, List<TournamentBet>> _matchBets = new();
    private const double HouseRake = 0.05; // 5% house take
    
    public class TournamentBet
    {
        public PlayerMobile Bettor { get; set; }
        public PlayerMobile TargetPlayer { get; set; }
        public int Amount { get; set; }
        public TournamentMatch Match { get; set; }
    }
    
    public static BetResult PlaceBet(PlayerMobile bettor, TournamentMatch match, PlayerMobile target, int amount)
    {
        // Validate match state
        if (match.InProgress || match.IsComplete)
            return BetResult.MatchAlreadyStarted;
            
        // Can't bet on own match
        if (match.IsParticipant(bettor))
            return BetResult.CannotBetOnSelf;
            
        // Validate target is in match
        if (!match.IsParticipant(target))
            return BetResult.InvalidTarget;
            
        // Check gold
        if (bettor.BankBox.TotalGold < amount)
            return BetResult.InsufficientGold;
            
        // Minimum bet
        if (amount < 1000)
            return BetResult.BetTooSmall;
            
        // Take gold
        bettor.BankBox.ConsumeTotal(typeof(Gold), amount);
        
        // Record bet
        var bet = new TournamentBet
        {
            Bettor = bettor,
            TargetPlayer = target,
            Amount = amount,
            Match = match
        };
        
        if (!_matchBets.ContainsKey(match))
            _matchBets[match] = new List<TournamentBet>();
            
        _matchBets[match].Add(bet);
        
        bettor.SendMessage(0x35, $"You bet {amount}g on {target.Name}!");
        
        return BetResult.Success;
    }
    
    public static void ResolveMatchBets(TournamentMatch match)
    {
        if (!_matchBets.TryGetValue(match, out var bets) || !bets.Any())
            return;
            
        var winner = match.Winner.Player;
        
        // Calculate total pools
        int winnerPool = bets.Where(b => b.TargetPlayer == winner).Sum(b => b.Amount);
        int loserPool = bets.Where(b => b.TargetPlayer != winner).Sum(b => b.Amount);
        int totalPool = winnerPool + loserPool;
        
        // House takes cut from losers
        int houseCut = (int)(loserPool * HouseRake);
        int payoutPool = loserPool - houseCut;
        
        // Distribute winnings proportionally
        foreach (var bet in bets.Where(b => b.TargetPlayer == winner))
        {
            // Original bet back + proportional share of payout pool
            double sharePercent = (double)bet.Amount / winnerPool;
            int winnings = bet.Amount + (int)(payoutPool * sharePercent);
            
            bet.Bettor.BankBox.DropItem(new Gold(winnings));
            bet.Bettor.SendMessage(0x35, $"Your bet won! You received {winnings}g!");
        }
        
        // Notify losers
        foreach (var bet in bets.Where(b => b.TargetPlayer != winner))
        {
            bet.Bettor.SendMessage(0x22, $"Your bet on {bet.TargetPlayer.Name} lost.");
        }
        
        // Gold sink logging
        EconomyTracker.LogGoldSink("TournamentBetting", houseCut);
        
        // Clean up
        _matchBets.Remove(match);
    }
}
```

---

## 10. Tournament Scheduling

### 10.1 Scheduler

```csharp
public static class TournamentScheduler
{
    private static Timer _schedulerTimer;
    
    // Tournament times (EST)
    private static readonly TimeSpan[] TournamentTimes = new[]
    {
        TimeSpan.FromHours(14), // 2 PM
        TimeSpan.FromHours(20)  // 8 PM
    };
    
    private static readonly DayOfWeek[] TournamentDays = new[]
    {
        DayOfWeek.Wednesday,
        DayOfWeek.Saturday,
        DayOfWeek.Sunday
    };
    
    public static void Initialize()
    {
        // Check every minute for upcoming tournaments
        _schedulerTimer = Timer.StartTimer(TimeSpan.FromMinutes(1), TimeSpan.FromMinutes(1), CheckSchedule);
    }
    
    private static void CheckSchedule()
    {
        var now = DateTime.UtcNow;
        var estNow = TimeZoneInfo.ConvertTimeFromUtc(now, TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time"));
        
        // Check if today is a tournament day
        if (!TournamentDays.Contains(estNow.DayOfWeek))
            return;
            
        // Check each tournament time
        foreach (var tournamentTime in TournamentTimes)
        {
            var registrationOpenTime = tournamentTime - TimeSpan.FromMinutes(15);
            
            // Check if we should open registration (within 1 minute of registration time)
            if (Math.Abs((estNow.TimeOfDay - registrationOpenTime).TotalMinutes) < 1)
            {
                if (TournamentManager.State == TournamentState.Idle)
                {
                    var tournamentDateTime = estNow.Date + tournamentTime;
                    TournamentRegistration.OpenRegistration(tournamentDateTime);
                }
            }
            
            // Check if we should start tournament (within 1 minute of tournament time)
            if (Math.Abs((estNow.TimeOfDay - tournamentTime).TotalMinutes) < 1)
            {
                if (TournamentManager.State == TournamentState.Registration)
                {
                    _ = TournamentManager.RunTournament();
                }
            }
        }
    }
    
    public static DateTime GetNextTournamentTime()
    {
        var now = DateTime.UtcNow;
        var estNow = TimeZoneInfo.ConvertTimeFromUtc(now, TimeZoneInfo.FindSystemTimeZoneById("Eastern Standard Time"));
        
        // Find next tournament
        for (int daysAhead = 0; daysAhead < 7; daysAhead++)
        {
            var checkDate = estNow.Date.AddDays(daysAhead);
            
            if (!TournamentDays.Contains(checkDate.DayOfWeek))
                continue;
                
            foreach (var time in TournamentTimes.OrderBy(t => t))
            {
                var tournamentDateTime = checkDate + time;
                if (tournamentDateTime > estNow)
                    return tournamentDateTime;
            }
        }
        
        return estNow.AddDays(7); // Fallback
    }
}
```

### 10.2 Admin Commands

```csharp
public class TournamentAdminCommands
{
    [Usage("[tournament start")]
    [Description("Immediately start a tournament")]
    public static void ForceStart(CommandEventArgs e)
    {
        if (TournamentManager.State != TournamentState.Idle)
        {
            e.Mobile.SendMessage(0x22, "A tournament is already in progress.");
            return;
        }
        
        TournamentRegistration.OpenRegistration(DateTime.UtcNow.AddMinutes(15));
        e.Mobile.SendMessage(0x35, "Tournament registration opened. Tournament starts in 15 minutes.");
    }
    
    [Usage("[tournament cancel")]
    [Description("Cancel current tournament")]
    public static void Cancel(CommandEventArgs e)
    {
        TournamentManager.CancelTournament();
        e.Mobile.SendMessage(0x35, "Tournament cancelled.");
    }
    
    [Usage("[tournament status")]
    [Description("Show tournament status")]
    public static void Status(CommandEventArgs e)
    {
        e.Mobile.SendMessage(0x35, $"State: {TournamentManager.State}");
        e.Mobile.SendMessage(0x35, $"Registered: {TournamentManager.RegisteredCount}");
        e.Mobile.SendMessage(0x35, $"Next Tournament: {TournamentScheduler.GetNextTournamentTime():g}");
    }
    
    [Usage("[tournament coins <player> <amount>")]
    [Description("Give tournament coins to a player")]
    public static void GiveCoins(CommandEventArgs e)
    {
        // Admin command to award coins
    }
}
```

---

## 11. Integration Points

### 11.1 Glicko Integration

- **Selection**: NOT used (random matching)
- **Kill Points**: Used to weight faction point rewards
- **Post-Tournament**: Match results update Glicko ratings
- **Future**: May be used for team balancing in CTF/KotH modes

### 11.2 Faction Integration

- Tournament kills award faction points (Glicko weighted)
- Tournament participation does NOT require faction membership
- Anyone can enter (except Young players)

### 11.3 Town Cryer Integration

- Announces registration opening (15 min before)
- Announces tournament start
- Announces winner

---

## 12. Configuration Reference

```csharp
public static class TournamentConfig
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
```

---

## 13. Testing Checklist

### Registration
- [ ] Registration opens 15 min before tournament
- [ ] Registration closes at tournament start
- [ ] Young players cannot register
- [ ] Dead players cannot register
- [ ] Players can withdraw before start
- [ ] Minimum 6 players enforced
- [ ] Town Cryer announces registration

### Bracket
- [ ] Random shuffle works correctly
- [ ] Byes calculated correctly
- [ ] Byes distributed in Round 1 only
- [ ] Bracket advances correctly

### Matches
- [ ] Players teleported to correct arenas
- [ ] Players refreshed on teleport
- [ ] Fight starts after countdown
- [ ] Death properly triggers match end
- [ ] Winner advances to next round
- [ ] Loser resurrected and moved to spectator area
- [ ] Disconnect handling works (character stays)

### Rewards
- [ ] Trophy awarded to winner
- [ ] Trophy displays correct info
- [ ] 100 coins awarded to winner
- [ ] Title granted and displays
- [ ] Title toggle command works
- [ ] Title expires after 1 week

### Betting
- [ ] Can bet on pending matches
- [ ] Cannot bet on started matches
- [ ] Cannot bet on own matches
- [ ] House rake calculated correctly
- [ ] Winners paid correctly
- [ ] Losers notified

### Scheduling
- [ ] Tournaments run on correct days
- [ ] Tournaments run at correct times
- [ ] Multiple tournaments per day work

---

## 14. Future Enhancements

| Enhancement | Priority | Notes |
|-------------|----------|-------|
| 2v2 Tournaments | Medium | Separate bracket system |
| Seasonal Championships | Low | End-of-season tournament |
| More Cosmetic Options | Medium | Additional coin purchases |
| Spectator Mode | Low | First-person viewing |
| Tournament History | Low | Database of past winners |
| Bracket Visualization | Medium | Web display of live brackets |

---

## 15. Change Log

### v1.0.0 - 2025-01-02 (Initial Specification)
- Created comprehensive tournament system specification
- Random selection (no seeding/Glicko for matching)
- 6 minimum players, uncapped maximum
- Bye system for uneven numbers
- 8 parallel arenas
- Free entry, fixed prizes (Trophy + 100 Coins)
- Tournament Coin cosmetic currency system
- Spectator betting with 5% house rake
- 7-day temporary title for winners
- Scheduled Wed/Sat/Sun at 2 PM and 8 PM EST