---
summary: How the game looks - HDRP, the colored-paper palette shader, player and team colours, building states and tints, outlines, day and night, building orientation and the camera, visual effects, dice, animation timings
---

# Rendering, visuals and animation

!!! info "Code blocks"
    Declarations from the IL2CPP dump of game version 1.0.4, trimmed to what each section uses. `// 0x..` is the field offset, `// RVA 0x..` the method address in `GameAssembly.dll` (in Ghidra `0x180000000 + RVA`). Names stay stable between updates, offsets and RVAs do not. Private members are public in the BepInEx interop assemblies.

## Pipeline

- **HDRP**, two pipeline assets for the two quality levels (`HDRenderPipelineAsset-Low`, `-Normal`; Normal is the default).
- Volume profiles: `Common`, `Day`, `Night`, `Altitude`, `Low`, `Normal`.
- One perspective camera `Viewport` (FOV 45, near 0.1, far 1000) renders the world and the UI (see [Interface behaviour](interface.md#hud-layout)).
- Game shaders are HDRP Shader Graphs: `ColoredPaper` (almost everything), `Flag`, `Sail`, `Water`, `Windy` (vegetation), `Particle`, `SolidColor`, `Silhouette Object`.

## The palette ("colored paper")

Buildings, units, ships and props use the `ColoredPaper` shader (`_ColorPalette`, `_EmissionPalette`, `_ColorTint`, `_EmissionTint`, `_EmissionFactor`, `_Scale`).
**Meshes have no colour textures**: their UVs point into one **8×8 palette** (`ColorPalette.png`), and each face takes the colour of one cell (use point filtering when you look at exported models).
Lit windows use the emission palette (cells (5,0) orange and (5,1) yellow are the only emissive ones).

### Player and team colours

The colour helpers:

<!-- code: ColoredPaperUtils | *props, *methods -->
```csharp
namespace DiceKingdoms.Interface;

public class ColoredPaperUtils : MonoBehaviour
{
    public PlayerColor PlayerColor { set; }
    private ColoredPaperOverrides.Overrides Overrides { set; }

    protected void Awake();                                                            // RVA 0x5A0550
    public void Tint(ColoredPaperUtils.TintColors tint);                               // RVA 0x5A0D00
    public void CopyFrom(ColoredPaperUtils other);                                     // RVA 0x5A07F0
    public static void SlowSetPlayerColor(Transform object, PlayerColor playerColor);  // RVA 0x5A09D0
    public static void SlowTint(Transform object, ColoredPaperUtils.TintColors tint);  // RVA 0x5A0C20
}
```
<!-- /code -->

1. `PlayerColors.Init` (`0x56DD40`) makes one `PlayerColor` per colour. Its constructor copies the palette texture and repaints **cell (7, 7)** with the player's colour (`CustomColorPalette`).
2. Faces that show the owner's colour have their UVs on cell (7, 7). On the Castle that is the **flag** and the King's clothes; the castle body itself has none.
3. `ColoredPaperUtils.set_PlayerColor` puts the player's palette on all renderers of an object and sets its outline colour; `BuildingVisual.Apply` does this for every building, and units,
   ships and inhabitants use the same helper.

| Colour (palette cell) | Group |
|---|---|
| Red (1,6), Pink (3,2), Orange (1,5) | 0 |
| Green (1,2), Carribean (4,7), Cyan (2,3) | 1 |
| Yellow (1,3), Ivory (0,5), Brown (0,3) | 2 |
| Blue (2,4), Grey (4,5), Purple (3,3) | 3 |

`colorsPerTeam = 3`: 12 colours in **4 groups**. Colours can be picked freely in both modes. The group only matters when the lobby option **Teams** is on: then `Team.Same(a, b)` = same group.
In free-for-all `Team.Same` is true only for the same player. A new colour list must stay a multiple of 3.

`ColoredPaperOverrides` replaces a renderer's materials with cached clones keyed by (base material, palette, colour tint, emission tint).

## Buildings

Prefab parts: `Visual`, `Damaged Visual` (a copy of the mesh), `Fires` (3 fire particle systems), `Damage Particles`, `Remains` particles, `onlyEnableWhenActive` objects.

A building's visual object:

<!-- code: BuildingVisual | visual, damagedVisual, firesHolder, onlyEnableWhenActive, buildingTints, icons, Apply, ApplyLater, OnPlace, OnHit, Icon -->
```csharp
namespace DiceKingdoms.Game;

public class BuildingVisual : MonoBehaviour
{
    private BuildingTints buildingTints;                                                  // 0x18
    private Transform firesHolder;                                                        // 0x30
    private Transform visual;                                                             // 0x38
    private Transform damagedVisual;                                                      // 0x40
    private List<GameObject> onlyEnableWhenActive;                                        // 0x48
    public Sprite[] icons;                                                                // 0x58

    public void ApplyLater(Building building, IslandGrid grid, float delay);              // RVA 0x60D130
    public virtual void Apply(Building building, IslandGrid grid, bool preview = False);  // RVA 0x60D370
    public void OnHit();                                                                  // RVA 0x60DE30
    public void OnPlace(Building target);                                                 // RVA 0x60DF40
    public Sprite Icon(PlayerColor color);                                                // RVA 0x60DDC0
}
```
<!-- /code -->

**Patch:** postfix `BuildingVisual.Apply` runs whenever a building is built or refreshed (see [Custom models](../guides/modding/custom-models.md)).

| State | What changes | Tint (`BuildingTints`) |
|---|---|---|
| normal | – | none |
| hungry (starvation) | inhabitant replaced by a ragdoll | brown (0.5, 0.284, 0.074, 0.85) |
| sick / any other disaster | inhabitant replaced by a ragdoll | green (0.133, 0.4, 0.067, 0.85) |
| destroyed (health ≤ 0) | `Visual` off, `Damaged Visual` on and **sinks**; first time a particle burst (size × 30) | burnt, dark blue-grey (0.149, 0.169, 0.267, 1) |

- Fires follow the building's health.
- **Sinking:** the remains sink to the average over their tiles of `−0.3 − 0.2 × distance to the coast` (further inland sinks deeper), moved with `SmoothDamp` (0.2 s smooth time, about 90 % after 0.8 s).
- Units killed in combat become ragdolls with the `deathInCombatTint` (0.635, 0.257, 0.297, 0.6), pushed along the hit direction.
- `onlyEnableWhenActive`: the inhabitant of Barracks, Castle, Church, Cottage, House and Sorcerer Tower (and the chimney smoke on some of them) is shown only while the building is active,
  so a hungry or sick building loses its smoke. The other 90 building prefabs have nothing in this list.
- **Placing:** no grow animation, only a particle burst (`BuildingVisual.OnPlace`, footprint tiles × 5 particles). **Hit:** hit sound + 10 damage particles.
- `BuildingVisual.ApplyLater(building, grid, delay)` applies a state change at the right moment of a phase animation.

**Outline:** `Utility.Silhouette`, a custom HDRP pass with a stencil shader; colour = player colour, or the tile cursor colour.

## Day and night

`DayNightCycle` (`DiceKingdoms.Environment`), `ApplyTimeOfDay` (`0x57CC70`):

The day / night component:

<!-- code: DayNightCycle | *fields, TimeOfDay, Cloudiness, ApplyTimeOfDay, Update -->
```csharp
namespace DiceKingdoms.Environment;

public class DayNightCycle : MonoBehaviour
{
    private Light sun;                                          // 0x18
    private Volume dayVolume;                                   // 0x20
    private Volume nightVolume;                                 // 0x28
    private Sound seaAmbientSound;                              // 0x30
    private Sound natureNightAmbientSound;                      // 0x38
    private AnimationCurve sunIntensity;                        // 0x40
    private float timeOfDaySmoothing;                           // 0x48
    private float cloudinessSmoothing;                          // 0x4C
    private float sunYaw;                                       // 0x50
    private float sunMaxIntensity;                              // 0x54
    private AudioSources audioSources;                          // 0x58
    private Nullable<AudioSources.AllocatedSource> seaAudio;    // 0x60
    private Nullable<AudioSources.AllocatedSource> nightAudio;  // 0x80
    private float timeOfDay;                                    // 0xA0
    private float cloudiness;                                   // 0xA4
    private float targetTimeOfDay;                              // 0xA8
    private float targetCloudiness;                             // 0xAC

    public float TimeOfDay { get; set; }
    public float Cloudiness { get; set; }

    protected void Update();                                    // RVA 0x57D150
    private void ApplyTimeOfDay(float timeOfDay);               // RVA 0x57CC70
}
```
<!-- /code -->

- Sun rotation = (timeOfDay × 360°, sunYaw, 0); `k = sunIntensity(timeOfDay) × (1 − cloudiness)`; sun intensity = k × max. **Day volume weight = k, Night volume weight = 1 − k.**
- `sunIntensity`: 0 → 0, 0.05 → 0.8, 0.25 → 1, 0.45 → 0.8, 0.5 → 0, 1 → 0. Day is 0 – 0.5, night 0.5 – 1.
- `timeOfDay` moves **forward only** toward `targetTimeOfDay` (wrapping past 1), smoothing 1 per second; cloudiness smoothing 5.
- **Every phase sets the time of day when it starts:** `TurnPhase.Enter` writes the phase's `timeOfDay` field into the target. All phases have **0.125** (morning, sun 45° up), so the game
  always shows morning, and the night half (Night profile, night ambience) is built but unused.
- Lightning and Thunderstorm darken the sky: target cloudiness 1, wait until it is above 0.9, strike, then back to 0.

!!! tip "Night or per-phase light"
    `TimeOfDay = 0.75f` on `Object.FindObjectOfType<DayNightCycle>()` gives midnight, but only until the next phase starts (confirmed in game). Re-apply it in a postfix on `TurnPhase.Enter`,
    or change each phase's `timeOfDay` field once at match start; that also allows a different light per phase. It is a local visual, safe in multiplayer. The night profile looks quite
    washed out on the colourful models.

## Building orientation and the camera

`GridTransform.Apply` (`0x60F050`): `GridTransform` = tile `position`, `orientation` 0..3, `mirroring`.

Orientation is applied by `GridTransform`:

<!-- code: GridTransform | *fields, Apply -->
```csharp
namespace DiceKingdoms.Game;

public struct GridTransform
{
    public int2 position;                                      // 0x0
    public GridTransform.Orientation orientation;              // 0x8
    public GridTransform.Mirroring mirroring;                  // 0xC
    public static readonly GridTransform IDENTITY;             // static

    public void Apply(Transform transform, Transform parent);  // RVA 0x60F050
}
```
<!-- /code -->

- Orientations: **0 EAST, 1 NORTH, 2 WEST, 3 SOUTH**. Each step turns the model **90° counter-clockwise seen from above**. Mirroring flips the local X scale.
- The model's pivot is the centre of the building's tile (0, 0); grid y = Unity Z. Islands are never rotated.
- **The camera yaw is fixed at 45°**: it always looks toward Unity +X +Z, so you only ever see the **south and west sides** of a building. Only height and pitch change.
- There is **no door or entrance concept** in the code; units leave from ships, never from buildings.

Castle gates of the original models (side at orientation 0): Europe and Middle East **east** (seen at orientations 2 and 3), Viking and Ancient **west** (seen at 0 and 1), Maya on all sides.
A Castle placed automatically gets orientation 0 (EAST is tried first), so Europe and Middle East show their back unless the player turns it with R. AI castles use a random start orientation.

The big flag (`flag_big`) has the same rotation in every castle prefab and turns with the building.

## Visual effects

All effects are **pooled**, free themselves and are **client-side only**; the game logic is driven by the disaster coroutines and combat ticks, the visuals follow.

| Visual | Used by | Behaviour | Values |
|---|---|---|---|
| `FireVisual` | Fire, Tornado | particle shape fitted to the building bounds, burst = tiles × 5, sound fades in / out | duration 3, sound fade 0.5 |
| `LightningVisual` | Lightning, Thunderstorm | random zig-zag bolt in a cone (bones tilted up to ±90°, retried until it fits) | duration 0.2, max distance 3 |
| `TornadoVisual` | Tornado | grows and shrinks, spins 200 °/s, funnel 1000 °/s, tilted −30° | duration 2.5 |
| `MeteoriteVisual` | Meteorite | flies a straight line from 90 m away at 45° elevation with a **random yaw**, 30 m/s, for 3 s; the crater is applied 3.2 s after the start | |
| `TsunamiVisual` | Tsunami | animated wave (clip 3.683 s), per-tile splashes, freed after 4 s | |

**Projectiles** (`ProjectileVisual`): Arrow / Archer arrow (gravity 0.6, friction 0.01, length 0.43), Fireball (gravity 0.2, friction 0.001). The motion is ballistic with linear drag, solved
exactly so the projectile lands on the combat tick its hit is resolved: `p(t) = start + (V × (1 − e^(−k t)) + g t) / k`. At most 256 at once.

**Harvest icons** (`HarvestedResourceVisual`): grow in 0.15 s, fly freely for 0.5 s (start 1200 px/s at ±45°), then a "magnet" pulls them at 2000 px/s to their counter (collected within 40 px).
The harvest phase waits at least 1.5 s for them.

## Dice

`Dice.Roll(destination, faceUp)` (`0x5C16A0`) pre-simulates a whole physical-looking throw (gravity −10, bounce 0.8 with 0.4 randomness, start velocity (−0.6, 0, 0) ± 0.1, angular 30, from
height 0.1), then **rotates it so the face that ends on top is `faceUp`** (decided by the game) and shifts it to `destination`. The die first glides 1 s to the start pose. Each bounce plays a
sound with volume = bounce strength (if ≥ 0.02). The throw only shows a result that is already known.

## Animation timings

Unit clips (seconds; every unit uses one controller with per-unit overrides):

| Unit | Idle | Walk | Run | Attack |
|---|---|---|---|---|
| Peasant, Bourgeois, Soldier, Templar, General, King | 1.3 | 1.3 | 0.633 | 0.433 |
| Archer | 1.3 | 1.3 | 0.633 | 1.333 |
| Priest | 1.3 | 1.3 | 0.633 | 1.3 |
| Sorcerer, Warlock | 1.3 | 1.3 | 0.633 | 1.3 (Europe 0.8) |

Maya clips are 1.25 × longer. Transitions: into attack 0.1 s, exit at 87 % of the attack; others 0.25 s. Inhabitants walk at 0.288 and run at 1.06 units/s.

| Thing | Value |
|---|---|
| Ship plank out / in | 0.8 s (Europe, Middle East), 0.667 s (others) |
| Soldiers leave the ship after landing (`DEBARK_DELAY`) | 5 ticks (0.5 s) |
| Windmill turn | 2.33 – 3.97 s per civilization |
| Combat tick | 0.1 s, max 3000 ticks |
| Phase banner | 2 s (+0.5 s to switch off) |
| Tooltip delay | 0.4 s |
| Log row | 30 s visible, 1 s fade |
| Victory warning | 6 s, 1 s fade |
| Music crossfade | 2 s |
| Camera pitch by height | 0 → 0°, 4 → −30°, 25 → −55°, 100 → −90° |

## Tips

- Recolour one object: `ColoredPaperUtils.SlowTint(transform, tint)` / `SlowSetPlayerColor(transform, color)`.
- New models: put player-coloured faces on palette cell (7, 7) and use the ColoredPaper material so tints and player colours work. See [Custom models](../guides/modding/custom-models.md).
- Effects are visual only: changing prefab values (durations, particles) is safe in multiplayer. For the meteorite keep the 3.2 s wait longer than the 3 s fall.
