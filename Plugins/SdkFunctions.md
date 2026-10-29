# SDK Function Reference

The functions a plugin uses most, grouped by task. All are declared in [`SDK/Includes/rb_plugin.h`](SDK/Includes/rb_plugin.h).
`<Type>` stands for one of: `Boolean`, `Integer`, `UInteger`, `Int8`, `Int16`, `Int32`, `Int64`, `UInt8`, `UInt16`,
`UInt32`, `UInt64`, `Single`, `Double`, `Currency`, `Color`, `String`, `Text`, `Object`, `Ptr`.

## Registration (call in `PluginEntry`)

| Function | Registers |
| -------- | --------- |
| `REALRegisterModule(REALmoduleDefinition *)` | a module (and the classes, interfaces, modules, controls inside it) |
| `REALRegisterClass(REALclassDefinition *)` | a class |
| `REALRegisterInterface(REALinterfaceDefinition *)` | an interface |
| `REALRegisterControl(REALcontrol *)` | a Desktop control |
| `REALRegisterMobileControl(REALmobileControl *)` | an iOS control |
| `SetClassConsoleSafe(REALclassDefinition *)` | marks a class console-safe (call before registering) |
| `SetClassWebOnly(REALclassDefinition *)` | limits a class to Web projects |

## Strings

| Function | Does |
| -------- | ---- |
| `REALBuildStringWithEncoding(const char *bytes, int byteCount, uint32_t encoding)` | new string you own |
| `REALGetStringData(REALstring, uint32_t encoding, REALstringData *out)` | bytes converted to `encoding`; `false` if impossible |
| `REALDisposeStringData(REALstringData *)` | frees what `REALGetStringData` filled |
| `REALStringLength(REALstring)` | length in bytes |
| `REALGetStringEncoding(REALstring)` | the string's encoding |
| `REALConvertString(REALstring, uint32_t encoding)` | converted copy you own |
| `REALAddStrings(REALstring, REALstring)` | concatenation you own |
| `REALCompareStrings(REALstring, REALstring)` | `<0`, `0`, `>0`, like Xojo's comparison |
| `REALLockString` / `REALUnlockString` | add / release a reference |

Encodings: `kREALTextEncodingUTF8` (default for Xojo strings), `kREALTextEncodingASCII`, `kREALTextEncodingUTF16`,
`kREALTextEncodingUTF16LE`, `kREALTextEncodingUTF16BE`, `kREALTextEncodingUTF32`, `kREALTextEncodingUTF32LE`,
`kREALTextEncodingUTF32BE`.

`REALstringData` has `data` (bytes, not NUL-terminated), `length` and `encoding`. Never copy the struct and dispose
both copies.

## Objects and classes

| Function | Does |
| -------- | ---- |
| `REALGetClassRef(const char *className)` | class reference for any Xojo or plugin class; cache it from `PluginEntry` |
| `REALnewInstanceWithClass(REALclassRef)` | new instance you own; Xojo `Constructor` methods are not called |
| `REALnewInstanceOfClass(REALclassDefinition *)` | new instance of your own plugin class; its `constructor` field runs |
| `REALObjectIsA(REALobject, REALclassRef)` | Xojo `IsA` |
| `REALGetClassData(REALobject, REALclassDefinition *)` | instance data (or use the `ClassData` macro) |
| `REALGetPluginData(REALobject, REALclassRef)` | instance data of a class from another plugin |
| `REALLockObject` / `REALUnlockObject` | add / release a reference |

## Properties of any object

| Function | Does |
| -------- | ---- |
| `REALGetPropValue<Type>(REALobject, const char *name, T *out)` | read a property by name; `false` if not found |
| `REALSetPropValue<Type>(REALobject, const char *name, T value)` | write a property by name |
| `REALGetPropValueCString`, `…WString`, `…PString` (and `Set…`) | string properties as C, wide or Pascal strings |

Examples: `REALGetPropValueInteger(control, "Handle", &h)`, `REALSetPropValueString(err, "Message", msg)`,
`REALGetPropValueString(variant, "StringValue", &s)`. Object values you read are yours to unlock.

## Calling Xojo code

| Function | Does |
| -------- | ---- |
| `REALLoadFrameworkMethod(const char *declaration)` | global framework function, e.g. `"Ticks() As Integer"` |
| `REALLoadObjectMethod(REALobject, const char *declaration)` | method of an object; call with the object first |
| `REALLoadSharedMethod(REALclassRef, const char *declaration)` | shared method; no object parameter |
| `REALInterfaceRoutine(REALobject, const char *interface, const char *method)` | interface method of an object |
| `REALGetDelegateInvoker(REALobject delegate)` | function for a delegate; call with the delegate first |
| `REALGetEventInstance(REALcontrolInstance, REALevent *)` | handler of one of your events, or `nullptr` |
| `REALIsEventHandled(REALobject, REALstring eventName)` | whether an event has a handler |
| `REALAddEventHandler(REALobject, REALstring eventName, void *handler)` | attach a C handler to any object's event (first parameter is the object) |
| `REALRemoveEventHandler(REALobject, REALstring eventName, void *handler)` | detach it |

