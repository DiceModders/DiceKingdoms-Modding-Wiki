---
summary: The combat phase - planning attacks, ships and landing, defenders, unit caps, the deterministic tick simulation, parry and PRD, projectiles, towers, buildings, pillage and the end of a fight
---

# Combat

!!! info "Code blocks"
    Declarations from the IL2CPP dump of game version 1.0.4, trimmed to what each section uses. `// 0x..` is the field offset, `// RVA 0x..` the method address in `GameAssembly.dll` (in Ghidra `0x180000000 + RVA`). Names stay stable between updates, offsets and RVAs do not. Private members are public in the BepInEx interop assemblies.

## Units

`Unit.Type`: ARCHER 0, BOURGEOIS 1, GENERAL 2, KING 3, PEASANT 4, PRIEST 5, SOLDIER 6, SORCERER 7, TEMPLAR 8, WARLOCK 9.
`Unit.Family`: INHABITANT 0, SOLDIER_MELEE 1, SOLDIER_RANGED 2.

<!-- code: Unit.Type |  -->
```csharp
public enum Unit.Type
{
    ARCHER = 0,
    BOURGEOIS = 1,
    GENERAL = 2,
    KING = 3,
    PEASANT = 4,
    PRIEST = 5,
    SOLDIER = 6,
    SORCERER = 7,
    TEMPLAR = 8,
    WARLOCK = 9,
}
```
<!-- /code -->

<!-- code: Unit.Family |  -->
```csharp
public enum Unit.Family
{
    INHABITANT = 0,
    SOLDIER_MELEE = 1,
    SOLDIER_RANGED = 2,
}
```
<!-- /code -->

<!-- code: Unit | *fields -->
```csharp
// DiceKingdoms.Game
public struct Unit
{
    public readonly Player owner;              // 0x0
    public readonly Unit.Type type;            // 0x8
    public readonly Deployment deployment;     // 0x10
    public readonly UnitStats.Stats stats;     // 0x18
    public int2 position;                      // 0x20
    public Unit.State state;                   // 0x28
    public int2 statePosition;                 // 0x2C
    public Nullable<Unit.Id> stateUnitTarget;  // 0x34
    public int stateEnterTick;                 // 0x3C
    public int stateExitTick;                  // 0x40
    public PseudoRandom parry;                 // 0x44
    public PseudoRandom block;                 // 0x50
    public PseudoRandom rangedMiss;            // 0x5C
}
```
<!-- /code -->

<!-- code: UnitStats.Stats | *fields -->
```csharp
public class UnitStats.Stats
{
    public int attackPeriod;                     // 0x10
    public int attackDelay;                      // 0x14
    public int agroDistance;                     // 0x18
    public UnitStats.Rational parryChance;       // 0x1C
    public UnitStats.Rational blockArrowChance;  // 0x24
    public Opt<UnitStats.Ranged> rangedOpt;      // 0x2C
    public Resource[] killRewardPool;            // 0x40
    public string lore;                          // 0x48
}
```
<!-- /code -->

**Patch:** the `UnitStats.Stats` objects can be changed at runtime; combat runs on every client, so every player needs the same change.

- **Soldiers are the "military" resource.** `UnDeployedUnits = stored military − DeployedUnits`.
- Inhabitants defend their own building (`BuildingTypeHousing.unitType`: Castle → King, Cottage → Peasant, Church → Priest, House → Bourgeois, Barracks → General, Sorcerer Tower → Sorcerer).
- **Units have no hit points.** A hit that is not parried or blocked kills.

`UnitStats` (times in combat ticks of 0.1 s):

| Unit | Attack period | Attack delay | Aggro distance | Parry | Block arrow | Ranged | Kill reward pool |
|---|---|---|---|---|---|---|---|
| Archer | 9 | 7 | 6 | 1/2 | – | range 6, archer arrow, miss 1/5 | – |
| Bourgeois | 8 | 4 | 256 | 1/5 | – | – | hammer, stone |
| General | 6 | 3 | 256 | 4/5 | 4/5 | – | gold, military |
| King | 4 | 2 | 256 | 4/5 | 4/5 | – | gold, hammer |
| Peasant | 12 | 6 | 256 | 1/6 | – | – | food, wood |
| Priest | 15 | 6 | 256 | 1/5 | – | – (converts) | culture |
| Soldier | 6 | 3 | 6 | 2/3 | – | – | – |
| Sorcerer | 10 | 4 | 184 | 1/3 | – | range 6, fireball, miss 1/2 | disaster |
| Templar | 6 | 3 | 6 | 3/4 | 2/3 | – | – |
| Warlock | 9 | 3 | 6 | 1/2 | – | range 6, fireball, miss 1/5 | – |

