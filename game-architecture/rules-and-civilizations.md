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

To make a new civilization selectable you have to: append an option to the `TMP_Dropdown`, append a boxed enum value (`Enum.ToObject`) to `values`, and answer
`TechTree(civilization)` for that number.
A player's biome is chosen with a switch over the built-in civilizations 0..4 and throws for any other value, so a new civilization has to map to one of them
for its visuals. Anything else the game looks up per civilization (for example `PlayerSkins` / `SkinRegistry` entries) must come from your mod.

## Scoreboard

`ScoreboardRow.Start` creates the row's stat cells by cloning the first cell: ping, six resources, population and technologies, and a trailing empty cell.
The row is `Scoreboard Row(Clone)` with a `Name` child (about 200 wide), the stat cells (about 96 wide) and the trailing empty `Stat(Clone)`, which holds the
`Button Send` child. That button appears on the resource cells after a market is built, to send resources to other players.

The board's frame grows by about 112 units for every cell that is added. The vanilla frame spans roughly x 269..1651 on a 1920 screen, so a mod column
should take the place of the empty trailing cell instead of widening the board.