All return `nullptr`/`false` on failure; always check before calling.

## Arrays

| Function | Does |
| -------- | ---- |
| `REALCreateArray(REALArrayType type, RBInteger bounds)` | new array you own; `bounds` is the last index, `-1` for empty |
| `REALGetArrayUBound(REALarray)` | last index (`-1` when empty) |
| `REALGetArrayValue<Type>(REALarray, RBInteger index, T *out)` | read an element |
| `REALSetArrayValue<Type>(REALarray, RBInteger index, T value)` | write an element |
| `REALInsertArrayValue<Type>(REALarray, RBInteger index, T value)` | insert an element |
| `REALGetArrayStructure(REALarray, RBInteger index, void *out)` | copy a structure element into your C struct |

`REALArrayType` values: `kTypeInteger` (follows the target), `kTypeSInt8`, `kTypeSInt16`, `kTypeSInt32`,
`kTypeSInt64`, `kTypeUInt8`, `kTypeUInt16`, `kTypeUInt32`, `kTypeUInt64`, `kTypeFloat32` (Single),
`kTypeFloat64` (Double), `kTypeBoolean`, `kTypeString`, `kTypeText`, `kTypeObject`, `kTypeColor`, `kTypeCurrency`,
`kTypePtr`, `kTypeStructure`.

## Variants

| Function | Does |
| -------- | ---- |
| `REALNewVariant<Type>(value)` | new Variant you own (`Integer`, `Int32`, `Int64`, `UInt32`, `UInt64`, `Double`, `Single`, `Boolean`, `Color`, `Currency`, `String`, `Text`, `Ptr`, `CString`, `WString`) |
| `REALNewVariantStructure(const void *data, size_t len)` | Variant holding a structure |
| `REALGetVariantStructure(REALobject variant, void *buffer, size_t len)` | copy it back out |

Read a Variant with the property functions: `REALGetPropValueInteger(v, "IntegerValue", &i)`,
`"DoubleValue"`, `"StringValue"`, `"BooleanValue"`, `"ColorValue"`, `"Int64Value"`, `"Type"`.

## Exceptions

| Function | Does |
| -------- | ---- |
| `REALRaiseException(REALobject exception)` | raise in the Xojo caller; return right after |
| `REALCallFunctionWithExceptionHandler(void (*fn)(void *), void *userData)` | call Xojo code and catch its exception; returns it (you own it) or `nullptr` |

## Pictures

| Function | Does |
| -------- | ---- |
| `REALBuildPictureFromBuffer(width, height, RBPixelType, void *buffer, rowBytes)` | new Picture from pixels (`kRBPixelRGB24`, `kRBPixelBGR24`, `kRBPixelXRGB32`, `kRBPixelBGRX32`) |
| `REALBuildPictureFromPictureDescription(REALpictureDescription *, bool passOwnership)` | Picture from a native image |
| `REALLockPictureDescription(REALpicture, REALpictureDescription *, uint32_t type)` | native image of a Picture; release with `REALUnlockPictureDescription` |

## Controls

| Function | Does |
| -------- | ---- |
| `REALGetControlBounds(REALcontrolInstance, Rect *)` | window-relative bounds |
| `REALGetControlVisible` / `REALSetControlVisible` | visibility |
| `REALGetControlEnabled(REALcontrolInstance)` | enabled state |
| `REALGetControlWindow(REALcontrolInstance)` | the window that holds the control |
| `REALGetControlData(REALcontrolInstance, REALcontrol *)` | instance data (or the `ControlData` macro) |
| `REALGetMobileControlData(REALcontrolInstance, REALmobileControl *)` | mobile instance data (or `MobileControlData`) |
| `REALIsDarkMode()` | app is in dark mode |
| `REALUsesWinUI()` | Windows app uses the WinUI framework |
| `REALUseNativeControlSizes()` | WinUI "native control sizes" build setting is on |

## Threads and runtime

| Function | Does |
| -------- | ---- |
| `REALIsXojoThread()` | current thread is a Xojo thread (Desktop, Console, Web) |
| `REALIsXojoMainThread()` | current thread is Xojo's main thread |
| `REALYieldToRB()` | let other Xojo threads run |
| `REALRegisterBackgroundTask(void (*fn)(void *), uint32_t period, void *data)` | call `fn` periodically on Xojo's thread; returns an id |
| `REALUnregisterBackgroundTask(int32_t id)` | stop it |
| `REALGetRBVersion()` | Xojo version as a double |
| `REALinRuntime()` | running in a built app rather than the IDE |
| `REALInDebugMode()` | running in a debug build |

`REALRegisterBackgroundTask` is a way to bring results from a native thread back to Xojo: the native thread stores
the result, the background task (on Xojo's thread) picks it up and raises the event.

## Avoid

Anything in `SDK/Includes/rb_plugin_deprecated.h` is deprecated. Prefer `REALGetStringData` over
`REALGetStringContents`, `REALstring` over `REALtext`, and the newer Graphics methods over legacy ones.
