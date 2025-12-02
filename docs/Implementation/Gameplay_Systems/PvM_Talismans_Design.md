# PvM with Talismans Deep Dive

## Document Metadata
- **Version**: v2.0.0
- **Last Updated**: 2025-01-03
- **Authors**: Sphere51a Development Team
- **Applicable ModernUO Version**: v24.0.0+
- **PR Ready**: No (Conceptual Design)

## Overview
Talisman types for PvM bonuses that deactivate during PvP. Includes acquisition via relics, crafting, and 5-minute PvP disable mechanic.

## Algorithms and Logic
PvP context detection for talisman deactivation. Timer mechanics and reactivation logic.

## Edge Cases
Multiple simultaneous PvP engagements, timer pausing behavior on unequip, relic loss in PvP zones.

## Implementation Details
TalismanManager for state tracking, deactivation events, and timer coordination.

## Testing Plan
PvP entry simulations, talisman deactivation, timer recovery after equip.

## Talisman Types

| Type              | Playstyle               | Key Bonus                  | Chivalry Access |
|-------------------|-------------------------|----------------------------|-----------------|
| Dexer            | Melee DPS               | +% weapon damage           | No              |
| Tamer            | Pet commands            | +% pet damage              | No              |
| Sampire          | Chivalry/Leech          | Life leech, Chivalry spells| Yes             |
| Treasure Hunter  | Exploration             | +% chest quality           | No              |

Equip (only one at a time) for PvM bonuses that deactivate in PvP.

## Acquisition

Farm relics from dungeons (see Relic Drop Rates).

Craft talisman using relics + crafting skill.

Timer does NOT start until first equip; Talismans can be traded/sold before first equip.

Once equipped, timer begins countdown.

Timer pauses when unequipped? (TBD - specify duration behavior).

## PvP Disable Mechanism

csharppublic static class TalismanManager
{
    private static readonly TimeSpan PvPDisableDuration = TimeSpan.FromMinutes(5);
    
    public static void TriggerPvPDisable(PlayerMobile player)
    {
        var talisman = player.FindItemOnLayer(Layer.Talisman) as BaseTalisman;
        if (talisman == null)
            return;
            
        talisman.IsActive = false;
        talisman.ReactivationTime = DateTime.UtcNow + PvPDisableDuration;
        
        player.SendMessage(0x22, "Your talisman has been disabled for 5 minutes due to PvP combat!");
        
        // Also reduce pet speed
        if (player.AllFollowers != null)
        {
            foreach (var pet in player.AllFollowers.OfType<BaseCreature>())
            {
                pet.ApplyPvPSpeedDebuff(PvPDisableDuration);
            }
        }
    }
}

## Change Log

### v2.0.0 - 2025-01-03 (Updated to align with Architecture_Master v2.0.0)
- Simplified to focus on talisman types, acquisition, and PvP disable mechanic.
- Removed build progression system not present in master.
- Added tables for talisman types and PvP disable code.

### v0.3.0 - 2024-XX-XX (Original Architecture Analysis)
- Initial design planning and progression mechanics conceptualization
- PvM build system framework established with dexer/tamer/sampire/treasurer hunter archetypes
- Damage tracking and PvP disable mechanisms outlined
- Persistence schema and implementation sketch completed
- Anti-abuse measures and balancing suggestions documented

High-level architecture (classes & responsibilities)

TalismanDefinition — immutable data loaded from JSON (build type, multipliers, thresholds, cooldowns).

TalismanItem : Item — in-game item that references a TalismanDefinition. Handles equip/unequip.

TalismanStatus — runtime state for a worn talisman (active/inactive, stacks, lastPvPEventTime).

BuildProgression — per-player persistent data: tracked points per PvM build, tier unlocks, earned abilities.

BuildManager (static service) — API to query/grant points, check whether build bonuses are enabled, and persist progression.

IPvMChecker — small interface to decide what counts as PvM (useful for specialization).

Hooks in PlayerMobile / Mobile for:

OnDamageGiven(target)

OnDamageReceived(attacker)

OnMobileDeath(killer, victim)

OnEquip/Unequip item

Key design decisions (brief)

PvP detection: any player <-> player damage event should mark talisman as disabled for an explicit DisableDuration (or until manual reset). You can choose immediate permanent disable until removed, or timed penalty — I recommend temporary disable (e.g., 5 minutes) so players aren't locked forever for a single slip.

