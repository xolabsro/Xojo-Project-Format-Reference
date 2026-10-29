# Building, Packaging and Installing

Every build compiles your sources **plus `GlueCode/PluginMain.cpp`** from [SDK/](SDK/), with `SDK/Includes` on the
include path, into one shared library. Build one library per target.

The commands below run from the plugin's working folder ([README](README.md#4-where-the-files-go)) and use `SDK`
for the full path of this kit's `Plugins/SDK` folder, e.g. `F:\Xojo_Templates_Kit\Plugins\SDK`.

## Windows (Visual Studio C++ toolchain)

`cl` needs the Visual Studio build environment, set up by `vcvars64.bat`. Find the Visual Studio folder with:

```bat
"%ProgramFiles(x86)%\Microsoft Visual Studio\Installer\vswhere.exe" -latest -products * -property installationPath
```

`vcvars64.bat` is in that folder under `VC\Auxiliary\Build`. Run the setup and the compiler in **one** `cmd`
call, so the environment carries over:

```bat
cmd /c "call "<VS folder>\VC\Auxiliary\Build\vcvars64.bat" >nul 2>&1 && cd /d <working folder> && if not exist build\x64 mkdir build\x64 && cl /nologo /LD /EHsc /O2 /MD /std:c++17 /utf-8 /DWIN32 /DNDEBUG /D_WINDOWS /D_USRDLL /I "<SDK>\Includes" src\MyPlugin.cpp "<SDK>\GlueCode\PluginMain.cpp" /Fo:build\x64\ /Fe:build\x64\MyPlugin.dll"
```

- **`/DWIN32` is required, even for 64-bit.** `PluginMain.cpp` exports `REALPluginMain` only when `WIN32` is
  defined, and the compiler doesn't define it on x64. Without it the DLL builds, but Xojo can't load it.
- Check the result with `dumpbin /exports build\x64\MyPlugin.dll` (in the same `cmd` call after `vcvars64.bat`):
  `REALPluginMain` must be listed.
- `/MD` links the Visual C++ runtime as a DLL; Xojo ships that runtime with every Windows build.
- `vcvars64.bat` may print `'vswhere.exe' is not recognized`. It's harmless; the build still works.
- Some agent shells can't run a `.bat` build script directly. The single inline `cmd /c "…"` call above works.
- Other architectures: `vcvars32.bat` for 32-bit x86, `vcvarsamd64_arm64.bat` for ARM64. Build each into its own
  folder (`build\x86`, `build\arm64`).
- Link extra Windows libraries by adding them at the end of the `cl` line: `user32.lib comctl32.lib`.

## macOS (Xcode command-line tools) *(untested)*

```sh
mkdir -p build/mac
echo "_REALPluginMain" > build/exports.txt
clang++ -dynamiclib -O2 -std=c++17 -arch x86_64 -arch arm64 -mmacosx-version-min=11.0 \
  -I "$SDK/Includes" -exported_symbols_list build/exports.txt \
  -framework CoreFoundation -framework Carbon \
  -o build/mac/MyPlugin.dylib src/MyPlugin.cpp "$SDK/GlueCode/PluginMain.cpp"
```

- Two `-arch` flags give one universal binary for `Mac Universal`.
- The exports file keeps `REALPluginMain` as the only exported symbol.
- The SDK headers include `<Carbon/Carbon.h>` on macOS, so link Carbon and CoreFoundation.
- For Cocoa views: compile your `.mm` files and `$SDK/GlueCode/PluginMainCocoa.mm` too, and add `-framework Cocoa`.
- iOS builds (`iOS Device`, `iOS Simulator`) need the iOS SDK: build with Xcode or `xcrun -sdk iphoneos clang++`
  (and `iphonesimulator`).

## Linux (g++) *(untested)*

```sh
mkdir -p build/linux
g++ -shared -fPIC -O2 -std=c++17 -I "$SDK/Includes" \
  -o build/linux/MyPlugin.so src/MyPlugin.cpp "$SDK/GlueCode/PluginMain.cpp"
```

