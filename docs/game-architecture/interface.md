---
summary: How the in-match interface works - phase screens, input maps, camera, build tools, cursors, StructuredText, tooltips, dialogs, game log, chat, timer, Ready, banners, scoreboard, settings widgets, HUD layout, tips for UI mods
---

# Interface behaviour

*Code: [where to patch and the declarations](#code-reference).*

How the in-match interface behaves. The visual building blocks (canvases, buttons, window art, settings rows, aspect ratios) are on [Game UI](ui.md).

## Who owns what

- `Interface` (one per match, `GameState.@interface`) references the `CameraMovement`, `Training`, `StartGame` and the arrays `requireStartGame` / `hideInterface`. It holds `GameState`,
  `Island` (the local player's island; null for spectators) and `Hidden`, and has events used by the tutorial: `OnBuild(BuildingType)`, `OnLearn(Technology)`, `OnPillage()`, `OnStarvation(...)`.
- One **phase interface** per phase: `StartInterface`, `HarvestInterface`, `BuildInterface`, `CombatInterface`, `WinInterface` (disasters only have visuals). Base class
  `PhaseInterface<TData, TIslandState>` with `Data`, `TurnPhase`, `GameState`, `Island`, `Owner`, `BannerDuration`, virtual `Enter()`, abstract `Sync()`, optional `PhaseBanner`.
- `TurnPhase.Enter` **enables** the phase interface component on clients and calls its `Enter()`; `TurnPhase.Exit` disables it. So a phase screen is only enabled during its phase:
  **its `OnEnable` / `OnDisable` / `Enter` are the hook points** for UI that belongs to one phase. Global alternatives: `GameState.onEnterPhase`, `onSyncPhase`, `onStartGame`, `onEndGame`.

`Interface.Update` (`0x5A6630`): **F3** toggles `Hidden` (every object in `hideInterface` is switched off: screenshot mode). The mouse cursor is hidden and locked while the camera is grabbed
or the HUD is hidden, otherwise visible and **confined to the window**.

## Input

| Map | Action | Binding |
|---|---|---|
| Camera | Move | WASD / arrow keys |
| Camera | Zoom | mouse wheel |
| Camera | Grab | Space or middle mouse |
| Camera | MouseMove | mouse delta while grabbed |
| Building | Rotate / Mirror | R / T |
| Building | Place, Select | left mouse |
| Building | Cancel | Esc or right mouse |
| Building | KeepPlacing | Shift (hold) |
| Interface | Scoreboard / Chat | Tab / Enter |

Bindings are physical key positions (layout independent). `InputActionMaps` (a ScriptableObject) has `StartDisable(map) → DisableToken`; `token.Stop()` re-enables. A map is disabled
while at least one token is active. The game uses it only for chat.

!!! tip "Text fields in a mod window"
    Get the asset with `Resources.FindObjectsOfTypeAll<InputActionMaps>()[0]` and take `StartDisable(Camera)` and `StartDisable(Building)` tokens while your field has focus, then `Stop()` both.
    Otherwise WASD moves the camera and R / T rotate while typing.

## Camera

`CameraMovement`: min height 4, max height 100, move speed 5, zoom speed 0.2, zoom smoothing 10, mouse move speed 0.002; pitch by height (0 → 0°, 4 → −30°, 25 → −55°,
100 → −90° straight down). The yaw is fixed at 45° (see [Rendering](rendering.md#building-orientation-and-the-camera)).

- `focus` (`Nullable<Focus>`): `new Focus(IslandGrid grid, float duration = 0.5, bool stopAfterEnd = true)` or `Focus(float2 position, float radius, float endTime, bool stopAfterEnd)`.
  **To fly the camera to an island:** `cameraMovement.focus = new Il2CppSystem.Nullable<CameraMovement.Focus>(new CameraMovement.Focus(island.grid, 1f, true))` (confirmed in game).
- While `focus` has a value, all camera input is ignored (`UpdatePositionFocus` zeroes it) until it arrives (`stopAfterEnd = true`) or is cleared. The disasters phase uses this to lock the
  camera ([Disasters](disasters.md#playback-and-the-camera)). The console's free camera skips the focus.
- `FocusedIsland` = the island nearest to the camera within 15 units. Clicking a player in the player list flies the camera to that island (`PlayerSummary.OnPointerClick`).

## Build phase tools

`BuildInterface` owns `BuildingPlacer` (single buildings), `BuildingBulkPlacer` (walls: drag a path), `CleanTileTool`, `BuildingSelector` (click a building → repair / burn),
`MarketInterface`, `UniversityInterface`, `TechnologyInterface`, `ConfirmModal`, `DisasterGauge` and `AvailableActions` (hint bar of usable keys).

- Flow: building button → `BuildingPlacer.SetCurrentBuildingType(type)` → left click → `BuildInterface.OnPlace` → `BuildData.PlaceBuilding`. With Shift held and still affordable the placer
  stays active.
- `BuildInterface.Update` refreshes the building buttons only when stored resources or the number of learnt technologies changed. Repair / burn are enabled when you can pay the tool cost and the
  building is on your island. Nothing filters the Castle ([Build phase](build-phase.md#burning-the-castle)).
- `TilePointer` = the tile under the mouse (island, position, orientation, mirroring, footprint preview).

## Mouse cursor

- `Utility.UI.CursorVariation` (ScriptableObject): `texture` + `hotSpot`; `Use()` = `Cursor.SetCursor` (hardware cursor).
- `Utility.UI.CursorHelper` keeps a **priority stack** of overrides: `StartOverride(variation, priority)` returns an `Override` (with a token); `EndOverride(override)` removes it. The highest priority
  is shown; when the list is empty the default cursor (`cursor-default.png`, hotspot 1, 1) is used.
- Users: `BaseButton` hover (priority **10**), `HooverCursor` component (0), clean-tile tool, building placers, combat planning (attack / ship, valid / invalid) (all 0). So a button hover always wins.
- Cursor assets: Hand (8, 1), Build (1, 16), Clean-Tile (1, 27), Attack / Attack-Invalid (1, 1), Ship / Ship-Invalid (5, 5).
- `TileCursor` / `TileCursorPart` are not the mouse cursor: they are the animated 3D tile highlights on the island.

For your own UI: add `HooverCursor`, or call `StartOverride` with priority above 10 and always `EndOverride` on exit / disable.

## Text: StructuredText

`StructuredText` is a list of `ITextElement` built from a C# interpolated string (`$"..."`); `Render()` gives the TextMeshPro string, `+` joins two. Each `{hole}` is converted by type:

| Value type | Becomes |
|---|---|
| `ITextElement` / `StructuredText` | itself / its elements |
| `int` | number |
| `Resource` | resource icon |
| `ResourceAmount` | for each resource > 0: the icon repeated N times (N ≤ 3 and total < 6), else "N" + icon |
| `Color` | colour start (close with `ITextElement.End.INSTANCE`) |
| `Player` / `Team` | coloured player name / team |
| `BuildingTypeHousing` | dice icon |
| `(BuildingType, Player)`, `(Unit.Type, Player)`, `(Technology, Player)` | building / unit / technology in that player's skin |
| `Disaster` | disaster |
| `string` | text, with **tokens** parsed |

Tokens inside strings (also inside localized texts): `{a.N}` (the N-th element argument), `{b.<building type>}`, `{d.<disaster>}`, `{r.<resource>}` (food, wood, stone, gold, hammer, culture,
military, disaster), `{o.<icon>}` (hungry, scoreboard, research, question-mark, ship, population, clean-tile). Unknown tokens are dropped. In a C# interpolated string the token braces
must be doubled: `$"{player} got 5 {{r.gold}}"`.

Other elements: `Localized(key, args)`, `OtherIcon` (AI, CLEAN_TILE, DISCONNECTED, HOST, HUNGRY, POPULATION, QUESTION_MARK, RESEARCH, SHIP, ATTACK_POWER, SHIELD, SCOREBOARD).

## Tooltips, dialogs, game log, chat

- **Tooltips:** add `HasToolTip` to a UI object and set `.text` (a `StructuredText`); shown after 0.4 s.
- **Dialogs:** `ConfirmModal.OpenConfirm(icon, description, onCancel, onConfirm, dontShowAgainId)` and `OpenInformation(icon, description, onUnderstood, dontShowAgainId)`: the game's own dialog, reusable.
- **Game log:** `Log` is networked. **`Log.AddMessage(StructuredText)` works on the server only** (a client just logs a warning); the entry syncs to every client, so a host-side mod can post
  messages everybody sees. Rows are visible 30 s and fade in 1 s.
- **Chat** (`ChatInterface`): Enter opens the field and disables the other input maps; sending calls `Player.Local.Chat(text)` (the host adds it to the log). While chat is open every chat and
  log row stays visible, and fades again 1 s after closing.

## Timer, Ready, banners

- **Timer:** turn number and **ceil(time remaining)** (`TimeRemaining` while ACTIVE, `TimeRemainingSync` while SYNC; empty without a limit). The clock hand: Start / Harvest 0°, Disasters 90°,
  Build 180°, Combat 270°. A tick sound plays for each of the **last 4 seconds**; a "waiting for you" sound plays once after 2 s when the others are ready and you are not.
- **Ready** button: click = `TurnPhase.PhaseData(myIsland).Skip()`. Clickable only while the phase is ACTIVE, after the 2 s banner, when you are not ready yet and the phase allows skipping.
  If the phase already considers you ready, it skips by itself.
- **Phase banner:** slides in from 1024 px above, leaves after **2 s** (`disappearAfter`, which is also `BannerDuration`), switched off at 2.5 s. Changing `disappearAfter` also changes when
  Ready becomes clickable.
- **Imminent victory warning:** on every phase start (not the Win phase) `WinPhase.CheckWin()` is evaluated; if the predicted winner changed, or during every Build phase while a Wonder win is
  pending, the warning shows for 6 s and fades in the last 1 s.
- **Counters** (57 animated numbers) move toward their target at `|target − value| × 2 + 10` per second.

## Screens

- **Harvest** (`HarvestInterface`): one `Dice` per housing building, a "keep" and a "re-roll" grid; clicking a die moves it. The face is decided by the game before the throw, so a mod that
  changes yields afterwards does not change the face shown.
- **Combat planning** (`CombatInterface`): a slider per unit family, the ship "ghost" (`shipPreview`) and `ClosestValidDebarkTransform` for the landing preview. No ghost on your own island.
- **Scoreboard** (Tab): win conditions (military: defeated / total; wonder: own vs best other, missing resources, win delay), ping, resources, population, technologies.
  **Send-resource arrows**: 1 per click, Shift 5, Ctrl 10, capped by what you have → `BuildData.SendResources`. Shown only in the Build phase, when you are not defeated, for other players
  (teammates only in team games), and **only with an available Market** (active, not placed this phase).
- **Player list** (`PlayerSummary`): attack / defence power, population, connection, disaster log; click = fly to that island.
- `TechTreeInterface` is data-driven (`TechTree`, `IsTechnologyLearnt/Learnable/DemoLocked`, `StoredResources`, `OnClickTechnology`, then `Populate()`) and can be reused for a mod's own tree.
- `WinInterface`: victory visual, Continue / Exit / Discord / Steam buttons, gold gained. `StartInterface`: placer + selector for the start buildings.

## Settings widgets

`SettingSetInterface.GenerateElements` (`0x54D2F0`) builds the settings window **by reflection** over a `SettingSet`: an optional header (`settings.<set>.header.<key>`), then for every
setting a row labelled `settings.<set>.setting.<name>` with a widget chosen **by the setting's value type**: `bool` → toggle, `float` → slider (range from the setting), enums
(`FullScreenMode`, `Language`, `Quality`, `TimerSpeed`) and `ScreenResolution` → drop-down, input path → key rebinding (keyboard keys only), lobby code. An unknown type throws.
So a new field of a supported type on a `SettingSet` appears in the game's own window (it needs the localization keys).

## HUD layout

Positions in 1920×1080 reference units; *off* = inactive in the scene, switched on by code.

| Object | Anchor | Position | Size | Component |
|---|---|---|---|---|
| Left (player list + log) | top-left | 0, 0 | 400×904 | `PlayerSummaries`, `LogInterface` |
| Resources *(off)* | bottom-left | 5, 5 | 360×166 | |
| Building Info | bottom-left | 637, 5 | 844×166 | `BuildingInformation` |
| Chat | bottom-center | −720, 300 | 450×400 | `ChatInterface` |
| Disasters *(off)* | middle-right | 0, 90 | 200×700 | `DisasterGauge` |
| Available Actions | bottom-right | −180, 176 | 160×176 | `AvailableActions` |
| Phases (Build, Combat, Harvest) *(off)* | stretch | full screen | | phase screens |
| StartGame *(off)* | stretch | | | `StartGame` |
| Timer *(off)* | stretch | | | `Timer`, `Ready` |
| Imminent victory warning (Canvas Over Dice) | middle-center | 0, 256 | 1024×128 | `ImminentVictoryWarning` |

The UI shares one perspective camera (`Viewport`, FOV 45) with the 3D world: `Canvas Over Dice` (plane distance 0.11) is in front of the 3D dice, `Canvas` (1) behind them, and
`Canvas Background` (1.1) further back. That is why the harvest screen exists on three canvases.

## Tips for UI mods

1. Find objects with `Object.FindObjectOfType<Interface>()`, `<BuildInterface>()` etc.; the interop exposes private serialized fields (`cameraMovement`, `buildingPlacer`, …).
2. UI for one phase: patch that phase interface's `OnEnable` / `OnDisable` / `Enter`, or subscribe to `GameState.onEnterPhase`.
3. Text fields: disable the Camera and Building input maps while typing.
4. Text with icons, colours and localization: `StructuredText`; tooltips: `HasToolTip`; dialogs: `ConfirmModal`.
5. Messages to all players: `Log.AddMessage` on the host.
6. If your panel should hide with F3, add its root object to `Interface.hideInterface`.

## Code reference

Declarations from the IL2CPP dump of game version 1.0.4, trimmed to the members a patch usually needs. `// 0x..` is the field offset, `// RVA 0x..` the method address in `GameAssembly.dll` (in Ghidra: `0x180000000 + RVA`). Names are stable between updates, offsets and RVAs are not. Private members are public in the BepInEx interop assemblies.

### Where to patch

| To | Patch / use |
|---|---|
| Fly the camera | `CameraMovement.focus = new Nullable<Focus>(new Focus(island.grid, 1f, true))` |
| Stop typing from moving the camera | `InputActionMaps.StartDisable(Camera)` / `(Building)`, then `token.Stop()` |
| Post a message everybody sees (host) | `Log.AddMessage(StructuredText)` |
| Show a dialog | `ConfirmModal.OpenConfirm` / `OpenInformation` |
| Tooltip on your UI | add `HasToolTip`, set `.text` |
| Hide your UI with F3 | add your root to `Interface.hideInterface` |

### Declarations

```csharp
// DiceKingdoms.Interface
public class Interface : MonoBehaviour
{
    private CameraMovement cameraMovement;                   // 0x28
    private GameObject[] hideInterface;                      // 0x30
    private Action<BuildingType> OnBuild;                    // 0x58
    private Action<Technology> OnLearn;                      // 0x60
    private Action OnPillage;                                // 0x68
    private Action<ValueTuple<int, int, int>> OnStarvation;  // 0x70

    public Island Island { get; set; }
    public bool Hidden { get; set; }
}
```

```csharp
// DiceKingdoms.Interface
public class CameraMovement : MonoBehaviour
{
    public Nullable<CameraMovement.Focus> focus;                   // 0xA0

    public bool Grabbed { get; }
    public Island FocusedIsland { get; set; }

    protected void Update();                                       // RVA 0x59D7E0
    private void UpdatePositionFocus(CameraMovement.Focus focus);  // RVA 0x59C870
}
```

```csharp
public struct CameraMovement.Focus
{
    public float2 Position { get; }
    public float Radius { get; }
    public float EndTime { get; }
    public bool StopAfterEnd { get; }

    public bool Equals(CameraMovement.Focus other);                                           // RVA 0x5A2740
    public override bool Equals(object obj);                                                  // RVA 0x5A27E0
    public override int GetHashCode();                                                        // RVA 0x5A2900
    public static bool op_Equality(CameraMovement.Focus left, CameraMovement.Focus right);    // RVA 0x5A2AD0
    public static bool op_Inequality(CameraMovement.Focus left, CameraMovement.Focus right);  // RVA 0x5A2B80
}
```

```csharp
// DiceKingdoms.Interface
public class InputActionMaps : ScriptableObject
{
    public InputActionMap Camera { get; set; }
    public InputActionMap Building { get; set; }
    public InputActionMap Interface { get; set; }

    protected void OnEnable();                                                  // RVA 0x5A5B90
    public InputActionMaps.DisableToken StartDisable(InputActionMap inputMap);  // RVA 0x5A5C60
}
```

```csharp
// DiceKingdoms.Game
public class Log : NetworkedBehaviour
{
    public void AddMessage(FormattableString message);  // RVA 0x6402C0
    public void AddMessage(StructuredText message);     // RVA 0x6401D0
}
```

```csharp
// DiceKingdoms.Interface
public class StructuredText
{
    public string Render();  // RVA 0x595700
}
```

```csharp
// DiceKingdoms.Interface
public class HasToolTip : MonoBehaviour, IPointerEnterHandler, IEventSystemHandler, IPointerExitHandler
{
    private string initialText;                    // 0x18
    private LocalizedString localizedInitialText;  // 0x20
    public StructuredText text;                    // 0x28
    private ToolTips toolTips;                     // 0x30
}
```

```csharp
// DiceKingdoms.Interface
public class ConfirmModal : MonoBehaviour
{
    public void OpenInformation(StructuredText icon, StructuredText description, Action onUnderstood, string dontShowAgainId);  // RVA 0x58A880
    public void OpenConfirm(StructuredText icon, StructuredText description, Action onCancel, Action onConfirm, string dontShowAgainId);  // RVA 0x58A840
}
```
