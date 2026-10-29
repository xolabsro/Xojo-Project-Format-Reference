# Complete Example

A small console-safe plugin with a module and a class. It shows module methods, a constant, a class with stored,
computed and string properties, an instance method, a shared method that creates an instance, an event, and an
exception. Tested on Windows x64 with Xojo 2026r3.1.

What it adds to Xojo:

```text
Module TextTools
  Const kVersion As String = "1.0"
  Function RepeatText(s As String, count As Integer) As String    // OutOfBoundsException if count < 0

Class Counter
  Property Value As Integer      // computed; setting it raises Changed
  Property StepSize As Integer   // stored
  Property Label As String       // computed, kept in instance data
  Sub Increment()                // Value = Value + StepSize
  Shared Function Create(start As Integer) As Counter
  Event Changed(newValue As Integer)
```

## src/TextTools.cpp

```cpp
#include "rb_plugin.h"
#include <string.h>
#include <string>

// ---------------------------------------------------------------------------
// Helpers

static REALclassRef gOutOfBoundsException = nullptr;   // set in PluginEntry

static REALstring MakeString(const std::string &s)
{
	return REALBuildStringWithEncoding(s.data(), (int)s.size(), kREALTextEncodingUTF8);
}

static std::string ToStdString(REALstring s)
{
	std::string out;
	REALstringData data;
	if (s && REALGetStringData(s, kREALTextEncodingUTF8, &data)) {
		out.assign((const char *)data.data, data.length);
		REALDisposeStringData(&data);
	}
	return out;
}

static void RaiseOutOfBounds(const char *message)
{
	REALobject err = REALnewInstanceWithClass(gOutOfBoundsException);
	REALstring msg = MakeString(message);
	REALSetPropValueString(err, "Message", msg);
	REALUnlockString(msg);
	REALRaiseException(err);
}

// ---------------------------------------------------------------------------
// Module TextTools

static REALstring RepeatText(REALstring s, RBInteger count)
{
	if (count < 0) {
		RaiseOutOfBounds("count must be 0 or more");
		return nullptr;
	}
	std::string piece = ToStdString(s);
	std::string result;
	result.reserve(piece.size() * (size_t)count);
	for (RBInteger i = 0; i < count; i++) result += piece;
	return MakeString(result);   // new string: returned as is
}

static REALmethodDefinition TextToolsMethods[] = {
	{ (REALproc)RepeatText, REALnoImplementation, "RepeatText(s As String, count As Integer) As String", REALconsoleSafe },
};

static REALconstant TextToolsConstants[] = {
	{ "kVersion As String = \"1.0\"" },
};

static REALmoduleDefinition TextToolsModule = {
	kCurrentREALControlVersion,
	"TextTools",
	TextToolsMethods, _countof(TextToolsMethods),
	TextToolsConstants, _countof(TextToolsConstants),
};

// ---------------------------------------------------------------------------
// Class Counter

struct CounterData {
	RBInteger value;
	RBInteger stepSize;
	REALstring label;
};

static void CounterInit(REALobject instance);
static void CounterDone(REALobject instance);
static RBInteger ValueGetter(REALobject instance, RBInteger param);
static void ValueSetter(REALobject instance, RBInteger param, RBInteger newValue);
static REALstring LabelGetter(REALobject instance, RBInteger param);
static void LabelSetter(REALobject instance, RBInteger param, REALstring newLabel);
static void Increment(REALobject instance);
static REALobject CreateCounter(RBInteger start);

static REALproperty CounterProperties[] = {
	{ "", "Value", "Integer", REALconsoleSafe, (REALproc)ValueGetter, (REALproc)ValueSetter },
	{ "", "StepSize", "Integer", REALconsoleSafe, REALstandardGetter, REALstandardSetter, FieldOffset(CounterData, stepSize) },
	{ "", "Label", "String", REALconsoleSafe, (REALproc)LabelGetter, (REALproc)LabelSetter },
};

static REALmethodDefinition CounterMethods[] = {
	{ (REALproc)Increment, REALnoImplementation, "Increment()", REALconsoleSafe },
};

static REALmethodDefinition CounterSharedMethods[] = {
	{ (REALproc)CreateCounter, REALnoImplementation, "Create(start As Integer) As Counter", REALconsoleSafe },
};

static REALevent CounterEvents[] = {
	{ "Changed(newValue As Integer)" },
};

static REALclassDefinition CounterDefinition = {
	kCurrentREALControlVersion,
	"Counter",
	nullptr,                      // superName
	sizeof(CounterData),
	0,                            // forSystemUse
	(REALproc)CounterInit,
	(REALproc)CounterDone,
	CounterProperties, _countof(CounterProperties),
	CounterMethods, _countof(CounterMethods),
	CounterEvents, _countof(CounterEvents),
	nullptr, 0,                   // eventInstances
	nullptr,                      // interfaces
	nullptr, 0,                   // attributes
	nullptr, 0,                   // constants
	0,                            // mFlags: set by SetClassConsoleSafe
	nullptr, 0,                   // sharedProperties
	CounterSharedMethods, _countof(CounterSharedMethods),
};

static void RaiseChanged(REALobject instance, RBInteger newValue)
{
	typedef void (*ChangedFn)(REALobject, RBInteger);
	ChangedFn fn = (ChangedFn)REALGetEventInstance((REALcontrolInstance)instance, &CounterEvents[0]);
	if (fn) fn(instance, newValue);
}

static void CounterInit(REALobject instance)
{
	ClassData(CounterDefinition, instance, CounterData, me);
	me->value = 0;
	me->stepSize = 1;
	me->label = nullptr;
}

static void CounterDone(REALobject instance)
{
	ClassData(CounterDefinition, instance, CounterData, me);
	REALUnlockString(me->label);
	me->label = nullptr;
}

static RBInteger ValueGetter(REALobject instance, RBInteger)
{
	ClassData(CounterDefinition, instance, CounterData, me);
	return me->value;
}

static void ValueSetter(REALobject instance, RBInteger, RBInteger newValue)
{
	ClassData(CounterDefinition, instance, CounterData, me);
	if (me->value == newValue) return;
	me->value = newValue;
	RaiseChanged(instance, newValue);
}

static REALstring LabelGetter(REALobject instance, RBInteger)
{
	ClassData(CounterDefinition, instance, CounterData, me);
	REALLockString(me->label);     // returning a stored string: lock it first
	return me->label;
}

static void LabelSetter(REALobject instance, RBInteger, REALstring newLabel)
{
	ClassData(CounterDefinition, instance, CounterData, me);
	REALUnlockString(me->label);   // release the old value
	me->label = newLabel;
	REALLockString(me->label);     // keep the new one
}

static void Increment(REALobject instance)
{
	ClassData(CounterDefinition, instance, CounterData, me);
	ValueSetter(instance, 0, me->value + me->stepSize);
}

static REALobject CreateCounter(RBInteger start)
{
	REALobject obj = REALnewInstanceOfClass(&CounterDefinition);   // runs CounterInit
	ClassData(CounterDefinition, obj, CounterData, me);
	me->value = start;
	return obj;                    // new object: returned as is
}

// ---------------------------------------------------------------------------

void PluginEntry(void)
{
	gOutOfBoundsException = REALGetClassRef("OutOfBoundsException");

	REALRegisterModule(&TextToolsModule);

	SetClassConsoleSafe(&CounterDefinition);
	REALRegisterClass(&CounterDefinition);
}
```