Tracking PvM progress: award "BuildPoints" for confirmed kills of monsters while talisman active. Use objective checks: last attacker is player, mob is NPC, not summoned player pet? (configurable).

Skill gain vs BuildPoints: don't directly auto-increment core skills (that interferes with server skill caps); instead grant BuildPoints which can either:

Auto-convert to temporary skill bonuses,

Or be spent to permanently increase a specialized "BuildSkill" or to unlock special passive/active abilities.

Anti-abuse: require minimum combat engagement (e.g., damage threshold) before awarding points to avoid click-kill farming.

Example JSON talisman definition (data-driven)
{
  "id": "talisman_dexer_01",
  "name": "Talisman of the Swiftblade",
  "build": "Dexer",
  "description": "Increases hunting skill gain for dex-based combat while worn. Disabled in PvP.",
  "skillGainMultiplier": 1.25,
  "buildPointPerKill": 5,
  "minDamageToCredit": 10,
  "maxStack": 1,
  "disableOnPvPMinutes": 5,
  "tierUnlockThresholds": [100, 300, 700],
  "abilities": [
    {"tier": 1, "name": "Quick Parry", "desc":"+2% dodge while out of combat"},
    {"tier": 2, "name": "Adrenal Surge", "desc":"Small burst of stamina on kill"}
  ]
}


Designers can add fields like allowedRegions, noDropOnDeath, soulbound, etc.

C# Implementation sketch (ModernUO / Sphere51a style)

Key points:

Mark Sphere-style edits with //Sphere-style edit as requested in your project memory.

The implementation below is sketch-ready — adapt class names to your codebase (e.g., PlayerMobile, Mobile, Item), register events where your server framework expects them.

//Sphere-style edit: Add Talisman build progression system (PvM-only).
using System;
using System.Collections.Generic;

public enum PvMBuild {
    Dexer,
    Tamer,
    Sampire,
    TreasureHunter,
    // add more
}

public class TalismanDefinition {
    public string Id { get; }
    public string Name { get; }
    public PvMBuild Build { get; }
    public double SkillGainMultiplier { get; }
    public int BuildPointsPerKill { get; }
    public int MinDamageToCredit { get; }
    public int DisableOnPvPMinutes { get; }
    public int[] TierUnlockThresholds { get; }
    public List<TalismanAbility> Abilities { get; }

    public TalismanDefinition(/*json parameters*/) { /* load from json */ }
}

public class TalismanAbility {
    public int Tier { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
}

public class TalismanItem : Item {
    public string DefinitionId { get; set; }
    public TalismanDefinition Definition { get; private set; }

    // runtime status (not persisted on item, we persist per player)
    public TalismanItem(string defId) {
        DefinitionId = defId;
        Definition = TalismanRegistry.Get(defId);
    }

    public override void OnEquip(Mobile wearer) {
        base.OnEquip(wearer);
        if (!(wearer is PlayerMobile pm)) return;
        BuildManager.OnTalismanEquipped(pm, this);
    }

    public override void OnUnequip(Mobile wearer) {
        base.OnUnequip(wearer);
        if (wearer is PlayerMobile pm) BuildManager.OnTalismanUnequipped(pm, this);
    }
}

public static class BuildManager {
    // store progression per player (persisted)
    private static readonly Dictionary<Guid, BuildProgression> Progressions = new();

    public static BuildProgression GetOrCreateProgression(PlayerMobile player) {
        if (!Progressions.TryGetValue(player.Id, out var prog)) {
            prog = PlayerPersistence.LoadProgression(player) ?? new BuildProgression(player.Id);
            Progressions[player.Id] = prog;
        }
        return prog;
    }

    public static void OnTalismanEquipped(PlayerMobile player, TalismanItem talisman) {
        var prog = GetOrCreateProgression(player);
        prog.ActiveTalismanId = talisman.DefinitionId;
        prog.ActiveTalismanActivationTime = DateTime.UtcNow;
        // optionally apply UI flag, buff icon, tooltip update
        player.SendMessage($"Talisman '{talisman.Definition.Name}' activated for {talisman.Definition.Build} PvM progression.");
    }

    public static void OnTalismanUnequipped(PlayerMobile player, TalismanItem talisman) {
        var prog = GetOrCreateProgression(player);
        if (prog.ActiveTalismanId == talisman.DefinitionId) {
            prog.ActiveTalismanId = null;
            player.SendMessage($"Talisman '{talisman.Definition.Name}' unequipped.");
        }
    }

