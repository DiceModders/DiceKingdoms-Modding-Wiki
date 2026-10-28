---
summary: How a match runs - GameState, the phase loop, phase states, ready and sync, timers, the start phase, world layout, snapshots and rejoin
---

# Turn loop and phases

!!! info "Code blocks"
    Declarations from the IL2CPP dump of game version 1.0.4, trimmed to what each section uses. `// 0x..` is the field offset, `// RVA 0x..` the method address in `GameAssembly.dll` (in Ghidra `0x180000000 + RVA`). Names stay stable between updates, offsets and RVAs do not. Private members are public in the BepInEx interop assemblies.

## Main objects

<!-- code: GameState | random, players, playerIds, lobbySettings, turnCounter, onIslandSpawned, onStartGame, onEnterPhase, onSyncPhase, onEndGame, onExit, onReset, onRejoin, onDisasterLog, Players, TurnCounter, TurnPhase, IsTraining, Island -->
```csharp
// DiceKingdoms.Game
public class GameState : NetworkedBehaviour
{
    private LobbySettings lobbySettings;                  // 0xD0
    public Random random;                                 // 0xD8
    private List<Player> players;                         // 0xE8
    private NetworkedList<ulong> playerIds;               // 0xF8
    public Action onIslandSpawned;                        // 0x128
    public Action onStartGame;                            // 0x130
    public Action onEnterPhase;                           // 0x138
    public Action onSyncPhase;                            // 0x140
    public Action onEndGame;                              // 0x148
    public Action<string> onExit;                         // 0x150
    public Action onReset;                                // 0x158
    public Action onRejoin;                               // 0x160
    public Action<Player, StructuredText> onDisasterLog;  // 0x168
    private NetworkedVar<int> turnCounter;                // 0x1B8

    public List<Player> Players { get; }
    public int TurnCounter { get; set; }
    public ITurnPhase TurnPhase { get; set; }
    public bool IsTraining { get; set; }

    public Island Island(Player owner);                   // RVA 0x5F2F60
}
```
<!-- /code -->

**Patch:** subscribe to the `GameState` events (`onEnterPhase`, `onSyncPhase`, …) or postfix the methods below; `gameState.Island(player)` gives a player's island.

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

<!-- code: GameState | phaseOrder, startPhase, harvestPhase, disastersPhase, buildPhase, combatPhase, winPhase -->
```csharp
// DiceKingdoms.Game
public class GameState : NetworkedBehaviour
{
    private ITurnPhase[] phaseOrder;       // 0x170
    public StartPhase startPhase;          // 0x178
    public HarvestPhase harvestPhase;      // 0x180
    public DisastersPhase disastersPhase;  // 0x188
    public BuildPhase buildPhase;          // 0x190
    public CombatPhase combatPhase;        // 0x198
    public WinPhase winPhase;              // 0x1A0
}
```
<!-- /code -->

<!-- code: TurnPhase | timeLimit, timeLimitSync, timeOfDay -->
```csharp
// DiceKingdoms.Game
public abstract class TurnPhase<TData, TIslandState> : NetworkedBehaviour, ITurnPhase
{
    public float timeLimit;
    public float timeLimitSync;
    public float timeOfDay;
}
```
<!-- /code -->

**Patch:** change `timeLimit` / `timeLimitSync` on a phase object (`gameState.buildPhase.timeLimit = 90`) to change its time.

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

<!-- code: PhaseState |  -->
```csharp
// DiceKingdoms.Game
public enum PhaseState
{
    ACTIVE = 0,
    SYNC = 1,
    INACTIVE = 2,
}
```
<!-- /code -->

<!-- code: GameState | UpdateTurnPhase, ChangeTurnPhase -->
```csharp
// DiceKingdoms.Game
public class GameState : NetworkedBehaviour
{
    private void ChangeTurnPhase(ITurnPhase turnPhase);  // RVA 0x5F2C20
    private void UpdateTurnPhase();                      // RVA 0x5F6E20
}
```
<!-- /code -->

**Patch:** postfix `GameState.ChangeTurnPhase` (private) to run code when a phase starts; its argument is the new `ITurnPhase`.

`GameState.UpdateTurnPhase` (`0x5F6E20`) runs every frame:

