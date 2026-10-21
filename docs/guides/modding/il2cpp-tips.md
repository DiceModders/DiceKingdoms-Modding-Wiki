---
summary: Common pitfalls when writing BepInEx 6 mods for an IL2CPP Unity game
---

# IL2CPP tips

Game classes are exposed through generated interop assemblies. They look like normal C# classes but are wrappers around native objects, which causes a few
surprises.

## Casting

`as` and `is` work on the wrapper type, not on the native type. A `Transform.parent` is returned as a `Transform` wrapper, so
`transform.parent as RectTransform` gives `null`. Use the interop cast:

```csharp
var rect = transform.parent.TryCast<RectTransform>();
if (rect != null) { /* ... */ }
```

## Delegates and callbacks

A C# delegate cannot be passed to the game directly. Convert it, and keep both objects alive for as long as the game may call it; otherwise the garbage collector
frees it and the game crashes later.

```csharp
Action handler = OnClick;
var native = DelegateSupport.ConvertDelegate<UnityEngine.Events.UnityAction>(handler);
button.onClick.AddListener(native);
keepAlive.Add(handler); keepAlive.Add(native);    // e.g. a list owned by the same Unity object
```

## Your own MonoBehaviours

A MonoBehaviour that the game must instantiate (for example to receive `Update`) has to be registered first, and needs a constructor taking an `IntPtr`:

```csharp
ClassInjector.RegisterTypeInIl2Cpp<MyBehaviour>();   // once, in Load()
public MyBehaviour(IntPtr ptr) : base(ptr) { }
```

The BepInEx log prints `Registered mono type ... in il2cpp domain` for each registered type.

## Cloning game UI

- Clone with `Object.Instantiate` under an **inactive** parent, strip what you do not want, configure it, then move it to the real parent. The game's `Awake`
  and `OnEnable` run when the clone becomes active, so they see the final setup.
- Remove the donor's click listeners and tooltip text, otherwise a cloned "Rules" button would still open the Rules screen.
- Do not clear a button's sprite: `BaseButton.Update` maps the shown sprite to a state and throws every frame for an unknown one.
- Font asset and material must be copied together from a live game text; a mismatched pair draws garbage glyphs.
- A TMP text cut with "..." shows nothing at all if its box is shorter than one line. Size the box from the font's line height.

## Debugging

- Read `BepInEx/LogOutput.log` first; `ErrorLog.log` usually holds little (launcher and crash-reporter lines).
- Dump a game object hierarchy once (name, rect, anchors, components) when you need the structure of a screen, and log it with a "log once" helper
  so it does not spam every frame.
- Il2CppDumper's `dump.cs` and Ghidra with the game's `GameAssembly.dll` loaded give names, fields and method bodies. See
  [Reverse-engineering notes](../../game-architecture/reverse-engineering.md).
- If the build says `succeeded` very quickly and nothing changed in the game, the DLL was not recompiled. Run `dotnet build -c Release --no-incremental`
  and copy the new DLL (check its timestamp).
