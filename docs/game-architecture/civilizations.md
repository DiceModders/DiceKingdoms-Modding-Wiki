---
summary: Everything in the game code that depends on a civilization - the enum, skins and biomes, tech trees, AI build orders, networking, text - and what each does with a number it does not know
---

# Civilizations in the game code

The game has five civilizations, the enum `DiceKingdoms.Interface.Civilization`:

| Value | Name | Biome it uses |
|---|---|---|
| 0 | `EUROPE` | `PLAINS` |
| 1 | `MIDDLE_EAST` | `DESERT` |
| 2 | `VIKING` | `SNOW` |
| 3 | `ANCIENT` | `MEDITERRANEAN` |
| 4 | `MAYA` | `JUNGLE` |

The civilization is almost not a gameplay concept: it chooses **how a player looks**, **which tech tree the player gets**, **which AI build orders an AI uses** and **what the civilization is called**.
Buildings, units, costs and the technologies themselves are the same for everybody. This page lists every place where the number matters and what that place does with a number it does not know,
which is what a mod civilization has to deal with. [Making a civilization](../guides/modding/making-a-civilization.md) is the practical guide.

## Where the number is used

| Where | What it does | With an unknown number |
|---|---|---|
| `PlayerSkins.Clone()` | Calls the constructor with the real number (`Rules.OnEnable` clones the lobby's skins when the Rules screen opens). | Throws the same exception when the Rules screen opens, which breaks it. |
| `PlayerSkins` constructor and `Civilization` setter | Stores the number and **derives the biome** with a switch over 0..4 (the table above). | Throws `ArgumentOutOfRangeException` (`Parameter name: civilization`). |
| `PlayerSkins.Get{Building,Unit,Nature,InnerCliff}...Prefab`, `GetShipVisualPrefab` | Ask the skin registry for the replacement of a default prefab for this civilization (see below). | Finds nothing and silently returns the default (European) prefab. |
| `PlayerSkinsSerialization` | Sends the civilization over the network as **one byte** (`WritePlayerSkins` writes a byte, `ReadPlayerSkins` and the network variable's stream reader read an `sbyte`) and builds `PlayerSkins` from it. | Throws in the constructor; the receiving game freezes. A number above 127 arrives as another number. |
| `Technologies.TechTree(civilization)` | `First(...)` over the game's list of `TechTree` assets, matching `TechTree.civilization`. The compiler inlined it into `Statistics.get_TechTree` and `Statistics.IsTechnologyLearnable`, so those throw too. | Throws `InvalidOperationException: Sequence contains no matching element`; `TechTreeInterface.Update` asks every frame, so the log floods and the research window is dead. |
| `AIPlay.SampleBuildOrder(random, civilization, strategies)` | Keeps the build orders written for exactly that civilization. | Throws `No build order matching civilization=N, strategies=...`. |
| `Player.NextCivilization` | The castle click on the match-start screen: steps through `Player.Inventory.UnlockedCivilizations` (Steam inventory) and wraps around. | Only unlocked civilizations are ever offered. |
| `ITextElement.Localized` with the key `civilization.{value}.name` | The civilization's name in the text table. | A missing-translation text. |
| `Rules` window | The civilization drop-down lists `Enum.GetValues(typeof(Civilization))`. | Not listed; a mod has to append its entries (see [Rules and civilizations](rules-and-civilizations.md)). |

Not used by gameplay: `IconRenderer` has one sprite atlas per civilization for buildings and units (an icon-rendering tool, with `buildingIconsFolder` and `testFolder` fields; the icons in
the game come from the prefabs, below), and `IslandGrid.testCivilization`, `TestCombat.attackCivilization/defenseCivilization` and `Player.TestInit` are test code.

## Skins: what a civilization looks like

`PlayerSkins` is a small object (civilization, biome, a `SkinRegistry`, and caches). It hands out prefabs:

| Method | Returns | Original (default) prefab comes from |
|---|---|---|
| `GetBuildingVisualPrefab(BuildingType)` | `BuildingVisual` | `BuildingType.visualPrefab` |
| `GetUnitVisualPrefab(Unit.Type)` | `UnitVisual` | `DefaultPrefabs.Unit(type)` |
| `GetNatureVisualHubPrefab(NatureTile)` | `NatureVisualHub` | `DefaultPrefabs.Nature(tile)` |
| `GetInnerCliffVisualHubPrefab(InnerCliff.Key)` | `InnerCliffVisualHub` | `DefaultPrefabs.InnerCliff(key)` |
| `GetShipVisualPrefab()` | `ShipVisual` | `DefaultPrefabs.Ship` |

Each call is `SkinRegistry.GetSkinVariant(original, biome, civilization, variant)`, which looks up a key of (original prefab, optional biome, optional civilization, variant text) in a dictionary
built from the registry's `entries` list (`SkinRegistry.Awake` runs `OnValidate`, which builds it) and **returns the original when there is no entry**. The player skins cache each result.
So the five built-in civilizations are not code: they are entries of the skin registry that replace default prefabs by civilization, and a civilization with no entries looks European.
Results are cached per `PlayerSkins`, so a skin change after a player has been drawn needs a new `PlayerSkins` object.

The types a prefab needs (fields of the game's own components; names are from the IL2CPP dump):

| Prefab | Component | Main parts |
|---|---|---|
| Building | `BuildingVisual` | `visual` and `damagedVisual` (the models), `firesHolder`, `onlyEnableWhenActive`, `buildingTints` (materials that take the player colour), `hitSound`, `fireSound`, `placeParticlesPrefab`, `damageParticles`, `remainsParticles`, `icons` |
| Unit | `UnitVisual` | `unitType`, `animator` (parameters Move, Attack, AttackSpeed, Run), `particlesParryPrefab`, `particlesDiePrefab`, `particlesConvertPrefab`, `buildingTints`, `icons`, sounds (parry, die, walk, convert) |
| Ship | `ShipVisual` | `unitVisualsHolder` (soldiers stand on it), `animator` (parameters Deploy, Sand, Move), `particles`, `hideParent`, `colliders`, sounds (sail, deploy) |
| Nature | `NatureVisualHub` | `natureTile` and a list of `NatureVisual` children; `SampleVisual(random)` picks one |
| Cliff | `InnerCliffVisualHub` | `key` and a list of transforms, one picked at random |

The tile kinds and unit kinds:

- `BuildingTypeName` (24): ARCHERY_RANGE, BARRACKS, CASTLE, CATHEDRAL, CHURCH, COTTAGE, DOCK, FARM, GUARD_TOWER, GUARD_TOWER_UP, HOSPITAL, HOUSE, LUMBER_CAMP, MARKET, SORCERER_TOWER, STONE_QUARRY,
  STORAGE, UNIVERSITY, WALL, WINDMILL, WONDER, FISHING_SHIP, LIGHTHOUSE, LAND_CAMP
- `Unit.Type` (10): ARCHER, BOURGEOIS, GENERAL, KING, PEASANT, PRIEST, SOLDIER, SORCERER, TEMPLAR, WARLOCK
- `NatureTile` (6): TREE, ROCK, SCORCHED_EARTH, LAVA, FLOWER, FISH

**Icons.** `BuildingVisual.Icon(PlayerColor)` and `UnitVisual.Icon(PlayerColor)` return `icons[color.Index]`: **one sprite per player colour**, so a model with its own icon needs a full set.
Technologies draw their icon with `Technology.Icon(PlayerSkins, PlayerColor)`: the "unlock building" technologies use the icon of the building in the player's skin (so a civilization's own model
changes the research icon too), every other technology (`IconTechnology`) has one fixed `icon` sprite.

**Biome.** The biome (ground, trees, rocks) is chosen with the civilization in `PlayerSkins` and **cannot be set separately**: a mod civilization takes the biome of the built-in civilization it
is built "like". There is no way to give it a sixth biome through the skins alone.

## Tech trees

- `Technologies` (one asset) holds the **22 shared technologies**; `Technology` is abstract (`name`, `cost`, `Icon`, `OnLearn`). A technology never belongs to a tree.
- `TechnologyName`: ADDITIONAL_ROLL 0, DOUBLE_CLEAN_TILE 1, UPGRADE_GUARD_TOWER 2, TEMPLARS 3, WARLOCKS 4, UNLOCK_STORAGE 5, UNLOCK_LUMBER_CAMP 6, UNLOCK_STONE_QUARRY 7, UNLOCK_GUARD_TOWER 8,
  UNLOCK_SORCERER_TOWER 9, UNLOCK_HOSPITAL 10, UNLOCK_MARKET 11, UNLOCK_CATHEDRAL 12, UNLOCK_WINDMILL 13, UNLOCK_ARCHERY_RANGE 14, UNLOCK_WONDER 15, UNLOCK_FISHING_SHIP 16, UNLOCK_LIGHTHOUSE 17,
  UNLOCK_LAND_CAMP 18, RITUALS 19, SHIP_HOLD_OPTIMIZATION 20, TRADE_EFFICIENCY 21.
- A `TechTree` asset (`civilization`, `slots`) chooses and arranges them. A `Slot` has a `technology`, a grid `position` (`int2`) and `links` (`toTechnology`, `toSlot`, `fromPort`, `toPort`).
  The research window draws exactly these: slots as boxes, links as lines.
- **Dependencies are the links.** `GetDependencies(technology)` returns the technologies of the slots that link to it, and caches the answer in a dictionary (`OnValidate` clears the cache).
- **Learnable rule** (`Statistics.IsTechnologyLearnable`): not already learnt; in the Steam demo only `UNLOCK_STORAGE` can be learnt; and either it has no dependencies or at least one dependency is learnt.
  Which tree applies is `Statistics.TechTree`, from `Statistics.island.Skins.civilization`.
- `Technologies.TechTree(civilization)` is the lookup that every tree request goes through (and is inlined into the two `Statistics` members above).

## AI build orders

`AIPlay` loads 15 build orders (3 per built-in civilization: `culture`, `wonder`, `military_macro`) from **text embedded in the game**, in a small format:

```
civilization=VIKING weight=2 strategy=MILITARY name=military_macro [
    technologies [ UNLOCK_FISHING_SHIP UNLOCK_ARCHERY_RANGE ... ]
    buildings [ house house church cot cot ... ]
]
```

The civilization is a `Civilization` enum name, the strategy is `WONDER` or `MILITARY`, and `buildings` is the order in which the AI builds, written with short words (house, cot,
church, farm, racks, dock, land, storage, mill, fish, guard, cat, university, market, lumber, quarry, archery, wonder, sorcerer; they read as HOUSE, COTTAGE, CHURCH, FARM, BARRACKS, DOCK, LAND_CAMP, STORAGE, WINDMILL,
FISHING_SHIP, GUARD_TOWER, CATHEDRAL, UNIVERSITY, MARKET, LUMBER_CAMP, STONE_QUARRY, ARCHERY_RANGE, WONDER, SORCERER_TOWER). The text is a compiled-in constant, so a mod cannot add orders; a mod civilization's AI uses
the orders of the civilization it is "like".

## Networking

A player's `PlayerSkins` travels as **one byte** (the sender writes a byte, the receiver reads an `sbyte` and calls the constructor). So a civilization number must fit in a signed byte, and 0..4 are taken:
**mod civilizations get 5..127** (123 numbers). Both sides must know the number: a client that does not have the mod would throw in the constructor on receiving it. A mod therefore has to
make sure every lobby member has a civilization before it can be picked (for example by publishing its civilizations in the Steam lobby member data, offering the castle click only civilizations
everybody has, and moving a player off a civilization somebody lacks).

## Localization

Text goes through Unity Localization. `StructuredTextStaticReferences.instance` has the `localizationSettings` and the string table (`localizationStringTable`); `ITextElement.Localized.Render` looks a key up
there. The civilization name key is `string.Format("civilization.{0}.name", civilization)`: `civilization.EUROPE.name` for a built-in one and `civilization.14.name` for a mod one (a boxed enum without a
name prints its number). The table of a language is created when the language loads, so an entry has to be added to the table of the language in use and again when the language changes.
