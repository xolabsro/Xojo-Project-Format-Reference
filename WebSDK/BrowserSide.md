# Browser Side: the JavaScript Class

The JavaScript half of a WebSDK control is a class that extends `XojoWeb.XojoVisualControl` (visual) or
`XojoWeb.XojoControl` (non-visual). All drawing happens here: the server never sends HTML for the control, only its
state as JSON.

The framework is written in ES2015 (ES6). Write plain ES2015 JavaScript; newer syntax may not run in every
supported browser. TypeScript is optional ([JavaScriptDelivery.md](JavaScriptDelivery.md#typescript-optional)).

## The class shape

Declare the class inside your namespace, never in the global scope:

```js
var Acme;
(function (Acme) {
    class Thing extends XojoWeb.XojoVisualControl {
        constructor(id, events) {
            super(id, events);
        }
    }
    Acme.Thing = Thing;
})(Acme || (Acme = {}));
```

The server's `JavaScriptClassName` returns `"Acme.Thing"`, exactly as written here.

## Lifecycle

| Method | Called | You do |
| ------ | ------ | ------ |
| `constructor(id, events)` | once, when the control is created. `id` is the control's id; `events` lists the events the user implemented. | call `super(id, events)` first, then set default values |
| `updateControl(data)` | whenever the server's `Serialize` runs. `data` is a JSON string. | call `super.updateControl(data)` first, then `JSON.parse(data)`, store the values, and call `this.refresh()` |
| `render()` | after `refresh()`, and whenever the control must be redrawn | draw the control from its stored values (visual controls) |
| `close()` | when the parent page or dialog closes | remove listeners, timers and anything else you created |

- `refresh()` asks for a render asynchronously, so several calls in a row cause one render.
- `render()` must not change the control's state. It draws from the stored values only.
- When `render()` is first called, the framework has created an empty `<div>` whose `id` is the control id. Draw
  inside it. The control may move within the DOM, so look the element up each time with `this.DOMElement()`.

A visual control's `render()`:

```js
render() {
    super.render();
    const el = this.DOMElement();
    if (!el) return;              // not attached yet; render runs again when it is
    this.setAttributes();         // position, size and locking

    // build the control's content inside el

    this.applyTooltip(el);        // the user's Tooltip property
    this.applyUserStyle();        // the user's Style, last
}
```

Read `this.enabled` once at the top of `render()` and store it if you use it more than once: it checks every parent
control, so it can be slow.

## Sending events to the server

```js
const params = new XojoWeb.JSONItem();
params.set("value", 3);
this.triggerServerEvent("ValueChanged", params, false);
```

- The name must match the case of what the server's `ExecuteEvent` checks, which is normally the Xojo event name.
- `params` is a `XojoWeb.JSONItem`, or `null`. It arrives in `ExecuteEvent` as a Xojo `JSONItem`.
- With `forced = false`, the event is sent only if the user implemented it (`this.implementsEvent("ValueChanged")`).
  Pass `true` for events whose value the server must always receive. Use it sparingly.
- **Throttle fast events** (mouse moves, scrolling, typing): send only what the app needs, for example the final
  value on `change` rather than every `input`.
- If the control holds changes back instead of sending them right away, override `sendDeferredChanges()` and send
  them there. The framework calls it before actions where the user expects every value to be current, such as the
  default button's `Pressed`.

## XojoControl

Available in every control. These are the documented members: Xojo may change or remove anything else without
notice.

| Member | Does |
| ------ | ---- |
| `constructor(id, events)` | see [Lifecycle](#lifecycle) |
| `updateControl(data)` | see [Lifecycle](#lifecycle) |
| `close()` | override for clean-up |
| `controlID(suffix)` | the control id, plus an optional suffix. Use it to build ids for your own child elements. |
| `DOMElement(suffix)` | the element whose id is the control id plus the optional suffix, or `null` |
| `implementsEvent(name)` | `true` if the user implemented the event at design time or with `AddHandler` |
| `triggerServerEvent(name, params, forced)` | see [Sending events to the server](#sending-events-to-the-server) |
| `sendDeferredChanges()` | override to send held-back changes |
| `scaleFactorChanged()` | override to react to a new `window.devicePixelRatio` (the window moved to another monitor) |
| `panelIndex` | getter: the panel index when the control is on a `WebPagePanel` or `WebTabPanel` |
| `controlName` | getter: the control's name. Valid in debug runs only. |
| `parentControl` | getter: the parent control or view, or `null` |
| `parentLayout` | getter: the page or dialog the control is on, or `null` |

The documentation shows `panelIndex`, `controlName`, `parentControl` and `parentLayout` as methods;
`XojoWebSDK.d.ts` declares them as getters (`this.panelIndex`, no parentheses). Follow the `.d.ts`.

## XojoVisualControl

Extends `XojoControl` with everything a control on a layout needs.

| Member | Does |
| ------ | ---- |
| `render()` | override to draw the control |
| `setAttributes()` | applies the position, size and locking. Call near the top of `render()`. |
| `applyTooltip(el)` | adds the user's tooltip to `el`, which must have an `id`. Call in `render()`. |
| `applyUserStyle(el)` | applies the user's `Style` to `el`, or to the control's main element if omitted. Call last in `render()`. |
| `applyCSSStyle(propertyName, value, inherited, domEl)` | override to control how each part of the user's style is applied |
| `applyColorAndSize(classes, colorPrefix)` | adds the Bootstrap class for the control's `Indicator` to the `classes` array. `colorPrefix` is the Bootstrap prefix, e.g. `"btn-"`. |
| `attachContextualMenu(parentID)` | attaches the user's contextual menu to the element with that id (default: the control) |
| `width()`, `height()` | the size set in the IDE |
| `tabIndex` | getter: the tab index |
| `enabled` | getter: `true` only if the control and all its parents are enabled. Setter: sets the control's own state. `this.mEnabled` holds the control's own value. |
| `visible`, `physicallyVisible` | getters: the `Visible` property; whether the control and all its parents are visible |
| `setEnabled(value)`, `setVisible(value)` | set the state and refresh. Overrides must call `super`. |
| `setCursor(name)` | sets the CSS cursor on the interactive element |
| `interactiveElement()` | the element users interact with: by default the first `button`, `input` or `textarea` inside the control, else the control's element. Override if that's wrong. |
| `focusElement()` | the element that takes focus: by default `interactiveElement()` |
| `setFocus()` | focuses `focusElement()` |
| `gotFocus()`, `lostFocus()` | override to react to focus changes, e.g. send the value on `lostFocus` |
| `resize(width, height)` | sets the size, then calls `resized()` |
| `resized()`, `parentResized()`, `parentPanelShown()` | override to react to size changes and to the parent panel being shown |

In TypeScript, `XojoWebSDK.d.ts` declares a required parameter for `setAttributes`, `applyUserStyle` and
`attachContextualMenu`, while the documentation's JavaScript examples call them without one. In plain JavaScript,
call them as shown above.

## Globals

| Member | Does |
| ------ | ---- |
| `XojoWeb.getNamedControl(controlID)` | the control with that id, or `null` |
| `XojoWeb.DOMParentControl(el)` | walks up from `el` to the control that contains it, or `null` |
| `XojoWeb.getStyleForSelector(selector, attribute)` | the value of a CSS attribute for a selector, searching the page's stylesheets from the last one |
| `XojoWeb.EncodeBase64(text)`, `XojoWeb.DecodeBase64(text)` | Base64 compatible with Xojo's `EncodeBase64(text, 0)` and `DecodeBase64` |
| `XojoWeb.sessionURL()` | the URL for sending requests to the app |
| `XojoWeb.isDebugMode()` | `true` when the app runs from the IDE |
| `XojoWeb.XojoConsole.log(message, always)` | `console.log` that prints only in debug runs, unless `always` is `true` |
| `overrideMetaKey(keyname, target, callback)` | intercepts a Cmd/Ctrl key combination. The browser keeps some, such as Cmd/Ctrl-Q. |
| `XojoWeb.apiVersion` | the version of the framework's JavaScript (`7`, unchanged from 2020r1 to 2026r3.1) |
| `XojoWeb.xojoVersion` | the Xojo version that built the app |

`XojoWeb.JSONItem` has `get(key)`, `set(key, value)`, `lookup(key, defaultValue)`, `length` and `serialize()`.

## Rules

From Xojo's WebSDK documentation. Breaking them can break the user's app, now or after a Xojo update.

- **Only change your own control's DOM.** Never change built-in controls, other developers' controls, `<body>` or
  the page itself. Don't add classes or styles outside your control.
- **Never change the `XojoWeb` namespace or framework**, and don't patch it. Use only the documented API listed on
  this page.
- **Stay inside the position and size the user set** in the IDE. If wrapped content draws outside its box, adjust
  your element so the result still fits that box.
- **Keep memory and traffic small.** Web apps are a single page that stays open, often on phones with slow
  connections.
- jQuery (`$`) and Bootstrap are loaded on every page: jQuery 3.5.1 and Bootstrap 5.3.8 in Xojo 2026r3.1. They can
  change with Xojo versions, so check them in the browser console before relying on either (`$.fn.jquery`,
  `bootstrap.Tooltip.VERSION`), and prefer plain DOM code. Code written for Bootstrap 4, like Xojo's older Bootstrap
  Toast example, may not work.
- For a third-party library, load it from the server through `SessionJavascriptURLs`
  ([JavaScriptDelivery.md](JavaScriptDelivery.md#libraries-and-css)), and tell the user its license.

## A complete class

The readable source of the `kJS` constant in [ServerSide.md](ServerSide.md#a-complete-class-file): a Bootstrap
button that shows the server's `Caption` and sends `Pressed`.

```js
var Acme;
(function (Acme) {
    class Thing extends XojoWeb.XojoVisualControl {
        constructor(id, events) {
            super(id, events);
            this.mCaption = "";
        }
        updateControl(data) {
            super.updateControl(data);
            const js = JSON.parse(data);
            this.mCaption = js.caption;
            this.refresh();
        }
        render() {
            super.render();
            const el = this.DOMElement();
            if (!el) return;
            this.setAttributes();
            const button = document.createElement("button");
            button.className = "btn btn-primary acme-thing";
            button.textContent = this.mCaption;
            button.disabled = !this.enabled;
            button.addEventListener("click", () => this.triggerServerEvent("Pressed", null, false));
            el.replaceChildren(button);
            this.applyTooltip(el);
            this.applyUserStyle();
        }
    }
    Acme.Thing = Thing;
})(Acme || (Acme = {}));
```

## Debugging

Ask the user to open the browser's developer tools while the app runs from the IDE:

- **Console:** JavaScript errors, and `XojoWeb.XojoConsole.log` output. When the control doesn't appear at all,
  check here first: a syntax error in the JavaScript, or a class name that doesn't match `JavaScriptClassName`.
- `XojoWeb.getNamedControl("<ControlID>")` in the console returns your control's instance, so you can inspect its
  stored values.
- **Elements:** the control's `<div>` (id = the control id) and what `render()` put inside it.

## Built-in controls you can extend *(unverified)*

Covered only by `XojoWebSDK.d.ts`; Xojo's examples for them are binary projects.

- **WebListBox cell renderers:** a `ListboxCellRenderer` subclass implements
  `render(controlID, row, data, rowIndex, columnIndex, cell)` and is registered with
  `XojoWeb.XojoListBox.addRenderer("your-identifier", new YourRenderer)` right after the class. A matching Xojo
  `WebListBoxCellRenderer` subclass sends `data`.
- **WebFileUploader:** `XojoUploaderEngine` runs the upload without the built-in UI: `addFile`, `removeFile`,
  `upload`, `abort`, `fileCount`, `totalBytes`, `setTimeout`, with progress reported to a callback.
- **WebProgressWheel** can show any SVG.
