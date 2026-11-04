# Complete Example: a Rating Control

`Acme_Rating` is a visual WebSDK control: a row of stars the user clicks to rate something. It shows every part of
a WebSDK control working together:

- state from Xojo to the browser: `Value` and `MaxValue` go through `Serialize` to the JavaScript's `updateControl`;
- events from the browser to Xojo: a click sends `triggerServerEvent`, and `ExecuteEvent` raises `ValueChanged`;
- JavaScript and CSS served from string constants through shared `WebFile`s;
- the user's tooltip, enabled state and style;
- drawing in the IDE's layout editor.

Tested on Windows with Xojo 2026r3.1. The control compiles, draws 5 stars with 3 lit, updates the label when a star
is clicked, clears when **Reset** is pressed, shows its tooltip, and draws gray dots in the layout editor.

What it adds to the project:

```text
Class Acme_Rating (Inherits WebSDKUIControl)
  Property Value As Integer          // 0 to MaxValue; setting it updates the browser
  Property MaxValue As Integer       // number of stars, at least 1 (default 5)
  Event ValueChanged(value As Integer)
```

Clicking a star sets the value to that star; clicking the current star clears the rating to 0.

## Source files (working folder)

Keep the readable sources outside the project, and regenerate the constants from them after every change
([JavaScriptDelivery.md](JavaScriptDelivery.md#string-constants-in-a-xojo_code-file)).

### acme-rating.js

```js
var Acme;
(function (Acme) {
    class Rating extends XojoWeb.XojoVisualControl {
        constructor(id, events) {
            super(id, events);
            this.mValue = 0;
            this.mMax = 5;
        }
        updateControl(data) {
            super.updateControl(data);
            const js = JSON.parse(data);
            // Accept whole numbers only (the backslash also tests the constant escaping)
            if (/^\d+$/.test(String(js.value))) this.mValue = js.value;
            if (/^\d+$/.test(String(js.max))) this.mMax = js.max;
            this.refresh();
        }
        render() {
            super.render();
            const el = this.DOMElement();
            if (!el) return;
            this.setAttributes();
            const enabled = this.enabled;
            const wrapper = document.createElement("div");
            wrapper.className = "acme-rating";
            for (let i = 1; i <= this.mMax; i++) {
                const star = document.createElement("button");
                star.type = "button";
                star.innerHTML = "&#9733;";
                star.setAttribute("aria-label", "Rate " + i + " of " + this.mMax);
                if (i <= this.mValue) star.classList.add("on");
                star.disabled = !enabled;
                star.addEventListener("click", () => this.select(i));
                wrapper.appendChild(star);
            }
            el.replaceChildren(wrapper);
            this.applyTooltip(el);
            this.applyUserStyle();
        }
        select(value) {
            if (value === this.mValue) value = 0;    // clicking the current star clears the rating
            this.mValue = value;
            this.refresh();
            const params = new XojoWeb.JSONItem();
            params.set("value", value);
            this.triggerServerEvent("ValueChanged", params, true);
        }
    }
    Acme.Rating = Rating;
})(Acme || (Acme = {}));
```

The regular expression also tests the constant escaping: `\d` is written as `\\d` in the constant.

### acme-rating.css

```css
.acme-rating { display: flex; height: 100%; align-items: center; }
.acme-rating button { background: none; border: 0; padding: 0 2px; font-size: 26px; line-height: 1; color: #c8c8c8; cursor: pointer; }
.acme-rating button.on { color: #f5b301; }
.acme-rating button:disabled { cursor: default; opacity: 0.5; }
```

## Acme_Rating.xojo_code

The class file exactly as tested. The two constant lines are generated from the source files above.

```text
#tag Class
Protected Class Acme_Rating
Inherits WebSDKUIControl
	#tag Event
		Sub DrawControlInLayoutEditor(g As Graphics)
		  Var maxValue As Integer = IntegerProperty("MaxValue")
		  If maxValue < 1 Then maxValue = 5
		  Var value As Integer = IntegerProperty("Value")
		  Var size As Double = Min(g.Height - 8, 26)

		  For i As Integer = 1 To maxValue
		    If i <= value Then
		      g.DrawingColor = &cF5B301
		    Else
		      g.DrawingColor = &cC8C8C8
		    End If
		    g.FillOval((i - 1) * (size + 4) + 2, (g.Height - size) / 2, size, size)
		  Next
		End Sub
	#tag EndEvent

	#tag Event
		Function ExecuteEvent(name As String, parameters As JSONItem) As Boolean
		  Select Case name
		  Case "ValueChanged"
		    Var newValue As Integer = parameters.Lookup("value", 0)
		    mValue = Max(0, Min(newValue, mMaxValue))
		    RaiseEvent ValueChanged(mValue)
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
		  Return "Acme.Rating"
		End Function
	#tag EndEvent

	#tag Event
		Sub Serialize(js As JSONItem)
		  js.Value("value") = mValue
		  js.Value("max") = mMaxValue
		End Sub
	#tag EndEvent

	#tag Event
		Function SessionCSSURLs(session As WebSession) As String()
		  If SharedCSSFile = Nil Then
		    SharedCSSFile = New WebFile
		    SharedCSSFile.Filename = "acme-rating.css"
		    SharedCSSFile.MIMEType = "text/css"
		    SharedCSSFile.Data = kCSS
		    SharedCSSFile.Session = Nil // one file for every session
		  End If

		  Return Array(SharedCSSFile.URL)
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
		    SharedJSFile.Filename = "acme-rating.js"
		    SharedJSFile.MIMEType = "text/javascript"
		    SharedJSFile.Data = kJS
		    SharedJSFile.Session = Nil // one file for every session
		  End If

		  Return Array(SharedJSFile.URL)
		End Function
	#tag EndEvent


	#tag Hook, Flags = &h0
		Event ValueChanged(value As Integer)
	#tag EndHook


	#tag ComputedProperty, Flags = &h0
		#tag Getter
			Get
			  Return mMaxValue
			End Get
		#tag EndGetter
		#tag Setter
			Set
			  mMaxValue = Max(1, value)
			  mValue = Min(mValue, mMaxValue)
			  UpdateControl
			End Set
		#tag EndSetter
		MaxValue As Integer
	#tag EndComputedProperty

	#tag ComputedProperty, Flags = &h0
		#tag Getter
			Get
			  Return mValue
			End Get
		#tag EndGetter
		#tag Setter
			Set
			  mValue = Max(0, Min(value, mMaxValue))
			  UpdateControl
			End Set
		#tag EndSetter
		Value As Integer
	#tag EndComputedProperty

	#tag Property, Flags = &h21
		Private mMaxValue As Integer = 5
	#tag EndProperty

	#tag Property, Flags = &h21
		Private mValue As Integer = 0
	#tag EndProperty

	#tag Property, Flags = &h21
		Private Shared SharedCSSFile As WebFile
	#tag EndProperty

	#tag Property, Flags = &h21
		Private Shared SharedJSFile As WebFile
	#tag EndProperty


	#tag Constant, Name = kCSS, Type = String, Dynamic = False, Default = \".acme-rating { display: flex; height: 100%; align-items: center; }\n.acme-rating button { background: none; border: 0; padding: 0 2px; font-size: 26px; line-height: 1; color: #c8c8c8; cursor: pointer; }\n.acme-rating button.on { color: #f5b301; }\n.acme-rating button:disabled { cursor: default; opacity: 0.5; }\n", Scope = Private
	#tag EndConstant

	#tag Constant, Name = kJS, Type = String, Dynamic = False, Default = \"var Acme;\n(function (Acme) {\n    class Rating extends XojoWeb.XojoVisualControl {\n        constructor(id\x2C events) {\n            super(id\x2C events);\n            this.mValue \x3D 0;\n            this.mMax \x3D 5;\n        }\n        updateControl(data) {\n            super.updateControl(data);\n            const js \x3D JSON.parse(data);\n            // Accept whole numbers only (the backslash also tests the constant escaping)\n            if (/^\\d+$/.test(String(js.value))) this.mValue \x3D js.value;\n            if (/^\\d+$/.test(String(js.max))) this.mMax \x3D js.max;\n            this.refresh();\n        }\n        render() {\n            super.render();\n            const el \x3D this.DOMElement();\n            if (!el) return;\n            this.setAttributes();\n            const enabled \x3D this.enabled;\n            const wrapper \x3D document.createElement(\"div\");\n            wrapper.className \x3D \"acme-rating\";\n            for (let i \x3D 1; i <\x3D this.mMax; i++) {\n                const star \x3D document.createElement(\"button\");\n                star.type \x3D \"button\";\n                star.innerHTML \x3D \"&#9733;\";\n                star.setAttribute(\"aria-label\"\x2C \"Rate \" + i + \" of \" + this.mMax);\n                if (i <\x3D this.mValue) star.classList.add(\"on\");\n                star.disabled \x3D !enabled;\n                star.addEventListener(\"click\"\x2C () \x3D> this.select(i));\n                wrapper.appendChild(star);\n            }\n            el.replaceChildren(wrapper);\n            this.applyTooltip(el);\n            this.applyUserStyle();\n        }\n        select(value) {\n            if (value \x3D\x3D\x3D this.mValue) value \x3D 0;    // clicking the current star clears the rating\n            this.mValue \x3D value;\n            this.refresh();\n            const params \x3D new XojoWeb.JSONItem();\n            params.set(\"value\"\x2C value);\n            this.triggerServerEvent(\"ValueChanged\"\x2C params\x2C true);\n        }\n    }\n    Acme.Rating \x3D Rating;\n})(Acme || (Acme \x3D {}));\n", Scope = Private
	#tag EndConstant


End Class
#tag EndClass
```

## The project manifest

One line, after the existing items. The ItemID is new and unique; the other items keep theirs:

```text
Class=Acme_Rating;Acme_Rating.xojo_code;&h000000004A7C2E91;&h0000000000000000;false
```

## WebPage1.xojo_code

The page from a new Web project, with three controls added inside `Begin WebPage WebPage1`, after its own
properties, and their event handlers after `#tag EndWindowCode`:

```text
#tag WebPage
Begin WebPage WebPage1
   AllowTabOrderWrap=   True
   Compatibility   =   ""
   ControlCount    =   0
   ControlID       =   ""
   CSSClasses      =   ""
   Enabled         =   False
   Height          =   400
   ImplicitInstance=   True
   Index           =   -2147483648
   Indicator       =   0
   IsImplicitInstance=   False
   LayoutDirection =   0
   LayoutType      =   0
   Left            =   0
   LockBottom      =   False
   LockHorizontal  =   False
   LockLeft        =   True
   LockRight       =   False
   LockTop         =   True
   LockVertical    =   False
   MinimumHeight   =   400
   MinimumWidth    =   600
   PanelIndex      =   0
   ScaleFactor     =   0.0
   TabIndex        =   0
   Title           =   "Untitled"
   Top             =   0
   Visible         =   True
   Width           =   600
   _ImplicitInstance=   False
   _mDesignHeight  =   0
   _mDesignWidth   =   0
   _mName          =   ""
   _mPanelIndex    =   -1
   Begin Acme_Rating Rating1
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
      TabIndex        =   0
      TabStop         =   True
      Tooltip         =   "Rate this"
      Top             =   20
      Visible         =   True
      Width           =   160
      _mPanelIndex    =   -1
   End
   Begin WebLabel ValueLabel
      Bold            =   False
      ControlID       =   ""
      CSSClasses      =   ""
      Enabled         =   True
      FontName        =   ""
      FontSize        =   0.0
      Height          =   38
      HTMLElement     =   0
      Index           =   -2147483648
      Indicator       =   ""
      Italic          =   False
      Left            =   200
      LockBottom      =   False
      LockedInPosition=   False
      LockHorizontal  =   False
      LockLeft        =   True
      LockRight       =   False
      LockTop         =   True
      LockVertical    =   False
      Multiline       =   False
      PanelIndex      =   0
      Scope           =   0
      TabIndex        =   1
      TabStop         =   True
      Text            =   "Rating: 0"
      TextAlignment   =   0
      TextColor       =   &c000000FF
      Tooltip         =   ""
      Top             =   20
      Underline       =   False
      Visible         =   True
      Width           =   200
      _mPanelIndex    =   -1
   End
   Begin WebButton ResetButton
      AllowAutoDisable=   False
      Cancel          =   False
      Caption         =   "Reset"
      ControlID       =   ""
      CSSClasses      =   ""
      Default         =   False
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
      Outlined        =   False
      PanelIndex      =   0
      Scope           =   0
      TabIndex        =   2
      TabStop         =   True
      Tooltip         =   ""
      Top             =   78
      Visible         =   True
      Width           =   100
      _mPanelIndex    =   -1
   End
End
#tag EndWebPage

#tag WindowCode
#tag EndWindowCode

#tag Events Rating1
	#tag Event
		Sub Opening()
		  Me.MaxValue = 5
		  Me.Value = 3
		  ValueLabel.Text = "Rating: " + Me.Value.ToString
		End Sub
	#tag EndEvent
	#tag Event
		Sub ValueChanged(value As Integer)
		  ValueLabel.Text = "Rating: " + value.ToString
		End Sub
	#tag EndEvent
#tag EndEvents
#tag Events ResetButton
	#tag Event
		Sub Pressed()
		  Rating1.Value = 0
		  ValueLabel.Text = "Rating: 0"
		End Sub
	#tag EndEvent
#tag EndEvents
```

`Value` and `MaxValue` aren't Inspector-visible, so the page sets them in `Rating1.Opening` instead of in the
`Begin Acme_Rating` block.

## Checking it

1. Run the project. With agentic mode on, follow the project's `XOJO-CODING.md`; otherwise ask the user to run it.
2. In the browser (`http://127.0.0.1:8080`):
   - 5 stars, 3 of them lit, and the label reads `Rating: 3`;
   - clicking the fifth star lights all 5 and the label reads `Rating: 5` (browser → Xojo);
   - **Reset** clears the stars and the label reads `Rating: 0` (Xojo → browser);
   - hovering the stars shows the tooltip `Rate this`;
   - the console shows no errors from `acme-rating.js`.
3. In the IDE, `Rating1` draws as a row of gray dots in the layout editor, and the Messages pane stays empty.
