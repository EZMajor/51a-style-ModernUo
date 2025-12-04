# 51alpha World Locations
## Felucca Map Coordinates Reference

### Document Metadata
- **Version**: v1.0.0
- **Last Updated**: 2025-01-02
- **Map**: Felucca (Map ID: 0)
- **Reference**: https://uo.com/wiki/ultima-online-wiki/world/facet-maps/

---

## 1. Overview

All 51alpha systems operate on the Felucca facet only. This document provides exact coordinates for all spawn points, regions, and NPC placements.

### Coordinate Format
```
(X, Y, Z) - Map: Felucca (0)
```

### Map Bounds
- Felucca: 0-7167 X, 0-4095 Y

---

## 2. Siege Cities

### 2.1 Jhelom

**City Center**: (1417, 3821, 0)

| Location | Coordinates | Notes |
|----------|-------------|-------|
| **Siege Region** | (1330, 3700) to (1530, 3900) | 200x200 area |
| **Sigil Spawn** | (1417, 3821, 0) | Center of main island |
| **Altar 1** | (1375, 3780, 0) | Northwest |
| **Altar 2** | (1460, 3780, 0) | Northeast |
| **Altar 3** | (1375, 3860, 0) | Southwest |
| **Altar 4** | (1460, 3860, 0) | Southeast |
| **Priest - Vampire** | (1350, 3750, 0) | Red robes, NW corner |
| **Priest - Daemon** | (1485, 3750, 0) | Orange robes, NE corner |
| **Priest - Goblin** | (1417, 3890, 0) | Green robes, South |
| **Banner 1** | (1417, 3800, 0) | North of sigil |
| **Banner 2** | (1400, 3821, 0) | West of sigil |
| **Banner 3** | (1435, 3821, 0) | East of sigil |
| **Banner 4** | (1417, 3845, 0) | South of sigil |
| **Bounty Board** | (1383, 3815, 0) | Near bank |
| **Silver Vendor** | (1385, 3815, 0) | Next to bounty board |

```csharp
// Jhelom siege region
public static readonly Rectangle2D JhelomRegion = new Rectangle2D(1330, 3700, 200, 200);
```

---

### 2.2 Skara Brae

**City Center**: (596, 2138, 0)

| Location | Coordinates | Notes |
|----------|-------------|-------|
| **Siege Region** | (530, 2070) to (700, 2220) | 170x150 area |
| **Sigil Spawn** | (596, 2138, 0) | Town center |
| **Altar 1** | (555, 2100, 0) | Northwest by rangers |
| **Altar 2** | (640, 2100, 0) | Northeast |
| **Altar 3** | (555, 2175, 0) | Southwest by docks |
| **Altar 4** | (640, 2175, 0) | Southeast |
| **Priest - Vampire** | (540, 2085, 0) | NW entrance |
| **Priest - Daemon** | (655, 2085, 0) | NE area |
| **Priest - Goblin** | (596, 2200, 0) | South docks |
| **Banner 1** | (596, 2120, 0) | North |
| **Banner 2** | (575, 2138, 0) | West |
| **Banner 3** | (617, 2138, 0) | East |
| **Banner 4** | (596, 2158, 0) | South |
| **Bounty Board** | (602, 2154, 0) | Near bank |
| **Silver Vendor** | (604, 2154, 0) | Next to board |

```csharp
public static readonly Rectangle2D SkaraBraeRegion = new Rectangle2D(530, 2070, 170, 150);
```

---

### 2.3 Yew

**City Center**: (542, 985, 0)

