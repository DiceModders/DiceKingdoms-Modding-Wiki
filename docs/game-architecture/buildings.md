---
summary: Building type ids, footprint shapes, rotation/mirror math and placement rules
---
# Buildings

## BuildingType

Each building kind is a `BuildingType` object held in a `BuildingTypes` collection (referenced by `RulesBuilding`). The **index in `BuildingTypes.Types` is the id used in save files and island codes**.

```csharp
namespace DiceKingdoms.Game;

[Serializable]
public class BuildingType
{
    public BuildingTypeName name;
    public int2[] localPositions;
    public int2[] localPositionsInWater;
    public HeightOverride[] heightOverrides;
    public BuildingVisual visualPrefab;
    public ResourceAmount storageCapacity;
    public ResourceAmount cost;
    public int tier;
    public int bulkSize;
    public int health;
    public bool fortification;
    public bool garrison;
    public ResourceAmount pillageReward;
    public ResourceAmount remainsReward;
    public Opt<Tower> tower;
    public bool hasEffectDescription;

    public string DisplayName { get; }
    public bool IsBuildable { get; }
    public bool HasEffectDescription { get; }

    public bool IsWaterPosition(int2 localPosition);
    public float? GetHeightOverride(int2 localPosition);
    public override string ToString();
    public virtual bool Equals(BuildingType other);
}
```

**Fields.** `name` - the type identifier from the `BuildingTypeName` enum. `localPositions` - the tiles the building occupies, in building-local coordinates. `localPositionsInWater` - footprint variant for water. `heightOverrides` - per-tile visual height overrides. `visualPrefab` - the visual prefab. `storageCapacity` - storage capacity granted by the building. `cost` - building cost. `tier` - technology tier (1–3). `bulkSize` - bulk placement size. `health` - maximum health. `fortification` / `garrison` - "fortification" and "garrison" flags. `pillageReward` / `remainsReward` - rewards for pillaging and for being reduced to remains. `tower` - optional tower data. `hasEffectDescription` - whether an effect description is shown.

**Properties.** `DisplayName` returns the localized name. `IsBuildable` determines whether the type appears in the build menu. `HasEffectDescription` wraps the field.

**Methods.** `IsWaterPosition` checks whether a local offset is a water tile for this type. `GetHeightOverride` returns the height override for a specific tile.

!!! note "Offsets are build specific"  
    Offsets and RVAs on this page come from one game build (see [Internals Reference](../reference/internals.md)). Field _names_ are stable; numbers may move after a game update.

| Field | Type | Offset | Notes |
| --- | --- | --- | --- |
| `id` | `int` | `0x10` | The type id (equals the index in `BuildingTypes.Types`) |
| `localPositions` | `int2[]` | `0x18` | Footprint, in building-local coordinates |
| `localPositionsInWater` | `int2[]` | `0x20` | Footprint variant used in water |
| cost block | struct | `0x58 - 0x74` | resource cost |
| `tier` | `int` | `0x78` | 1-3, see the table below |
| `maxHealth` | `int` | `0x80` | Maximum health, see Health |

### Class hierarchy

Not every building type is a plain `BuildingType`. The game uses a small hierarchy of subclasses to attach specialized behaviour:

*   `BuildingType` (base)

    *   `BuildingTypeHousing` - housing (rolls a die for yield)

        *   `BuildingTypeCastle` - castle

        *   Cottage, Church, House, Barracks, Sorcerer Tower

    *   `BuildingTypeProducer` (abstract)

        *   `BuildingTypeProducerFixed` - Farm

        *   `BuildingTypeProducerNature` - Lumber Camp, Stone Quarry, Fishing Ship

        *   `BuildingTypeProducerAmplifier` - Cathedral, Windmill, Market

    *   `BuildingTypeDock` - Dock

    *   `BuildingTypeHospital` - Hospital

    *   `BuildingTypeSoldierFamily` - Archery Range

    *   `BuildingTypeTech` - University

    *   `BuildingTypeWonder` - Wonder

    *   `BuildingTypeApplyModifierInRange` - Lighthouse

    *   `BuildingTypeLandExtension` - Land Camp (implements `IBuildingTypeUpdateOnHarvest`)

    *   Plain `BuildingType`: Wall, Guard Tower, Guard Tower Up, Storage

