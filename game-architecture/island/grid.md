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
