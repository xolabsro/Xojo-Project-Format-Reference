# Controls

A plugin control appears in the IDE's Library and is placed on layouts like a built-in control. Desktop controls use
`REALcontrol` + `REALcontrolBehaviour`; iOS controls use `REALmobileControl` + `REALmobileControlBehaviour`.

## REALcontrol (Desktop)

Register with `REALRegisterControl(&def)`.

| # | Field | Value |
| - | ----- | ----- |
| 1 | `version` | `kCurrentREALControlVersion` |
| 2 | `name` | control class name, e.g. `"DesktopGauge"` |
| 3 | `dataSize` | `sizeof(MyControlData)` |
| 4 | `flags` | control flags (below); always include `REALdesktopControl` |
| 5–6 | `toolbarPICT`, `toolbarDownPICT` | Library icon number, usually `0, 0` (see Library icon) |
| 7–8 | `defaultWidth`, `defaultHeight` | size when dropped on a layout |
| 9–10 | `properties`, `propertyCount` | getters/setters take the control instance |
| 11–12 | `methods`, `methodCount` | functions take the control instance first |
| 13–14 | `events`, `eventCount` | events the Xojo code can handle |
| 15 | `behaviour` | `&MyBehaviour` |
| 16 | `forSystemUse` | `0` |
| 17–18 | `eventInstances`, `eventInstanceCount` | `nullptr, 0` |
| 19 | `interfaces` | comma-separated names, or `nullptr` |
| 20–21 | `attributes`, `attributeCount` | |
| 22–23 | `constants`, `constantCount` | |
| 24–25 | `sharedProperties`, `sharedPropertyCount` | |
| 26–27 | `sharedMethods`, `sharedMethodCount` | |
| 28–29 | `delegates`, `delegateCount` | |
| 30–31 | `enums`, `enumCount` | |

The instance is a `REALcontrolInstance` (the same as `REALobject`). Get its data with
`ControlData(MyDefinition, control, MyControlData, data);`.

### Control flags

| Flag | Meaning |
| ---- | ------- |
| `REALdesktopControl` | It's a `DesktopUIControl` (required for Desktop projects) |
| `REALcontrolAcceptFocus` | Can take keyboard focus |
| `REALcontrolFocusRing` | Draws a focus ring when focused |
| `REALinvisibleControl` | Not visible at run time (like a Timer on a layout) |
| `REALopaqueControl` | Paints its whole area |
| `REALcontrolOwnsCursor` | Sets its own mouse cursor |
| `REALdontEraseBackground` | Windows: don't erase the background before painting (less flicker) |
| `REALcontrolHandlesKeyboardNavigation` | Windows: suppress default arrow-key/mnemonic focus moves |
| `REALcontrolHandlesTabKey` | Receives the Tab key instead of moving focus |
| `REALxamlControl` | WinUI: the control hosts XAML content |
| `REALallowDrawControlInLayoutEditor` | Exposes the `DrawControlInLayoutEditor` event to subclasses (2026r1+) |

### REALcontrolBehaviour

Callbacks in this order. Give `nullptr` for any you don't need, and stop after the last one you use.

| # | Callback | Signature |
| - | -------- | --------- |
| 1 | constructor | `void (REALcontrolInstance)` — set defaults |
| 2 | destructor | `void (REALcontrolInstance)` — release references, destroy native views |
| 3 | redraw | `void (REALcontrolInstance, REALgraphics g)` — paint at run time, control-relative |
| 4 | click | `RBBoolean (REALcontrolInstance, int x, int y, int modifiers)` — return `true` to track drag/up |
| 5 | mouseDrag | `void (REALcontrolInstance, int x, int y)` |
| 6 | mouseUp | `void (REALcontrolInstance, int x, int y)` |
| 7 | gainedFocus | `void (REALcontrolInstance)` |
| 8 | lostFocus | `void (REALcontrolInstance)` |
| 9 | keyDown | `RBBoolean (REALcontrolInstance, int charCode, int keyCode, int modifiers)` |
| 10 | open | `void (REALcontrolInstance)` — the control is on its window; create native views here |
| 11 | close | `void (REALcontrolInstance)` |
| 12 | backgroundIdle | `void (REALcontrolInstance)` |
| 13 | drawOffscreen | `void (REALcontrolInstance, REALgraphics g)` — paint in the IDE layout editor, window-relative |
| 14 | setSpecialBackground | `void (REALcontrolInstance)` |
| 15 | constantChanging | `void (REALcontrolInstance, REALstring)` |
| 16 | droppedNewInstance | `void (REALcontrolInstance)` — dropped on a layout in the IDE |
| 17 | mouseEnter | `void (REALcontrolInstance)` |
| 18 | mouseExit | `void (REALcontrolInstance)` |
| 19 | mouseMove | `void (REALcontrolInstance, int x, int y)` |
| 20 | stateChanged | `void (REALcontrolInstance, uint32_t changedField)` |
| 21 | actionEvent | `void (REALcontrolInstance, uint32_t reserved)` |
| 22 | controlHandleGetter | `void *(REALcontrolInstance)` |
| 23 | mouseWheel | `RBBoolean (REALcontrolInstance, int x, int y, int delta)` |
| 24 | enableMenuItems | `void (REALcontrolInstance)` |
| 25 | menuItemAction | `RBBoolean (REALcontrolInstance, REALobject menuItem)` |
| 26 | controlAcceptFocus | `RBBoolean (REALcontrolInstance)` |
| 27 | keyUp | `RBBoolean (REALcontrolInstance, int charCode, int keyCode, int modifiers)` |
| 28 | redrawWithRects | `void (REALcontrolInstance, REALgraphics g, const Rect *rects, int count)` — preferred over redraw when set |
| 29 | unfilteredKeyDown | `RBBoolean (REALcontrolInstance, int keyCode, int modifiers)` |
| 30 | scaleFactorChanged | `void (REALcontrolInstance, double newScaleFactor)` |

