---
summary: ModBridge, the plugin that shows ModCore's lobby rules and civilizations on UiKit's screens; who depends on what
---

# ModBridge and the three libraries

ModBridge is a small plugin that connects [ModCore](modcore.md) (data) with [UiKit](uikit.md) (screens). It has **no API of its own**: install it and the lobby settings box shows the lobby rules and the
Rules screen lists the registered civilizations.

!!! info "Source code"
    [DiceKingdoms-ModBridge](https://github.com/DiceModders/DiceKingdoms-ModBridge) (plugin, README, `docs/FIRST_MOD.md` tutorial).

| Library | Knows about | Does not know about |
|---|---|---|
| **UiKit** | Screens, windows, buttons, settings rows, assets | The lobby, rules, civilizations |
| **ModCore** | Steam lobby, lobby rules, compatibility, civilizations | Any screen |
| **ModBridge** | Both of them | Nothing else |

- **Lobby rules:** every rule registered with `ModCore.Lobby.Rules` gets a row in the lobby's settings box (section "Rules", grouped per mod): a checkbox for a toggle, the game's own slider for a number.
  The host edits the rows before the match; everybody else sees the host's value. Under the rules the compatibility warnings are listed.
- **Civilizations:** every civilization registered with `ModCore.Civilizations.Add` gets an entry in the Rules screen's civilization drop-down.

## Who needs what

| | Mod author | Player |
|---|---|---|
| Window, top bar button, settings row | reference UiKit | UiKit.dll |
| Lobby rule, civilization, lobby state | reference ModCore | ModCore.dll (ModBridge for the screens) |
| Anything | **never** reference ModBridge, never bundle a library's DLL (`Private=false`) | all DLLs the mods list; a mod manager installs them as dependencies |

A mod that uses ModCore does not depend on ModBridge: without it the rules still work, they just have no rows. List ModBridge as an optional dependency in your manifest.