| Location | Coordinates | Notes |
|----------|-------------|-------|
| **Siege Region** | (450, 900) to (650, 1100) | 200x200 area |
| **Sigil Spawn** | (542, 985, 0) | Abbey courtyard |
| **Altar 1** | (490, 940, 0) | Northwest woods |
| **Altar 2** | (595, 940, 0) | Northeast |
| **Altar 3** | (490, 1030, 0) | Southwest |
| **Altar 4** | (595, 1030, 0) | Southeast near courts |
| **Priest - Vampire** | (465, 920, 0) | Deep NW |
| **Priest - Daemon** | (620, 920, 0) | NE woods |
| **Priest - Goblin** | (542, 1060, 0) | South of abbey |
| **Banner 1** | (542, 965, 0) | North |
| **Banner 2** | (520, 985, 0) | West |
| **Banner 3** | (565, 985, 0) | East |
| **Banner 4** | (542, 1005, 0) | South |
| **Bounty Board** | (548, 992, 0) | Near empath abbey |
| **Silver Vendor** | (550, 992, 0) | Next to board |

```csharp
public static readonly Rectangle2D YewRegion = new Rectangle2D(450, 900, 200, 200);
```

---

### 2.4 Trinsic

**City Center**: (1867, 2780, 0)

| Location | Coordinates | Notes |
|----------|-------------|-------|
| **Siege Region** | (1800, 2680) to (2000, 2880) | 200x200 area |
| **Sigil Spawn** | (1867, 2780, 0) | Center plaza |
| **Altar 1** | (1830, 2720, 0) | Northwest |
| **Altar 2** | (1905, 2720, 0) | Northeast |
| **Altar 3** | (1830, 2840, 0) | Southwest |
| **Altar 4** | (1905, 2840, 0) | Southeast |
| **Priest - Vampire** | (1815, 2700, 0) | NW gate area |
| **Priest - Daemon** | (1920, 2700, 0) | NE area |
| **Priest - Goblin** | (1867, 2865, 0) | South gate |
| **Banner 1** | (1867, 2760, 0) | North |
| **Banner 2** | (1845, 2780, 0) | West |
| **Banner 3** | (1890, 2780, 0) | East |
| **Banner 4** | (1867, 2800, 0) | South |
| **Bounty Board** | (1897, 2772, 0) | Near bank |
| **Silver Vendor** | (1899, 2772, 0) | Next to board |

```csharp
public static readonly Rectangle2D TrinsicRegion = new Rectangle2D(1800, 2680, 200, 200);
```

---

## 3. Tournament System

### 3.1 Tournament Stadium (Britain)

**Location**: East Britain, near the large coliseum area

**Stadium Center**: (1500, 1620, 10)

| Location | Coordinates | Notes |
|----------|-------------|-------|
| **Registration NPC** | (1495, 1580, 10) | North entrance |
| **Waiting Area** | (1450, 1580, 10) to (1550, 1600, 10) | 100x20 area |
| **Spectator Area** | (1480, 1640, 10) to (1520, 1660, 10) | South stands |
| **Trophy Display** | (1500, 1575, 10) | Behind registrar |

### 3.2 Arena Coordinates

8 arenas in a 4x2 grid, 30 tiles apart:

| Arena | Corner 1 | Corner 2 | Center | Spawn 1 | Spawn 2 |
|-------|----------|----------|--------|---------|---------|
| Arena 1 | (1420, 1600, 10) | (1440, 1620, 10) | (1430, 1610, 10) | (1425, 1610, 10) | (1435, 1610, 10) |
| Arena 2 | (1450, 1600, 10) | (1470, 1620, 10) | (1460, 1610, 10) | (1455, 1610, 10) | (1465, 1610, 10) |
| Arena 3 | (1480, 1600, 10) | (1500, 1620, 10) | (1490, 1610, 10) | (1485, 1610, 10) | (1495, 1610, 10) |
| Arena 4 | (1510, 1600, 10) | (1530, 1620, 10) | (1520, 1610, 10) | (1515, 1610, 10) | (1525, 1610, 10) |
| Arena 5 | (1420, 1630, 10) | (1440, 1650, 10) | (1430, 1640, 10) | (1425, 1640, 10) | (1435, 1640, 10) |
| Arena 6 | (1450, 1630, 10) | (1470, 1650, 10) | (1460, 1640, 10) | (1455, 1640, 10) | (1465, 1640, 10) |
| Arena 7 | (1480, 1630, 10) | (1500, 1650, 10) | (1490, 1640, 10) | (1485, 1640, 10) | (1495, 1640, 10) |
| Arena 8 | (1510, 1630, 10) | (1530, 1650, 10) | (1520, 1640, 10) | (1515, 1640, 10) | (1525, 1640, 10) |

