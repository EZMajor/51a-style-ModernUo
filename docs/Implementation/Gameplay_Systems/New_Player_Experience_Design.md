New Player Experience Specification
51alpha Onboarding System
Document Metadata

Version: v2.0.0
Last Updated: 2025-01-03
Authors: 51alpha Development Team
Applicable ModernUO Version: v24.0.0+
PR Ready: No (Conceptual Design)
Inspiration: UO Outlands Wiki, Imagine Nation skill progression


1. Executive Summary
The new player experience is designed to get players into meaningful gameplay quickly while teaching core mechanics. The system uses a ferry-based island quest chain in Trammel (safe from PK) that grants combat skills through simple hunting tasks, combined with a 2-week Young Player protection system and comprehensive wiki documentation.
Design Philosophy
PrincipleImplementationFast to competence~2-3 hours to GM combat skills via questsLearn by doingQuests teach mechanics through gameplaySafe learning environmentTrammel islands, no PK possibleNo AFK grindingQuest-based progression, not time-gatedEncourage social playFactions require guild membership

2. Young Player Protection System
2.1 Overview
ModernUO includes a built-in Young Player system. We extend it with the following configuration:
csharppublic static class YoungPlayerConfig
{
    // Duration of young player status
    public static TimeSpan YoungDuration = TimeSpan.FromDays(14); // 2 weeks calendar time
    
    // Young player restrictions
    public static bool CanBeAttackedByPlayers = false;
    public static bool CanAttackPlayers = false;
    public static bool CanLootPlayerCorpses = false;
    public static bool CanBeStolen = false;
    public static bool CanEnterFelucca = true; // Can enter but protected
    public static bool CanJoinGuild = true;
    public static bool CanJoinFaction = false; // Must renounce young status first
    
    // Visual indicator
    public static int YoungHue = 0x35; // Bright green text color for name
}
2.2 Young Player Rules
RuleBehaviorPvP ProtectionCannot be attacked by or attack other playersTheft ProtectionCannot be stolen fromCorpse ProtectionCorpse cannot be looted by other playersMonster CombatNormal - can fight and be killed by monstersGuild MembershipAllowed - can join guildsFaction ParticipationBlocked until young status renouncedDungeon AccessAll dungeons accessibleTradingNormal - can trade with players
2.3 Renouncing Young Status
csharppublic class YoungPlayerManager
{
    // Player can renounce at any time via command or NPC
    [Usage("[renounce")]
    public static void RenounceYoungStatus(CommandEventArgs e)
    {
        if (e.Mobile is PlayerMobile pm && pm.Young)
        {
            pm.SendGump(new ConfirmRenounceGump(pm));
        }
    }
    
    // Confirmation gump
    public class ConfirmRenounceGump : Gump
    {
        public override void OnResponse(NetState sender, RelayInfo info)
        {
            if (info.ButtonID == 1) // Confirm
            {
                var pm = sender.Mobile as PlayerMobile;
                pm.Young = false;
                pm.SendMessage(0x35, "You have renounced your Young Player status. You may now participate in faction warfare, but you are no longer protected from PvP.");
            }
        }
    }
    
    // Auto-expire after 2 weeks
    public static void CheckYoungExpiry(PlayerMobile player)
    {
        if (player.Young && DateTime.UtcNow - player.CreationTime > YoungPlayerConfig.YoungDuration)
        {
            player.Young = false;
            player.SendMessage(0x22, "Your Young Player protection has expired. You are now vulnerable to PvP combat.");
        }
    }
}
2.4 Young Player Visual Indicators
csharp// In PlayerMobile.cs or name display handler
public override void GetProperties(ObjectPropertyList list)
{
    base.GetProperties(list);
    
    if (Young)
    {
        list.Add(1041926); // "Young Player" - uses existing localization
    }
}

// Name color override
public override int GetHue()
{
    if (Young)
        return YoungPlayerConfig.YoungHue; // Green tint
        
    return base.GetHue();
}

