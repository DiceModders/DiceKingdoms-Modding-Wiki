---
summary: Names of the models (prefabs) and sprites a civilization is made of - buildings, units, ship, nature, cliffs, icons - and the naming patterns to follow for a new one
---

# Civilization assets: models and sprites

What a civilization looks like is a set of prefabs (3D models) and sprites (icons). The names below were listed from the game's loaded assets (`Resources.FindObjectsOfTypeAll`, game version 1.0.4), so they are the real names. How the game picks them is in [Civilizations in the game code](civilizations.md); how a mod supplies its own is in [Making a civilization](../guides/modding/making-a-civilization.md).

## Models (prefabs)

Every model name is `<thing> <civilization>`, with the civilization written `Europe`, `Middle East`, `Viking`, `Ancient` or `Maya`. A mod civilization replaces any of them with its own prefab; whatever it leaves out keeps the European one.

### Buildings (`BuildingVisual`)

One prefab per building type and civilization, 24 types x 5 civilizations. The European one is the default (`BuildingType.visualPrefab`):

`Archery Range`, `Barracks`, `Castle`, `Cathedral`, `Church`, `Cottage`, `Dock`, `Farm`, `Fishing Ship`, `Guard Tower`, `Guard Tower Up`, `Hospital`, `House`, `Land Camp`, `Lighthouse`, `Lumber Camp`, `Market`, `Sorcerer Tower`, `Stone Quarry`, `Storage`, `University`, `Wall`, `Windmill`, `Wonder`.

Example: `Castle Maya`, `Guard Tower Up Viking`. The building type enum (`BuildingTypeName`) has the same 24 names in upper case with underscores, in this order: `ARCHERY_RANGE`, `BARRACKS`, `CASTLE`, `CATHEDRAL`, `CHURCH`, `COTTAGE`, `DOCK`, `FARM`, `GUARD_TOWER`, `GUARD_TOWER_UP`, `HOSPITAL`, `HOUSE`, `LUMBER_CAMP`, `MARKET`, `SORCERER_TOWER`, `STONE_QUARRY`, `STORAGE`, `UNIVERSITY`, `WALL`, `WINDMILL`, `WONDER`, `FISHING_SHIP`, `LIGHTHOUSE`, `LAND_CAMP`.

A few buildings have several scene instances named `Cottage Maya (1)`, `House Ancient (2)` and so on. Those are variants placed in the scene, not separate prefabs.

### Units (`UnitVisual`)

Ten unit types, each in every civilization: `Archer`, `Bourgeois`, `General`, `King`, `Peasant`, `Priest`, `Soldier`, `Sorcerer`, `Templar`, `Warlock` (example: `Soldier Europe`). The skin registry also holds `Inhabitant King <civ>` and `Inhabitant Sorcerer <civ>` models.

### Ship (`ShipVisual`)

`Ship Europe`, `Ship Middle East`, `Ship Viking`, `Ship Ancient`, `Ship Maya`.

### Nature (`NatureVisualHub`)

Per biome (`Plains`, `Desert`, `Snow`, `Mediterranean`, `Jungle`): `Tree <biome>`, `Rock <biome>`, `Flower <biome>`. Beyond those: `Fish Plains`, `Lava` and `Scorched Earth` (these have no biome of their own). `Rock Mediterranean` was not found in memory.

### Inner cliffs (`InnerCliffVisualHub`)

Five shapes per biome: `Inner Cliff Main 3x3`, `Inner Cliff Main 2x2`, `Inner Cliff Extension 2x2`, `Inner Cliff Extension 1x2`, `Inner Cliff Extension 1x1`, followed by the biome (`Plains`, `Desert`, `Snow`, `Jungle`). No Mediterranean cliff hubs were found: the Mediterranean biome seems to reuse another biome's.

### Skin variants (`SkinVariant`)

Every prefab above that the skin registry swaps carries a `SkinVariant` component with the biome, civilization and variant name it stands for. That component and the registry's `entries` are what `SkinRegistry.GetSkinVariant` reads. A mod does not need to touch either: `ModCivilization.Skins` swaps the result of the lookup.

