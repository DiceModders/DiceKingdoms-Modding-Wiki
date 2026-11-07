---
summary: ModCore match and players API - match start/end, phases and turns, match settings, the player list with humans and AI, changing a player's civilization
---

# Match and players

## Match events

The events come from the game's own `GameState` events, so they are raised on every machine (host and clients), on the main thread. Lobby rules are already frozen when
`Started` fires and follow the lobby again after `Ended`.

```csharp
Core.Match.Started      += () => Log.LogInfo("match started");
Core.Match.PhaseEntered += phase => Log.LogInfo($"turn {Core.Match.Turn}: {phase}");
Core.Match.PhaseSynced  += phase => Log.LogInfo($"{phase} ends, actions are applied now");
Core.Match.GameOver     += () => Log.LogInfo("somebody won");
Core.Match.Ended        += () => Log.LogInfo("back in the lobby or menu");
```

| Member | Meaning |
|---|---|
| `Running`, `Phase`, `Turn` | Between `Started` and `Ended`; current `MatchPhase` (`Start`, `Harvest`, `Disasters`, `Build`, `Combat`, `Win`); turn from 1 |
| `PhaseSynced(phase)` | The phase's time ran out or everybody was ready: actions are applied now (AI players move at this moment) |
| `IslandsSpawned`, `Rejoined`, `Reset` | Islands loaded for everybody; this machine reconnected; the game reset its match state |
| `Settings` | `TimerSpeed`, `MultiMeteor`, `Teams`, `WonderVictory`, read live |
| `IsSolo`, `IsTraining` | Fewer than two humans; the tutorial |

## Players

`Core.Players` lists the players of the lobby screen and the match, humans and AI. Values are read live, so keep the `MatchPlayer` and read it when you need it.

```csharp
foreach (MatchPlayer p in Core.Players.All)
    Log.LogInfo($"{p.Name} ai={p.IsAI} {p.Difficulty} {p.CivilizationName} {p.ColorName} team={p.Team} defeated={p.IsDefeated}");

MatchPlayer me = Core.Players.Local, host = Core.Players.Host;
```

| `MatchPlayer` | Meaning |
|---|---|
| `Id` | Steam id for humans, a made-up id for AI (stable, used for reconnects) |
| `Name`, `IsAI`, `IsLocal`, `IsHost` | Who it is |
| `Difficulty`, `Strategy` | AI only; the strategy is picked when the Start phase ends |
| `Civilization`, `CivilizationName` | 0..4 built in, 5..127 mod civilizations |
| `ColorIndex`, `ColorName`, `Color`, `Team` | Colour 0 Red .. 11 Purple; team (colours in groups of three) |
| `Island`, `IsDefeated`, `IsConnected`, `Game` | Island once spawned, defeat, connection, the game's own `Player` |

## Changing a player's civilization

The host can set the civilization of any player (an AI too) before the match, when the player is not ready. Mod civilizations are allowed only when every member has them.

```csharp
foreach (MatchPlayer p in Core.Players.All)
    if (p.IsAI && !Core.Players.TrySetCivilization(p, civ.Id, out string why))
        Log.LogWarning($"{p.Name}: {why}");   // not the host, match running, player ready, unknown civilization ...
```
