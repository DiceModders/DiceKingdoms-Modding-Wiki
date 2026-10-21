---
summary: How the Rules screen, civilizations, tech trees and the match scoreboard work in the game code
---

# Rules and civilizations

## Rules screen

The Rules screen (`Rules`) has an index on the left (How to Play, Game Phases, Buildings, Disasters, Units, Technologies) and a scroll view with the content.
The Technologies tab shows the tech tree of the civilization chosen in the civilization drop-down.

- `Rules.Awake` fills the drop-down (`civilizationDropdown`, a `DiceKingdoms.Interface.Dropdown`) from `Enum.GetValues(typeof(Civilization))`. Each entry is a
  tuple of a structured text and the boxed enum value. The wrapper keeps the boxed values in `values` and its `TMP_Dropdown` calls `onSelect(values[index])`.
- `Rules.OnSelectCivilization(object)` unboxes a `Civilization`, sets `PlayerSkins.Civilization` and rebuilds the Technologies tab with `Technologies.TechTree(civilization)`.
- `Technologies.TechTree(Civilization)` is `First(t => t.civilization == civ)` over the game's own list, so it **throws for an unknown civilization**.
- Building cards (`Rules Building`) are 378x256, three to a row.

### Adding a civilization

Every place the game uses a civilization number, and what it does with an unknown one, is listed in [Civilizations in the game code](civilizations.md); the practical guide is
[Making a civilization](../guides/modding/making-a-civilization.md). In short: answer `TechTree(civilization)` for that number (also where the compiler inlined it), let `PlayerSkins` and the
network readers accept it (the number is one signed byte, so 5..127), give AIs a build order, make it selectable (the Rules drop-down and the castle click, below) and provide the
per-civilization data the game looks up (skins, tech tree asset, name). [ModCore](../guides/modding/modcore.md#civilizations) does all of that;
[ModBridge](../guides/modding/modbridge.md) adds the drop-down entry through UiKit.

- `Technologies.TechTree` throws for an unknown civilization (`First` over the game's list). ModCore returns the registered tree, or the tree of the built-in civilization the new one is "like".
- `PlayerSkins.Civilization`'s setter derives the biome with a switch over 0..4 and throws `ArgumentOutOfRangeException` for anything else, and so does the `PlayerSkins` **constructor**
  (the log reads `Parameter name: civilization`). A new civilization must therefore be built for a built-in one first and the real number stored afterwards (`PlayerSkins.civilization` field).

### Choosing a civilization for play (the castle click)

On the match-start screen, clicking a player's castle runs `Player.NextCivilization`. On the host it steps through `Player.Inventory.UnlockedCivilizations` (the civilizations the
player's Steam inventory has unlocked), wraps around, and sets `Player.skins` to `new PlayerSkins(civilization, skinRegistry)`; a client sends `NextCivilizationRPC` to the host, which runs the same code.
It does nothing once the player is ready (`StartGameReady`).

!!! warning "Do not replace or patch the player's inventory"
    `Player.NetworkStart` (host side, when a player is spawned) also picks the **random starting civilization** with `RandomExtensions.NextSample<Civilization>` from the same array, and a random
    colour with `NextSample<PlayerColor>`. `NextSample` indexes `array[(state * length) >> 32]`, so an empty array throws `IndexOutOfRangeException`. A mod that rebuilt `PlayerInventory`
    (a postfix on `PlayerInventory.FromSteamworks`, writing `PlayerInventory.FALLBACK`) made the spawn throw: no player existed and the lobby screen showed only the top bar.
    Patching the `UnlockedCivilizations` getter instead crashed the game at start: it is a few bytes of native code that IL2CPP shares between identical getters.
    ModCore adds its civilizations in a **prefix on `Player.NextCivilization`** (after the last unlocked civilization come the registered ones, then the game's own wrap-around).

## Scoreboard

`ScoreboardRow.Start` creates the row's stat cells by cloning the first cell: ping, six resources, population and technologies, and a trailing empty cell.
The row is `Scoreboard Row(Clone)` with a `Name` child (about 200 wide), the stat cells (about 96 wide) and the trailing empty `Stat(Clone)`, which holds the
`Button Send` child. That button appears on the resource cells after a market is built, to send resources to other players.

The board's frame grows by about 112 units for every cell that is added. The vanilla frame spans roughly x 269..1651 on a 1920 screen, so a mod column
should take the place of the empty trailing cell instead of widening the board. UiKit does this: it clones a stat cell, destroys the clone's `ScoreboardStat`
(so the game stops driving it) and writes the text from your function.
