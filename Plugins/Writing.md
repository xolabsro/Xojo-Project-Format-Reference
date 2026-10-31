# Writing a Plugin

The source of a plugin is plain C++ built against the SDK headers. Field-by-field struct layouts are in
[Definitions.md](Definitions.md); controls are in [Controls.md](Controls.md).

## Skeleton

```cpp
#include "rb_plugin.h"   // brings REALplugin.h and the rest of the SDK headers

// ... functions, member arrays and definitions ...

void PluginEntry(void)
{
	// Register every module, class, interface and control here.
	REALRegisterModule(&MyModuleDefinition);
}
```

- `PluginEntry` is the only function you must provide. `GlueCode/PluginMain.cpp` exports `REALPluginMain`, which
  Xojo calls when it loads the plugin (in the IDE, and in the built app); that calls your `PluginEntry`.
- One plugin can register any number of items.
- To be safe, register an interface before a class that implements it, and a class before a class that inherits
  from it.
- Use `nullptr` (or `NULL`) for empty fields. The SDK also defines `nil` and `REALnoImplementation`, which mean the same.

## Declarations are Xojo source strings

Every method, property, event, constant, structure field and enum value is declared with a string **in Xojo
syntax**, paired with the C function that implements it:

```cpp
static RBInteger AddNumbers(RBInteger a, RBInteger b)
{
	return a + b;
}

static REALmethodDefinition MyModuleMethods[] = {
	{ (REALproc)AddNumbers, REALnoImplementation, "AddNumbers(a As Integer, b As Integer) As Integer", REALconsoleSafe },
};

static REALconstant MyModuleConstants[] = {
	{ "kVersion As String = \"1.0\"" },
};
```

- Cast every function to `(REALproc)`.
- **Names must be valid Xojo identifiers, not reserved words.** Keywords such as `Step`, `Next`, `Loop`, `End`,
  `Select`, `Case`, `To`, `In`, `Is`, `Mod`, `Me`, `Self`, `Super`, `Nil`, `New`, `Var`, `Property`, `Event`,
  `Object`, and type names (`Integer`, `String`, …), can't be member or parameter names. When a name is also a Xojo
  keyword, pick a longer one (`StepSize`, not `Step`).
- Declaration strings aren't checked when the plugin is compiled or loaded. A bad one shows up only when the user
  compiles a Xojo project that uses the plugin: a *Syntax error* whose **File** is the plugin item (`TextTools`) and
  whose **Location** is the member (`Counter.Step`). Ask the user for that error text, fix the declaration, rebuild
  and have them reinstall.
- The C signature must match the declaration exactly: same parameter order, matching C types (see the type table),
  plus the hidden first parameter for instance members. A mismatch compiles but crashes at run time.
- Default values, `ByRef`, `Optional`, `ParamArray`, `Assigns` and `Extends` are written in the declaration the
  same way as in Xojo code.
