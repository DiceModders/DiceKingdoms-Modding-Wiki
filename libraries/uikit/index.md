---
summary: UiKit getting started - one scope per mod, WhenReady, the main-thread and null contract, native handles and their states, theme and config
---

# UiKit

Windows, widgets and additions to the game's own screens, in the game's look. Source and releases: [UiKit](https://github.com/DiceModders/UiKit).

## Project setup

Reference the DLL without copying it and depend on it.

```xml
<Reference Include="DiceModders.UiKit">
  <HintPath>libs\DiceModders.UiKit.dll</HintPath>
  <Private>false</Private>
</Reference>
```

```csharp
[BepInPlugin("author.my-mod", "My Mod", "1.0.0")]
[BepInDependency("DiceModders.UiKit")]
public class MyModPlugin : BasePlugin { /* ... */ }
```

## Scopes

A mod never creates the UI runtime. `CreateScope` returns your scope (null when UiKit is not available), and everything made through it is removed when you dispose it.
`WhenReady` runs once on the main thread, after the shared UI layers and the game's top bar exist; registered later, it runs at once.

```csharp
using DiceModders.UiKit;
using DiceModders.UiKit.Custom;
using DiceModders.UiKit.Native;

public class MyModPlugin : BasePlugin {
    private UiScope? _ui;

    public override void Load() {
        _ui = UiKit.CreateScope("author.my-mod");
        _ui?.WhenReady(ui => {
            ui.Native.TopBar.AddIconButton("hammer", "Open My Mod", () => OpenWindow(ui));
        });
    }

    public override bool Unload() {
        _ui?.Dispose();   // removes every window and native addition of this mod
        return true;
    }
}
```

`WhenReady` does not wait for match-only screens (Settings, scoreboard, Rules). Native additions registered early stay `Pending` and attach when their screen appears.

## Contract

- Call UiKit on the main thread only.
- Factories return `null` when the scope is gone, an argument is invalid or building failed, so use `?.`. Fluent methods do nothing on an invalid handle and return it.
- `Dispose` can be called any number of times.
- Your callbacks run behind an error boundary: one failure does not stop other mods or the game.

```csharp
var button = ui.Native.TopBar.AddIconButton("hammer", "Open", Open);
if (button is { State: NativeState.Failed })
    Log.LogWarning("top bar button: " + button.Status);
button?.SetInteractable(false);
```

| Native handle state | Meaning |
|---|---|
| `Pending` | Waiting for its screen, asset or template |
| `Attached` | Live in the game's UI |
| `Detached` | Was attached, its screen is gone; reattaches when it returns |
| `Failed` | The last attempt failed; `Status` says why |
| `Unsupported` | The game or the `Templates` setting does not allow it |
| `Disposed` | Removed for good |

## Theme

The game's sprites, icons, font and colours, for your own widgets. Lookups return `null` for something not loaded.

```csharp
var theme = ui.Custom.Theme;
Sprite? hammer = theme.Icon("hammer");
Sprite? panel  = theme.Sprite("Background-3");
TMP_FontAsset? font = theme.Font;
Color text = theme.TextColor;
```

Players choose in UiKit's config whether the game's own widget templates are used:

```ini
[Widgets]
Templates = PreferNative   # PreferNative, ManagedOnly or NativeOnly
```

## Next

- [Windows and widgets](windows-and-widgets.md): your own windows and layout.
- [Game screens](game-screens.md): top bar, Settings, scoreboard, Rules civilizations.
