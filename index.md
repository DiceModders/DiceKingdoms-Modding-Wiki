# DiceModders

This site is intended for the documentation of DiceKingdoms in order to make modding the game easier.

## Where to start

- **Playing with mods:** [Mods Installation](guides/mod-installation.md).
- **Writing a mod:** [Project Setup](guides/modding/project-setup.md), [IL2CPP tips](guides/modding/il2cpp-tips.md), then the tested [Hook recipes](guides/modding/hook-recipes.md).
- **How the game works:** start with the [Overview](game-architecture/overview.md) and the [Turn loop](game-architecture/turn-phases.md), then one page per phase:
  [Harvest](game-architecture/harvest.md), [Disasters](game-architecture/disasters.md), [Build phase](game-architecture/build-phase.md), [Combat](game-architecture/combat.md),
  [Win conditions](game-architecture/win-conditions.md). The [AI](game-architecture/ai.md) and all numbers in [Game data](game-architecture/game-data.md).
- **Islands:** [Grid](game-architecture/island/grid.md), [Generation](game-architecture/island/generation.md), [Save format](game-architecture/island/save-format.md) (island codes).
- **Multiplayer:** [Networking](game-architecture/networking.md) (what runs on the host, what every player needs) and [Lobby, menus and Steam](game-architecture/lobby-and-steam.md).
- **Interface and visuals:** [Game UI](game-architecture/ui.md), [Interface behaviour](game-architecture/interface.md), [Rendering](game-architecture/rendering.md),
  [Audio, settings and localization](game-architecture/audio-settings-localization.md), [Developer console](game-architecture/dev-console.md).
- **New looks and civilizations:** [Custom models](guides/modding/custom-models.md), [Making a civilization](guides/modding/making-a-civilization.md),
  [Civilizations in the game code](game-architecture/civilizations.md), [Civilization models and sprites](game-architecture/civilization-assets.md).
- **Addresses and offsets:** [Reverse-engineering notes](game-architecture/reverse-engineering.md) and the [Internals reference](reference/internals.md) (game version 1.0.4).
- **Sharing your mod:** [Publishing a mod](guides/modding/publishing.md).
