---
summary: ModBridge - shows ModCore's lobby rules, compatibility warnings and civilizations on UiKit's screens; when a mod needs it and how to add a new bridge
---

# ModBridge

Connects [ModCore](modcore/index.md) (data, no screens) with [UiKit](uikit/index.md) (screens, no game logic). It has no API: install it and these appear, with nothing for a
mod author to call. Source and releases: [DiceKingdoms-ModBridge](https://github.com/DiceModders/DiceKingdoms-ModBridge).

- **Lobby rules**: every rule registered with `Core.Lobby.Rules` gets a row in the Settings box of the lobby screen, grouped per mod. The host edits it before the match, everybody
  else sees the host's value read-only. Rows follow the lobby live (a member joins, the host changes a value, the match starts and rows turn read-only).
- **Compatibility warnings**: `Core.Lobby.Compatibility.Warnings` are listed under the rules.
- **Civilizations**: every civilization added with `Core.Civilizations.Add` gets an entry in the Rules drop-down.

So a mod that registers a rule needs no UI code at all:

```csharp
[BepInPlugin("author.my-mod", "My Mod", "1.0.0")]
[BepInDependency("DiceModders.ModCore")]
public class MyModPlugin : BasePlugin {
    public override void Load() {
        var storms = Core.Lobby.Rules.Toggle("author.my-mod.storms", "Storms", true,
                                             description: "Storms can sink ships.");   // ModBridge draws the row
        Core.Match.PhaseEntered += phase => { if (storms.Value && phase == MatchPhase.Disasters) MaybeStorm(); };
    }
}
```

Mods that use ModCore do not depend on ModBridge: without it their rules still work (with their defaults, or the host's value set by another tool); there are just no rows.
List it as an optional dependency, or required if your mod is useless without the screens.

## Adding a bridge

When a new ModCore feature needs a screen, the screen goes into ModBridge so that ModCore and UiKit stay independent: one file per screen with an `Install(UiScope scope)`
method that reads `Core.*`, draws with the scope's handles and subscribes to ModCore's events to update them, called from `WhenReady`.

```csharp
internal static class KillCounterScreen {
    public static void Install(UiScope scope) {
        var column = scope.Native.Scoreboard.AddColumn("modbridge.kills", "Kills");
        Core.Match.PhaseSynced += _ => {
            foreach (var p in Core.Players.All)
                if (p.Island != null) column?.SetValue(p.Island, CountKills(p).ToString());
        };
        Core.Match.Ended += () => column?.ClearValues();
    }
}

// Plugin.Load():  scope.WhenReady(s => { LobbyRulesScreen.Install(s); CivilizationEntries.Install(s); KillCounterScreen.Install(s); });
```
