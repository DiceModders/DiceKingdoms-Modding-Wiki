---
summary: All the numbers - phase times, prices, every building (tiles, tier, HP, cost, storage, pillage and remains rewards), technologies, power values, colours and official descriptions (game version 1.0.4)
---

# Game data

Values read from the game's assets (game version 1.0.4). Unit stats are on [Combat](combat.md), die faces on [Harvest](harvest.md), disasters on [Disasters](disasters.md),
AI values on [AI](ai.md).

## Match

| Phase | Time limit (timer FAST) |
|---|---|
| Start ("Settle") | 60 s |
| Harvest | 40 s (+5 s sync) |
| Disasters | none |
| Build | 70 s |
| Combat | 35 s |
| Win | none |

Timer SLOW = ×1.5, UNLIMITED = no limit; solo games are always UNLIMITED. Lobby defaults: timer FAST, multi-meteor off, teams off, wonder victory on.
Start resources: **3 food, 3 wood, 2 stone, 4 hammers, 8 culture**.

Build phase prices: tool cost (clean a tile, repair, burn) **1 hammer**; repair restores 60 HP. Market: **1 gold → 2 food, 1 wood, 1 stone, 1 military or 1 hammer** (×2 with Trade Efficiency).

## Buildings

| Building (id) | Tiles | Tier | HP | Cost | Storage | Pillage reward | Remains | Notes |
|---|---|---|---|---|---|---|---|---|
| Castle (2) | 16 | 1 | 200 | – | 15 food, 10 wood, 10 stone, 999 gold, 999 hammer, 999 culture, 8 military, 999 disaster | 3 gold | – | tower range 60 / every 30 ticks; die; defender King; garrison |
| Wall (18) | 1 | 1 | 50 | 1 stone, 1 hammer | – | 1 stone | – | placed in bulk of 10; fortification |
| Farm (7) | 8 | 1 | 15 | 1 wood, 1 hammer | – | – | 1 food | 1 food per harvest |
| Cottage (5) | 2 | 1 | 10 | 2 wood, 1 hammer | – | 2 food, 1 wood, 1 hammer | 1 wood | die; defender Peasant |
| Church (4) | 3 | 1 | 15 | 1 wood, 1 gold, 1 hammer | – | 1 gold, 2 culture | 1 wood | die; defender Priest |
| House (11) | 3 | 1 | 15 | 2 stone, 1 hammer | – | 1 stone, 1 gold, 2 hammer | 1 stone | die; defender Bourgeois |
| Barracks (1) | 6 | 1 | 60 | 2 wood, 6 stone, 2 gold, 1 hammer | 8 military | 2 stone, 4 gold | 2 wood, 4 stone | die; defender General; garrison |
| Dock (6) | 12 (11 water) | 1 | 50 | 5 wood, 2 stone, 1 hammer | – | 1 wood, 1 stone, 1 gold | 3 wood, 1 stone | one attack per Dock per turn |
| University (17) | 7 | 1 | 50 | 1 wood, 1 stone, 1 gold, 1 hammer | – | 3 gold, 2 culture | 1 wood, 1 stone | unlocks tier 2; research |
| Storage (16) | 6 | 2 | 20 | 4 wood, 2 stone, 1 hammer | 10 food, 20 wood, 20 stone | 4 food, 4 wood, 4 stone, 3 gold | 2 wood, 1 stone | |
| Guard Tower (8) | 4 | 2 | 40 | 6 wood, 2 stone, 1 gold, 1 hammer | – | 6 wood, 2 stone | 3 wood, 1 stone | tower range 30 / every 20 ticks |
| Guard Tower Up (9) | 4 | 2 | 40 | same | – | 6 wood, 2 stone | 3 wood, 1 stone | tower range 45 / every 13 ticks |
| Sorcerer Tower (14) | 3 | 2 | 20 | 1 stone, 1 gold, 1 hammer | – | 4 gold, 3 disaster | 3 disaster | die; defender Sorcerer |
| Hospital (10) | 5 | 2 | 40 | 2 wood, 2 stone, 1 hammer | – | 2 wood, 2 gold | 1 wood, 1 stone | protects 2 steps from Disease / Plague |
| Cathedral (3) | 6 | 2 | 40 | 4 wood, 4 stone, 6 gold, 1 hammer | – | 2 wood, 2 stone, 4 gold, 15 culture | 2 wood, 2 stone | 2 culture per Church + 1 per adjacent Church |
| Windmill (19) | 4 | 2 | 20 | 3 wood, 2 stone, 1 hammer | – | 5 food, 3 wood, 3 stone | 2 wood, 1 stone | 3 food per adjacent Farm |
| Archery Range (0) | 5 | 2 | 60 | 3 wood, 3 stone, 1 gold, 1 hammer | 8 military | 4 wood, 4 stone, 6 gold | 2 wood, 2 stone | allows ranged units (8 each); garrison |
| Fishing Ship (21) | 2 (2 water) | 2 | 30 | 3 wood, 1 hammer | – | – | 1 food, 1 wood | food per adjacent fish |
| Lighthouse (22) | 4 (4 water) | 3 | 50 | 4 wood, 2 stone, 1 hammer | – | 3 wood, 3 stone | 2 wood, 1 stone | tower range 15 / every 10 ticks; doubles Fishing Ships within 20 |
| Lumber Camp (12) | 4 | 3 | 20 | 6 wood, 2 stone, 1 hammer | – | 10 wood | 3 wood, 1 stone | wood per adjacent tree |
| Stone Quarry (15) | 3 | 3 | 20 | 3 wood, 2 stone, 1 hammer | – | 8 stone | 1 wood, 1 stone | stone per adjacent rock |
| Market (13) | 4 | 3 | 10 | 2 wood, 1 stone, 1 gold, 1 hammer | – | 3 food, 3 wood, 3 stone, 3 gold, 3 hammer | 2 wood | trading; needed to send resources |
| Land Camp (23) | 4 | 3 | 20 | 6 wood, 1 stone, 1 hammer | – | 10 wood | 2 wood, 1 stone | turns 24 water tiles per turn into land within 6 |
| Wonder (20) | 15 | 3 | 150 | 60 wood, 20 stone, 10 gold, 1 hammer | – | 10 gold, 100 culture | 10 wood, 10 stone | wonder victory |