## Sprites

The game has about 1,900 sprites. Most are icons that exist once per player colour.

### Player colours

Twelve colours, used as the last word of an icon name: `blue`, `brown`, `carribean` (sic), `cyan`, `green`, `grey`, `ivory`, `orange`, `pink`, `purple`, `red`, `yellow`.

### Building icons

`building-icon-<building>-<civilization>-<colour>`, with the words of the name joined by `-`, for example `building-icon-archery-range-ancient-blue`. `BuildingVisual.Icon(PlayerColor)` returns the sprite at the colour index, so a building visual needs **12 sprites in colour order**. The dump lists 120 building/civilization pairs, each with a `-red` sprite; only some pairs (walls and guard towers in every civilization, and most Viking and Ancient buildings) also have the other eleven colours as separate sprites, so check the dump for the pair you copy.

The icons are packed in one atlas per civilization (`Buildings Europe Atlas`, `... Middle East ...`, `... Viking ...`, `... Maya ...`, `... Ancient ...`), 256x256 each.

### Unit icons

`unit-icon-<unit>-<civilization>-<colour>`, for example `unit-icon-archer-ancient-blue`, 256x256, one atlas per civilization (`Units <civ> Atlas`). About 325 distinct unit sprites; every unit has a `-red` one and many have the other colours too. `UnitVisual.Icon(PlayerColor)` works the same way as for buildings.

### Everything else

| Group | Names |
|---|---|
| Resources | `food`, `wood`, `stone`, `gold`, `hammer`, `culture`, `military`, `research`, `population`, `trade-efficiency` |
| Disasters | `plague`, `drought`, `fire`, `lightning`, `meteorite`, `tornado`, `tsunami`, `thunderstorm`, `starvation`, `revolt`, `robbers`, `pirates`, `disease`, `conflagration`, `Disaster-Gauge-*`, `Disaster-Icon-Background(-Active)` |
| Pennants | `pennant-<colour>` (12) |
| Dice | `dice-icon-<building>-red` (barracks, castle, church, cottage, house, sorcerer-tower) |
| Technology slots | `Technology-Slot-Background-` and `Technology-Slot-Foreground-` + `Hovered`, `Learnable`, `Learnt`, `Locked` |
| Buttons and frames | `Button-1-*`, `Button-2-*` (`Disabled`, `Hovered`, `Pressed`, `Ready`), `Ornament-1..5`, `Rounded_Rect_10px`, `Scroll-Handle`, `Knob`, `Checkmark`, `Background-3`, `Background-4`, `Event-Banner` |
| Rules screens | `rules-building-1..4`, `rules-units-1..2`, `rules-upgrades-1`, `rules-disasters-1`, `rules-game-phases-1..4`, `rules-how-to-play-1..3`, `rules-title-banner` |
| Menu | `settings`, `host`, `exit`, `ai`, `steam`, `discord`, `about`, `key`, `logo-title`, `back-to-menu`, `scoreboard`, `scoreboard-button` |

The complete list can be printed from a mod with `Resources.FindObjectsOfTypeAll<Sprite>()` (name, `texture.name`, size); sprites in an atlas appear twice, once as `repacked` and once with the atlas texture.

## Naming a new civilization's assets

Follow the same pattern so the assets sort and read like the game's: `Castle Atlantis` for a prefab, `building-icon-castle-atlantis-red` for a sprite. The game never reads the names of a mod civilization's assets (the lookups are by type), so the names are only for you and for tools like the dump.

## What a complete skin needs

| Part | Count | Notes |
|---|---|---|
| Building prefabs | up to 24 | Anything missing falls back to Europe. |
| Building icons | 12 per building | One per player colour, in colour order. |
| Unit prefabs | up to 10 | Same fallback. |
| Unit icons | 12 per unit | Same. |
| Ship prefab | 1 | |
| Nature hubs | 3 per biome | Which biome a mod civilization uses is set by the civilization it looks like (`skinsLike`). |
