---
summary: How to setup C# project for modding DiceKingdoms with BepInEx
---

## Installing BepInEx

DiceKingdoms uses the IL2CPP backend. This means you need BepInEx 6.x for modding.

!!! info "BepInEx documentation:" 
    [Installing BepInEx on Il2Cpp Unity](https://docs.bepinex.dev/master/articles/user_guide/installation/unity_il2cpp.html?tabs=tabid-linux)

**Installation steps:**

1. Download BepInEx 6.x for IL2CPP. You can find the latest builds on the [github release page](https://github.com/BepInEx/BepInEx/releases#release-v6.0.0-pre.2).

2. Extract the archive into the game's root folder. This is the folder containing DiceKingdoms.exe. Simply copy the entire contents of the BepInEx archive there.

3. Run the game once. This is required to generate configuration files and perform initial setup. The first launch may take longer than usual while BepInEx analyzes the game.

4. Verify the installation. After launching, you should see LogOutput.txt and a config folder with BepInEx.cfg inside the BepInEx folder. If there are no critical errors in the log and the plugins folder has been created, the installation was successful.

## Creating a project

!!! info "BepInEx documentation:" 
    [Creating a new plugin project](https://docs.bepinex.dev/articles/dev_guide/plugin_tutorial/2_plugin_start.html)

BepInEx provides project templates for convenient development.

1. Install the .NET SDK. You will need .NET 6.0 SDK or newer. Download it from the [official Microsoft website](https://dotnet.microsoft.com/en-us/download).

2. Install the BepInEx templates. Open a terminal and run the following command to install the templates for BepInEx 6:

        :::bash
        dotnet new install BepInEx.Templates::2.0.0-be.4 --nuget-source https://nuget.bepinex.dev/v3/index.json

3. Navigate to the folder where you want to keep your project and run the command that creates an IL2CPP plugin structure:

        :::bash
        dotnet new bep6plugin_unity_il2cpp -n PluginName

This will create a `PluginName` folder containing Plugin.cs and `PluginName.csproj`.

## Setting up access to game code (Interop)

For your mod to access DiceKingdoms classes and methods, the project needs access to the Interop assemblies. BepInEx generates these on the first game launch, and they contain managed wrappers for the game's native code. The most convenient way is to create a symbolic link to the interop folder in the game root. This lets the project see up-to-date assemblies without copying them manually.

/// tab | Linux

    :::bash
    ln -s "/path/to/game/BepInEx/interop" "/path/to/your/project/PluginName/interop"
///

/// tab | Windows

    :::powershell
    mklink /D "C:\path\to\your\project\PluginName\interop" "C:\path\to\game\BepInEx\interop"
///

Now you need to tell the project where to find the game DLLs. Open `PluginName.csproj` and add following element:

```xml
<ItemGroup>
    <Reference Include="interop/*.dll" />
</ItemGroup>
```

## Building and installing the mod

1. Build the project. In the terminal, from the project folder, run:

        :::bash
        dotnet build

2. Copy the mod into the game. After a successful build, your mod's DLL will be located at `bin/Debug/net6.0/PluginName.dll`. Copy it to the `BepInEx/plugins` folder in the game directory.

3. It is recommended to create script for copying mod

/// tab | Linux

    :::bash
    cp -f "bin/Debug/net6.0/PluginName.dll" "/path/to/game/BepInEx/plugins/"
///

/// tab | Windows

    :::powershell
    copy /Y "bin\Debug\net6.0\PluginName.dll" "C:\path\to\game\BepInEx\plugins\"
///

After these steps, you can launch the game and BepInEx will load your mod.
