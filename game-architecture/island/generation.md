# Island Generation

The island generation system turns a **seed** and a set of **tuning parameters** into a playable island. It runs in two stages: first the **tile** stage - every grid cell gets a ground type and possibly a nature object; then the **mesh** stage - the tile layer is turned into a `PolygonMesh` with a `HeightMap`. Both stages are Burst‑compiled `IJob`s launched from static methods of `IslandGeneration`.

## IslandGeneration

`IslandGeneration` is the central class responsible for the entire island generation pipeline. The bodies of nested types (`Parameters`, `NoiseLayers`, `GenerateJob`, etc.) are expanded in the following sections.

```c#
namespace DiceKingdoms.Game;

public static class IslandGeneration
{
    public static float EPSILON;
    public static float WATER_MAX_DEPTH;

    public sealed class ConnectedComponents<T> : Il2CppSystem.ValueType
        where T : new() { /* section "ConnectedComponents" */ }

    [Serializable, StructLayout(LayoutKind.Explicit)]
    public struct Parameters { /* section "Parameters" */ }

    public sealed class References : Il2CppSystem.ValueType
    { /* section "References" */ }

    [StructLayout(LayoutKind.Explicit)]
    public struct NoiseLayers { /* section "NoiseLayers" */ }

    public sealed class GenerateOutput : Il2CppSystem.ValueType
    { /* section "GenerateOutput" */ }

    public sealed class GenerateJob : Il2CppSystem.ValueType
    { /* section "GenerateJob" */ }

    public sealed class HeightMap : Il2CppSystem.ValueType
    { /* section "HeightMap" */ }

    public struct Bounds { /* section "Bounds" */ }

    public sealed class MeshOutput : Il2CppSystem.ValueType
    { /* section "MeshOutput" */ }

    public sealed class MeshJob : Il2CppSystem.ValueType
    { /* section "MeshJob" */ }


    public static GenerateOutput Generate(
        ref Parameters parameters,
        ref IEnumerable<InnerCliff.Key> innerCliffKeys,
        ref int2 size,
        ref uint seed);

    public static MeshOutput BuildMesh(
        IslandGrid.Layer<GroundTile> layer,
        Biome biome,
        int heightMapSubdivide,
        PolygonMesh lavaTileMesh,
        PolygonMesh pierTileMesh);

    public static PolygonMesh BuildAltitudeMesh(
        ref Parameters parameters,
        float2 center,
        int2 size,
        ref Unity.Mathematics.Random random);

    public static PolygonMesh BuildHeightMapMesh(
        IslandGrid.Layer<GroundTile> layer,
        Biome biome,
        int heightMapSubdivide,
        PolygonMesh lavaTileMesh,
        PolygonMesh pierTileMesh);

    public static void HeightMapFromMesh(
        ref PolygonMesh mesh,
        ref HeightMap heightMap);

    public static void MaxFilter(
        ref HeightMap heightMap,
        ref float radius);

    private static ConnectedComponents<T> ComputeConnectedComponents<T, P>(
        UnsafeLayer<T> layer, P predicate, Allocator allocator)
        where T : unmanaged;
}
```

Generation is split into two sequential stages.

**The first stage - tiles.** The entry point is `Generate(ref parameters, ref innerCliffKeys, ref size, ref seed)`. First, `NoiseLayers.Generate(...)` is called: it fixes random offsets for the altitude and moisture noise and produces `NoiseLayers` - an immutable struct from which `Altitude(position)` and `Moisture(position)` can later be sampled. Then a `GenerateJob` is created, which in its Burst `Execute()` walks every grid cell and makes four passes:

- **AssignGround** - picks the ground type from altitude, moisture, and `HardLimit`.

```c#
namespace DiceKingdoms.Game;

public enum GroundTile
{
	WATER,
	WATER_ROCK,
	GRASS,
	SAND,
	INNER_CLIFF,
	METEORITE,
	PIER
}
```

- **PlaceInnerCliffs** - places inland cliffs using `numberMains`, `numberExtensions`, `minDistance`, `minSpacing`.
- **PlaceBlobs** - plants trees and rocks as clusters ("blobs"), filtered by `moistureRange` and `minDistance`.
- **PlaceDots** - places point objects (fish) via a probability mask; flowers are distributed along the way by `flowerProbability`.
- **ScoreTerritory** - computes three scores (`buildSpace`, `naturalResources`, `cliffs`) and their sum in `ScoreData`.

