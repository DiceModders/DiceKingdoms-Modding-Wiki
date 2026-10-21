---
summary: The island code format (IslandGrid.SyncData), footprints of buildings and the building id table
---

# Island code

The game exports and imports islands as a text code (`IslandGrid.ExportIsland` / `ImportIsland`). The format below was decoded from the game code and checked by
round-trip (encode, decode, encode again gives the same bytes).

## Encoding layers

1. The code is **base64** text.
2. The bytes are **raw DEFLATE** (no zlib header).
3. The inflated bytes are a stream of packed integers (the MLAPI packing) holding a `IslandGrid.SyncData`.

### Packed integers

Each integer is zigzag encoded and written as a variable-length 64-bit value:

| First byte `b` | Bytes | Value |
|---|---|---|
| `b <= 0xF0` | 1 | `b` |
| `0xF1`..`0xF8` | 2 | `240 + 256 * (b - 0xF1) + next` |
| `0xF9` | 3 | `0x8F0 + next + 256 * next2` |
| `0xFA`..`0xFF` | `1 + (b - 0xF7)` (3 to 8) | the following `b - 0xF7` bytes as a little-endian number |

## SyncData content

In order:

- **buildings[]** and **remains[]**: each entry has `typeId`, `x`, `y`, `orient`, `mirror`, `health`, a nullable disaster (a flag and two 64-bit payloads) and `turn`.
- **groundTiles**: run-length encoded.
- **natureTiles**: run-length encoded, `-1` means no nature object.
- **innerCliffs[]**: a count, then seven packed integers per entry.

The grid side is the square root of the ground tile count (49 by default).

## Building footprints

The shape of a building is `BuildingType.localPositions` (an `int2[]`). `orient` rotates it in steps of 90 degrees and `mirror` flips the x axis:

```text
orient 0..3 -> (cos, sin) = (1,0) (0,1) (-1,0) (0,-1)
x' = (dx * cos - dy * sin) * scaleX        scaleX = -1 when mirrored, otherwise 1
y' =  dx * sin + dy * cos
world position = building position + (x', y')
```

## Ground and nature ids

| Nature id | Object |
|---|---|
| 0 | Tree |
| 1 | Rock |
| 2 | Scorched earth |
| 3 | Lava |
| 4 | Flower |
| 5 | Fish |

Ground tile ids seen in the disaster code: 1 is skipped by a meteorite, 4 has inner cliffs that a meteorite removes, and 5 is scorched earth (what a meteorite leaves).

## Building types (version 1.0.4)

The ids are the indices in `BuildingTypes.Types`, in save-file order.

| Id | Building | Id | Building |
|---|---|---|---|
| 0 | Archery Range | 12 | Lumber Camp |
| 1 | Barracks | 13 | Market |
| 2 | Castle | 14 | Sorcerer Tower |
| 3 | Cathedral | 15 | Stone Quarry |
| 4 | Church | 16 | Storage |
| 5 | Cottage | 17 | University |
| 6 | Dock | 18 | Wall |
| 7 | Farm | 19 | Windmill |
| 8 | Guard Tower | 20 | Wonder |
| 9 | Guard Tower Up | 21 | Fishing Ship |
| 10 | Hospital | 22 | Lighthouse |
| 11 | House | 23 | Land Camp |

Docks, Fishing Ships and Lighthouses may stand on water (`localPositionsInWater`). Maximum health examples: Castle 200, Wonder 150, Dock 50, House 15.
A castle placed on the very edge of the island does not appear.