Add `$(pkg-config --cflags --libs gtk+-3.0)` when the plugin uses GTK. Build on each architecture you need
(x86-64, arm64, …) or use a cross compiler. A plugin that ships a dependency in `Libs/` also needs
`-Wl,-rpath,'$ORIGIN'` (see [Dependency libraries](#dependency-libraries-libs)).

## Packaging

A plugin is a **zip archive with the `.xojo_plugin` extension**. Build the layout in a staging folder, zip the
staging folder's *contents*, and rename the `.zip` to `.xojo_plugin`. **Folder names are case-sensitive.**

Contents of the archive:

```text
PluginInfo.json                optional, shown in the IDE's About box
MyPlugin/                      the plugin name
├── Build Resources/
│   ├── Windows x86-64/        MyPlugin.dll   (64-bit)
│   ├── Windows arm64/         MyPlugin.dll
│   ├── Windows/               MyPlugin.dll   (32-bit)
│   ├── Mac Universal/         MyPlugin.dylib (x86_64 + arm64)
│   ├── Linux x86-64/          MyPlugin.so
│   └── Linux arm64/           MyPlugin.so
└── IDE Resources/             optional
    ├── Controls Palette/      Library icons (see Controls.md)
    └── Help/                  plugin docs, must contain index.html
```

All `Build Resources` folder names:

| Folder | Binary |
| ------ | ------ |
| `Windows` | 32-bit x86 DLL |
| `Windows x86-64` | 64-bit x86 DLL |
| `Windows arm64` | ARM64 DLL |
| `WinUI x86`, `WinUI x86-64`, `WinUI arm64` | WinUI builds; each DLL must have the same file name as the one in the matching `Windows …` folder |
| `Mac Universal` | one dylib with x86_64 and arm64 |
| `Mac x86-64`, `Mac arm64` | single-architecture dylibs (use instead of `Mac Universal`) |
| `Linux X86`, `Linux x86-64`, `Linux ARM`, `Linux arm64` | `.so` per architecture |
| `iOS Device` (arm64), `iOS Simulator` | iOS builds |

- Include only the targets you built. A project can only be built for targets that have a binary.
- Library file names may differ between folders (`MyPlugin64.dll`), except the WinUI rule above.

### Dependency libraries (`Libs/`)

When the plugin uses a third-party shared library, either link it into the plugin or ship it in a `Libs/` folder
inside the target's `Build Resources` folder (Xojo 2024r1+):

```text
MyPlugin/Build Resources/
├── Linux x86-64/
│   ├── MyPlugin.so
│   └── Libs/
│       └── libfoo.so.1
└── Windows x86-64/
    ├── MyPlugin.dll
    └── Libs/
        └── foo.dll
```

- **Load order:** Xojo loads every library in `Libs/` first, then the plugin's main library. The dependency is
  already loaded when the main library needs it.
- `Libs/` works in any target folder. Xojo's own database plugins use it for Linux and link statically elsewhere,
  so the Linux setup below is the proven one; Windows and macOS use of `Libs/` is *(unverified)*.
- Each target folder needs its own `Libs/` with that target's build of the dependency. Targets that link the
  dependency statically have no `Libs/` folder; mixing both approaches across targets is fine.
- **The file name must match what the main library asks for.** On Linux that's the dependency's soname
  (`libfoo.so.1`, not `libfoo.so`); on Windows, the DLL name in the import table. Check with `readelf -d MyPlugin.so`
  (`NEEDED` lines) or `dumpbin /dependents MyPlugin.dll`.
- **Linux: link the main library with `-Wl,-rpath,'$ORIGIN'`**, so it looks for its dependencies next to itself.
  Xojo's own plugins that use `Libs/` are built this way. Keep the single quotes so the shell doesn't expand
  `$ORIGIN`:

```sh
g++ -shared -fPIC -O2 -std=c++17 -I "$SDK/Includes" -I deps/include \
  -o build/linux/MyPlugin.so src/MyPlugin.cpp "$SDK/GlueCode/PluginMain.cpp" \
  -L deps/lib -lfoo -Wl,-rpath,'$ORIGIN'
readelf -d build/linux/MyPlugin.so | grep -E 'NEEDED|RUNPATH'    # expect libfoo.so.1 and $ORIGIN
```

- macOS *(unverified)*: give the dependency a loader-relative install name before linking against it
  (`install_name_tool -id @loader_path/libfoo.dylib libfoo.dylib`), and ship a universal (x86_64 + arm64) build of it.
- Ship the dependency's license file if its license requires it (put it in `IDE Resources/Help/` or tell the user).

**Link statically** (the default) when:

- the library is available as a static library (`.lib`, `.a`; on Linux built with `-fPIC`) and its license allows
  static linking;
- you want one self-contained binary per target, with no risk that another plugin or the system provides a
  different version of the same library.

**Ship it in `Libs/`** when:

- only a shared library is available (a vendor binary, or a library that's impractical to build statically);
- its license requires dynamic linking, or lets users replace it (LGPL);
- it's large and you want the plugin's own library to stay small, or the dependency is updated separately.

Libraries in `Libs/` are loaded by name, so two plugins shipping different versions of the same library can clash.
Prefer static linking unless one of the reasons above applies.

Zip it with `tar`, which ships with Windows 10+ and macOS and writes the forward-slash paths a zip needs
(`-a` picks the zip format from the `.zip` extension):

```sh
tar -a -c -f build/MyPlugin.zip -C build/package PluginInfo.json MyPlugin
mv build/MyPlugin.zip MyPlugin.xojo_plugin
```

On Windows PowerShell use `Move-Item` instead of `mv`. Leave `PluginInfo.json` out of the `tar` line if you don't
have one. Check the archive: its top level must be `PluginInfo.json` and the `MyPlugin/` folder, not a
`package/` folder.

`PluginInfo.json` (all keys optional):

```json
{
  "Version": "1.0",
  "Copyright": "Your name",
  "Description": "What the plugin does",
  "BuildDate": "2026-10-29",
  "IDEVersion": "2026.03"
}
```

## Installing (the user does this)

Ask the user to:

1. Quit Xojo.
2. Copy `MyPlugin.xojo_plugin` into the **`Plugins` folder next to the Xojo application** (on Windows it's inside the
   Xojo folder under `Program Files`, so copying needs admin rights).
3. Start Xojo and open the project.
4. Confirm that the plugin's items autocomplete in the code editor (and that controls appear in the Library).

If the plugin doesn't show up, check, in order: that it's a zip archive and its top level is right; the folder names
and their case; that the binary for the user's OS and architecture is in the right `Build Resources` folder; on
Windows, that `REALPluginMain` is exported (`/DWIN32`); that every dependency is linked in or present in `Libs/`
under the exact name the library asks for; that `PluginEntry` registers every item.

If the plugin loads but the project fails to compile with an error located in the plugin (**File** = the plugin's
module or class, **Location** = a member), a declaration string is invalid, often because it uses a reserved word.
See [Writing.md](Writing.md#declarations-are-xojo-source-strings).

## Updating

Xojo loads plugins only at launch. After every rebuild, the user must quit Xojo, replace the `.xojo_plugin`, and
start Xojo again.

## Using the plugin from Xojo

- Plugin items are **not project items**. Nothing about the plugin goes into the `.xojo_project` manifest or any
  project file.
- Use them like framework code: `MyModule.AddNumbers(1, 2)`, `Var c As New Counter`, `AddHandler c.Changed, …`.
  Members flagged `REALScopeGlobal` need no module prefix.
- A plugin control appears in the Library. Ask the user to place it on the layout in the IDE the first time, then
  read the saved layout file to see how the IDE writes it, before writing such blocks yourself.
- The project compiles only where the plugin is installed. Tell the user that anyone who opens the project needs the
  same `.xojo_plugin` in their `Plugins` folder.
- Built apps include the plugin automatically; nothing extra needs to ship.