Aggro distance 256 = any enemy on the island. Ranges are in combat cells: the combat grid is finer than the island grid (`CombatGrid.FromIsland` / `ToIsland`), the Rules screen shows
range 6 as "3 tiles".

## Planning an attack

`CombatData.PlanAttack(AttackPlan)` (`0x62E8E0`) → `PlanAttackRPC`. `AttackPlan` = player, `shipPosition` (where the ship was dropped), `target` (building id), `unitCounts` per family,
`sourceDock`. The plan is **silently ignored** if:

<!-- code: CombatData.AttackPlan | *fields -->
```csharp
public struct CombatData.AttackPlan
{
    private readonly NetRef<Player> playerRef;                  // 0x0
    public readonly int2 shipPosition;                          // 0x8
    public readonly Building.Id target;                         // 0x10
    public readonly ValueTuple<Unit.Family, int>[] unitCounts;  // 0x18
    public readonly Building.Id sourceDock;                     // 0x20
}
```
<!-- /code -->

<!-- code: CombatData | AvailableDocks, DeployedUnits, UnDeployedUnits, PlannedAttacks, PlanAttack, PlanAttackRPC, IsSoldierFamilyUnlocked, SoldierFamilyUnitType -->
```csharp
// DiceKingdoms.Game
public class CombatData : PhaseData<CombatData, CombatIslandState>
{
    public List<Building.Id> AvailableDocks { get; }
    public int DeployedUnits { get; set; }
    public int UnDeployedUnits { get; }
    public List<CombatData.AttackPlan> PlannedAttacks { get; }

    public bool IsSoldierFamilyUnlocked(Unit.Family soldierFamily);                           // RVA 0x62E620
    public Unit.Type SoldierFamilyUnitType(Unit.Family soldierFamily);                        // RVA 0x62F690
    public static Unit.Type SoldierFamilyUnitType(Island island, Unit.Family soldierFamily);  // RVA 0x62F450
    private void PlanAttackRPC(CombatData.AttackPlan plan);                                   // RVA 0x62E880
    public void PlanAttack(CombatData.AttackPlan plan);                                       // RVA 0x62E8E0
}
```
<!-- /code -->

**Patch:** prefix `CombatData.PlanAttack` to read or block attack plans.

1. the units sum to more than `UnDeployedUnits`;
2. the target player is on the **same team** (you cannot attack yourself or a teammate);
3. a family with count > 0 is not unlocked;
4. the dock is not in `AvailableDocks`. The dock is then removed from the list: **one attack per dock per turn**.

`DeployedUnits` grows by the unit count. Not checked here: the ship capacity and the family cap. Those are enforced only by the attack UI and the AI, so a modified client could send more.

- **Ship capacity** `Statistics.ShipCapacity` = 8 × (1 + Ship Hold Optimization factor) = 8, or 16 with the technology.
- **Family cap** `CombatPhase.FamilyUnitCap` (`0x63FA80`): melee 999; other families 8 × the number of buildings of that family (8 ranged units per Archery Range).
- **Templars replace Soldiers, Warlocks replace Archers** (`SoldierFamilyUnitType`: base Soldier / Archer, replaced by a learnt `ReplaceSoldiers` technology), for attackers and defenders.

## Deployments

`CombatPhase.ApplyChanges` (`0x63D870`) builds the deployments for every island (`CreateDeployments` `0x63E280`), then the server prepares a combat for each island. The phase ends
when every island reported its fight finished.

<!-- code: CombatPhase | familyDeployOrder, ApplyChanges, CreateDeployments, FamilyUnitCap, ComputeShore, ComputeConnectedComponents -->
```csharp
// DiceKingdoms.Game
public class CombatPhase : TurnPhase<CombatData, CombatIslandState>
{
    public static Unit.Family[] familyDeployOrder;                                              // static

    public static void ComputeShore(Island island, List<int2> shores);                          // RVA 0x63DF70
    public static void ComputeConnectedComponents(Island island, IslandGrid.Layer<int2> connectedComponents);  // RVA 0x63DD90
    private void CreateDeployments(List<Deployment>[] deployments);                             // RVA 0x63E280
    public static int FamilyUnitCap(Island island, Unit.Family family);                         // RVA 0x63FA80
    protected override void ApplyChanges(Action done);                                          // RVA 0x63D870
}
```
<!-- /code -->

