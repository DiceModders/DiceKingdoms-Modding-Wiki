---
summary: ModCore mod switch - an Enabled entry in the mod's config that players can flip in game, with match-safe timing and what switching off does to rules, stats and trees
---

# Mod switch

`Switch` adds `[General] Enabled` to your mod's BepInEx config. An in-game config editor (such as the DiceModders ConfigMenu) shows it as a toggle, and your mod reacts at once
through `WhenOn` (it runs now and on every change; nothing is polled).

```csharp
public override void Load() {
    var sw = Core.Mods.Switch(Config, affectsSimulation: true);
    sw.WhenOn(on: () => StartMyFeature(), off: () => StopMyFeature());
}
```

| Parameter | Meaning |
|---|---|
| `affectsSimulation: true` | A change made during a match waits until the match ends, so a match never changes halfway |
| `restartNeeded: true` | The value is read at start only (for mods that cannot undo their patches) |
| `description` | Text shown for the config entry |

While a mod is switched off:

- its lobby rules are not announced to the lobby, so other players see the mod as missing;
- its unit stats, building / technology values and tech trees are not used, so everybody plays the game's values;
- `sw.IsOn` is false and `Core.Mods.IsOn("author.my-mod")` answers the same for other mods.

```csharp
if (Core.Mods.IsOn("author.other-mod")) UseItsFeature();
sw.Changed += on => Log.LogInfo("my mod is now " + (on ? "on" : "off"));
```