Pillage reward = what an attacker gets when the building is destroyed in combat; remains = what cleaning the ruins gives.

## Technologies

All are paid with culture. Which ones a civilization can research, and in which order, is its tech tree ([Civilizations in the game code](civilizations.md#tech-trees)).

| Technology | Cost | Effect |
|---|---|---|
| Additional Roll | 8 | +1 re-roll |
| Double Clean Tile | 12 | cleaning gives double |
| Upgrade Guard Tower | 40 | Guard Tower → Guard Tower Up (+50 % range, +50 % attack speed) |
| Templars | 35 | Templars replace Soldiers |
| Warlocks | 180 | Warlocks replace Archers |
| Unlock Storage / Guard Tower / Market / Windmill / Archery Range | 5 each | unlocks the building |
| Unlock Lumber Camp / Stone Quarry / Lighthouse | 8 each | |
| Unlock Sorcerer Tower / Hospital / Fishing Ship | 3 each | |
| Unlock Cathedral / Land Camp | 10 each | |
| Unlock Wonder | 30 | |
| Rituals | 150 | 1 disaster point per Sorcerer Tower every turn to every opponent |
| Ship Hold Optimization | 80 | ships carry +100 % (16) |
| Trade Efficiency | 50 | market gives ×2 |

## Other data

- **Power values** (`PowerConstants`, used for the attack / defence power display and the AI): Warlock 3, Templar 2, Sorcerer 2, Guard Tower Up 16, Guard Tower 8, Castle 4, …
- **Player colours:** 12, in 4 groups of 3 ([Rendering](rendering.md#player-and-team-colours)).
- **Building tints:** hungry, sick, burnt, died in combat ([Rendering](rendering.md#buildings)).

## Official descriptions worth knowing

From the game's own texts:

- A Dock ship transports **8 soldiers**; Ship Hold Optimization: **+100 %**.
- Upgrade Guard Tower: **+50 % range, +50 % attack speed**.
- Hospital: protects its neighbours **and their neighbours** from Disease and Plague.
- Lighthouse: doubles the food of Fishing Ships in range (**does not stack**).
- Wonder: "Provides victory after one turn".
- Rituals: sends **1 disaster point per Sorcerer Tower every turn to every opponent**.
- Double Clean Tile: cleaning tiles gives twice the resources.
