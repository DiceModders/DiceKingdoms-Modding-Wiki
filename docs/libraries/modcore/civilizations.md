---
summary: ModCore civilizations API - adding a civilization, its own tech tree, names and texts per civilization, own models and island colours, reskinning or replacing the tree of a built-in civilization, AI build orders
---

# Civilizations

## Add a civilization

`Add` registers a civilization the game accepts everywhere it uses the number (tech tree, skins, network, castle click, AI). The key is hashed into an id (5..127); when
another civilization on the same computer already has it, the next free one is used. `skinsLike` is the built-in civilization whose biome and looks are used for everything you
do not replace (0 Europe, 1 Middle East, 2 Viking, 3 Ancient, 4 Maya). Players can only pick it when every lobby member has the mod (same version).

The id is local: another computer with other civilizations installed may give the same civilization another number. In a lobby the host publishes a table
(lobby data `modcore.civtable`, key = number) and ModCore translates the player skins the game sends (local number -> host number -> the receiver's number), so you
never see a foreign number. The table only grows and is frozen when the islands spawn. If your mod puts a civilization number into data other players read (a lobby rule,
a message), convert it with `Core.Civilizations.ToLobbyNumber(civ.Id)` and back with `FromLobbyNumber(n)` (-1: not listed / not installed here).

```csharp
GameTechTree myTree = null;
var civ = Core.Civilizations.Add("author.my-civ", "My Civ", () => myTree, skinsLike: 2);

Core.Civilizations.WhenGameDataLoaded(() => {
    myTree = Core.Civilizations.CloneTree(2);   // a copy of the Viking tree to edit (slots, positions, unlocks)
});
```

| Member | Meaning |
|---|---|
| `civ.Id`, `civ.Key`, `civ.Name` | The number the game uses, your key, the display name |
| `civ.SkinsLike` | The built-in look-alike |
| `civ.Skins` | Own models, pictures and colours (below) |
| `civ.Remove()` | Takes it out again |
| `Civilizations.All`, `Added`, `Removed` | For tools and screens |

## Names and texts

Every text the game draws from a localization key can be replaced while the civilization being looked at is yours (the one picked in an open Rules window, otherwise the local
player's). Works for built-in and mod civilizations, with translations by language code. Visual and local: nothing is sent to other players.

```csharp
var de = new Dictionary<string, string> { ["de"] = "Fußsoldat" };
Core.Civilizations.RenameUnit(civ.Id, Unit.Type.SOLDIER, "Footman", de);
Core.Civilizations.SetUnitLore(civ.Id, Unit.Type.SOLDIER, "Cheap and many.");
Core.Civilizations.RenameBuilding(civ.Id, "WINDMILL", "Water Mill");
Core.Civilizations.SetText(civ.Id, "technology.TEMPLARS.description", "Trains knights instead of templars.");
Core.Civilizations.Rename(2, "Norse");   // a built-in civilization's own name
```

Keys are the game's localization keys: `unit.SOLDIER.name`, `unit.SOLDIER.lore`, `building.WINDMILL.name`, `technology.TEMPLARS.name` / `.description`. Pass `null` to give the
game's text back.

## Own models and pictures

On the first lookup ModCore copies the look-alike's prefab once, puts your mesh in, and the game uses the copy everywhere (match, pools, burnt state, lobby card, build menu,
Rules screen). Footprint and rules stay those of the original building. Mesh UVs point at the game's 8x8 colour palette; pictures are one per player colour (12, 237x235 px).

```csharp
Mesh castle   = Tools.Meshes.LoadDkmesh(@"models\castle.dkmesh", out Vector3? flag, out float yaw);
Sprite[] icons = Tools.Sprites.LoadPlayerColorSet(@"icons\castle");   // 00.png .. 11.png
civ.Skins.BuildingModel(BuildingTypeName.CASTLE, castle, flag, yaw, icons);
```

No pictures of your own? ModCore can take them in the game: one per player colour, coloured the way the game colours the model, from the angle of the game's
pictures. They are rendered once after the game data has loaded (one colour per frame) and cached, so later launches read one file.

```csharp
Core.Civilizations.RenderIcons(civ.Id);                                    // every own building without pictures
Core.Civilizations.RenderIcons(civ.Id, null,
    (type, sprites) => Log.LogInfo(type + ": " + sprites.Length + " pictures"),
    BuildingTypeName.HOUSE, BuildingTypeName.WINDMILL);                     // only these, and get the sprites (e.g. for TextSprites)
```

For full control, copy the game's own prefabs and hand the copies over (units, inhabitants, nature, ships too):

```csharp
Core.Civilizations.WhenGameDataLoaded(() => {
    var look = new PlayerSkins((Civilization)2, Resources.FindObjectsOfTypeAll<SkinRegistry>()[0]);
    foreach (var type in Resources.FindObjectsOfTypeAll<BuildingTypes>()[0].Types) {
        var copy = Tools.Prefabs.Copy(look.GetBuildingVisualPrefab(type));   // inactive, persistent, original untouched
        // change meshes, materials, icons on the copy here
        civ.Skins.Building(type.name, copy);
    }
});
```

Island colours are 8x8 palette cells (x from the left, y from the bottom), changed on that civilization's islands only:

```csharp
civ.Skins.GroundColour(4, 3, new Color(0.45f, 0.62f, 0.30f))   // grass
         .GroundColour(4, 4, new Color(0.40f, 0.55f, 0.27f))
         .NatureColour(2, 2, new Color(0.30f, 0.50f, 0.25f));  // trees
```

## Change a built-in civilization

`Reskin` gives a built-in civilization new looks (the same `CivilizationSkins` as above). It is visual and local, so players with and without the mod play together, and anything
you leave out stays the game's original. `ReplaceTree` changes gameplay, so it is guarded by a lobby rule: the new tree is used only in a local game, or when the host's rule is
on and every member has the mod; otherwise everybody plays the original tree. The decision is frozen at match start.

```csharp
var viking = Core.Civilizations.Reskin(2);
viking.Unit(6, mySoldierPrefab).GroundColour(4, 3, grass);

GameTechTree vikingTree = null;
Core.Civilizations.WhenGameDataLoaded(() => vikingTree = EditTree(Core.Civilizations.CloneTree(2)));
Core.Civilizations.ReplaceTree(2, () => vikingTree, "author.viking-tree", "New Viking tech tree");
```

## AI build orders

AI players pick a build order (technologies to research, buildings to place) for their civilization and strategy. Give your civilization its own in the game's format; the
`civilization=` value is replaced with yours. Each block is checked first (weight above 0, known strategy, technology and building names), broken ones are left out with a
log line, and technologies are arranged to fit the civilization's tech tree (each one after a prerequisite; ones the tree lacks are dropped).

```csharp
Core.Civilizations.AddBuildOrders(civ.Id, @"
civilization=ANY weight=2 strategy=MILITARY name=rush [
    technologies [ UNLOCK_STORAGE UNLOCK_ARCHERY_RANGE UNLOCK_GUARD_TOWER TEMPLARS ]
    buildings [ house cot house farm racks racks archery storage guard racks ]
]
civilization=ANY weight=1 strategy=WONDER name=culture [
    technologies [ UNLOCK_STORAGE UNLOCK_CATHEDRAL UNLOCK_WONDER ]
    buildings [ house cot church church farm cat storage wonder wonder wonder ]
]");
```

- A strategy your civilization has no order for uses the look-alike's orders, rearranged to your tree, so the AI still plays your civilization's tree.
- Orders for a built-in civilization (0..4) are added next to the game's.
- Strategies: `MILITARY`, `WONDER`. Building names: see [AI build orders](../../game-architecture/ai.md#build-orders).
- Calling again replaces your earlier orders for that civilization; `RemoveBuildOrders(civ.Id)` removes them.

## Changing looks during a match

Models are made when a building is placed, so a change to a civilization's looks (for example with `Reskin`) only shows on new buildings. `RefreshLooks` rebuilds the building models of every island (or of one player's island) from the current looks at once, without the building animation. Units, ships and nature take their models when they are made, so they follow the new looks from then on. It only changes what this computer draws.

```csharp
Core.Civilizations.Reskin(2).Building(GameBuildingTypeName.CASTLE, europeCastlePrefab);   // Viking castles look like Europe ones
Core.Civilizations.RefreshLooks();                                                     // every island, now
Core.Civilizations.Reskin(2).Building(GameBuildingTypeName.CASTLE, null);               // null: the game's own model again
Core.Civilizations.RefreshLooks(Core.Players.Local);                                   // only your island
```
