---
summary: UiKit extension points for the game's own screens - top bar, Settings, Rules civilizations, scoreboard
---

# UiKit: the game's own screens

UiKit can add content to screens that already exist in the game. Ids must be unique per mod, so prefix them with your mod name.

## Top bar

```csharp
UiKit.TopBar.Add("mymod.panel", "hammer", () => ShowPanel(), "My panel", order: 10);
```

The button is a clone of the game's top-right buttons (Scoreboard, Rules, Settings, Back To Menu). Buttons are placed left of the game's own,
sorted by `order` and then by registration, and re-attach themselves after scene changes. The returned `TopBarButton` has `SetIcon`, `SetTooltip`,
`SetVisible`, `SetInteractable` and `Remove`.

## Settings rows

```csharp
UiKit.Settings.Add(SettingsTab.Game, "My Mod", s => {
    s.Toggle("Enabled", () => cfg.Enabled, v => cfg.Enabled = v);
    s.Stepper("Speed", () => cfg.Speed.ToString("0.0"), d => cfg.Speed += d * 0.5f);
    s.Button("Reset", "Reset", () => cfg.Reset());
    s.Info("Changes apply immediately.");
});
```

The callback runs each time the game rebuilds that tab, so no polling is needed. Toggle and stepper rows refresh while the screen is open
(four times a second), and both buttons of a stepper update the shown value at once. Tabs: `Video`, `Audio`, `Input`, `Game` and `Lobby`
(the settings box of the match-start screen).

```csharp
UiKit.Settings.Opened += screen => Log.LogInfo("Settings screen shown");
// hidden unlock: 5 quick clicks in the bottom-left corner of the Settings screen
UiKit.Settings.AddHotspot("mymod.unlock", SettingsCorner.BottomLeft, 5, () => Unlock());
```

## Civilizations on the Rules screen

```csharp
var civ = UiKit.Rules.AddCivilization("authorname.mycivilization", "My civilization", () => myTechTree, skinsLike: 2);
UiKit.Rules.Opened += screen => { /* the Rules screen was shown */ };
```

- **Key:** it becomes the civilization number. The same key gives the same id on every machine (1000 and up), so several mods never collide and
  multiplayer and saves agree. Use `civ.Id` to tag your own data.
- **Tech tree:** the function returns the `DiceKingdoms.Game.TechTree` of your civilization. UiKit returns it whenever the game asks for the tree of that
  number, on the Rules screen and in a match. It may be `null` if your mod adds the tree to the game's list itself.
- **`skinsLike`:** the game sets a player's biome with a switch over the built-in civilizations 0..4 and throws for anything else. The biome and visuals of the
  chosen built-in civilization are used instead: 0 Europe, 1 Middle East, 2 Viking, 3 Ancient, 4 Maya.
- **Your part:** a civilization also needs everything else the game looks up per civilization (skins, colours, the tech tree asset). That data belongs
  to your mod. See [Rules and civilizations](../../game-architecture/rules-and-civilizations.md) for how the game handles it.

## Scoreboard columns

```csharp
UiKit.Scoreboard.AddColumn("mymod.gold", island => GoldOf(island).ToString(), "Gold income");
```

Adds one cell to every row of the match scoreboard, copied from the game's own stat cells and refreshed four times a second. The game's trailing empty cell
(it holds the "send resources" button) keeps working; the board grows by the width of your column. See
[Rules and civilizations](../../game-architecture/rules-and-civilizations.md#scoreboard) for the row structure.
