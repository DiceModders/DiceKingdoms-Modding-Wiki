---
summary: How a match runs - GameState, the phase loop, phase states, ready and sync, timers, the start phase, world layout, snapshots and rejoin
---

# Turn loop and phases

*Code: [where to patch and the declarations](#code-reference).*

## Main objects

- `GameState : NetworkedBehaviour` is the match controller, one per match. It holds the players, islands, phase objects, `Log`, `Chat`, `SnapshotStore`, `AIPlay`, `LobbySettings`
  and a match-wide `Unity.Mathematics.Random random`.
- `Island : NetworkedBehaviour` belongs to one player: `owner` (`NetworkedVar<Player>`, `0x90`), `grid` (`IslandGrid`, `0x98`), `statistics` (`Statistics`, `0xA0`: resources and learnt technologies).
- `Player : NetworkedBehaviour`: name, colour, skins / civilization, AI state, `isDefeated`, `isAwaited`, ping, start-game slot.

Useful `GameState` fields: `players` `0xE8`, `playerIds` `0xF8`, `lobbySettings` `0xD0`, `phaseOrder` `0x170`, `startPhase` `0x178`, `harvestPhase` `0x180`, `disastersPhase` `0x188`,
`buildPhase` `0x190`, `combatPhase` `0x198`, `winPhase` `0x1A0`, `turnCounter` (`NetworkedVar<int>`) `0x1B8`, `TurnPhase` `0x1C0`, `IsTraining` `0x1C8`.

`GameState` has `Action` fields that are good hook points for mods: `onIslandSpawned`, `onStartGame`, `onEnterPhase`, `onSyncPhase`, `onEndGame`, `onExit(string)`, `onReset`, `onRejoin`,
`onDisasterLog(Player, StructuredText)`.

## Phase order

`GameState.Awake` (`0x5F26A0`) sets

```
phaseOrder = [ HarvestPhase, DisastersPhase, BuildPhase, CombatPhase, WinPhase ]
```

`StartPhase` ("Settle") is **not** in the loop: it runs once at the start of the match. `Awake` also seeds `random`, shuffles the player colours into teams and wraps the lobby settings in a
`NetworkedVar`.

| Phase | Time limit (timer FAST) | Notes |
|---|---|---|
| Start ("Settle") | 60 s | once per match |
| Harvest | 40 s (+5 s sync) | [Harvest and dice](harvest.md) |
| Disasters | none | ends when every client has played the animations, [Disasters](disasters.md) |
| Build | 70 s | [Build phase](build-phase.md) |
| Combat | 35 s | planning time; the fight is simulated and animated afterwards, [Combat](combat.md) |
| Win | none | [Win conditions](win-conditions.md) |

Timer speed (lobby option `TimerSpeed`): **FAST ×1, SLOW ×1.5, UNLIMITED** = no limit (`TurnPhase.get_TimeLimit` `0xFE0460`). Games with fewer than two human players are forced to
UNLIMITED (`GameState.StartGame`). The values are fields of the phase components in the scene (`timeLimit`, `timeLimitSync`), not code.

## Phase states

Each phase is a `TurnPhase<TData, TIslandState>` with a `PhaseState`: `ACTIVE = 0`, `SYNC = 1`, `INACTIVE = 2`. For every island the phase holds one `PhaseData` object (for example `BuildData`);
per-player state is in `islandStates` (keyed by client id).

`GameState.UpdateTurnPhase` (`0x5F6E20`) runs every frame:

- **ACTIVE**: when `TurnPhase.IsReady()` is true, call `TurnPhase.Sync()`.
- **SYNC**: when `TurnPhase.IsSynced()` is true, go to the next phase in `phaseOrder` whose `ShouldSkip()` is false. Wrapping from the last phase back to the first increments `turnCounter`.
  `ChangeTurnPhase` (`0x5F2C20`) calls `old.Exit()` then `new.Enter(false)` and flashes the taskbar if the window is not focused (not for a defeated player).

**The turn counter goes up when the Win phase wraps back to the Harvest phase.**

### Ready

`TurnPhase.IsReady` (`0xFDD8B0`):

- For every island: `ready = IsReady(island)`, and `Player.IsAwaited = !ready` (the "waiting for player" marker).
- False while the phase banner is still showing (`TimeSinceEnter < BannerDuration`, 2 s).
- Otherwise true when **all islands are ready**, or when the timer ran out (`TimeRemaining < 0`).
- An island is ready when its phase data has nothing to do, or the player pressed **Ready** (`PhaseData.Skip()` → `SkipRPC`, flag `HasSkipped`).

### Sync

`TurnPhase.Sync` (`0xFDEB40`):

1. Store `syncTime`; the server sends `SyncRPC(syncTime)` to all clients.
2. Server: for every island owned by an **AI**, call `AIPlay.PlayPhase(phaseData, aiState)`. **The AI makes its moves when the phase ends, not during it.**
3. `State = SYNC`, fire `GameState.onSyncPhase`.
4. Server: `ApplyChanges(done)`. Each phase overrides it to apply the planned actions of all players.

After every phase's `ApplyChanges`, `TurnPhase.Recompute` recomputes every island (storage capacity, defeat).

### Entering a phase

`TurnPhase.Enter` (`0xFDCD00`) runs on every machine. The server tells all clients to enter (`EnterRPC` with the start time), every island's `PhaseData.Enter` runs (it also sets
the `Preview` flag, see [Build phase](build-phase.md#preview-instant-or-at-sync)), and on clients the phase's interface is enabled. It also sets the time of day: each phase has a
`timeOfDay` field (all phases 0.125 = morning) that is written into the `DayNightCycle` target, see [Rendering](rendering.md#day-and-night).

## Clock

`ITurnPhase.Time` (`0x56D8D0`) is the clock all phases use. It returns `Cheats.NetworkTime()`: the network time multiplied by the developer `speed` factor.

## Start phase ("Settle")

`StartData.Reset` (`0x572C40`) fills the island's `BuildingsToPlace` list: **Castle, Cottage ×3, Farm, Dock**. `StartData.PlaceBuilding` (`0x5728C0`) places one of them if the normal tile rules
pass (`IslandGrid.CanPlaceBuilding`) and the type is still in the list; each listed type can be placed exactly as often as it appears.

There is no check for buildings that already exist on the island: an island loaded from an [island code](island/save-format.md) that already has a Castle still gets the start Castle, so the player
has two.

At the end (`StartPhase.ApplyChanges` `0x573370`), for every island:

1. Placed buildings are added (if they still fit).
2. **Every type still in `BuildingsToPlace` is placed automatically** at `IslandGrid.PlaceableBuilding(type)` (`0x6179E0`). This happens whether the timer ran out or everybody pressed Ready.
3. The island is recomputed and gets the start resources: **3 food, 3 wood, 2 stone, 4 hammers, 8 culture**.

`PlaceableBuilding` walks the island in a **spiral from the centre outwards** (the same spiral the AI uses), tries the 4 orientations without mirroring, and takes the first spot where
`CanPlaceBuilding` passes and the building does not raise disease risk (`Disease.DoesRiskMoreDisease`). So auto-placed buildings end up as close to the island centre as allowed.
(The AI places its farms far from the centre on purpose, see [AI](ai.md).)

## World layout

`GameState.SpawnIslands` (`0x5F3F70`) runs on the server at game start with the match random generator, so the layout is the same for everybody.

**Island positions** (one per player):

- `g` = island grid size (49 × 49 by default), `halfDiag` = √((g.x/2)² + (g.y/2)²) ≈ 34.6, `step` = halfDiag × 0.001.
- Starting at radius `r = step`, take a random point in an ellipse with radii (r × 16/9, r), turned 45°. Reject it if it overlaps an island already placed (|dx| ≤ g.x and |dy| ≤ g.y);
  otherwise `r` grows by `step` and it tries again. Islands are packed tightly, in a 16:9 oval turned 45°; more players make it wider.
- The points are sorted by angle around the origin and given to the players in list order, so neighbours in the player list are neighbours around the circle.

**Island shapes (fairness):** 256 islands are generated and sorted by score (natural resources + build space + cliffs). Only the **middle N** (N = number of players) are kept and shuffled,
so every player gets an island of about median quality. Generation itself: [Island generation](island/generation.md).

A layout mod can postfix `SpawnIslands` to move islands or replace grids, but it must run on the host so every client gets the same world.

## Snapshots and rejoin

`TurnPhase.TakeSnapshot` stores `(Island, IslandGrid.SyncData, Statistics.SyncData)` for every island when a phase is entered; `SnapshotStore` keeps them under `(player, turn, phase)`.
`GameState.Restore()`, `RestoreTurnPhase` and `Rejoin(Player)` let a player who reconnects rebuild the state (`PhaseData.RejoinClientRPC`).

## Training

`GameState.IsTraining` is the tutorial game (strings `training.information.*`).

## Code reference

Declarations from the IL2CPP dump of game version 1.0.4, trimmed to the members a patch usually needs. `// 0x..` is the field offset, `// RVA 0x..` the method address in `GameAssembly.dll` (in Ghidra: `0x180000000 + RVA`). Names are stable between updates, offsets and RVAs are not. Private members are public in the BepInEx interop assemblies.

### Where to patch

| To | Patch / use |
|---|---|
| Do something when a phase starts | postfix `GameState.ChangeTurnPhase` (private), or add to `GameState.onEnterPhase` |
| Do something after a phase's rules were applied | postfix `<Phase>.ApplyChanges` (for example `BuildPhase.ApplyChanges`) or `GameState.onSyncPhase` |
| Change phase time limits | `TurnPhase.timeLimit` / `timeLimitSync` on each phase object (`gameState.buildPhase` etc.) |
| Change the start buildings | postfix `StartData.Reset` and edit `BuildingsToPlace` |
| Change the start resources | `StartPhase.startResourceAmount` (private field, public in the interop) |
| Change where auto-placed buildings go | `IslandGrid.PlaceableBuilding` |
| Move or replace islands at match start (host) | postfix `GameState.SpawnIslands` |
| Press Ready from code | `gameState.TurnPhase.PhaseData(island).Skip()` |

### Declarations

```csharp
// DiceKingdoms.Game
public class GameState : NetworkedBehaviour
{
    public const int MAX_PLAYERS = 12;
    private LobbySettings lobbySettings;                                         // 0xD0
    public Random random;                                                        // 0xD8
    private List<Player> players;                                                // 0xE8
    private NetworkedList<ulong> playerIds;                                      // 0xF8
    public Action onIslandSpawned;                                               // 0x128
    public Action onStartGame;                                                   // 0x130
    public Action onEnterPhase;                                                  // 0x138
    public Action onSyncPhase;                                                   // 0x140
    public Action onEndGame;                                                     // 0x148
    public Action<string> onExit;                                                // 0x150
    public Action onReset;                                                       // 0x158
    public Action onRejoin;                                                      // 0x160
    public Action<Player, StructuredText> onDisasterLog;                         // 0x168
    private ITurnPhase[] phaseOrder;                                             // 0x170
    public StartPhase startPhase;                                                // 0x178
    public HarvestPhase harvestPhase;                                            // 0x180
    public DisastersPhase disastersPhase;                                        // 0x188
    public BuildPhase buildPhase;                                                // 0x190
    public CombatPhase combatPhase;                                              // 0x198
    public WinPhase winPhase;                                                    // 0x1A0
    private NetworkedVar<int> turnCounter;                                       // 0x1B8

    public List<Player> Players { get; }
    public int TurnCounter { get; set; }
    public ITurnPhase TurnPhase { get; set; }
    public bool IsTraining { get; set; }

    public Island Island(Player owner);                                          // RVA 0x5F2F60
    public void AddAI(Nullable<Player.AIState> aiState, Nullable<int> slotOpt);  // RVA 0x5F2000
    private void ChangeTurnPhase(ITurnPhase turnPhase);                          // RVA 0x5F2C20
    private void StartGame();                                                    // RVA 0x5F50E0
    private void UpdateTurnPhase();                                              // RVA 0x5F6E20
    public bool Restore();                                                       // RVA 0x5F33E0
    private void SpawnIslands();                                                 // RVA 0x5F3F70
}
```

```csharp
// DiceKingdoms.Game
public enum PhaseState
{
    ACTIVE = 0,
    SYNC = 1,
    INACTIVE = 2,
}
```

```csharp
// DiceKingdoms.Game
public interface ITurnPhase
{
    public abstract string Name { get; }
    public abstract float BannerDuration { get; }
    public abstract float TimeLimit { get; }
    public abstract PhaseState State { get; }
    public abstract float TimeSinceEnter { get; }
    public abstract float TimeRemaining { get; }
    public abstract float TimeSinceSync { get; }
    public abstract float TimeRemainingSync { get; }
    public abstract IPhaseInterface Interface { get; }
    public static float Time { get; }

    public abstract void Enter(bool restore);
    public abstract void Sync();
    public abstract void Exit();
    public abstract void Rejoin(Player player);
    public abstract IPhaseData PhaseData(Island island);
    public abstract bool IsReady(Island island);
    public abstract bool IsReady();
    public abstract bool IsSynced();
    public abstract bool ShouldSkip();
}
```

```csharp
// DiceKingdoms.Game
public abstract class TurnPhase<TData, TIslandState> : NetworkedBehaviour, ITurnPhase
{
    public float timeLimit;
    public float timeLimitSync;
    public float timeOfDay;

    public GameState GameState { get; set; }
    public float BannerDuration { get; }
    public PhaseState State { get; set; }
    public float TimeSinceEnter { get; }
    public float TimeRemaining { get; }

    public IPhaseData PhaseData(Island island);                      // RVA 0xFDE010 (shared generic body)
    public virtual void Enter(bool restore = False);                 // RVA 0xFDD3C0 (shared generic body)
    private void Enter(bool restore, Nullable<float> enterTimeOpt);  // RVA 0xFDCD00 (shared generic body)
    public void Sync();                                              // RVA 0xFDEB20 (shared generic body)
    private void Sync(Nullable<float> syncTimeOpt);                  // RVA 0xFDEB40 (shared generic body)
    public virtual void Exit();                                      // RVA 0xFDD420 (shared generic body)
    public virtual void Rejoin(Player player);                       // RVA 0xFDE3C0 (shared generic body)
    public bool IsReady();                                           // RVA 0xFDD8B0 (shared generic body)
    public bool IsSynced();                                          // RVA 0xFDDBA0 (shared generic body)
    public virtual bool ShouldSkip();                                // RVA 0x4A1F20 (shared generic body)
    public bool IsReady(Island island);                              // RVA 0xFDD780 (shared generic body)
    private void TakeSnapshot();                                     // RVA 0xFDF1D0 (shared generic body)
    private static void Recompute(Island island);                    // RVA 0xFDE2E0 (shared generic body)
    private static void RecomputeIsDefeated(Island island);          // RVA 0xFDE040 (shared generic body)
    protected abstract void ApplyChanges(Action done);
}
```

```csharp
// DiceKingdoms.Game
public abstract class PhaseData<TData, TState> : NetworkedBehaviour, IPhaseData
{
    public TurnPhase<TData, TState> turnPhase;

    public bool HasSkipped { get; set; }
    public Island Island { get; set; }
    public bool Preview { get; set; }
    public abstract bool NothingToDo { get; }

    private void SkipRPC();                     // RVA 0xE60860 (shared generic body)
    public void Skip();                         // RVA 0xE608B0 (shared generic body)
    public void Enter(bool restore);            // RVA 0xE60440 (shared generic body)
}
```

```csharp
// DiceKingdoms.Game
public class StartData : PhaseData<StartData, StartIslandState>
{
    public List<BuildingType> BuildingsToPlace { get; }
    public List<Building> PlacedBuildings { get; }
    public BuildingType NextBuildingType { get; }
    public override bool NothingToDo { get; }

    private void PlaceBuildingRPC(Building building);             // RVA 0x572860
    private void PlaceBuildingOtherClientRPC(Building building);  // RVA 0x572800
    public void PlaceBuilding(Building building);                 // RVA 0x5728C0
    protected override void Reset();                              // RVA 0x572C40
    protected override void OnRejoinClient();                     // RVA 0x572630
}
```

```csharp
// DiceKingdoms.Game
public class StartPhase : TurnPhase<StartData, StartIslandState>
{
    private ResourceAmount startResourceAmount;         // 0xE8

    public override void Reset();                       // RVA 0x475740
    protected override void ApplyChanges(Action done);  // RVA 0x573370
}
```
