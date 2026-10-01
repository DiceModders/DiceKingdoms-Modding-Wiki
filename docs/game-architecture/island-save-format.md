---
summary: Binary layout of the island code (clipboard / save string) and how to decode and encode it
---
# Island Save Format

An **island code** is the string the game's dev console copies to the clipboard (`saveisland`) and reads back (`loadisland`). It is a serialized `IslandGrid.SyncData` - the same structure that is network-synced between players.

!!! info "Status"
    Fully reverse engineered and validated with decode -> encode -> byte-identical round trips on real islands.

## Pipeline

```
island code (text)
  -> base64 decode
  -> raw DEFLATE inflate         (NO zlib header, NO gzip header)
  -> MLAPI packed-int stream     (IslandGrid.SyncData)
```

Encoding is the same pipeline in reverse. In JavaScript the browser primitives `DecompressionStream('deflate-raw')` / `CompressionStream('deflate-raw')` do exactly this; in C# use `DeflateStream` (not `ZLibStream`).

## Packed integers

Every number in the stream is an MLAPI `UInt64Packed` varint. Signed values (`int32`) are **zigzag-encoded first**: `0, -1, 1, -2, 2 ...` -> `0, 1, 2, 3, 4 ...`

| First byte `b` | Total size | Value |
|---|---|---|
| `0x00 - 0xF0` | 1 byte | `b` |
| `0xF1 - 0xF8` | 2 bytes | `(b - 0xF1) * 256 + next + 0xF0` |
| `0xF9` | 3 bytes | `hi * 256 + lo + 0x8F0` |
| `0xFA - 0xFF` | `1 + (b - 0xF7)` bytes | the following `n = b - 0xF7` bytes, little endian |

## Stream layout

All counts and values below are packed ints (`i32` = zigzag, `u64` = raw).

```text
buildings[]   : i32 count, then count x Building
remains[]     : i32 count, then count x Building      (destroyed buildings, same layout)
groundTiles   : RLE  (i32 totalLength, then runs of {i32 runLength, i32 value})
natureTiles   : RLE  (same layout)
innerCliffs[] : i32 count, then count x { i32 c, i32 a, i32 b, i32 x, i32 y, i32 orient, i32 mirror }
```

`Building` (used by both `buildings` and `remains`):

| Field | Type | Meaning |
|---|---|---|
| `type` | i32 | Building type id (see [Buildings](buildings.md)) |
| `x`, `y` | i32 | Grid position of the building origin |
| `orient` | i32 | Rotation, 0-3 (see [Footprint and rotation](buildings.md#footprint-and-rotation)) |
| `mirror` | i32 | `0` / `1` - mirrors X |
| `health` | i32 | Current health |
| `disaster` | `u64` flag, then `u64` payload if flag != 0 | Nullable disaster state |
| `turn` | i32 | Turn counter |

The tile arrays are row-major, `index = y * side + x`, with `totalLength == side * side`.

### Ground tile values

| Value | Name |
|---|---|
| 0 | `WATER` |
| 1 | `WATER_ROCK` (a *ground* tile, not a nature tile) |
| 2 | `GRASS` |
| 3 | `SAND` |
| 4 | `INNER_CLIFF` |
| 5 | `METEORITE` |
| 6 | `PIER` |

### Nature tile values

| Value | Name |
|---|---|
| -1 | none |
| 0 | `TREE` |
| 1 | `ROCK` |
| 2 | `SCORCHED_EARTH` |
| 3 | `LAVA` |
| 4 | `FLOWER` |
| 5 | `FISH` |

## Reference decoder (C#)

```c#
using System.IO.Compression;

public sealed class PackedReader
{
    readonly byte[] b; int p;
    public PackedReader(byte[] data) { b = data; }
    public int Position => p;

    byte Byte() => p < b.Length ? b[p++] : throw new EndOfStreamException();

    public ulong U64()
    {
        int first = Byte();
        if (first <= 0xF0) return (ulong)first;
        if (first < 0xF9) return (ulong)((first - 0xF1) * 256 + Byte() + 0xF0);
        if (first == 0xF9) { int hi = Byte(), lo = Byte(); return (ulong)(hi * 256 + lo + 0x8F0); }
        int n = first - 0xF7;
        ulong v = 0;
        for (int i = 0; i < n; i++) v |= (ulong)Byte() << (8 * i);
        return v;
    }

    // zigzag: even -> x/2, odd -> -(x+1)/2
    public int I32() { ulong x = U64(); return (x & 1) == 0 ? (int)(x >> 1) : -(int)((x + 1) >> 1); }
}

public static byte[] Unpack(string islandCode)
{
    byte[] compressed = Convert.FromBase64String(islandCode.Trim());
    using var input = new MemoryStream(compressed);
    using var deflate = new DeflateStream(input, CompressionMode.Decompress); // raw DEFLATE
    using var output = new MemoryStream();
    deflate.CopyTo(output);
    return output.ToArray();
}
```

Reading the whole structure (works the same inside a BepInEx plugin):

```c#
List<Building> ReadBuildings(PackedReader r)
{
    int n = r.I32();
    var list = new List<Building>(n);
    for (int i = 0; i < n; i++)
    {
        var b = new Building {
            Type = r.I32(), X = r.I32(), Y = r.I32(),
            Orient = r.I32(), Mirror = r.I32(), Health = r.I32()
        };
        if (r.U64() != 0) b.Disaster = r.U64();
        b.Turn = r.I32();
        list.Add(b);
    }
    return list;
}

int[] ReadRle(PackedReader r)
{
    int total = r.I32();
    var values = new List<int>(total);
    while (values.Count < total)
    {
        int run = r.I32(), v = r.I32();
        for (int i = 0; i < run; i++) values.Add(v);
    }
    return values.ToArray();
}
```

A complete JavaScript implementation (reader, writer, RLE) is in the [Island Editor](../tools/island-editor.md) source.

!!! warning "Round-trip rule"
    When editing, always re-encode with the exact same field order. Unknown or unedited lists (`remains`, `innerCliffs`) must be written back untouched - the game rejects or mis-loads streams that are even one packed int off.
