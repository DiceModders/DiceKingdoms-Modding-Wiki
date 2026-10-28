---
summary: How the AI plays - difficulties, the rubber band, build orders and the custom build-order file, AIConfiguration values, how it harvests, builds, places buildings, cleans, trades and attacks
---

# AI

!!! info "Code blocks"
    Declarations from the IL2CPP dump of game version 1.0.4, trimmed to what each section uses. `// 0x..` is the field offset, `// RVA 0x..` the method address in `GameAssembly.dll` (in Ghidra `0x180000000 + RVA`). Names stay stable between updates, offsets and RVAs do not. Private members are public in the BepInEx interop assemblies.

- `AIPlay` (a component on the `GameState` object) runs **on the server only**. `AIPlay.PlayPhase(phaseData, AIState) → AIState` (`0x567640`) is called **once per phase, when the phase ends
  (at sync)**, for every AI island; it dispatches to `PlayStart`, `PlayHarvest`, `PlayBuild` and the combat code. So the AI never acts during a phase, only at its end.
- `Player.AIState` (networked): `Difficulty`, `Strategy`, `TechnologiesToResearch`, `BuildingToPlace` (its plan, kept between phases).
- Difficulties: PASSIVE 0, PEASANT, BOURGEOIS, GENERAL, KING, SORCERER. The lobby offers Peasant to Sorcerer; **Passive is only used by the tutorial**.
- Strategies: MILITARY 0, WONDER 1. WONDER is only allowed when the lobby option Wonder Victory is on.
- `GameState.AddAI` adds a bot; matchmaking can fill empty slots with AIs.

## AIConfiguration

A MonoBehaviour asset inside `sharedassets1.assets` (not a loose file). Changing it needs a plugin (edit the object at runtime).

The AI tuning values and the per-player AI state:

<!-- code: AIConfiguration | *fields -->
```csharp
namespace DiceKingdoms.Game;

public class AIConfiguration : ScriptableObject
{
    public int hammerToKeep;                                        // 0x18
    public int shipPerBuilding;                                     // 0x1C
    public int sorcererTowerLimit;                                  // 0x20
    public int turnCountLimit;                                      // 0x24
    public int topBuildingCountToPlace;                             // 0x28
    public float attackScoreRandomness;                             // 0x2C
    public AIConfiguration.DifficultyLevelConfiguration passive;    // 0x30
    public AIConfiguration.DifficultyLevelConfiguration peasant;    // 0x60
    public AIConfiguration.DifficultyLevelConfiguration bourgeois;  // 0x90
    public AIConfiguration.DifficultyLevelConfiguration general;    // 0xC0
    public AIConfiguration.DifficultyLevelConfiguration king;       // 0xF0
    public AIConfiguration.DifficultyLevelConfiguration sorcerer;   // 0x120
    public int turnCountBerserk;                                    // 0x150
    public float soldierFactor;                                     // 0x154
    public float archerFactor;                                      // 0x158
    public float templarFactor;                                     // 0x15C
    public float sorcererFactor;                                    // 0x160
    public float requireShipFactor;                                 // 0x164
    public float guardTowerFactor;                                  // 0x168
    public float towerUpgradeFactor;                                // 0x16C
    public float wallFactor;                                        // 0x170
    public float inhabitantFactor;                                  // 0x174
    public float builtResourcesFactor;                              // 0x178
    public float storedResourcesFactor;                             // 0x17C
    public float castleFactor;                                      // 0x180
}
```
<!-- /code -->

<!-- code: Player.AIState | *fields -->
```csharp
public struct Player.AIState
{
}
```
<!-- /code -->

<!-- code: Player.AIState.DifficultyLevel |  -->
```csharp
public enum Player.AIState.DifficultyLevel
{
    PASSIVE = 0,
    PEASANT = 1,
    BOURGEOIS = 2,
    GENERAL = 3,
    KING = 4,
    SORCERER = 5,
}
```
<!-- /code -->