    // Called from damage hooks
    public static void OnDamageGiven(PlayerMobile attacker, Mobile target, int damage) {
        // if target is NPC and talisman active -> maybe award credit on kill
        if (target.IsNpc && attacker != null) {
            var prog = GetOrCreateProgression(attacker);
            if (prog.IsTalismanActive() && damage >= prog.ActiveTalismanMinDamageThreshold()) {
                prog.MarkDamageOn(target, damage);
            }
        } else if (target.IsPlayer) {
            // attacker attacked a player -> disable talisman
            DisableTalismanForPvP(attacker);
        }
    }

    public static void OnDamageReceived(PlayerMobile victim, Mobile attacker, int damage) {
        if (attacker.IsPlayer) {
            // victim got hit by a player -> disable talisman
            DisableTalismanForPvP(victim);
        }
    }

    private static void DisableTalismanForPvP(PlayerMobile player) {
        var prog = GetOrCreateProgression(player);
        if (prog.ActiveTalismanId == null) return;
        var def = TalismanRegistry.Get(prog.ActiveTalismanId);
        prog.SetDisabledUntil(DateTime.UtcNow.AddMinutes(def.DisableOnPvPMinutes));
        player.SendMessage($"Your talisman '{def.Name}' has been temporarily disabled due to PvP.");
        // optionally remove active buffs and update UI
    }

    // Called on mob death - check which players get credit
    public static void OnMobileDeath(Mobile killed, Mobile killer) {
        // iterate lastDamage table in BuildProgression for involved players -> credit the player who dealt the most qualifying damage
        if (!(killed.IsNpc)) return;
        foreach (var kv in Progressions) {
            var prog = kv.Value;
            if (prog.HasDamageOn(killed.Id)) {
                var topDamage = prog.GetDamageOn(killed.Id);
                if (topDamage.Damage >= prog.ActiveTalismanMinDamageThreshold()) {
                    // award points
                    var def = TalismanRegistry.Get(prog.ActiveTalismanId);
                    prog.AddBuildPoints(def.Build, def.BuildPointsPerKill);
                    PlayerMobile player = PlayerManager.FindById(prog.PlayerId);
                    player?.SendMessage($"You earned {def.BuildPointsPerKill} {def.Build} points for killing {killed.Name}.");
                    // clear damage record for this mob
                    prog.ClearDamageOn(killed.Id);
                }
            }
        }
    }
}

public class BuildProgression {
    public Guid PlayerId { get; }
    public Dictionary<PvMBuild,int> Points { get; } = new();
    public string ActiveTalismanId { get; set; }
    public DateTime ActiveTalismanActivationTime { get; set; }
    public DateTime? DisabledUntil { get; private set; }

    // simple in-memory damage tracking keyed by mob id
    private Dictionary<Guid, int> LastDamageByMob = new();

    public BuildProgression(Guid playerId) { PlayerId = playerId; }

    public void MarkDamageOn(Mobile mob, int damage) {
        LastDamageByMob[mob.Id] = Math.Max(LastDamageByMob.GetValueOrDefault(mob.Id), damage);
    }

    public bool HasDamageOn(Guid mobId) => LastDamageByMob.ContainsKey(mobId);
    public (Guid MobId, int Damage) GetDamageOn(Guid mobId) => (mobId, LastDamageByMob[mobId]);
    public void ClearDamageOn(Guid mobId) => LastDamageByMob.Remove(mobId);

    public void AddBuildPoints(PvMBuild build, int points) {
        Points.TryGetValue(build, out int current);
        current += points;
        Points[build] = current;
        // optionally check for tier unlocks and auto-grant abilities
        CheckTierUnlock(build, current);
        PlayerPersistence.Save(this);
    }

    private void CheckTierUnlock(PvMBuild build, int totalPoints) {
        var def = TalismanRegistry.Get(ActiveTalismanId);
        if (def == null) return;
        for (int i = 0; i < def.TierUnlockThresholds.Length; ++i) {
            int threshold = def.TierUnlockThresholds[i];
            if (totalPoints >= threshold && !HasUnlockedTier(build, i+1)) {
                UnlockTier(build, i+1, def.Abilities.FindAll(a => a.Tier == i+1));
            }
        }
    }

