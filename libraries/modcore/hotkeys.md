---
summary: ModCore hotkeys - keys a mod reacts to on the game's own input system, stored in the mod's config, rebindable in Settings > Input through ModBridge
---

# Hotkeys

`Core.Hotkeys.Add` gives your mod a key on the game's own input system (the Unity Input System, the same one the game's keys and its Settings > Input page use). Unity calls
your code when the key is pressed, so nothing checks the keyboard every frame.

```csharp
public override void Load() {
    Core.Hotkeys.Add(Config, "openWindow", "Open my window", "Ctrl+M", MyWindow.Toggle);
}
```

The key is stored in your config file as `[Hotkeys] openWindow = Ctrl+M`. Players change it there, in an in-game config editor, or in **Settings > Input > Mod keys** when
[ModBridge](../modbridge.md) is installed. The keys are bound once the game data has loaded; calling `Add` in `Load()` is fine.

## When the key fires

Your action runs on the main thread, and only:

- with **exactly** the modifiers of the binding held (`E` does not fire while Ctrl is held, so `E` and `Ctrl+E` are different keys);
- while no text field has the focus (chat, lobby name ...), so typing never triggers mods;
- while your mod is switched on ([Mod switch](mod-switch.md)).

```csharp
var undo = Core.Hotkeys.Add(Config, "undo", "Undo", "Ctrl+Z", Editor.Undo,
                            when: () => Editor.IsOpen);         // only while the editor window is open
```

`when` is checked at the moment of the press. Use it for keys that only make sense in one screen, instead of adding and removing the key.

## Key names

Keys are written as modifiers plus one key, joined with `+`: `F9`, `Ctrl+E`, `Shift+Alt+H`. The key names are Unity's `Key` names (`Digit1`, `PageUp`, `Numpad5` ...).
An empty default means "not bound until the player sets one".

```csharp
Core.Hotkeys.Add(Config, "dump", "Write the debug dump", "", Dump.Run);   // unbound by default
```

Avoid the game's own keys (camera, build mode, F1-F3, F10): both would react.

## Changing a key from code

```csharp
var key = Core.Hotkeys.Add(Config, "screenshot", "Screenshot", "F12", TakeScreenshot);
key.Set("Ctrl+F12");                     // also written to the config file; "" unbinds
key.Reset();                             // back to the default
key.Capture(keys => Log.LogInfo("now: " + keys));   // waits for the next key press (Escape cancels)
key.Pressed += () => Log.LogInfo("pressed");        // more listeners
```

`Capture` is what the Settings > Input screen uses: the next key press, with the held Ctrl / Shift / Alt, becomes the binding.

## Listing every mod's keys

`Core.Hotkeys.All` lists the keys of every mod in registration order, with `ModName`, `Title`, `Keys` and `Default`. `Added` and `Changed` fire when a key is added or
rebound, so a screen that shows them never polls.

```csharp
foreach (var key in Core.Hotkeys.All) Log.LogInfo($"{key.ModName}: {key.Title} = {key.Keys}");
Core.Hotkeys.Changed += key => Log.LogInfo($"{key.Title} is now {key.Keys}");
```

ModBridge's **Mod keys** block is built from exactly this: one heading per mod, one row per key with the game's own key box (click it, press the new key), and a *Default* button that puts every mod key back.