3. Starter Quest System - Ferry Islands
3.1 Concept Overview
New players take a special Quest Ferry from Britain docks to a series of training islands in Trammel. Each island has a simple hunting quest that rewards a specific combat skill to GM (100).
┌─────────────────────────────────────────────────────────────────────────┐
│                      STARTER QUEST FERRY ROUTE                          │
└─────────────────────────────────────────────────────────────────────────┘

  BRITAIN DOCKS         ISLAND 1          ISLAND 2          ISLAND 3
  ─────────────────►─────────────────►─────────────────►─────────────────►
  
  Quest Ferry           Rabbit Island     Skeleton Isle     Orc Camp
  NPC gives intro       Kill 20 rabbits   Kill 15 skeletons Kill 10 orcs
                        Reward: 100 Sword Reward: 100 Tact  Reward: 100 Mace
  
  
  ISLAND 4              ISLAND 5          RETURN
  ─────────────────►─────────────────►─────────────────►
  
  Lizardman Lair        Imp Grotto        Britain Docks
  Kill 10 lizardmen     Kill 5 imps       Quest complete!
  Reward: 100 Fencing   Reward: 90 Magery Gold bonus: 5,000
3.2 Quest Ferry NPC
Located at Britain Docks (Trammel side):
csharppublic class QuestFerrymaster : BaseVendor
{
    public override void OnDoubleClick(Mobile from)
    {
        if (!(from is PlayerMobile pm))
            return;
            
        // Check if player has completed all quests
        if (pm.StarterQuestsCompleted)
        {
            Say("You have already completed your training, adventurer. Good luck out there!");
            return;
        }
        
        // Check current quest progress
        int currentQuest = pm.CurrentStarterQuest;
        
        if (currentQuest == 0)
        {
            // New player - give introduction
            pm.SendGump(new StarterQuestIntroGump(pm, this));
        }
        else
        {
            // Returning player - offer ferry to current quest island
            pm.SendGump(new FerryToIslandGump(pm, this, currentQuest));
        }
    }
    
    public void TransportToIsland(PlayerMobile player, int islandNumber)
    {
        var destination = GetIslandLocation(islandNumber);
        
        // Visual effect
        Effects.SendLocationParticles(
            EffectItem.Create(player.Location, player.Map, EffectItem.DefaultDuration),
            0x3728, 10, 10, 2023
        );
        
        player.MoveToWorld(destination, Map.Trammel);
        player.SendMessage(0x35, $"Welcome to Training Island {islandNumber}!");
        
        // Give quest details
        GiveQuestObjective(player, islandNumber);
    }
    
    private static Point3D GetIslandLocation(int island)
    {
        return island switch
        {
            1 => new Point3D(/* Rabbit Island coords */),
            2 => new Point3D(/* Skeleton Isle coords */),
            3 => new Point3D(/* Orc Camp coords */),
            4 => new Point3D(/* Lizardman Lair coords */),
            5 => new Point3D(/* Imp Grotto coords */),
            _ => new Point3D(1496, 1628, 10) // Britain docks fallback
        };
    }
}
3.3 Quest Definitions
csharppublic static class StarterQuestDefinitions
{
    public static readonly StarterQuest[] Quests = new[]
    {
        // Quest 1: Swordsmanship
        new StarterQuest
        {
            QuestNumber = 1,
            Name = "Blade Training",
            Description = "Prove your worth with a blade. Hunt the rabbits that infest this island.",
            TargetCreature = typeof(Rabbit),
            KillCount = 20,
            RewardSkill = SkillName.Swords,
            RewardSkillValue = 100.0,
            IslandName = "Rabbit Island",
            IntroText = "Welcome, young warrior! Before you can face the dangers of Britannia, " +
                       "you must master the blade. These islands are infested with creatures " +
                       "perfect for training. Slay 20 rabbits to prove your swordsmanship!"
        },
        
        // Quest 2: Tactics
        new StarterQuest
        {
            QuestNumber = 2,
            Name = "Combat Tactics",
            Description = "Learn the art of combat tactics against the undead.",
            TargetCreature = typeof(Skeleton),
            KillCount = 15,
            RewardSkill = SkillName.Tactics,
            RewardSkillValue = 100.0,
            IslandName = "Skeleton Isle",
            IntroText = "The undead feel no pain and show no mercy. Fighting them will " +
                       "teach you valuable combat tactics. Destroy 15 skeletons!"
        },
        
        // Quest 3: Mace Fighting
        new StarterQuest
        {
            QuestNumber = 3,
            Name = "Crushing Blows",
            Description = "Master the mace against the orc raiders.",
            TargetCreature = typeof(Orc),
            KillCount = 10,
            RewardSkill = SkillName.Macing,
            RewardSkillValue = 100.0,
            IslandName = "Orc Camp",
            IntroText = "Orcs are tough and brutal. A mace is the perfect weapon against " +
                       "their crude armor. Crush 10 orcs to master mace fighting!"
        },
        
        // Quest 4: Fencing
        new StarterQuest
        {
            QuestNumber = 4,
            Name = "Precision Strikes",
            Description = "Learn fencing against the quick lizardmen.",
            TargetCreature = typeof(Lizardman),
            KillCount = 10,
            RewardSkill = SkillName.Fencing,
            RewardSkillValue = 100.0,
            IslandName = "Lizardman Lair",
            IntroText = "Lizardmen are quick and cunning. To defeat them, you must be " +
                       "even quicker. Master the art of fencing by slaying 10 lizardmen!"
        },
        
        // Quest 5: Magery (partial)
        new StarterQuest
        {
            QuestNumber = 5,
            Name = "Arcane Fundamentals",
            Description = "Channel magic against the imps.",
            TargetCreature = typeof(Imp),
            KillCount = 5,
            RewardSkill = SkillName.Magery,
            RewardSkillValue = 90.0, // Only 90, not 100
            IslandName = "Imp Grotto",
            IntroText = "Magic is powerful but dangerous. These imps are minor demons " +
                       "perfect for practice. Destroy 5 imps using magic to begin " +
                       "your arcane journey! Note: True mastery requires further study."
        }
    };
}