- **ACTIVE**: when `TurnPhase.IsReady()` is true, call `TurnPhase.Sync()`.
- **SYNC**: when `TurnPhase.IsSynced()` is true, go to the next phase in `phaseOrder` whose `ShouldSkip()` is false. Wrapping from the last phase back to the first increments `turnCounter`.
  `ChangeTurnPhase` (`0x5F2C20`) calls `old.Exit()` then `new.Enter(false)` and flashes the taskbar if the window is not focused (not for a defeated player).

**The turn counter goes up when the Win phase wraps back to the Harvest phase.**

### Ready

`TurnPhase.IsReady` (`0xFDD8B0`):

<!-- code: TurnPhase | IsReady, BannerDuration, TimeSinceEnter, TimeRemaining -->
```csharp
// DiceKingdoms.Game
public abstract class TurnPhase<TData, TIslandState> : NetworkedBehaviour, ITurnPhase
{
    public float BannerDuration { get; }
    public float TimeSinceEnter { get; }
    public float TimeRemaining { get; }

    public bool IsReady();               // RVA 0xFDD8B0 (shared generic body)
    public bool IsReady(Island island);  // RVA 0xFDD780 (shared generic body)
}
```
<!-- /code -->

<!-- code: PhaseData | HasSkipped, NothingToDo, Skip, SkipRPC -->
```csharp
// DiceKingdoms.Game
public abstract class PhaseData<TData, TState> : NetworkedBehaviour, IPhaseData
{
    public bool HasSkipped { get; set; }
    public abstract bool NothingToDo { get; }

    private void SkipRPC();  // RVA 0xE60860 (shared generic body)
    public void Skip();      // RVA 0xE608B0 (shared generic body)
}
```
<!-- /code -->

**Patch:** `gameState.TurnPhase.PhaseData(island).Skip()` presses Ready from code.

- For every island: `ready = IsReady(island)`, and `Player.IsAwaited = !ready` (the "waiting for player" marker).
- False while the phase banner is still showing (`TimeSinceEnter < BannerDuration`, 2 s).
- Otherwise true when **all islands are ready**, or when the timer ran out (`TimeRemaining < 0`).
- An island is ready when its phase data has nothing to do, or the player pressed **Ready** (`PhaseData.Skip()` → `SkipRPC`, flag `HasSkipped`).

### Sync

`TurnPhase.Sync` (`0xFDEB40`):

<!-- code: TurnPhase | Sync, IsSynced, ApplyChanges, ShouldSkip, Recompute, RecomputeIsDefeated -->
```csharp
// DiceKingdoms.Game
public abstract class TurnPhase<TData, TIslandState> : NetworkedBehaviour, ITurnPhase
{
    public void Sync();                                      // RVA 0xFDEB20 (shared generic body)
    private void Sync(Nullable<float> syncTimeOpt);          // RVA 0xFDEB40 (shared generic body)
    public bool IsSynced();                                  // RVA 0xFDDBA0 (shared generic body)
    public virtual bool ShouldSkip();                        // RVA 0x4A1F20 (shared generic body)
    private static void Recompute(Island island);            // RVA 0xFDE2E0 (shared generic body)
    private static void RecomputeIsDefeated(Island island);  // RVA 0xFDE040 (shared generic body)
    protected abstract void ApplyChanges(Action done);
}
```
<!-- /code -->

**Patch:** postfix a phase's `ApplyChanges` (for example `BuildPhase.ApplyChanges`) to run after its rules were applied. `TurnPhase` is generic: its methods share one compiled body, so patch the concrete override (`ApplyChanges`) rather than the shared ones.

1. Store `syncTime`; the server sends `SyncRPC(syncTime)` to all clients.
2. Server: for every island owned by an **AI**, call `AIPlay.PlayPhase(phaseData, aiState)`. **The AI makes its moves when the phase ends, not during it.**
3. `State = SYNC`, fire `GameState.onSyncPhase`.
4. Server: `ApplyChanges(done)`. Each phase overrides it to apply the planned actions of all players.

After every phase's `ApplyChanges`, `TurnPhase.Recompute` recomputes every island (storage capacity, defeat).

### Entering a phase

