---
summary: ModCore, the library for lobby state, host-controlled lobby rules, desync warnings and mod civilizations
---

# ModCore: lobby, rules and civilizations

ModCore is a BepInEx 6 (IL2CPP) plugin with the **game and network logic** that multiplayer mods keep needing. It has no interface at all (screens are [UiKit](uikit.md);
[ModBridge](modbridge.md) connects the two) and it does not depend on UiKit.

!!! info "Source code"
    [DiceKingdoms-ModCore](https://github.com/DiceModders/DiceKingdoms-ModCore) (library, README, `docs/NOTES.md`, `docs/EXAMPLE_MOD.md`).

!!! note "Desync protection, not anticheat"
    The game has no anticheat or ban system: it uses Steam to connect the players and its own synchronisation for the match. A mod that changes the simulation on one machine only makes that
    machine disagree with the others. ModCore's compatibility check tells the players about this *before* the match; it cannot stop anybody from playing.

| You want | Use |
|---|---|
| Know whether the player is in a Steam lobby, who is in it and who hosts it | `ModCore.Lobby.InLobby / IsHost / Members / Changed` |
| React when the lobby turns into a match (and when it ends) | `ModCore.Lobby.MatchStarted / MatchEnded` |
| A setting the host chooses in the lobby and everybody follows | `ModCore.Lobby.Rules.Toggle / Number` |
| Find out who in the lobby lacks a mod | `ModCore.Lobby.Compatibility.Warnings` |
| Add a civilization the game accepts | `ModCore.Civilizations.Add(...)` |

## Using it in a mod

```csharp
[BepInPlugin("author.mymod", "My Mod", "1.0.0")]
[BepInDependency("DiceModders.ModCore")]
public class MyModPlugin : BasePlugin { /* ... */ }
```

Reference `DiceModders.ModCore.dll` with `Private=false` and never copy it into your package. The namespace `DiceModders.ModCore` contains a class `ModCore`; inside a namespace that
starts with `DiceModders.` write `using Core = DiceModders.ModCore.ModCore;`.

## Lobby and rules

```csharp
Core.Lobby.MatchStarted += () => Freeze();
var meteors = Core.Lobby.Rules.Toggle("mymod.meteors", "Meteors", defaultValue: true, description: "Disasters hit the islands.");
var speed   = Core.Lobby.Rules.Number("mymod.speed", "Speed", defaultValue: 1, min: 1, max: 5, step: 1);
if (meteors.Value) SpawnMeteor();     // the host's value in a lobby, frozen when the match starts
```

A *rule* lives in the Steam lobby data, so every client sees the host's value, and it is frozen at match start. Read `Value` in gameplay code (never cache it); `Live` is the current lobby value
for a settings screen. When the host changes a rule from its default and a member does not have the mod, `Core.Lobby.Compatibility.Warnings` has a line such as
"Alice does not have Meteors, this rule may desync." Steam sends no event when the local player leaves a lobby, so ModCore polls four times a second.

## Civilizations

```csharp
var civ = Core.Civilizations.Add("authorname.mycivilization", "My civilization", () => myTechTree, skinsLike: 2);
```

The key becomes the civilization number (the same key gives the same id on every machine, 1000 and up). ModCore makes the game accept it: the tech tree lookup returns your tree (or the tree of the
`skinsLike` built-in civilization), `PlayerSkins` borrows that civilization's biome, and clicking a castle on the match-start screen reaches your civilization after the last unlocked one.
Skins, colours and the tech tree asset are still yours. How and why the game behaves like this (and what must **not** be patched) is in
[Rules and civilizations](../../game-architecture/rules-and-civilizations.md#choosing-a-civilization-for-play-the-castle-click).
