---
summary: The built-in developer console - when cheats are available, every command, debug keys (F1, F2, F3) and other hidden shortcuts
---

# Developer console and debug keys

*Code: [where to patch and the declarations](#code-reference).*

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

## Code reference

Declarations from the IL2CPP dump of game version 1.0.4, trimmed to the members a patch usually needs. `// 0x..` is the field offset, `// RVA 0x..` the method address in `GameAssembly.dll` (in Ghidra: `0x180000000 + RVA`). Names are stable between updates, offsets and RVAs are not. Private members are public in the BepInEx interop assemblies.

### Where to patch

| To | Patch / use |
|---|---|
| Unlock the console everywhere | force `Cheats.get_Available` / `CheckAvailable` to return true |
| Use a cheat from code | set the field on `Cheats.instance` (`rich`, `defLimit`, `speedFactor`, ...) |

### Declarations

```csharp
// DiceKingdoms.Meta
public class Cheats : MonoBehaviour
{
    private Technologies technologies;                                 // 0x18
    private Disasters disasters;                                       // 0x20
    private static Cheats instance;                                    // static
    private GameState gameState;                                       // 0x28
    private int defLimit;                                              // 0x30
    private bool enabledDisasters;                                     // 0x34
    private Disaster forcedDisaster;                                   // 0x38
    private bool rich;                                                 // 0x40
    private float speedFactor;                                         // 0x44
    private float networkTimeReference;                                // 0x48
    private float scaledNetworkTimeReference;                          // 0x4C

    private bool Available { get; }

    protected void Awake();                                            // RVA 0x528BB0
    protected void OnDestroy();                                        // RVA 0x52A1B0
    private void OnSceneLoaded(Scene scene, LoadSceneMode mode);       // RVA 0x52A2D0
    private void Reload();                                             // RVA 0x52A2E0
    private void Reset();                                              // RVA 0x52A3F0
    protected void Update();                                           // RVA 0x52A800
    private static bool CheckAvailable();                              // RVA 0x528D40
    private void ResetDefLimit();                                      // RVA 0x52A390
    private void UpdateDefLimit();                                     // RVA 0x475740
    private void DoDefLimit(int value);                                // RVA 0x4965C0
    private static void CommandDefLimit(int value);                    // RVA 0x528E20
    private static void FrontCommandDefLimit(CommandArg[] args);       // RVA 0x5299B0
    public static int DefLimit();                                      // RVA 0x5293B0
    private void ResetDisasters();                                     // RVA 0x52A3A0
    private void UpdateDisasters();                                    // RVA 0x52A420
    private void DoDisasters(bool active);                             // RVA 0x529400
    private static void CommandDisasters(bool active);                 // RVA 0x528E70
    private static void FrontCommandDisasters(CommandArg[] args);      // RVA 0x529A30
    private void ResetForceDisaster();                                 // RVA 0x52A3B0
    private void UpdateForceDisaster();                                // RVA 0x475740
    private void DoForceDisaster(string name);                         // RVA 0x529410
    private static void CommandForceDisaster(string name);             // RVA 0x528EC0
    private static void FrontCommandForceDisaster(CommandArg[] args);  // RVA 0x529AB0
    public static Disaster ForcedDisaster();                           // RVA 0x529960
    private void DoLearnAll();                                         // RVA 0x529520
    private static void CommandLearnAll();                             // RVA 0x529000
    private static void FrontCommandLearnAll(CommandArg[] args);       // RVA 0x529C20
    private void ResetLoadIsland();                                    // RVA 0x475740
    private void UpdateLoadIsland();                                   // RVA 0x475740
    private void DoLoadIsland();                                       // RVA 0x529670
    private static void CommandLoadIsland();                           // RVA 0x529050
    private static void FrontCommandLoadIsland(CommandArg[] args);     // RVA 0x529C80
    public static int LoadIsland();                                    // RVA 0x52A0C0
    private void ResetRich();                                          // RVA 0x52A3D0
    private void UpdateRich();                                         // RVA 0x52A5E0
    private void DoRich(bool active);                                  // RVA 0x476AA0
    private static void CommandRich(bool active);                      // RVA 0x5291C0
    private static void FrontCommandRich(CommandArg[] args);           // RVA 0x529E10
    private float DoNetworkTime();                                     // RVA 0x5297B0
    private void ResetSpeed();                                         // RVA 0x52A3E0
    private void UpdateSpeed();                                        // RVA 0x475740
    private void DoSpeed(float factor);                                // RVA 0x529820
    private static void CommandSpeed(float factor);                    // RVA 0x529210
    private static void FrontCommandSpeed(CommandArg[] args);          // RVA 0x529E90
    public static float NetworkTime();                                 // RVA 0x52A110
    private void UpdateTimer();                                        // RVA 0x475740
    private void ResetTimer();                                         // RVA 0x475740
    private void DoTimer(bool value);                                  // RVA 0x5298F0
    private static void CommandTimer(bool value);                      // RVA 0x529310
    private static void FrontCommandTimer(CommandArg[] args);          // RVA 0x529FE0
}
```

```csharp
// DiceKingdoms.Meta
public static class MiscCommands
{
    public static bool Cinematic { get; set; }
    public static bool FreeCam { get; set; }
    public static bool DebugBuild { get; set; }
    public static bool DebugAttack { get; set; }
    public static bool DebugLobbies { get; set; }
}
```
