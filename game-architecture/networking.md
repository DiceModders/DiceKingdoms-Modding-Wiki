---
summary: Host-authoritative MLAPI over Steam P2P - what runs where, every client-to-server RPC, shared state, join approval, player ids, reconnects, and what this means for mods
---

# Networking

!!! info "Code blocks"
    Declarations from the IL2CPP dump of game version 1.0.4, trimmed to what each section uses. `// 0x..` is the field offset, `// RVA 0x..` the method address in `GameAssembly.dll` (in Ghidra `0x180000000 + RVA`). Names stay stable between updates, offsets and RVAs do not. Private members are public in the BepInEx interop assemblies.

## Model

- **Host-authoritative MLAPI over Steam P2P.** The host (lobby owner) is the MLAPI server and also a client; the other players are clients.
- Clients **only send intents**. The server checks them, applies the rules, rolls every random number and sends the results (ClientRPCs and `NetworkedVar`s). Phase logic (`TurnPhase.Update`,
  `Sync`, `ApplyChanges`) and the [AI](ai.md) run only where `IsServer` is true.
- Random results are either rolled on the server (dice, disaster selection) or come from seeds the server sends (disaster occurrences, the per-island generator, combat).
- [Combat](combat.md) is simulated **on every client** from the same deployment list with software floats and a fixed tick, and a per-tick hash is logged to catch desyncs.
- There is no separate anti-cheat. `Utility.AntiCheat.EncryptedStorage` only hides the resource values in memory ([Harvest](harvest.md#encrypted-storage)).

!!! warning "Mods and multiplayer"
    A rule change must run on the **host** to take effect for everybody. A change made only on a client changes that client's display, and it can desync the combat simulation,
    which every client runs itself.

## Client → server RPCs

The `Player` RPCs and their public wrappers:

<!-- code: Player | NextColorRPC, NextCivilizationRPC, JoinStartGameSlotRPC, AcknowledgeRPC, ChatRPC, Chat, Acknowledge -->
```csharp
namespace DiceKingdoms.Game;

public class Player : NetworkedBehaviour
{
    private void NextColorRPC();                                   // RVA 0x5706E0
    private void NextCivilizationRPC();                            // RVA 0x5704F0
    private void JoinStartGameSlotRPC(int slot);                   // RVA 0x56F9F0
    public void Acknowledge(Acknowledgeable acknowledgeable);      // RVA 0x56F270
    private void AcknowledgeRPC(Acknowledgeable acknowledgeable);  // RVA 0x56F240
    public void Chat(string message);                              // RVA 0x56F550
    private void ChatRPC(string message);                          // RVA 0x56F390
}
```
<!-- /code -->

| Class | RPC | Purpose |
|---|---|---|
| `Player` | `NextColorRPC`, `NextCivilizationRPC`, `JoinStartGameSlotRPC(int)` | lobby choices |
| `Player` | `AcknowledgeRPC(Acknowledgeable)` | "I finished watching" (harvest, disasters and combat animations) |
| `Player` | `ChatRPC(string)` | chat |
| `PhaseData<,>` | `SkipRPC` | Ready |
| `StartData` | `PlaceBuildingRPC(Building)` | start buildings |
| `HarvestData` | `ReRollRPC(Building.Id[])` | re-roll dice |
| `BuildData` | `PlanActionRPC(BuildData.Action)` | every build action |
| `CombatData` | `PlanAttackRPC(AttackPlan)` | attack plans |

There are 17 ClientRPCs in `DiceKingdoms.Game` (results, echoes, enter / sync / exit, win, rejoin).

## Shared state

- `NetworkedVar` / `NetworkedList` on `GameState` (player ids, lobby settings, teams, turn counter) and `Player` (ready, AI state, defeated, awaited, ping, colour, skins, slot).
- `IslandSyncedBehaviour<TSync>` (base of `IslandGrid` and `Statistics`) syncs full state with `SyncData` structs: `IslandGrid.SyncData` is the [island code](island/save-format.md) format,
  `Statistics.SyncData` = storage + technologies.
- `DiceKingdoms.Network.*Serialization` components register MLAPI serializers for every game type. `NetRef<T>` is a network id reference; `NetworkedObjectsOrdering` keeps the spawn order stable;
  `Acknowledgeable` waits until every player acknowledged.

## Player ids

The id field and the members that use it:

<!-- code: Player | initialOwnerId, InitialOwnerId, IsInitialOwner, LocalPlayer, IsAI, UserName, UpdateUserName, NetworkStart -->
```csharp
namespace DiceKingdoms.Game;

public class Player : NetworkedBehaviour
{
    private readonly NetworkedVar<ulong> initialOwnerId;  // 0xC8

    public ulong InitialOwnerId { get; }
    public bool IsInitialOwner { get; }
    public bool IsAI { get; }
    public static Player LocalPlayer { get; set; }
    public string UserName { get; set; }

    public override void NetworkStart();                  // RVA 0x56FBA0
    public void UpdateUserName();                         // RVA 0x570DC0
}
```
<!-- /code -->

**Patch:** `Player.LocalPlayer` is you; the host is the player with `InitialOwnerId == gameState.OwnerClientId`.

- The MLAPI client id is the transport's id for a connection. With the game's Steam transport this is the player's **SteamId** (64 bit).
- `Player.initialOwnerId` (`NetworkedVar<ulong>` `0xC8`) is set by the host in `Player.NetworkStart` and never changes: humans = their client id, **AIs = fake ids counting down from
  `ulong.MaxValue`**. It is the player's identity for the whole match, also after a reconnect. `Player.LocalPlayer` = the player with `initialOwnerId == LocalClientId`.

Places that need a **real Steam id** (they call Steam with it):

| Where | What |
|---|---|
| Inventory check (host, on join) | `InventoryResult.BelongsTo(clientId)`; fails with "Inventory belongs to someone else." |
| Player name (`Player.UpdateUserName` `0x570DC0`) | AI: "<colour> <difficulty>" (for example "Red Peasant"); human: the Steam name, unless the console's cinematic mode is on (then the colour name) |
| Lobby list | Steam avatar, friend state, rich presence (`FriendVisual`) |
| Steam lobby data | `players` (`AI<name>` / `P<id>`), `player_<SteamId>` keys at game start, member data `matchmaking_fill_ai` |
| Main menu | friends list, invites, join requests, rich presence |

Places where it is **only an identity** (any stable, unique number works): join approval, reconnect and rejoin (players are matched by `InitialOwnerId`), disconnects, kicks, the host
marker (`InitialOwnerId == GameState.OwnerClientId`), per-player lookups.

A different transport (players without Steam) would need: a **stable** id per player that survives reconnects, kept out of the Steam id range and away from the AI ids; and replacements for the
Steam-only places above (inventory, name, avatar, lobby data).

## Joining

`GameState.StartNetworking` sets up MLAPI connection approval (`<StartNetworking>b__69_5`):

- Game started (or islands spawned): approve **only** a client that already has a `Player` (`InitialOwnerId == clientId`) → rejoin; otherwise reject.
- Before the start: reject if 12 players already; otherwise approve and create the player (`OnJoin`), loading the joiner's Steam inventory from the connection data
  (see [Lobby and Steam](lobby-and-steam.md#inventory-and-civilizations)).
- Disconnect before the start removes the player; after the start the player is kept and can rejoin.
- Version gate: `Cannot join a lobby with a different version. Local=... Remote=...` (lobby key `version`).

`AutoHost` (`NetworkingManager.StartHost()` in `Start`) is a leftover developer component, not placed in the shipped scenes.

## Reconnect

`TurnPhase.TakeSnapshot` stores every island's state when a phase starts. A reconnecting player is matched by `InitialOwnerId` and rebuilt from the snapshots
(`GameState.Restore`, `RestoreTurnPhase`, `TurnPhase.Rejoin`, `PhaseData.Rejoin`, `CombatData.Rejoin`, `DisastersPhase.Rejoin`, `IslandSyncedBehaviour.RestoreSnapshot`).