`Generate` returns a `GenerateOutput` — a job handle. The caller can either `Complete()` it and pull the layers via `GetGround` / `GetNature`, or compare several `GenerateOutput`s via `CompareTo` (sorted by score) and keep the best.

**The second stage - mesh.** `BuildMesh(layer, biome, …)` takes the already classified tile layer and builds geometry. Inside `MeshJob.Execute()` it walks the four diagonals of each cell (`Diagonal`), classifies corners (`CornerKind`), and assembles the `PolygonMesh`. Height differences between neighboring cells become vertical walls via `WallHorizontalStep` and `WallVerticalStep`. The result is `MeshOutput`: the `PolygonMesh` itself and the `HeightMap`.

Both stages are fully deterministic: the same `seed` and `Parameters` produce the same island.

## Parameters

!!! info "Example mode:" 
    [IslandConfig](https://github.com/DiceModders/IslandConfig) - adds configuration for island generation parameters

`IslandGeneration.Parameters` carries every tunable value.

```c#
[Serializable]
public struct Parameters
{
    [FieldOffset(0)]  public HardLimit   hardLimit;
    [FieldOffset(8)]  public Altitude    altitude;
    [FieldOffset(36)] public Moisture    moisture;
    [FieldOffset(44)] public Sand        sand;
    [FieldOffset(52)] public InnerCliffs innerCliffs;
    [FieldOffset(76)] public Nature      nature;

    public struct HardLimit   { /* section 1 */ }
    public struct Altitude    { /* section 2 */ }
    public struct Moisture    { /* section 3 */ }
    public struct Sand        { /* section 4 */ }
    public struct InnerCliffs { /* section 5 */ }
    public struct Nature      { /* section 6 */ }
    public struct Gaussian    { public float average; public float deviation; }
}
```

### 1. HardLimit

```c#
public struct HardLimit
{
    public int radius;
    public int power;
    public bool IsWithin(int2 size, int2 position);
}
```

| Field    | Value | Meaning                                                                       |
|----------|-------|-------------------------------------------------------------------------------|
| `radius` | 22    | Radius of the hard circular boundary (in tiles). Beyond it there is no land.  |
| `power`  | 3     | no info |

`IsWithin(size, position)` is a predicate called for every cell. It cuts off everything outside the `radius` circle before any altitude thresholds kick in.

### 2. Altitude

```c#
public struct Altitude
{
    public float noiseHorizontalOffsetScale;
    public float noiseHorizontalScale;
    public float distancePower;
    public float noiseVerticalScale;
    public float distanceVerticalScale;
    public float verticalOffset;
    public int   islandMinSize;
}
```

| Field                        | Value | Meaning                                                                 |
|------------------------------|-------|-------------------------------------------------------------------------|
| `noiseHorizontalOffsetScale` | 5     | noise offset — determines where hills and hollows land.                 |
| `noiseHorizontalScale`       | 0.05  | Horizontal noise scale. Larger - smoother terrain features.             |
| `distancePower`              | 3.5   | Exponent of distance from the center. Larger it is, the flatter the interior and the steeper the shore. |
| `noiseVerticalScale`         | 0.4   | Amplitude of the vertical noise.                                        |
| `distanceVerticalScale`      | 1.4   | How quickly height drops toward the island edge.                        |
| `verticalOffset`             | 0.61  | Global vertical shift.                                                  |
| `islandMinSize`              | 6     | Minimum island size in cells, smaller ones are rejected.                |


### 3. Moisture

```c#
public struct Moisture
{
    public float horizontalOffsetScale;
    public float horizontalScale;
}
```

| Field                   | Value | Meaning                                                            |
|-------------------------|-------|--------------------------------------------------------------------|
| `horizontalOffsetScale` | 1     | Moisture noise offset - shifts drought and humidity zones.         |
| `horizontalScale`       | 0.05  | Moisture noise scale. Larger - bigger and smoother zones.          |

### 4. Sand

```c#
public struct Sand
{
    public float margin;
    public int   minSize;
}
```

| Field     | Value | Meaning                                                                 |
|-----------|-------|-------------------------------------------------------------------------|
| `margin`  | 2     | Width of the sand strip along the shore (offset from water).            |
| `minSize` | 8     | Minimum size of a sand patch - prevents micro‑beaches.                  |

Sand is applied to cells whose height sits just above the water level within `margin` of the shore. `minSize` prunes isolated sand tiles.

### 5. InnerCliffs

```c#
public struct InnerCliffs
{
    public int2 numberMains;
    public int2 numberExtensions;
    public float minDistance;
    public float minSpacing;
}
```

| Field              | Value    | Meaning                                                     |
|--------------------|----------|-------------------------------------------------------------|
| `numberMains`      | `[2..3]` | (min, max) number of main cliff formations per island.      |
| `numberExtensions` | `[1..3]` | (min, max) number of branches per main cliff.               |
| `minDistance`      | 8        | Minimum distance of a main cliff from the grid centre (keeps the castle area free). |
| `minSpacing`       | 5        | Minimum distance between cliffs (`Free` checks it against every cliff already placed). |

Main cliffs are tried at random positions of the whole grid, 256 attempts in total; every placed main then gets its branches (256 attempts each). The numbers are per island, not per tile.

#### Hill pieces (`InnerCliff`)

A hill on the island is a list of pieces, `IslandGrid.innerCliffs` (`List<InnerCliff>`). Each piece is a key and a transform:

```c#
public struct InnerCliff
{
    public Key key;                 // which piece
    public GridTransform transform; // position, orientation 0-3, mirroring 0/1 - the same transform as buildings
    public struct Key { public int2 size; public Kind kind; }
    public enum Kind { MAIN = 0, EXTENSION = 1 }
}
```

The game has five pieces, each with several 3D models per civilization (`InnerCliffVisualHub`):

| Kind | Size | Looks like |
|---|---|---|
| `MAIN` | 3 x 3 | the big connected hill |
| `MAIN` | 2 x 2 | a smaller hill |
| `EXTENSION` | 2 x 2 | a slope added to a hill |
| `EXTENSION` | 1 x 2 | a narrow slope |
| `EXTENSION` | 1 x 1 | a single tile sticking out of a hill |

- **Tiles:** `InnerCliff.Positions()` is the box between `transform.Transform(0, 0)` and `transform.Transform(size - 1)`, so a rotated piece covers its size turned by 90 degrees. `GridTransform.Transform(p) = position + (rotate(p) with x * (mirror ? -1 : 1))`, the same rule as building footprints.
- **Ground:** the tiles under a piece are ground `INNER_CLIFF` (4). `AddInnerCliff` does **not** set the ground; it only adds the piece to the list and spawns a model. Ground 4 with no piece is a hill without a model: it looks like flat ground but blocks building like a hill. `RemoveInnerCliff` sets the tiles back to grass.
- **Model:** `AddInnerCliff` asks `PlayerSkins.GetInnerCliffVisualHubPrefab(key)` and picks one of the hub's models at random (`RandomExtensions.NextSample`), so the same island looks a little different every time it is loaded or applied.
- **Unknown keys break the island.** `DefaultPrefabs.InnerCliff(key)` reads a dictionary; a key the game has no piece for throws `KeyNotFoundException`, and `AddInnerCliff` then fails with a `NullReferenceException` **after** the piece was already added to the list. `ImportIsland` stops halfway, and the broken entries make later rebuilds (and quitting the match) fail again and again. Check keys before importing an island code. The field order in island codes is in [Save format](save-format.md).

### 6. Nature

```c#
public struct Nature
{
    public Blobs trees;
    public Blobs rocks;
    public Dots  fish;
    public float flowerProbability;

    [Serializable] public struct Blobs { /* 6.1 */ }
    [Serializable] public struct Dots  { /* 6.2 */ }
}
```

#### 6.1. Blobs

```c#
[Serializable]
public struct Blobs
{
    public float2   moistureRange;
    public float    minDistance;
    public Gaussian count;
    public Gaussian size;
}
```

| Field            | Value (Trees)  | Value (Rocks)  | Meaning                                                        |
|------------------|----------------|----------------|----------------------------------------------------------------|
| `moistureRange`  | `[-0.5 .. 1]`  | `[-1 .. 0.5]`  | Humidity range in which this type can appear.                  |
| `minDistance`    | 4              | 5              | Minimum distance between neighboring blobs.                    |
| `count`          | `N(3.3, 0.4)`  | `N(8.5, 1)`    | Number of blobs: mean and sigma of a Gaussian                  |
| `size`           | `N(30, 10)`    | `N(4, 0.8)`    | Blob radius: mean and sigma.                                   |

Trees and rocks are generated as **blobs** — clusters of same‑type cells rather than single tiles. `count` and `size` are Gaussian, so a single run produces natural variety. The `moistureRange` of trees and rocks overlaps — formally, both types may land on the same cell; `PlaceBlobs` resolves overlaps by iteration order.

#### 6.2. Dots

```c#
[Serializable]
public struct Dots
{
    public float idealMoisture;
    public float maximumMoistureAmplitude;
    public float probabilityAtIdealMoisture;
    public float probabilityAtMaximumMoistureAmplitude;
    public float probabilityPower;
    public float maximumDistance;
    public int   validMarginDistance;
    public int   maxCount;
}
```

| Field                                    | Value | Meaning                                                                |
|------------------------------------------|-------|------------------------------------------------------------------------|
| `idealMoisture`                          | 0.4   | Humidity at which fish probability is maximal.                         |
| `maximumMoistureAmplitude`               | 0.3   | Maximum deviation from `idealMoisture` where fish is still possible.   |
| `probabilityAtIdealMoisture`             | 0.6   | Fish probability at the ideal point (0..1).                            |
| `probabilityAtMaximumMoistureAmplitude`  | 0     | Probability at maximum deviation.                                      |
| `probabilityPower`                       | 1.5   | Curve exponent: how fast probability falls off with deviation.         |
| `maximumDistance`                        | 22    | Maximum distance from the **grid centre** (not the shore) where fish may appear. |
| `validMarginDistance`                    | 2     | Every tile within this distance of a fish must be valid for fish (water): fish keep this far from land. |
| `maxCount`                               | 20    | Hard cap on the number of fish dots.                                   |

How `Dots` places fish (`GenerateJob.<Execute>g__Dots|9_6`):

1. Walk every tile **row by row, starting at the bottom row** (`y = 0`), left to right.
2. Skip tiles that already have nature.
3. Skip the tile unless every tile within `validMarginDistance` (a spiral around it) is valid for the nature type (`NatureTileExtensions.ValidOn`; water for fish).
4. Skip it unless it lies within `maximumDistance` of the grid centre.
5. Moisture check: `|moisture - idealMoisture| < maximumMoistureAmplitude`, then a random roll against the probability for that moisture.
6. Stop as soon as `maxCount` fish are placed.

So the fish are a band of open water between "two tiles from land" and the `maximumDistance` circle. Because of step 6, when the band has more candidates than `maxCount`, the fish fill up from the bottom and the top of the island gets none. On the game's own islands the band is narrow enough that this rarely shows; on a bigger island it always does (see [Island size for modders](#island-size-for-modders)).

#### 6.3. Flowers

| Field               | Value | Meaning                                           |
|---------------------|-------|---------------------------------------------------|
| `flowerProbability` | 0.05  | Probability of flowers on a suitable cell (0..1). |

Flowers are neither blobs nor dots — just a simple per‑tile probability. It applies to cells whose ground type is suitable.

### 7. Gaussian

```c#
public struct Gaussian
{
    public float average;
    public float deviation;
}
```

| Field       | Meaning                                              |
|-------------|------------------------------------------------------|
| `average`   | Mean value.                                          |
| `deviation` | Standard deviation (σ) of the normal distribution.   |

A helper struct for `count` and `size` in `Blobs`. Sampling goes through `Unity.Mathematics.Random` so it works in Burst.

---

## NoiseLayers

`NoiseLayers` is the deterministic sampler consumed by both stages.

```c#
[StructLayout(LayoutKind.Explicit)]
public struct NoiseLayers
{
    [FieldOffset(0)]   public readonly Parameters parameters;
    [FieldOffset(168)] public readonly float2     center;
    [FieldOffset(176)] public readonly int2       size;
    [FieldOffset(184)] public readonly float2     altitudeOffset;
    [FieldOffset(192)] public readonly float2     moistureOffset;

    public float Distance(float2 p);
    public float Altitude(float2 position);
    public float Moisture(float2 position);

    public static NoiseLayers Generate(
        ref Parameters parameters, float2 center, int2 size,
        ref Unity.Mathematics.Random random);
}
```

| Field            | Meaning                                                      |
|------------------|--------------------------------------------------------------|
| `parameters`     | A copy of `Parameters`                                       |
| `center`         | Island center in tile coordinates.                           |
| `altitudeOffset` | Random altitude noise offset, chosen once in `Generate`.     |
| `moistureOffset` | Random moisture noise offset.                                |

## GenerateOutput, GenerateJob and ScoreData

```c#
public sealed class GenerateOutput : Il2CppSystem.ValueType
{
    public GenerateJob job;
    public JobHandle   handle;

    public Il2CppStructArray<InnerCliff> InnerCliffs { get; }
    public ScoreData                     Score       { get; }

    public GenerateOutput(GenerateJob job, JobHandle handle);
    public void Complete();
    public void GetGround(IslandGrid.Layer<GroundTile> ground);
    public void GetNature(IslandGrid.Layer<Nullable<NatureTile>> nature);
    public virtual int CompareTo(GenerateOutput other);
}
```

| Member                       | Meaning                                                             |
|------------------------------|---------------------------------------------------------------------|
| `Complete()`                 | Wait for the job to finish.                                         |
| `GetGround(ground)`          | Copy the job result into the ground layer.                          |
| `GetNature(nature)`          | Copy the job result into the nature layer.                          |
| `CompareTo(other)`           | Compare by score — allows iterating several `GenerateOutput`s and keeping the best. |

`ScoreData` is a component‑wise quality estimate:

```c#
public struct ScoreData
{
    public float buildSpace;        // room for building
    public float naturalResources;  // trees, rocks, fish
    public float cliffs;            // inland cliffs
    public float Total { get; }     // sum of the three
}
```

`GenerateJob` is the Burst executor. Besides its fields (`parameters`, `references`, `seed`, `ground`, `nature`, `innerCliffs`, `score`), `Execute` uses these local functions (compiler names `<Execute>g__Name|9_n`):

| Function | What it does |
|---|---|
| `InRange(value, range)`, `Sample(gaussian)` | helpers; `Sample` draws from a `Gaussian` |
| `Altitude(position)`, `Moisture(position)` | noise values of a tile |
| `TileAltitude(tile)`, `Max(left, right)` | order of ground types: water lowest, then sand, then grass |
| `Allow(position, tile, ground, allowed)` | sea-level flood fill (see below) |
| `Free(cliff, ...)`, `Add(cliff, ...)` | inner cliffs: spacing check, then place |
| `Blobs(blobs, ...)`, `Blob(maxSize, ...)`, `Valid(position, ...)` | trees and rocks clusters |
| `Dots(dots, ...)` | fish |
| `BuildSpace`, `NaturalResources`, `Cliffs` | the three parts of `ScoreData` |

Order in `Execute`: altitude and ground per tile, the connected land is kept (`ComputeConnectedComponents`), the sea level, inner cliffs, trees, rocks, fish, flowers, then the score.

**Sea level.** `Allow` starts at tile (0, 0) with water. Every tile gets the lowest ground type that a path from the open sea reaches it through, never lower than its own. In the game this function calls itself for the four neighbours of every tile it changes, so its depth grows with the sea area. See "Island size for modders" below.

!!! warning "Patching job code"
    The game runs the **Burst** copy of `GenerateJob` (`lib_burst_generated.dll`), not the IL2CPP method. A Harmony patch on `GenerateJob.Execute` or its local functions never runs. Changing what a Burst job does needs a native patch of the Burst library.

## BuildMesh, MeshOutput and MeshJob

```c#
public static MeshOutput BuildMesh(
    IslandGrid.Layer<GroundTile> layer,
    Biome biome,
    int heightMapSubdivide,
    PolygonMesh lavaTileMesh,
    PolygonMesh pierTileMesh);
```

`BuildMesh` takes the classified tile layer and builds 3D geometry.

```c#
public sealed class MeshOutput : Il2CppSystem.ValueType
{
    public MeshJob   job;
    public JobHandle handle;

    public bool         IsCompleted { get; }
    public PolygonMesh  Mesh        { get; }
    public HeightMap    HeightMap   { get; }

    public MeshOutput(MeshJob job, JobHandle handle);
    public void Complete();
}
```

| Member        | Meaning                             |
|---------------|-------------------------------------|
| `IsCompleted` | Whether the job is ready.           |
| `Mesh`        | The built `PolygonMesh`.            |
| `HeightMap`   | The height map extracted from mesh. |
| `Complete()`  | Wait for the job to finish.         |

```c#
public sealed class MeshJob : Il2CppSystem.ValueType
{
    public enum CornerKind : byte { EMPTY, STRAIGHT, CORNER, FULL }

    public struct Diagonal
    {
        public int2  CenterToCornerOffset { get; }
        public Angle Angle                { get; }
        public int   X                    { get; }
        public int   Y                    { get; }
        public Diagonal Next              { get; }
        public Diagonal Previous          { get; }
        public Diagonal Opposite          { get; }
        public int2 Corner(int2 center);
        public int2 Center(int2 corner);
    }

    public struct Angle
    {
        public readonly byte value;   // 0..255 -> 0..2π
        public static Angle ZERO;
        public static Angle PI;
        public float Sin { get; }
        public float Cos { get; }
        // +, -, *, /, Lerp, ==, !=
    }

    public struct WallHorizontalStep { /* ... */ }
    public struct WallVerticalStep   { /* ... */ }
}
```

`MeshJob` walks the four diagonals of each cell. `CornerKind` classifies a corner by how the cell connects to its neighbors (`EMPTY` — isolated, `FULL` — connected on all sides). `Angle` is packed into a single byte (`0..255` → `0..2π`) with precomputed sin/cos — compact, cache‑friendly, and Burst‑safe. Vertical geometry on height differences is emitted via `WallHorizontalStep` and `WallVerticalStep`.

## HeightMap

```c#
public sealed class HeightMap : Il2CppSystem.ValueType
{
    public struct PositionsEnumerable { /* ... */ }

    public int2  Size { get; }
    public float this[float2 position] { get; }
    public float this[int2 position]   { get; set; }

    public static HeightMap operator +(HeightMap a, HeightMap b);

    public PositionsEnumerable Indexes(float threshold);
    public HeightMap Copy(Allocator allocator);
}
```

| Member                       | Meaning                                                             |
|------------------------------|---------------------------------------------------------------------|
| `Size`                       | Map size in cells.                                                  |
| `this[float2]` / `this[int2]`| Access to the height at a position.                                 |
| `operator +`                 | Element‑wise addition of two height maps.                           |
| `Indexes(threshold)`         | Enumerates all positions above a threshold.                         |
| `Copy(allocator)`            | A copy of the map with a given allocator.                           |

## Helper types

### Bounds

```c#
public struct Bounds
{
    public float2 min;
    public float2 max;
    public static Bounds EMPTY;
    public Bounds(float2 min, float2 max);
    public void Add(float2 point);
}
```

| Member        | Meaning                                                       |
|---------------|---------------------------------------------------------------|
| `min`, `max`  | AABB corners.                                                 |
| `EMPTY`       | Empty bounds (for accumulation via `Add`).                    |
| `Add(point)`  | Extend bounds so they include the point.                      |

### ConnectedComponents

```c#
public sealed class ConnectedComponents<T> : Il2CppSystem.ValueType
    where T : new()
{
    public NativeList<Component> components;

    public sealed class Component : Il2CppSystem.ValueType
    {
        public UnsafeList<int2> positions;
        public void Add(int2 position);
    }

    public ConnectedComponents(Allocator allocator);
    public void Add(Component component);
}
```

| Member        | Meaning                                                        |
|---------------|-----------------------------------------------------------------|
| `components`  | List of found connected components.                            |
| `Add(position)`| Add a position to the component.                              |

A BFS over `UnsafeLayer<T>` grouping same‑type cells. Driven by a predicate `P` - the caller decides what counts as "connectivity". Used, for example, to reject nature blobs that are too small or severed pieces of the island.

### References

```c#
public sealed class References : Il2CppSystem.ValueType
{
    public UnsafeList<InnerCliff.Key> innerCliffKeys;

    public References(IEnumerable<InnerCliff.Key> innerCliffKeys, Allocator allocator);
}
```

| Member           | Meaning                                                     |
|------------------|--------------------------------------------------------------|
| `innerCliffKeys` | Cliff prefab keys used by `PlaceInnerCliffs` to pick a cliff type. |

## Island size for modders

- The altitude falls off with `distance / distance to the grid corner`, so the slope towards the sea is set by the grid size (49 x 49), not by `hardLimit.radius`. Raising the radius alone (22 -> 24) adds almost no land, because the noise is already under sea level near the old limit; to fill a bigger radius, divide `altitude.distanceVerticalScale` by the same factor.
- `generationParameters` is read from the island grid when the island is generated, so a prefix on `IslandGrid.Awake` can change it. The host generates the islands in `GameState.SpawnIslands` and sends every tile to the clients (`IslandGrid.ApplySync`, run-length encoded), so every computer's grid must have the same size.
- The radius is useful up to half the grid (24). A bigger grid means changing `IslandGrid.size`:
    - on the island prefab (`GameState.islandPrefab.grid`) before `GameState.SpawnIslands`, because the islands are spaced with the prefab's size (see [Turn phases](../turn-phases.md));
    - on every island in an `IslandGrid.Awake` prefix, on every computer: `Awake` builds every tile layer from it.
- The islands exist before `GameState.StartGame` runs, so a lobby rule that resizes them must be read when they spawn. ModCore freezes the rules at that moment; check `ModCore.Lobby.Rules.Fixed` so menu and editor islands stay normal.
- What follows the size by itself: the combat map (`CombatGrid.Init`, twice the island), the meshes (32-bit indices) and the network data (run-length, so it grows with the coast, not with the area).
- The camera: `CameraMovement.Focus` gets its radius from the grid, and the start view is `radius / tan(fov / 2)` high, so a big island opens far away. The panning area (`baseHorizontalBound`) is measured around the map centre and does not grow by itself.
- Put the prefab and the camera back on `ModCore.Match.Reset`: they live on after the match.
- **Grids above 119 tiles crash without ModCore.** The sea-level fill (`Allow`) recurses once per sea tile and runs on a Unity job thread with a 1 MB stack, which holds about 9,300 calls. At 121 x 121 the game closes with a stack overflow while `SpawnIslands` generates the islands, with nothing in the log. ModCore replaces the Burst function with an iterative version that gives the same islands (see [ModCore](../../libraries/modcore/index.md)). Tested up to 201.
- **Amounts do not grow by themselves.** `innerCliffs.numberMains`, `nature.trees.count`, `nature.rocks.count` and `nature.fish.maxCount` are per island. A bigger land keeps the same number of hills, forests and rock fields unless they are scaled too. Grow the counts with the land area and fish with the coast length; leave the cluster `size` alone.
- **Fish on a big island.** `fish.maximumDistance` is measured from the grid centre, so it has to grow with the land radius or every fish lands inside the land's circle and none can be placed. Once it does, the band of open water holds far more candidates than `maxCount`, and because `Dots` fills rows from the bottom (see [Dots](#62-dots)) the fish pile up below the island. A clean fix on the host: lift `maxCount` during generation, then keep a random selection of the fish (up to the target count) before the island is drawn and sent. `GameState.SpawnIslands` calls `IslandGrid.SetGenerateOutput`, which copies the ground and nature layers in (`GenerateOutput.GetGround` / `GetNature`) and then calls `SetInnerCliffs`, `BuildMesh` and `UpdateNatureVisuals`; a prefix on `UpdateNatureVisuals` while `SpawnIslands` runs sees the finished nature layer. Clients receive the finished island (`ApplySync`), so only the host changes it.
- **Island codes are tied to the grid size.** `IslandGrid.ImportIsland` loads into the grid the island already has and throws `IndexOutOfRangeException` for a code of another size. Re-centre the tiles, buildings, remains and inner cliffs on the target size first.
