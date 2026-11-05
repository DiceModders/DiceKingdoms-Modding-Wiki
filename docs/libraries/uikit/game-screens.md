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
