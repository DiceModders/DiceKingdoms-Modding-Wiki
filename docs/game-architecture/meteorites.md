---
summary: How the Meteorite disaster decides where to hit and when the castle is protected
---

# Meteorites and castle protection

The logic is in `Meteorite.<Execute>d__4.MoveNext` (state 0), decompiled from game version 1.0.4. The `Meteorite` object has these integer fields:

| Field | Meaning |
|---|---|
| `radius` | Radius of the impact in tiles. |
| `minimumCastleTiles` | Minimum number of castle tiles inside the radius for a hit on the castle to be accepted. |
| `meteoriteTilesToFullyAllowHitCastle` | The "castle protection limit" (T below). |

All three are read live for every meteor and are not clamped by the game. A limit such as 1000 in a mod's settings is the mod's own clamp, not the game's.

## Choosing the impact tile

1. `scorched = IslandGrid.CountGround(grid, 5)`: the number of scorched earth tiles (ground tile 5).
2. One random roll per meteor: `roll` is uniform in `[0, T)`, computed as `(uint)(random.state * (int)T) >> 32`.
3. Up to 256 times: pick a random grass tile, count the castle tiles inside the radius (`IslandGrid.MeteoriteNumberCastleTiles`, castle tiles with squared distance at most `radius^2`),
   and **accept the pick** if `castleTiles <= 0`, or if `scorched > roll` and `castleTiles >= minimumCastleTiles`.
4. If all 256 picks were rejected, the last pick is used anyway.

## Consequences

- The castle can be hit with a probability of `min(1, scorched / T)`.
- `T = 0` protects the castle only while there are no scorched tiles (protection for the first meteor).
- A huge `T` (up to `int.MaxValue`) makes a hit on the castle practically impossible.

## What the impact does

`IslandGrid.MeteoriteDisaster`: for tiles within the radius that are inside the island limit (`HardLimit.IsWithin`): skip ground tile 1; remove inner cliffs if the ground is 4;
set the ground to 5 (scorched earth); apply the disaster to or remove the buildings there; add the nature object 3 (lava).