```csharp
public static readonly TournamentArena[] Arenas = new[]
{
    new TournamentArena(1, new Point3D(1430, 1610, 10), new Point3D(1425, 1610, 10), new Point3D(1435, 1610, 10)),
    new TournamentArena(2, new Point3D(1460, 1610, 10), new Point3D(1455, 1610, 10), new Point3D(1465, 1610, 10)),
    new TournamentArena(3, new Point3D(1490, 1610, 10), new Point3D(1485, 1610, 10), new Point3D(1495, 1610, 10)),
    new TournamentArena(4, new Point3D(1520, 1610, 10), new Point3D(1515, 1610, 10), new Point3D(1525, 1610, 10)),
    new TournamentArena(5, new Point3D(1430, 1640, 10), new Point3D(1425, 1640, 10), new Point3D(1435, 1640, 10)),
    new TournamentArena(6, new Point3D(1460, 1640, 10), new Point3D(1455, 1640, 10), new Point3D(1465, 1640, 10)),
    new TournamentArena(7, new Point3D(1490, 1640, 10), new Point3D(1485, 1640, 10), new Point3D(1495, 1640, 10)),
    new TournamentArena(8, new Point3D(1520, 1640, 10), new Point3D(1515, 1640, 10), new Point3D(1525, 1640, 10))
};
```

---

## 4. Daily Faction Quest Locations

### 4.1 Swamp Locations

**Fens of the Dead** (Northwest of Britain)
| Location | Coordinates | Notes |
|----------|-------------|-------|
| **Boss Spawn** | (5765, 3190, 0) | Open swamp area |
| **Region** | (5700, 3130) to (5830, 3250) | 130x120 |

**Bog of Desolation** (South of Wrong dungeon)
| Location | Coordinates | Notes |
|----------|-------------|-------|
| **Boss Spawn** | (2038, 238, 0) | Deep swamp |
| **Region** | (1980, 180) to (2100, 300) | 120x120 |

### 4.2 Desert Locations

**Scorched Sands** (East of Compassion desert)
| Location | Coordinates | Notes |
|----------|-------------|-------|
| **Boss Spawn** | (1955, 2680, 0) | Open desert |
| **Region** | (1900, 2620) to (2020, 2740) | 120x120 |

**Sun Temple Ruins** (Northwest desert near Shame)
| Location | Coordinates | Notes |
|----------|-------------|-------|
| **Boss Spawn** | (582, 1301, 0) | Ruins area |
| **Region** | (520, 1240) to (640, 1360) | 120x120 |

```csharp
public static readonly FactionQuestLocation[] QuestLocations = new[]
{
    new FactionQuestLocation(1, "Fens of the Dead", "swamp", new Point3D(5765, 3190, 0)),
    new FactionQuestLocation(2, "Bog of Desolation", "swamp", new Point3D(2038, 238, 0)),
    new FactionQuestLocation(3, "Scorched Sands", "desert", new Point3D(1955, 2680, 0)),
    new FactionQuestLocation(4, "Sun Temple Ruins", "desert", new Point3D(582, 1301, 0))
};
```

---

## 5. New Player Experience

### 5.1 Britain Starting Area

**New Player Dock**: (1496, 1629, 10) - East Britain docks

