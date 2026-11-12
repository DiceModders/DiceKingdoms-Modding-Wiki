---
summary: IslandGrid layout, island radius limit, tile layers and the console cheats that expose island import/export
---
# Island Grid

`IslandGrid` holds the tile layers and buildings of one island and is wrapped by `Island`, which is owned by a `Player`.

## Class relationships

```text
Player --get_Island--> Island --grid--> IslandGrid
                          \--owner (NetworkedVar<Player>)
PlayerSummary --island--> Player's island
GameState --Island(Player)--> Island of a given player
```

## IslandGrid

| Member | Type | Offset | Notes |
|---|---|---|---|
| `size` | `int2` | `0xF0` | Grid dimensions |
| `generationParameters` | `IslandGeneration.Parameters` | `0x1E8` | `HardLimit.radius` is the first field (`+0`) |
| `snapshotIslandSize` | | `0x298` | not investigated |
| `arraySize` | | `0x2AC` | not investigated |

Methods of interest: `Awake`, `Generate`, `GenerateJob`, `CanPlaceBuilding`, `ExportIsland`, `ImportIsland` (private), `SaveIslandToClipboard`, `LoadIslandFromClipboard`. RVAs are in the [Internals Reference](../../reference/internals.md).

## Island radius (`HardLimit.radius`)

The playable land is limited by `Parameters.HardLimit.radius`, default **22**. Setting it to **24** in a patch on `IslandGrid.Awake` produced slightly larger *real* land through the normal generation path (found with a native hook before moving to BepInEx; the same field applies).

!!! warning "Do not resize the tile arrays or scale the grid"
    Multiplying `size` (an earlier "size scale" approach) caused frustum-error log spam, a disappearing mouse cursor and compounding corruption on import. Raising `HardLimit.radius` is the safe approach. Pushing land toward the edge of the allocated array (roughly radius 24-26) is the danger zone - test incrementally.

See also: [Island Generation](generation.md).

## Tile layers in memory

Each tile layer is an `IslandGrid.Layer<T>`: one flat array, row by row from the bottom (`index = y * width + x`).

| Offset in `Layer<T>` | Field |
|---|---|
| +0x10 | `T[]` contents (array elements start at +0x20 of the array, its length is at +0x18) |
| +0x18 | width |
| +0x1C | height |
| +0x20 | `modified` (byte; set it after writing so the game redraws) |

The layers on `IslandGrid` (this build): buildings +0x108, building remains +0x118, nature +0x128, ground +0x138, inner cliffs `List<InnerCliff>` +0x140, `size` +0xF0. Element sizes: `GroundTile` is a 4-byte enum; `Nullable<NatureTile>`, `Nullable<Building.Id>` and `Nullable<Unit.Id>` are 8 bytes (a `hasValue` byte, then the 4-byte value at +4).

Reading these directly is the fast and reliable way to scan a whole island from a mod. Through Il2CppInterop, methods that return a `Nullable<>` struct (`BuildingAt`, `RemainsAt`) came back `null`, and per-tile calls are slow on big grids. The offsets change with game updates: re-check them against the dump.

## Clipboard import / export

`SaveIslandToClipboard` and `LoadIslandFromClipboard` go through `GUIUtility.systemCopyBuffer`, which was unreliable when called outside `OnGUI`. A mod should call `ExportIsland` / `ImportIsland` directly and do its own clipboard handling, then use the [format](save-format.md) page to process the string.

## Developer cheats

The in-game dev console (including `saveisland` / `loadisland`) is gated by two methods on `Cheats`:

| Method | Effect when forced to return `true` |
|---|---|
| `Cheats.get_Available` | console/cheat commands become available |
| `Cheats.CheckAvailable` | same gate, second path |

Both need to return `true`. With BepInEx a Harmony postfix does this (untested example - check exact member names in the interop assemblies):

```c#
[HarmonyPatch(typeof(Cheats), nameof(Cheats.CheckAvailable))]
static class CheckAvailablePatch { static void Postfix(ref bool __result) => __result = true; }

[HarmonyPatch(typeof(Cheats), nameof(Cheats.Available), MethodType.Getter)]
static class AvailablePatch { static void Postfix(ref bool __result) => __result = true; }
```

The BepInEx guide covers [setting up the project](../../guides/modding/project-setup.md).

## Multiplayer notes (theory, not verified)

Island state is an `IslandSyncedBehaviour`. The working theory is that the host is the only authoritative sync source, so loading an island string onto another player's island on the host would replace it for everyone. This was never observed working live.

True host migration (reassigning the MLAPI listen server) is a different problem class and is not achievable by patching memory.
