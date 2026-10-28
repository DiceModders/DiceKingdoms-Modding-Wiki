---
summary: The disasters phase - which disaster happens, who it hits, what each one does, food upkeep, playback order and the camera lock
---

# Disasters

*Code: [where to patch and the declarations](#code-reference).*

## Data

The `Disasters` asset holds one object of each disaster class, in this order (`Values`, index 0..12):
starvation, robbers, disease, lightning, fire, thunderstorm, revolt, tornado, plague, satanism, pirates, tsunami, meteorite.
Base class `Disaster`: `name` `0x10`, `icon` `0x18`, `affectedOthers` `0x20`, `threshold` `0x24`.

| # | Disaster | Threshold (disaster points) | Hits | Parameters |
|---|---|---|---|---|
| 0 | Starvation | −1 (every turn) | self | – |
| 1 | Robbers | 1 | 1 other player | losable: food, wood, stone |
| 2 | Disease | 2 | self | 50 % of candidates |
| 3 | Lightning | 3 | 1 other player | 2 tiles |
| 4 | Fire | 4 | self | – |
| 5 | Thunderstorm | 5 | 1 other player | 5 tiles |
| 6 | Revolt | 6 | self | 50 % of military lost |
| 7 | Tornado | 7 | 2 other players | – |
| 8 | Plague | 8 | self | 50 % of candidates |
| 9 | Satanism | 9 | 2 other players | – |
| 10 | Pirates | 10 | self | losable: wood, stone, gold; 50 % lost |
| 11 | Tsunami | 11 | all other players | radius 5, start distance 28, 2550 ms to impact |
| 12 | Meteorite | 12 | self | radius 4; see [Meteorites](meteorites.md) |

## Which disaster happens

`DisastersPhase.RandomOccurrences` (`0x646940`), for every island:

1. `d` = the island's stored **disaster points**.
2. `idx = FindLastIndex(Values, v => v.threshold >= 0 && v.threshold <= d)`. The thresholds go up in order, so **only one disaster per island happens: the strongest one whose threshold
   was reached.** None if `d` is 0.
3. Normally one occurrence. With the lobby option **Multi-meteor** and the meteorite: `d − 12 + 1` meteors (every point over 12 adds one).
4. Targets:
    - the developer command `forcedisaster` makes the chosen disaster hit the island itself;
    - `affectedOthers < 1`: it hits the island that caused it;
    - otherwise the other islands whose owner is on another team (free-for-all: every other player) and **not defeated** are shuffled with the match random generator, and the first
      `affectedOthers` become targets. Teammates and defeated players are never targeted.
5. Each `DisasterOccurrence(source, target, disaster, seed)` gets its own seed from the match random generator, so every client plays it the same way.

Then:

- **Starvation is always added** for every island (source = target = self), after its normal disaster.
- **After all disasters ran, every island's disaster points are reset to 0.**

`ExecuteOccurrencesRPC` sends the list to all clients; each `DisasterOccurrence.Execute(GameState)` is a coroutine of delays.

## Affected buildings

`Building.Active` is simply `affectedByDisaster == null` (`0x5F15A0`). An affected building (hungry, sick, …) **does not roll its die** and does not count as active population for that turn.
The building tints (hungry / sick / burnt / died in combat) show the state.

## Effects

| Disaster | Effect |
|---|---|
| Starvation (food upkeep) | `needed` = number of **active housing** buildings (Castle, Cottage, House, Church, Barracks, Sorcerer Tower); each eats **1 food**; `min(needed, stored food)` is removed. For every missing food one random active housing building becomes **hungry**. |
| Robbers | Picks a random resource among food / wood / stone that the target has: **the target loses 1, the sender gains 1**. |
| Disease | Candidates = buildings that **can get sick** and touch a building that **can contaminate**; half of them (rounded down) become sick for the turn. |
| Lightning | A random free grass tile and a connected blob of up to 2 free tiles get **scorched earth** (`IslandGrid.StormDisaster`). Never hits buildings; scorched earth blocks building until it is cleaned. |
| Fire | One random **burnable** building on your island is destroyed. |
| Thunderstorm | Like Lightning with 5 tiles. |
| Revolt | You lose half your stored military (rounded down): 1 soldier → 0 lost, 3 → 1. |
| Tornado | Like Fire, on each target island. |
| Plague | Like Disease, but every building that can get sick is a candidate (no contaminating neighbour needed); Hospitals still protect. |
| Satanism | A random house is replaced by a **Sorcerer Tower** (inside the old tiles, full HP), which then produces more disaster points. |
| Pirates | You lose half (rounded down) of your wood, stone and gold. Nobody receives them. |
| Tsunami | See below. |
| Meteorite | Crater of radius 4, scorched earth and lava, with castle protection: [Meteorites](meteorites.md). |

**Burnable** = not a fortification and not a Castle. Only the Wall is a fortification, so Fire and Tornado can destroy anything except Walls and Castles (Guard Towers included).

**Can get sick** (`IslandGrid.CanGetSick` `0x613D70`): housing, producers except the Farm, technology buildings, the Market; but **not** if a Hospital touches it or touches a
non-wall neighbour of it (the Hospital protects its neighbours and their neighbours; walls block it).
**Can contaminate**: not a wall, not a Farm, not a Hospital, and no Hospital next to it.

### Tsunami

- Impact point (`GetTsunamiImpact` `0x615C00`): from a random land tile, a 3-wide ray toward the island centre; the first land tile with water next to it is the impact.
- Tiles (`GetTsunamiTiles` `0x625890`): land tiles in rings up to radius 5 around the impact with a clear straight line from it. The line may not cross a hill edge (`INNER_CLIFF`),
  so **tiles behind a hill are safe** (confirmed in game).
- Each tile (`IslandGrid.TsunamiDisaster` `0x61C970`): **any building on it is removed** (there is no Castle exception: a tsunami touching one tile of a Castle burns the whole Castle;
  confirmed in game), then grass, sand, meteorite ground and piers become **water** and lose their nature. Water rocks and hill edges stay. A Land Camp can turn the water back into land.

## Playback and the camera

`DisastersPhase.ExecuteOccurrences` (`0x645900`) runs on every client and builds one sequence:

1. **Small disasters** (everything except tornado, tsunami and meteorite): the camera flies to **your own island** (0.8 s) and is locked; they all play **at the same time** on all islands.
2. **Big disasters** (tornado, tsunami, meteorite), grouped by target island: for each, the camera flies to that island (0.8 s, locked), waits 0.5 s, plays them one after another,
   then waits 1 s.
3. The camera flies back to your island (0.5 s) and is released. If you are defeated, it is just released where it is.
4. Each client then acknowledges the end; the server moves on when every client has.

While `CameraMovement.focus` has a value, `CameraMovement.Update` zeroes all camera input, so **the player cannot move the camera during the disaster animations**. The developer console's
free camera (`MiscCommands.FreeCam`) skips the focus. A mod that wants free camera control can clear `cameraMovement.focus` (set it to `null`) every frame during this phase; it is display only and does not affect sync.

The visual effects of each disaster are listed in [Rendering](rendering.md#visual-effects).

## Code reference

Declarations from the IL2CPP dump of game version 1.0.4, trimmed to the members a patch usually needs. `// 0x..` is the field offset, `// RVA 0x..` the method address in `GameAssembly.dll` (in Ghidra: `0x180000000 + RVA`). Names are stable between updates, offsets and RVAs are not. Private members are public in the BepInEx interop assemblies.

### Where to patch

| To | Patch / use |
|---|---|
| Change which disaster happens | `DisastersPhase.Disaster(int, out int)` (picks by disaster points) or `DisastersPhase.RandomOccurrences` |
| Change a disaster's numbers | fields of the disaster objects in `gameState.disastersPhase.disasters` (`Robbers.losableResources`, `Disease.percentageAffected`, ...) |
| Change what a disaster does | the disaster's `Execute` coroutine, or the `IslandGrid` method it calls (`StormDisaster`, `TsunamiDisaster`, `SatanismDisaster`, ...) |
| Change food upkeep | `Starvation.Execute` / `IslandGrid.ActivePopulation` |
| Free camera during disasters | clear `CameraMovement.focus` every frame while `DisastersPhase` is active |

### Declarations

```csharp
// DiceKingdoms.Game
public abstract class Disaster
{
    public string name;                                                                         // 0x10
    public Sprite icon;                                                                         // 0x18
    public int affectedOthers;                                                                  // 0x20
    public int threshold;                                                                       // 0x24

    public abstract IEnumerable<int> Execute(GameState gameState, DisasterOccurrence occurrence, Random random);
}
```

```csharp
// DiceKingdoms.Game
public class Disasters : ScriptableObject
{
    public Starvation starvation;      // 0x18
    public Robbers robbers;            // 0x20
    public Disease disease;            // 0x28
    public Lightning lightning;        // 0x30
    public Fire fire;                  // 0x38
    public Thunderstorm thunderstorm;  // 0x40
    public Revolt revolt;              // 0x48
    public Tornado tornado;            // 0x50
    public Plague plague;              // 0x58
    public Satanism satanism;          // 0x60
    public Pirates pirates;            // 0x68
    public Tsunami tsunami;            // 0x70
    public Meteorite meteorite;        // 0x78

    public Disaster[] Values { get; set; }

    private void Init();               // RVA 0x6476D0
    private void OnEnable();           // RVA 0x647C10
}
```

```csharp
// DiceKingdoms.Game
public class DisasterOccurrence
{
    private NetRef<Player> SourceRef { get; }
    private NetRef<Player> TargetRef { get; }
    public Disaster Disaster { get; }
    public Player Source { get; }
    public Player Target { get; }
    public uint Seed { get; }

    public IEnumerable<int> Execute(GameState gameState);  // RVA 0x645370
}
```

```csharp
// DiceKingdoms.Game
public class DisastersPhase : TurnPhase<DisastersData, DisastersIslandState>
{
    private Acknowledgeable disastersEnd;                                             // 0xE8
    private Disasters disasters;                                                      // 0xF0
    private DisasterOccurrence[] occurrencesSnapshot;                                 // 0xF8

    private void ExecuteOccurrencesRPC(DisasterOccurrence[] occurrences);             // RVA 0x6457D0
    private void ExecuteOccurrences(DisasterOccurrence[] occurrences, Action onEnd);  // RVA 0x645900
    public Disaster Disaster(int disasters, out int count);                           // RVA 0x645630
    private DisasterOccurrence[] RandomOccurrences();                                 // RVA 0x646940
    public override void Reset();                                                     // RVA 0x647650
    protected override void ApplyChanges(Action done);                                // RVA 0x645530
    public override void Rejoin(Player player);                                       // RVA 0x647520
    public override void Exit();                                                      // RVA 0x6468F0
}
```

```csharp
// DiceKingdoms.Game
public class Robbers : Disaster
{
    private Resource[] losableResources;                     // 0x28
    public ResourceParticle resourceParticlePrefab;          // 0x30
    private readonly List<Resource> storedLosableResources;  // 0x38
}
```

```csharp
// DiceKingdoms.Game
public class Disease : Disaster
{
    private Transform visualPrefab;                                                             // 0x28
    private int percentageAffected;                                                             // 0x30

    public override IEnumerable<int> Execute(GameState gameState, DisasterOccurrence occurrence, Random random);  // RVA 0x6481A0
    public static bool CanContaminate(Building building, Island island);                        // RVA 0x647DB0
    public static bool CanAffect(Building building, Island island);                             // RVA 0x647C30
    public static bool DoesRiskMoreDisease(Building building, Island island);                   // RVA 0x648000
}
```

```csharp
// DiceKingdoms.Game
public class Lightning : Disaster
{
    public int nbTile;                     // 0x28
    private LightningVisual visualPrefab;  // 0x30
    private DayNightCycle dayNightCycle;   // 0x38
}
```

```csharp
// DiceKingdoms.Game
public class Thunderstorm : Disaster
{
    public int nbTile;                     // 0x28
    private LightningVisual visualPrefab;  // 0x30
    private DayNightCycle dayNightCycle;   // 0x38
}
```

```csharp
// DiceKingdoms.Game
public class Revolt : Disaster
{
    public ResourceParticle resourceParticlePrefab;  // 0x28
    private int percentageLost;                      // 0x30
}
```

```csharp
// DiceKingdoms.Game
public class Plague : Disaster
{
    private Transform visualPrefab;  // 0x28
    private int percentageAffected;  // 0x30
}
```

```csharp
// DiceKingdoms.Game
public class Pirates : Disaster
{
    private Resource[] losableResources;             // 0x28
    private int percentageLost;                      // 0x30
    public ResourceParticle resourceParticlePrefab;  // 0x38
}
```

```csharp
// DiceKingdoms.Game
public class Tsunami : Disaster
{
    private int radius;                             // 0x28
    private int startDistance;                      // 0x2C
    private int startToImpactDurationMilliseconds;  // 0x30
    private TsunamiTileVisual tileVisualPrefab;     // 0x38
    private TsunamiVisual visualPrefab;             // 0x40
}
```

```csharp
// DiceKingdoms.Game
public class Meteorite : Disaster
{
    private MeteoriteVisual visual;                   // 0x28
    private int radius;                               // 0x30
    private int minimumCastleTiles;                   // 0x34
    private int meteoriteTilesToFullyAllowHitCastle;  // 0x38
}
```

```csharp
// DiceKingdoms.Game
public class IslandGrid : IslandSyncedBehaviour<IslandGrid.SyncData>
{
    public int ActivePopulation { get; }

    public void MeteoriteDisaster(int2 impact, int radius);             // RVA 0x616890
    public bool CanGetSick(BuildingType buildingType);                  // RVA 0x614280
    public bool CanGetSick(Building building);                          // RVA 0x613D70
    public void SatanismDisaster(Building house);                       // RVA 0x61A1F0
    public void StormDisaster(int2 position, bool4 connected);          // RVA 0x61AD40
    public Nullable<Building> GetRandomBurnable(ref Random random);     // RVA 0x615710
    public void FireDisaster(Building.Id buildingId);                   // RVA 0x614AF0
    public Nullable<int2> GetTsunamiImpact(ref Random random);          // RVA 0x615C00
    public IEnumerable<int2> GetTsunamiTiles(int2 center, int radius);  // RVA 0x615EB0
    public void TsunamiDisaster(int2 position);                         // RVA 0x61C970
}
```

```csharp
// DiceKingdoms.Game
public struct Building : IEquatable<Building>
{
    public Disaster affectedByDisaster;  // 0x18

    public bool Active { get; }
}
```