## Specialized types

### BuildingTypeHousing

Residential buildings that produce a unit and yield resources via a die roll.

```c#
namespace DiceKingdoms.Game;

[Serializable]
public class BuildingTypeHousing : BuildingType
{
    public ResourceAmount[] possibleYields;
    public Color diceColor;
    public Unit.Type unitType;
    public Dice dicePrefab;
}
```

**Fields.** `possibleYields` - possible die outcomes. `diceColor` - die color. `unitType` - the unit type produced. `dicePrefab` - die prefab.

### BuildingTypeCastle

Castle. Extends `BuildingTypeHousing` without adding any fields of its own.

```c#
namespace DiceKingdoms.Game;

[Serializable]
public class BuildingTypeCastle : BuildingTypeHousing
{
}
```

### BuildingTypeProducer

Abstract base class for all resource producers. Defines a single abstract method.

namespace DiceKingdoms.Game;

```c#
[Serializable]
public abstract class BuildingTypeProducer : BuildingType
{
    public abstract ResourceAmount Yield(Island island, Building building);
}
```

**Methods.** `Yield` returns the amount of resources produced for a specific building on a specific island. Each subclass implements it differently.

### BuildingTypeProducerFixed

A producer with a fixed yield - for example, a farm.

```c#
namespace DiceKingdoms.Game;

[Serializable]
public class BuildingTypeProducerFixed : BuildingTypeProducer
{
    public ResourceAmount yield;

    public override ResourceAmount Yield(Island island, Building building);
}
```

**Fields.** `yield` - always the same amount of resources, regardless of surroundings. `Yield` simply returns `yield`.

### BuildingTypeProducerNature

A producer whose yield depends on nearby nature tiles - lumber camp, stone quarry, fishing ship.

```c#
namespace DiceKingdoms.Game;

[Serializable]
public class BuildingTypeProducerNature : BuildingTypeProducer
{
    public Resource yieldResource;
    public NatureTile natureTile;
    public int tilesNeeded;

    public Resource YieldResource { get; }
    public NatureTile NatureTile { get; }
    public int TilesNeeded { get; }

    public int NatureNeighborsCount(Island island, Building building);
    public override ResourceAmount Yield(Island island, Building building);
    public float AverageYield(Island island, Building[] buildings);
}
```

**Fields.** `yieldResource` - which resource is produced. `natureTile` - which nature tile is counted. `tilesNeeded` - how many such tiles are needed for full output.

**Methods.** `NatureNeighborsCount` counts the nature tiles of the required type around the building. `Yield` multiplies that count by the base rate. `AverageYield` computes the average yield across an array of buildings - used in UI and for previews.

### BuildingTypeProducerAmplifier

Amplifier - boosts the production of another building type. For example, windmill, market, cathedral.

```c#
namespace DiceKingdoms.Game;

[Serializable]
public class BuildingTypeProducerAmplifier : BuildingTypeProducer
{
    public BuildingTypes buildingTypes;
    public Resource yieldResource;
    public BuildingTypeName amplifiedBuildingTypeName;
    public int amplifyFactor;
    public int amplifyFactorProximityBonus;

    public Resource YieldResource { get; }
    public int AmplifyFactor { get; }
    public int ProximityBonus { get; }
    public BuildingType AmplifiedBuildingType { get; }

    public int AmplifiedCount(Island island);
    public int AmplifiedProximityCount(Island island, Building building);
    public float AverageAmplifiedProximityCount(Island island, Building[] buildings);
    public override ResourceAmount Yield(Island island, Building building);
    public float AverageFactor(Island island, Building[] buildings);
}
```

**Fields.** `buildingTypes` - the type registry. `yieldResource` - the resource produced. `amplifiedBuildingTypeName` - which building type is amplified. `amplifyFactor` - the base amplification factor. `amplifyFactorProximityBonus` - bonus for being near the amplified building.

**Methods.** `AmplifiedCount` counts all amplified buildings on the island. `AmplifiedProximityCount` - those within range. `Yield` combines the base factor and the proximity bonus. `AverageFactor` returns the average multiplier across an array of buildings.

### BuildingTypeSoldierFamily

Archery range - binds the building to a unit family.