<!-- code: Player.AIState.AIStrategy |  -->
```csharp
public enum Player.AIState.AIStrategy
{
    MILITARY = 0,
    WONDER = 1,
}
```
<!-- /code -->

**Patch:** change the fields of `AIPlay.configuration` at runtime (host).

| Constant | Value | Use |
|---|---|---|
| `hammerToKeep` | 5 | |
| `shipPerBuilding` | 2 | ships per target building |
| `sorcererTowerLimit` | 2 | extra Sorcerer Towers are burnt |
| `turnCountLimit` | 3 | early-game rules end after turn 3 |
| `topBuildingCountToPlace` | 6 | how far it looks ahead in its build list |
| `attackScoreRandomness` | 200 | |
| `turnCountBerserk` | 11 | after turn 11 + random(−1..2) it targets Castles |

| Difficulty | `limitBuildingScore` | `aggressiveness` | `defensiveness` | `maxLavaToClean` |
|---|---|---|---|---|
| Passive | 0.7 | 0.9 | 1 | 2 |
| Peasant | 0.8 | 1.1 | 0.8 | 2 |
| Bourgeois | 1.2 | 1.1 | 1 | 2 |
| General | 2 | 1 | 1.1 | 2 |
| King | 10 | 1.3 | 1.2 | 50 |
| Sorcerer | 10 | 1.2 | 1.4 | 50 |

Each difficulty also has a **`harvestMultiplier` per resource** that multiplies the AI's final harvest (for example Peasant: food ×1.5, wood/stone/gold/hammer ×0.9, culture/military ×0.7;
Bourgeois: food ×1.5, wood/stone/hammer ×1.1; Sorcerer: food/wood/stone/hammer ×1.7). **AI income is not the plain dice result.**

Scoring weights: soldier 4, archer 8, templar 9, sorcerer 18, requireShip 0.3, guardTower 32, towerUpgrade 72, wall 0.2, inhabitant 0.4, builtResources 2, storedResources 1.2, castle 10.

## Build orders

`AIPlay.InitBuildOrders` (`0x564260`) loads 15 build orders (5 civilizations × `culture`, `wonder`, `military_macro`) from a text compiled into the game. **If the file
`custom-build-orders.txt` exists in the game folder (next to `Dice-Kingdoms.exe`), it is read instead** (confirmed in game; the log still says "Loaded 15 build orders."). So the AI's build
orders can be changed with a text file alone, no plugin needed. Keep the same format:

Build orders are loaded and picked in `AIPlay`:

<!-- code: AIPlay | customBuildOrdersPath, InitBuildOrders, SampleBuildOrder -->
```csharp
namespace DiceKingdoms.Game;

public class AIPlay : MonoBehaviour
{
    private string customBuildOrdersPath;                                                       // 0x40

    private void InitBuildOrders();                                                             // RVA 0x564260
    private AIPlay.BuildOrder SampleBuildOrder(ref Random random, Civilization civilization, Player.AIState.AIStrategy[] validStrategies);  // RVA 0x568710
}
```
<!-- /code -->

```
civilization=VIKING weight=2 strategy=MILITARY name=military_macro [
    technologies [ UNLOCK_FISHING_SHIP UNLOCK_ARCHERY_RANGE ... ]
    buildings [ house house church cot cot ... ]
]
```

Building short names: house, cot (Cottage), church, farm, racks (Barracks), dock, land (Land Camp), storage, mill (Windmill), fish (Fishing Ship), guard, cat (Cathedral), university,
market, lumber, quarry, archery, wonder, sorcerer. `SampleBuildOrder` (`0x568710`) picks one at random, weighted, among those for the AI's civilization and allowed strategies
(an unknown civilization throws). The log line `AI <name> has strategy MILITARY and build order military_macro` comes from `PlayStart`.

## Start phase

