# Drawing in the Layout Editor

A `WebSDKUIControl` can draw itself in the IDE's layout editor, so users see roughly what it will look like instead
of an empty box. Implement the `DrawControlInLayoutEditor(g As Graphics)` event, even if it draws nothing, so users
of the control don't see the event.

[Example.md](Example.md) draws its stars this way (tested). The event's code doesn't run in your app. The IDE runs it as XojoScript with an API 2 `Graphics` context, so:

- draw as in a `DesktopCanvas.Paint` event: `g.DrawingColor`, `g.FillRectangle`, `g.DrawRoundRectangle`,
  `g.DrawText`, `g.Width`, `g.Height`…;
- you can't call the class's own methods or read its properties directly. Read them with the helpers below;
- if the code doesn't compile or fails while running, the IDE draws an amber warning icon on the control and sends
  the errors to the **Messages** pane. Ask the user to check it.

```xojo
Sub DrawControlInLayoutEditor(g As Graphics)
  g.DrawingColor = CSSColorValue(".btn-primary", "background-color", &c0D6EFD)
  g.FillRoundRectangle(0, 0, g.Width, g.Height, 8, 8)
  g.DrawingColor = &cFFFFFF
  g.DrawText(StringProperty("Caption"), 8, g.Height / 2 + 5)
End Sub
```

## Helpers

| Helper | Returns |
| ------ | ------- |
| `BooleanProperty(name As String) As Boolean` | the current value of a property of the control, as set in the Inspector |
| `ColorProperty(name As String) As Color` | |
| `DoubleProperty(name As String) As Double` | |
| `IntegerProperty(name As String) As Integer` | |
| `PictureProperty(name As String) As Picture` | |
| `StringProperty(name As String) As String` | |
| `ConstantValue(name As String) As String` | the value of a constant of the class |
| `PictureConstant(name As String) As Picture` | a constant of the class as a picture |
| `TextColor() As Color`, `FillColor() As Color` | the IDE platform's text and fill colors |
| `IsDarkMode() As Boolean` | whether the system is in dark mode |
| `ColorGroup.NamedColor(name As String) As ColorGroup` | an OS named color, macOS only; names are case-sensitive (`"secondaryLabelColor"`) |
| `FontCount() As Integer`, `FontAt(index As Integer) As String` | the system's fonts |
| `CSSColorValue(selector As String, propertyName As String, fallbackColor As Color) As Color` | a color from the project's current Bootstrap theme; reads `#RRGGBB`, `rgb()` and `rgba()` |
| `CSSPixelValue(selector As String, propertyName As String, fallbackValue As Double) As Double` | a size from the theme in pixels, converting `mm`, `pt` and `rem` (rem × the body font size) |
| `CSSStringValue(selector As String, propertyName As String) As String` | the raw value from the theme |
| `DebugLog(msg As String)` | sends `msg` to the Messages pane |

- The `CSS…` helpers make the drawing match the project's theme. Use them for colors and sizes rather than fixed
  values.

## Icons

A constant with one of these names in the class replaces the control's default icon:

- `NavigatorIcon`: the icon in the Navigator;
- `LibraryIcon`: the icon in the Library.

Its value is Base64 icon data made with Xojo's WebSDKIconConverter tool. That tool and the data format aren't in this
kit *(unverified)*: leave the icons out unless the user gives you the converted data.
