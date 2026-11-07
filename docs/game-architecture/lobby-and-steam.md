---
summary: Main menu, custom games, matchmaking, the in-game lobby, match start, lobby settings, Steam lobby keys, rich presence, inventory, civilizations, item drops and the store
---

# Lobby, menus and Steam

!!! info "Code blocks"
    Declarations from the IL2CPP dump of game version 1.0.4, trimmed to what each section uses. `// 0x..` is the field offset, `// RVA 0x..` the method address in `GameAssembly.dll` (in Ghidra `0x180000000 + RVA`). Names stay stable between updates, offsets and RVAs do not. Private members are public in the BepInEx interop assemblies.

## Main menu

`MainMenu.Awake` (`0x532D60`):

| Button | Does |
|---|---|
| Training | solo match with the tutorial |
| Solo | hosts locally with your own inventory, no Steam lobby |
| Matchmaking | starts or cancels the search |
| Custom game | opens the custom game window (pre-fills a lobby code from the clipboard) |
| → Create | `SteamMatchmaking.CreateLobbyAsync(12)` and host it |
| → Join code | find the lobby with that code and join |
| → Join friend | opens the Steam friends overlay |
| Discord / Steam | open the Discord invite / the store page |
| About, Settings, Rules, Store | toggle that window |
| Exit | quit |

At start the menu syncs time with NTP servers (for lobby keep-alive timestamps) and leaves a Steam lobby it was still in.

`MainMenu.Update` runs every 0.1 s: friends, matchmaking, joining. While matchmaking or joining, Training, Solo and Custom game are disabled. Join errors: "Lobby not found.", "Inventory not found.",
"Failed retrieving inventory: ...", "Cannot join a lobby with a different version.", "Canceled.".

## Matchmaking

`UpdateMatchmaking` (`0x539F70`):

1. Every 1 s, query Steam lobbies with the same `version`, `matchmaking = True` and `progress = p` (preparing).
2. Drop lobbies that are not alive: `keep-alive` is an NTP timestamp, alive if within **10 s** of now.
3. None left: create an own invisible lobby (8 members) and wait in it.
4. Otherwise pick the best by **2 × members, +1 if I own it, −100 if not alive**, and switch only to a better one, so waiting players merge into the biggest lobby.

