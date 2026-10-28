---
summary: The build phase - action types, planning and committing, the Preview flag (what you see instantly), every check, clean-tile yields, market, technologies, burning the Castle
---

# Build phase

!!! info "Code blocks"
    Declarations from the IL2CPP dump of game version 1.0.4, trimmed to what each section uses. `// 0x..` is the field offset, `// RVA 0x..` the method address in `GameAssembly.dll` (in Ghidra `0x180000000 + RVA`). Names stay stable between updates, offsets and RVAs do not. Private members are public in the BepInEx interop assemblies.

During the build phase the client **plans** actions. Each one goes to the server (`[ServerRPC] PlanActionRPC(BuildData.Action)`), the server stores it and echoes it to the other clients
(`PlanActionClientRPC`). The checks and resource changes happen in `BuildData.DoAction`, either right away (on a [Preview](#preview-instant-or-at-sync) island) or at sync in
`BuildPhase.ApplyChanges`.

## Action types

`BuildData.Action.Type`:

<!-- code: BuildData.Action.Type |  -->
```csharp
public enum BuildData.Action.Type
{
    REPAIR_BUILDING = 0,
    BURN_BUILDING = 1,
    CLEAN_TILE = 2,
    PLACE_BUILDING = 3,
    PLACE_BUILDING_BULK = 4,
    BUY_RESOURCE = 5,
    LEARN_TECHNOLOGY = 6,
    SEND_RESOURCES = 7,
    RECEIVE_RESOURCES = 8,
}
```
<!-- /code -->

<!-- code: BuildData.Action | *fields -->
```csharp
public struct BuildData.Action
{
    public BuildData.Action.Type type;     // 0x0
    public int2 position;                  // 0x4
    public Building.Id buildingId;         // 0xC
    public Building building;              // 0x10
    public Building[] buildings;           // 0x38
    public Resource resource;              // 0x40
    public Technology technology;          // 0x48
    public ResourceAmount resourceAmount;  // 0x50
    public Player player;                  // 0x70
}
```
<!-- /code -->

<!-- code: BuildData | RepairBuilding, BurnBuilding, CleanTile, PlaceBuilding, PlaceBuildingBulk, BuyResource, LearnTechnology, SendResources -->
```csharp
// DiceKingdoms.Game
public class BuildData : PhaseData<BuildData, BuildIslandState>
{
    public bool CleanTile(int2 position);                                          // RVA 0x628AB0
    public bool RepairBuilding(Building.Id buildingId);                            // RVA 0x62B1E0
    public bool BurnBuilding(Building.Id buildingId);                              // RVA 0x6282C0
    public bool PlaceBuilding(Building building);                                  // RVA 0x62A920
    public bool PlaceBuildingBulk(Building[] buildings);                           // RVA 0x62A860
    public bool BuyResource(Building.Id marketId, Resource resource);              // RVA 0x628340
    public bool LearnTechnology(Building.Id universityId, Technology technology);  // RVA 0x62A1A0
    public bool SendResources(Player target, ResourceAmount resourceAmount);       // RVA 0x62B300
}
```
<!-- /code -->

**Patch:** prefix one of these public methods to block or change one action type for the local player.

| # | Type | Public method | Notes |
|---|---|---|---|
| 0 | `REPAIR_BUILDING` | `RepairBuilding(id)` `0x62B1E0` | costs `toolCost` (1 hammer), restores `repairToolAmount` (60) HP, up to the type's maximum |
| 1 | `BURN_BUILDING` | `BurnBuilding(id)` `0x6282C0` | costs `toolCost` |
| 2 | `CLEAN_TILE` | `CleanTile(pos)` `0x628AB0` | costs `toolCost`; gives resources back (below) |
| 3 | `PLACE_BUILDING` | `PlaceBuilding(b)` `0x62A920` | |
| 4 | `PLACE_BUILDING_BULK` | `PlaceBuildingBulk(b[])` `0x62A860` | walls (bulk size 10) |
| 5 | `BUY_RESOURCE` | `BuyResource(marketId, res)` `0x628340` | needs a Market |
| 6 | `LEARN_TECHNOLOGY` | `LearnTechnology(universityId, tech)` `0x62A1A0` | needs a University |
| 7 | `SEND_RESOURCES` | `SendResources(target, amount)` `0x62B300` | to another player; arrives 1:1 |
| 8 | `RECEIVE_RESOURCES` | `ReceiveResources(src, amount)` | server side of 7 |

`Action` struct: `type` `0x0`, `position` (int2) `0x4`, `buildingId` `0xC`, `building` `0x10`, `buildings[]` `0x38`, `resource` `0x40`, `technology` `0x48`, `resourceAmount` `0x50`, `player` `0x70`.

## Planning

The public methods only fill an `Action` and call `PlanAction` (`0x62AB20`):

<!-- code: BuildData | Actions, PlanAction, PlanActionRPC, PlanActionClientRPC, ReceiveResources -->
```csharp
// DiceKingdoms.Game
public class BuildData : PhaseData<BuildData, BuildIslandState>
{
    public List<BuildData.Action> Actions { get; }

    private void PlanActionRPC(BuildData.Action action);                  // RVA 0x62AA90
    private void PlanActionClientRPC(BuildData.Action action);            // RVA 0x62AA00
    private bool PlanAction(BuildData.Action action);                     // RVA 0x62AB20
    private void ReceiveResources(Player source, ResourceAmount amount);  // RVA 0x62AFD0
}
```
<!-- /code -->

**Patch:** prefix `BuildData.PlanAction` (private) to see or block every planned action in one place.

1. Return false if the build phase is not `ACTIVE`: no actions are accepted after the timer ends.
2. If this island's `BuildData.Preview` is true, run `DoAction` now; if it fails, the action is dropped.
3. The owner sends `PlanActionRPC` to the server; the server sends `PlanActionClientRPC` to everybody except the owner. Each side calls `PlanAction` again.
4. Append the action to the island's action list (`BuildIslandState.Actions`).
5. On the server, `SEND_RESOURCES` also calls `ReceiveResources` on the target's `BuildData` right away.

## Committing

`BuildPhase.ApplyChanges` (`0x62C6C0`), at sync: for every island, every stored action whose island is not `Preview` goes through `DoAction`; **failed actions are skipped silently**.
Successful placements and technologies are counted for the game log (hammer icon followed by `NxBuilding`, culture icon followed by the technologies).

<!-- code: BuildPhase | *fields, ApplyChanges -->
```csharp
// DiceKingdoms.Game
public class BuildPhase : TurnPhase<BuildData, BuildIslandState>
{
    private readonly Dictionary<BuildingType, int> buildingTypeCount;          // 0xE8
    private readonly List<ValueTuple<Island, Technology>> learntTechnologies;  // 0xF0

    protected override void ApplyChanges(Action done);                         // RVA 0x62C6C0
}
```
<!-- /code -->

So the server checks every action, but for islands it does not preview it checks them only at sync.

## Preview: instant or at sync

`PhaseData.Preview` decides whether this machine applies an island's actions **the moment they arrive** (true) or **only at the end of the phase** (false). It has nothing to do with the
camera. `PhaseData.Enter` (`0xE60440`) sets it for every island, on every machine, at the start of every phase (and on rejoin):

<!-- code: PhaseData | Preview, Island, Enter -->
```csharp
// DiceKingdoms.Game
public abstract class PhaseData<TData, TState> : NetworkedBehaviour, IPhaseData
{
    public Island Island { get; set; }
    public bool Preview { get; set; }

    public void Enter(bool restore);  // RVA 0xE60440 (shared generic body)
}
```
<!-- /code -->

1. The local machine controls the island's owner (`Player.IsInitialOwner`) → **true** (your own island).
2. Else, on the server, if the owner is an **AI** → **true** (the host runs the AIs).
3. Else, in **team mode**: true for teammates (`Team.Same(owner, Player.Local)`), false for enemies.
4. Else (free-for-all): `Player.Local.IsDefeated`: false while you are alive, **true for every island once you are defeated** (spectator).

| Island | Its changes appear on your screen |
|---|---|
| Your own | instantly |
| AI islands, when you host | instantly |
| Teammates' (team mode) | instantly |
| Other human players' | when the build phase ends |
| Every island, after you are defeated | instantly |

In a free-for-all with other humans you do not see what they build until the phase ends, and they see your buildings the same way. On the host, a remote human's island is not previewed
either; the host checks and applies it at sync like everybody else. (Confirmed in game for the host with AI islands: all previewed.)

## Checks and effects

`FixAction` (inlined at the start of `DoAction` `0x628BA0`): a single placed building's type goes through `UpgradedBuildingType` (Guard Tower → Guard Tower Up once the upgrade is
learnt), and every placed building gets `SetNewlyBuilt(turnCounter)`.

<!-- code: BuildData | toolCost, repairToolAmount, marketOffers, FixAction, DoAction, JustPlacedBuilding, Offer, AvailableMarket, AvailableUniversity, UnavailableUniversity -->
```csharp
// DiceKingdoms.Game
public class BuildData : PhaseData<BuildData, BuildIslandState>
{
    public ResourceAmount toolCost;                                   // 0xB8
    public int repairToolAmount;                                      // 0xD8
    public BuildData.MarketOffer[] marketOffers;                      // 0xE0

    public Nullable<Building.Id> AvailableUniversity { get; set; }
    public Nullable<Building.Id> UnavailableUniversity { get; set; }
    public Nullable<Building.Id> AvailableMarket { get; set; }

    private void FixAction(ref BuildData.Action action);              // RVA 0x629F70
    public bool DoAction(ref BuildData.Action action);                // RVA 0x628BA0
    public bool JustPlacedBuilding(Building.Id buildingId);           // RVA 0x62A090
    public Nullable<BuildData.MarketOffer> Offer(Resource resource);  // RVA 0x62A470
}
```
<!-- /code -->

**Patch:** `DoAction` is where every action is checked and applied (on preview islands at once, otherwise at sync). Prices: `toolCost`, `repairToolAmount`, `marketOffers`.

| # | Checks (in order) | Effect |
|---|---|---|
| 0 Repair | can pay `toolCost`; building exists; `health < type.health` | pay; `IslandGrid.RepairBuilding(id, repairToolAmount)` = min(health + 60, max) |
| 1 Burn | can pay `toolCost`; building exists | pay; `IslandGrid.RemoveBuilding(id)` |
| 2 Clean | can pay `toolCost`; `CanRemoveNature(pos)` or `CanRemoveRemains(pos)` | pay; clean (below) |
| 3 Place | `CanBuildType(type)`; `IslandGrid.CanPlaceBuilding` | add the building; pay `type.cost` |
| 4 Bulk | all the same type; `CanBuildType`; can pay `cost × n`; every building passes `CanPlaceBuilding` | add all; pay `cost × n`, **n = ceil(count / bulkSize)** |
| 5 Buy | an offer for the resource exists; can pay its cost; storage of that resource **not full**; the Market was **not placed this turn**; building active and a Market | pay `offer.cost`; add `offer.amount × TradeFactor` (capped by storage) |
| 6 Learn | `Statistics.IsTechnologyLearnable`; can pay; the University was not placed this turn; building active and a University | pay `tech.cost`; `Statistics.LearnTechnology` |
| 7 Send | can pay `amount` | remove `amount` |
| 8 Receive | – | add `amount` |

- A **Market or University placed this turn** cannot be used until the next build phase (`JustPlacedBuilding`: a single `PLACE_BUILDING` with that id in this turn's list).
- **Market:** `TradeFactor` (Trade Efficiency) multiplies the **amount received**, not the price, and applies in the same phase it is learnt. Offers: 1 gold → 2 food, 1 wood, 1 stone,
  1 military or 1 hammer; ×2 with Trade Efficiency (confirmed in game). The trade happens inside `DoAction`; `BuildData.BuyResource` is not on this path, so a hook there does not see trades.
- **Bulk:** the buildings inside one bulk action are not checked against each other. The normal client never sends overlapping ones (`BuildingBulkPlacer` skips taken positions).
  `IslandGrid.AddBuilding` re-checks only inside `Assert.IsTrue`, and Unity's `Assert` only logs by default.
- A building's id is its anchor position: `Building.id = (y << 16) | x`.

### Check functions

<!-- code: BuildData | CanBuildType, Unlocked, DemoLocked, UpgradedBuildingType, CanAfford, CanAffordBulk, CanAffordMarket, CanAffordTechnology, CanAffordTool, BulkCount, CanRepair, CanBurn, CanRemoveNature, CanRemoveRemains -->
```csharp
// DiceKingdoms.Game
public class BuildData : PhaseData<BuildData, BuildIslandState>
{
    public bool CanAffordTool { get; }

    private bool CanBuildType(BuildingType buildingType);                 // RVA 0x628760
    public static int BulkCount(BuildingType buildingType, int count);    // RVA 0x628290
    private bool CanRepair(Building.Id buildingId);                       // RVA 0x6289E0
    private bool CanBurn(Building.Id buildingId);                         // RVA 0x628830
    public bool CanRemoveNature(int2 position);                           // RVA 0x6288B0
    public bool CanRemoveRemains(int2 position);                          // RVA 0x628960
    public bool CanAfford(BuildingType buildingType);                     // RVA 0x6286E0
    public bool CanAffordBulk(BuildingType buildingType, int count);      // RVA 0x6283D0
    public bool CanAffordMarket(BuildData.MarketOffer offer);             // RVA 0x6284E0
    public bool CanAffordTechnology(Technology technology);               // RVA 0x628660
    public bool Unlocked(BuildingType buildingType);                      // RVA 0x62BAA0
    public bool DemoLocked(BuildingType buildingType);                    // RVA 0x628B60
    public BuildingType UpgradedBuildingType(BuildingType buildingType);  // RVA 0x62BE70
}
```
<!-- /code -->

<!-- code: IslandGrid | CanPlaceBuilding, AddBuilding, RemoveBuilding, RepairBuilding, BuildingAt -->
```csharp
// DiceKingdoms.Game
public class IslandGrid : IslandSyncedBehaviour<IslandGrid.SyncData>
{
    public Nullable<Building.Id> BuildingAt(int2 position);                // RVA 0x613A20
    public void RepairBuilding(Building.Id buildingId, int repairAmount);  // RVA 0x619E70
    public bool CanPlaceBuilding(Building building);                       // RVA 0x6143A0
    public void AddBuilding(Building building);                            // RVA 0x611500
    public void RemoveBuilding(Building.Id id, float delay = 0);           // RVA 0x619360
}
```
<!-- /code -->

**Patch:** postfix `IslandGrid.CanPlaceBuilding` for tile rules, `BuildData.Unlocked` / `CanBuildType` for which types may be built.

| Function | Rule |
|---|---|
| `BuildingType.IsBuildable` `0x5F0F10` | `cost != ZERO`. The Castle has no cost, so it can only be placed in the start phase. |
| `BuildData.CanBuildType` `0x628760` | `IsBuildable` and `Unlocked(type)` and the demo rule (Steam demo: only tier 1 or Storage) and can pay `type.cost`. |
| `BuildData.Unlocked` `0x62BAA0` | Locked if an `UnlockBuildingTechnology` for this type exists and is not learnt. An upgraded type (Guard Tower Up) only through its upgrade technology. The Wonder is locked when the lobby option **Wonder Victory** is off. |
| `BuildData.UpgradedBuildingType` `0x62BE70` | Guard Tower → Guard Tower Up once `upgradeGuardTower` is learnt. |
| `CanAffordBulk` / `BulkCount` | `BulkCount(type, n) = n < 1 ? 0 : (n − 1) / bulkSize + 1`. |
| `CanRepair` `0x6289E0` | building exists and `health < type.health`. |
| `CanBurn` `0x628830` | building exists, nothing else. |
| `CanRemoveNature` `0x6288B0` | nature is TREE, ROCK, SCORCHED_EARTH or LAVA (its own check; `NatureTile.Cleanable` is not called). |
| `CanRemoveRemains` `0x628960` | there are remains on the tile. |
| `IslandGrid.CanPlaceBuilding` `0x6143A0` | For every tile: inside the grid; no building; ground **WATER** for the type's water tiles, otherwise **GRASS or PIER**; no nature except FLOWER (removed when placed); no remains. |

- Sand, cliffs and meteorite ground are never buildable.
- Water tiles (`localPositionsInWater`): Dock 11 of 12 tiles (the pier), Fishing Ship both tiles, Lighthouse all 4 tiles (2×2 on water).
- **Fish block placement**: fish is a nature tile, so a Dock, Fishing Ship or Lighthouse cannot cover it. Scorched earth and lava block building until cleaned.

## Cleaning a tile

`DoCleanTile`:

<!-- code: BuildData | NatureYields, onCleanTile -->
```csharp
// DiceKingdoms.Game
public class BuildData : PhaseData<BuildData, BuildIslandState>
{
    public Action<int2, Nullable<NatureTile>, ResourceAmount> onCleanTile;              // 0xF8

    public static CleanTileTool.YieldAmount NatureYields(Nullable<NatureTile> nature);  // RVA 0x62A270
    private CleanTileTool.YieldAmount NatureYields(int2 position);                      // RVA 0x62A3D0
}
```
<!-- /code -->

<!-- code: IslandGrid | RemoveNature, RemoveBuildingRemains -->
```csharp
// DiceKingdoms.Game
public class IslandGrid : IslandSyncedBehaviour<IslandGrid.SyncData>
{
    public void RemoveNature(int2 position);                             // RVA 0x619DB0
    public void RemoveBuildingRemains(Building.Id id, float delay = 0);  // RVA 0x618F40
}
```
<!-- /code -->

**Patch:** postfix `BuildData.NatureYields` to change clean rewards; `onCleanTile` fires after every clean.

1. Remains on the tile: reward = the remains' `remainsReward`; the remains are removed.
2. Nature (`NatureYields` `0x62A270`), ×2 with **Double Clean Tile**:

    | Nature | Yield |
    |---|---|
    | Tree | 1 wood |
    | Rock | 1 stone |
    | Scorched earth | 1 disaster point |
    | Lava | 1 random: wood, stone or disaster point, **fixed per tile** (hash of the position, seed `y*128 + x + 62`) |

    Flowers and fish are never cleaned. `IslandGrid.RemoveNature` also turns **meteorite ground back into grass**, so one clean makes a lava tile buildable again.
3. The reward is added (capped by storage) and `onCleanTile(position, nature, amount)` fires.

## Technologies

Classes: `UnlockBuildingTechnology`, `AddRollTechnology` (+1 re-roll), `DoubleCleanTileTechnology`, `ReplaceBuildings` (Guard Tower upgrade), `ReplaceSoldiers` (Templars replace Soldiers,
Warlocks replace Archers), `RitualsTechnology`, `ShipCapacityTechnology` (`Statistics.ShipCapacity`), `TradeEfficiencyTechnology` (`Statistics.TradeFactor`). Costs: [Game data](game-data.md#technologies);
trees: [Civilizations in the game code](civilizations.md#tech-trees).

<!-- code: ReplaceBuildings | *fields, *methods -->
```csharp
// DiceKingdoms.Game
public class ReplaceBuildings : Technology
{
    private BuildingTypes buildingTypes;                                // 0x38
    private BuildingTypeName baseBuildingType;                          // 0x40
    private BuildingTypeName upgradedBuildingType;                      // 0x44

    public override Sprite Icon(PlayerSkins skins, PlayerColor color);  // RVA 0x6215D0
    public override void OnLearn(Island island);                        // RVA 0x621650
    private bool <OnLearn>b__8_0(Building b);                           // RVA 0x621900
}
```
<!-- /code -->

<!-- code: Statistics | LearnTechnology, IsTechnologyLearnable -->
```csharp
// DiceKingdoms.Game
public class Statistics : IslandSyncedBehaviour<Statistics.SyncData>
{
    public void LearnTechnology(Technology technology);        // RVA 0x623A90
    public bool IsTechnologyLearnable(Technology technology);  // RVA 0x6237C0
}
```
<!-- /code -->

**Guard Tower upgrade:** `ReplaceBuildings.OnLearn` (`0x621650`) replaces every Guard Tower on the island with a Guard Tower Up at the same place, **keeping its health number** (a damaged tower
is not healed). New Guard Towers are placed as the upgraded type from then on.

## Burning the Castle

Nothing stops it: `CanBurn` only checks that the building exists and is not even called by the UI; the burn button (`BuildInterface`) and the building selector have no type filter.
So a player can burn their own Castle in the build phase for 1 hammer. What happens then: [Win conditions](win-conditions.md#losing-a-castle).
