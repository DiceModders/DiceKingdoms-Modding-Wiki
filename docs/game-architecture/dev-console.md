---
summary: The built-in developer console - when cheats are available, every command, debug keys (F1, F2, F3) and other hidden shortcuts
---

# Developer console and debug keys

The game ships **CommandTerminal** (`Terminal : MonoBehaviour`, commands registered with `[RegisterCommand]`). The commands are in `DiceKingdoms.Meta.Cheats` and `MiscCommands`.

## When cheats work

`Cheats.get_Available` (`0x52A840`):

```
Available = Cheats.gameState != null && gameState.players.Count(p => !p.IsAI) < 2
```

So the console commands work **in a match with at most one human** (you plus any number of AIs). Not in the main menu, not with two or more humans. There is no developer-build,
Steam-id or setting check. `CheckAvailable` (`0x528D40`) logs "Cheats unavailable." otherwise. Forcing `get_Available` / `CheckAvailable` to return true unlocks them everywhere.

Each command has a `Front…` handler (parses the arguments) and a `Command…` handler; both only write a field of the local `Cheats` object (`defLimit` `0x30`, `enabledDisasters` `0x34`,
`forcedDisaster` `0x38`, `rich` `0x40`, `speedFactor` `0x44`), which the game logic then reads. There is no network message: it works match-wide only because the single human is the host.

## Commands

| Command | Effect |
|---|---|
| `deflimit <n>` | limits the number of defenders (`Cheats.DefLimit`, default 999) |
| `disasters <bool>` | disasters on / off |
| `forcedisaster <name>` | the chosen disaster hits the island itself (used in `DisastersPhase.RandomOccurrences`) |
| `learnall` | learns every technology |
| `loadisland` | loads an island from the clipboard ([island code](island/save-format.md)) |
| `rich <bool>` | unlimited resources: storage is refilled to capacity continuously |
| `speed <factor>` | multiplies time (`Cheats.NetworkTime`) |
| `timer <bool>` | phase timer on / off |
| cinematic / freecam (`MiscCommands`) | smooth camera and anonymous players (colour names instead of Steam names); free camera also ignores the [disaster camera lock](disasters.md#playback-and-the-camera) |
| debug build / attack / lobbies (`MiscCommands`) | debug output (every build action logged, AI attack scoring, lobby info) |
| net order | prints the networked object order |
| save island | copies your island code to the clipboard (`IslandGrid.SaveIslandToClipboard`) |

## Keys

| Key | What |
|---|---|
| F1 | toggle the developer console |
| Shift+F1 | console in full-size mode |
| F2 | performance monitor (Tayx Graphy) on / off |
| Ctrl+F3 | Graphy: next display mode (also counts as F3) |
| F3 | hide / show the whole HUD; hiding it also hides and locks the mouse cursor |
| Tab | scoreboard |
| R / T | rotate / mirror the building being placed |
| Shift | keep placing |
| Shift / Ctrl + send arrow | send 5 / 10 resources instead of 1 (scoreboard) |

Key bindings (except the debug keys) can be changed in the settings; see [Game UI](ui.md).
