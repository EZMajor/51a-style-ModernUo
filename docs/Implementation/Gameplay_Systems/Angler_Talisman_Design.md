# Mariner (Angler) Talisman & Sea-Hunter Playstyle Deep Dive

## Document Metadata
- **Version**: v1.0.0
- **Last Updated**: 2026-07-03
- **Authors**: Sphere51a Development Team
- **Applicable ModernUO Version**: v24.0.0+
- **PR Ready**: No (Conceptual Design)
- **Dependencies**: Fishing / Harvest System, PvM Talisman System (`PvM_Talismans_Design.md`, `specs/talisman-system.md`), CraftResource / Leather Armor, Tailoring, Sea Creatures (SeaSerpent/DeepSeaSerpent/Kraken/WaterElemental/Leviathan), BuildManager progression

---

## Overview

A complete gather-and-hunt playstyle for the ocean. The **Mariner Talisman** is the 5th PvM archetype alongside Dexer / Tamer / Sampire / Treasure Hunter. It converts fishing from a flat, skill-gated activity into a tiered progression, gates the valuable ocean rewards behind the talisman, and grants escalating **combat bonuses versus sea creatures and the Leviathan**. It is paired with a dedicated **Mariner's Leather armor path** whose resists are tuned to the ocean threat profile, culminating in an end-game sea-hunter that feels powerful and worth wearing — without trivializing the Leviathan.

### The problem it solves

On Sphere51a **everyone can GM every skill**, so `Fishing 100` is a floor, not a differentiator. In stock ModernUO (`Projects/UOContent/Engines/Harvest/Fishing.cs`), all valuable catches are skill-gated on the curve `chance = (skill − 80) / 4000`, which means every player rolls the identical ~0.5% and receives the entire SOS → treasure → Leviathan → artifact loop for free. Fishing today confers **zero build identity**.

The fix: **move the gate from skill to the talisman.** Re-home the valuable outcomes onto an equipped, active Mariner talisman, so the skill floor gives you plain fishing and the talisman gives you the *career*.

### Core Design Philosophy

| Principle | Implementation |
|-----------|----------------|
| Talisman is the build | Valuable ocean rewards read the equipped Mariner tier, not `Skills.Fishing` |
| Hard where it matters | SOS / treasure / Fabled-net / Leviathan / artifact loop is **impossible** without an active Mariner talisman |
| Soft where it flavors | Big fish & magic fish still drop at reduced rates so casual fishing has texture |
| Earned, not bought | Tiers advance via **Fathoms** (build points) from rare catches and sea kills, with anti-AFK caps |
| Gear follows the theme | Sea creatures are physical/cold; the Mariner's Leather line and set bonus answer physical/cold |
| End-game feels good, not free | Full T5 suit makes Leviathan farmable but still demanding (1500 HP, 25–33 dmg, cold breath AoE) |
| PvM only | All talisman & set combat bonuses inherit the 5-minute PvP disable; armor resist itself is passive |

---

## How fishing & the ocean threat work today (baseline facts)

All numbers below are read directly from the current codebase and are the anchors for tuning.

### Fishing mutate table — `Fishing.cs:13-31`, applied in `MutateType` (`:162`)
Chance `= (skillValue − minSkill) / (maxSkill − minSkill)`, evaluated in order, first hit wins; deep-water entries require `SpecialFishingNet.FullValidation`.

| Catch | ReqSkill | Deep water | ~Chance @100 |
|---|---|---|---|
| SpecialFishingNet | 80 | ✅ | ~0.5% |
| BigFish | 80 | ✅ | ~0.5% |
| TreasureMap (lvl 1) | 90 | ✅ | ~0.5% |
| MessageInABottle | 100 | ✅ | ~0.5% |
| Magic fish (Prized/Wondrous/TrulyRare/Peculiar) | 0 | any | ~0.9% |
| Waterlogged footwear | 0 | any | ~0.95% |
| WhitePearl (ML bonus resource, `:111`) | 80 | any | 0.6% |

Supporting hooks: `Construct` (`:213`, SOS chest fill + shipwreck junk), `Give` (`:324`, serpent ambush / item handoff), `SpecialFishingNet.cs` (net → 2–5 sea creatures; `FabledFishingNet` → +4 & a Leviathan), `Leviathan.OnDeath` (25% artifact from a 23-item table).

### Sea creature damage & defense profile (combat anchors)

