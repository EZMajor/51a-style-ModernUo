# Fishing, Talisman & Leviathan Armor — Research Findings

## Document Metadata
- **Version**: v1.0.0
- **Last Updated**: 2026-07-03
- **Authors**: Sphere51a Development Team
- **Type**: Research / findings note (feeds `Angler_Talisman_Design.md`)
- **Applicable ModernUO Version**: v24.0.0+

---

## Provenance & confidence (read first)

This note records research for a fishing → talisman → sea-hunter (Leviathan) armor playstyle. Sources and their trust level:

| Area | Source | Confidence |
|---|---|---|
| Fishing mechanics | Live code in local checkout (`51a-style-ModernUo`) | **High — verified** |
| Sea-creature combat data | Live code (creature stat blocks) | **High — verified** |
| Talisman framework | Local design docs (`PvM_Talismans_Design.md`, `specs/talisman-system.md`) + stock `BaseTalisman.cs` | **High for stock code; design-only for the Sphere51a build system** |
| **Leviathan / fishing armor** | **Stock ModernUO leather in local checkout** | **LOW for phoenix-shard — see the access limitation below** |
| Fisher-fighter builds | UO community wikis (Outlands, Second Age, UOGuide) | Medium — external inspiration |

> **Access limitation (important):** The live shard repo `https://git.icelabs.cc/imaginenation/phoenix-shard` is **not reachable from this environment**. It sits behind a mesh VPN (`mesh.icelabs.cc`) and my sandbox's egress proxy refuses the connection (`CONNECT tunnel failed, 403`) before the request ever reaches it. All armor findings below therefore reflect **stock ModernUO** as present in the local checkout. **The shard maintainers have confirmed phoenix-shard replaced the stock leather tiers (Spined/Horned/Barbed) with a custom Leather / Bone / Scale system tied to tailoring-crafted armor.** That system is **not present in this checkout and has not been read** — Section 4 flags exactly what must be reconciled.

---

## 1. Fishing system (verified)

Fishing is a subclass of the shared harvest engine, not a bespoke system.

- **Engine:** `Projects/UOContent/Engines/Harvest/Fishing.cs` (`Fishing : HarvestSystem` singleton)
- **Framework:** `Projects/UOContent/Engines/Harvest/Core/` (`HarvestSystem`, `HarvestDefinition`, `HarvestResource`, `HarvestVein`, `HarvestBank`, `BonusHarvestResource`)
- **Tool:** `Items/Skill Items/Fishing/FishingPole.cs` — item `0x0DC0`, two-handed, weight 8. Not `IUsesRemaining`, so the stock pole never actually breaks.

**Loop:** target water within 4 tiles → each 8×8 water patch is a `HarvestBank` of 5–15 fish, respawn 10–20 min (Elves 25% faster) → success always yields a plain `Fish`; **skill only gates what the fish mutates into.** Can't fish mounted; one action at a time.

**Mutate table** (`Fishing.cs:13-31`, applied in `MutateType` `:162`), chance `= (skill − minSkill) / (maxSkill − minSkill)`, deep-water entries require `SpecialFishingNet.FullValidation`:

| Catch | ReqSkill | Deep water | ~Chance @100 |
|---|---|---|---|
| SpecialFishingNet | 80 | ✅ | ~0.5% |
| BigFish | 80 | ✅ | ~0.5% |
| TreasureMap (lvl 1) | 90 | ✅ | ~0.5% |
| MessageInABottle | 100 | ✅ | ~0.5% |
| Magic fish (Prized/Wondrous/TrulyRare/Peculiar) | 0 | any | ~0.9% |
| Waterlogged footwear (Boots/Shoes/Sandals/ThighBoots) | 0 | any | ~0.95% |
| WhitePearl (ML bonus resource `:111`) | 80 | any | 0.6% |

**Key finding for a GM-all shard:** every value catch shares the curve `(skill − 80) / 4000`, so at GM everyone rolls the identical ~0.5% and gets the whole loot loop for free. **Fishing confers zero build identity today** — the design rationale for gating rewards behind a talisman instead of skill.

