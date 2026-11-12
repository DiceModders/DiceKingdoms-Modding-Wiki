---
summary: ModCore lobby API - members and host, lobby rules the host sets for everybody (frozen at match start), per-civilization rule groups, the version and compatibility check
---

# Lobby and rules

## The lobby

`Core.Lobby` tells you whether the player is in a Steam lobby, who is in it and who hosts it. `Changed` fires when somebody joins, leaves or is kicked, or the host changes.
AI players are not Steam members; list them with [`Core.Players`](match-and-players.md).

```csharp
if (Core.Lobby.InLobby && Core.Lobby.IsHost) Log.LogInfo("I am the host");

foreach (LobbyMember m in Core.Lobby.Members)
    Log.LogInfo($"{m.Name} id={m.SteamId} host={m.IsHost} me={m.IsLocal}");

Core.Lobby.Changed += () => RefreshMyWindow();
```

| Member | Meaning |
|---|---|
| `InLobby`, `IsHost` | In a Steam lobby; the local player owns it |
| `Members` | `SteamId`, `Name`, `IsHost`, `IsLocal`, `IsConnected` |
| `LocalSteamId`, `Current` | The local Steam id; the raw `Steamworks.Data.Lobby?` |
| `Changed` | Joined / left, member joined / left / kicked, owner changed |

## Lobby rules

A rule is a setting the host edits before the match. Its value lives in the Steam lobby data, so everybody reads the same value, and it is **frozen for the match** from the moment the islands start to spawn (a little before `Match.Started`).
Outside a Steam lobby (a local game) the player sets it for himself. Always read `Value` in gameplay code.

```csharp
var meteors = Core.Lobby.Rules.Toggle("author.my-mod.meteors", "Meteors", defaultValue: true,
                                      description: "Disasters hit the islands.");
var speed   = Core.Lobby.Rules.Number("author.my-mod.speed", "Speed", defaultValue: 1, min: 1, max: 5, step: 1);

meteors.Changed += on => Log.LogInfo("meteors: " + on);   // on every machine
if (meteors.Value) SpawnMeteor();                          // host's value, frozen during the match
```

| Member of `LobbyRule<T>` | Meaning |
|---|---|
| `Value` | What to play with: host's value in a lobby, the local value outside one, a frozen copy during a match |
| `Live` | The lobby's value ignoring the freeze (what a settings screen shows) |
| `Editable` | The local player may change it now (host before the match, or a local game) |
| `TrySet(value)` | Change it; false when not allowed |
| `Visible` | Your `visible:` callback; screens hide invisible rules (the value still applies) |
| `IsNonDefault` | Changed from the default (this triggers compatibility warnings) |
| `AffectsSimulation` | `false` for purely visual rules: they never cause a warning |
| `Group` / `InGroup(group)` | The heading it is listed under (below) |

Prefix keys with your mod id; registering a key twice returns the same rule. `Core.Lobby.Rules.All` lists every rule with the mod that registered it, `Registered` fires for new ones. `Core.Lobby.Rules.Fixed` is true while the values are frozen for a match; a rule that changes the islands (read in `IslandGrid.Awake`) should act only then, so menu and editor islands stay normal.

## Rule groups (per civilization)

A lobby screen with many rules can show headings and open one at a time. Every civilization has a heading (`ForCivilization`, created automatically for added civilizations);
a mod can make its own. Rules without a group belong to the general part. A tech tree replacement for a built-in civilization is put under that civilization automatically.

```csharp
var civ = Core.Civilizations.Add("author.my-civ", "My Civ", () => myTree, skinsLike: 2);

Core.Lobby.Rules.Toggle("author.my-civ.archers", "Long-range archers", false)
    .InGroup(Core.Lobby.Groups.ForCivilization(civ.Id));

var events = Core.Lobby.Groups.Get("author.events", "Random events", order: 600);
Core.Lobby.Rules.Toggle("author.events.storms", "Storms", true).InGroup(events);
```

A screen reads the groups and their rules; `Changed` fires when a group appears or a rule moves.

```csharp
foreach (LobbyGroup g in Core.Lobby.Groups.All) {          // sorted: built-in civilizations, mod civilizations, mod groups
    DrawHeading(g.Title, collapsed: g.StartsCollapsed);
    foreach (ILobbyRule r in g.Rules) if (r.Visible) DrawRow(r);
}
Core.Lobby.Groups.Changed += RebuildList;
```

## Version and compatibility check

Every player publishes the rules and civilizations he has, each with its mod's version. A gameplay change (a rule, a tech tree, unit or building values, a civilization) is
used only when every member has the **same version** of that mod, so different versions cannot desync. The warnings name the problem:

```csharp
foreach (string line in Core.Lobby.Compatibility.Warnings)
    Log.LogWarning(line);   // "Alice has My Mod 1.0.0, the host has 1.1.0: this rule will be off for everyone in the match." / "Bob does not have My Mod, this rule will be off for everyone in the match."
bool ok = Core.Lobby.Compatibility.IsCompatible;
```

ModCore is desync protection, not anticheat: it warns before the match and falls back to the game's own values. During a match the host refuses a player who joins (or rejoins) without every rule the match uses, in the same version, through the network layer's own connection approval; the others keep playing.

## Choices and text rules

`Rules.Choice` is one value out of a few named ones; ModBridge shows the game's own drop-down. Use it instead of several toggles that exclude each other.

```csharp
var castle = Core.Lobby.Rules.Choice("author.my-mod.castle", "Castle hits", 0, new[] { "Game rule", "Any meteor", "Never" });
if (castle.Value == 2) ProtectCastles();
```

`Rules.Text` carries a text value, for data your own window edits (for example many changed numbers in one compact string). It is synced, frozen at match start and checked like every rule, but ModBridge draws no row for it. Steam keeps up to about 8 KB per value.

```csharp
var values = Core.Lobby.Rules.Text("author.my-mod.values", "My values", "", visible: () => false);
values.Changed += text => Apply(Parse(text));   // every player, whenever the host changes it
if (values.Editable) values.TrySet("archer.range=8;castle.health=400");   // host, before the match
```
