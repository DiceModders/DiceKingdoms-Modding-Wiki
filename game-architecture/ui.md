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
