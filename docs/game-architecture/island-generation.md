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
| `numberMains`      | `[2..3]` | (min, max) number of main cliff formations.                 |
| `numberExtensions` | `[1..3]` | (min, max) number of branches per main cliff.               |
| `minDistance`      | 8        | Minimum distance between neighboring cliffs.                |
| `minSpacing`       | 5        | Minimum gap between branches within a single cliff.         |

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
| `maximumDistance`                        | 22    | Maximum distance from the shore where fish may appear.                 |
| `validMarginDistance`                    | 2     | Allowed offset from the water edge.                                    |
| `maxCount`                               | 20    | Hard cap on the number of fish dots.                                   |

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

`GenerateJob` is the Burst executor. Besides its fields (`parameters`, `references`, `seed`, `ground`, `nature`, `innerCliffs`, `score`) it carries a set of static helpers:

```
IsInIsland(distance, moistureRange)
Gaussian(ref displayClass)
Height(int2 position, ref displayClass)
Moisture(int2 position, ref displayClass)
GetGroundTile(GroundTile a, GroundTile b)  // blending neighboring types
AssignGround(...)                          // ground type from alt/moisture/limit
PlaceInnerCliffs(...)
PlaceBlobs(...)
PlaceDots(...)
ScoreTerritory(...)
```

`AssignGround` is where a concrete `GroundTile` is picked from altitude, moisture, and `HardLimit`. It runs first because all subsequent nature passes filter by ground type.

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

