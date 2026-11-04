# Delivering the JavaScript and CSS

The browser gets a control's JavaScript once per session, from the URLs that `SessionJavascriptURLs` returns. The
usual way, used by all of Xojo's examples: keep the JavaScript in a private string constant of the class and serve
it from one shared `WebFile`.

## The shared WebFile

```xojo
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
```

- `SharedJSFile` is a `Private Shared` property, so the file is created once for the whole app.
- `Session = Nil` makes the file available to every session. Without it the file belongs to the first session only.
- Give each control's file its own name: namespace plus control (`acme-thing.js`).

## String constants in a .xojo_code file

The IDE writes a string constant on one line, inside the class, with a `#tag EndConstant` line after it:

```text
	#tag Constant, Name = kJS, Type = String, Dynamic = False, Default = \"<escaped JavaScript>", Scope = Private
	#tag EndConstant
```

The value opens with `\"` and closes with a plain `"`. Inside it, these characters are escaped and every other
character is written as is:

| Character | Written as |
| --------- | ---------- |
| newline | `\n` |
| tab | `\t` |
| `"` | `\"` |
| `'` | `\'` |
| `,` | `\x2C` |
| `=` | `\x3D` |
| `?` | `\?` |
| `\` | `\\` |

These rules reproduce exactly the two JavaScript constants the IDE wrote in Xojo's WebSDK examples (about 5 KB of
JavaScript). The `\\` rule is tested: `/^\\d+$/` in the constant reaches the browser as `/^\d+$/`. Use LF line
endings in the source and keep the JavaScript ASCII-only: write symbols as HTML entities (`&#9733;`) or JavaScript
escapes. Send user-visible text from the server through `Serialize` rather than writing it into the JavaScript.

Escape with a script rather than by hand. For example, in Python:

```python
import sys

ESCAPES = {"\\": "\\\\", '"': '\\"', "'": "\\'", "\n": "\\n", "\t": "\\t",
           ",": "\\x2C", "=": "\\x3D", "?": "\\?"}

def xojo_string_constant(name, text, scope="Private"):
    text = text.replace("\r\n", "\n")
    value = "".join(ESCAPES.get(ch, ch) for ch in text)
    return (f'\t#tag Constant, Name = {name}, Type = String, Dynamic = False, '
            f'Default = \\"{value}", Scope = {scope}\n\t#tag EndConstant\n')

print(xojo_string_constant("kJS", open(sys.argv[1], encoding="utf-8").read()), end="")
```

Keep the readable JavaScript in a working folder outside the project, and regenerate the constant line after every
change. The project's copy is what runs.

## Libraries and CSS

- **CSS:** put it in a `kCSS` constant and serve it the same way from `SessionCSSURLs` (`WebSDKUIControl` only),
  with `MIMEType = "text/css"`. The file is shared by every instance in every session, so select by a class name
  your `render()` adds (`.acme-thing`), never by a control id.
- **A JavaScript library:** serve it from the app the same way, from its own constant and shared `WebFile`, and list
  its URL before your control's file in `SessionJavascriptURLs`. The documentation doesn't say whether the files
  load in that order *(unverified)*, so check in the browser that the library exists before your class uses it.
  Avoid links to a CDN: the app would stop working offline, or when the CDN changes. Tell the user the library's
  license.
- Anything for the page's `<head>` (a `<meta>` or `<link>` tag) goes in `SessionHead`.

## Loading from disk while developing

Xojo's Bootstrap Toast example has a switch for loading the JavaScript from a file instead of the constant, so a
changed file needs no edit to the constant:

```xojo
Const UseRealFile = False

If SharedJSFile = Nil Then
  #If UseRealFile
    Var f As FolderItem = SpecialFolder.Resources.Child("Toast.js")
    SharedJSFile = WebFile.Open(f)
  #Else
    SharedJSFile = New WebFile
    SharedJSFile.Data = kJS
  #EndIf
  ...
End If
```

The file must reach the app's Resources folder through a Copy Files build step ([BuildAutomation.md](../BuildAutomation.md)).
For an agent, regenerating the constant is usually simpler. If you use this switch, set it back to `False` before
handing the project over.

## TypeScript (optional)

TypeScript adds type checking and autocomplete for the `XojoWeb` API. The compiled JavaScript goes into the
constant as above.

1. In a working folder outside the project, make two sibling folders, `typescript/` and `dist/`. Don't nest them.
2. Copy [SDK/XojoWebSDK.d.ts](SDK/XojoWebSDK.d.ts) into `typescript/`.
3. Add `typescript/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2015",
    "strict": true,
    "removeComments": true,
    "noImplicitReturns": true,
    "skipLibCheck": true,
    "outDir": "../dist"
  }
}
```

4. Write the class in a namespace, with no top-level `import` or `export`, so it compiles to a plain script:

```ts
namespace Acme {
    export class Thing extends XojoWeb.XojoVisualControl {
        constructor(id: string, events: string[]) {
            super(id, events);
        }
    }
}
```

5. Compile from `typescript/` with `npx tsc` (or `npx tsc -w` to recompile on every save). It needs Node.js. Using
   jQuery from TypeScript also needs its typings: `npm install --save-dev @types/jquery`.
6. Turn `dist/<name>.js` into the `kJS` constant line.

`target` `ES2015` matches the framework. A later target (`ES2017`…) emits syntax that older supported browsers may
not run.
