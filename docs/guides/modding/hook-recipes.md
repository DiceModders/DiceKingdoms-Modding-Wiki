---
summary: Tested Harmony recipes - read resources, react to phases, change harvest income, give or take resources, block an action, change combat numbers, change storage, mod the AI without code
---

# Hook recipes

All recipes were tested in game version 1.0.4 with BepInEx 6 (IL2CPP) in a custom match against AIs, as the host. The page [Networking](../../game-architecture/networking.md) explains which
side runs what.

```csharp
[BepInPlugin("author.mymod", "My Mod", "0.1.0")]
public class MyPlugin : BasePlugin
{
    public override void Load() => new Harmony("author.mymod").PatchAll(typeof(MyPlugin).Assembly);
}
```

- Private methods are patched by name: `[HarmonyPatch(typeof(GameState), "ChangeTurnPhase")]`. The interop assemblies make private members public, so private setters can be written.
- Structs like `ResourceAmount` can be created and edited directly: `var a = new ResourceAmount(); a.hammer = 3;`.

## Read resources

`island.statistics.Storage` returns the decrypted `ResourceStorage` (`stored`, `capacity`). Do not read memory directly: the storage is encrypted
([Harvest](../../game-architecture/harvest.md#encrypted-storage)). All players: `Object.FindObjectOfType<GameState>().Players`; a player's island: `player.Island`.

## React to phase changes

Postfix the private `GameState.ChangeTurnPhase(ITurnPhase)`. Phase name: `turnPhase.Cast<Il2CppSystem.Object>().GetIl2CppType().Name` (HarvestPhase, DisastersPhase, BuildPhase,
CombatPhase; WinPhase only when a win is due). For "after the phase was applied", postfix `<Phase>.ApplyChanges`. The `GameState` events (`onEnterPhase`, `onSyncPhase`, …) work too.

## Change harvest income

Prefix `HarvestData.AddResources`: read `__instance.TotalYields`, write a changed copy back. Gains are still capped by storage. The harvest screen keeps showing the original dice values;
to change what the dice show, patch the roll (`HarvestData.Roll` / `RollResultsRPC`).

## Give or take resources

`island.statistics.AddResource(ResourceAmount)` / `RemoveResource(...)`, capped by capacity.

## Block a player action

Prefix with `ref bool __result`, set `__result = false` and `return false`. Example: a prefix on `BuildData.BurnBuilding` that refuses Castles; nothing happens and no hammer is spent.

- Find a building by id by walking `island.grid.Buildings` and matching `id.value == (pos.y << 16) | pos.x`. **Do not call `IslandGrid.GetBuilding`**: it returns a `Nullable<Building>`,
  which crashes the game natively through the interop.
- Type test: `b.type.TryCast<BuildingTypeCastle>() != null`.
- Postfixes of other patches still run when your prefix skips the original.
- To block any build action in one place, prefix the private `BuildData.PlanAction`. Market trades never call `BuyResource`; they happen inside `DoAction`.

## Change a combat number

Postfix the static `ProjectileTypeExtensions.BuildingDamage(Projectile.Type)` and change `__result` for `FIREBALL` (default 8). The same pattern works for `Duration` and
`CombatGrid.MeleeBuildingDamage`. Tested: with 100 a single fireball destroyed a Dock (50 HP). The Rules screen's unit page reads the same function, so it shows the changed value.

Combat is simulated on every client, so such a mod must be installed on every player's game.

## Change storage capacity

Postfix the private `Island.RecomputeStorageCapacities`: take `statistics.Storage.capacity`, change it, call `statistics.SetCapacity(cap)`. `SetCapacity` clamps the stored amounts, so never
set it lower than you want to keep. It runs after every phase, so the bonus is re-applied each time.

## Mod the AI without code

Put `custom-build-orders.txt` next to `Dice-Kingdoms.exe`; the game reads it instead of its built-in build orders. Format and details: [AI](../../game-architecture/ai.md#build-orders).

## More hook points

- Camera to an island: `cameraMovement.focus = new Il2CppSystem.Nullable<CameraMovement.Focus>(new CameraMovement.Focus(island.grid, 1f, true))`.
- Night: postfix `TurnPhase.Enter` and set `DayNightCycle.TimeOfDay` ([Rendering](../../game-architecture/rendering.md#day-and-night)).
- Messages everybody sees: `Log.AddMessage(new StructuredText(...))` on the host ([Interface behaviour](../../game-architecture/interface.md#tooltips-dialogs-game-log-chat)).
- Unlock the console: force `Cheats.get_Available` to true ([Developer console](../../game-architecture/dev-console.md)).

## Which mods must every player have

- Changes to rules that run on the server (harvest, resources, build checks, AI, storage) take effect for everybody when the **host** has the mod.
- Changes to combat numbers must be on **every** client, because each client simulates the fights.
- Read-only and visual mods (logging, camera, colours, models, effects) are safe on one machine.
