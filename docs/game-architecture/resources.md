# Resources

## Resource

Resources in the game are represented by the `DiceKingdoms.Game.Resource` enum:

```c#
namespace DiceKingdoms.Game;

public enum Resource
{
	FOOD,
	WOOD,
	STONE,
	GOLD,
	HAMMER,
	CULTURE,
	MILITARY,
	DISASTER
}
```
The enum values act as identifiers: the code uses them to address a resource in indexers, arrays, and lists, pass it to methods, and serialize it in network messages.

## ResourceAmount

Resource amounts are represented by `the DiceKingdoms.Game.ResourceAmount` struct:

```c#
namespace DiceKingdoms.Game;

[Serializable]
[StructLayout(LayoutKind.Explicit)]
public struct ResourceAmount
{
    [FieldOffset(0)]  public int food;
    [FieldOffset(4)]  public int wood;
    [FieldOffset(8)]  public int stone;
    [FieldOffset(12)] public int gold;
    [FieldOffset(16)] public int hammer;
    [FieldOffset(20)] public int culture;
    [FieldOffset(24)] public int military;
    [FieldOffset(28)] public int disaster;

    public static ResourceAmount ZERO { get; set; }

    public int this[Resource resource] { get; set; }

    public int Total { get; }

    public override bool Equals(ResourceAmount other);
    public override bool Equals(object obj);

    public override int GetHashCode();

    public override string ToString();

    public static ResourceAmount operator +(ResourceAmount a);
    public static ResourceAmount operator -(ResourceAmount a);

    public static ResourceAmount operator +(ResourceAmount a, ResourceAmount b);
    public static ResourceAmount operator -(ResourceAmount a, ResourceAmount b);

    public static ResourceAmount operator *(int a, ResourceAmount b);

    public static bool operator >=(ResourceAmount a, ResourceAmount b);
    public static bool operator <=(ResourceAmount a, ResourceAmount b);
    public static bool operator ==(ResourceAmount a, ResourceAmount b);
    public static bool operator !=(ResourceAmount a, ResourceAmount b);

    public static ResourceAmount Max(ResourceAmount a, ResourceAmount b);
    public static ResourceAmount Min(ResourceAmount a, ResourceAmount b);
}
```

The indexer by Resource allows reading and writing the amount of a specific resource through the enum, e.g. `amount[Resource.WOOD]`. Binary and unary `+` and `-` are supported, along with multiplication by an int on the left (`int * ResourceAmount`). Operators `>=`, `<=`, `==`, `!=` - there are no separate `<` and `>`. The `Max` and `Min` methods return the component-wise maximum and minimum of two ResourceAmount values. The `Total` property returns the sum of all resources. The static `ZERO` is a constant with all fields set to zero.

## ResourceStorage

The player's resource storage is represented by the `DiceKingdoms.Game.ResourceStorage` struct:

```c#
namespace DiceKingdoms.Game;

[StructLayout(LayoutKind.Explicit)]
public struct ResourceStorage
{
    [FieldOffset(0)]  public readonly ResourceAmount stored;
    [FieldOffset(32)] public readonly ResourceAmount capacity;

    public ResourceStorage(ResourceAmount stored, ResourceAmount capacity);

    public ResourceStorage WithCapacity(ResourceAmount capacity);
    public ResourceStorage WithStored(ResourceAmount stored);

    public override string ToString();
}
```

Two ResourceAmount fields - `stored` (the current amount of resources) and `capacity` (the storage's capacity). Both fields are declared readonly, making the struct immutable: once created, its state cannot be changed. `WithCapacity` returns a copy with the capacity replaced, `WithStored` — a copy with the stored amount replaced. The original instance remains unchanged.

## ResourceStorageInterface

A single resource storage widget is represented by the `DiceKingdoms.Interface.ResourceStorageInterface` class:

```c#
namespace DiceKingdoms.Interface;

public class ResourceStorageInterface : MonoBehaviour
{
    public ColorPalette colorPalette;
    public Resource resource;
    public Counter counterStored;
    public Counter counterCapacity;
    public Counter counterGain;
    public Image icon;
    public Ripple ripple;
    public RectTransform counters;
    public Sound incrementSound;
    public Interface @interface;

    public Resource Resource { get; }
    public int Stored { get; set; }
    public int Capacity { get; set; }
    public int? Gain { get; set; }
    public Image Icon { get; }

    protected void Awake();
    protected void OnEnable();
    protected void OnDisable();
    protected void Update();

    private void SetValuesFromIsland();
    public void Increment(int value);
}
```
This is a UI component for a single storage slot in the harvest panel (`HarvestInterface`). `HarvestInterfac`e holds an array of eight such widgets - one for each resource in the `Resource` enum - and uses them to display the storage values. 

Ten public fields bound in the prefab via the Unity Inspector: `colorPalette`, `resource` (the resource type this widget is responsible for), three counters `counterStored` / `counterCapacity` / `counterGain` (a custom UI component Counter), `icon` (resource icon), `ripple`, `counters` (a RectTransform container for the counters), `incrementSound` (gain sound), and `@interface` (a reference to the main Interface).

`SetValuesFromIsland()` - private, synchronizes the widget's values with the current storage state on the island. `Increment(int value)` - public, animates a resource gain: plays a sound and triggers a ripple effect.



