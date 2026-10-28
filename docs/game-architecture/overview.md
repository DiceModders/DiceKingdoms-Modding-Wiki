---
summary: What the game is built with - engine, libraries, assemblies, namespaces, scenes, settings files - and the conventions used on the other pages
---

# Overview

## Engine and libraries

- Unity **2021.3.36f1**, **HDRP** (High Definition Render Pipeline), **IL2CPP**, Windows x64. Burst and the Job System are used for island generation and combat pathfinding.
- **Networking: MLAPI** (the old Unity Netcode, `MLAPI.Runtime.dll`): `NetworkedBehaviour`, `NetworkedVar<T>`, `NetworkedList<T>`, `[ServerRPC]` / `[ClientRPC]`. The transport is **Steam P2P**.
  See [Networking](networking.md).
- **Steam:** `Fivemid.FiveSteam.dll`, a fork of Facepunch.Steamworks (namespaces `Steamworks`, `Steamworks.Data`, `Steamworks.Ugc`, `Steamworks.ServerList`) plus wrappers in `Utility.Steamworks`.
- `Fivemid.FiveTime.dll` (time helpers), `Utility.dll` (the developers' shared utilities: pooling, noise, splines, silhouettes, taskbar flash, `AntiCheat.EncryptedStorage`),
  `Utility.SoftFloat.dll` (deterministic software float `sfloat`).
- Other libraries: A* Pathfinding Project, Addressables, Unity Localization (6 languages), TextMeshPro, the new Input System, Timeline, VFX Graph, **Tayx.Graphy** (FPS overlay),
  **CommandTerminal** (the in-game developer console, see [Developer console](dev-console.md)), **Backtrace.Unity** (crash reports), Newtonsoft.Json.
- `Assembly-CSharp.dll` is almost empty. **All game code is in `DiceKingdoms.dll`.**

!!! note "Crash reports"
    Backtrace sends crash reports (with minidumps) to the developers. A crash caused by a mod reaches them too.

## Namespaces

| Namespace | Types | Contents |
|---|---|---|
| `DiceKingdoms.Game` | 336 | Rules and simulation: `GameState`, phases, `Island` / `IslandGrid`, buildings, resources, technologies, combat, disasters, AI, players, lobby settings |
| `DiceKingdoms.Interface` | 274 | In-match UI and visuals: buttons, phase interfaces, dice, camera, building placer, tooltips, `StructuredText` |
| `DiceKingdoms.Meta` | 149 | Outside the match: main menu, matchmaking, store, rules screen, settings, developer cheats, version, Steam items |
| `DiceKingdoms.Network` | 34 | MLAPI serializers for the game types, snapshots, `AutoHost`, spawn handlers, `NetRef` |
| `DiceKingdoms.Environment` | 11 | Clouds, day/night cycle, seagulls, quality toggles |

## Scenes

The addressable scenes are `Scenes/MainMenu` and `Scenes/SteamMultiplayer`. A match runs in `SteamMultiplayer`.

## Settings files

`<game folder>/GameSettings/*.json`: `audio.json`, `game.json` (language), `input.json` (key bindings as Input System paths, for example rotate `/Keyboard/r`, mirror `/Keyboard/t`,
keep placing `/Keyboard/shift`, scoreboard `/Keyboard/tab`) and `video.json`. Audio, game and input are stored in Steam Cloud when Steam runs; video is local only.
Details: [Audio, settings and localization](audio-settings-localization.md).

## Useful constants

- `GameState.MAX_PLAYERS = 12`.
- Steam app id **1501690**. The update check calls `https://api.steamcmd.net/v1/info/{appid}` and compares the depot branches.
- The About window shows `{0} - build #{1} on branch {2}` (`LocalVersion` asset + `Version` struct).

## Conventions on these pages

- Class, method and field names are from the IL2CPP dump (Il2CppDumper). They are stable between updates; **addresses and offsets are not**.
- Addresses are **RVAs** in `GameAssembly.dll` of game version **1.0.4**. In Ghidra (or any tool that loads the DLL at its preferred base) the address is `0x180000000 + RVA`.
- Generic methods (`TurnPhase<,>`, `PhaseData<,>`) share one compiled body for reference types (`TurnPhase<object,object>`); its RVA is listed under the generic method in `dump.cs`.
- Standard IL2CPP layouts: object header `0x10`; array length `+0x18`, data `+0x20`; string length `+0x10`, UTF-16 characters `+0x14`; `List<T>` items `+0x10`, size `+0x18`.
  Struct field offsets in `dump.cs` are given without the header.
- "Confirmed in game" means the behaviour was also checked in a live match; everything else was read in the decompiled code.
