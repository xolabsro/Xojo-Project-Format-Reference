# Definition Structs

Every struct is initialized **positionally**: fields must be given in exactly this order. Trailing fields you leave
out are zero. To skip a field in the middle, give `nullptr` (pointers) or `0` (counts and flags).

These layouts match the SDK headers' default `kCurrentREALControlVersion` (13). Always put
`kCurrentREALControlVersion` in the `version` field, and never define that macro yourself.

Watch the **attribute pair order**: member structs list `attributeCount, attributes`; class, module and control
definitions list `attributes, attributeCount`.

## Members

### REALmethodDefinition

```cpp
{ function, setterFunction, declaration, flags, attributeCount, attributes }
```

| Field | Value |
| ----- | ----- |
| `function` | `(REALproc)MyFunction`; `nullptr` in an interface definition |
| `setterFunction` | always `REALnoImplementation` |
| `declaration` | Xojo signature: `"Name(a As Integer, b As String = \"x\") As Boolean"` |
| `flags` | `REALconsoleSafe`, `REALScopeGlobal`, scope flags, or `0` |

### REALproperty

```cpp
{ group, name, type, flags, getter, setter, param, editor, enumCount, enumEntries, attributeCount, attributes }
```

| Field | Value |
| ----- | ----- |
| `group` | Inspector group for control properties (`"Appearance"`, `"Behavior"`); `""` otherwise |
| `name` | property name |
| `type` | Xojo type name: `"Integer"`, `"String"`, `"Color"`, `"MyEnum"`, a class name |
| `flags` | `REALconsoleSafe`, `REALScopeGlobal`, scope flags, `REALpropRuntimeOnly` (hidden from the Inspector) |
| `getter` / `setter` | `(REALproc)Fn`, `REALstandardGetter` / `REALstandardSetter`, or `nullptr` for none |
| `param` | passed to computed getters/setters; with standard getters/setters: `FieldOffset(Data, field)` |
| `editor`, `enumCount`, `enumEntries` | leave `nullptr, 0, nullptr` |

### REALevent

```cpp
{ declaration, forSystemUse, attributeCount, attributes }
```

`declaration` is the event signature: `"Changed(newValue As Integer)"`. `forSystemUse` is `0`.

### REALeventInstance (handle a built-in superclass event)

```cpp
{ name, implementation }    // { "Action", (REALproc)MyActionHandler }
```

### REALconstant

```cpp
{ declaration, reserved1, flags, attributeCount, attributes }
```

`declaration` is `"kName As Type = value"` or `"kName = value"`: `"kMax As Integer = 10"`,
`"kTint = &cFF00FF"`, `"kGreeting As String = \"Hello\""`. `reserved1` is `nullptr`.

### REALdelegate *(unverified)*

```cpp
{ declaration, flags, attributeCount, attributes }
```

`declaration` is the delegate name and signature like a method: `"ProgressCallback(percent As Double) As Boolean"`.
To call a delegate object the Xojo code passes in, get its function with `REALGetDelegateInvoker(delegateObj)`; the
function takes the delegate object first, then the parameters:
`typedef RBBoolean (*ProgressFn)(REALobject, double);`.

### REALstructure

```cpp
static const char *PointFields[] = { "X As Double", "Y As Double", "Name As String * 32" };
static REALstructure MyStructures[] = {
	{ "PointD", REALScopePublic, PointFields, sizeof(PointFields) / sizeof(PointFields[0]) },
};
// fields: { name, flags, fields, numFields, attributeCount, attributes }
```

The matching C struct must have the same fields, order and sizes: `struct PointD { double X; double Y; char Name[32]; };`.
`String * n` is a fixed `char[n]`. Structures live in a module (`structures` field). Use `REALScopeGlobal` to make
the structure usable without the module prefix.

### REALenum

```cpp
static const char *ModeValues[] = { "Off", "Low = 10", "High" };
static REALenum MyEnums[] = {
	{ "Mode", nullptr, REALScopePublic, ModeValues, sizeof(ModeValues) / sizeof(ModeValues[0]) },
};
// fields: { name, type, flags, fields, numFields, attributeCount, attributes }
```