`changedField` is a combination of `kActivationChanged`, `kEnabledChanged`, `kVisibilityChanged`, `kBoundsChanged`,
`kThemeChanged`.

### Drawing

- Get the bounds with `REALGetControlBounds(control, &r)` (`Rect` has `top, left, bottom, right`). They are
  **window-relative**. In `redraw`, subtract `left`/`top` to draw control-relative. In `drawOffscreen` (the layout
  editor), draw window-relative as they are.
- Draw through the `Graphics` object's Xojo methods, loaded with `REALLoadObjectMethod`. The declaration string
  must match the framework method's parameter types:

```cpp
typedef void (*FillRectFn)(REALgraphics, double, double, double, double);
static FillRectFn fillRect = (FillRectFn)REALLoadObjectMethod((REALobject)g,
	"FillRectangle(x As Double, y As Double, width As Double, height As Double)");
REALSetPropValueColor((REALobject)g, "DrawingColor", data->color);
if (fillRect) fillRect(g, 0, 0, width, height);
```

- Ask for a repaint from your own code with the control's `Refresh` method:
  `REALLoadObjectMethod((REALobject)control, "Refresh(immediately As Boolean = False)")`, called as
  `fn(control, false)`.
- `REALIsDarkMode()` tells you whether the app is in dark mode; repaint on `kThemeChanged`.

### Hosting a native OS control

Windows works differently from macOS and Linux.

**Windows (Win32).** The control always has its own backing window; you create your control as its child.

- Create it in **open** (the handle doesn't exist before): get the parent `HWND` from the control's `Handle`
  property, `RBInteger h; REALGetPropValueInteger(control, "Handle", &h);`, then
  `CreateWindowW(…, WS_CHILD | WS_VISIBLE, 0, 0, width, height, (HWND)h, …)`. Positions are relative to that parent.
- Resize it in **stateChanged** on `kBoundsChanged`, enable/disable it on `kEnabledChanged`
  (`REALGetPropValueBoolean(control, "Enabled", &enabled)`), destroy it in **destructor**.

**macOS (Cocoa) and Linux (GTK).** Create the native view or widget in the **constructor**, keep it in the control
data, and return it from **controlHandleGetter** (callback 22). The framework embeds it and handles bounds and
visibility.

- macOS: return an `NSView *`. Release it in the destructor. To forward mouse and keyboard events to Xojo, use an
  `NSView` subclass with the `REAL_VIEW_MIXINS(control)` macro from `rb_plugin_cocoa.h`. Compile those files as
  Objective-C++ (`.mm`), add `GlueCode/PluginMainCocoa.mm`, and link `-framework Cocoa`.
- Linux: return a `GtkWidget *` (GTK 3). The framework destroys the widget; don't destroy it yourself.

In every case, paint a placeholder in **drawOffscreen** so the control is visible in the layout editor.

**Windows (WinUI).** XAML content needs C++/WinRT and XAML Islands, the `REALxamlControl` flag, and a separate build
whose DLL has the same file name as the Win32 one. `REALUsesWinUI()` says whether the app uses the WinUI framework.
This is advanced: build and test the Win32 version first.

## REALmobileControl (iOS)

Register with `REALRegisterMobileControl(&def)`. The fields are the same as `REALcontrol` except:

- `flags`: don't use `REALdesktopControl`;
- `behaviour` points to a `REALmobileControlBehaviour`;
- get data with `MobileControlData(MyDefinition, control, MyControlData, data);`.

### REALmobileControlBehaviour

| # | Callback | Signature |
| - | -------- | --------- |
| 1 | constructor | `void (REALcontrolInstance)` |
| 2 | destructor | `void (REALcontrolInstance)` |
| 3 | controlHandleGetter | `void *(REALcontrolInstance)` |
| 4 | redraw | `void (REALcontrolInstance, REALobject g)` |
| 5 | pointerDown | `RBBoolean (REALcontrolInstance, REALobject position, REALobject pointerInfoArray)` |
| 6 | pointerUp | same as pointerDown |
| 7 | pointerDrag | same as pointerDown |

Register mobile controls only in iOS builds:

```cpp
void PluginEntry(void)
{
#if TARGET_OS_IPHONE
	REALRegisterMobileControl(&MyMobileDefinition);
#else
	REALRegisterControl(&MyDesktopDefinition);
#endif
}
```

## Library icon

Icons go in the package's `IDE Resources/Controls Palette` folder, named after the `toolbarPICT` number. With `0`:

```text
IDE Resources/Controls Palette/
├── 0-library-small.tiff
├── 0-library-large.tiff
├── Mac Cocoa/   0-library-small.tiff, 0-library-large.tiff
└── Windows/     0-library-small.tiff, 0-library-large.tiff
```

Icons are optional. Without them the IDE shows a generic icon.
