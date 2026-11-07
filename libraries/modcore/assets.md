---
summary: ModCore models, pictures and caches - loading meshes, sprites and player-colour icon sets, prefab copies, pictures inside game texts, GPU-compressed texture and icon atlas caches, the shared disk cache
---

# Models, pictures and caches

All helpers are in `DiceModders.ModCore.Tools` and work without `ModCore.Init()`.

## Meshes

`Create` builds a mesh from arrays (normals are calculated). `LoadDkmesh` reads the plain-text `.dkmesh` format: `v x y z u v` (vertex with a UV on the game's 8x8 palette),
`t a b c` (triangle), optional `flag x y z yaw` (where a building's flag stands). The parsed result is kept in the disk cache, so later launches skip the parsing; call `Prefetch`
early so the cached files are read in the background.

```csharp
string[] files = Directory.GetFiles(modelFolder, "*.dkmesh");
Meshes.Prefetch(files);                                                           // in Load()

Mesh house = Meshes.LoadDkmesh(Path.Combine(modelFolder, "house.dkmesh"), out Vector3? flag, out float yaw);
Mesh quad  = Meshes.Create("quad", vertices, uvs, triangles);
Mesh none  = Meshes.Empty;                                                         // blank out a part of a model
```

A model embedded in your DLL is read from a stream; with your mod's version it is cached like a file:

```csharp
var asm = typeof(MyModPlugin).Assembly;
Mesh tower = Meshes.LoadDkmesh("tower", () => asm.GetManifestResourceStream("MyMod.models.tower.dkmesh"), "1.0.0", out var f, out var y);
```

## Sprites and prefab copies

```csharp
Sprite logo        = Sprites.Load(@"icons\logo.png", pivot: new Vector2(0.5f, 0.5f));
Texture2D readable = Sprites.LoadTexture(@"icons\big.png", readable: true);
Sprite[] perColour = Sprites.LoadPlayerColorSet(@"icons\castle");   // 00.png .. 11.png, one per player colour

GameObject copy = Prefabs.Copy(gamePrefab);                         // inactive, persistent, same name; change it freely
```

## Pictures inside game texts

The game writes `<sprite name="...">` in texts (tooltips, Rules screen) with the name of the icon it shows. `AddLike` adds a sprite sheet with your pictures as a fallback of the
game's "Icons Text" sheet. Each entry is your sprite name and the game sprite to copy the size from; pass a loader so the picture is only decoded when the sheet has to be drawn.
With a cache key the finished sheet is cached, and the pictures are not loaded at all on later launches.

```csharp
string gameSprite = "<name of a game icon sprite of the same kind>";   // listed in Civilization Models and Sprites
var entries = new List<TextSprites.Entry> {
    new("building-icon-castle-mycivname-red", gameSprite, () => Sprites.LoadTexture(@"icons\castle\00.png", true)),
};
TextSprites.AddLike("My Civ Text", entries, cacheKey: "v1");
```

## Texture cache and icon atlas

Decoding many PNGs takes time on every launch. `IconAtlas.Build` packs pictures into 4096x4096 pages with mipmaps (soft edges at any size), GPU-compressed (DXT5), and keeps them in
the cache keyed by the files' paths, sizes and change times: the first launch builds it, later launches read one file. Change a PNG and it is rebuilt on the next launch.

```csharp
TextureCache.Prefetch("My Civ icons");                                    // in Load(): read in the background
var atlas = IconAtlas.Build("My Civ icons", pngPaths, 256, 256);          // cell size
Sprite castleRed = atlas.Sprite(0, new Vector2(0.5f, 0.5f), "building-icon-castle-mycivname-red");
bool rebuilt = atlas.Rebuilt;                                             // true when the PNGs were read this launch
```

`TextureCache.Save(name, key, textures)` / `Load(name, key)` do the same for any textures you build yourself.

## Pictures rendered in the game

`IconRenderer` photographs any model inside the game: a copy of it on a layer only its own camera sees, once per player colour (the game's own colouring), with a
transparent background (two shots, black and white) and smooth edges (rendered at twice the size). Spread over frames, one colour per frame, and cached by the
model, view, size and game version. `IconView` sets the angle; `IconView.Unit` keeps every unit at the same scale, standing on the bottom of the picture.

```csharp
IconRenderer.Prefetch("author.my-civ archer");                                 // in Load()
Core.Civilizations.WhenGameDataLoaded(() => {
    IconRenderer.PlayerColours("author.my-civ archer", myArcherPrefab.gameObject, IconView.Unit, 256, 256, sprites => {
        // 12 sprites in the game's colour order, e.g. for TextSprites entries
    });
});

var view = new IconView { Pitch = 30f, Yaw = 45f, Padding = 0.1f };            // your own angle
```

Set `[Debug] SaveIconPictures = true` in ModCore's config to get every rendered sheet as a PNG in `BepInEx\cache\ModCore\icon-pictures`.

## Disk cache

Everything above lives in `BepInEx\cache\ModCore\<owner>\`. A file not used for 3 launches is removed, so old versions do not pile up, and deleting the folder is always safe.
Use it for your own cached data:

```csharp
string path = DiskCache.PathFor("author.my-mod", "lookup.bin");   // marks the file as used this launch
DiskCache.Prefetch(path);                                        // start reading it in the background
byte[] data = File.Exists(path) ? DiskCache.ReadAllBytes(path) : BuildAndSave(path);
```
