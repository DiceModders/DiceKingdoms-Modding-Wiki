---
summary: The harvest phase - which buildings roll dice, die faces, re-rolls, passive producers, Rituals, storage caps and food upkeep
---

# Harvest and dice

!!! info "Code blocks"
    Declarations from the IL2CPP dump of game version 1.0.4, trimmed to what each section uses. `// 0x..` is the field offset, `// RVA 0x..` the method address in `GameAssembly.dll` (in Ghidra `0x180000000 + RVA`). Names stay stable between updates, offsets and RVAs do not. Private members are public in the BepInEx interop assemblies.

## Who rolls dice

`HarvestData.Roll` (`0x649BC0`): only buildings whose type is **`BuildingTypeHousing`** (or its subclass `BuildingTypeCastle`) roll, and only while `Building.Active`
(a hungry or sick building is inactive).

Dice come from housing buildings; a building hit by a disaster does not roll:

<!-- code: BuildingTypeHousing | *fields -->
```csharp
namespace DiceKingdoms.Game;

public class BuildingTypeHousing : BuildingType
{
    public ResourceAmount[] possibleYields;  // 0xE8
    public Color diceColor;                  // 0xF0
    public Unit.Type unitType;               // 0x100
    public Dice dicePrefab;                  // 0x108
}
```
<!-- /code -->

<!-- code: Building | affectedByDisaster, Active -->
```csharp
namespace DiceKingdoms.Game;

public struct Building : IEquatable<Building>
{
    public Disaster affectedByDisaster;  // 0x18

    public bool Active { get; }
}
```
<!-- /code -->

| `BuildingTypeHousing` field | Offset | Meaning |
|---|---|---|
| `possibleYields` | `0xE8` | `ResourceAmount[]`, the faces of the die |
| `diceColor` | `0xF0` | `Color` |
| `unitType` | `0x100` | the inhabitant that defends this building in combat |
| `dicePrefab` | `0x108` | `Dice` visual |

The result for one building is `RandomExtensions.NextSample(possibleYields)`: a uniform pick of one face, using the **per-island random generator** (`IslandGrid + 0xFC`).
A missing, inactive or non-housing building gives `ResourceAmount.ZERO`.

"Disaster" is one of the eight resources, so dice faces can give disaster points (see [Disasters](disasters.md)).

## Die faces

| Building | Faces |
|---|---|
| Castle | 2 food · 1 food, 1 hammer · 1 gold · 2 military · 1 food, 1 stone · 1 gold, 1 military, 1 disaster |
| Cottage | 4 food · 3 wood · 1 wood, 1 stone · 1 food, 2 hammer · 1 food, 1 stone · 1 wood, 1 hammer, 1 disaster |
| Church | 2 food · 1 wood, 1 stone · 1 food, 2 culture · 1 wood, 2 culture · 1 stone, 2 culture · 3 culture, 1 disaster |
| House | 3 stone · 3 hammer · 1 stone, 1 gold · 1 wood, 1 gold · 1 gold, 1 culture · 1 stone, 1 hammer, 1 disaster |
| Barracks | 1 wood, 2 military · 2 military · 3 military · 4 military · 1 gold, 1 military · 2 military, 1 disaster |
| Sorcerer Tower | 1 wood · 1 stone · 4 gold · 5 culture · 3 disaster · 4 disaster |

Every die except the Sorcerer Tower's has exactly one face with 1 disaster point; the Sorcerer Tower has two faces with 3 and 4.

## Re-rolls

`HarvestData.Reset` (`0x649520`):

Rolls and re-rolls are handled by `HarvestData`:

<!-- code: HarvestData | BASE_RE_ROLLS, ReRollsRemaining, TotalReRolls, Reset, ReRoll, ReRollRPC, Roll, RollResultsRPC, FirstRoll -->
```csharp
namespace DiceKingdoms.Game;

public class HarvestData : PhaseData<HarvestData, HarvestIslandState>
{
    private const int BASE_RE_ROLLS = 2;

    public int ReRollsRemaining { get; set; }
    public int TotalReRolls { get; set; }

    private void ReRollRPC(Building.Id[] buildings);                                            // RVA 0x6491D0
    public void ReRoll(Building.Id[] buildingIds);                                              // RVA 0x649380
    private IEnumerator FirstRoll();                                                            // RVA 0x648FB0
    private void Roll(Building.Id[] buildingIds);                                               // RVA 0x649BC0
    private void RollResultsRPC(ValueTuple<Building.Id, ResourceAmount>[] results, int reRollsRemaining);  // RVA 0x6498E0
    protected override void Reset();                                                            // RVA 0x649520
}
```
<!-- /code -->