| Location | Coordinates | Notes |
|----------|-------------|-------|
| **Ferry Arrival** | (1496, 1629, 10) | Where ferry docks |
| **Welcome NPC** | (1500, 1633, 10) | Greeter NPC |
| **Quest Board** | (1504, 1633, 10) | Starter quests |
| **Skill Trainer** | (1508, 1633, 10) | Quick skill grants |

### 5.2 Training Island

**Haven-style Training Area**: (3495, 2770, 0) - Using Haven coordinates

| Location | Coordinates | Notes |
|----------|-------------|-------|
| **Arrival Point** | (3495, 2770, 0) | Ferry arrival |
| **Combat Trainer** | (3500, 2775, 0) | Melee/magic basics |
| **Target Dummies** | (3510, 2775, 0) | Practice targets |
| **Resource Area** | (3520, 2780, 0) | Mining/lumber nodes |
| **Crafting Station** | (3490, 2780, 0) | Basic crafting |
| **Quest NPC - Combat** | (3485, 2770, 0) | Combat quest giver |
| **Quest NPC - Crafting** | (3485, 2780, 0) | Crafting quest giver |
| **Quest NPC - Gathering** | (3485, 2790, 0) | Resource quest giver |
| **Ferry Return** | (3475, 2770, 0) | Back to Britain |
| **Exit Gate** | (3480, 2765, 0) | Leave training |

```csharp
public static readonly Point3D TrainingIslandArrival = new Point3D(3495, 2770, 0);
public static readonly Point3D TrainingIslandExit = new Point3D(3480, 2765, 0);
public static readonly Point3D BritainDock = new Point3D(1496, 1629, 10);
```

---

## 6. Britain Hub Locations

### 6.1 Main City NPCs

| NPC | Coordinates | Notes |
|-----|-------------|-------|
| **Main Bounty Board** | (1438, 1700, 0) | West Britain bank |
| **Main Silver Vendor** | (1440, 1700, 0) | Next to board |
| **Faction Registrar** | (1495, 1630, 10) | Near tournament |
| **Town Cryer** | (1430, 1694, 0) | Bank entrance |
| **Tournament Registrar** | (1495, 1580, 10) | Stadium north |

### 6.2 Britain Bank (West)

**Bank Entrance**: (1436, 1694, 0)

```csharp
public static readonly Point3D BritainWestBank = new Point3D(1436, 1694, 0);
```

---

## 7. Dungeon Entrances

For bounty board placement near dungeons:

| Dungeon | Entrance | Bounty Board |
|---------|----------|--------------|
| **Shame** | (511, 1565, 0) | (515, 1565, 0) |
| **Despise** | (1298, 1081, 0) | (1302, 1081, 0) |
| **Deceit** | (4111, 434, 5) | (4115, 434, 5) |
| **Destard** | (1176, 2640, 2) | (1180, 2640, 2) |
| **Wrong** | (2043, 238, 10) | (2047, 238, 10) |
| **Covetous** | (2499, 921, 0) | (2503, 921, 0) |
| **Hythloth** | (4722, 3824, 0) | (4726, 3824, 0) |
| **Fire** | (2923, 3409, 8) | (2927, 3409, 8) |
| **Ice** | (1999, 81, 4) | (2003, 81, 4) |

---

## 8. Complete Location Registry

### 8.1 C# Location Classes

