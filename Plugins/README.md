# Xojo Plugins — Guide for AI Agents

How to decide whether an app needs a plugin, then write, build, package and install it, and use it from a Xojo
project (Xojo 2026r3+). A plugin is a native shared library (`.dll`, `.dylib`, `.so`) written in C/C++ with the
**Xojo Plugins SDK**. It adds modules, classes, interfaces, controls, structures and enums to the Xojo language.

This folder is self-contained: the guide covers everything you need, and [SDK/](SDK/) holds the SDK headers and
glue code you compile against.

> **Status:** derived from the Plugins SDK 2026r1 headers, glue code and examples. [Example.md](Example.md)
> and the [Database.md](Database.md) example are tested end to end on Windows x64 (build, zip package, install, use
> in Xojo 2026r3.1). The macOS and Linux commands
> are not tested. Items marked *(unverified)* are inferred from the headers only.

## Files

| File | Read it when |
| ---- | ------------ |
| [Writing.md](Writing.md) | Writing the plugin source: skeleton, declarations, C signatures, types, strings, events |
| [Definitions.md](Definitions.md) | Filling a definition struct: every field, in order, for every struct |
| [Controls.md](Controls.md) | The plugin adds a Desktop or iOS control, or hosts a native OS control |
| [Database.md](Database.md) | The plugin adds a `Database` subclass for an engine Xojo doesn't support |
| [SdkFunctions.md](SdkFunctions.md) | Looking up an SDK function: strings, objects, arrays, variants, framework calls |
| [Building.md](Building.md) | Compiling, packaging as a `.xojo_plugin` zip, installing, updating |
| [Example.md](Example.md) | A complete small plugin and the Xojo code that uses it |
| [SDK/](SDK/) | `Includes/` (headers) and `GlueCode/` (`PluginMain.cpp`), compiled into every plugin |

## 1. Does the app need a plugin?

Usually not. Pick the first option that does the job:

1. **Plain Xojo code** and the Xojo framework.
2. **`Declare` / `Soft Declare`** into an OS API or an existing shared library. This covers most "call a native
   function" needs, with no C++ toolchain.
3. **A Xojo library** (`.xojo_library`) when the goal is to share Xojo code between projects.
4. **A C++ plugin**, only when one of these is true:
   - the app needs a C/C++ library linked in, and declaring into it is impractical (C++ classes, macros, inline
     functions, many structs or callbacks);
   - the work needs native threads or callbacks that Xojo can't receive safely through a Declare;
   - the app must host a native OS control (Win32, Cocoa, GTK, XAML) inside a Xojo layout;
   - the app needs a database engine Xojo doesn't support, through its C client library
     ([Database.md](Database.md));
   - a hot loop needs native speed.

Don't suggest commercial third-party plugins.

## 2. Propose it to the user first

Never start a plugin unannounced. Tell the user:

- **why** a plugin is needed, and why options 1–3 don't do the job;
- **which targets** it must be built for. Each OS and architecture is a separate binary, normally built on that OS;
  iOS builds need macOS;
- **what they need:** a C++ toolchain (Visual Studio with the "Desktop development with C++" workload on Windows,
  Xcode or its command-line tools on macOS, `g++` on Linux);
- **what they'll do:** copy the finished plugin into Xojo's `Plugins` folder and restart the IDE.

Continue only if the user agrees.

## 3. Workflow

1. Write the plugin source ([Writing.md](Writing.md)).
2. Build one binary per target against [SDK/](SDK/) ([Building.md](Building.md)). Fix compile errors before going on.
3. Package the binaries as a `Name.xojo_plugin` **zip archive** ([Building.md](Building.md#packaging)).
4. **Stop and ask the user to install it:** copy `Name.xojo_plugin` into the `Plugins` folder next to the Xojo
   application, then restart the IDE.
5. **Wait for the user to confirm it loaded:** the plugin's module or class autocompletes in the code editor, and a
   control appears in the Library. Don't write Xojo code against the plugin before that.
6. Continue the Xojo project ([Building.md](Building.md#using-the-plugin-from-xojo)).

## 4. Where the files go

- **SDK:** use `Plugins/SDK/Includes` and `Plugins/SDK/GlueCode` from this kit as they are, by path. Never edit them
  and don't copy them into the plugin's folder.
- **Plugin source and build output:** in a working folder of their own, outside the Xojo project folder and outside
  this kit:

```text
MyPlugin/
├── src/MyPlugin.cpp       your code
├── build/                 compiler output and the packaging staging folder
└── MyPlugin.xojo_plugin   the zip archive you hand to the user
```
