---
summary: ModCore getting started - project setup, the name clash alias, when you can call it, and a map of every feature
---

# ModCore

Shared game and multiplayer logic, no interface. Source and releases: [DiceKingdoms-ModCore](https://github.com/DiceModders/DiceKingdoms-ModCore).

## Project setup

Reference the DLL without copying it and make your plugin depend on it, so ModCore loads first.

```xml
<Reference Include="DiceModders.ModCore">
  <HintPath>libs\DiceModders.ModCore.dll</HintPath>
  <Private>false</Private>
</Reference>
```

```csharp
[BepInPlugin("author.my-mod", "My Mod", "1.0.0")]
[BepInDependency("DiceModders.ModCore")]
public class MyModPlugin : BasePlugin { /* ... */ }
```

## The name clash

The namespace `DiceModders.ModCore` contains a class `ModCore`. Inside a namespace that starts with `DiceModders.` the compiler reads `ModCore` as the namespace, so use an alias.
Outside `DiceModders.*` a plain `using DiceModders.ModCore;` is enough.

```csharp
using DiceModders.ModCore;                   // types: LobbyMember, ModCivilization, LobbyRule<T> ...
using DiceModders.ModCore.Tools;             // helpers: Patcher, Meshes, Sprites, TextureCache ...
using Core = DiceModders.ModCore.ModCore;    // the API: Core.Lobby, Core.Match, Core.Civilizations ...
```

## When you can call it

Everything is safe to call from `Load()`. ModCore installs its hooks the first time you touch it. Game objects (prefabs, tech trees, buildings) exist once the game data has
loaded; put code that needs them into `WhenGameDataLoaded`, which runs at once if the data is already there.

```csharp
public override void Load() {
    Core.Lobby.Changed += () => Log.LogInfo("lobby changed");          // fine at any time
    Core.Civilizations.WhenGameDataLoaded(() => {
        Log.LogInfo("prefabs, trees and buildings exist now");         // main menu, once
    });
}
```

## Features

| You want | Page | API |
|---|---|---|
| Know the lobby, its members and host; settings the host chooses for everybody | [Lobby and rules](lobby.md) | `Core.Lobby`, `Core.Lobby.Rules`, `Core.Lobby.Groups` |
| React to the match, its phases and players | [Match and players](match-and-players.md) | `Core.Match`, `Core.Players` |
| Add a civilization, rename texts, reskin a built-in one, give AI build orders | [Civilizations](civilizations.md) | `Core.Civilizations` |
| Change unit, building and technology values safely | [Gameplay values](gameplay-values.md) | `Core.Units`, `Core.Balance` |
| Own models, pictures, fast cached loading | [Models, pictures and caches](assets.md) | `Tools.Meshes`, `Tools.Sprites`, `Tools.TextureCache` ... |
| Let players switch your mod on and off in game | [Mod switch](mod-switch.md) | `Core.Mods.Switch` |
| Patch game methods by name, reflection, raw memory | [Tools](tools.md) | `Tools.Patcher`, `Tools.Reflect`, `Tools.Mem` |