```csharp
namespace Sphere51a.Locations
{
    public static class SiegeLocations
    {
        public static readonly Dictionary<string, SiegeCityData> Cities = new()
        {
            ["Jhelom"] = new SiegeCityData
            {
                Name = "Jhelom",
                Region = new Rectangle2D(1330, 3700, 200, 200),
                SigilSpawn = new Point3D(1417, 3821, 0),
                AltarSpawns = new[]
                {
                    new Point3D(1375, 3780, 0),
                    new Point3D(1460, 3780, 0),
                    new Point3D(1375, 3860, 0),
                    new Point3D(1460, 3860, 0)
                },
                PriestSpawns = new Dictionary<FactionId, Point3D>
                {
                    [FactionId.Vampire] = new Point3D(1350, 3750, 0),
                    [FactionId.Daemon] = new Point3D(1485, 3750, 0),
                    [FactionId.Goblin] = new Point3D(1417, 3890, 0)
                },
                BannerLocations = new[]
                {
                    new Point3D(1417, 3800, 0),
                    new Point3D(1400, 3821, 0),
                    new Point3D(1435, 3821, 0),
                    new Point3D(1417, 3845, 0)
                },
                BountyBoard = new Point3D(1383, 3815, 0),
                SilverVendor = new Point3D(1385, 3815, 0)
            },
            
            ["Skara Brae"] = new SiegeCityData
            {
                Name = "Skara Brae",
                Region = new Rectangle2D(530, 2070, 170, 150),
                SigilSpawn = new Point3D(596, 2138, 0),
                AltarSpawns = new[]
                {
                    new Point3D(555, 2100, 0),
                    new Point3D(640, 2100, 0),
                    new Point3D(555, 2175, 0),
                    new Point3D(640, 2175, 0)
                },
                PriestSpawns = new Dictionary<FactionId, Point3D>
                {
                    [FactionId.Vampire] = new Point3D(540, 2085, 0),
                    [FactionId.Daemon] = new Point3D(655, 2085, 0),
                    [FactionId.Goblin] = new Point3D(596, 2200, 0)
                },
                BannerLocations = new[]
                {
                    new Point3D(596, 2120, 0),
                    new Point3D(575, 2138, 0),
                    new Point3D(617, 2138, 0),
                    new Point3D(596, 2158, 0)
                },
                BountyBoard = new Point3D(602, 2154, 0),
                SilverVendor = new Point3D(604, 2154, 0)
            },
            
            ["Yew"] = new SiegeCityData
            {
                Name = "Yew",
                Region = new Rectangle2D(450, 900, 200, 200),
                SigilSpawn = new Point3D(542, 985, 0),
                AltarSpawns = new[]
                {
                    new Point3D(490, 940, 0),
                    new Point3D(595, 940, 0),
                    new Point3D(490, 1030, 0),
                    new Point3D(595, 1030, 0)
                },
                PriestSpawns = new Dictionary<FactionId, Point3D>
                {
                    [FactionId.Vampire] = new Point3D(465, 920, 0),
                    [FactionId.Daemon] = new Point3D(620, 920, 0),
                    [FactionId.Goblin] = new Point3D(542, 1060, 0)
                },
                BannerLocations = new[]
                {
                    new Point3D(542, 965, 0),
                    new Point3D(520, 985, 0),
                    new Point3D(565, 985, 0),
                    new Point3D(542, 1005, 0)
                },
                BountyBoard = new Point3D(548, 992, 0),
                SilverVendor = new Point3D(550, 992, 0)
            },
            
            ["Trinsic"] = new SiegeCityData
            {
                Name = "Trinsic",
                Region = new Rectangle2D(1800, 2680, 200, 200),
                SigilSpawn = new Point3D(1867, 2780, 0),
                AltarSpawns = new[]
                {
                    new Point3D(1830, 2720, 0),
                    new Point3D(1905, 2720, 0),
                    new Point3D(1830, 2840, 0),
                    new Point3D(1905, 2840, 0)
                },
                PriestSpawns = new Dictionary<FactionId, Point3D>
                {
                    [FactionId.Vampire] = new Point3D(1815, 2700, 0),
                    [FactionId.Daemon] = new Point3D(1920, 2700, 0),
                    [FactionId.Goblin] = new Point3D(1867, 2865, 0)
                },
                BannerLocations = new[]
                {
                    new Point3D(1867, 2760, 0),
                    new Point3D(1845, 2780, 0),
                    new Point3D(1890, 2780, 0),
                    new Point3D(1867, 2800, 0)
                },
                BountyBoard = new Point3D(1897, 2772, 0),
                SilverVendor = new Point3D(1899, 2772, 0)
            }
        };
    }
    
    public static class TournamentLocations
    {
        public static readonly Point3D RegistrationNPC = new Point3D(1495, 1580, 10);
        public static readonly Rectangle2D WaitingArea = new Rectangle2D(1450, 1580, 100, 20);
        public static readonly Rectangle2D SpectatorArea = new Rectangle2D(1480, 1640, 40, 20);
        public static readonly Point3D TrophyDisplay = new Point3D(1500, 1575, 10);
        
        public static readonly TournamentArena[] Arenas = new[]
        {
            new TournamentArena(1, new Point3D(1430, 1610, 10), new Point3D(1425, 1610, 10), new Point3D(1435, 1610, 10)),
            new TournamentArena(2, new Point3D(1460, 1610, 10), new Point3D(1455, 1610, 10), new Point3D(1465, 1610, 10)),
            new TournamentArena(3, new Point3D(1490, 1610, 10), new Point3D(1485, 1610, 10), new Point3D(1495, 1610, 10)),
            new TournamentArena(4, new Point3D(1520, 1610, 10), new Point3D(1515, 1610, 10), new Point3D(1525, 1610, 10)),
            new TournamentArena(5, new Point3D(1430, 1640, 10), new Point3D(1425, 1640, 10), new Point3D(1435, 1640, 10)),
            new TournamentArena(6, new Point3D(1460, 1640, 10), new Point3D(1455, 1640, 10), new Point3D(1465, 1640, 10)),
            new TournamentArena(7, new Point3D(1490, 1640, 10), new Point3D(1485, 1640, 10), new Point3D(1495, 1640, 10)),
            new TournamentArena(8, new Point3D(1520, 1640, 10), new Point3D(1515, 1640, 10), new Point3D(1525, 1640, 10))
        };
    }
    
    public static class FactionQuestLocations
    {
        public static readonly FactionQuestLocation[] Locations = new[]
        {
            new FactionQuestLocation(1, "Fens of the Dead", "swamp", new Point3D(5765, 3190, 0)),
            new FactionQuestLocation(2, "Bog of Desolation", "swamp", new Point3D(2038, 238, 0)),
            new FactionQuestLocation(3, "Scorched Sands", "desert", new Point3D(1955, 2680, 0)),
            new FactionQuestLocation(4, "Sun Temple Ruins", "desert", new Point3D(582, 1301, 0))
        };
    }
    
    public static class NPELocations
    {
        public static readonly Point3D BritainDock = new Point3D(1496, 1629, 10);
        public static readonly Point3D TrainingIslandArrival = new Point3D(3495, 2770, 0);
        public static readonly Point3D TrainingIslandExit = new Point3D(3480, 2765, 0);
        public static readonly Point3D WelcomeNPC = new Point3D(1500, 1633, 10);
        public static readonly Point3D QuestBoard = new Point3D(1504, 1633, 10);
    }
    
    public static class BritainLocations
    {
        public static readonly Point3D WestBank = new Point3D(1436, 1694, 0);
        public static readonly Point3D MainBountyBoard = new Point3D(1438, 1700, 0);
        public static readonly Point3D MainSilverVendor = new Point3D(1440, 1700, 0);
        public static readonly Point3D TownCryer = new Point3D(1430, 1694, 0);
        public static readonly Point3D FactionRegistrar = new Point3D(1495, 1630, 10);
    }
}
```