**Patch:** postfix `HarvestData.Reset` and set `TotalReRolls` / `ReRollsRemaining` for more re-rolls; change the faces in `HarvestData.Roll`.

- `BASE_RE_ROLLS = 2`; `TotalReRolls = 2 + ` the sum of `AddRollTechnology.value` over learnt technologies (technology `ADDITIONAL_ROLL`, +1).
- The server makes the first roll. The player picks dice and calls `ReRoll(Building.Id[])` → `ReRollRPC`; the server rolls them again and broadcasts `RollResultsRPC(results, reRollsRemaining)`.
- Dice are animated client-side; the result is decided on the server before the animation (see [Rendering](rendering.md#dice)).

## Total yield

`HarvestData.ComputeTotalYields` (`0x648530`) builds `TotalYields` from scratch:

The total is computed and paid out in `HarvestData`:

<!-- code: HarvestData | TotalYields, Yield, ComputeTotalYields, SetAIYields, AddResources -->
```csharp
namespace DiceKingdoms.Game;

public class HarvestData : PhaseData<HarvestData, HarvestIslandState>
{
    public ResourceAmount TotalYields { get; set; }

    public ResourceAmount Yield(Building.Id buildingId);  // RVA 0x649F60
    private void ComputeTotalYields();                    // RVA 0x648530
    public void SetAIYields(ResourceAmount yields);       // RVA 0x649EE0
    public void AddResources();                           // RVA 0x648470
}
```
<!-- /code -->

**Patch:** prefix `HarvestData.AddResources` and change `__instance.TotalYields` to change what a player receives (the dice on screen keep their faces).

1. every dice result of this harvest;
2. plus `BuildingTypeProducer.Yield(building, island)` of every **active** producer building (passive income, below);
3. **Rituals:** for every other island whose owner is not on my team (in free-for-all: every other player) and who has learnt Rituals, each active Sorcerer Tower there adds
   `skullPerSorcererTower` (1) **disaster** point to my yield. In team mode Rituals never hurts teammates.

`HarvestData.AddResources` (`0x648470`) adds `TotalYields` to the island's `Statistics` with `AddResource`, **capped at the storage capacity**, then resets `TotalYields`.

## Passive producers

Subclasses of `BuildingTypeProducer` implement `Yield(Island, Building)`:

Passive income comes from the producer building types:

<!-- code: BuildingTypeProducer | *methods -->
```csharp
namespace DiceKingdoms.Game;

public abstract class BuildingTypeProducer : BuildingType
{
    public abstract ResourceAmount Yield(Island island, Building building);
}
```
<!-- /code -->

<!-- code: BuildingTypeProducerFixed | *fields -->
```csharp
namespace DiceKingdoms.Game;

public class BuildingTypeProducerFixed : BuildingTypeProducer
{
    public ResourceAmount yield;  // 0xE8
}
```
<!-- /code -->

<!-- code: BuildingTypeProducerNature | *fields -->
```csharp
namespace DiceKingdoms.Game;

public class BuildingTypeProducerNature : BuildingTypeProducer
{
    private Resource yieldResource;  // 0xE8
    private NatureTile natureTile;   // 0xEC
    private int tilesNeeded;         // 0xF0
}
```
<!-- /code -->

<!-- code: BuildingTypeProducerAmplifier | *fields -->
```csharp
namespace DiceKingdoms.Game;

public class BuildingTypeProducerAmplifier : BuildingTypeProducer
{
    private BuildingTypes buildingTypes;                 // 0xE8
    private Resource yieldResource;                      // 0xF0
    private BuildingTypeName amplifiedBuildingTypeName;  // 0xF4
    private int amplifyFactor;                           // 0xF8
    private int amplifyFactorProximityBonus;             // 0xFC
}
```
<!-- /code -->

**Patch:** postfix `Yield` of the producer class to change a building's passive income.

| Class | Formula | Buildings |
|---|---|---|
| `BuildingTypeProducerFixed` | fixed `yield` (`0xE8`) | Farm: 1 food |
| `BuildingTypeProducerNature` (`0x60B890`) | (neighbouring nature tiles / `tilesNeeded`) × (2 if a Lighthouse covers it) | Lumber Camp: wood per TREE; Stone Quarry: stone per ROCK; Fishing Ship: food per FISH |
| `BuildingTypeProducerAmplifier` (`0x60B300`) | count on island × `amplifyFactor` + count next to it × `amplifyFactorProximityBonus` | Cathedral: 2 culture per Church on the island + 1 per adjacent Church; Windmill: 3 food per adjacent Farm |

`BuildingTypeApplyModifierInRange` applies `BuildingModifier.DOUBLE_FISH` to Fishing Ships within range 20 of a Lighthouse (it does not stack).
`BuildingTypeLandExtension` (Land Camp) implements `IBuildingTypeUpdateOnHarvest`: each harvest it turns up to 24 water tiles within range 6 into land.

## Storage capacity

`Island.RecomputeStorageCapacities` (`0x61F270`): the capacity is the **sum of `BuildingType.storageCapacity` over all buildings** on the island. Nothing else adds to it.

Capacities are recomputed on the island and stored in its `Statistics`:

<!-- code: Island | statistics, Recompute, RecomputeStorageCapacities -->
```csharp
namespace DiceKingdoms.Game;

public class Island : NetworkedBehaviour
{
    public Statistics statistics;               // 0xA0

    public void Recompute();                    // RVA 0x61F590
    private void RecomputeStorageCapacities();  // RVA 0x61F270
}
```
<!-- /code -->

<!-- code: Statistics | Storage, SetCapacity -->
```csharp
namespace DiceKingdoms.Game;

public class Statistics : IslandSyncedBehaviour<Statistics.SyncData>
{
    public ResourceStorage Storage { get; set; }

    public void SetCapacity(ResourceAmount capacity);  // RVA 0x624010
}
```
<!-- /code -->

**Patch:** postfix `Island.RecomputeStorageCapacities` (private), change `statistics.Storage.capacity` and call `SetCapacity`.

| Building | Storage |
|---|---|
| Castle | 15 food, 10 wood, 10 stone, 999 gold, 999 hammers, 999 culture, 8 military, 999 disaster |
| Storage | 10 food, 20 wood, 20 stone |
| Barracks, Archery Range | 8 military each |

- Gold, hammers, culture and disaster are only raised by more Castles; Storage buildings only raise food, wood and stone. The Storage research only unlocks the building.
- It is recomputed after every phase, so a building placed in the build phase raises the capacity at the end of that phase (confirmed in game).
- `Statistics.SetCapacity` → `ResourceStorage.WithCapacity` clamps every stored amount to the new capacity. That is why losing buildings throws away resources above the new capacity,
  and why burning the last Castle empties everything.

### Encrypted storage

The resources are kept in `Utility.AntiCheat.EncryptedStorage<ResourceStorage>` inside `Statistics`: the 64 bytes of `ResourceStorage` XOR a random per-write key XOR a per-launch static key.
A memory scanner will not find plain values. **Mods do not need to decrypt anything:** read `Statistics.get_Storage` (plain `ResourceStorage`) and change resources with
`Statistics.AddResource` / `RemoveResource` / `SetCapacity`.

Read and change resources through `Statistics`, never the raw memory:

<!-- code: Statistics | Storage, AddResource, RemoveResource, HasResource, ShipCapacity, TradeFactor -->
```csharp
namespace DiceKingdoms.Game;

public class Statistics : IslandSyncedBehaviour<Statistics.SyncData>
{
    public ResourceStorage Storage { get; set; }
    public int ShipCapacity { get; }
    public int TradeFactor { get; }

    public void AddResource(ResourceAmount resourceAmount);     // RVA 0x622C90
    public bool HasResource(ResourceAmount resourceAmount);     // RVA 0x623650
    public void RemoveResource(ResourceAmount resourceAmount);  // RVA 0x623B20
}
```
<!-- /code -->

## Food upkeep

Each turn the [Disasters](disasters.md#effects) phase charges **1 food per active housing building**. Missing food makes random buildings hungry (inactive, so no dice next harvest).
