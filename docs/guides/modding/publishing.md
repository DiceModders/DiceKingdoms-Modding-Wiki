---
summary: Releasing a BepInEx mod - GitHub releases and Thunderstore packages
---

# Publishing a mod

## GitHub release

A mod is one DLL plus optional files. Do not commit the build output (`bin/`, `obj/`) or the `interop/` folder: the interop assemblies are generated
from your own copy of the game and contain the game's code, so they cannot be shared. For the same reason a CI service (GitHub Actions) cannot build a mod that
references them; build on your machine.

1. `dotnet build -c Release --no-incremental` (a plain `dotnet build` may skip compiling when it thinks the DLL is up to date; `--no-incremental` always compiles).
2. Zip `YourMod.dll` (and any files it needs) as `YourMod-1.0.0.zip`.
3. On GitHub create a release with the tag `v1.0.0` and attach the zip. Players copy the DLL to `BepInEx/plugins`.

## Thunderstore

A Thunderstore package is a zip with these files at its root:

| File | Content |
|---|---|
| `manifest.json` | `name`, `version_number`, `website_url`, `description`, `dependencies` |
| `README.md` | The page shown on Thunderstore |
| `icon.png` | 256x256 picture |
| `YourMod.dll` | The mod. Put it in a `plugins` folder if you want the archive to merge into the game folder by hand. |

List other mods in `dependencies` (for example a library mod yours needs) in the form `Author-Name-1.0.0`, instead of copying their DLLs. Mod managers install dependencies
automatically. In code add `[BepInDependency("plugin.guid")]` so that the dependency loads first.

!!! tip
    Test the package by installing it with a mod manager into a clean game folder before you upload it.
