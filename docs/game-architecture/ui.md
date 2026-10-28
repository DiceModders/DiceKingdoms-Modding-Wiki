---
summary: How the game's user interface is built - canvases, buttons, window art, fonts, top-right bar
---

# Game UI

The interface is Unity uGUI with TextMeshPro. The facts below were collected from the game version 1.0.4.

## Canvases

| Canvas | Notes |
|---|---|
| `[Graphy]` | Overlay of the performance display, sorting order 30000. |
| `Canvas Over Dice`, `Canvas Background`, `Canvas` | Screen Space - Camera, sorting 0, reference resolution 1920x1080 (scale 1). The HUD is under `Canvas`. |

Mod windows are parented to the HUD canvas (found through components such as `BuildInterface`, `ChatInterface`, `LogInterface`).

## Buttons

Game buttons are components of the game, not plain Unity colours:

- `DiceKingdoms.Interface.BaseButton` holds the Unity `Button` (`unityButton`), the state, per-state cursor sprites, the `Interactable` property and the `onClick` event.
  `BaseButton.Update` maps the texture of the shown sprite to a state, so a button must always keep one of its known sprites.
- Subclasses: `ButtonIcon` (an icon, used by the top-right bar: `Top > Icon`) and `ButtonLabel` (a text button: `Top > Label`).
- Art: `Button-1-{Ready,Hovered,Pressed,Disabled}` (384x384, 9-slice border 32, pixels per unit 200, type Sliced, 32x32 units in the HUD). `Button-2-*` is the second
  style, used for the selected look. The Unity `Button` uses a sprite swap, plus a `Shadow` component.
- A sliced button needs a size of about twice its border. At exactly 32 units the centre slice is empty and rounding leaves hairlines.
- Label: `TextMeshProUGUI`, font `Main-Font-Normal`, size 20 with auto size, colour RGBA(0.969, 0.940, 0.893), material `Main-Font-Normal-Shadow`, centred.

## Windows and frames

- Window background: an `Image` with sprite `Background-3` (2048x2048, Simple, white with alpha about 0.95; `Background-4` is used less) and a child `Ornament-N` (Sliced, white).
- Ornament borders: 1 = 16, 2 = 142, 3 = 32 (most used), 4 = 32, 5 = 150.
- Fonts: TMP assets `Main-Font-Normal` (almost every text) and `Main-Font-Big`; materials `Main-Font-Normal-Shadow`, `Main-Font-Big-Shadow`, `Main-Font-Normal-Flat`.
- Icons: `hammer` (512x512), settings, back-to-menu, question-mark, and `building-icon-<type>-ancient-<colour>` (256x280).

## Top-right bar

`Canvas > Meta > Buttons` is a `HorizontalLayoutGroup` with a `ContentSizeFitter` (anchors top right, 140x32), so its children flow automatically.
The children are 32x32: Scoreboard, Rules (the "?" icon, opens the long rules screen), Settings, Back To Menu. Each has `Top > Icon` (24x24).
In the main menu the same style is under `Top Right/Top Right Buttons`. The Rules button has one persistent click listener, which must be removed when it is cloned.

## Scenes

The Rules screen, Settings screen, About, Store and Scoreboard are separate screens that exist in the hierarchy but may be inactive. Cloning a template from an
inactive screen works, but the screen must have been loaded.

## Settings rows and the lobby settings box

The Settings window (`SettingSetsInterface`) has four tabs (`Video`, `Audio`, `Input`, `Game`), each a `SettingSetInterface` with a `Vertical Scroll` (`ScrollRect`) whose `Viewport` is a
`RectMask2D` and whose `Content` is a `VerticalLayoutGroup` + `ContentSizeFitter`. The settings box of the match-start screen (the "Lobby" tab) is built the same way, 430x424 units.
`SettingSetInterface.GenerateElements` fills `Content` with an entry per setting; rows are made from prefabs the set references (`settingEntryInterfacePrefab`, `settingInterfacePrefabs`).

- **Entry:** `Setting Entry` is a `HorizontalLayoutGroup` + `SettingEntryInterface` with a `Label` (125 wide in the lobby box) and a `Content` holder (`contentHolder`).
  In the lobby box the holder is **305 wide and a `VerticalLayoutGroup`**; in the other tabs it is 500 wide and has no layout group. Its width is `0` until the layout has run, so a row that needs the width
  has to measure it again later. Controls put into a holder with a layout group are stretched or stacked by it.
- **Controls:** `SettingInterfaceBool` (a checkbox, `Toggle`), `SettingInterfaceFloat` (the slider of the Frame Rate Limit setting), `SettingInterfaceDropdown<T>` / `SettingInterfaceEnum*`,
  `SettingInterfaceInputPath`, `SettingInterfaceLobbyCode`. Each is bound to a `Setting<T>` of a `SettingSet`; a clone has no setting, so the binding component is destroyed **before** the clone wakes up
  (`SettingInterfaceFloat.Awake` adds a slider listener that writes `setting.Value` and would throw on every drag otherwise).
- **Slider row:** `Label` (the value text, 34 wide) + `Slider` > `Background` (4 high), `Fill Area` > `Fill`, `Handle Slide Area` > `Handle` (20 wide, the height of the slider). The handle is round only when the slider's
  rectangle is as tall as the handle is wide (20), and it sticks out about 5-10 units past both ends of the track, so the track must stay clear of any clipping edge.
- **Scroll view of the lobby box:** the `Viewport` mask is as big as the whole frame (430x424), so a row scrolled to the edge is drawn over the frame's yellow border (the game's own rows never reach it);
  the clip can be pulled in with `RectMask2D.padding` (2 units is the border's width). The `Scroll Bar` is 6x418 and spans the frame; its `Handle` has a `Shadow`, which at the bottom of the track hangs out of the frame.
- The lobby box shows only the game's own rows in the unmodded game (they fit, so it does not scroll); once a mod adds rows, the scrollbar, the clip and the row widths all matter.

## Canvas size and aspect ratio

The game's canvases use `CanvasScaler` in `ScaleWithScreenSize` mode with a 1920x1080 reference. The scene data (not the code) decides the match mode: the main `Canvas` matches **height**
(always 1080 units tall, width = 1080 x ratio), `Canvas Over Dice` matches **width** (always 1920 wide, height = 1920 / ratio).

| Ratio | Main canvas | Canvas Over Dice |
|---|---|---|
| 16:9 | 1920x1080 | 1920x1080 |
| 16:10 | 1728x1080 | 1920x1200 |
| 4:3 | 1440x1080 | 1920x1440 |
| 5:4 | 1350x1080 | 1920x1536 |
| 21:9 | 2520x1080 | 1920x823 |
| 32:9 | 3840x1080 | 1920x540 |

Everything that is sized in units (the lobby box, setting rows, windows) scales with the screen. Anything placed by its top-left corner must be kept inside the canvas when the size changes. `Canvas.scaleFactor` (screen pixels per unit) is what a mouse delta has to be divided by when dragging.