`TurnPhase.Enter` (`0xFDCD00`) runs on every machine. The server tells all clients to enter (`EnterRPC` with the start time), every island's `PhaseData.Enter` runs (it also sets
the `Preview` flag, see [Build phase](build-phase.md#preview-instant-or-at-sync)), and on clients the phase's interface is enabled. It also sets the time of day: each phase has a
`timeOfDay` field (all phases 0.125 = morning) that is written into the `DayNightCycle` target, see [Rendering](rendering.md#day-and-night).

<!-- code: TurnPhase | Enter, Exit, Rejoin -->
```csharp
// DiceKingdoms.Game
public abstract class TurnPhase<TData, TIslandState> : NetworkedBehaviour, ITurnPhase
{
    public virtual void Enter(bool restore = False);                 // RVA 0xFDD3C0 (shared generic body)
    private void Enter(bool restore, Nullable<float> enterTimeOpt);  // RVA 0xFDCD00 (shared generic body)
    public virtual void Exit();                                      // RVA 0xFDD420 (shared generic body)
    public virtual void Rejoin(Player player);                       // RVA 0xFDE3C0 (shared generic body)
}
```
<!-- /code -->

<!-- code: PhaseData | turnPhase, Island, Preview, Enter -->
```csharp
// DiceKingdoms.Game
public abstract class PhaseData<TData, TState> : NetworkedBehaviour, IPhaseData
{
    public TurnPhase<TData, TState> turnPhase;

    public Island Island { get; set; }
    public bool Preview { get; set; }

    public void Enter(bool restore);            // RVA 0xE60440 (shared generic body)
}
```
<!-- /code -->

## Clock

`ITurnPhase.Time` (`0x56D8D0`) is the clock all phases use. It returns `Cheats.NetworkTime()`: the network time multiplied by the developer `speed` factor.

<!-- code: ITurnPhase | Time, TimeLimit, TimeRemaining, TimeRemainingSync -->
```csharp
// DiceKingdoms.Game
public interface ITurnPhase
{
    public abstract float TimeLimit { get; }
    public abstract float TimeRemaining { get; }
    public abstract float TimeRemainingSync { get; }
    public static float Time { get; }
}
```
<!-- /code -->

## Start phase ("Settle")

`StartData.Reset` (`0x572C40`) fills the island's `BuildingsToPlace` list: **Castle, Cottage ×3, Farm, Dock**. `StartData.PlaceBuilding` (`0x5728C0`) places one of them if the normal tile rules
pass (`IslandGrid.CanPlaceBuilding`) and the type is still in the list; each listed type can be placed exactly as often as it appears.

<!-- code: StartData | BuildingsToPlace, PlacedBuildings, PlaceBuilding, Reset -->
```csharp
// DiceKingdoms.Game
public class StartData : PhaseData<StartData, StartIslandState>
{
    public List<BuildingType> BuildingsToPlace { get; }
    public List<Building> PlacedBuildings { get; }

    public void PlaceBuilding(Building building);  // RVA 0x5728C0
    protected override void Reset();               // RVA 0x572C40
}
```
<!-- /code -->

<!-- code: StartPhase | startResourceAmount, ApplyChanges -->
```csharp
// DiceKingdoms.Game
public class StartPhase : TurnPhase<StartData, StartIslandState>
{
    private ResourceAmount startResourceAmount;         // 0xE8

    protected override void ApplyChanges(Action done);  // RVA 0x573370
}
```
<!-- /code -->

<!-- code: IslandGrid | PlaceableBuilding -->
```csharp
// DiceKingdoms.Game
public class IslandGrid : IslandSyncedBehaviour<IslandGrid.SyncData>
{
    public Nullable<Building> PlaceableBuilding(BuildingType buildingType);  // RVA 0x6179E0
}
```
<!-- /code -->

**Patch:** postfix `StartData.Reset` to change `BuildingsToPlace`; change `StartPhase.startResourceAmount` for other start resources.

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

<!-- code: GameState | SpawnIslands, StartGame -->
```csharp
// DiceKingdoms.Game
public class GameState : NetworkedBehaviour
{
    private void StartGame();     // RVA 0x5F50E0
    private void SpawnIslands();  // RVA 0x5F3F70
}
```
<!-- /code -->

**Patch:** postfix `GameState.SpawnIslands` on the host to move islands or replace their grids.

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

<!-- code: TurnPhase | TakeSnapshot -->
```csharp
// DiceKingdoms.Game
public abstract class TurnPhase<TData, TIslandState> : NetworkedBehaviour, ITurnPhase
{
    private void TakeSnapshot();  // RVA 0xFDF1D0 (shared generic body)
}
```
<!-- /code -->

<!-- code: GameState | Restore -->
```csharp
// DiceKingdoms.Game
public class GameState : NetworkedBehaviour
{
    public bool Restore();  // RVA 0x5F33E0
}
```
<!-- /code -->

## Training

`GameState.IsTraining` is the tutorial game (strings `training.information.*`).