---

## 9. SQL Seed Data

```sql
-- Siege city configuration
INSERT INTO s51a_siege_cities (city_name, region_x1, region_y1, region_x2, region_y2, sigil_spawn, altar_spawns, priest_spawns, banner_locations) VALUES
('Jhelom', 1330, 3700, 1530, 3900, 
    '{"x":1417,"y":3821,"z":0}',
    '[{"x":1375,"y":3780,"z":0},{"x":1460,"y":3780,"z":0},{"x":1375,"y":3860,"z":0},{"x":1460,"y":3860,"z":0}]',
    '[{"faction":1,"x":1350,"y":3750,"z":0},{"faction":2,"x":1485,"y":3750,"z":0},{"faction":3,"x":1417,"y":3890,"z":0}]',
    '[{"x":1417,"y":3800,"z":0},{"x":1400,"y":3821,"z":0},{"x":1435,"y":3821,"z":0},{"x":1417,"y":3845,"z":0}]'
),
('Skara Brae', 530, 2070, 700, 2220,
    '{"x":596,"y":2138,"z":0}',
    '[{"x":555,"y":2100,"z":0},{"x":640,"y":2100,"z":0},{"x":555,"y":2175,"z":0},{"x":640,"y":2175,"z":0}]',
    '[{"faction":1,"x":540,"y":2085,"z":0},{"faction":2,"x":655,"y":2085,"z":0},{"faction":3,"x":596,"y":2200,"z":0}]',
    '[{"x":596,"y":2120,"z":0},{"x":575,"y":2138,"z":0},{"x":617,"y":2138,"z":0},{"x":596,"y":2158,"z":0}]'
),
('Yew', 450, 900, 650, 1100,
    '{"x":542,"y":985,"z":0}',
    '[{"x":490,"y":940,"z":0},{"x":595,"y":940,"z":0},{"x":490,"y":1030,"z":0},{"x":595,"y":1030,"z":0}]',
    '[{"faction":1,"x":465,"y":920,"z":0},{"faction":2,"x":620,"y":920,"z":0},{"faction":3,"x":542,"y":1060,"z":0}]',
    '[{"x":542,"y":965,"z":0},{"x":520,"y":985,"z":0},{"x":565,"y":985,"z":0},{"x":542,"y":1005,"z":0}]'
),
('Trinsic', 1800, 2680, 2000, 2880,
    '{"x":1867,"y":2780,"z":0}',
    '[{"x":1830,"y":2720,"z":0},{"x":1905,"y":2720,"z":0},{"x":1830,"y":2840,"z":0},{"x":1905,"y":2840,"z":0}]',
    '[{"faction":1,"x":1815,"y":2700,"z":0},{"faction":2,"x":1920,"y":2700,"z":0},{"faction":3,"x":1867,"y":2865,"z":0}]',
    '[{"x":1867,"y":2760,"z":0},{"x":1845,"y":2780,"z":0},{"x":1890,"y":2780,"z":0},{"x":1867,"y":2800,"z":0}]'
);

-- Faction quest locations
UPDATE s51a_faction_quest_locations SET spawn_x = 5765, spawn_y = 3190, spawn_z = 0 WHERE location_id = 1;
UPDATE s51a_faction_quest_locations SET spawn_x = 2038, spawn_y = 238, spawn_z = 0 WHERE location_id = 2;
UPDATE s51a_faction_quest_locations SET spawn_x = 1955, spawn_y = 2680, spawn_z = 0 WHERE location_id = 3;
UPDATE s51a_faction_quest_locations SET spawn_x = 582, spawn_y = 1301, spawn_z = 0 WHERE location_id = 4;
```

---

## 10. Verification Checklist

After placing items/NPCs, verify:

- [ ] All spawn points are accessible (not inside walls)
- [ ] All NPCs are not blocking pathways
- [ ] Siege regions don't overlap with guard zones incorrectly
- [ ] Training island is accessible only via ferry
- [ ] Tournament arenas have clear line of sight
- [ ] Quest boss locations are in appropriate terrain

---

## Change Log

### v1.0.0 - 2025-01-02 (Initial Coordinates)
- All 4 siege cities mapped
- Tournament stadium layout
- 4 faction quest locations
- New player experience path
- Britain hub NPCs
- Dungeon bounty boards
- Complete C# location classes
- SQL seed data