`PlayStart` (`0x567FE0`) places the start buildings with a spiral from the centre, 4 orientations from a random start, **Farms only at Manhattan distance ≥ 15 from the centre**,
no extra disease risk. Then it picks its strategy and build order.

Every phase starts in `AIPlay.PlayPhase`:

<!-- code: AIPlay | PlayPhase, PlayStart -->
```csharp
namespace DiceKingdoms.Game;

public class AIPlay : MonoBehaviour
{
    public Player.AIState PlayPhase(IPhaseData data, Player.AIState state);  // RVA 0x567640
    public Player.AIState PlayStart(StartData data, Player.AIState state);   // RVA 0x567FE0
}
```
<!-- /code -->

**Patch:** prefix / postfix `AIPlay.PlayPhase` to change what the AI does at the end of any phase.

## Harvest

`PlayHarvest` (`0x566620`), after the AI's dice were rolled, for each re-roll:

The harvest decision:

<!-- code: AIPlay | PlayHarvest -->
```csharp
namespace DiceKingdoms.Game;

public class AIPlay : MonoBehaviour
{
    public Player.AIState PlayHarvest(HarvestData data, Player.AIState state);  // RVA 0x566620
}
```
<!-- /code -->

- It predicts the next disaster. If that would be Plague, Disease (after turn 3), Pirates (WONDER AI) or Revolt (MILITARY AI), it re-rolls up to 5 dice **without** a disaster point,
  lowest yield first (dice showing a disaster point cannot be re-rolled, for players either).
- Only before turn 3: food management (re-roll Cottages that gave no food when food is short) and the strategy re-roll (MILITARY: every Barracks die under 3 military; WONDER: none).
- Its hammer yield is raised so it ends with at least 2 hammers.
- The yields are multiplied by the difficulty's `harvestMultiplier` (`HarvestData.SetAIYields`).
- Passive AI only: military limited to half the highest human's stored military, culture to 1.5 × the highest.

## Build

`PlayBuild` (`0x565500`):

Building, cleaning and market use:

<!-- code: AIPlay | PlayBuild, BuildAndClean, WhereToPlaceBuilding, PlaceBuildingSomewhere, PlaceBuildingAt, CleanRemains, CleanNature, UseMarket -->
```csharp
namespace DiceKingdoms.Game;

public class AIPlay : MonoBehaviour
{
    public Player.AIState PlayBuild(BuildData data, Player.AIState state);                      // RVA 0x565500
    private void BuildAndClean(BuildData data, Player.AIState state, List<BuildingType> buildingToPlace);  // RVA 0x560AD0
    private Nullable<ValueTuple<Building, ResourceAmount>> WhereToPlaceBuilding(BuildData data, BuildingType buildingType, HashSet<int2> wonderReserved);  // RVA 0x56AAF0
    private bool PlaceBuildingSomewhere(BuildData data, BuildingType buildingType, HashSet<int2> wonderReserved, List<BuildingType> buildingToPlace);  // RVA 0x565300
    private void UseMarket(BuildData data, Player.AIState state);                               // RVA 0x56A620
    private void CleanRemains(BuildData data, Player.AIState state, List<BuildingType> buildingToPlace);  // RVA 0x561D90
    private bool CleanNature(BuildData data);                                                   // RVA 0x5616A0
    private void PlaceBuildingAt(BuildData data, Building building, int wallsThickness, List<BuildingType> buildingToPlace);  // RVA 0x564C10
}
```
<!-- /code -->

**Patch:** postfix `WhereToPlaceBuilding` to choose other spots.

1. **Research**, if it has an active University: walk its technology list; skip ones not yet learnable; stop at the first learnable one it cannot afford.
2. Clean remains (rebuild-first), burn Sorcerer Towers above the limit (a House is queued for each), build and clean, use the Market, build and clean again.