**Special catches & rewards** (verified):
- **SOS / MIB** (`Misc/MessageInABottle.cs`, `Misc/SOS.cs`): MIB rolls level 1–3, **1-in-25** chance of level 4 "Ancient" (AOS). SOS points to a deep-water wreck; fishing within **60 tiles** yields shipwreck junk via `Random(8)`, ~1-in-8 the actual chest (`TreasureMapChest.Fill`, level 1–4). Ancient → gold chest + `FabledFishingNet`; normal → `SpecialFishingNet`.
- **Serpent ambush** (`Fishing.Give:324`): catching a TreasureMap/MIB/SpecialFishingNet spawns a serpent holding it (25% DeepSeaSerpent, else SeaSerpent).
- **BigFish** (`Resources/Fishing/BigFish.cs`): weight randomized 3–200 stones; ≥20 stamps "Caught by ~name~" + weight (trophy record). Drops at feet.
- **Magic fish** (`Resources/Fishing/MagicFish.cs`): eaten for a 1-min buff — Prized +5 Int, Wondrous +5 Dex, TrulyRare +5 Str, Peculiar +10 Stam.
- **Nets** (`Misc/SpecialFishingNet.cs`): summon 2–5 sea creatures on deep water; 1% rare hue. **FabledFishingNet** → +4 spawns and a **Leviathan**.
- **Aquarium** (`Items/Aquarium/`): shallow-water `AquariumFishNet` (needs Fishing ≥10) catches decorative fish; tended aquariums dispense décor rewards.

---

## 2. Sea creatures & combat profile (verified)

| Creature | File | Hits | Damage | Damage type | Notes |
|---|---|---|---|---|---|
| WaterElemental | `Monsters/Elemental/Magic/WaterElemental.cs` | 76–93 | 7–9 | **100% physical** | — |
| SeaSerpent | `Monsters/Reptile/Magic/SeaSerpent.cs` | 110–127 | 7–13 | **100% physical** | FireBreath |
| DeepSeaSerpent | `.../DeepSeaSerpent.cs` | 151–255 | 6–14 | **100% physical** | FireBreath |
| Kraken | `Monsters/Reptile/Melee/Kraken.cs` | 454–468 | 19–33 | **70% phys / 30% cold** | TreasureLvl 4 |
| **Leviathan** | `.../Magic/Leviathan.cs` | **1500** | 25–33 | **70% phys / 30% cold** | Cold breath (30 cold AoE), Str 1000, TreasureLvl 5 |

**Defensive priority for the whole family: Physical ≫ Cold > Fire (breaths).** This is the north-star for any fishing-armor design regardless of the material system.