```c#
namespace DiceKingdoms.Game;

[Serializable]
public class BuildingTypeSoldierFamily : BuildingType
{
    public Unit.Family soldierFamily;

    public Unit.Family SoldierFamily { get; }
}
```

**Fields.** `soldierFamily` - the unit family produced by the building.

### BuildingTypeTech

```c#
namespace DiceKingdoms.Game;

[Serializable]
public class BuildingTypeTech : BuildingType
{
    public int unlocksTier;
}
```

### BuildingTypeApplyModifierInRange

Lighthouse - applies a modifier to buildings of a specific type within range.

```c#
namespace DiceKingdoms.Game;

[Serializable]
public class BuildingTypeApplyModifierInRange : BuildingType
{
    public BuildingTypes buildingTypes;
    public int range;
    public BuildingModifier modifier;
    public BuildingTypeName affectedBuildingType;

    public int Range { get; }
    public BuildingModifier Modifier { get; }
    public BuildingType AffectedBuildingType { get; }

    public bool Affects(Building source, Building target);
    public static bool HasModifier(Island island, BuildingModifier modifier, Building building);
}
```

**Fields.** `buildingTypes` - the type registry. `range` - the range of effect. `modifier` - the modifier applied (see `BuildingModifier`). `affectedBuildingType` - the building type the modifier acts on.

**Methods.** `Affects` checks whether a specific target is affected by the source. `HasModifier` - a static helper: whether a building on the island has the given modifier.

### BuildingTypeLandExtension

Land camp - extends land on each harvest.

```c#
namespace DiceKingdoms.Game;

[Serializable]
public class BuildingTypeLandExtension : BuildingType, IBuildingTypeUpdateOnHarvest
{
    public int range;
    public int tilesPerTurn;

    public void ApplyHarvest(Island island, Building building);
    public int2[] AffectedPositions(Island island, Building building);
}
```

**Fields.** `range` - the extension radius. `tilesPerTurn` - how many tiles are added per harvest.

**Methods.** `ApplyHarvest` is called by the engine after the harvest phase. `AffectedPositions` returns the list of tiles that will be affected.

### IBuildingTypeUpdateOnHarvest

Interface for types that react to harvest.

```c#
namespace DiceKingdoms.Game;

public interface IBuildingTypeUpdateOnHarvest
{
    void ApplyHarvest(Island island, Building building);
}
```

**Methods.** `ApplyHarvest` is called for every building of that type after the harvest phase completes.

## HeightOverride

Per-tile visual height override.

```c#
public struct HeightOverride
{
    public int2 position;
    public float height;
}
```

**Fields.** `position` - the tile's local coordinate. `height` - the visual height at that tile.

## BuildingModifier

Modifier applied by effect buildings. Currently only one exists.

```c#
public enum BuildingModifier
{
    DOUBLE_FISH
}
```

**Values.** `DOUBLE_FISH` - doubles fish production for buildings within range.

## Type ids

Ids `0-20` are alphabetical by display name; `21-23` were added later and sit at the end. There is no in-game build menu - buildings are panels at the bottom of the screen; picking one makes it follow the cursor until you left-click.

| Id | Name | Tiles | Tier | Max HP |
| --- | --- | --- | --- | --- |
| 0 | Archery Range | 5 | 2 | 60 |
| 1 | Barracks | 6 | 1 | 60 |
| 2 | Castle | 16 | 1 | 200 |
| 3 | Cathedral | 6 | 2 | 40 |
| 4 | Church | 3 | 1 | 15 |
| 5 | Cottage | 2 | 1 | 10 |
| 6 | Dock | 12 | 1 | 50 |
| 7 | Farm | 8 | 1 | 15 |
| 8 | Guard Tower | 4 | 2 | 40 |
| 9 | Guard Tower Up | 4 | 2 | 40 |
| 10 | Hospital | 5 | 2 | 40 |
| 11 | House | 3 | 1 | 15 |
| 12 | Lumber Camp | 4 | 3 | 20 |
| 13 | Market | 4 | 3 | 10 |
| 14 | Sorcerer Tower | 3 | 2 | 20 |
| 15 | Stone Quarry | 3 | 3 | 20 |
| 16 | Storage | 6 | 2 | 20 |
| 17 | University | 7 | 1 | 50 |
| 18 | Wall | 1 | 1 | 50 |
| 19 | Windmill | 4 | 2 | 20 |
| 20 | Wonder | 15 | 3 | 150 |
| 21 | Fishing Ship | 2 | 2 | 30 |
| 22 | Lighthouse | 4 | 3 | 50 |
| 23 | Land Camp | 4 | 3 | 20 |

