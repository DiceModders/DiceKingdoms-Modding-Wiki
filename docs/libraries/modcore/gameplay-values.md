---
summary: ModCore gameplay values - unit stats per civilization or for all, a mod civilization's own defaults, building and technology values, mod priority and the multiplayer safety rules
---

# Gameplay values

Unit, building and technology values change the simulation, so every machine must use the same ones. ModCore applies a mod's values only when that is safe, and the
game's own values otherwise:

- **Local game** (no Steam lobby): when the mod's lobby switch is on.
- **Steam lobby**: when the host's switch is on, every member has the mod in the **same version**, and the mod is switched on.
- The decision is **frozen at match start**. Changes made during a match wait until it ends.

Each `Register` call creates that switch as a lobby rule (on by default), so the host can turn your changes off for a match.

## Unit stats

Change attack timing, aggro distance, range, parry, arrow block and miss chance for one civilization or for all. Fields you leave `null` keep the value below them.

```csharp
var stats = Core.Units.Register("author.balance", "Faster archers", priority: 10);

stats.Set(Unit.Type.ARCHER, new UnitStatChange { AttackPeriod = 7 });                 // every civilization
stats.Set(civ.Id, Unit.Type.ARCHER, new UnitStatChange { Range = 8, AggroDistance = 9 });   // one civilization
stats.Set(Unit.Type.SOLDIER, new UnitStatChange { Parry = new Chance(1, 3) });       // 1 in 3
```

| Field | Meaning |
|---|---|
| `AttackPeriod`, `AttackDelay` | Ticks between attacks (at least 1); delay of the attack (at least 0) |
| `AggroDistance` | How far a unit notices enemies |
| `Range` | Ranged units only, in half tiles: the Rules screen shows `Range / 2` |
| `Parry`, `BlockArrow` | `Chance(numerator, denominator)` |
| `MissChance` | Ranged units only, `Chance(numerator, denominator)` |

A mod civilization can have its **own default values**: they replace the game's for that civilization only, without a lobby switch of their own (they belong to the civilization,
which already needs every player to have the mod). A civilization without own defaults uses the game's values.

```csharp
Core.Units.SetCivilizationDefaults(civ, Unit.Type.SOLDIER, new UnitStatChange { Parry = new Chance(3, 4) });
```

Order, lowest first: the game's values → the civilization's own defaults → all-civilization changes → per-civilization changes. Between mods the higher `priority` wins; equal
priorities go by key, so every machine resolves them the same way.

A lobby screen can list everything:

```csharp
foreach (var m in Core.Units.Mods)
    Log.LogInfo($"{m.ModKey} priority={m.ModPriority} active={m.IsActive} changes={m.AllChanges.Count}");

UnitStatValues game = Core.Units.Default(Unit.Type.ARCHER);        // the game's values
UnitStatValues now  = Core.Units.Effective(civ.Id, Unit.Type.ARCHER);   // what this civilization plays with right now
```

## Buildings and technologies

Buildings and technologies are shared by every player (one object per type), so these changes apply to all civilizations. Values: building cost, storage capacity, pillage and
remains rewards (all as resources), health, tower range and period; technology cost.

```csharp
var balance = Core.Balance.Register("author.economy", "Cheaper farms", priority: 0);

balance.Set(BuildingTypeName.FARM, new BuildingChange {
    Cost = new ResourceChange { Wood = 2, Stone = 0 },   // unset resources keep the game's amount
    Health = 4,
});
balance.Set(BuildingTypeName.GUARD_TOWER, new BuildingChange { TowerRange = 6, TowerPeriod = 3 });
balance.SetCost(TechnologyName.UNLOCK_MARKET, new ResourceChange { Gold = 3 });
```

```csharp
BuildingValues farmNow  = Core.Balance.Building(BuildingTypeName.FARM);          // with the active mods
BuildingValues farmGame = Core.Balance.DefaultBuilding(BuildingTypeName.FARM);   // the game's own
ResourceValues market   = Core.Balance.TechnologyCost(TechnologyName.UNLOCK_MARKET);
```

ModCore restores the game's values and applies the active mods again whenever the set of active mods changes (switch, lobby members, match start and end), so turning a mod
off always gives the game's values back.

## Removing changes

```csharp
stats.Clear(civ.Id, Unit.Type.ARCHER);   // one entry
stats.ClearAll();
balance.Clear(BuildingTypeName.FARM);
balance.ClearAll();
```