public class StarterQuest
{
    public int QuestNumber { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public Type TargetCreature { get; set; }
    public int KillCount { get; set; }
    public SkillName RewardSkill { get; set; }
    public double RewardSkillValue { get; set; }
    public string IslandName { get; set; }
    public string IntroText { get; set; }
}
3.4 Quest Progress Tracking
csharp// Add to PlayerMobile.cs
public partial class PlayerMobile
{
    // Starter quest tracking
    private int _currentStarterQuest = 0;
    private int _starterQuestKills = 0;
    private bool _starterQuestsCompleted = false;
    
    [CommandProperty(AccessLevel.GameMaster)]
    public int CurrentStarterQuest
    {
        get => _currentStarterQuest;
        set { _currentStarterQuest = value; Delta(MobileDelta.Noto); }
    }
    
    [CommandProperty(AccessLevel.GameMaster)]
    public int StarterQuestKills
    {
        get => _starterQuestKills;
        set => _starterQuestKills = value;
    }
    
    [CommandProperty(AccessLevel.GameMaster)]
    public bool StarterQuestsCompleted
    {
        get => _starterQuestsCompleted;
        set => _starterQuestsCompleted = value;
    }
}
3.5 Kill Tracking & Quest Completion
csharppublic static class StarterQuestManager
{
    public static void OnCreatureKilled(BaseCreature creature, PlayerMobile killer)
    {
        // Only process if player is on a starter quest
        if (killer.CurrentStarterQuest == 0 || killer.StarterQuestsCompleted)
            return;
            
        var quest = StarterQuestDefinitions.Quests[killer.CurrentStarterQuest - 1];
        
        // Check if this is the right creature type
        if (!creature.GetType().IsAssignableTo(quest.TargetCreature))
            return;
            
        // Increment kill count
        killer.StarterQuestKills++;
        
        // Notify player
        int remaining = quest.KillCount - killer.StarterQuestKills;
        if (remaining > 0)
        {
            killer.SendMessage(0x35, $"{quest.TargetCreature.Name} slain! {remaining} remaining.");
        }
        else
        {
            // Quest complete!
            CompleteQuest(killer, quest);
        }
    }
    
    private static void CompleteQuest(PlayerMobile player, StarterQuest quest)
    {
        // Award skill
        player.Skills[quest.RewardSkill].Base = quest.RewardSkillValue;
        
        // Fanfare
        player.PlaySound(0x5B5); // Level up sound
        player.FixedParticles(0x376A, 9, 32, 5030, EffectLayer.Waist);
        
        player.SendMessage(0x35, $"Quest Complete! Your {quest.RewardSkill} is now {quest.RewardSkillValue}!");
        
        // Reset kill counter
        player.StarterQuestKills = 0;
        
        // Check if more quests available
        if (quest.QuestNumber < StarterQuestDefinitions.Quests.Length)
        {
            player.CurrentStarterQuest = quest.QuestNumber + 1;
            player.SendMessage(0x35, "Return to the Quest Ferrymaster for your next challenge!");
        }
        else
        {
            // All quests complete!
            CompleteAllQuests(player);
        }
    }
    
    private static void CompleteAllQuests(PlayerMobile player)
    {
        player.StarterQuestsCompleted = true;
        player.CurrentStarterQuest = 0;
        
        // Final reward - gold bonus
        player.AddToBackpack(new Gold(5000));
        
        player.SendMessage(0x35, "Congratulations! You have completed all starter quests!");
        player.SendMessage(0x35, "You received 5,000 gold as a completion bonus!");
        player.SendMessage(0x35, "You are now ready to explore Britannia. Good luck, adventurer!");
        
        // Teleport back to Britain
        player.MoveToWorld(new Point3D(1496, 1628, 10), Map.Trammel);
    }
}
3.6 Anti-Farming Protection
csharppublic static class StarterQuestManager
{
    // Prevent completed players from farming islands
    public static bool CanEnterStarterIsland(PlayerMobile player, int islandNumber)
    {
        // Already completed all quests - no access
        if (player.StarterQuestsCompleted)
        {
            player.SendMessage(0x22, "You have already completed your training. These islands are for new adventurers.");
            return false;
        }
        
        // Can only access current quest island
        if (player.CurrentStarterQuest != islandNumber)
        {
            player.SendMessage(0x22, "You must complete your current quest before proceeding.");
            return false;
        }
        
        return true;
    }
    
    // No gold drops on starter islands
    public static void ModifyLoot(BaseCreature creature, Container corpse)
    {
        if (IsStarterIsland(creature.Region))
        {
            // Remove all gold from corpse
            foreach (var gold in corpse.FindItemsByType<Gold>().ToList())
            {
                gold.Delete();
            }
        }
    }
}
3.7 Island Region Definitions
csharppublic class StarterIslandRegion : BaseRegion
{
    private readonly int _islandNumber;
    
    public StarterIslandRegion(int islandNumber, string name, Map map, int priority, params Rectangle3D[] area)
        : base(name, map, priority, area)
    {
        _islandNumber = islandNumber;
    }
    
    public override bool OnBeginSpellCast(Mobile m, ISpell s)
    {
        // Allow spellcasting for magery quest
        return true;
    }
    
    public override void OnEnter(Mobile m)
    {
        if (m is PlayerMobile pm)
        {
            if (!StarterQuestManager.CanEnterStarterIsland(pm, _islandNumber))
            {
                // Bounce back to Britain
                Timer.StartTimer(TimeSpan.FromSeconds(1), () =>
                {
                    pm.MoveToWorld(new Point3D(1496, 1628, 10), Map.Trammel);
                });
            }
        }
    }
    
    public override bool AllowHousing(Mobile from, Point3D p) => false;
    public override bool AllowVehicles => false;
}

4. Website Wiki System
4.1 Wiki Structure
Based on UO Outlands wiki (https://wiki.uooutlands.com/Main_Page), create comprehensive documentation:
51alpha Wiki Structure
├── Getting Started
│   ├── Creating Your Character
│   ├── Young Player Protection
│   ├── Starter Quests Guide
│   ├── Basic Controls
│   └── Your First Day
├── Combat
│   ├── Sphere-Style PvP
│   ├── Spell System
│   │   ├── Spell Circles
│   │   ├── Fizzle Mechanics
│   │   └── Movement While Casting
│   ├── Weapon Skills
│   └── Tactics & Anatomy
├── Progression
│   ├── Talisman System
│   │   ├── Talisman Types
│   │   ├── Crafting Talismans
│   │   └── PvP Disable Mechanic
│   ├── Relic Hunting
│   └── House Upgrades
├── Factions
│   ├── Joining a Faction (via Guild)
│   ├── VvV Sigil Battles
│   ├── Daily Bounties
│   ├── Faction Rewards
│   └── Seasonal System
├── Dungeons
│   ├── Dungeon Levels (1-5)
│   ├── Rotating Dungeons
│   ├── Safe Dungeon (Weekly)
│   └── Boss Encounters
├── Crafting
│   ├── Bulk Order Deeds
│   ├── Talisman Crafting
│   └── House Crafting
├── Economy
│   ├── Gold Sources & Sinks
│   ├── Trading
│   ├── Duel Pits
│   └── Texas Hold'em
├── Housing
│   ├── Placing a House
│   ├── House Tiers
│   └── Relic Requirements
└── Rules & Guidelines
    ├── Server Rules
    ├── PvP Etiquette
    └── Reporting Issues
4.2 Key Wiki Pages Content
4.2.1 Sphere-Style PvP Page
markdown# Sphere-Style PvP Combat

51alpha uses **Sphere-style** combat mechanics, which differ significantly from OSI/EA Ultima Online.

## Key Differences from OSI

| Mechanic | OSI Style | Sphere Style (51alpha) |
|----------|-----------|------------------------|
| Cast Flow | Cast → Freeze → Target → Hit | Target → Cast → Free Movement → Hit |
| Movement | Frozen during cast | **Free movement always** |
| Fizzle | Rare | Common (movement, actions, damage) |
| Resource Loss | Sometimes refunded | **Always consumed on fizzle** |

## Fizzle Conditions

Your spell will **fizzle** (fail, consuming mana and reagents) if:

- You cast another spell before the first lands
- You toggle War Mode on/off
- You apply a bandage
- You lose Line of Sight when the spell should land
- You are hit by certain attacks

## Movement

You can **always move** while casting unless paralyzed. Use this to:
- Kite melee attackers
- Break Line of Sight before enemy spells land
- Position for AoE spells

## Tips for New PvPers

1. **Don't panic cast** - Casting while another spell is in flight wastes resources
2. **Use terrain** - Break LoS around corners to cause enemy fizzles
3. **Watch your mana** - Fizzles are expensive
4. **Join a guild** - Faction PvP requires guild membership
4.2.2 Talisman System Page
markdown# Talisman System

Talismans are powerful items that grant **PvM bonuses** to specific playstyles. They are disabled in PvP combat.

## Talisman Types

| Talisman | Playstyle | Key Bonus |
|----------|-----------|-----------|
| Dexer Talisman | Melee DPS | +% weapon damage |
| Tamer Talisman | Pet commands | +% pet damage |
| Sampire Talisman | Chivalry/Leech | Access to Chivalry spells |
| Treasure Hunter Talisman | Exploration | +% chest quality |

## Obtaining Talismans

1. **Farm relics** from dungeons and bosses
2. **Craft the talisman** using relics + crafting skill
3. **Equip the talisman** (only one at a time)

## PvP Disable Mechanic

When you engage in PvP (damage a player or are damaged by one):
- Your talisman **deactivates for 5 minutes**
- All talisman bonuses are lost
- **Chivalry spells become inaccessible** (Sampire)
- Pet speed reduced by 50%

This ensures PvP remains skill-based without PvM advantages.

## Timer

- Talisman timer only starts when **first equipped**
- You can trade/sell talismans before equipping
- Once equipped, timer begins counting down

5. In-Game Help System
5.1 Help Menu NPC
csharppublic class NewPlayerGuide : BaseVendor
{
    public override void OnDoubleClick(Mobile from)
    {
        if (!(from is PlayerMobile pm))
            return;
            
        pm.SendGump(new NewPlayerGuideGump(pm));
    }
}

public class NewPlayerGuideGump : Gump
{
    public NewPlayerGuideGump(PlayerMobile player) : base(50, 50)
    {
        AddBackground(0, 0, 400, 500, 9200);
        AddLabel(150, 20, 0x35, "New Player Guide");
        
        // Topic buttons
        int y = 60;
        AddButton(20, y, 4005, 4007, 1, GumpButtonType.Reply, 0);
        AddLabel(55, y, 0, "Getting Started");
        
        y += 30;
        AddButton(20, y, 4005, 4007, 2, GumpButtonType.Reply, 0);
        AddLabel(55, y, 0, "Combat Basics");
        
        y += 30;
        AddButton(20, y, 4005, 4007, 3, GumpButtonType.Reply, 0);
        AddLabel(55, y, 0, "Starter Quests");
        
        y += 30;
        AddButton(20, y, 4005, 4007, 4, GumpButtonType.Reply, 0);
        AddLabel(55, y, 0, "Joining a Faction");
        
        y += 30;
        AddButton(20, y, 4005, 4007, 5, GumpButtonType.Reply, 0);
        AddLabel(55, y, 0, "Visit Wiki (opens browser)");
        
        // Young player status
        if (player.Young)
        {
            var remaining = YoungPlayerConfig.YoungDuration - (DateTime.UtcNow - player.CreationTime);
            AddLabel(20, 400, 0x35, $"Young Player Protection: {remaining.Days} days remaining");
            AddButton(20, 430, 4005, 4007, 100, GumpButtonType.Reply, 0);
            AddLabel(55, 430, 0x22, "Renounce Protection (enables PvP)");
        }
    }
    
    public override void OnResponse(NetState sender, RelayInfo info)
    {
        var pm = sender.Mobile as PlayerMobile;
        
        switch (info.ButtonID)
        {
            case 1: pm.SendGump(new HelpTopicGump("Getting Started", GettingStartedText)); break;
            case 2: pm.SendGump(new HelpTopicGump("Combat Basics", CombatBasicsText)); break;
            case 3: pm.SendGump(new HelpTopicGump("Starter Quests", StarterQuestsText)); break;
            case 4: pm.SendGump(new HelpTopicGump("Joining a Faction", FactionText)); break;
            case 5: pm.LaunchBrowser("https://wiki.51alpha.com"); break;
            case 100: pm.SendGump(new ConfirmRenounceGump(pm)); break;
        }
    }
}
5.2 Context-Sensitive Tips
csharppublic static class NewPlayerTips
{
    private static HashSet<(Serial, string)> _shownTips = new();
    
    public static void ShowTipOnce(PlayerMobile player, string tipKey, string message)
    {
        if (!player.Young)
            return;
            
        var key = (player.Serial, tipKey);
        if (_shownTips.Contains(key))
            return;
            
        _shownTips.Add(key);
        player.SendMessage(0x35, $"[TIP] {message}");
    }
    
    // Trigger points
    public static void OnFirstSpellCast(PlayerMobile player)
    {
        ShowTipOnce(player, "spell_cast", 
            "In 51alpha, you can move while casting! Use this to dodge enemy attacks.");
    }
    
    public static void OnFirstFizzle(PlayerMobile player)
    {
        ShowTipOnce(player, "fizzle",
            "Your spell fizzled! This happens if you cast again, toggle war mode, or lose line of sight.");
    }
    
    public static void OnFirstDeath(PlayerMobile player)
    {
        ShowTipOnce(player, "death",
            "You died! As a Young Player, other players cannot loot your corpse. Find a healer to resurrect.");
    }
    
    public static void OnEnterDungeon(PlayerMobile player)
    {
        ShowTipOnce(player, "dungeon",
            "Dungeons contain valuable relics for crafting talismans and upgrading houses!");
    }
    
    public static void OnFirstGuildInvite(PlayerMobile player)
    {
        ShowTipOnce(player, "guild",
            "Joining a guild lets you participate in Faction warfare for bonus rewards!");
    }
}

6. First Login Experience
6.1 Character Creation Flow
┌─────────────────────────────────────────────────────────────────────────┐
│                     FIRST LOGIN EXPERIENCE                               │
└─────────────────────────────────────────────────────────────────────────┘

  CHARACTER CREATED     SPAWN LOCATION      IMMEDIATE GUIDANCE
  ─────────────────►───────────────────►──────────────────────────────────►
  
  │                    │                    │
  │ Default stats      │ Britain Bank       │ Welcome message
  │ Default skills     │ (Trammel)          │ Highlight Quest Ferry
  │ Starter equipment  │                    │ New Player Guide NPC
  │                    │                    │ Young status explained
6.2 Welcome Message
csharppublic static class FirstLoginHandler
{
    public static void OnFirstLogin(PlayerMobile player)
    {
        // Teleport to Britain Bank (Trammel - safe)
        player.MoveToWorld(new Point3D(1438, 1690, 0), Map.Trammel);
        
        // Welcome message sequence
        Timer.StartTimer(TimeSpan.FromSeconds(2), () =>
        {
            player.SendMessage(0x35, "═══════════════════════════════════════════");
            player.SendMessage(0x35, "Welcome to 51alpha!");
            player.SendMessage(0x35, "═══════════════════════════════════════════");
        });
        
        Timer.StartTimer(TimeSpan.FromSeconds(4), () =>
        {
            player.SendMessage(0x35, "You have 2 weeks of Young Player protection.");
            player.SendMessage(0x35, "During this time, other players cannot attack or steal from you.");
        });
        
        Timer.StartTimer(TimeSpan.FromSeconds(6), () =>
        {
            player.SendMessage(0x35, "► Visit the QUEST FERRY at Britain Docks to begin your training!");
            player.SendMessage(0x35, "► Speak to the NEW PLAYER GUIDE here for help.");
            player.SendMessage(0x35, "► Type [help for commands or visit wiki.51alpha.com");
        });
        
        // Arrow pointing to Quest Ferry NPC (if client supports)
        Timer.StartTimer(TimeSpan.FromSeconds(8), () =>
        {
            player.QuestArrow = new QuestArrow(player, new Point3D(/* Ferry location */));
        });
    }
}
6.3 Starter Equipment
csharppublic static class StarterEquipment
{
    public static void EquipNewPlayer(PlayerMobile player)
    {
        // Basic weapon (for starter quests)
        var sword = new Longsword();
        player.EquipItem(sword);
        
        // Basic armor
        player.EquipItem(new LeatherChest());
        player.EquipItem(new LeatherLegs());
        player.EquipItem(new LeatherArms());
        player.EquipItem(new LeatherGloves());
        
        // Reagents for magery quest
        player.AddToBackpack(new BagOfReagents(50));
        
        // Bandages
        player.AddToBackpack(new Bandage(50));
        
        // Food
        player.AddToBackpack(new Apple(10));
        
        // Small gold for NPC interactions
        player.AddToBackpack(new Gold(500));
        
        // Spellbook (empty, for magery quest)
        var book = new Spellbook();
        // Add basic spells for quest
        book.Content = 0xFF; // First circle spells
        player.AddToBackpack(book);
    }
}

7. Skill Progression Summary
7.1 Post-Quest Skill State
After completing all starter quests:
SkillValueSourceSwordsmanship100.0Quest 1: Rabbit IslandTactics100.0Quest 2: Skeleton IsleMace Fighting100.0Quest 3: Orc CampFencing100.0Quest 4: Lizardman LairMagery90.0Quest 5: Imp Grotto
7.2 Remaining Progression
Players still need to develop:

Magery to 100 (final 10 points through normal play)
Resisting Spells (critical for PvP)
Evaluating Intelligence (spell damage)
Meditation (mana regeneration)
Anatomy (healing effectiveness)
Healing (bandage skill)
Archery (if desired)
Crafting skills (if desired)

This ensures players are combat-ready quickly but still have meaningful progression goals.

8. Testing Checklist
8.1 Young Player System

 Young status lasts exactly 14 days
 Cannot be attacked by players
 Cannot attack players
 Can be killed by monsters
 Can join guilds
 Cannot join factions until renounced
 Renounce command works correctly
 Visual indicator (green name) displays

8.2 Starter Quests

 Quest Ferry NPC functional
 Transport to each island works
 Kill tracking accurate
 Skills awarded correctly
 Cannot re-enter completed islands
 No gold drops on starter islands
 Final gold reward given
 Teleport back to Britain on completion

8.3 New Player Guide

 NPC accessible at Britain Bank
 All help topics display correctly
 Wiki link opens browser
 Context-sensitive tips trigger once only


9. Change Log
v1.0.0 - 2025-01-02 (Initial Specification)

Created comprehensive new player experience specification
Defined 2-week Young Player protection system
Designed 5-quest ferry island training system
Skills awarded: Swords, Tactics, Macing, Fencing, 90 Magery
Specified wiki structure based on UO Outlands
Created in-game help system with context-sensitive tips
Defined first login experience flow
