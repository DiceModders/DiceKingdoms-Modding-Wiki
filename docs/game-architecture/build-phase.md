---
summary: The build phase - action types, planning and committing, the Preview flag (what you see instantly), every check, clean-tile yields, market, technologies, burning the Castle
---

# Build phase

During the build phase the client **plans** actions. Each one goes to the server (`[ServerRPC] PlanActionRPC(BuildData.Action)`), the server stores it and echoes it to the other clients
(`PlanActionClientRPC`). The checks and resource changes happen in `BuildData.DoAction`, either right away (on a [Preview](#preview-instant-or-at-sync) island) or at sync in
`BuildPhase.ApplyChanges`.

## Action types

`BuildData.Action.Type`:

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

1. Return false if the build phase is not `ACTIVE`: no actions are accepted after the timer ends.
2. If this island's `BuildData.Preview` is true, run `DoAction` now; if it fails, the action is dropped.
3. The owner sends `PlanActionRPC` to the server; the server sends `PlanActionClientRPC` to everybody except the owner. Each side calls `PlanAction` again.
4. Append the action to the island's action list (`BuildIslandState.Actions`).
5. On the server, `SEND_RESOURCES` also calls `ReceiveResources` on the target's `BuildData` right away.

## Committing

`BuildPhase.ApplyChanges` (`0x62C6C0`), at sync: for every island, every stored action whose island is not `Preview` goes through `DoAction`; **failed actions are skipped silently**.
Successful placements and technologies are counted for the game log (hammer icon followed by `NxBuilding`, culture icon followed by the technologies).

So the server checks every action, but for islands it does not preview it checks them only at sync.

## Preview: instant or at sync

`PhaseData.Preview` decides whether this machine applies an island's actions **the moment they arrive** (true) or **only at the end of the phase** (false). It has nothing to do with the
camera. `PhaseData.Enter` (`0xE60440`) sets it for every island, on every machine, at the start of every phase (and on rejoin):

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

**Guard Tower upgrade:** `ReplaceBuildings.OnLearn` (`0x621650`) replaces every Guard Tower on the island with a Guard Tower Up at the same place, **keeping its health number** (a damaged tower
is not healed). New Guard Towers are placed as the upgraded type from then on.

## Burning the Castle

Nothing stops it: `CanBurn` only checks that the building exists and is not even called by the UI; the burn button (`BuildInterface`) and the building selector have no type filter.
So a player can burn their own Castle in the build phase for 1 hammer. What happens then: [Win conditions](win-conditions.md#losing-a-castle).