- `ByRef` parameters arrive as pointers (`RBInteger *`, `REALstring *`, `MyStruct *`).
- Count arrays with `sizeof(arr) / sizeof(arr[0])` (or the SDK's `_countof(arr)`).

## Members and their C functions

| Member | Where | C signature |
| ------ | ----- | ----------- |
| Module method | module `methods` | parameters only: `RBInteger F(RBInteger a)` |
| Instance method | class `methods` | instance first: `void F(REALobject instance, REALstring s)` |
| Shared method | class `sharedMethods` | parameters only, no instance |
| Extends method | module `methods` | the extended object is the first parameter: `"Ext(Extends f As FolderItem) As String"` → `REALstring Ext(REALobject f)` |
| Instance property getter | class `properties` | `T Get(REALobject instance, RBInteger param)` |
| Instance property setter | class `properties` | `void Set(REALobject instance, RBInteger param, T value)` |
| Module / shared property getter | module `properties`, class `sharedProperties` | `T Get(void)` |
| Module / shared property setter | same | `void Set(T value)` |
| Stored instance property | class `properties` | getter `REALstandardGetter`, setter `REALstandardSetter`, param `FieldOffset(Data, field)` |
| Read-only property | any | setter `nullptr` |

`param` is the `param` field of the `REALproperty`; you can use one getter for several properties and tell them apart
by it. `REALstandardGetter`/`Setter` only work on instance properties of a class or control, never on shared or module
properties.

## Instance data

Classes and controls get a block of memory per instance. Describe it as a C struct and give its size in the
definition:

```cpp
struct CounterData {
	RBInteger count;
	REALstring label;
};

static void CounterConstructor(REALobject instance)
{
	ClassData(CounterDefinition, instance, CounterData, me);   // declares CounterData *me
	me->count = 0;
	me->label = nullptr;
}

static void CounterDestructor(REALobject instance)
{
	ClassData(CounterDefinition, instance, CounterData, me);
	REALUnlockString(me->label);   // release every string/object you hold
}
```

- The definition's `constructor` runs when an instance is created, before any Xojo `Constructor` method; use it to
  set defaults. The `destructor` runs when the instance is destroyed; release references there.
- `ClassData` is for classes, `ControlData` for desktop controls, `MobileControlData` for mobile controls. The first
  argument is the definition variable itself (not a pointer).
- The definition must be declared before any function that uses `ClassData` on it; forward-declare the functions,
  then the arrays and the definition, then the function bodies.

## Xojo → C types

| Xojo | C | Notes |
| ---- | - | ----- |
| `Integer` | `RBInteger` | 64-bit on 64-bit targets, 32-bit on 32-bit targets |
| `UInteger` | `RBUInteger` | |
| `Int8`, `Int16`, `Int32`, `Int64` | `int8_t`, `int16_t`, `int32_t`, `RBInt64` | |
| `UInt8`, `UInt16`, `UInt32`, `UInt64` | `uint8_t`, `uint16_t`, `uint32_t`, `RBUInt64` | |
| `Boolean` | `RBBoolean` | |
| `Single`, `Double` | `float`, `double` | |
| `Currency` | `REALcurrency` | Int64 holding value × 10000 |
| `Color` | `RBColor` | `0xAARRGGBB`; alpha `00` is opaque |
| `String` | `REALstring` | reference counted |
| `Text` | `REALtext` | reference counted (legacy) |
| `Ptr` | `void *` | |
| `Variant` | `REALobject` | read with `REALGetPropValue…(v, "IntegerValue", …)` |
| any class instance | `REALobject` | reference counted |
| `Graphics`, `Picture`, `FolderItem`, `MemoryBlock` | `REALgraphics`, `REALpicture`, `REALfolderItem`, `REALmemoryBlock` | all are `REALobject` |
| array `T()` | `REALarray` | see [SdkFunctions.md](SdkFunctions.md#arrays) |
| plugin structure | your C struct | same field order and sizes; passed `ByRef` as a pointer |
| plugin enum | the enum's integer type | `RBInteger` by default |

Use `RBInteger` for every `Integer`. **Never `long`:** it's 32-bit on 64-bit Windows, so the value gets truncated.

## Strings and reference counting

Strings (`REALstring`) and objects (`REALobject`) are reference counted.

| Situation | Do |
| --------- | -- |
| You built a string or created an object | You own one reference. Return it, or unlock it when done. |
| A string/object was passed in as a parameter | Use it during the call. Lock it only if you store it. |
| You store a string/object in instance or global data | Unlock the old value, assign, lock the new value. |
| You return a string/object you have stored | Lock it, then return it. |
| An event or framework call returned a string/object to you | You own it. Unlock it when done. |
| Destructor | Unlock everything you stored. |

`REALUnlockString(nullptr)` is safe. Strings: `REALLockString` / `REALUnlockString`. Objects:
`REALLockObject` / `REALUnlockObject`.

Building and reading strings (UTF-8 is the default encoding for Xojo strings):

```cpp
// C string → Xojo String
REALstring s = REALBuildStringWithEncoding(text, (int)strlen(text), kREALTextEncodingUTF8);

// Xojo String → UTF-8 bytes
REALstringData data;
if (REALGetStringData(s, kREALTextEncodingUTF8, &data)) {
	// data.data points to data.length bytes, not NUL-terminated
	std::string copy((const char *)data.data, data.length);
	REALDisposeStringData(&data);
}
```

A `nullptr` `REALstring` is the empty string.

## Events

A class or control declares events that the Xojo code handles:

```cpp
static REALevent CounterEvents[] = {
	{ "Changed(newValue As Integer)" },
	{ "Validate(value As Integer) As Boolean" },
};
```

To raise one, get the handler; it's `nullptr` when the Xojo code didn't implement the event:

```cpp
typedef void (*ChangedFn)(REALobject instance, RBInteger newValue);
ChangedFn fn = (ChangedFn)REALGetEventInstance((REALcontrolInstance)instance, &CounterEvents[0]);
if (fn) fn(instance, me->count);
```

The handler's first parameter is always the instance, then the event's parameters. If the event returns a string or
object, unlock it after use.

To handle an event of a **built-in superclass** instead (for example a class whose `superName` is `"Timer"`), list it
in the definition's `eventInstances`: `{ "Action", (REALproc)MyActionHandler }`.

## Calling the Xojo framework

Load any framework function or method by its Xojo declaration and call it through a function pointer:

```cpp
typedef RBInteger (*TicksFn)(void);
TicksFn ticks = (TicksFn)REALLoadFrameworkMethod("Ticks() As Integer");
RBInteger t = ticks ? ticks() : 0;

// Method on an object: the object is the first parameter
typedef void (*FillOvalFn)(REALgraphics, RBInteger, RBInteger, RBInteger, RBInteger);
FillOvalFn fillOval = (FillOvalFn)REALLoadObjectMethod((REALobject)g,
	"FillOval(x As Integer, y As Integer, w As Integer, h As Integer)");
```

- Always check for `nullptr`: a wrong declaration string returns `nullptr`.
- Read and write object properties with `REALGetPropValue<Type>` / `REALSetPropValue<Type>` (for example
  `REALSetPropValueColor(g, "DrawingColor", 0x00FF0000)`).
- Cache loaded pointers in `static` variables if you call them often.

## Raising exceptions

```cpp
static REALclassRef gRuntimeException;   // set in PluginEntry: REALGetClassRef("RuntimeException")

static void Fail(const char *message)
{
	REALobject err = REALnewInstanceWithClass(gRuntimeException);
	REALstring msg = REALBuildStringWithEncoding(message, (int)strlen(message), kREALTextEncodingUTF8);
	REALSetPropValueString(err, "Message", msg);
	REALUnlockString(msg);
	REALRaiseException(err);
}
```

Return from the plugin function right after `REALRaiseException`; the exception is raised in the Xojo code when the
call returns, and a `Try`/`Catch` around the call catches it. With **Break On Exceptions** on, the debugger stops on
the calling line first; that's normal. Use specific classes where they fit (`OutOfBoundsException`, `NilObjectException`,
`InvalidArgumentException`, `UnsupportedOperationException`). Never let a C++ exception escape into Xojo: catch it
and raise a Xojo exception instead.

## Threads

- Call SDK functions only on a Xojo thread. `REALIsXojoThread()` and `REALIsXojoMainThread()` tell you where you are.
- Do long work on your own native thread, store the result, and let the Xojo side collect it (for example a method
  the app polls from a `Timer`). Never raise an event or touch Xojo objects from a native thread.
- `REALYieldToRB()` lets other Xojo threads run during a long call on a Xojo thread.

## Targets, scope and console safety

Set these in the member's flags field:

| Flag | Meaning |
| ---- | ------- |
| `REALconsoleSafe` | Available in Console and Web projects (and Desktop). Without it, the member is GUI-only. |
| `REALScopeGlobal` | On a module member: usable without the module prefix, like a global. |
| `REALScopePublic`, `REALScopeProtected`, `REALScopePrivate` | Scope of the member. |

- Everything a console-safe member uses (parameter and return types) must be console-safe too.
- For a console-safe **class**, also call `SetClassConsoleSafe(&MyClassDefinition)` in `PluginEntry` before
  `REALRegisterClass`. `SetClassWebOnly` limits a class to Web projects.
- A module is always available; mark its members individually.

## Platform code

Use the SDK's target macros to separate platform code:

```cpp
#if TARGET_WIN32
	// Windows (Win32 and WinUI)
#elif TARGET_COCOA
	// macOS
#elif X_WINDOW
	// Linux
#endif
#if TARGET_64BIT
	// 64-bit build
#endif
```

`TARGET_OS_IPHONE` (from Apple's headers) is set for iOS builds.