**Leviathan** (`Leviathan.cs`): 1500 HP, LootPack FilthyRich×5, tracks its summoning `Fisher`; on death gives that fisher an artifact **25%** of the time from a 23-item table (GhostShipAnchor, GoldBricks, DreadPirateHat, Captain Quacklebush's Cutlass, etc.), plus paragon chance. This is the intended fishing end-game payoff.

---

## 3. Talisman system (verified stock; Sphere51a build system is design-only)

**Stock `BaseTalisman.cs`** already supports: a `Skill` + `SuccessBonus` + `ExceptionalBonus` mechanic (**craft-skill list only — Fishing is NOT in it**), `AosSkillBonuses`, `Killer`/`Protection` creature-type bonuses, `Summoner`, `Slayer`, charges. Equipped on `Layer.Talisman`.

**Sphere51a PvM talisman framework** (`PvM_Talismans_Design.md`, `specs/talisman-system.md`) — **design-only, not yet implemented** (no `BuildManager`/`TalismanState` code, no `Systems/Sphere51a/` folder):
- Archetypes: Dexer / Tamer / Sampire / Treasure Hunter. **One equipped at a time.**
- PvM bonuses **disable for 5 minutes on any PvP hit**; timer pauses on unequip, resumes on re-equip (`TalismanState`).
- Data-driven `TalismanDefinition` JSON; `BuildPoints` per qualifying kill; `tierUnlockThresholds`; anti-AFK caps.

**Implication:** a fishing/Mariner talisman is a clean 5th archetype using the same framework, and the natural place to move fishing's value on a GM-all-skills shard.

---

## 4. Leviathan / fishing armor — PARTIAL, needs reconciliation

> ⚠️ **This section is the weak point.** The numbers below are **stock ModernUO from the local checkout**. The shard has replaced this with a **Leather / Bone / Scale** system I could not read. Treat stock values as *context for the design intent only*, not as the shard's real data.

### 4a. Stock ModernUO leather (local checkout — for reference/contrast)
- Leather is the weakest material: `LeatherChest` base resist Phys 2 / Fire 4 / Cold 3 / Poison 3 / Energy 3 (`Items/Armor/Leather/LeatherChest.cs`). Resist per piece = `Base + resourceAttrs + protOffset + bonuses` (`BaseArmor.cs:509`).
- Stock leather resource tiers (`Misc/ResourceInfo.cs`) and their runic roll params:

| Stock tier | Base resist P/F/C/Po/E | Runic props | Runic intensity (ML) | Kit charges |
|---|---|---|---|---|
| Spined | +5/0/0/0/0 (+40 Luck) | 1–3 | 40–100% | 45 |
| Horned | +2/+3/+2/+2/+2 | 3–4 | 45–100% | 30 |
| Barbed | +2/+1/+2/+3/+4 | 4–5 | 50–100% | 15 |

- Resist cap **70** (`Mobile.MaxPlayerResistance`; GMs 100). Enhancing exists and can fail/destroy (`Engines/Craft/Core/Enhance.cs`). Runic pipeline special-cases leather (`BaseRunicTool.cs:643`).
- **No armor set-bonus system exists** in the codebase.
- Hides come from `SkinningKnife` carve via `BaseCreature.OnCarve` (`HideType` enum).

### 4b. What phoenix-shard actually uses (reported by maintainers — NOT yet verified in code)
- Stock **Spined / Horned / Barbed leather tiers were removed.**
- Armor tiers are now **Leather → Bone → Scale**, all **tailoring-crafted** ("tied to leather/tailoring crafted armor").
- **Stat tier still comes from the runic system: there are only 3 runic sewing kit tiers.**

### 4c. Open reconciliation questions (blockers for a correct armor design)
1. Are **Leather / Bone / Scale** three ascending material tiers, or parallel material lines?
2. Do the **3 runic sewing kits** map 1:1 onto Leather/Bone/Scale, or are they a separate stat axis applied on top of any material?
3. What are the **base resist profiles** of Leather / Bone / Scale (per piece)?
4. What **attributes/intensities** does each of the 3 runic tiers roll, and what are the kit charge counts?
5. Where do **Bone** and **Scale** hides/resources come from (creature carve? tailoring resource?), and does that give a natural hook to gate a "sea/Leviathan" material?
6. Is the resist **cap still 70**?

**Files needed to answer** (paste here or mirror to `ezmajor/51a-style-modernuo`): the Leather/Bone/Scale armor classes, the resource/resist definitions (shard's `ResourceInfo` equivalent), the 3 runic sewing-kit definitions, and the tailoring craft definitions.

### 4d. Design intent that survives the system change (system-agnostic)
Regardless of the material model, the sea-hunter armor goal is fixed by the verified combat data (Section 2): **prioritize physical, then cold, then fire (breaths).** A "Leviathan-hunter" armor should be the tailoring-crafted material that best answers physical + cold, gated so it's obtained through the fishing/talisman loop (e.g. a sea-sourced Bone/Scale hide) rather than by skill — consistent with "the talisman gives the value."

---

## 5. Fisher-fighter builds (external research / inspiration)

On other shards the fisher-fighter is a real archetype. Recurring patterns:
- **Archer / hide-and-shoot fisher** — Fishing + Archery/Tactics/Anatomy/Healing, Hiding while sailing, shooting surfacing serpents.
- **Fisher-tamer** — Fishing 120 + Taming/Lore/Vet so a pet tanks the sea spawn.
- **Bard fisher** — Provocation/Peacemaking to turn or pacify sea creatures.
- **Harpooner (UO Outlands model)** — the Fishing skill grants ranged **ocean weapons** whose special is **Impale** (hits nearby targets); leather **fishing nets are tailor-crafted from the leather tiers**.

**For a GM-all-skills shard a "template" is not a skill spread — it's a talisman + gear identity.** The Mariner talisman + sea-tuned tailoring armor supports all of the above; the **harpooner** is the closest match to a leather/tailoring sea-hunter and is the recommended signature.

Sources: [UO Outlands – Fishing](https://wiki.uooutlands.com/Fishing), [UO Outlands – Templates](https://wiki.uooutlands.com/Templates), [UO Addicts – Outlands builds](https://uoaddicts.com/builds/uo-outlands), [UO Second Age – Fishing](https://wiki.uosecondage.com/Fishing), [UOGuide – Fishing](https://www.uoguide.com/Fishing), [UO.com – Fishing](https://uo.com/wiki/ultima-online-wiki/skills/fishing/).

---

## 6. Summary & next step

- **Solid to build on now:** fishing mechanics, sea-creature combat data, talisman framework, and the "move value from skill → talisman" rationale.
- **Blocked pending shard code:** the armor mapping, because phoenix-shard's Leather/Bone/Scale + 3-runic-tier system differs from stock and is unreadable from this environment.
- **To unblock:** supply the four file groups in Section 4c (paste or mirror to the in-scope GitHub repo). Then Section 4 and the design doc's Part 4 can be rewritten against the real system.

## Change Log
### v1.0.0 - 2026-07-03
- Initial research consolidation: fishing system, sea-creature combat data, talisman framework, stock-armor reference with explicit phoenix-shard divergence flags, and fisher-fighter build inspiration. Records the phoenix-shard access limitation and the exact reconciliation questions/files needed for the armor design.
