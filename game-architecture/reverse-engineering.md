---
summary: Tools and method for reading the game code, plus RVAs and field offsets of version 1.0.4
---

# Reverse-engineering notes

!!! warning "Version"
    Addresses and offsets below are from game version 1.0.4. They change with every game update. Prefer names and fields from the interop assemblies
    when you can, and use offsets only for hooks you check against your build.

## Method

- **Il2CppDumper** produces `dump.cs` with all classes, fields and method RVAs. Its naming for methods is `Namespace.Class$$MethodName`.
- **Ghidra** with `GameAssembly.dll` loaded: the Ghidra address is `0x180000000 + RVA` (for example `180616890` is RVA `0x616890`).
- Standard il2cpp layouts: array length at `+0x18` and data at `+0x20`; string length at `+0x10` and UTF-16 characters at `+0x14`; `List<T>._size` at `+0x18`.

## Island

| Item | Value |
|---|---|
| `IslandGrid.size` (int2) | `0xF0` |
| `IslandGrid.generationParameters` | `0x1E8` (`HardLimit.radius` at `+0`, default 22) |
| `IslandGrid.Awake` | RVA `0x6130C0` |
| `IslandGrid.CanPlaceBuilding` | `0x6143A0` |
| `IslandGrid.ImportIsland` (private) | `0x616340` |
| `IslandGrid.ExportIsland` | `0x614900` |
| `IslandGrid.Generate` / `GenerateJob` | `0x614EB0` / `0x614D90` |
| `IslandGrid.MeteoriteDisaster` | `0x616890` |
| `IslandGrid.MeteoriteNumberCastleTiles` | `0x616DF0` |
| `Island.grid` / `owner` | `0x98` / `0x90` |

The radius of the island can be changed in the `Awake` hook; real land is then produced by normal generation. Resizing arrays does not work.
`SaveIslandToClipboard` (`0x61A690`) and `LoadIslandFromClipboard` (`0x616810`) go through `GUIUtility.systemCopyBuffer`, which is unreliable outside `OnGUI`;
use `ExportIsland` / `ImportIsland` and handle the clipboard yourself. The code format is in [Island code](island-code.md).

## Players and game state

| Item | Value |
|---|---|
| `Player.UserName` (string) | `0x110`, getter `0x473BB0` |
| `Player.get_Island` | `0x508770` |
| `Player.get_StartGameReady` | `0x571AB0` |
| `PlayerSummary.island` / `attackers` / `gameState` | `0xD0` / `0xE8` / `0xC8` |
| `GameState.Island(Player)` | `0x5F2F60` |

## Buildings

| Item | Value |
|---|---|
| `BuildData.CanBuildType` | `0x628760`: `IsBuildable`, then `Unlocked`, then `SteamDemo`, then `HasResource` |
| `BuildData.CanRemoveNature` | `0x6288B0`: its own inline check, allows only trees, rocks, scorched earth and lava. It does not call `NatureTile.Cleanable`, so a hook on `Cleanable` has no effect. |
| `BuildData.CleanTile` | `0x628AB0` |
| `BuildingType.id` | `+0x10` (this is the type id, not a category) |
| `BuildingType.localPositions` / `localPositionsInWater` | `0x18` / `0x20` |
| `BuildingType.tier` / max health | `+0x78` / `+0x80` |
| `BuildingType.get_DisplayName` | `0x5F0E10` |
| `BuildingTypes.get_Types` | `0x5717B0` (save-file order, alphabetical by name) |

The pickaxe tool (`CleanTileTool`) only works on the builder's own island and cannot remove water rocks, because those are ground tiles, not nature.

## Cheats

`Cheats.get_Available` (`0x52A840`) and `Cheats.CheckAvailable` (`0x528D40`) return true when hooked, which unlocks the developer console commands.

## Meteorites

See [Meteorites and castle protection](meteorites.md).

## Nature tile enum

`TREE = 0`, `ROCK = 1`, `SCORCHED_EARTH = 2`, `LAVA = 3`, `FLOWER = 4`, `FISH = 5`.