### `BuildingTypeName` enum

The `BuildingTypeName` enum lists all types in the same order as the ids.

```c#
public enum BuildingTypeName
{
    ARCHERY_RANGE, BARRACKS, CASTLE, CATHEDRAL, CHURCH, COTTAGE,
    DOCK, FARM, GUARD_TOWER, GUARD_TOWER_UP, HOSPITAL, HOUSE,
    LUMBER_CAMP, MARKET, SORCERER_TOWER, STONE_QUARRY, STORAGE,
    UNIVERSITY, WALL, WINDMILL, WONDER, FISHING_SHIP, LIGHTHOUSE,
    LAND_CAMP
}
```

## Footprint and rotation

`GridTransform.TransformVector` rotates by `orient` (0-3) in 90 degree steps, then `mirror` flips X:

| `orient` | `(cos, sin)` |
| --- | --- |
| 0 | `(1, 0)` |
| 1 | `(0, 1)` |
| 2 | `(-1, 0)` |
| 3 | `(0, -1)` |

```c#
static (int x, int y) Transform(int dx, int dy, int orient, bool mirror)
{
    int[] cos = { 1, 0, -1, 0 };
    int[] sin = { 0, 1, 0, -1 };
    int c = cos[((orient % 4) + 4) % 4], s = sin[((orient % 4) + 4) % 4];

    int nx = dx * c - dy * s;
    int ny = dx * s + dy * c;
    if (mirror) nx = -nx; 
    return (nx, ny);
}

IEnumerable<(int x, int y)> Footprint(BuildingType t, int bx, int by, int orient, bool mirror)
 => t.localPositions.Select(o => { var (x, y) = Transform(o.x, o.y, orient, mirror); return (bx + x, by + y); });
```

**Logic.** The `Transform` function rotates the local offset by `orient * 90°` counter-clockwise, then, if `mirror = true`, reflects the result across X. `Footprint` applies this transformation to all `localPositions` and adds the world position of the building.

Local offsets `(dx, dy)` per type, exactly as read from `localPositions`:

0.  Archery Range `[[0,0],[-1,0],[0,-1],[-1,-1],[-1,-2]]`
1.  Barracks `[[-1,1],[0,1],[0,0],[0,-1],[1,0],[1,-1]]`
2.  Castle `[[-1,-1],[-1,0],[-1,1],[-1,2],[0,-1],[0,0],[0,1],[0,2],[1,-1],[1,0],[1,1],[1,2],[2,-1],[2,0],[2,1],[2,2]]`
3.  Cathedral `[[1,0],[0,0],[-1,0],[-2,0],[0,1],[0,-1]]`
4.  Church `[[0,0],[-1,0],[-2,0]]`
5.  Cottage `[[0,0],[0,1]]`
6.  Dock `[[0,0],[1,0],[2,0],[2,1],[3,0],[3,1],[4,-1],[4,0],[4,1],[5,-1],[5,0],[5,1]]`
7.  Farm `[[0,1],[0,0],[0,-1],[0,-2],[1,1],[1,0],[1,-1],[1,-2]]`
8.  Guard Tower `[[0,0],[-1,0],[0,-1],[-1,-1]]`
9.  Guard Tower Up `[[0,0],[-1,0],[0,-1],[-1,-1]]`
10. Hospital `[[0,0],[-1,0],[1,0],[0,-1],[0,1]]`
11. House `[[0,0],[-1,0],[0,-1]]`
12. Lumber Camp `[[0,0],[-1,0],[-1,1],[-2,1]]`
13. Market `[[0,0],[1,0],[1,1],[0,1]]`
14. Sorcerer Tower `[[0,0],[1,0],[0,1]]`
15. Stone Quarry `[[0,0],[0,1],[1,0]]`
16. Storage `[[0,0],[1,0],[0,-1],[1,-1],[0,1],[1,1]]`
17. University `[[0,1],[0,0],[0,-1],[1,1],[1,-1],[2,1],[2,-1]]`
18. Wall `[[0,0]]`
19. Windmill `[[0,0],[-1,0],[-1,1],[0,1]]`
20. Wonder `[[3,-1],[3,0],[3,1],[2,-1],[2,0],[2,1],[1,-1],[1,0],[1,1],[0,-1],[0,0],[0,1],[-1,-1],[-1,0],[-1,1]]`
21. Fishing Ship `[[0,0],[0,1]]`
22. Lighthouse `[[0,0],[-1,0],[0,-1],[-1,-1]]`
23. Land Camp `[[0,0],[1,1],[0,1],[-1,1]]`