| Creature | File | Hits | Damage | Damage type | Notable |
|---|---|---|---|---|---|
| WaterElemental | `Monsters/Elemental/Magic/WaterElemental.cs` | 76–93 | 7–9 | **100% physical** | — |
| SeaSerpent | `Monsters/Reptile/Magic/SeaSerpent.cs` | 110–127 | 7–13 | **100% physical** | FireBreath |
| DeepSeaSerpent | `.../DeepSeaSerpent.cs` | 151–255 | 6–14 | **100% physical** | FireBreath |
| Kraken | `Monsters/Reptile/Melee/Kraken.cs` | 454–468 | 19–33 | **70% phys / 30% cold** | — |
| **Leviathan** | `.../Magic/Leviathan.cs` | **1500** | 25–33 | **70% phys / 30% cold** | Cold breath (30 cold AoE), Str 1000, TreasureLvl 5 |

**Defensive priority for the whole family: Physical ≫ Cold > Fire (breaths).** This is the design north-star for the armor path.

### Leather armor facts

- Leather is the **weakest** material: `LeatherChest` base resist = **Phys 2 / Fire 4 / Cold 3 / Poison 3 / Energy 3** (`Items/Armor/Leather/LeatherChest.cs`). Resist per piece = `Base + resourceAttrs + protOffset + bonuses` (`BaseArmor.cs:509`).
- Leather resource tiers add resists via `CraftAttributeInfo` (`Misc/ResourceInfo.cs`):

| Leather | Phys | Fire | Cold | Poison | Energy | Character |
|---|---|---|---|---|---|---|
| Spined | +5 | — | — | — | — | flat physical |
| Horned | +2 | +3 | +2 | +2 | +2 | balanced |
| Barbed | +2 | +1 | +2 | +3 | +4 | energy-leaning |

**No vanilla leather is cold-heavy** (only White dragon scales are, at +10 cold). That gap is exactly what the Mariner line and set bonus fill.
- Resist cap is **70**. Enhancing exists (`Engines/Craft/Core/Enhance.cs`) and can fail/destroy.
- **There is no armor set-bonus system in the codebase** — the Mariner set is a new Sphere51a mechanic.

---

## Part 1 — Re-gating the fishing rewards

Catches are split into **soft-gated** (reduced without a talisman) and **hard-gated** (impossible without an active Mariner talisman of the required tier). The gate reads `BuildManager.GetMarinerTier(from)` in place of raw skill inside `MutateType` / `Construct` / `Give`.

| Reward | Gate | No talisman | T1 | T2 | T3 | T4 | T5 |
|---|---|---|---|---|---|---|---|
| Plain Fish / footwear / shells | none | full | full | full | full | full | full |
| **BigFish** | soft | 25% rate, weight cap 60 | 60%, cap 100 | 80%, cap 140 | 100%, cap 180 | 100%, cap 200 | 100%, cap 200 + hued record |
| **Magic fish** (stat buffs) | soft | 25% | 50% | 75% | 100% | 100% | 100% + 2-min duration |
| **SpecialFishingNet** | hard | ❌ | ✅ base | ✅ | ✅ | ✅ | ✅ |
| **Rare-hue net** | hard | — | 1% | 1% | 2% | 4% | 8% |
| **TreasureMap / MIB / SOS loop** | hard | ❌ (deep water yields only shipwreck junk) | ❌ | ✅ | ✅ | ✅ | ✅ |
| **Ancient SOS chance** | hard | — | — | 1/25 | 1/18 | 1/12 | 1/8 |
| **WhitePearl + shard mat** | soft | ❌ | 0.3% | 0.6% | 0.9% | 1.2% | 1.5% |
| **Fabled net / Leviathan summon** | hard | ❌ | ❌ | ❌ | ✅ | ✅ | ✅ |
| **Leviathan artifact roll** | hard | — | — | — | 25% | 33% | 45% |
| **Mariner's Leather hides** (armor mat) | hard | ❌ | carve serpents (low yield) | carve serpents | carve Kraken | carve Kraken | carve Leviathan (high yield) |

> **Rule of thumb:** without an active Mariner talisman, deep-sea fishing is *stock Britannia fishing minus the good parts* — fish, boots, junk. Everything that drives the economy or the chase is talisman-locked. This is what makes it "hard to do without the talisman."

---

## Part 2 — The Mariner Talisman: tiers & progression

