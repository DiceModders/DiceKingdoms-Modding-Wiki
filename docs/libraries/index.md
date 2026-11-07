---
summary: The shared modding libraries - ModCore (game and multiplayer logic), UiKit (native interface) and ModBridge (connects the two) - what each one is for and how a mod uses them
---

# Modding Libraries

Three libraries that mods can share instead of each mod patching the same game code again. They are separate BepInEx 6 plugins: a player installs each one once,
and every mod that needs it lists it as a dependency.

| Library | What it does | Has screens |
|---|---|---|
| [ModCore](modcore/index.md) | Steam lobby, lobby rules the host sets for everybody, match events, players, new civilizations, unit / building / technology values, AI build orders, models, pictures and caches | No |
| [UiKit](uikit/index.md) | Windows, buttons, sliders and settings rows in the game's own look; top bar buttons, Settings tabs, scoreboard columns | Yes |
| [ModBridge](modbridge.md) | Shows ModCore's data on UiKit's screens (lobby rule rows, civilizations in the Rules drop-down). No API of its own | Yes |

ModCore and UiKit do not know each other, so a mod can use either one alone. ModBridge is the only place where they meet.

## Install (players)

Copy the DLLs into `Dice-Kingdoms\BepInEx\plugins`. A mod manager does this for you when a mod lists them as dependencies.

```
DiceModders.ModCore.dll
DiceModders.UiKit.dll
DiceModders.ModBridge.dll
```

## Use them in a mod

Reference the DLLs with `Private=false` (never copy a library into your own package: two copies would both patch the game), then declare the dependencies so BepInEx loads
the libraries before your plugin.

```xml
<ItemGroup>
  <Reference Include="interop/*.dll"><Private>false</Private></Reference>
  <Reference Include="DiceModders.ModCore"><HintPath>libs\DiceModders.ModCore.dll</HintPath><Private>false</Private></Reference>
  <Reference Include="DiceModders.UiKit"><HintPath>libs\DiceModders.UiKit.dll</HintPath><Private>false</Private></Reference>
</ItemGroup>
```

```csharp
using BepInEx;
using BepInEx.Unity.IL2CPP;
using DiceModders.ModCore;
using DiceModders.UiKit;
using Core = DiceModders.ModCore.ModCore;   // the class has the same name as its namespace
using Ui   = DiceModders.UiKit.UiKit;

[BepInPlugin("author.my-mod", "My Mod", "1.0.0")]
[BepInDependency("DiceModders.ModCore")]
[BepInDependency("DiceModders.UiKit")]
public class MyModPlugin : BasePlugin {
    public override void Load() {
        var meteors = Core.Lobby.Rules.Toggle("author.my-mod.meteors", "Meteors", true);   // ModBridge shows it in the lobby
        var ui = Ui.CreateScope("author.my-mod");
        ui?.WhenReady(s => s.Native.TopBar.AddIconButton("hammer", "My Mod", () => Log.LogInfo("clicked")));
    }
}
```

## Rules every library follows

- **Event driven.** Nothing polls every frame: the libraries react to the game's own events and hand you events.
- **Never throws into the game.** Hooks are wrapped; a failure is logged once (`[ModCore]`, `[UiKit]`, `[ModBridge]` in `BepInEx\LogOutput.log`).
- **Multiplayer safe.** Anything that changes the simulation is used only when every player can agree on it (see [Lobby and rules](modcore/lobby.md)).