`type` is the integer type (`"UInt8"`, `"Int32"`); `nullptr` means `Integer`. In C, an enum value is that integer type.
Enums go in a module (`enums` field) or a class.

### REALattribute

```cpp
{ name, value }      // { "Deprecated", "" }
```

Attributes are optional everywhere. Leave the pair as `0, nullptr` (or `nullptr, 0`).

## Containers

### REALmoduleDefinition

Register with `REALRegisterModule(&def)`.

| # | Field | Value |
| - | ----- | ----- |
| 1 | `version` | `kCurrentREALControlVersion` |
| 2 | `name` | module name |
| 3–4 | `methods`, `methodCount` | `REALmethodDefinition[]` |
| 5–6 | `constants`, `constantCount` | `REALconstant[]` |
| 7–8 | `properties`, `propertyCount` | `REALproperty[]` (getter/setter take no instance) |
| 9–10 | `structures`, `structureCount` | `REALstructure[]` |
| 11–12 | `enums`, `enumCount` | `REALenum[]` |
| 13–14 | `attributes`, `attributeCount` | `REALattribute[]` |
| 15–16 | `delegates`, `delegateCount` | `REALdelegate[]` |
| 17–18 | `classes`, `classesCount` | `REALclassDefinition *[]` (array of pointers) |
| 19–20 | `interfaces`, `interfaceCount` | `REALinterfaceDefinition[]` |
| 21–22 | `modules`, `moduleCount` | `REALmoduleDefinition *[]` |
| 23–24 | `controls`, `controlCount` | `REALcontrol *[]` |

Classes, interfaces, modules and controls listed in a module live in its namespace (`MyModule.MyClass`). Register
only the outer module; don't also register the items it contains.

### REALclassDefinition

Register with `REALRegisterClass(&def)`.

| # | Field | Value |
| - | ----- | ----- |
| 1 | `version` | `kCurrentREALControlVersion` |
| 2 | `name` | class name |
| 3 | `superName` | super class name (`"Dictionary"`, `"Timer"`, another plugin class), or `nullptr` |
| 4 | `dataSize` | `sizeof(MyClassData)`, or `0` |
| 5 | `forSystemUse` | `0` |
| 6 | `constructor` | `(REALproc)Init`, `void Init(REALobject instance)`, or `nullptr` |
| 7 | `destructor` | `(REALproc)Done`, `void Done(REALobject instance)`, or `nullptr` |
| 8–9 | `properties`, `propertyCount` | instance properties |
| 10–11 | `methods`, `methodCount` | instance methods |
| 12–13 | `events`, `eventCount` | events the Xojo code can handle |
| 14–15 | `eventInstances`, `eventInstanceCount` | superclass events this class handles |
| 16 | `interfaces` | interfaces it implements, comma-separated: `"Readable, Writeable"`, or `nullptr` |
| 17–18 | `attributes`, `attributeCount` | `REALattribute[]` |
| 19–20 | `constants`, `constantCount` | `REALconstant[]` |
| 21 | `mFlags` | `0`; set with `SetClassConsoleSafe(&def)` / `SetClassWebOnly(&def)` instead |
| 22–23 | `sharedProperties`, `sharedPropertyCount` | getter/setter take no instance |
| 24–25 | `sharedMethods`, `sharedMethodCount` | functions take no instance |
| 26–27 | `delegates`, `delegateCount` | `REALdelegate[]` |
| 28–29 | `enums`, `enumCount` | `REALenum[]` |

A class that implements an interface must provide every method the interface declares, with the same signature.

### REALinterfaceDefinition *(unverified)*

Register with `REALRegisterInterface(&def)`.

```cpp
static REALmethodDefinition ShapeMethods[] = {
	{ nullptr, REALnoImplementation, "Area() As Double" },
};
static REALinterfaceDefinition ShapeInterface = {
	kCurrentREALControlVersion, "Shape", ShapeMethods, sizeof(ShapeMethods) / sizeof(ShapeMethods[0]),
	nullptr, 0,   // attributes, attributeCount
};
```

To call an interface method on any object that implements it: `REALInterfaceRoutine(obj, "Shape", "Area")` returns
the function; call it with the object first.

### REALcontrol and REALmobileControl

See [Controls.md](Controls.md).