**Rubber band:** a building score is the cost total of an island's buildings (bulk-placed pieces like Walls do not count). The AI only places a building if
`own score + building cost ≤ (best human score) × limitBuildingScore`. So Peasant and Passive AIs never build more than 80 % / 70 % of the best human's building value; King and Sorcerer are
practically unlimited (confirmed in game: a Peasant AI stopped building until the player built more). The score is checked once per round of the build loop, so it can overshoot slightly.

- **Defences:** Guard Towers in its list are skipped unless its defence score is below the strongest enemy attack score × `defensiveness`.
- **Choosing:** of the first 6 items in its build list that are unlocked and placeable, the first it can afford and that fits under the rubber band. If it is already winning, it only builds Guard Towers.
- WONDER AIs reserve a Wonder spot first.

**Placing** (`WhereToPlaceBuilding` `0x56AAF0`): spiral from the centre outwards, all 4 orientations from a random start; Farms only at distance ≥ 15; no extra disease risk up to turn 3.
Buildings that profit from neighbours (Land Camp, Cathedral, Windmill, Lumber Camp, Stone Quarry, Fishing Ship) go where they touch the most useful tiles (not placed at all if none);
everything else at the first free spot closest to the centre. `PlaceBuildingAt` clears blocking nature and remains, places the building, and puts a **ring of Walls** around it:
1 thick around Cathedral, Market, Storage, University and Windmill, 2 around the Wonder.

**Cleaning:** remains of destroyed buildings are cleaned and **that building is put at the front of the build list** (it rebuilds first). Remains on lava: at most `maxLavaToClean` per phase.
When nothing can be built it cleans one rock (if it has less stone than wood) or tree, not next to its own Quarry / Lumber Camp.

**Market:** MILITARY AIs buy military with all gold above 10, WONDER AIs buy hammers with all gold above 15. Nothing else.

Never used by the AI: repairing buildings.

## Attack

At the end of the combat phase (inside `PlayPhase`):

Target choice:

<!-- code: AIPlay | GetWarTarget, DefenseScore, SoldierFactor -->
```csharp
namespace DiceKingdoms.Game;

public class AIPlay : MonoBehaviour
{
    private Nullable<ValueTuple<Island, Building[]>> GetWarTarget(CombatData data, Player.AIState state, int targetMaxCount);  // RVA 0x562FA0
    private float SoldierFactor(Island x);                                                      // RVA 0x568FB0
    private float DefenseScore(Island x);                                                       // RVA 0x5623F0
}
```
<!-- /code -->

**Patch:** postfix `GetWarTarget` to choose other targets.

1. No attack for a Passive AI, or without soldiers or a free Dock. Ships = min(free Docks, soldiers / ship capacity); targets = max(1, ships / 2).
2. `GetWarTarget` (`0x562FA0`):
    - **Stop the winner:** if somebody not on its team is about to win, attack their Castle (or Wonder for a wonder win).
    - Otherwise: `attack = (0.7 × military + 0.3 × min(military, ship capacity × docks)) × SoldierFactor` (SoldierFactor = 18 with Warlocks, 9 with Templars, 8 with archers, else 4).
    - `DevelopmentScore` = 2 × building cost totals + 1.2 × stored wood, stone, gold, hammers and culture.
    - `DefenseScore` = military × its SoldierFactor + 32 × Guard Towers + 72 × Guard Towers Up + 0.2 × Walls + 0.4 × active population + 10 × Castles.
    - Candidates: enemies with `DefenseScore ≤ attack × aggressiveness` and development at least (1 − r²) × the best (r random). One is picked at random, weighted
      exp((development − defense) / 200).
    - Target buildings: after the berserk turn its Castle(s); before, the richest-loot buildings (not Farm, Wall, Castle, towers, Wonder, Sorcerer Tower).
3. It sends ranged units if unlocked (melee Templars if it has Templars but not Warlocks), full ships, spread over the targets; the drop point is the target building.
