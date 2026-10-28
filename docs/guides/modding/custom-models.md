---
summary: Showing your own 3D models in the game - extracting the original models, the palette, Blender to Unity axes, the castle prefab structure, replacing a mesh at runtime, flicker and missing faces
---

# Custom models

This page shows how to replace the look of a building with your own model at runtime (tested in game with the Castle). It only changes what the local player sees: footprint,
rules and networking stay the original, and other players still see the original model. To give a whole civilization its own models through the game's skin system, see
[Making a civilization](making-a-civilization.md#3-models).

## Getting the original models

- **AssetRipper** exports the game's assets (game version 1.0.4 uses Unity **2021.3.36f1**) as plain files and as a Unity project. Its free glTF export gives rigged units as skeletons only.
- For rigged and animated models (units, inhabitants, ships, buildings with moving parts): open the exported project in Unity 2021.3.36f1 and export FBX with the FBX Exporter
  (include model and animation). The FBX SDK crashes now and then inside the export call; exporting in small batches with a restart is reliable.
- Unit models have the takes idle, walk, run and attack; ships move and extend / retract the plank; the Windmill has its sails.
- In Blender, import FBX with "Ignore Leaf Bones"; each take becomes an Action.

## The palette

All game models use one material with an **8×8 palette texture** (`ColorPalette.png`). A face's colour is the cell its UVs point to; put every vertex of a face inside one cell, near its centre,
and use point filtering in Blender. Cell **(7, 7)** is repainted with the player colour (see [Rendering](../../game-architecture/rendering.md#the-palette-colored-paper)).

Tested observations:

- (0, 3) renders brown/orange; (0, 4) ("sand") renders clearly **yellow** in game lighting.
- Good ground greens: (5, 3) and (5, 4).
- The in-game light is brighter and warmer than Blender's preview, so check colours in the game.
- The original roofs and trim use (3, 4); lit windows (5, 0) orange and (5, 1) yellow (the only emissive cells); metal door fittings (3, 6).

## Blender → Unity axes

Calibrated with the Castle's flag:

- Prefab-local = Blender root-local **/ 100**, with **Unity x = −Blender x, Unity y = Blender y, Unity z = Blender z** (in the FBX root space, which is already Y-up).
- That map is a mirror (determinant −1), so **reverse the triangle order** when you write the indices, otherwise the faces turned toward the camera are missing in game.
- In Blender (after importing a game FBX): east = −X, north = −Y, west = +X, south = +Y. The game camera sees orientation 0 from Blender's +X +Y corner, so a gate meant to face the camera at the
  default orientation goes on **+X** (like the Viking and Ancient castles). See [Rendering](../../game-architecture/rendering.md#building-orientation-and-the-camera).
- Keep the model inside the building's tiles (a Castle is 4×4: tiles −1..2).

## What a building looks like at runtime

Taking the Castle as the example:

- Buildings are `BuildingVisual` objects. The prefab root is `Castle <civ>`; pooled copies are named like `Castle Viking <49> #0001`, `Castle Europe (1)` or `...(Clone)`. Cut the name at the first
  ` <`, ` (` or `(` to get the civilization (Europe, Middle East, Viking, Ancient, Maya).
- The body mesh is `castle-<civ>-mesh_LOD0/1/2` (MeshFilters under a `LODGroup`), present **twice**: under `Visual` and under `Damaged Visual`. Replace both, or the destroyed state shows the old model.
- **Castle Viking contains two LOD groups** (`castle-europe-mesh_LOD*` and `castle-viking-mesh_LOD*`). Filling both draws the model twice and the copies flicker. Fill only the civilization's own
  group (fallback `castle-europe`) and give the others an empty mesh.
- The team flag is a separate MeshFilter `flag-big` with its pivot at the foot of the pole (Europe: local (0.385, 2.38, 0.545)). Move it with `localPosition` / `localRotation` onto your model.
- The King is a `SkinnedMeshRenderer` under `Inhabitant King <civ>`.

## Replacing the mesh

- Hook: a Harmony **postfix on `BuildingVisual.Apply`** runs every time a building is built or refreshed. Replacing the mesh there is enough; doing it again is harmless.
- Build the `Mesh` once and reuse it: vertices, UVs (palette cells), triangles, normals. Use `IndexFormat.UInt32` above 65 535 vertices and set `hideFlags = HideFlags.DontUnloadUnusedAsset`
  so a scene change does not unload it.
- **`Mesh.RecalculateNormals()` is stripped** from this build and throws. Compute the normals yourself (cross product per triangle, summed per vertex, normalized). The bounds update when
  you set the vertices.
- Setting `MeshFilter.sharedMesh` keeps the game's materials (player colour, tints, burnt look).

## Flicker and missing faces

- The game camera is far away, so depth precision is low: faces that are coplanar or nearly touching flicker much more than in Blender.
  - In the model: never let two parts share a face exactly (make a footing 1–2 cm wider than the wall on it; do not end two walls in the same plane).
  - At export: push each connected part out along its normals by a tiny amount that is larger for small parts (details on top of big parts), for example
    `0.0015 × (1 + 4 × (1 − size/4)²)`.
- Missing faces: check the winding (above). Writing every triangle twice (both windings) hides parts whose normals are flipped in Blender.
- A mesh cleanup that removes "back-to-back face pairs" also deletes deliberately double-sided thin parts; build thin walls as solid wedges or boxes instead.

## Limits

- Only the local player sees the model; footprint, flag logic, fire / smoke anchors and the inhabitant's walking area stay those of the original prefab.
- One mesh for all LOD levels is fine at building size.
- The destroyed state uses the same model (sunk and tinted); a separate damaged model needs a second mesh.
