---
summary: Step by step guide to adding a civilization to Dice Kingdoms with ModCore - registering it, its tech tree, models, icons and name - with the full list of assets a civilization can have
---

# Making a civilization

A civilization in Dice Kingdoms is a number plus data the game looks up by that number. The game only knows five (`EUROPE`, `MIDDLE_EAST`, `VIKING`, `ANCIENT`, `MAYA`) and throws or freezes for any other,
so the library [ModCore](modcore.md) makes the game accept more. This page is the practical guide; [Civilizations in the game code](../../game-architecture/civilizations.md) explains why each part exists.

!!! note "What is tested"
    **Tested in a two-player match:** registering, choosing it by clicking the castle (the host only offers it when everybody has the mod), the lobby and the match start, an AI playing it, the research
    window with its tech tree, the look of the borrowed built-in civilization.
    **Tested in a solo lobby and match:** own models through `civ.Skins` (castle, house, soldier and ship borrowed from the Maya civilization showed up for a civilization that otherwise
    looks like Viking) and the Rules screen with the civilization chosen.
    **Tested in a solo match:** `CloneTree` - a copy of the Viking tree with one slot moved was shown in the research window of the civilization (the provider ran once, the copy was reused).
    **Written against the game code but not yet run:** the localized name (the game has no screen that shows a civilization's name in the lobby, so it is only a safety net).
    If it misbehaves, the log line starting with `[modcore]` says which.

## What you get without any work

```csharp
var civ = ModCore.Civilizations.Add("authorname.mycivilization", "My civilization", skinsLike: 2);
```

This is a complete, playable civilization: it can be chosen on the match-start screen by every player who has the mod, it is synchronised in multiplayer, AIs can play it, and its tech tree, building
models, units and ground are those of the built-in civilization in `skinsLike` (0 Europe, 1 Middle East, 2 Viking, 3 Ancient, 4 Maya; the biome follows it: Plains, Desert, Snow, Mediterranean, Jungle).
Everything below replaces one part of that borrowed look.

| Parameter | |
|---|---|
| `key` | Unique and stable, prefixed with your name. It is hashed into the number, so it must never change after release (saves and lobbies use it). |
| `name` | Display name, written to the game's text table as `civilization.<id>.name` (new). |
| `techTree` | Function returning the tree; `null` (or returning `null`) uses the tree of `skinsLike`. |
| `skinsLike` | The built-in civilization to borrow biome, models, AI and tree from. |

The number is `civ.Id`, **5 to 127** (the game sends it as one byte): 123 civilizations can exist at once across all mods; a collision is resolved by moving the later mod to the next free number
and logging a warning.

## 1. The tech tree

The 22 technologies are shared; a tree picks some of them and arranges them (a slot is a technology, a grid position and links to the technologies it unlocks; a technology is learnable when it has no
incoming link or one of the technologies linking to it is learnt). To make a tree, start from a copy of a built-in one:

```csharp
TechTree cached = null;
Func<TechTree> tree = () => {
    if (cached != null) return cached;                                    // the game asks on every lookup
    var t = ModCore.Civilizations.CloneTree(fromCivilization: 2);        // copy of the Viking tree
    // change t.Slots: move a slot (copy it out, change position, write it back), change links, change which technology sits in a slot
    return cached = t;
};
var civ = ModCore.Civilizations.Add("authorname.mycivilization", "My civilization", tree, skinsLike: 2);
```

The provider is called every time the game needs the tree (many times a second while the research window is open), so create the tree once and return the same object. The technologies and their
costs are shared by every civilization and cannot differ per tree. Slot names, positions and links of every built-in tree are in the file the dump tool writes (below).

## 2. Models (new)

The game finds the model of everything through the skin registry, keyed by the **default prefab** and the civilization. `civ.Skins` returns your prefab instead, and anything you do not set keeps the
borrowed look, so you can replace one building at a time:

```csharp
civ.Skins
   .Building(BuildingTypeName.HOUSE, myHouse)       // BuildingVisual
   .Building(BuildingTypeName.CASTLE, myCastle)
   .Unit(6, mySoldier)                        // UnitVisual, 6 = SOLDIER (numbers of Unit.Type)
   .Nature(NatureTile.TREE, myTrees)                // NatureVisualHub
   .Ship(myShip);                                   // ShipVisual
```

A prefab is a game object carrying the game's own component (`BuildingVisual`, `UnitVisual`, ...) with its references filled in: the model objects, the materials that take the player colour, sounds,
particles, the icons. The game's scripts cannot be added in an ordinary Unity project, so build yours **from a copy of the default prefab** at runtime:

1. Find the default prefab: `Resources.FindObjectsOfTypeAll<BuildingVisual>()` and pick it by name (the dump tool lists the names), or take `BuildingType.visualPrefab` for a building.
2. `Object.Instantiate` it under an inactive parent object that you keep (`DontDestroyOnLoad`), so it stays a template and nothing is drawn.
3. Replace what differs: the meshes and materials of the objects the component's `visual` and `damagedVisual` fields point to (a house in two states: whole and damaged), keeping the material
   setup that takes the player colour (`buildingTints`), the `firesHolder` positions, the particles and sounds if you do not need different ones.
4. Load your own meshes, textures and sprites from an **asset bundle** built with the same Unity version as the game (`2021.3.36f1`) or from files (`Texture2D.LoadImage`, `Sprite.Create`).

Rules the components follow that a replacement has to keep: a building covers the cells listed in its `BuildingType` (`localPositions`, not your prefab's size); a unit's animator needs the
parameters Move, Attack, AttackSpeed and Run (a ship's: Deploy, Sand, Move); a `NatureVisualHub` is a parent whose `NatureVisual` children are alternatives the game picks from at random.

## 3. Icons (new)

`BuildingVisual.Icon(PlayerColor)` and `UnitVisual.Icon(PlayerColor)` return `icons[color.Index]`: **one sprite per player colour** (the array's length is the number of colours the game has,
`PlayerColors.Colors.Length`). These sprites appear in the build bar, the research window (the "unlock building" technologies show the building from the player's skin) and the unit lists. When you replace a model
and want a matching icon, fill the `icons` array of your prefab copy with one sprite per colour; otherwise the copy keeps the default prefab's icons. The other technologies have one fixed sprite each, the same
for every civilization.

## 4. The name

`ITextElement.Localized` writes the name from the key `civilization.<id>.name`. ModCore adds it to the text table of the language in use for you (new); for several languages, register your text with the
game's own localization after your civilization is added. Without this the name shows as a missing translation.

## 5. AI

AIs play from build orders embedded in the game (3 per built-in civilization), written for exactly one civilization, so a mod civilization's AI uses those of `skinsLike`. A new build order would
need a game update; there is no hook for it.

## What cannot be changed

- **The biome** (ground, trees, rocks, cliffs' base look) comes from the built-in civilization in `skinsLike`; there is no sixth biome.
- **Technology costs and effects**, building costs, unit stats: the same for everybody.
- **Unlocking:** the Steam inventory decides which built-in civilizations a player has (`UnlockedCivilizations`). ModCore adds your civilization after them in the castle-click cycle and never touches the inventory.
- **Number of civilizations:** 5..127 (see above).
- **Everybody in the match needs your mod.** A player without it cannot be sent the number; the host's castle click only offers civilizations everybody has, and the host moves a player off a civilization that
  somebody lacks.

## Everything a civilization can have

| Asset | Count | Type | If you do not provide it |
|---|---|---|---|
| Building models | 24 (`BuildingTypeName`: ARCHERY_RANGE, BARRACKS, CASTLE, CATHEDRAL, CHURCH, COTTAGE, DOCK, FARM, GUARD_TOWER, GUARD_TOWER_UP, HOSPITAL, HOUSE, LUMBER_CAMP, MARKET, SORCERER_TOWER, STONE_QUARRY, STORAGE, UNIVERSITY, WALL, WINDMILL, WONDER, FISHING_SHIP, LIGHTHOUSE, LAND_CAMP) | `BuildingVisual` prefab | look of `skinsLike` |
| Unit models | 10 (ARCHER, BOURGEOIS, GENERAL, KING, PEASANT, PRIEST, SOLDIER, SORCERER, TEMPLAR, WARLOCK) | `UnitVisual` prefab | look of `skinsLike` |
| Ship | 1 | `ShipVisual` prefab | look of `skinsLike` |
| Nature | 6 (TREE, ROCK, SCORCHED_EARTH, LAVA, FLOWER, FISH) | `NatureVisualHub` prefab | look of `skinsLike` |
| Cliffs | keyed by `InnerCliff.Key` | `InnerCliffVisualHub` prefab | look of `skinsLike` (no hook in `civ.Skins` yet) |
| Building and unit icons | one per player colour per model | `Sprite[]` in the prefab | the prefab's own |
| Tech tree | 1 | `TechTree` asset | tree of `skinsLike` |
| Name | 1 per language | text table entry | missing-translation text |
| AI build orders | - | embedded text | those of `skinsLike` |

## Finding the game's own prefabs and sprites

Set `[Debug] DumpGameData = true` in `BepInEx/config/DiceModders.ModCore.cfg`, start the game and wait in the main menu. ModCore writes two files next to `LogOutput.log`:

- `BepInEx/ModCore-game-data.txt`: the 22 technologies with their classes and costs, every tech tree (slots, positions, links), every building type (tier, size, health, cost, default prefab), the number of skin registry entries (the entries themselves cannot be read safely), the default prefabs, every prefab of the visual types the game has loaded, and the civilization name keys.
- `BepInEx/ModCore-sprites.txt`: every sprite with the texture (atlas) it is on and its size.

Switch it off again afterwards; it is a developer tool. The prefab names it lists are the models to follow (a Viking house is `House Viking`); the full lists of models and sprites are in [Civilization models and sprites](../../game-architecture/civilization-assets.md).
