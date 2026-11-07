---
summary: How a match is won or lost - defeat (no Castle left), losing a Castle, several Castles, military and wonder victory in free-for-all and teams, draws, the end screen
---

# Win conditions

!!! info "Code blocks"
    Declarations from the IL2CPP dump of game version 1.0.4, trimmed to what each section uses. `// 0x..` is the field offset, `// RVA 0x..` the method address in `GameAssembly.dll` (in Ghidra `0x180000000 + RVA`). Names stay stable between updates, offsets and RVAs do not. Private members are public in the BepInEx interop assemblies.

There are only two ways to win: **military** (everybody else is out) and **wonder victory** (lobby option, on by default). `WinState { winnerRef, reason (MILITARY 0 / WONDER 1), turn }`.

## Defeat

`TurnPhase.RecomputeIsDefeated` (`0xFDE040`):

```
Player.IsDefeated = !island.grid.buildings.Any(b => b.type is BuildingTypeCastle)
```

A player is out when **no Castle is left**: destroyed in combat, by a disaster (a tsunami touching it), or burnt by the player (see [Build phase](build-phase.md#burning-the-castle)).
It is recomputed for every island after every phase's `ApplyChanges`, so it is **not permanent**: if a Castle exists again (for example from an island code), the player is back in.

## Losing a Castle

`IslandGrid.RemoveBuilding` (`0x619360`) runs for every removed building (burn, combat, disasters). It clears the tiles and leaves **remains** (walls do not leave remains). If the building
is a Castle:

The grid methods that remove a Castle:

<!-- code: IslandGrid | RemoveBuilding, BurnAllBuildings -->
```csharp
namespace DiceKingdoms.Game;

public class IslandGrid : IslandSyncedBehaviour<IslandGrid.SyncData>
{
    public void RemoveBuilding(Building.Id id, float delay = 0);  // RVA 0x619360
    private void BurnAllBuildings(int2 center);                   // RVA 0x613A80
}
```
<!-- /code -->

- `IslandGrid.BurnAllBuildings` (`0x613A80`) removes **every building that is not a Castle**, in order of distance from the Castle with a growing delay (the spreading fire).
- `Statistics.SetCapacity(ZERO)`: all storage capacity becomes 0, which also **empties everything stored**, hammers included. The capacity comes back at the end of the phase
  (from the remaining buildings), but the stored amounts stay at 0.

**Several Castles** (possible with an [island code](island/save-format.md)): other Castles are not burnt, and the player is not defeated while any Castle is left
(confirmed in game: burning one of three Castles burnt everything else, the other two stayed and the game went on). Because the hammers are gone, only one Castle can be burnt per build phase.

## Free-for-all

`WinPhase.CheckWin` (`0x57AB10`): a game with only one island never ends by itself. Otherwise it calls `CheckWinTeam` when the lobby option **Teams** is on, else `CheckWinFFA` (`0x579EA0`):

The checks are in `WinPhase`; the result is a `WinState`:

<!-- code: WinPhase | *fields, CheckWin, CheckWinFFA, CheckWinTeam, WinNow, ShouldSkip -->
```csharp
namespace DiceKingdoms.Game;

public class WinPhase : TurnPhase<WinData, WinIslandState>
{
    public const int WONDER_TURN_DELAY = 1;

    private Nullable<WinPhase.WinState> WinNow();        // RVA 0x57B340
    public Nullable<WinPhase.WinState> CheckWin();       // RVA 0x57AB10
    private Nullable<WinPhase.WinState> CheckWinFFA();   // RVA 0x579EA0
    private Nullable<WinPhase.WinState> CheckWinTeam();  // RVA 0x57A410
    public override bool ShouldSkip();                   // RVA 0x57B2D0
}
```
<!-- /code -->

<!-- code: WinPhase.WinState | *fields -->
```csharp
public struct WinPhase.WinState : IEquatable<WinPhase.WinState>
{
    private NetRef<Player> winnerRef;        // 0x0
    public WinPhase.WinState.Reason reason;  // 0x8
    public int turn;                         // 0xC
}
```
<!-- /code -->

<!-- code: WinPhase.WinState.Reason |  -->
```csharp
public enum WinPhase.WinState.Reason
{
    MILITARY = 0,
    WONDER = 1,
}
```
<!-- /code -->

**Patch:** postfix `WinPhase.CheckWin` to change the win rules (host).

1. `alive` = islands whose owner is not defeated.
2. `alive == 0` → **draw** (it happens when the last players lose their last Castle in the same combat phase).
3. `alive == 1` → that player wins, reason **military**.
4. Otherwise, with wonder victory on: sort by `WonderCount`. If the top player has more Wonders than the second, they win with reason **wonder**, at
   `turn = WonderBuildTurn(top, second + 1) + 1` (`WONDER_TURN_DELAY = 1`): the turn after the leader first had more Wonders than anyone else, so the others get one turn to answer
   (destroy it or build their own). A tie: nobody wins.

So in free-for-all **the match ends only when one player is left**: a human losing does not end a match that still has two or more AIs. Gold for a victory needs at least 5 turns
(see [Lobby and Steam](lobby-and-steam.md#item-drops-and-the-store)).

`WinNow` (`0x57B340`) applies a win once its turn has come; `ShouldSkip` (`0x57B2D0`) skips the Win phase when no win is due. `WinRPC` broadcasts it.

## Teams

`CheckWinTeam` (`0x57A410`):

- Collect the teams of all players that are not defeated. None → draw. Exactly one → that team wins (military); the stored winner is the first such player in the player list.
- Two or more teams → only a wonder win: teams are compared by `MaxWonderCount`, **the most Wonders any single player of the team has** (teammates' Wonders are not added together).
  The winner turn is `MinWonderBuildTurn(top team, second team's count + 1) + 1`.

## End screen

`WinInterface.PlayAnimation` (`0x5EEA70`):

- The camera flies to the winner's island (on a draw: your own island).
- Title: a draw text, or one text with the winner: the **team** in team mode, the **player name** in free-for-all. The losing player sees the same "Victory!" title with the winner's name.
- Icon: draw → icon 7, military → the sword (icon 6), wonder → the winner's Wonder icon in their player colour.
- **Continue** closes the screen and leaves you as a spectator: free camera, your scoreboard row shows a skull, empty resources, inactive build menu. As a defeated player every island's
  actions are shown instantly ([Preview](build-phase.md#preview-instant-or-at-sync)).

The "imminent victory" warning (a Wonder win coming next turn) shows for 6 s and fades out in 1 s (`ImminentVictoryWarning`).