The owner starts the match when the members agree on a host (each member's `connected-to` data votes), or when the lobby has at least 2 members and either the wait is at least
**(6 − members) × 150 s** (2 players: 10 min, 5 players: 2.5 min, 6+: at once) or every member set `matchmaking_start_anyways`.

## Custom lobby

The owner sets the lobby to friends-only, 12 members, and writes `version`, `matchmaking`, `players` (`P<own SteamId>`), `progress = p` and `keep-alive`. The game scene loads at once;
the lobby is then the in-game **StartGame** panel:

- Rows: humans with Steam avatar and name, a Ready toggle (not for the host), host icon; AIs with name and difficulty drop-down (host only); colour and civilization; a "next colour" button
  (own row, only without teams) and a "next civilization" button (own row; the castle click, `Player.NextCivilization`); kick (host only, not in matchmaking).
- Teams on: rows are grouped under team headers and the host re-slots players per team.
- Buttons: lobby settings (only the host may edit), **Play** (host; enabled when every player is an AI, the host, or Ready), Add AI, Invite friends. All hidden in matchmaking.

## Match start

`GameState.UpdateInit` (`0x5F69F0`) → `StartGame` (`0x5F50E0`):

1. Spawn the islands ([World layout](turn-phases.md#world-layout)).
2. Wait until every human has loaded every island and all island meshes are generated.
3. `GameStarted = true`, turn counter = 1, `StartGameRPC`.
4. **Fewer than 2 human players → the timer is forced to UNLIMITED.**
5. `onStartGame`, then the host enters the start phase.

Leaving: the win screen's **Continue** keeps you as a spectator; "Back to menu" asks for confirmation (`information.back-to-menu`); closing the game while networking first shuts the
network down.

## Lobby settings

`LobbySettings : SettingSet`: `lobbyCode`, `timerSpeed` (FAST / SLOW / UNLIMITED), `multiMeteor`, `teams`, `wonderVictory`. Defaults: FAST, multi-meteor off, teams off, wonder victory on.
Synced to clients through `GameState.lobbySettingsNetworked`. The lobby code is 7 characters from `ABCDEFGHJKMPQRST23456789`.

The settings object:

<!-- code: LobbySettings | *fields -->
```csharp
namespace DiceKingdoms.Game;

public class LobbySettings : SettingSet
{
    public SettingSet.Setting<LobbySettings.LobbyCode> lobbyCode;    // 0x28
    public SettingSet.Setting<LobbySettings.TimerSpeed> timerSpeed;  // 0x30
    public SettingSet.Setting<bool> multiMeteor;                     // 0x38
    public SettingSet.Setting<bool> teams;                           // 0x40
    public SettingSet.Setting<bool> wonderVictory;                   // 0x48
}
```
<!-- /code -->

<!-- code: LobbySettings.TimerSpeed |  -->
```csharp
public enum LobbySettings.TimerSpeed
{
    FAST = 0,
    SLOW = 1,
    UNLIMITED = 2,
}
```
<!-- /code -->

## Steam lobby keys

`MainMenu.LobbyConsts`:

| Key | Values |
|---|---|
| `version` | game version (must match to join) |
| `players` | `AI<name>` / `P<SteamId>` entries |
| `progress` | `p` preparing, `r` running, `e` ended |
| `keep-alive` | heartbeat timestamp written by the owner |
| `connected-to` | host vote (member data) |
| `matchmaking`, `matchmaking_start_anyways`, `matchmaking_fill_ai` | quick-match flags |
| `member` / `player_<SteamId>` | per-player value |
| `code` | lobby code |

**Rich presence** (`RichPresenceConsts`): `connect`, `status` = MAIN_MENU / SEARCHING / IN_LOBBY / IN_GAME, `game_type` = training / solo / quick-match / custom-game, `turn`,
`steam_display`, `steam_player_group(_size)`.

## Inventory and civilizations

Steam item definitions:

The Steam inventory of a player:

<!-- code: PlayerInventory | *fields, *props, *methods -->
```csharp
namespace DiceKingdoms.Game;

public struct PlayerInventory
{
    public static readonly PlayerInventory FALLBACK;                      // static

    public Civilization[] UnlockedCivilizations { get; }

    public static PlayerInventory FromSteamworks(InventoryItem[] items);  // RVA 0x56E170
}
```
<!-- /code -->

<!-- code: Player | NextCivilization -->
```csharp
namespace DiceKingdoms.Game;

public class Player : NetworkedBehaviour
{
    public void NextCivilization();  // RVA 0x570500
}
```
<!-- /code -->

**Patch:** postfix `PlayerInventory.FromSteamworks` to add civilizations for local tests (solo, or when you host).

| Id | Name | Meaning |
|---|---|---|
| 11001 | GOLD | in-game currency item |
| 11002 | GOLD_PACK | a pack of gold |
| 12002 | GOLD_PACK_BUNDLE | a bundle of packs |

Item properties set on Steam's side: `dice_kingdoms_unlocks_civilization` (a civilization name) and `dice_kingdoms_drop_when` (`GAME_START` or `QUICK_MATCH_VICTORY`).

- `PlayerInventory.FromSteamworks` (`0x56E170`): every owned item with the unlock property gives its civilization (`UnlockedCivilizations`). Nothing → `FALLBACK` = [EUROPE], so Europe is
  always available. The demo build always uses the fallback.
- **Joiners:** the client sends its serialized Steam inventory as connection data; the **host** deserializes it (`SteamInventory.DeserializeAsync`) and rejects it if expired or if it
  belongs to someone else. So a client cannot claim civilizations it does not own; a modified host can give its own players anything.
- At spawn (`Player.NetworkStart` `0x56FBA0`, host): colour = random among the free ones, **civilization = random among the unlocked ones**, start slot = first free of 0..11.
- In the lobby, `Player.NextCivilization` (`0x570500`, host, before the start) cycles through the unlocked civilizations. See [Rules and civilizations](rules-and-civilizations.md) for
  what not to patch.

## Item drops and the store

- **Game start drops** (`MainMenu.CheckManualDrops`): eligible `GAME_START` items are requested with `SteamInventory.AddPromoItemAsync`; Steam decides if they are granted.
- **Quick-match victory drops** (`RegisterWinReward`): only when the match lasted **at least 5 turns** and the **winner is on your team**. The gold gained (item 11001 before/after) is
  shown on the win screen.
- `Store.CanBeBought(item)`: the item has a price (bought in the Steam overlay) or a Steam exchange recipe (bought with gold). Purchases are done by Steam.

Drops and gold come from Steam's servers; a mod cannot create them. For local testing a postfix on `PlayerInventory.FromSteamworks` can add civilizations in solo games and when you host.