<!-- code: Deployment | Owner, UnitTypes, Counts, SourceBuilding, Left, OnLeave, AttackDeployment, DefenseSoldierDeployment, DefenseHouseDeployment, Leave -->
```csharp
// DiceKingdoms.Game
public class Deployment
{
    public Player Owner { get; }
    public Unit.Type[] UnitTypes { get; }
    public int[] Counts { get; }
    public Building.Id SourceBuilding { get; }
    public bool Left { get; set; }
    public Action<Unit.Id[]> OnLeave { get; set; }

    public void Leave(Unit.Id[] survivors);                                                     // RVA 0x644E00
    public static Deployment DefenseSoldierDeployment(Island island, Building.Id garrison, int count, Unit.Type soldierType);  // RVA 0x644950
    public static Deployment DefenseHouseDeployment(Island island, Building.Id house);          // RVA 0x6444B0
    public static Deployment AttackDeployment(Player owner, Building.Id sourceDock, Island island, IslandGrid.Layer<bool> shipOccupied, IslandGrid.Layer<int2> connectedComponents, Building.Id target, int2 shipPosition, ValueTuple<Unit.Type, int>[] counts, Random random);  // RVA 0x643680
}
```
<!-- /code -->

**Patch:** postfix `CombatPhase.FamilyUnitCap` for other unit caps; `CreateDeployments` (private) decides who defends.

**Attacks:**

1. All plans of all players are **shuffled** with the match random generator; the order decides who gets the better landing spot.
2. For each plan the target's shore tiles are tried **nearest first** to where the ship was dropped (`Deployment.AttackDeployment` `0x643680`). If none works, the attack is
   **dropped**: no ship sails, nothing is lost, and those soldiers defend at home this turn (confirmed in game).
3. Soldiers sent on an attack **do not defend their own island**.

**Landing** (`Deployment.Ship.BestDebarkTransform` `0x64CF60`): the ship needs a free 2×3 spot of water (sand allowed at the two front corners), not taken by another ship, in one of the 4
orientations, whose landing tile is in the **same connected land area as the target building** (or is a tile of it). Walls or buildings that cut the target off from every reachable
shore make the attack impossible. 8 landing cells per ship; more soldiers come off in waves.

**Defenders:**

- `d = min(stored military − units sent away, DefLimit)` (`DefLimit` 999; developer command `deflimit`).
- Garrison buildings (Castle, Barracks, Archery Range) get them, **ranged first**: `n = min(d, family cap)` per family, split evenly over that family's buildings.
  Example: 20 military, 1 Archery Range, Castle + 1 Barracks → 8 archers at the range, 12 melee split 6/6.
- Every **active housing** building also gets its inhabitants as defenders.

An island only gets a fight if somebody attacks it; its own defence deployments alone do not start one.

## The simulation

`CombatGrid` runs a **deterministic, tick-based** fight **on every client**: the server sends the deployment list, every machine simulates it with software floats (`sfloat`), grid
positions, a Burst pathfinding job and seeded randomness. `CombatGrid.Hash()` is logged every 100 ticks (`Turn {0} - Island {1} - Tick {2} - Hash {3:x8}`) to catch desyncs.

<!-- code: CombatGrid | TICK_FREQUENCY, MAX_TICKS, MAX_IDLE_TICKS, BUILDING_PROTECT_UNIT_CHANCE_PERCENT, Init, SimulationStep, Hash, End, FromIsland, ToIsland -->
```csharp
// DiceKingdoms.Game
public class CombatGrid : MonoBehaviour
{
    private const int TICK_FREQUENCY = 10;
    private const int MAX_TICKS = 3000;
    private const int MAX_IDLE_TICKS = 10;
    private const int BUILDING_PROTECT_UNIT_CHANCE_PERCENT = 25;

    public void Init(IslandGrid islandGrid, Deployment[] deployments, Func<float> time, Action done);  // RVA 0x633060
    public static int2 ToIsland(int2 combatPosition);                                           // RVA 0x63C4B0
    public static int2[] FromIsland(int2 islandPosition);                                       // RVA 0x632510
    private void End();                                                                         // RVA 0x631C40
    private bool SimulationStep(CombatGrid.PathfindingJob completedJob);                        // RVA 0x636E10
    public int Hash();                                                                          // RVA 0x632720
}
```
<!-- /code -->

