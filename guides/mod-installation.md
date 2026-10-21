# Mods Installation

## Installing BepInEx

BepInEx is a mod loader. It allows the game to load mods. You only need to install it once.

!!! info "BepInEx documentation:"
    [Installing BepInEx on Il2Cpp Unity](https://docs.bepinex.dev/master/articles/user_guide/installation/unity_il2cpp.html?tabs=tabid-linux)

**Installation steps:**

1. **Download BepInEx 6.x.**
   Go to the [BepInEx releases page](https://github.com/BepInEx/BepInEx/releases). Find the latest release and download the file for **Unity IL2CPP**. It has name in this format `BepInEx-Unity.IL2CPP-OS-x64-6.X.X.zip`.

2. **Find the game folder.**
   This is the folder where `DiceKingdoms.exe` is located.  
   - Open Steam.
   - Right-click on **Dice Kingdoms** in your library.
   - Select **Manage** → **Browse local files**.
   - A folder will open. This is the game folder.

3. **Extract BepInEx into the game folder.**
   - Right-click the downloaded BepInEx archive and choose **Extract All**.
   - Copy everything from inside the archive into the game folder.
   - Make sure the `BepInEx` folder and files like `doorstop_config.ini` and `winhttp.dll` appear next to `DiceKingdoms.exe`.

4. **Run the game once.**
   - Start the game normally.
   - The first launch may take longer than usual. This is normal. BepInEx is setting itself up.
   - You can close the game after it reaches the main menu.

5. **Check that BepInEx is installed.**
   - Open the game folder.
   - You should see a `BepInEx` folder. Inside it, there should be:
     - `LogOutput.txt`
     - a `config` folder with `BepInEx.cfg` inside.
   - If you see these, BepInEx is installed correctly.

## Installing Mods

After BepInEx is installed, installing mods is usually very simple.

1. **Download the mod.**
   Mods are usually shared as `.zip` or `.rar` files.

2. **Extract the mod archive.**
   - Right-click the archive and choose **Extract All**.
   - Extract it to a temporary folder, like your Desktop.

3. **Look inside the extracted folder.**
   - The main mod file has a `.dll` extension. This is the mod itself.
   - A mod can also be a set of files: for example, one `.dll` plus some folders or other files. All of them belong together.
   - Some mods include a `README.txt` or similar. **Read it first** — it may have special instructions.
   - Sometimes the archive already contains a `plugins` folder. If you see that, the mod is packaged so that you can simply extract the archive contents directly into the game's root folder (the folder with `DiceKingdoms.exe`). The `plugins` folder will merge with `BepInEx/plugins`, and the mod will be placed correctly automatically.

4. **If there is no `plugins` folder in the archive, install manually:**
   - Go to your game folder.
   - Open `BepInEx`.
   - If there is no `plugins` folder, create one and name it exactly `plugins`.
   - Copy **all** mod files (the `.dll` and any accompanying files or folders) into `BepInEx/plugins`.
   - If the mod has its own folder structure, keep it. For example, if the mod has a folder `MyMod` containing `MyMod.dll` and an `assets` folder, copy the whole `MyMod` folder into `BepInEx/plugins`.
   - Example: `C:\Games\DiceKingdoms\BepInEx\plugins\MyMod.dll`

5. **Run the game.**
   - The mod should load automatically.
   - If the mod has settings, a config file may appear in `BepInEx/config` after you run the game. You can open it with Notepad and change values.
