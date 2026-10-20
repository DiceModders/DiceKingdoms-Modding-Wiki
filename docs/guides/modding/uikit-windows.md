---
summary: UiKit windows - layout, buttons, text, pictures, sliders, dropdowns, scroll views, title bar
---

# UiKit: windows and widgets

## Window basics

```csharp
UiKit.Window(w => {
    w.Size(420, 520).Position(80, 80).Draggable().Closable().Title("My window");
    // w.Body is the container for your content
});
```

| Call | Effect |
|---|---|
| `Size(w, h)`, `Position(x, y)` | Size and position in canvas units (1920x1080 reference). |
| `Draggable()` | The user can drag the window by its background. Without it the window is locked. A dragged window stays inside the screen. |
| `Fullscreenable()` | Allows the full-screen state (see `FullscreenButton`). |
| `Closable()` | The game's own X button, top right. It gets its own header row, so content never runs under it. |
| `Title(text)` / `SetTitle(text)` | Title text in the header row. A too long title is cut with "...". |
| `TitleBar` | Container on the right of the header row for buttons (see below). |
| `FullscreenButton()` | A "Full screen / Window" toggle in the header row. |
| `Show()`, `Hide()`, `Close()` | Visibility. A closed window cannot be reopened; create a new one. |

The header row is about 48 px tall and exists as soon as the window has an X, a title or title bar buttons. A window without any of them has no header row and
uses its full height. The body cuts content that does not fit (nothing is drawn outside the window); put long or changing content into a `ScrollView`.

## Layout

Containers lay out their children with the game's uGUI layout groups.

```csharp
w.Body.Vertical(v => { v.Label("A"); v.Label("B"); }).WeightH(1).Spacing(4);
w.Body.Horizontal(row => { row.Button("OK", ok); row.Button("Cancel", cancel); }).FixedH(32);
w.Body.Grid(g => { for (int i = 0; i < 9; i++) g.Button($"{i}", null); }).Columns(3).CellSize(80, 80);
```

| Call | Meaning |
|---|---|
| `Fixed(px)`, `FixedW(px)`, `FixedH(px)` | Fixed size. `Fixed` sets both sides. |
| `Weight(w)`, `WeightW(w)`, `WeightH(w)` | Share the free space in proportion to the weight. |
| `Spacing(px)`, `Padding(...)` | Gap between children and inner margin. |
| `Columns(n)`, `CellSize(w, h)` | Grid settings. |

A common mistake is giving a widget both a fixed and a weighted size on the same side; the last call wins.

## Widgets

```csharp
var status = w.Body.Label("Ready");                         // one line, cut with "..." when too long
w.Body.Button("Run", () => Run());                          // 32 high, shares a row's width
w.Body.IconButton("hammer", () => Tools(), "Tooltip");      // square, game sprite name
w.Body.Toggle("Option", false, on => Set(on));              // the game's checkbox
w.Body.Slider(0, 100, 40, v => Set(v), "0");                // slider with a value text
w.Body.Dropdown(new[] { "A", "B" }, 0, i => Pick(i));       // the game's dropdown
w.Body.Space(8);                                            // empty gap
```

- **Handles:** every call returns a handle (`LabelHandle`, `ButtonHandle`, `ToggleHandle`, `SliderHandle`, `DropdownHandle`) with setters such as `SetText`,
  `SetInteractable`, `SetValue`, `SetIndex`, `SetOptions`. Setters used for state sync (`SetValue`, `SetIndex`) do not raise the change callback.
- **Buttons:** `ButtonHandle` also has `OnClick`, `SetSelected(bool)` (the game's selected/unselected art), `SetIcon` and `SetTooltip`.
- **Icons:** the icon names are the game's sprite names; list them with `Assets.SpriteNames("filter")`.

## Text that wraps

```csharp
var text = w.Body.Paragraph(longText);        // wraps at the width it gets, its height follows the text
text.SetText(otherText);                      // re-measures
w.Body.Label("Wrapped label").Wrap();         // a plain Label is one line unless you call Wrap() / AutoHeight()
w.Body.ScrollView(sv => sv.Paragraph(longText)).WeightH(1);   // also fine inside scroll views
```

`Paragraph(text, fontSize = 16)` returns a `LabelHandle`. On it: `Wrap(bool on = true)`, `AutoHeight(bool on = true)`, `Align(TextAlignmentOptions)`,
`Left()`, `Center()`, `Right()`. A wrapping label has no pixel width of its own, so it never pushes a row wider; its height is re-measured when the width,
the text or the font size changes.

## Title bar

```csharp
w.TitleBar.Button("Help", () => ShowHelp());                   // as wide as its text
w.TitleBar.IconButton("hammer", Tools, "Tools");               // square, same height
w.FullscreenButton();
```

Buttons on the title bar are as wide as their text; everywhere else `Button` shares the row. The strip keeps clear of the X in any call order.

## Pictures

```csharp
var map = w.Body.Image(texture).Filter(FilterMode.Point).FixedH(200);
map.OnPointer(e => {                                    // Down, Up, Click, Move, Drag, Enter, Exit
    if (e.Kind == PointerKind.Drag && e.Button == 1) Pan(e.Delta);   // e.Uv is 0..1 inside the picture
});
map.OnScroll(notches => Zoom(notches));                 // mouse wheel over the picture
// draw into the texture, then:
map.Refresh();
```

- A picture keeps the texture's proportions (square pixels stay square). `KeepAspect(false)` stretches it to its rectangle instead.
- The mouse wheel is read from the Input System, because the game's UI does not deliver wheel events reliably. A picture with an `OnScroll` handler takes
  the wheel before a scroll view around it.

## Scroll views

```csharp
w.Body.ScrollView(list => { for (int i = 0; i < 30; i++) list.Label("row " + i); }).WeightH(1);
```

The scrollbar uses the game's art and appears when the content is taller than the area. A scroll view stays at the top until the user scrolls,
also while its content grows. Use `WeightH(1)` to fill the rest of the window or `FixedH(px)` for a set height.