### Currency: **Fathoms** (build points)

Awarded in `Fishing.OnHarvestFinished` (`:473`) and on sea-creature death, only while a Mariner talisman is equipped and active. Anti-abuse: per-hour cap, minimum-damage credit for kills (mirrors `PvM_Talismans_Design.md`).

| Source | Fathoms |
|---|---|
| Rare catch (net / big fish / magic fish) | 3 |
| Treasure map / MIB fished up | 10 |
| SOS chest recovered | 25 |
| Sea serpent / water elemental killed | 2 |
| Kraken killed | 20 |
| Leviathan killed | 150 |
| Ancient SOS chest recovered | 60 |

**Per-hour cap:** 400 Fathoms/hour (tunable) to kill AFK macro-fishing.

### Tier ladder

| Tier | Name | Fathoms | Unlocks (fishing) | vs-Sea dmg **dealt** | vs-Sea dmg **taken** | Ability |
|---|---|---|---|---|---|---|
| T1 | Deckhand | 0 (equip) | Nets, soft big/magic fish | +5% | −5% | **Old Salt** — sea creatures are 25% less likely to aggro first |
| T2 | Fisher | 500 | SOS/treasure loop, ancient 1/25 | +10% | −8% | **Chum the Water** — 60s: +50% rare-mutate odds, 10-min CD |
| T3 | Mariner | 2,000 | Fabled net & **Leviathan summon**, artifact 25% | +15% | −12% | **Harpoon** — targeted sea creature takes +30% from your next hit, 15s CD |
| T4 | Sea Wolf | 6,000 | Ancient 1/12, artifact 33%, rare-hue net 4% | +20% | −15% | **Steady Hand** — pity timer: guarantees a rare mutate within 40 casts |
| T5 | Leviathan's Bane | 15,000 | Ancient 1/8, artifact 45%, hue 8%, big-fish records | +25% | −20% | **Krakensbane** — vs sea creatures only: +5% Hit Life Leech and executes sea creatures under 8% HP |

Notes:
- All **combat** numbers are **PvM-only** and disabled for 5 minutes on any PvP hit (inherits `TalismanState` from `specs/talisman-system.md`). Fishing itself is unaffected by PvP disable except that talisman-gated *catches* also pause while disabled — flagged deep-sea fishing yields only junk.
- One talisman equipped at a time (`Layer.Talisman`), so choosing Mariner is a real opportunity cost against Dexer/Sampire/etc.
- Fathoms persist on the player (`BuildProgression`), so the talisman build is personal and non-fungible — swapping archetypes leaves your ocean tier behind. This is the retention hook.

### "Sea creature" definition

Introduce `ISeaCreature` (marker interface) implemented by SeaSerpent, DeepSeaSerpent, WaterElemental, Kraken, Leviathan (and future sea drakes / charybdis). All Mariner combat bonuses and the set bonus check `target is ISeaCreature`. Avoids brittle type lists and is the single extension point for new ocean content.

---

## Part 3 — Combat vs sea monsters & the Leviathan

The Mariner is a **leather dexer of the sea**: light armor, high stamina, melee/throwing, sustained by talisman leech and the set bonus rather than heavy plate. Because sea creatures are overwhelmingly physical, a plate character trivializes them — so the design keeps the Mariner in leather and closes the survivability gap *only vs sea creatures*, preserving general balance.

### Offense
- Flat `+dmg%` vs `ISeaCreature` from the talisman tier (table above), applied in the damage pipeline (`Misc/AOS.cs` `GetDamage` / `BuildManager.ApplyTalismanBonus`) with the existing PvM check.
- **Harpoon** (T3) and **Krakensbane** (T5) give the burst + sustain needed to solo a 1500-HP Leviathan without a plate suit.

### Defense (talisman)
- Flat `−dmg% taken` from sea creatures (tier table), applied on incoming damage when `attacker is ISeaCreature`.
- T5 **Krakensbane** leech turns Leviathan's own AoE into your sustain.

### Defense (armor) — see Part 4
The Mariner's Leather line supplies the raw **physical + cold** resist; the set bonus adds a sea-only top-up and the T5 breath immunity.