- One tick = 0.1 s (`TICK_FREQUENCY` 10). The fight stops at **3000 ticks** (5 minutes) or after the grid was idle for more than 9 ticks.
- It starts after `clamp(5 s − time since the phase sync, 0, 5 s)`.

`SimulationStep` (`0x636E10`), one tick:

1. Take pathfinding results.
2. Step every unit, deployment (landing), tower and projectile.
3. Resolve the unit hits of this tick in random order: the target's **parry** roll, else it **dies**.
4. Each death: the dead unit's owner loses **1 military**; if the killer is an enemy, the killer's owner gets **+1 of a random resource from the dead unit's kill reward pool**.
5. Conversions (Priests): the old owner loses 1 military, the new owner gains 1, and the unit changes sides.
6. Building hits: HP −= damage.
7. Each destroyed building: if the attacker is not the island's owner, the attacker gets the building's **pillage reward**; the building is removed and leaves remains.

### Units

<!-- code: CombatGrid | UnitHit, ConvertUnit, BuildingHit, MeleeBuildingDamage -->
```csharp
// DiceKingdoms.Game
public class CombatGrid : MonoBehaviour
{
    private void UnitHit(Player source, Unit.Id target, float3 hitDirection);     // RVA 0x63C6F0
    private void ConvertUnit(Player source, Unit.Id target);                      // RVA 0x631AC0
    private void BuildingHit(Player source, Building.Id target, int damage = 1);  // RVA 0x631720
    private int MeleeBuildingDamage(in Unit unit, in Building.Id buildingId);     // RVA 0x634C20
}
```
<!-- /code -->

**Patch:** postfix `CombatGrid.MeleeBuildingDamage` to change melee damage to buildings.

- States: idle, moving, attacking. A hit lands at `enterTick + attackDelay`.
- **Targets** (`FindTarget` `0x631D10`): an enemy within 2 tiles always wins; else a defender within aggro distance; else (attackers) the nearest standing tile of the **target building**,
  or, once it is gone, a free exit tile on the ship. Defenders and inhabitants always go for the nearest enemy.
- An idle unit attacks anything enemy next to it (`AttackAnyOrIdle`): attackers also hit other enemy buildings they stand next to.
- **Priests** do not hit: they convert the unit on the target tile (only soldiers convert).
- **Melee on a building tile**: hits the building when no unit stands there, or with a **25 %** roll when one does (`BUILDING_PROTECT_UNIT_CHANCE_PERCENT`).
- Melee damage to buildings is **1**, or **2 on a Wall** when the attacker stands on its landing area.
- A hit on a Wall also damages every Wall directly next to it (one step, not a chain), so wall lines take damage in groups.

### Projectiles and towers

<!-- code: Projectile.Type |  -->
```csharp
public enum Projectile.Type
{
    ARROW_TOWER = 0,
    ARROW_ARCHER = 1,
    FIREBALL = 2,
}
```
<!-- /code -->

<!-- code: ProjectileTypeExtensions | *methods -->
```csharp
// DiceKingdoms.Game
public static class ProjectileTypeExtensions
{
    public static int Duration(Projectile.Type type, int2 source, int2 target);  // RVA 0x64C2B0
    public static int BuildingDamage(Projectile.Type type);                      // RVA 0x64C1F0
    public static float HeightEndOffset(Projectile.Type type);                   // RVA 0x64C3C0
}
```
<!-- /code -->

<!-- code: CombatGrid | ShootProjectile -->
```csharp
// DiceKingdoms.Game
public class CombatGrid : MonoBehaviour
{
    private void ShootProjectile(Player owner, int2 source, int2 target, Projectile.Type type, ref PseudoRandom miss, float visualStartHeight);  // RVA 0x636BF0
}
```
<!-- /code -->

**Patch:** postfix `ProjectileTypeExtensions.BuildingDamage` (tested: the Rules screen shows the new value too).

| Projectile | Building damage | Flight time (ticks, d = distance in tiles) |
|---|---|---|
| Tower arrow | 0 | d × 20 / 100 + 5 |
| Archer arrow | 1 | d × 50 / 100 + 1 |
| Fireball (Sorcerer, Warlock) | 8 | d + 1 |

