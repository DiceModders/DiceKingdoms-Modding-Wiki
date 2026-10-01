---
summary: Confirmed classes, methods, RVAs and field offsets from reverse engineering (single game build)
---
# Internals Reference

Classes, methods and field offsets found by reverse engineering the game. In BepInEx code, use the **names**; the numbers are for locating the same member in a dump or decompiler.

## How these were obtained

- **Names, field offsets and RVAs** come from the `dump.cs` produced by [Il2CppDumper](https://github.com/Perfare/Il2CppDumper) (input: `GameAssembly.dll` and `global-metadata.dat`).
- **Behavior notes** (what a method really checks, call order, data layouts) come from decompiling the same binary in [Ghidra](https://ghidra-sre.org/) after applying Il2CppDumper's Ghidra script. For a PE file Ghidra's address is `0x180000000` + RVA.
- Anything marked as tested was also confirmed by a live run in game.

!!! warning "Build specific"
    All RVAs and offsets are from **one game build**. They move after a game update; re-run Il2CppDumper to get new ones, and rely on the names. <!-- TODO: add Steam build id / game version here -->

RVAs are relative to the base of `GameAssembly.dll`.

## Methods

| Class | Method | RVA | Notes |
|---|---|---|---|
| `Cheats` | `get_Available` | `0x52A840` | force `true` to enable dev console |
| `Cheats` | `CheckAvailable` | `0x528D40` | force `true` |
| `IslandGrid` | `Awake` | `0x6130C0` | good place to set `HardLimit.radius` |
| `IslandGrid` | `CanPlaceBuilding` | `0x6143A0` | |
| `IslandGrid` | `ImportIsland` | `0x616340` | private, patchable |
| `IslandGrid` | `ExportIsland` | `0x614900` | |
| `IslandGrid` | `Generate` | `0x614EB0` | |
| `IslandGrid` | `GenerateJob` | `0x614D90` | |
| `IslandGrid` | `SaveIslandToClipboard` | `0x61A690` | unreliable (see below) |
| `IslandGrid` | `LoadIslandFromClipboard` | `0x616810` | unreliable (see below) |
| `Island` | `get_Owner` | `0x61F7D0` | |
| `Player` | `get_UserName` | `0x473BB0` | |
| `Player` | `get_StartGameReady` | `0x571AB0` | |
| `Player` | `get_Island` | `0x508770` | |
| `PlayerSummary` | `Update` | `0x5AA420` | |
| `PlayerSummary` | `OnSyncPhase` | `0x5AA1A0` | |
| `GameState` | `Island(Player)` | `0x5F2F60` | |
| `GameState` | `Update` | `0x5F70E0` | per-frame patch point |
| `BuildData` | `CanBuildType` | `0x628760` | `IsBuildable` -> `Unlocked` -> `SteamDemo` -> `HasResource` |
| `BuildData` | `CanRemoveNature` | `0x6288B0` | inline check: TREE / ROCK / SCORCHED_EARTH / LAVA |
| `BuildData` | `CleanTile` | `0x628AB0` | |
| `NatureTile` | `Cleanable` | `0x61FE70` | not called by `CanRemoveNature` |
| `CleanTileTool` | `Use` | `0x589B90` | only fires a `UnityEvent` |
| `CleanTileTool` | `Yield` | `0x589CF0` | |
| `BuildingType` | `get_DisplayName` | `0x5F0E10` | |
| `BuildingTypes` | `get_Types` | `0x5717B0` | ids in save-file order; did not fire in testing, see below |
| `RulesBuilding` | `OnEnable` | `0x5471E0` | holds `BuildingTypes` at `+0x18` |
| `GridTransform` | `Transform` | `0x6105D0` | point + rotation + translation |

## Fields

| Class | Field | Offset |
|---|---|---|
| `IslandGrid` | `size` (`int2`) | `0xF0` |
| `IslandGrid` | `generationParameters` | `0x1E8` (`HardLimit.radius` at `+0`) |
| `IslandGrid` | `snapshotIslandSize` | `0x298` |
| `IslandGrid` | `arraySize` | `0x2AC` |
| `Island` | `grid` | `0x98` |
| `Island` | `owner` (`NetworkedVar<Player>`) | `0x90` |
| `Player` | `UserName` (`string`) | `0x110` |
| `PlayerSummary` | `island` | `0xD0` |
| `PlayerSummary` | `attackers` (`List<Image>`) | `0xE8` (`_size` at `+0x18`) |
| `PlayerSummary` | `gameState` | `0xC8` |
| `BuildingType` | `localPositions` (`int2[]`) | `0x18` |
| `BuildingType` | `localPositionsInWater` | `0x20` |
| `BuildingType` | category / tier | `0x10` / `0x78` |
| `BuildingType` | cost block | `0x58 - 0x74` |
| `BuildData` | island grid chain | `+0xA0` -> grid `+0x98` |
| `UnlockBuildingTechnology` | `BuildingTypes*` / id | `0x38` / `0x40` |

## Known quirks

- `BuildingTypes.get_Types` did not fire when patched late, even with buildings on screen: the `BuildingTypes` object is created before a late patch is applied (`RulesBuilding.OnEnable`). `BuildData.Unlocked(BuildingType)` is called repeatedly, so a patch on it can capture a `BuildingType`, and from there the `BuildingType[]` that contains it. Patching early (plugin `Load`) may avoid the problem - untested.
- `SaveIslandToClipboard` / `LoadIslandFromClipboard` rely on `GUIUtility.systemCopyBuffer` and were unreliable when called outside `OnGUI`. Prefer calling `ExportIsland` / `ImportIsland` and handling the clipboard yourself.
