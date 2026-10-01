---
summary: Building type ids, footprint shapes, rotation/mirror math and placement rules
---
# Buildings

## BuildingType

Each building kind is a `BuildingType` object held in a `BuildingTypes` collection (referenced by `RulesBuilding`). The **index in `BuildingTypes.Types` is the id used in save files and island codes**.

| Field | Type | Offset | Notes |
|---|---|---|---|
| `localPositions` | `int2[]` | `0x18` | Footprint, in building-local coordinates |
| `localPositionsInWater` | `int2[]` | `0x20` | Footprint variant used in water |
| category | `int` | `0x10` | meaning of values not mapped yet |
| tier | `int` | `0x78` | probably the tier; only partly verified |
| cost | struct | `0x58 - 0x74` | resource cost block |

!!! note "Offsets are build specific"
    Offsets and RVAs on this page come from one game build (see [Internals Reference](../reference/internals.md)). Field *names* are stable; numbers may move after a game update.

## Type ids

Ids `0-20` are alphabetical by display name; `21-23` were added later and sit at the end. There is no in-game build menu - buildings are panels at the bottom of the screen; picking one makes it follow the cursor until you left-click.

| Id | Name | Tiles | Id | Name | Tiles |
|---|---|---|---|---|---|
| 0 | Archery Range | 5 | 12 | Lumber Camp | 4 |
| 1 | Barracks | 6 | 13 | Market | 4 |
| 2 | Castle | 16 | 14 | Sorcerer Tower | 3 |
| 3 | Cathedral | 6 | 15 | Stone Quarry | 3 |
| 4 | Church | 3 | 16 | Storage | 6 |
| 5 | Cottage | 2 | 17 | University | 7 |
| 6 | Dock | 12 | 18 | Wall | 1 |
| 7 | Farm | 8 | 19 | Windmill | 4 |
| 8 | Guard Tower | 4 | 20 | Wonder | 15 |
| 9 | Guard Tower Up | 4 | 21 | Fishing Ship | 2 |
| 10 | Hospital | 5 | 22 | Lighthouse | 4 |
| 11 | House | 3 | 23 | Land Camp | 4 |

### Footprint data

Local offsets `(dx, dy)` per type, exactly as read from `localPositions`:

```text
0: Archery Range   [[0,0],[-1,0],[0,-1],[-1,-1],[-1,-2]]
1: Barracks        [[-1,1],[0,1],[0,0],[0,-1],[1,0],[1,-1]]
2: Castle          [[-1,-1],[-1,0],[-1,1],[-1,2],[0,-1],[0,0],[0,1],[0,2],[1,-1],[1,0],[1,1],[1,2],[2,-1],[2,0],[2,1],[2,2]]
3: Cathedral       [[1,0],[0,0],[-1,0],[-2,0],[0,1],[0,-1]]
4: Church          [[0,0],[-1,0],[-2,0]]
5: Cottage         [[0,0],[0,1]]
6: Dock            [[0,0],[1,0],[2,0],[2,1],[3,0],[3,1],[4,-1],[4,0],[4,1],[5,-1],[5,0],[5,1]]
7: Farm            [[0,1],[0,0],[0,-1],[0,-2],[1,1],[1,0],[1,-1],[1,-2]]
8: Guard Tower     [[0,0],[-1,0],[0,-1],[-1,-1]]
9: Guard Tower Up  [[0,0],[-1,0],[0,-1],[-1,-1]]
10: Hospital       [[0,0],[-1,0],[1,0],[0,-1],[0,1]]
11: House          [[0,0],[-1,0],[0,-1]]
12: Lumber Camp    [[0,0],[-1,0],[-1,1],[-2,1]]
13: Market         [[0,0],[1,0],[1,1],[0,1]]
14: Sorcerer Tower [[0,0],[1,0],[0,1]]
15: Stone Quarry   [[0,0],[0,1],[1,0]]
16: Storage        [[0,0],[1,0],[0,-1],[1,-1],[0,1],[1,1]]
17: University     [[0,1],[0,0],[0,-1],[1,1],[1,-1],[2,1],[2,-1]]
18: Wall           [[0,0]]
19: Windmill       [[0,0],[-1,0],[-1,1],[0,1]]
20: Wonder         [[3,-1],[3,0],[3,1],[2,-1],[2,0],[2,1],[1,-1],[1,0],[1,1],[0,-1],[0,0],[0,1],[-1,-1],[-1,0],[-1,1]]
21: Fishing Ship   [[0,0],[0,1]]
22: Lighthouse     [[0,0],[-1,0],[0,-1],[-1,-1]]
23: Land Camp      [[0,0],[1,1],[0,1],[-1,1]]
```

## Footprint and rotation

`GridTransform.TransformVector` rotates by `orient` (0-3) in 90 degree steps, then `mirror` flips X:

| `orient` | `(cos, sin)` |
|---|---|
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
    if (mirror) nx = -nx;               // scaleX = -1
    return (nx, ny);
}

// world tile = buildingPos + Transform(local offset)
IEnumerable<(int x, int y)> Footprint(BuildingType t, int bx, int by, int orient, bool mirror)
    => t.localPositions.Select(o => { var (x, y) = Transform(o.x, o.y, orient, mirror); return (bx + x, by + y); });
```

## Placement rules

`BuildData.CanBuildType` checks, in order: `IsBuildable` -> `Unlocked` -> `SteamDemo` -> `HasResource`. The per-tile check is `IslandGrid.CanPlaceBuilding`.

A building will be refused (silently, when loaded through an island code) if any footprint tile is:

- outside the grid / beyond the island's hard radius (see [Island Grid](island-grid.md)),
- on water (only `Dock`, `Fishing Ship`, `Lighthouse` are water buildings),
- covered by a nature tile (tree, rock, ...),
- overlapping another building.

!!! bug "Open issue: Castle placed from an edited code does not appear"
    A Castle (id 2, 16 tiles) added through an edited island code never showed up after `loadisland`, with no error, while castles from the game's own saves loaded fine (even duplicates). Other building types placed the same way were fine. Cause is not confirmed; leading suspect is `CanPlaceBuilding` refusing the 4x4 footprint because of nature or terrain under it. If you find the answer, please update this page.

## Unlocking and removing

- `BuildData.Unlocked(BuildingType)` gates availability. The unlock technology (`UnlockBuildingTechnology`) stores a `BuildingTypes*` at `+0x38` and an `int` id at `+0x40` and resolves it with `BuildingTypes.Get(types, id)`.
- Nature removal (`BuildData.CanRemoveNature`) has its own inline check that only allows `TREE`, `ROCK`, `SCORCHED_EARTH` and `LAVA`. It does **not** call `NatureTile.Cleanable`, so patching `Cleanable` does nothing.
- The pickaxe (`CleanTileTool`) only works on the `BuildData`'s own island; it cannot target another player's island or remove `WATER_ROCK` (a ground tile).
