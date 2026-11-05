---
summary: ModCore tools - patching game methods by name with clear logging, finding game types, name-based reflection on interop objects, raw field reads at offsets
---

# Tools

Small helpers in `DiceModders.ModCore.Tools` that work without `ModCore.Init()`.

## Patcher

Patches a game method by type and method name (private ones too). Every patch is logged as `[patch] OK` or `[patch] FAIL` with a hint when a game update renamed something, and
it never throws, so one broken patch does not stop the rest of your mod.

```csharp
var patcher = new Patcher("author.my-mod", Log);    // your plugin's ManualLogSource
patcher.Patch("GameState", "ChangeTurnPhase", typeof(MyPatches), postfix: nameof(MyPatches.AfterPhase));
patcher.Summary();                                   // "[patch] 3/3 patches active."
int failed = patcher.Failed;
Harmony harmony = patcher.Harmony;                   // for anything else

static class MyPatches {
    public static void AfterPhase() { /* ... */ }
}
```

## GameTypes

Finds a game type by its simple or full name in the interop assemblies, and suggests similar names when it is not found.

```csharp
Type grid = GameTypes.Find("IslandGrid");
if (GameTypes.Find("IslandGird") == null) Log.LogInfo(GameTypes.Suggest("IslandGird"));   // similar type names
Log.LogInfo(GameTypes.SuggestMethods(grid, "PlaceBuilding"));                              // similar method names
```

## Reflect

Name-based access to interop objects, private members included. A plain `int` is accepted for enum parameters.

```csharp
object result = Reflect.Call(gameState, "ChangeTurnPhase", 2);
if (Reflect.TryGetMember(island, "owner", out object owner)) Log.LogInfo(owner.ToString());
Reflect.TrySetMember(config, "hammerToKeep", 8);
object wrapped = Reflect.Wrap(grid, pointer);       // an interop object from a raw pointer
```

## Mem

Raw reads and writes at a field offset, for members the interop does not expose. Offsets change with every game build, so keep them in one place and check them after updates.

```csharp
int tier = Mem.ReadInt(buildingType.Pointer, 0x78);
Mem.WriteInt(buildingType.Pointer, 0x80, 5);        // health
string text = Mem.Str(Mem.ReadPtr(obj.Pointer, 0x10)); // a string field
int count = Mem.ArrayLength(Mem.ReadPtr(obj.Pointer, 0x18));
```

Never use it on encrypted values (resource storage).