## Building

```c#
namespace DiceKingdoms.Game;

public sealed class Building
{
    public struct Id
    {
        public readonly int value;

        public Id(int value);
        public bool Equals(Id other);
        public override bool Equals(object obj);
        public override int GetHashCode();
        public static Id OfPosition(int2 position);
        public static bool operator ==(Id lhs, Id rhs);
        public static bool operator !=(Id lhs, Id rhs);
        public override string ToString();
        public int CompareTo(Id other);
    }
}
```

**Nested `Id`.** A wrapper over an `int` that identifies a specific building. `OfPosition` creates an id from a grid position - useful when you need to find a building by coordinate without scanning the island's full building list. `Equals`, `GetHashCode`, the `==` / `!=` operators and `CompareTo` make `Id` a full-fledged key for dictionaries and sorting.

## BuildingTypes (registry)

A ScriptableObject-like registry holding every `BuildingType`.

```c#
namespace DiceKingdoms.Game;

public class BuildingTypes
{
    public BuildingType castle;
    public BuildingType wall;
    public BuildingType farm;
    public BuildingType cottage;
    public BuildingType house;
    public BuildingType church;
    public BuildingType barracks;
    public BuildingType guardTower;
    public BuildingType guardTowerUp;
    public BuildingType storage;
    public BuildingType lumberCamp;
    public BuildingType stoneQuarry;
    public BuildingType windmill;
    public BuildingType market;
    public BuildingType cathedral;
    public BuildingType archeryRange;
    public BuildingType dock;
    public BuildingType hospital;
    public BuildingType sorcererTower;
    public BuildingType university;
    public BuildingType wonder;
    public BuildingType fishingShip;
    public BuildingType lighthouse;
    public BuildingType landCamp;

    public BuildingType[] Types { get { } }
    public Dictionary<string, BuildingType> typesByName { get { } }

    public BuildingType Get(BuildingTypeName name) { }

    public void Init() { }
    public void OnEnable() { }
}
```

**Fields.** One strongly-typed field per building kind (`castle`, `wall`, `farm`, `cottage`, …). This is how prefabs and other assets reference a specific type directly, bypassing the array.

**Properties.** `Types` - an array of all types; **index = id**. `typesByName` - a name → type mapping.

**Methods.** `Get` resolves a type by enum value. `Init` builds the lookup structures. `OnEnable` - the standard Unity hook.

## Placement rules

`BuildData.CanBuildType` checks, in order: `IsBuildable` ->`Unlocked` ->`SteamDemo` ->`HasResource`. The per-tile check is `IslandGrid.CanPlaceBuilding`.

A building will be refused (silently, when loaded through an island code) if any footprint tile is:

*   outside the grid / beyond the island's hard radius (see [Island Grid](./island/grid.md)),

*   on water (only `Dock`, `Fishing Ship`, `Lighthouse` are water buildings),

*   covered by a nature tile (tree, rock, ...),

*   overlapping another building.

## Unlocking and removing

*   `BuildData.Unlocked(BuildingType)` gates availability. The unlock technology (`UnlockBuildingTechnology`) stores a `BuildingTypes* ` at `+0x38` and an `int` id at `+0x40` and resolves it with `BuildingTypes.Get(types, id)`.

*   `BuildingTypeTech.unlocksTier` controls which tier a University unlocks when built.

*   Nature removal (`BuildData.CanRemoveNature`) has its own inline check that only allows `TREE`, `ROCK`, `SCORCHED_EARTH` and `LAVA`. It does **not** call `NatureTile.Cleanable`, so patching `Cleanable` does nothing.

*   The pickaxe (`CleanTileTool`) only works on the `BuildData`'s own island; it cannot target another player's island or remove `WATER_ROCK` (a ground tile).
