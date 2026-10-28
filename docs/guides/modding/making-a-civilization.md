---
summary: Step by step guide to adding a civilization to Dice Kingdoms - which game code a mod has to change, its tech tree, models, icons and name - with the full list of assets a civilization can have
---

# Making a civilization

A civilization in Dice Kingdoms is a number plus data the game looks up by that number. The game only knows five (`EUROPE`, `MIDDLE_EAST`, `VIKING`, `ANCIENT`, `MAYA`) and throws or freezes
for any other number, so a mod has to change a few places in the game code first. This page is the practical guide; [Civilizations in the game code](../../game-architecture/civilizations.md)
explains why each part exists. Everything here is for game version 1.0.4.

## 1. Pick a number and make the game accept it

The civilization travels over the network as one signed byte and 0..4 are taken, so a mod civilization needs a number from **5 to 127**. Choose it from a stable key (for example a hash of
`authorname.mycivilization`) so it is the same on every machine and in every version of your mod. Then change these places (Harmony prefixes/postfixes; the full list with the exact behaviour
is the table in [Civilizations in the game code](../../game-architecture/civilizations.md#where-the-number-is-used)):

| Place | What to do |
|---|---|
| `PlayerSkins` constructor, `Civilization` setter, `PlayerSkins.Clone()` | They derive the biome with a switch over 0..4 and throw for anything else. Build the skins for the built-in civilization yours is based on, then store your number in the `PlayerSkins.civilization` field. |
| `PlayerSkinsSerialization` (network reader) | Receives the byte and calls the constructor, so it needs the same treatment, or the receiving game freezes. |
| `Technologies.TechTree(civilization)` | Return your tree (or the tree of the civilization yours is based on). It was inlined into `Statistics.get_TechTree` and `Statistics.IsTechnologyLearnable`, so patch those two as well. |
| `AIPlay.SampleBuildOrder` | Use the build orders of the civilization yours is based on (AI build orders are compiled into the game). |
| `Player.NextCivilization` | The castle click. A prefix can step through your civilizations after the last unlocked one, then let the game wrap around. Do not replace or patch the player's inventory (see [Rules and civilizations](../../game-architecture/rules-and-civilizations.md)). |
| Rules screen drop-down | Append an option and a boxed enum value (`Enum.ToObject`) so the tech tree tab can show your civilization. |

A sketch of the tech tree patch:

```csharp
[HarmonyPatch(typeof(Technologies), nameof(Technologies.TechTree))]
static class TechTreePatch
{
    static bool Prefix(Civilization civilization, ref TechTree __result)
    {
        if ((int)civilization < 5) return true;      // built-in: let the game answer
        __result = MyMod.Tree;                       // your tree, created once
        return false;
    }
}
```

**Everybody in the match needs your mod.** A player without it cannot read the number. Before a mod civilization is picked, make sure every lobby member has it (for example by publishing your
civilizations in the Steam lobby member data and only offering ones everybody has).

## 2. The tech tree

The 22 technologies are shared; a tree picks some of them and arranges them (a slot is a technology, a grid position and links to the technologies it unlocks; a technology is learnable when it has no
incoming link or one of the technologies linking to it is learnt). The easiest start is a copy of a built-in tree: take it from `Technologies.TechTree(<built-in civilization>)`, copy the asset
(`Object.Instantiate`) and change the copy.

The grid of the built-in trees uses even x values 0, 2, 4, 6, 8 (the odd ones are the link ports) and y values 0 to 4. Two slots on the same cell are drawn on top of each other, so move a slot to a
free cell. `Slots` is an array of structs: copy a slot out, change its `position`, write it back (`tree.Slots[i] = slot`).

The game asks for the tree very often (many times a second while the research window is open), so create it once and return the same object. The technologies and their costs are shared by every
civilization and cannot differ per tree.

## 3. Models

The game finds the model of everything through the skin registry, keyed by the **default prefab** and the civilization (`SkinRegistry.GetSkinVariant`; it returns the default when it has no entry).
A postfix there can return your prefab for your civilization, and anything you do not replace keeps the look of the civilization yours is based on, so you can replace one building at a time.
Results are cached per `PlayerSkins` object.

A prefab is a game object carrying the game's own component (`BuildingVisual`, `UnitVisual`, ...) with its references filled in: the model objects, the materials that take the player colour, sounds,
particles, the icons. The game's scripts cannot be added in an ordinary Unity project, so build yours **from a copy of the default prefab** at runtime:

1. Find the default prefab: `Resources.FindObjectsOfTypeAll<BuildingVisual>()` and pick it by name ([Civilization models and sprites](../../game-architecture/civilization-assets.md) lists the names), or take `BuildingType.visualPrefab` for a building.
2. `Object.Instantiate` it under an inactive parent object that you keep (`DontDestroyOnLoad`), so it stays a template and nothing is drawn.
3. Replace what differs: the meshes and materials of the objects the component's `visual` and `damagedVisual` fields point to (a building in two states: whole and damaged), keeping the material
   setup that takes the player colour (`buildingTints`), the `firesHolder` positions, the particles and sounds if you do not need different ones.
4. Load your own meshes, textures and sprites from an **asset bundle** built with the same Unity version as the game (`2021.3.36f1`) or from files (`Texture2D.LoadImage`, `Sprite.Create`).

Rules the components follow that a replacement has to keep: a building covers the cells listed in its `BuildingType` (`localPositions`, not your prefab's size); a unit's animator needs the
parameters Move, Attack, AttackSpeed and Run (a ship's: Deploy, Sand, Move); a `NatureVisualHub` is a parent whose `NatureVisual` children are alternatives the game picks from at random.

## 4. Icons

`BuildingVisual.Icon(PlayerColor)` and `UnitVisual.Icon(PlayerColor)` return `icons[color.Index]`: **one sprite per player colour** (the array's length is the number of colours the game has,
`PlayerColors.Colors.Length`). These sprites appear in the build bar, the research window (the "unlock building" technologies show the building from the player's skin) and the unit lists. When you replace a model
and want a matching icon, fill the `icons` array of your prefab copy with one sprite per colour; otherwise the copy keeps the default prefab's icons. The other technologies have one fixed sprite each, the same
for every civilization.

## 5. The name

`ITextElement.Localized` writes the name from the key `civilization.<number>.name` (a boxed enum without a name prints its number). Add that entry to the string table of the language in use,
and again when the language changes (each language's table is created when it loads). Without it the name shows as a missing translation.

## 6. AI

AIs play from build orders embedded in the game (3 per built-in civilization), written for exactly one civilization, so a mod civilization's AI uses those of the civilization it is based on.
New build orders would need a game update; there is no hook for adding them.

## What cannot be changed

- **The biome** (ground, trees, rocks, cliffs' base look) comes from the built-in civilization yours is based on; there is no sixth biome.
- **Technology costs and effects, building costs and stats, unit stats**: the same for everybody. Only the models (and the tech tree) differ per civilization.
- **Unlocking:** the Steam inventory decides which built-in civilizations a player has (`UnlockedCivilizations`).
- **Number of civilizations:** 5..127 (see above).

## Everything a civilization can have

| Asset | Count | Type | If you do not provide it |
|---|---|---|---|
| Building models | 24 (`BuildingTypeName`: ARCHERY_RANGE, BARRACKS, CASTLE, CATHEDRAL, CHURCH, COTTAGE, DOCK, FARM, GUARD_TOWER, GUARD_TOWER_UP, HOSPITAL, HOUSE, LUMBER_CAMP, MARKET, SORCERER_TOWER, STONE_QUARRY, STORAGE, UNIVERSITY, WALL, WINDMILL, WONDER, FISHING_SHIP, LIGHTHOUSE, LAND_CAMP) | `BuildingVisual` prefab | look of the base civilization |
| Unit models | 10 (ARCHER, BOURGEOIS, GENERAL, KING, PEASANT, PRIEST, SOLDIER, SORCERER, TEMPLAR, WARLOCK) | `UnitVisual` prefab | look of the base civilization |
| Ship | 1 | `ShipVisual` prefab | look of the base civilization |
| Nature | 6 (TREE, ROCK, SCORCHED_EARTH, LAVA, FLOWER, FISH) | `NatureVisualHub` prefab | look of the base civilization |
| Cliffs | keyed by `InnerCliff.Key` | `InnerCliffVisualHub` prefab | look of the base civilization |
| Building and unit icons | one per player colour per model | `Sprite[]` in the prefab | the prefab's own |
| Tech tree | 1 | `TechTree` asset | tree of the base civilization |
| Name | 1 per language | text table entry | missing-translation text |
| AI build orders | - | embedded text | those of the base civilization |

## Finding the game's own prefabs and sprites

The loaded prefabs of each visual type can be listed from a mod with `Resources.FindObjectsOfTypeAll<T>()` (`BuildingVisual`, `UnitVisual`, `ShipVisual`, `NatureVisualHub`,
`InnerCliffVisualHub`, `Sprite`) once the main menu is loaded. A Viking house is `House Viking`; the full lists of models and sprites are in
[Civilization models and sprites](../../game-architecture/civilization-assets.md).
