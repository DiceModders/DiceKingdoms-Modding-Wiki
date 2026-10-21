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

The callback runs each time the game rebuilds that tab, so no polling is needed. Toggle, stepper and slider rows refresh while the screen is open
(four times a second), and both buttons of a stepper update the shown value at once. Tabs: `Video`, `Audio`, `Input`, `Game` and `Lobby`
(the settings box of the match-start screen).

A number between two limits is best shown with the game's own slider (the control of its Frame Rate Limit setting), which UiKit clones:

```csharp
s.Slider("Speed", min: 1, max: 5, get: () => speed, set: v => speed = (int)v, wholeNumbers: true, format: "0");
```

Rows can be disabled or hidden while the screen is open with a `SettingsRowOptions { Enabled = ..., Visible = ... }` (re-read four times a second).
The layout of these rows, and why the library places the controls itself, is described in [Game UI](../../game-architecture/ui.md#settings-rows-and-the-lobby-settings-box).

```csharp
UiKit.Settings.Opened += screen => Log.LogInfo("Settings screen shown");
// hidden unlock: 5 quick clicks in the bottom-left corner of the Settings screen
UiKit.Settings.AddHotspot("mymod.unlock", SettingsCorner.BottomLeft, 5, () => Unlock());
```

The spot is an invisible 44x44 button in the corner of the Settings panel, drawn above the game's own controls. An empty or `null` section title adds
the rows without a header. Rows stay clear of the Close button and the window edge, and the mouse wheel scrolls the tab. To see invisible elements
(the hotspot in red, the wheel catcher in green) while debugging, set `[Debug] ShowInvisible = true` in `BepInEx/config/DiceModders.UiKit.cfg`.

## Civilizations on the Rules screen

UiKit only adds an **entry to the civilization drop-down** of the Rules screen; it knows nothing about how a civilization works in a match.

```csharp
UiKit.Rules.AddCivilizationEntry(civilizationId, "My civilization");
UiKit.Rules.Opened += screen => { /* the Rules screen was shown */ };
```

Registering the civilization itself (its number, tech tree, skins) is [ModCore](modcore.md#civilizations); [ModBridge](modbridge.md) mirrors every
registered civilization into the drop-down, so a mod does not call this itself. See [Rules and civilizations](../../game-architecture/rules-and-civilizations.md)
for how the game handles civilizations.

## Scoreboard columns

```csharp
UiKit.Scoreboard.AddColumn("mymod.gold", island => GoldOf(island).ToString(), "Gold income");
```

Adds one cell to every row of the match scoreboard, copied from the game's own stat cells and refreshed four times a second. The game's trailing empty cell
(it holds the "send resources" button) keeps working; the board grows by the width of your column. See
[Rules and civilizations](../../game-architecture/rules-and-civilizations.md#scoreboard) for the row structure.
