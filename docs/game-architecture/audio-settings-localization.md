---
summary: Audio (sound pool, music selection, mixer and volume curve), saved settings and Steam Cloud, the tutorial, and the localization system with its language quirks
---

# Audio, settings, localization and tutorial

## Audio

| Piece | What it is |
|---|---|
| `Sound` (ScriptableObject, 48 assets) | `clips[]`, `mixerGroup`, `minVolume` / `maxVolume`, `minPitch` / `maxPitch`, `loop` |
| `AudioSources` (static `instance`) | a pool of **2048** AudioSources |
| `AudioSources.AllocatedSource` | the handle `Play` returns: source + **ticket**; an old handle can never stop a sound that reused the source |
| `Main` mixer | Master › Main › Effects, Music, Ambient, Interface; exposed `Main_Volume`, `Effects_Volume`, `Music_Volume`, `Ambient_Volume`, `Interface_Volume` |

`AudioSources.Play(sound, position?)` (`0x580C20`) takes the first free source (none free → error, no sound), 3D at the position or 2D without one, random pitch and volume in the sound's ranges,
**a random clip**, and plays it.

### Music

`Music` (match scene): calm track per civilization (Europe 6 clips, Middle East 3, Viking 9, Ancient 9, Maya 6), combat, decisive combat, winning and losing tracks. `Music.get_Track`
(`0x590F70`), every frame:

1. Win phase with a result → winning if the winner is on your team, else losing.
2. Combat phase, 5 s into the fight → **decisive** if somebody could win this turn (`WinPhase.CheckWin()` at phase start), else combat.
3. Otherwise the calm track of **your** civilization. When a song ends, a new random clip of the same track starts (a shuffled playlist).

Changes crossfade over **2 s**. The main menu has its own `menuMusic`.

### Volume

Five sliders (0–100) in the settings. Each writes its mixer parameter as `dB = clamp(20 × log10(value / 100) + offset, −80, 0)`:

| Slider | Offset | Default |
|---|---|---|
| Main | 0 | 70 |
| Effects | +6 | 100 |
| Music | −8 | 50 |
| Ambient | 0 | 100 |
| Interface | +5 | 100 |

Ambience: the sea, and a night-nature loop that only plays at night (see [Rendering](rendering.md#day-and-night)).

For mods: `AudioSources.instance.Play(sound, null)` plays any `Sound`; keep the handle to stop it. Music can be replaced by changing the `clips` of the Music-* assets or by a postfix on `Music.get_Track`.

## Settings and saved data

The game saves only **four settings files** (`video`, `audio`, `input`, `game`, JSON) and the flag file **`GameSettings/.did-tutorial`**. There are no save games, no PlayerPrefs and no Steam
stats, achievements or leaderboards.

`SettingSets` (`localFolder = "GameSettings"`, path relative to the game folder):

| Set | Stored |
|---|---|
| video | always the local file `GameSettings/video.json` |
| audio, input, game | **Steam Cloud** when Steam runs, otherwise the local file |

- **Save** when the settings window closes (`SettingSetsInterface.OnDisable`): each set is exported and stored.
- **Load** at start: a cloud set with Steam running is read from Steam Cloud only (missing → defaults, no fallback to the local copy); otherwise the local file or defaults.
- Defaults: video quality Normal, full screen mode 2, 1920×1080, frame limit 144, VSync on; audio as above; input bindings (rotate `/Keyboard/r`, mirror `/Keyboard/t`, keep placing
  `/Keyboard/shift`, scoreboard `/Keyboard/tab`, …); game language 0.

For a mod's own settings a BepInEx config file is easiest. To use Steam Cloud like the game: `SettingSets.Store(true, "GameSettings/mymod.json", bytes)` / `SettingSets.Get(true, ...)`.
How the settings window is built: [Interface behaviour](interface.md#settings-widgets).

## Tutorial ("Training")

Main menu → Training → `MainMenu.LoadTraining`: writes `.did-tutorial` to Steam Cloud **as soon as it starts** (the "recommend tutorial" hint is hidden from then on), adds **two Passive
AIs** (the only use of that difficulty) and marks the match as training.

`Training` (in the match scene) has `TrainingStep` children with `TrainingObjective`s; all the data is in the scene:

| # | Step | Objectives |
|---|---|---|
| 1 | Initial buildings | Castle, 3 Cottages, Farm, Dock |
| 2 | Get more dice | 2 Cottages, 2 Houses |
| 3 | Stable food supply | 2 Farms |
| 4 | Culture | a Church; 12 culture |
| 5 | See opponents' progress | open the scoreboard |
| 6 | Military | Barracks; 8 military |
| 7 | Attack | a Dock; press Ready in combat; pillage once |
| 8 | Research | University; research Storage; build a Storage |
| 9 | Win | military win |

At the start every building button and Ready are hidden and **combat phases are skipped**; each step shows the buttons it needs, and the Pillage step turns combat on. One-time hints
(`ConfirmModal.OpenInformation`) explain the harvest, disasters, attack, pillage, starvation and the University.

## Localization

Unity **Localization** with SmartFormat, string tables in Addressables bundles, 421 keys × 6 languages. Game code goes through `StructuredTextStaticReferences.instance`
(`LocalizeString(key)`).

**Language at start** (`Localization.asset` startup selectors):

1. `SteamLocaleSelector.GetStartupLocale` (`0x550170`), only with Steam: english → en; schinese, tchinese → zh; french → fr; russian → ru; **ukrainian → ru**; anything else, **German included → en**.
2. Fallback `en`.

So a German Steam client starts in English, and a Ukrainian one in Russian although a Ukrainian table exists.

**Language setting** (`GameSettings.language`, Steam Cloud): 0 Steam default (selectors run again), 1 English, 2 French, 3 Chinese (simplified), 4 Russian, 5 German. There is **no Ukrainian**
entry; only a mod can select it (`LocalizationSettings.SelectedLocale`).

Texts are `StructuredText` with `ITextElement.Localized(key, args)`: the translation can contain the same icon tokens and `{a.N}` argument slots (see
[Interface behaviour](interface.md#text-structuredtext)). Rendered text is cached per locale and labels redraw when the language changes.

For mods: add entries to the string table at runtime (`LocalizationSettings.StringDatabase.GetTable(...)` → `AddEntry`) for the language in use (and again when it changes), or write your own
`StructuredText`; icon tokens work without localization. The startup quirks can be fixed with a postfix on `SteamLocaleSelector.GetStartupLocale`.