### Encounter math (the "feels good but not easy" target)
Target for a **full T5 Mariner** vs Leviathan:
- Effective mitigation vs sea ≈ **70 physical / ~65 cold** (Mariner's Leather resists + set bonus, capped at 70), plus talisman −20% taken.
- Leviathan's 25–33 hits land for a survivable but real chunk; the **cold breath** (30 cold AoE) is the pressure test — T5 set grants **breath resistance** (halves the slow/damage), not immunity.
- Result: a skilled T5 in a **Barbed-runic Mariner's Leather** suit can farm Leviathans solo in ~2–4 minutes each; a T3 in a **Spined/Horned-runic** Mariner's suit can *start* but will struggle and need consumables. Nothing below T3 can summon one at all.

### Fisher-fighter builds — how players actually run this

On other shards the fisher-fighter is a real, popular archetype. The recurring patterns (for inspiration):
- **Archer / "hide-and-shoot" fisher** — GM Fishing + Archery/Tactics/Anatomy/Healing + a little Magery, **Hiding** while sailing to avoid aggro, then shooting the serpents/elementals that surface ([UO Second Age forums](https://forums.uosecondage.com/viewtopic.php?t=46814)).
- **Fisher-tamer** — Fishing 120 + Taming/Lore/Vet (+Eval) so a pet tanks the sea spawn while you fish ([UO Outlands templates](https://wiki.uooutlands.com/Templates), [UO Addicts builds](https://uoaddicts.com/builds/uo-outlands)).
- **Bard fisher** — Provocation/Peacemaking to turn or pacify sea creatures (safest solo).
- **Harpooner (Outlands model)** — the Fishing skill itself grants **ranged "ocean weapons"** whose special attack is **Impale** (hits nearby targets too); leather **fishing nets are tailor-crafted from the leather tiers** ([UO Outlands Fishing wiki](https://wiki.uooutlands.com/Fishing)). This is the closest match to our leather-armor sea-hunter fantasy.

**On Sphere51a everyone GMs every skill, so a "template" isn't a skill spread — it's a talisman + gear identity.** The Mariner talisman and Mariner's Leather suit support all four of the above (leather fits archer/dexer/bard/tamer alike; the vs-sea bonuses are combat-style agnostic). We lean into the **harpooner** as the signature.

### Signature weapon — the Harpoon (ocean weapon)

A new craftable **Harpoon** (tailor/tinker; also a fishing catch) is the Mariner's flagship weapon, borrowing Outlands' model:
- Ranged throwing weapon usable **with a fishing pole’s stamina economy** (no mount needed — you already can't fish mounted).
- Special: **Impale** — a short-range line/AoE strike, ideal for the 2–5 creature net swarms.
- The talisman's vs-sea `+dmg%` (Part 3) and T3 **Harpoon** / T5 **Krakensbane** abilities are tuned around it, so a leather Mariner out-damages a plate character *against sea creatures only*.
- Non-Mariners can swing it, but without an active Mariner talisman it gets none of the vs-sea scaling — an ordinary, unremarkable weapon.

---

## Part 4 — The Mariner's Leather armor path

**One** new leather material — **Mariner's Leather** — obtained by carving sea creatures, tuned to the ocean threat (physical + cold). It follows our existing standards exactly: it is a `CraftResource`, its **base resist profile** is its identity, and its **magic-property tier comes from the existing runic sewing kits — of which there are only three** (Spined, Horned, Barbed). We do **not** add a 4th runic tier.

### How our leather tiers actually work (the standard we follow)

Each leather resource carries two things (`Misc/ResourceInfo.cs`): a flat **base resist** bonus (`ArmorXResist`) applied to every crafted piece, and the **runic roll parameters** (`RunicMinAttributes`/`MaxAttributes`, `RunicMinIntensity`/`MaxIntensity`) used when a matching **runic sewing kit** enchants the piece. The runic kit is where the *stat tier* comes from. Confirmed values:

| Leather tier | Base resist (P/F/C/Po/E) | Runic kit props | Runic intensity (ML) | Kit charges |
|---|---|---|---|---|
| Regular | 0 / 0 / 0 / 0 / 0 | — | — | — |
| **Spined** | **+5** / 0 / 0 / 0 / 0 (+40 Luck) | 1–3 | 40–100% | 45 |
| **Horned** | +2 / +3 / +2 / +2 / +2 | 3–4 | 45–100% | 30 |
| **Barbed** | +2 / +1 / +2 / +3 / **+4** | 4–5 | 50–100% | 15 |

So the three runic sewing kits (Spined / Horned / Barbed) **are** the three stat tiers. `RunicSewingKit` charges = `60 − type*15` (`Rewards.cs:843`). Note the runic pipeline already special-cases leather: for `RegularLeather..BarbedLeather` it removes Lower-Requirements and Durability-Bonus from the roll pool (`BaseRunicTool.cs:643`).

### Where Mariner's Leather stands in the list

It stands at the **top of the leather list, as a premium 4th material** — but it introduces **no new runic tier**. Its magic stats are rolled with the **existing three runic kits**, exactly like Barbed. What makes it distinct is a **sea-tuned base resist profile** (physical + cold), which no vanilla leather offers (Spined is pure physical; nothing vanilla exceeds +2 cold):

```
Regular  <  Spined  <  Horned  <  Barbed  <  Mariner's   (material list)
                    Spined / Horned / Barbed              (the 3 runic stat tiers — unchanged)
```

| New resource (`Misc/ResourceInfo.cs`, new `CraftResource` + `CraftAttributeInfo`) | Phys | Fire | Cold | Poison | Energy | Total | Runic |
|---|---|---|---|---|---|---|---|
| **Mariner's Leather** | +4 | +1 | +4 | +1 | +1 | 11 | uses existing Spined/Horned/Barbed kits |

Total (11) is a **sidegrade**, not power-creep — equal to Horned, under Barbed (12) — but redistributed into physical + cold, i.e. best-in-slot **only vs the sea**. Your suit's stat tier is still the kit you finish it with: a **Barbed-runic Mariner's Leather** suit (4–5 mods @ 50–100% + sea resists) is the end-game piece.

### Acquisition & crafting (`Engines/Craft/DefTailoring.cs`)
- **Hides:** add one `HideType.Mariner` value; set `Hides`/`HideType` on sea creatures so a `SkinningKnife` cut yields **Mariner's Leather** (mirrors `BaseCreature.OnCarve`). Leviathan yields the most; serpents/Kraken yield less. Acquisition is hard-gated by Mariner tier (Part 1) — everyone can GM Tailoring, but only an active Mariner can *get the hide*, which is the whole "talisman gives the value" principle.
- **Sub-resource:** add Mariner's Leather to the tailoring `AddSubRes` list (skill gate moot on a GM-all shard; the real gate is hide access).
- **Runic range:** extend the leather special-case (`BaseRunicTool.cs:643`) to include Mariner's Leather so it behaves like leather (no lower-req / durability mods).
- **Enhance / exceptional:** standard `Enhance` flow (can fail/destroy — a sink) and the normal exceptional resist distribution (14–15 pts, or 6 under runic) apply unchanged.
- **Bootstrap:** T3 lets you summon & kill Leviathans in a **Spined- or Horned-runic** Mariner's suit (hides farmed from freely-summoned Kraken/serpents), then re-craft to **Barbed-runic** as Mariner's Leather accumulates. No hard wall.

### The Mariner Set bonus (new system: `MarinerSet`)

There is **no set-bonus system in the codebase today**, so this is a new Sphere51a mechanic (consistent with how our other systems are added). A `MarinerSetBonus` service counts equipped **Mariner's Leather** pieces and, **while a Mariner talisman is equipped and active**, applies escalating bonuses that only matter vs `ISeaCreature`. Set bonuses are **PvM-only** and disable in PvP with the talisman; the armor's own resist is passive and always on.

| Pieces | Set bonus (vs sea creatures unless noted) |
|---|---|
| 3 | +5 physical & +5 cold resist vs sea creatures; +10 max stamina |
| 5 | +10 damage vs sea creatures; +5% Hit Stamina Leech vs sea creatures |
| 6 (full) | **Sea Legs**: cold-breath resistance (halve Leviathan/Kraken cold breath), water-walk while a fishing pole is equipped, +5% all sea bonuses |

Set magnitude scales with talisman tier (`0.4×` @ T1 → `1.0×` @ T5), so a full suit previews at T1 and pays off at T5. Resist top-ups still respect the **70 cap** (`Mobile.MaxPlayerResistance`).

### Why leather (not plate)
- Thematic: anglers/sailors wear leather.
- Balance: sea creatures are ~physical, which plate already resists well — routing the sea power budget through *leather-only, sea-only* bonuses keeps the Mariner strong at its job without buffing general plate PvM.
- Synergy: leather keeps stamina/swing high (no med-armor penalty), matching the dexer/harpooner sea-hunter fantasy (Part 5).

---

## Part 5 — End-game feel & balance

The "max tier feels good, worth wearing, not super easy" target:

- **Worth wearing:** the Mariner is the *only* way to access the SOS/treasure/Leviathan/artifact economy and the only efficient Leviathan killer. A GM-everything character without it catches junk and fights Leviathan in generic leather (base ~2 phys/piece) — brutal.
- **Feels good:** T5 + full Leviathan-hide suit hits the ~70/65 sea-mitigation target, leeches through the cold breath, and clears Leviathans in minutes with best-in-slot artifact/ancient-SOS odds.
- **Not super easy:** Leviathan keeps 1500 HP, 25–33 physical + 30 cold breath AoE; below T3 you can't even summon one; the top hide only drops from Leviathans; ~15,000 Fathoms is weeks of committed play; enhancing can destroy pieces.
- **Chase items:** big-fish weight records (3–200 stones, hued at T5), 1%→8% rare-hue nets, and the 23-item Leviathan artifact table give the vertical its own prestige loot.

### Anti-abuse
- Fathoms per-hour cap; minimum-damage credit on kills; shared credit by damage on group Leviathan kills.
- PvP disable applies to all combat + set bonuses and pauses talisman-gated catches.
- Rare-hue net & artifact rolls server-logged for balance tuning.

---

## Part 6 — Economy & acquisition

- **Talisman acquisition:** craft from relics (per `PvM_Talismans_Design.md`) or a fishing-quest reward; tradeable until first equip, then bound to the earner's Fathoms progression.
- **Anglers are suppliers:** rare-hue nets, White Pearls, a shard-unique catch (e.g. *Abyssal Roe* → alchemy/cooking), **Mariner's Leather hides**, and Leviathan artifacts all originate from Mariners, feeding tailoring, cooking, aquarium décor, and the artifact market. Value comes from the wider economy, not a solo stat stick.
- **Sinks:** enhancing destruction, pole/net consumption, boat upkeep, travel.

---

## Part 7 — Code integration points

| Change | Location | Risk |
|---|---|---|
| Add `Mariner` to talisman archetype enum + JSON definition | `PvMBuild`/`TalismanType`, `Data/Talismans/` | Low |
| `BuildManager.GetMarinerTier(Mobile)` + Fathoms award/caps | new `Systems/Sphere51a/BuildManager.cs` | Med |
| Re-gate mutate table by tier | `Fishing.MutateType` (`Fishing.cs:162`) | Med |
| Gate SOS chest fill / shipwreck-only-without-talisman | `Fishing.Construct` (`:213`) | Med |
| Gate net/treasure/Leviathan handoff | `Fishing.Give` (`:324`) | Med |
| Gate Fabled net & Leviathan summon | `SpecialFishingNet.cs` (`FabledFishingNet`) | Low |
| Gate & scale artifact roll by tier | `Leviathan.OnDeath` | Low |
| Award Fathoms on catch / sea kill | `Fishing.OnHarvestFinished` (`:473`), creature `OnDeath` | Med |
| `ISeaCreature` marker on 5 creatures | each creature file | Low |
| One new `CraftResource` + `CraftAttributeInfo` (Mariner's Leather; reuses existing runic tiers) | `Misc/ResourceInfo.cs` | Low |
| Extend leather range special-case to include Mariner's Leather | `BaseRunicTool.cs:643` | Low |
| One new `HideType.Mariner`; hide drops on sea-creature carve (tier-gated) | `HideType` enum, creature `OnCarve`/loot | Low |
| Mariner's Leather sub-resource + craftables | `Engines/Craft/DefTailoring.cs` | Low |
| Harpoon ocean weapon + Impale special | new `Items/Weapons/…`, weapon ability | Med |
| `MarinerSetBonus` service + resist/combat hooks | new `Systems/Sphere51a/MarinerSet.cs`, `BaseArmor`/`AOS.cs` | High |
| vs-sea damage bonus/reduction in damage pipeline | `Misc/AOS.cs` / `BuildManager.ApplyTalismanBonus` | High |

Mark all edits `//Sphere-style edit` per project convention.

---

## Part 8 — Testing matrix

| Test | Setup | Expected |
|---|---|---|
| No talisman, GM Fishing, deep water | no Mariner equipped | only fish/boots/junk; no net/SOS/big-fish |
| T1 fishing | Deckhand | nets + soft big/magic fish; no SOS loop |
| T2 SOS loop | Fisher | can fish up & recover SOS chests; ancient 1/25 |
| T3 Leviathan summon | Mariner, Fabled net | Leviathan spawns; artifact 25% on kill |
| Tier scaling | T5 | ancient 1/8, artifact 45%, hue 8%, big-fish records |
| vs-sea dmg bonus | T5 vs Kraken | +25% dealt, −20% taken |
| vs-sea bonus in PvP | flagged, hit player | all sea bonuses + set bonus disabled 5 min; armor resist unchanged |
| Set bonus gating | full suit, no talisman | no set bonus (needs active Mariner) |
| Set bonus vs non-sea | full suit vs dragon | no set bonus applies |
| Sea Legs | 6-pc T5 vs Leviathan breath | cold breath halved; water-walk with pole |
| Resist cap | stacked cold | capped at 70 |
| Fathom cap | farm >400/hr | overflow not credited |
| Enhance destroy | enhance with Mariner's Leather | can fail/destroy piece |
| Runic tier | Barbed-runic vs Spined-runic Mariner's suit | Barbed rolls 4–5 mods @50–100%; Spined 1–3 @40–100% |

---

## Part 9 — Configuration

```csharp
// Sphere51aConfig.cs additions (all tunable without recompile where possible)
public int   MarinerFathomsPerHourCap        { get; set; } = 400;
public double MarinerSeaDamageBonusT5         { get; set; } = 0.25;
public double MarinerSeaDamageReductionT5     { get; set; } = 0.20;
public int[]  MarinerTierThresholds           { get; set; } = { 0, 500, 2000, 6000, 15000 };
public double MarinerAncientSosChanceT5       { get; set; } = 1.0 / 8.0;
public double MarinerLeviathanArtifactChanceT5{ get; set; } = 0.45;
public double MarinerSetTierScaleMin          { get; set; } = 0.4; // T1 multiplier
```

| Key | Type | Default |
|-----|------|---------|
| `mariner.fathoms_per_hour_cap` | int | 400 |
| `mariner.tier_thresholds` | int[] | 0,500,2000,6000,15000 |
| `mariner.sea_damage_bonus_t5` | decimal | 0.25 |
| `mariner.sea_damage_reduction_t5` | decimal | 0.20 |
| `mariner.ancient_sos_chance_t5` | decimal | 0.125 |
| `mariner.leviathan_artifact_chance_t5` | decimal | 0.45 |
| `mariner.set_tier_scale_min` | decimal | 0.4 |

---

## Open design questions

1. **Throwing/archery vs melee** for the sea-hunter — should Sea Legs’ water-walk enable kiting Leviathan, or keep it melee-committed?
2. **Charybdis / High Seas content** — none exists today; a Charybdis boss could sit above Leviathan as a T5 group target.
3. **Aquarium tie-in** — should Mariner tier boost aquarium reward-fish odds to link the two fishing systems?
4. **Talisman acquisition** — relic-craft (reuse PvM path) vs a dedicated fishing quest chain.

## Change Log

### v1.1.0 - 2026-07-03
- **Armor corrected to shard standards:** collapsed the three invented hide tiers into a **single** Mariner's Leather `CraftResource`; stat tiers now come from the **existing three runic sewing kits** (Spined/Horned/Barbed) — no new runic tier. Documented the confirmed runic values (props/intensity/charges) and placed Mariner's Leather at the top of the material list as a physical/cold **sidegrade** (total 11).
- Added **fisher-fighter build archetypes** (archer/hide-and-shoot, fisher-tamer, bard, harpooner) drawn from UO Outlands/Second Age, framed for a GM-all-skills shard where identity = talisman + gear.
- Added the signature **Harpoon** ocean weapon with an **Impale** special (Outlands-inspired), tuned around the talisman's vs-sea bonuses.
- Updated Part 1 mats row, encounter math, economy, testing matrix, and code-hook table accordingly.

### v1.0.0 - 2026-07-03
- Initial full playstyle design: fishing reward re-gate (hard SOS/Leviathan/artifact, soft big/magic fish), 5-tier Mariner talisman with Fathoms progression, sea-creature & Leviathan combat bonuses, linked Mariner's Leather armor path, end-game balance targets, economy, PvP integration, and code hook map.