- When a projectile lands: building tile → hits the building (no unit there, or the 25 % roll); else a unit on the tile: **fireball kills**, an arrow kills unless the unit's
  **block** roll succeeds. Arrow and fireball deaths do not use the parry roll.
- **There is no team check** where a projectile lands: allies and the defender's own buildings can be hit. Defenders' fireballs can destroy their own buildings (no pillage reward then).
- **Miss**: the shooter's miss roll moves the landing tile one tile in a random direction; the projectile still lands and hits what is there. Shooters aim where the target will be.
- **Towers** (Castle, Guard Tower, Guard Tower Up, Lighthouse): every tick the cooldown counts down; at 0 the tower picks among enemy units in range with score = random(0..255) − distance²
  (close units strongly preferred), shoots with its own miss roll, and waits `period − 4 + random(0..8)` ticks.

| Tower | Range | Period (ticks) |
|---|---|---|
| Castle | 60 | 30 |
| Guard Tower | 30 | 20 |
| Guard Tower Up | 45 | 13 |
| Lighthouse | 15 | 10 |

### Parry, block and miss rolls (PRD)

Parry, arrow block, ranged miss and tower miss use `PseudoRandom` (`Sample` `0x64C790`), like Dota's pseudo-random distribution: the chance drops after a success and rises after a failure,
always pulled back toward the base value, so long streaks are rarer than with a plain random roll.

<!-- code: PseudoRandom | *fields, *methods -->
```csharp
// DiceKingdoms.Game
public struct PseudoRandom
{
    public sfloat probability;                 // 0x0
    private readonly bool init;                // 0x4
    private sfloat nextProbability;            // 0x8
    private static readonly sfloat factor;     // static
    private static readonly sfloat adjusted;   // 0x4

    private static sfloat Decrease(sfloat x);  // RVA 0x64C4C0
    private static sfloat Increase(sfloat x);  // RVA 0x64C600
    public bool Sample(ref Random rand);       // RVA 0x64C790
    public sfloat P(int i);                    // RVA 0x64C6A0
}
```
<!-- /code -->

```
next = 0.9 × f(next) + 0.1 × base
f = Decrease(next) after a success,  Increase(next) after a failure
Decrease(x) = 1 − (1 − x) × exp(5 × x² × (1 − x)²)
Increase(x) = 1 − Decrease(1 − x) = x × exp(5 × x² × (1 − x)²)
```

Example: Decrease(0.5) = 0.317, Increase(0.5) = 0.683.

## End of a fight

<!-- code: Deployment | Leave, OnLeave -->
```csharp
// DiceKingdoms.Game
public class Deployment
{
    public Action<Unit.Id[]> OnLeave { get; set; }

    public void Leave(Unit.Id[] survivors);  // RVA 0x644E00
}
```
<!-- /code -->

<!-- code: CombatGrid | End -->
```csharp
// DiceKingdoms.Game
public class CombatGrid : MonoBehaviour
{
    private void End();  // RVA 0x631C40
}
```
<!-- /code -->

- A fight normally ends when all attackers are dead or the target's Castle burns. Surviving attackers then walk back to their exit tiles and `Deployment.Leave(survivors)` sails them home.
- `CombatGrid.End()` (`0x631C40`) calls `Leave` with an empty list for deployments that did not leave yet (only visual).
- **Soldiers are only lost by dying** (−1 military each) or being converted; the fight ending loses nobody (confirmed in game).

## Power

`PowerConstants` gives each unit type and the Castle, Guard Tower, Guard Tower Up and Wall a power value (Warlock 3, Templar 2, Sorcerer 2, Guard Tower Up 16, Guard Tower 8, Castle 4, …).
`AttackPower(island)` / `DefensePower(island)` are shown in the player list and used by the [AI](ai.md).

<!-- code: PowerConstants | *methods -->
```csharp
// DiceKingdoms.Game
public class PowerConstants : ScriptableObject
{
    public float UnitPower(Unit.Type unitType);                     // RVA 0x621490
    public float BuildingPower(BuildingTypeName buildingTypeName);  // RVA 0x620EC0
    public float AttackPower(Island island);                        // RVA 0x620BE0
    public float DefensePower(Island island);                       // RVA 0x620F00
}
```
<!-- /code -->

## Developer helpers

`deflimit` limits the number of defenders; `MiscCommands.DebugAttack` shows attack debug info. See [Developer console](dev-console.md).