## Build and package (Windows x64)

From the working folder `TextTools\` (with `src\TextTools.cpp`), where `<VS folder>` comes from `vswhere` and
`<SDK>` is this kit's `Plugins\SDK` (see [Building.md](Building.md)):

```bat
cmd /c "call "<VS folder>\VC\Auxiliary\Buildcvars64.bat" >nul 2>&1 && if not exist buildd mkdir buildd && cl /nologo /LD /EHsc /O2 /MD /std:c++17 /utf-8 /DWIN32 /DNDEBUG /D_WINDOWS /D_USRDLL /I "<SDK>\Includes" src\TextTools.cpp "<SDK>\GlueCode\PluginMain.cpp" /Fo:buildd\ /Fe:buildd\TextTools.dll"
```

```powershell
New-Item -ItemType Directory -Force "build\package\TextTools\Build Resources\Windows x86-64" | Out-Null
Copy-Item buildd\TextTools.dll "build\package\TextTools\Build Resources\Windows x86-64\"
tar -a -c -f build\TextTools.zip -C build\package TextTools
Move-Item build\TextTools.zip TextTools.xojo_plugin
```

Then ask the user to copy `TextTools.xojo_plugin` into Xojo's `Plugins` folder and restart Xojo. Other platforms:
see [Building.md](Building.md).

## Using it from Xojo

Once the user confirms `TextTools.` and `Counter` autocomplete:

```xojo
Var s As String = TextTools.RepeatText("ab", 3)   // "ababab"

Var c As Counter = Counter.Create(10)
c.StepSize = 5
c.Label = "Clicks"
AddHandler c.Changed, AddressOf CounterChanged
c.Increment                                       // Value = 15, raises Changed

Try
  s = TextTools.RepeatText("x", -1)
Catch err As OutOfBoundsException
  System.DebugLog(err.Message)                    // "count must be 0 or more"
End Try
```

with a method on the same window or class:

```xojo
Sub CounterChanged(sender As Counter, newValue As Integer)
  System.DebugLog(sender.Label + ": " + newValue.ToString)
End Sub
```

Expected output in the Messages pane: `Clicks: 15`, then `count must be 0 or more`. When **Project ▸ Break On
Exceptions** is on (the default), the debugger first stops on the `RepeatText("x", -1)` line: that's the plugin's
exception being raised. Resume and the `Catch` handles it.

A `Counter` subclass in the project can also implement the `Changed` event directly. Nothing about the plugin is
written into the project files; only the code above refers to it.
