---
summary: UiKit additions to the game's own screens - top bar buttons, Settings tabs and blocks with native rows, scoreboard columns, entries in the Rules civilization drop-down
---

# Game screens

Additions to the game's own UI. You describe what to add once; UiKit attaches it when the screen exists, takes it off when the screen goes and puts it back when it returns. The
handles stay the same the whole time. Additions are add-only: existing game elements are not changed or removed. Types are in `DiceModders.UiKit.Native`.

## Top bar

An icon button in the game's top bar. Lower `Order` comes first among mod buttons.

```csharp
var open = ui.Native.TopBar.Add(new TopBarButtonSpec {
    Id = "author.my-mod.open",
    IconName = "hammer",           // a theme icon; or Icon = mySprite
    Tooltip = "Open My Mod",
    Order = 10,
    OnClick = () => OpenWindow(ui),
});
open?.SetTooltip("Open My Mod (3 new)");
open?.SetVisible(Core.Match.Running);
```

## Settings

A whole tab of your own, or a block appended to a built-in page (`Video`, `Audio`, `Input`, `Game`, `Lobby`). Rows are the game's own row prefabs.

```csharp
ui.Native.Settings.AddTab("author.my-mod", "My Mod", tab => {
    tab.AddHeader("My Mod settings");
    tab.AddToggle("Enable feature", true, on => { });
    tab.AddSlider("Volume", 0f, 100f, 70f, v => { });
    tab.AddDropdown("Mode", new[] { "Simple", "Advanced" }, 0, i => { });
    tab.AddInfo("Saved in the mod's config.");
});

ui.Native.Settings.AddBlock(new SettingsBlockSpec {
    Id = "author.my-mod.game",
    Page = SettingsPage.Game,
    Title = "My Mod",
    Order = 10,
    Build = block => {
        block.AddSlider("Intensity", 0f, 10f, 5f, v => { }, format: "0.0", wholeNumbers: false);
        block.AddButtons("Actions", ("Reset", () => Reset()), ("Apply", () => Apply()));
    },
});
```

| Row | Result |
|---|---|
| `AddHeader(text)` | Group header |
| `AddToggle(label, value, changed)` | Toggle |
| `AddSlider(label, min, max, value, changed, format, wholeNumbers)` | Slider with its value |
| `AddDropdown(label, options, index, changed)` | Drop-down |
| `AddButton(label, buttonText, clicked)`, `AddButtons(label, (text, clicked)...)` | Buttons |
| `AddStepper(label, valueText, step)` | `-` value `+` |
| `AddKey(label, keyText, clicked)` | The game's own key box from Settings > Input (click it to capture a key; show the key with `SetText`) |
| `AddInfo(text)` | Wrapped paragraph |

Rows return handles; update them instead of rebuilding. Text providers are read when the row attaches and after each step, never polled.

```csharp
int count = 3;
SettingsRowHandle? info = block.AddInfo("Ready.");
SettingsRowHandle? stepper = block.AddStepper("Count", () => count.ToString(), delta => count += delta);
SettingsRowHandle? toggle = block.AddToggle("Demo", true, on => { info?.SetText("Demo: " + on); });

toggle?.SetValue(false);
info?.SetEnabled(false);
stepper?.SetVisible(Core.Lobby.IsHost);
```

A header (a block's `Title` or `AddHeader`) is centred on the row like the game's own section titles. A title long enough to reach over the label column is moved right so it starts where the controls start (or as far as the right edge allows). This is measured on the visible page after layout, because the lobby box and the Settings pages use different row templates and a hidden page has no sizes yet.

`SetText` changes the text of an info or stepper row. The text on a button row's buttons changes with `SetButtonText(text, index)`; `index` counts the buttons of an
`AddButtons` row from 0.

A key row uses the game's own key box, so it looks like the game's keys in Settings > Input. Clicking it calls you; show the key with `SetText`:

```csharp
SettingsRowHandle? row = null;
row = block.AddKey("Open my window", key.Keys, () => {
    row?.SetText("...");                                    // waiting for the key
    key.Capture(keys => row?.SetText(keys));                 // ModCore hotkey: the next key press
});
```

```csharp
SettingsRowHandle? keyRow = null;
keyRow = block.AddButtons("Open my window",
    ("Open", () => MyWindow.Open()),
    ("Close", () => MyWindow.Close()));
keyRow?.SetButtonText("Hide", 1);                                       // the second button
```

## Scoreboard

A column on every current and future scoreboard row, cloned from the game's own cell. Values are pushed by you and kept, so a value set before the scoreboard exists shows up
when it appears.

```csharp
var column = ui.Native.Scoreboard.AddColumn("author.my-mod.kills", "Kills");
column?.SetIcon(mySprite);
column?.SetValue(island, kills.ToString());
column?.SetValues(new Dictionary<DiceKingdoms.Game.Island, string> { [islandA] = "12", [islandB] = "7" });
column?.ClearValues();
```

## Rules civilizations

An entry in the civilization drop-down of the Rules screen. UiKit only draws it: the civilization itself (number, tech tree, looks, sync) must be registered elsewhere, for example
with [ModCore](../modcore/civilizations.md), otherwise the game fails when the entry is picked. [ModBridge](../modbridge.md) does this automatically for ModCore civilizations.

```csharp
var entry = ui.Native.Rules.AddCivilization(civ.Id, "My Civ");
entry?.SetName("My Civ (beta)");
entry?.Dispose();   // removes the entry
```

## Dialogs

The game's own yes/no dialog and information box (`ConfirmModal`). They exist on the match screen; elsewhere the calls return `false` and nothing opens.

```csharp
bool shown = ui.Native.Dialogs.Confirm(
    "Island grid 121 is experimental. Use it?",
    onConfirm: () => Accept(),
    onCancel: () => Revert(),
    icon: DialogsApi.WarningIcon);                 // a sharp warning sign; null = no icon, like the game's own questions
ui.Native.Dialogs.Info("Saved.", onClose: null);
```

`icon` is `null` (no icon), `DialogsApi.WarningIcon`, or a text icon token such as `{o.question-mark}`. A text icon is a small glyph scaled up to the icon box and looks blurry; the warning sign is drawn by UiKit at full resolution and is removed again when the dialog closes, so the game's own questions never show it. `dontShowAgainId` adds the game's "don't show again" box.

- The callbacks run just after the button press, not inside it: the game calls a button's callback **before** it closes the dialog, so a dialog opened from inside the callback would be closed at once (and the first dialog's cancel would run). With UiKit, opening another dialog from `onConfirm` is safe.
- With a ticked "don't show again" id the game does not open the dialog and confirms at once.
- While any game dialog is open (UiKit's or the game's own, such as Back to menu), the `AboveGame` layer is hidden: it is a screen overlay and would cover the dialog, and the game accepts no other input then. It returns when the dialog closes.