    // placeholder methods
    private bool HasUnlockedTier(PvMBuild build, int tier) { /*check persistent store*/ return false; }
    private void UnlockTier(PvMBuild build, int tier, List<TalismanAbility> abilities) { /* grant; persist */ }

    public bool IsTalismanActive() {
        if (ActiveTalismanId == null) return false;
        if (DisabledUntil.HasValue && DisabledUntil.Value > DateTime.UtcNow) return false;
        return true;
    }

    public void SetDisabledUntil(DateTime until) { DisabledUntil = until; PlayerPersistence.Save(this); }

    public int ActiveTalismanMinDamageThreshold() {
        if (ActiveTalismanId == null) return int.MaxValue;
        var def = TalismanRegistry.Get(ActiveTalismanId);
        return def?.MinDamageToCredit ?? int.MaxValue;
    }
}


The above is a focused sketch that shows the core flow:

equip talisman → OnTalismanEquipped

while active → OnDamageGiven records damage to NPCs, OnMobileDeath credits points to the appropriate player(s)

any player-vs-player damage → DisableTalismanForPvP triggers (temporary disable)

progression persists via PlayerPersistence.Save(...)

Progression / point system (recommended)

Points per kill: talisman BuildPointsPerKill (configurable by monster tier). Use a multiplier for elite monsters.

Minimum damage: MinDamageToCredit prevents spawn-stealing / last-hit abuses.

Kill credit rules:

award to player who dealt the highest qualifying damage in last X seconds (configurable)

optionally award fractional points to multiple participants (advanced)

Tier thresholds: thresholds array [100, 300, 700] unlock tier 1, 2, 3 respectively.

Spend vs auto-grant:

Passive model (recommended): points auto-convert when thresholds reached into unlocks (abilities or permanent perks).

Spend model (alternative): open a gump store where player spends points for perks — more player agency.

Abilities & disabling behavior

Passive abilities (e.g., +5% damage vs animals) applied while talisman active.

Active abilities (e.g., special attack) could be toggled — acquire through tier unlock.

On PvP: remove all passive and active talisman-derived bonuses immediately. Optionally, start a DisabledUntil cooldown or force the user to unequip to reset.

On death: if the player is killed in PvP, you may decide to:

Keep points but leave talisman disabled for longer, or

Temporarily reduce stored points (discourages reckless PvP) — design choice.

UI / Feedback

Talisman tooltip should show:

build name (e.g., Dexer)

current points and next-tier threshold

active/inactive state and reason (e.g., "Disabled due to PvP until 2025-12-01 15:23 UTC")

Add a small buff icon overlay when active.

Optional gump: "PvM Progression" showing points, tiers, abilities unlocked, and a "Redeem" or "Details" tab.

Example talisman behaviors mapped to builds

Dexer — faster swing speed, small passive dex → while active: SkillGainMultiplier = 1.2, BuildPointsPerKill = 3

Tamer — more effective pet taming / loyalty gains: while active, increase taming skill gain multiplier and pet loyalty points.

Sampire — hybrid: life-drain on creatures only, vampiric-style self-heal on kill — disabled on PvP.

TreasureHunter — increased detection/chance of extra loot & lockpicking skill gain multiplier.

Cross-System Interactions

**Spell System Integration:**
- PvP/PvM separation in Spell.cs GetNewAosDamage() references BuildManager.CanUsePvMBonuses()
- Chivalry Spells: Entire spell school validity depends on talisman active state - PvP disable automatically prevents Chivalry casting
- PvP Context Detection: Spells check IsPvPContext() to conditionally apply talisman bonuses only when !PvP && active

**Combat System Integration:**
- Pet PvP Mechanics: Player-controlled pets (BaseCreature{Controlled: true}) get 50% speed reduction during PvP engagement
- PvP State Tracking: CombatTimerManager handles pet debuff timing (synced with 5-minute talisman cooldown)
- Damage Hooks: BuildManager.OnDamageGiven/Received called from combat damage resolution

**Creature Behavior Integration:**
- Pet Casting: Pets bypass talisman bonuses (Caster.Player == false), but AoE PvP triggers still apply
- Monster Abilities: Out of scope (abilities ≠ spells), but PvP state could be extensible
- Summon Handling: Player summons treated as pets for PvP contexts

Anti-abuse & balancing suggestions

Cap per-hour points: maxPointsPerHour per talisman to prevent AFK farming.

Mob tier multiplier: stronger mobs grant more points.

Shared credit: if multiple players cooperate, share points proportionally to damage contribution (discourages zerg alt farms).

Cooldown on re-enabling: prevent immediate on/off toggling to dodge PvP penalties.

Audit logs: server log of awarded points for debugging / balancing.

Testing: simulate corner cases: pet kills, poison dot kills, damage from traps, summoned NPCs.

Persistence & DB layout (simple)

Table: BuildProgressions

PlayerId (GUID, PK)

BuildPointsJson (json blob: {"Dexer":120,"Tamer":40})

ActiveTalismanId (string)

DisabledUntilUtc (datetime)

UnlockedTiersJson (json blob)

LastSavedUtc (datetime)

Table: TalismanDefinitions (or load from file into memory on server start)

Integration checklist (what to modify in your codebase)

Add TalismanRegistry to load JSON into TalismanDefinition.

Add TalismanItem as a new item type (or subclass existing talisman wearable).

Hook BuildManager.OnDamageGiven and OnDamageReceived into your combat pipeline. For Sphere51a/ModernUO, this is usually where damage events are handled.

Hook BuildManager.OnMobileDeath into the mob death pipeline.

Create PlayerPersistence methods to save/load BuildProgression.

Add tooltips, buff icons, and optional gump UI for progression display.

Add admin commands for debugging: grant points, reset talisman, view player's build progression.

Example: How this prevents PvP exploitation

Player equips talisman, hunts NPCs, gains points.

Player is attacked by player B: OnDamageReceived triggers, DisableTalismanForPvP sets DisabledUntil = Now + X minutes, removes bonuses, shows message.

If player A attacks another player, the same disable path occurs.

To regain functionality, player must wait DisableOnPvPMinutes or unequip/re-equip only if you allow that reset (choose policy).

Sample JSON for a set of talismans
[
  {
    "id":"talisman_dexer_01",
    "name":"Talisman of the Swiftblade",
    "build":"Dexer",
    "skillGainMultiplier":1.25,
    "buildPointPerKill":5,
    "minDamageToCredit":10,
    "disableOnPvPMinutes":5,
    "tierUnlockThresholds":[100,300,700],
    "abilities":[{"tier":1,"name":"Quick Parry","desc":"Chance to reduce incoming damage by 5% if out of combat."}]
  },
  {
    "id":"talisman_tamer_01",
    "name":"Talisman of the Beastcaller",
    "build":"Tamer",
    "skillGainMultiplier":1.5,
    "buildPointPerKill":3,
    "minDamageToCredit":5,
    "disableOnPvPMinutes":10,
    "tierUnlockThresholds":[50,200,500],
    "abilities":[{"tier":1,"name":"Pet Loyalty","desc":"Pets gain +5 loyalty on kill."}]
  }
]

Quick example: how to code-pair this into an existing kill flow

In your damage resolution code (the place where player attacks triggers damage to NPC):

After damage applied: call BuildManager.OnDamageGiven(attacker, target, damage).

In your MobileDeath event (mob dies and ownership resolved):

At the end of the death handler, call BuildManager.OnMobileDeath(deadMob, killer).

In player vs player damage handling:

call BuildManager.OnDamageReceived(victim, attacker, damage) and BuildManager.OnDamageGiven(attacker, victim, damage) accordingly.

Final suggestions / options (pick one)

Simple & fast: implement the point-per-kill and temporary disable on PvP (recommended minimum viable).

Medium: add shared credit algorithm and per-hour caps.

Full: add unlockable abilities, UI gumps to spend points, and conversion to permanent perks (requires more balancing).


1. DEXER (PURE MELEE / BUSHIDO / PARRY / NINJITSU VARIANTS)
Concept

The “Dexer” is the most straightforward melee build: deals sustained weapon damage, relies on stamina for swing speed, and uses special moves and tactical positioning. In modern UO, Dexers are extremely itemization-dependent due to:

Stamina → Swing Speed scaling

Hit Chance Increase (HCI)

Damage Increase (DI)

Leech properties

Bushido parry & Honor mechanics

Common Dexer Variants

Swords/Bushido/Parry Dexer – tanky, high parry, strong single-target.

“Ninja Dexer” – uses mirror images, animal form for speed, poison.

"Nerve Strike Dexer" (Fencing + Bushido) – PvP oriented, but usable PvM.

"Double Axe Whirlwind Dexer" – crowd-clearing spin-to-win (PvM farming).

Core Skills
Pure Bushido/Parry Dexer (most popular PvM setup)
Skill	Target	Function
Weapon Skill (Swords/Mace/Fencing)	120	Accuracy, damage, ability to land specials
Tactics	120	Damage multiplier
Bushido	120	Honor, Confidence, Evasion, LS crit
Parry	120	35% parry chance (one-handed), 40% with shield but Bushido negates shield bonus
Anatomy	120	Damage and heal bonus
Chivalry	60–100	Enemy of One, Consecrate, Remove Curse, healing
Optional: Healing or Resist for utility.		
Key Mechanics

Honor → Perfection: up to 100% damage bonus

Lightning Strike: 20% crit chance

Evasion: short invulnerability window

Confidence: HP regen during fights

Gear Priorities
Caps

45 HCI / 45 DCI

100 DI (from items + 300% from tactics/anatomy/bushido)

180 stamina (for max swing speed)

Max resists 70/70/70/70/75 (post-enhanced)

Hit Lower Defense / Attack

Leech mods: Hit Life Leech, Hit Mana Leech, Hit Stamina Leech

Artifacts

Mace & Shield Reading Glasses

Crimson Cincture

Conjurer’s Trinket

Ranger’s Cloak of Augmentation

Mana Phase Orb

Playstyle

Dexers are single target assassins with simple rotation:

Honor → Enemy

Lightning Strike spam

Evasion on big hits

Confidence every cooldown

Maintain stamina (stamina damage reduces swing speed)

Strengths

Very high burst on a single boss

Self-healing via Confidence/Leeches

Beginner friendly

Weaknesses

– Cannot handle curses / mana drains without Resist
– Highly gear dependent
– Struggles vs high-damage AoE bosses

2. TAMER (MODERN PET MASTER / CHIVALRY PET META)
Concept

Tamers rely on a fully trained pet as the primary damage source. Modern UO’s pet revamp (post-Publish 97) completely redefined tamers:

Pet training system (level-up, points allocation)

Specialty abilities (AI, Dragon Breath, Armor Ignore, Chiv/AI pets)

Magical pet variants (Rune Beetles, Tritons, Chivalry Cu Sidhes)

Modern Tamer Archetypes

Chivalry Cu Sidhe Tamer – most versatile

Triton AI Tamer – top single target

Rune Beetle + Fire Beetle Pair – armor debuffs + high DPS

Greater Dragon – burst tank (less meta now)

Magery Mastery Tamer – caster with support buffs

Skill Template (Standard 120-cap)
Skill	Target	Function
Animal Taming	120	Control chances, bonding
Animal Lore	120	Pet control, heals
Veterinary	120	Pet heals & cures
Magery	100–120	Utility, travel, heals
Eval Int / Med / Chivalry	varies	Buffs, defense, passive healing
Popular Template

Taming 120 / Lore 120 / Vet 120 / Magery 120 / Eval 120 / Meditation 120

A more support-heavy Tamer might run:
Taming 120 / Lore 120 / Vet 120 / Chivalry 100 / Resist 120 / Magery 80

Pet Builds
Chivalry Cu Sidhe

Abilities: AI + Chiv spells

Roles: PvM bossing, survivability

Triton (Spawned from fishing nets)

Best pure melee DPS pet

AI + innate high stamina/damage

Rune Beetle

Armor Corruption → −30 resists

Combos with: Fire Beetle (for Armor Pierce)

Greater Dragon

High burst (fire breath)

High stats, low DPS compared to modern Chiv pets

Gear Priorities (Tamer)

LMC 40%

Mana regen

Spell damage (if caster)

+Skill items for taming tree

HP/stam/mana increase for survivability

Playstyle

Send in pet with All Kill

Keep distance

Use bandies (Vet) + Magery heals

Maintain pet buffs (Chiv pets especially)

Use Discord (optional) to reduce enemy resist

Strengths

Safe playstyle

Pet does 90% of work

Can solo nearly everything

Weaknesses

– Pet positioning micromanagement
– High maintenance
– Some bosses have anti-pet mechanics

3. SAMPIRE (BUSHIDO + NECROMANCY + VAMPIRIC EMBRACE)
Concept

The Sampire is the strongest solo PvM build in UO history. It combines:

Bushido (LS crit, Evasion)

Necromancy: Vampiric Embrace (20% life leech on all damage)

Double Axe whirlwind for crowd AOE leeching

This creates a self-sustaining DPS monster.

Standard Sampire Template
Skill	Target	Function
Weapon Skill	120	Hit chance, specials
Bushido	120	Crits, Evasion, perfection
Parry	120	Defense
Necromancy	99	Vampiric Embrace
Spirit Speak	20–40	Enhance necro form
Tactics	120	Damage
Anatomy / Resist / Chiv	varies	Utility choices
Popular Build:

120 Weapon / 120 Bushido / 120 Parry / 120 Tactics / 120 Anatomy / 99 Necro / 60 Chiv

Why 99 Necromancy?

Vampiric Embrace requires 99

Going higher reduces curse resistance

Gear
Weapon: Double Axe (Whirlwind) or Radiant Scimitar

Required mods:

Hit Life Leech

Hit Mana/Stam Leech

HSL (critical for max swing)

Area effects (optional)

Armor Goals

45 HCI / 45 DCI

100 DI

40 LMC

180 stamina

Swing Speed Increase 20–30%

Resists compensate for –25 fire resist from Vampiric Embrace

Artifacts

Crimson Cincture

Despicable Quiver / Ranger’s Cloak

Mace & Shield Glasses

Playstyle

Cast Vampiric Embrace

Honor → Target

Close in, spam Whirlwind vs crowds

Against bosses: spam Armor Ignore / LS

Use Evasion to survive heavy hits

Tank by leeching more than you're damaged

Strengths

Highest sustained DPS in the game

Essentially unkillable vs mobs

Solo nearly all peerless/spawns

Weaknesses

– Weak vs undead (cannot life leech)
– Requires 150+ Stam / top gear
– Struggles in curse-heavy zones

4. TREASURE HUNTER (T-HUNTER / CARTOGRAPHER / DISCO-MAGE)
Concept

Treasure Hunters specialize in:

Map decoding

Digging

Managing spawn waves

Loot optimization (global loot revamp)

Modern T-hunters use Masteries, Mysticism, Discordance, or Taming depending on playstyle.

Treasure Hunter Templates
1. Mage Treasure Hunter

Most traditional.

Skill	Target	Function
Cartography	100	Needed to decode maps
Lockpicking	100	Open chest
Remove Trap	100	Required in modern UO
Magery	120	DPS + utility
Eval Int	120	Spell damage
Meditation	120	Mana
Resist	120	Protection vs spawn
2. Mystic Disco T-Hunter

Higher AoE control:

Mysticism 120

Focus 120

Discordance 120

Remove Trap 100

Cartography 100

Lockpicking 100

Remaining points in Magery

3. Tamer Treasure Hunter

Uses pets to clear spawn:

Taming 120 / Lore 120 / Vet 120

Cartography 100

Lockpick 100

Remove Trap 100

Great for high-end chests.

Chest Levels
Level	Difficulty	Drop Quality
Stash	Low	Basic loot
Supply	Low-mid	Useful items
Cache	Mid	Artifacts possible
Hoard	High	Named reagents, arties
Trove	Very high	Best treasure in game
Loot System

Modern treasure chests include:

Imbue-quality gear

Artifact chances

Loot intensity scaled by chest tier

Special resources

Playstyle

Decode → Dig

Kill first wave

Open chest (lockpick → disarm)

Kill reinforcements

Loot quickly before despawn

Mark rune, move to next

Strengths

High profit

Flexible playstyles

Spawns are controlled and predictable

Weaknesses

– Replace-heavy resources (lockpicks, maps)
– Remove Trap is mandatory now
– Slowest kill speed among builds

SUMMARY TABLE (FAST OVERVIEW)
Build	Role	Damage Source	Difficulty	Notes
Dexer	Burst melee	Weapon swings	Moderate	Best single-target non-vamp build
Tamer	Pet master	Pet DPS	Easy	Strongest early-to-midgame solo class
Sampire	Life-leech melee god	Weapon + leech	Hard (gear)	Best overall PvM build in UO
Treasure Hunter	Utility, gold farm	Magic/pet	Easy–Moderate	Best money-maker, unique gameplay

## Development Prerequisites

### ModernUO Base Requirements
- **Minimum Version**: ModernUO v24.0.0 or later (Assumes clean master branch)
- **Branch**: Use `feature/pvm_talismans` for implementation
- **Dependencies**: Must implement PlayerPersistence.SaveOad() for player data; JSON configuration system for TalismanDefinitions

### Required Framework Knowledge
- ModernUO Item system (Item.cs, OnEquip/Unequip events)
- Player persistence API (PlayerMobile.Save/Serialize)
- JSON serialization in C#
- Event-driven combat system (Mobile.OnDamageGiven/Received)
- GUID-based identification for damage tracking

### Pre-Implementation Checklist
- [ ] JSON configuration system operational for game data loading
- [ ] Player persistence layer supports complex objects like BuildProgression
- [ ] Combat damage hook points identified and accessible
- [ ] Mobile death event handler established
- [ ] Admin command infrastructure active for debugging

## Code Integration Guide

### Step 1: Data Definition Setup (Low Risk)
1. Create `Data/Talismans/` directory with JSON definition files
2. Implement `TalismanRegistry.Load()` to parse definitions on server startup

### Step 2: Core Classes Implementation (Medium Risk)
1. Add `TalismanDefinition.cs` in `Systems/Sphere51a/TalismanDefinition.cs`
2. Add `TalismanItem.cs` in `Items/Special/TalismanItem.cs`
3. Add `BuildProgression.cs` in `Systems/Sphere51a/BuildProgression.cs`
4. Add `BuildManager.cs` in `Systems/Sphere51a/BuildManager.cs`

### Step 3: Combat System Integration (High Risk)
1. Hook `BuildManager.OnDamageGiven()` in combat damage resolution
2. Hook `BuildManager.OnDamageReceived()` in damage receipt handling
3. Hook `BuildManager.OnMobileDeath()` in Mobile.OnDeath()

### Step 4: Persistence Integration (Medium Risk)
1. Extend PlayerMobile Serialize/Deserialize for BuildProgression
2. Add automated saving on progression changes
3. Implement data migration for existing players

### Step 5: Testing Integration
1. Unit tests for BuildProgression point calculations
2. Integration tests for damage/die event chaining
3. Endurance tests for persistence under load

## Performance Benchmarks

### Expected Performance Impact
- `BuildManager.GetOrCreateProgression()`: 1-3ms (first lookup, subsequent cache hits)
- Combat hooks: Minimal (<0.5ms per damage event)
- Persistence: JSON serialize on save operations (scales with data Complexity)

### Monitoring Recommendations
- Track talisman equip/unequip events for adoption metrics
- Monitor PvP disable frequency for balance tuning
- Log persistence operation timings for optimization

## Maintenance Notes

### Future Enhancements
- Expand build types (add Necromancer, Paladin specializations)
- Implement achievement system for PvM milestones
- Add guild-wide progression for clan PvM competitions
- Integrate with virtual economies (purchase talismans with ingame currency)

### Database Considerations
- Consider binary serialization for large progression objects
- Implement data cleanup for inactive players
- Add index on PlayerId for quick progression lookups

### Rollback Procedures
1. Stop server and back up BuildProgression data
2. Comment out BuildManager hook calls
3. Remove talisman equip events
4. Clear all active talisman states (in-memory)
5. Resume with notifications to offline players

## Testing Procedures

### Unit Testing (Automated)
```csharp
[TestMethod]
public void BuildProgression_AddBuildPoints_IncreasesPoints()
{
    var prog = new BuildProgression(Guid.NewGuid());
    prog.AddBuildPoints(PvMBuild.Dexer, 5);
    Assert.AreEqual(5, prog.Points[PvMBuild.Dexer]);
}

[TestMethod]
public void TalismanItem_OnEquip_ActivatesTalisman()
{
    var pm = new PlayerMobile();
    var def = new TalismanDefinition(/*params*/);
    var item = new TalismanItem(def.Id);
    item.OnEquip(pm);
    // Verify BuildManager called with correct params
}
```

### Integration Testing (Live Server)
1. **Talisman Equip/Unequip**: Verify state activation/deactivation and messaging
2. **Damage Tracking**: Combat monster and verify progression accumulation
3. **PvP Disable**: Engage player combat, confirm talisman temporary disable
4. **Persistence**: Relog and confirm point/build state retention
5. **Death Events**: Kill attributed correctly with points awarded

### Load Testing
- Simulate 100 players with talismans in concurrent combat
- Verify progression saving under high-frequency events
- Test memory usage with large active talisman populations

## Change Log
[To be filled]
