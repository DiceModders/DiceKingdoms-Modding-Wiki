---
summary: The combat phase - planning attacks, ships and landing, defenders, unit caps, the deterministic tick simulation, parry and PRD, projectiles, towers, buildings, pillage and the end of a fight
---

# Combat

## Units

`Unit.Type`: ARCHER 0, BOURGEOIS 1, GENERAL 2, KING 3, PEASANT 4, PRIEST 5, SOLDIER 6, SORCERER 7, TEMPLAR 8, WARLOCK 9.
`Unit.Family`: INHABITANT 0, SOLDIER_MELEE 1, SOLDIER_RANGED 2.

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

- States: idle, moving, attacking. A hit lands at `enterTick + attackDelay`.
- **Targets** (`FindTarget` `0x631D10`): an enemy within 2 tiles always wins; else a defender within aggro distance; else (attackers) the nearest standing tile of the **target building**,
  or, once it is gone, a free exit tile on the ship. Defenders and inhabitants always go for the nearest enemy.
- An idle unit attacks anything enemy next to it (`AttackAnyOrIdle`): attackers also hit other enemy buildings they stand next to.
- **Priests** do not hit: they convert the unit on the target tile (only soldiers convert).
- **Melee on a building tile**: hits the building when no unit stands there, or with a **25 %** roll when one does (`BUILDING_PROTECT_UNIT_CHANCE_PERCENT`).
- Melee damage to buildings is **1**, or **2 on a Wall** when the attacker stands on its landing area.
- A hit on a Wall also damages every Wall directly next to it (one step, not a chain), so wall lines take damage in groups.

### Projectiles and towers

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

```
next = 0.9 × f(next) + 0.1 × base
f = Decrease(next) after a success,  Increase(next) after a failure
Decrease(x) = 1 − (1 − x) × exp(5 × x² × (1 − x)²)
Increase(x) = 1 − Decrease(1 − x) = x × exp(5 × x² × (1 − x)²)
```

Example: Decrease(0.5) = 0.317, Increase(0.5) = 0.683.

## End of a fight

- A fight normally ends when all attackers are dead or the target's Castle burns. Surviving attackers then walk back to their exit tiles and `Deployment.Leave(survivors)` sails them home.
- `CombatGrid.End()` (`0x631C40`) calls `Leave` with an empty list for deployments that did not leave yet (only visual).
- **Soldiers are only lost by dying** (−1 military each) or being converted; the fight ending loses nobody (confirmed in game).

## Power

`PowerConstants` gives each unit type and the Castle, Guard Tower, Guard Tower Up and Wall a power value (Warlock 3, Templar 2, Sorcerer 2, Guard Tower Up 16, Guard Tower 8, Castle 4, …).
`AttackPower(island)` / `DefensePower(island)` are shown in the player list and used by the [AI](ai.md).

## Developer helpers

`deflimit` limits the number of defenders; `MiscCommands.DebugAttack` shows attack debug info. See [Developer console](dev-console.md).
