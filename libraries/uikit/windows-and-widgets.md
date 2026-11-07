---
summary: UiKit windows and widgets - windows, rows, columns, grids and scroll views, buttons, toggles, sliders, dropdowns, text, images with pointer and wheel input, updating values
---

# Windows and widgets

## Windows

`Custom.Window` creates a window; the builder is also the handle. `Body` is where the content goes.

```csharp
var window = ui.Custom.Window(w => {
    w.Title("Kingdom overview")
     .Size(520f, 360f)
     .Center()
     .Layer(Layer.AboveGame)
     .Draggable()
     .Closable()
     .FullscreenButton()
     .Ornament(OrnamentStyle.Border);

    w.OnClose(() => Log.LogInfo("closed"));
    w.Body.Paragraph("Window content goes here.");
});
window?.BringToFront();
```

| Member | Purpose |
|---|---|
| `Title`, `Size`, `Position`, `Center` | Layout of the window |
| `Layer`, `SortingOrder` | `BehindGame`, `Game` or `AboveGame`, and the order inside it |
| `Draggable`, `Closable`, `Fullscreenable`, `FullscreenButton` | Behaviour |
| `Ornament` | `None`, `Border`, `BorderWithTwoCorners`, `BorderWithFourCorners`, `Classic` |
| `Show`, `Hide`, `Toggle`, `Close`, `Fullscreen`, `Restore`, `Dispose` | State |
| `TitleBar` | A container in the title bar for extra buttons |

## Layout

Rows, columns, grids and scroll views are containers; size helpers say how much space each child gets. Sizes are in canvas units and scale with the screen.

```csharp
w.Body.Row(row => {
    row.Column(left => left.Text("Fixed"))?.FixedW(140f);
    row.Column(right => right.Button("Stretch", () => { })?.FillWidth())?.WeightW(1f);
}).Height(48f);

w.Body.Grid(3, 100f, 80f, grid => {                 // 3 columns, cell 100 x 80
    for (int i = 0; i < 6; i++) grid.Button("Cell " + i, () => { })?.Fill();
}).Height(180f);

var list = w.Body.Scroll(s => {
    for (int i = 0; i < 20; i++) s.Text("Row " + (i + 1));
}).Height(240f).FillWidth();
list?.ToBottom();
```

| Size helper | Effect |
|---|---|
| `Fixed(w, h)`, `FixedW`, `FixedH`, `Width`, `Height` | Fixed size |
| `Weight`, `WeightW`, `WeightH` | Share the free space |
| `Fill`, `FillWidth`, `FillHeight` | Weight 1 |

Containers also have `Spacer`, `Separator`, `Spacing`, `Padding`, `Clear` and `Remove`.

## Widgets

Widgets use the game's own skins when available. Each factory returns a handle you keep to update the widget later.

```csharp
var toggle   = w.Body.Toggle("Enable feature", true, on => Log.LogInfo("on: " + on));
var slider   = w.Body.Slider(0f, 100f, 50f, v => Log.LogInfo("value: " + v));
var dropdown = w.Body.Dropdown(new[] { "Low", "Medium", "High" }, 1, i => Log.LogInfo("index: " + i));
var button   = w.Body.Button("Apply", Apply);
button?.Icon("Misc-Check");
button?.Tooltip("Apply changes");
var status   = w.Body.Text("Ready.");
```

Update the handles instead of rebuilding the window:

```csharp
if (status != null) status.Value = "Running";
button?.Text("Stop");
button?.Interactable(false);
toggle?.Set(false);
slider?.Set(75f);
dropdown?.Options(new[] { "A", "B", "C" });
dropdown?.Set(2);
```

| Factory | Useful members |
|---|---|
| `Text`, `Paragraph` | `Value`, `Wrap`, `Align`, `Color`, `Size` (Paragraph wraps and grows) |
| `Button`, `IconButton` | `Text`, `Icon`, `Tooltip`, `Selected`, `Interactable`, `OnClick` |
| `Toggle` | `Value`, `Set`, `Text`, `Interactable` |
| `Slider` | `Value`, `Set`, `Range`, `Format`, `WholeNumbers`, `Interactable` |
| `Dropdown` | `Index`, `Selected`, `Count`, `Set`, `Options`, `Interactable` |

## Images and pointer input

An image can react to the mouse: position inside the picture (0..1), drag with any button, wheel. While the pointer is over an image with a wheel handler, the image gets the
wheel; otherwise the surrounding scroll view does.

```csharp
var map = w.Body.Image(myTexture);
map?.KeepAspect(true);
map?.OnPointer(e => {
    if (e.Kind == UiPointerKind.Click && e.Button == 0) SelectTile(e.Uv);
    if (e.Kind == UiPointerKind.Drag) Pan(e.Delta);
});
map?.OnScroll(notches => Zoom(notches));   // positive = up
```

Pointer kinds: `Down`, `Up`, `Click`, `Move`, `Drag`, `Enter`, `Exit`. Events carry `Button` (0 left, 1 right, 2 middle), `Uv`, `Position` and `Delta`.
