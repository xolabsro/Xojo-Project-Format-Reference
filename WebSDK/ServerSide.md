# Server Side: the Xojo Class

The Xojo half of a WebSDK control is an ordinary class in the project. It tells the framework which JavaScript to
load, sends the control's state to the browser, and receives the browser's events.

## Choosing the super

| Super | For | JavaScript super | Like |
| ----- | --- | ---------------- | ---- |
| `WebSDKUIControl` | a control the user sees, with a position and size on the layout | `XojoWeb.XojoVisualControl` | `WebButton` |
| `WebSDKControl` | code that runs in the browser with no UI of its own | `XojoWeb.XojoControl` | `WebTimer` |

Most controls are visual. `WebSDKUIControl` inherits from `WebUIControl`, so it has `Left`, `Top`, `Width`,
`Height`, the locks, `Enabled`, `Visible`, `Tooltip`, `Indicator`, `Style`, `CSSClasses` and the `WebUIControl`
events (`Opening`, `Shown`…). `WebSDKControl` inherits from `WebControl` only.

## The SDK events

Both supers add these events. Every one of them is implemented in your class, never by the user (see
[Hide the SDK events](#hide-the-sdk-events)). The signatures are written exactly as the IDE writes them.

| Event | Fires | Return |
| ----- | ----- | ------ |
| `Function JavaScriptClassName() As String` | when the control is first created | the full JavaScript class name, case-sensitive: `"Acme.Rating"` |
| `Function SessionJavascriptURLs(session As WebSession) As String()` | once per new session | URLs of the JavaScript files the control needs: its own class and any library |
| `Function SessionCSSURLs(session As WebSession) As String()` | once per new session (`WebSDKUIControl` only) | URLs of the stylesheets the control needs |
| `Function SessionHead(session As WebSession) As String` | once per new session | HTML to add to the page's `<head>`, or `""` |
| `Sub Serialize(js As JSONItem)` | when the control's first state is sent, when a `WebControl` property (`Enabled`, `Visible`…) changes, and after `UpdateControl` or `UpdateBrowser` | nothing: fill `js` with the state the browser needs |
| `Function ExecuteEvent(name As String, parameters As JSONItem) As Boolean` | when the JavaScript calls `triggerServerEvent` | `True` if you handled the event |
| `Function HandleRequest(request As WebRequest, response As WebResponse) As Boolean` | for HTTP requests addressed to this control | `True` if you handled it |
| `Sub DrawControlInLayoutEditor(g As Graphics)` | in the IDE's layout editor (`WebSDKUIControl` only) | nothing: draw the control ([LayoutEditor.md](LayoutEditor.md)) |

- **Session setup events run on a throw-away instance.** `SessionJavascriptURLs`, `SessionCSSURLs` and
  `SessionHead` fire on an instance that isn't on any page and is destroyed right after `Session.PrepareSession`.
  Don't use instance properties in them; anything that must persist goes in a `Shared` property.
- Xojo tries not to send the same URL twice to one session, but return the same URL every time anyway, from a
  shared `WebFile` ([JavaScriptDelivery.md](JavaScriptDelivery.md)).
- The documentation gives `SessionJavaScriptURLs` returning `String` in one place; the examples and the IDE use
  `String()`. The IDE spells the event `SessionJavascriptURLs` (lowercase `s`).

### HandleRequest *(unverified)*

`HandleRequest` works like `WebApplication.HandleURL`, but only for requests addressed to one control. The
documentation and the examples disagree on the URL: `/sdk/<ControlID>` (documentation), and
`/<session identifier>/sdk/<ControlID>/<path>` or `/<session identifier>/data/<ControlID>/<path>` (comments in the
two example controls). Test the URL before relying on it. Most controls don't need this event: send data through
`Serialize` and `triggerServerEvent` instead.

## Methods and constants

| Member | Does |
| ------ | ---- |
| `UpdateControl(sendImmediately As Boolean = False)` | marks the control as changed so `Serialize` runs and the new state goes to the browser. With `False` (the default), the update waits until the current event finishes and repeated calls are combined; `True` sends it now. |
| `UpdateBrowser()` | marks the control as changed and sends the update immediately |
| `ControlID As String` | the control's id, the same as the browser-side `controlID`; the id of its DOM element |
| `WebSDKControl.BrowserCompatibility(currentSession As WebSession, key As String) As Variant` | shared; a browser feature test result from Modernizr, e.g. `WebSDKControl.BrowserCompatibility(Session, "xhr2")`. To list the keys, type `Modernizr` in the browser console of a running app. |
| `APIVersion As Integer` | constant: the WebSDK API version (`7`, unchanged from 2020r1 to 2026r3.1) |

A `WebSDKUIControl` subclass can replace its IDE icons with `NavigatorIcon` and `LibraryIcon` constants
([LayoutEditor.md](LayoutEditor.md#icons)).

## State: Xojo → browser

Keep the state in private properties, expose it with computed properties, and call `UpdateControl` from every
setter. `Serialize` writes the **whole** state each time; the browser's `updateControl` reads it.

```xojo
// Setter of the computed property Caption
mCaption = value
UpdateControl
```

```xojo
Sub Serialize(js As JSONItem)
  js.Value("caption") = mCaption
End Sub
```

Send as little as possible: short keys, numbers instead of strings (`1`/`0` rather than `"true"`/`"false"`), and
only what the browser needs to draw.

## Events: browser → Xojo

1. Define the event users will implement, with a `Hook`: `Event ValueChanged(value As Integer)`.
2. In the JavaScript, call `this.triggerServerEvent("ValueChanged", params, forced)`
   ([BrowserSide.md](BrowserSide.md#sending-events-to-the-server)).
3. In `ExecuteEvent`, read the parameters, update your properties and raise the event:

```xojo
Function ExecuteEvent(name As String, parameters As JSONItem) As Boolean
  Select Case name
  Case "ValueChanged"
    mValue = parameters.Value("value")
    RaiseEvent ValueChanged(mValue)
    Return True
  End Select
End Function
```

The browser sends an event only if the user implemented it on the instance, unless the JavaScript passes
`forced = true`. Force events that carry a value the server must always know (the new value of an input), even
when nobody handles the event.

Text from the browser can contain any characters. To send arbitrary text safely, Base64-encode it in the browser
(`XojoWeb.EncodeBase64`) and decode it in Xojo (`DecodeBase64`). In the other direction, use
`EncodeBase64(text, 0)`: the line length must be `0` ([BrowserSide.md](BrowserSide.md#globals)).

## Hide the SDK events

Users of the control shouldn't see the SDK events. Implement **every** SDK event in your class, with a comment
where it has nothing to do, so it disappears from the user's instances:

```xojo
Function SessionHead(session As WebSession) As String
  // Not used. Implemented so it's hidden from users of the control.
End Function
```

For the `String()` events, return an empty array explicitly (`Var urls() As String` / `Return urls`).

## A complete class file

`Acme_Thing.xojo_code`: a visual control that shows a button with a caption and raises `Pressed` when it's clicked.
Members are in the order the IDE writes them: events alphabetically, then hooks, properties and constants.

```text
#tag Class
Protected Class Acme_Thing
Inherits WebSDKUIControl
	#tag Event
		Sub DrawControlInLayoutEditor(g As Graphics)
		  g.DrawingColor = &c0D6EFD
		  g.FillRoundRectangle(0, 0, g.Width, g.Height, 8, 8)
		  g.DrawingColor = &cFFFFFF
		  g.DrawText(StringProperty("Caption"), 8, g.Height / 2 + 5)
		End Sub
	#tag EndEvent

	#tag Event
		Function ExecuteEvent(name As String, parameters As JSONItem) As Boolean
		  Select Case name
		  Case "Pressed"
		    RaiseEvent Pressed
		    Return True
		  End Select
		End Function
	#tag EndEvent

	#tag Event
		Function HandleRequest(request As WebRequest, response As WebResponse) As Boolean
		  // Not used. Implemented so it's hidden from users of the control.
		End Function
	#tag EndEvent

	#tag Event
		Function JavaScriptClassName() As String
		  Return "Acme.Thing"
		End Function
	#tag EndEvent

	#tag Event
		Sub Serialize(js As JSONItem)
		  js.Value("caption") = mCaption
		End Sub
	#tag EndEvent

	#tag Event
		Function SessionCSSURLs(session As WebSession) As String()
		  // Not used. Implemented so it's hidden from users of the control.
		  Var urls() As String
		  Return urls
		End Function
	#tag EndEvent

	#tag Event
		Function SessionHead(session As WebSession) As String
		  // Not used. Implemented so it's hidden from users of the control.
		End Function
	#tag EndEvent

	#tag Event
		Function SessionJavascriptURLs(session As WebSession) As String()
		  If SharedJSFile = Nil Then
		    SharedJSFile = New WebFile
		    SharedJSFile.Filename = "acme-thing.js"
		    SharedJSFile.MIMEType = "text/javascript"
		    SharedJSFile.Data = kJS
		    SharedJSFile.Session = Nil // one file for every session
		  End If
		  
		  Return Array(SharedJSFile.URL)
		End Function
	#tag EndEvent


	#tag Hook, Flags = &h0
		Event Pressed()
	#tag EndHook


	#tag ComputedProperty, Flags = &h0
		#tag Getter
			Get
			  Return mCaption
			End Get
		#tag EndGetter
		#tag Setter
			Set
			  mCaption = value
			  UpdateControl
			End Set
		#tag EndSetter
		Caption As String
	#tag EndComputedProperty

	#tag Property, Flags = &h21
		Private mCaption As String = "Untitled"
	#tag EndProperty

	#tag Property, Flags = &h21
		Private Shared SharedJSFile As WebFile
	#tag EndProperty


	#tag Constant, Name = kJS, Type = String, Dynamic = False, Default = \"var Acme;\n(function (Acme) {\n    class Thing extends XojoWeb.XojoVisualControl {\n        constructor(id\x2C events) {\n            super(id\x2C events);\n            this.mCaption \x3D \"\";\n        }\n        updateControl(data) {\n            super.updateControl(data);\n            const js \x3D JSON.parse(data);\n            this.mCaption \x3D js.caption;\n            this.refresh();\n        }\n        render() {\n            super.render();\n            const el \x3D this.DOMElement();\n            if (!el) return;\n            this.setAttributes();\n            const button \x3D document.createElement(\"button\");\n            button.className \x3D \"btn btn-primary acme-thing\";\n            button.textContent \x3D this.mCaption;\n            button.disabled \x3D !this.enabled;\n            button.addEventListener(\"click\"\x2C () \x3D> this.triggerServerEvent(\"Pressed\"\x2C null\x2C false));\n            el.replaceChildren(button);\n            this.applyTooltip(el);\n            this.applyUserStyle();\n        }\n    }\n    Acme.Thing \x3D Thing;\n})(Acme || (Acme \x3D {}));\n", Scope = Private
	#tag EndConstant


End Class
#tag EndClass
```

- `kJS` holds the JavaScript, escaped as the IDE writes string constants. Its readable source is in
  [BrowserSide.md](BrowserSide.md#a-complete-class), and the escaping rules are in
  [JavaScriptDelivery.md](JavaScriptDelivery.md).
- Don't write a `#tag ViewBehavior` block. The kit's [README](../README.md#common-to-all-project-types) explains
  why. A class file without one compiles and works (tested); the IDE adds the block when it next saves the file.
- To show a property in the Inspector, ask the user to right-click the control's class in the Navigator, choose
  **Inspector Behavior** and tick the property. Until then, users set it in code, for example in the control's
  `Opening` event on the page.

## Using the control in the project

**Manifest:** add one line to the `.xojo_project`, with a new unique ItemID (format in the kit's
[README](../README.md#common-to-all-project-types)):

```text
Class=Acme_Thing;Acme_Thing.xojo_code;&h000000001A2B3C4D;&h0000000000000000;false
```

**On a page:** add a `Begin … End` block inside the page's `Begin WebPage` block, after the existing controls. A
visual control takes the same properties as other web controls. These come from an IDE-written page:

```text
   Begin Acme_Thing Thing1
      ControlID       =   ""
      CSSClasses      =   ""
      Enabled         =   True
      Height          =   38
      Index           =   -2147483648
      Indicator       =   0
      Left            =   20
      LockBottom      =   False
      LockedInPosition=   False
      LockHorizontal  =   False
      LockLeft        =   True
      LockRight       =   False
      LockTop         =   True
      LockVertical    =   False
      PanelIndex      =   0
      Scope           =   0
      TabIndex        =   1
      TabStop         =   True
      Tooltip         =   ""
      Top             =   20
      Visible         =   True
      Width           =   120
      _mPanelIndex    =   -1
   End
```

A non-visual control has no position, size or appearance properties:

```text
   Begin Acme_Helper Helper1
      ControlID       =   ""
      Enabled         =   True
      Index           =   -2147483648
      LockedInPosition=   False
      PanelIndex      =   0
      Scope           =   0
      _mPanelIndex    =   -1
   End
```

- Give `TabIndex` the next free number on the page.
- Custom properties go in the block only once they're Inspector-visible (see above). Otherwise set them in code.
- Event handlers go in a `#tag Events` block after `#tag EndWindowCode`, as for any control. Check the format in
  [WebAppStructure/WebPage1.xojo_code](../WebAppStructure/WebPage1.xojo_code):

```text
#tag Events Thing1
	#tag Event
		Sub Opening()
		  Me.Caption = "Save"
		End Sub
	#tag EndEvent
	#tag Event
		Sub Pressed()
		  MessageBox("Pressed")
		End Sub
	#tag EndEvent
#tag EndEvents
```
