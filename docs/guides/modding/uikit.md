---
summary: UiKit, a library that builds mod windows, buttons and settings rows from the game's own UI
---

# UiKit: native UI for mods

UiKit is a small BepInEx 6 (IL2CPP) plugin for mods that need user interface. Everything it shows is a **clone of the game's own UI**: the same buttons,
sprites, fonts, tooltips, sounds and cursor changes. Mod windows therefore look and behave like the game, and several mods can use UiKit side by side.

!!! info "Source code"
    [UIKit_Another_Version](https://github.com/DiceModders/UIKit_Another_Version) (library, demo and README).

| You want | Use | Page |
|---|---|---|
| A window with buttons, labels, toggles, sliders, pictures | `UiKit.Window(...)` | [Windows and widgets](uikit-windows.md) |
| A button in the top-right bar (next to Rules / Settings) | `UiKit.TopBar.Add(...)` | [Game screens](uikit-game-screens.md) |
| Rows in the game's Settings screen | `UiKit.Settings.Add(...)` | [Game screens](uikit-game-screens.md) |
| A new civilization in the Rules screen | `UiKit.Rules.AddCivilization(...)` | [Game screens](uikit-game-screens.md) |
| An extra column in the match scoreboard | `UiKit.Scoreboard.AddColumn(...)` | [Game screens](uikit-game-screens.md) |

## Using UiKit in your mod

1. Build UiKit (or download `DiceModders.UiKit.dll` from a release) and put it into `BepInEx/plugins`. It is its own plugin, installed once for all mods.
2. In your project, reference `DiceModders.UiKit.dll` and make your plugin depend on it, so that UiKit loads first:

```csharp
[BepInPlugin("author.mymod", "My Mod", "1.0.0")]
[BepInDependency("DiceModders.UiKit")]
public class Plugin : BasePlugin { /* ... */ }
```

3. Do not copy `DiceModders.UiKit.dll` into your own package: two copies would both patch the game. On Thunderstore, list UiKit in the `dependencies`
   of your `manifest.json` instead (see [Publishing a mod](publishing.md)).

## A first window

```csharp
UiKit.Window(w => {
    w.Size(360, 240).Draggable();
    int hp = 100;
    var label = w.Body.Label($"HP: {hp}");
    w.Body.Button("+10", () => { hp += 10; label.SetText($"HP: {hp}"); });
    w.Body.Toggle("God mode", false, on => Log.LogInfo(on));
    w.Body.Horizontal(row => {
        row.IconButton("hammer", () => Log.LogInfo("hi"), "Tooltip text");
        row.Button("Close", () => w.Close());
    }).FixedH(32);
});
```

Create windows after the game's UI exists (for example from a button press or a key), because UiKit clones templates from live game screens.
If a template is missing, creating the widget throws an error saying which template was not found.

## Try the demo

The library contains demo windows, a top bar button and a "UiKit demo" settings block. They are off by default; set `[Demo] Enabled = true`
in the UiKit config file (`BepInEx/config/*UiKit*.cfg`) to see them.

## Design rules

- Everything UiKit creates is named `DM ...`, so mods can tell it from the game's objects.
- Buttons are 32 units high (31.75, see [Game UI](../../game-architecture/ui.md)); the layout system is the game's own uGUI layout groups.
- Ids passed to `TopBar`, `Scoreboard`, `Rules` and `Settings` must be unique per mod: prefix them with your mod name.
