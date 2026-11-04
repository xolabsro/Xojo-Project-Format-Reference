# Xojo WebSDK — Guide for AI Agents

How to decide whether a Xojo web app needs a custom control, then write one with the **Xojo WebSDK** and use it in a
web project (Xojo 2026r3+, project `Type=Web2`). A WebSDK control is two pieces of code that work together:

- a **Xojo class** in the project, subclassing `WebSDKUIControl` (visual) or `WebSDKControl` (non-visual);
- a **JavaScript class** that runs in the browser, subclassing `XojoWeb.XojoVisualControl` or `XojoWeb.XojoControl`.

The server sends the JavaScript to each browser session once. After that, the server sends the control's state as
JSON, the JavaScript draws the control, and the JavaScript sends events back to the server.

There is nothing to build or install. Both pieces live in the Xojo project: the JavaScript is a string constant in
the Xojo class.

This folder is self-contained: the guide covers everything you need, and [SDK/](SDK/) holds the TypeScript typings
for the browser API.

> **Status:** derived from Xojo's WebSDK 2.0 documentation, its example projects and `XojoWebSDK.d.ts`.
> [Example.md](Example.md) is tested end to end on Windows with Xojo 2026r3.1: compile, run, both directions of
> communication, CSS, tooltip and layout editor drawing. Where the documentation and the examples disagree, the
> guide follows the examples and says so. Items marked *(unverified)* aren't confirmed by a test or an example.

## Files

| File | Read it when |
| ---- | ------------ |
| [ServerSide.md](ServerSide.md) | Writing the Xojo class: supers, events, properties, events for the user, the project files |
| [BrowserSide.md](BrowserSide.md) | Writing the JavaScript class: lifecycle, the `XojoControl` and `XojoVisualControl` API, rules |
| [JavaScriptDelivery.md](JavaScriptDelivery.md) | Putting the JavaScript and CSS into the project: string constants, `WebFile`, TypeScript |
| [LayoutEditor.md](LayoutEditor.md) | Making a visual control draw itself in the IDE's layout editor |
| [Example.md](Example.md) | A complete, tested rating control and the page that uses it |
| [SDK/](SDK/) | `XojoWebSDK.d.ts`, the browser API typings, for TypeScript only |

For the text project format itself (manifest lines, tags, indentation), see the kit's [README](../README.md) and
[WebAppStructure/](../WebAppStructure/README.md).

## 1. Does the app need a WebSDK control?

Usually not. Pick the first option that does the job:

1. **Built-in web controls**, styled with `WebStyle`, `CSSClasses` or a container (`WebContainer`) that combines
   them.
2. **`ExecuteJavaScript`** on the page or session for one-off browser calls, or a **`WebHTMLViewer`** for static
   HTML or an embedded third-party widget that doesn't need to talk to the app.
3. **A WebSDK control**, only when one of these is true:
   - the control has state that must stay in sync between the browser and the app (a value the user changes);
   - the browser must send events to the app (clicks, changes, drag and drop);
   - it wraps a JavaScript library (a chart, an editor, a map) as a reusable control;
   - it draws something no built-in control can.

Use free, open-source JavaScript libraries only, and tell the user their licenses. Don't suggest commercial
third-party controls.

## 2. Propose it to the user first

Never add a WebSDK control unannounced. Tell the user:

- **why** a custom control is needed, and why options 1–2 don't do the job;
- **what it adds:** one class in their project (the class file and a line in the `.xojo_project` manifest), plus any
  JavaScript library it loads;
- **what they'll do:** run the project and check the control in the browser.

Then ask them for a **namespace** (see section 4). Continue only if the user agrees.

## 3. Workflow

1. Choose the super: `WebSDKUIControl` for a control the user sees, `WebSDKControl` for one that only runs code in
   the browser ([ServerSide.md](ServerSide.md#choosing-the-super)).
2. Write the JavaScript class ([BrowserSide.md](BrowserSide.md)).
3. Write the Xojo class file with the JavaScript in a string constant
   ([ServerSide.md](ServerSide.md), [JavaScriptDelivery.md](JavaScriptDelivery.md)), and add it to the manifest.
4. Place the control on a page ([ServerSide.md](ServerSide.md#using-the-control-in-the-project)).
5. **Run the project.**
   - If the project folder has an `XOJO-CODING.md` file, the IDE is in agentic mode. That file describes how to ask
     the IDE to run the project and where it writes compile errors. Follow it.
   - Otherwise, ask the user to run the project and report any compile errors. If the IDE was open while you
     edited the files, ask them to close and reopen the project first, so the IDE loads your changes.
6. **Check the control in the browser.** If you can open the running app (it serves on `http://127.0.0.1:8080` by
   default), check it yourself: the console, the control's elements, a click in each direction
   ([BrowserSide.md](BrowserSide.md#debugging)). Otherwise ask the user to check the console for JavaScript errors and
   confirm the control looks and behaves as intended. Ask the user to look at the page in the layout editor too.
7. Fix and repeat until it works. Then continue with the rest of the project.

## 4. Namespaces and names

Every control's JavaScript lives in a namespace, so it can't clash with the Xojo framework, other controls, jQuery
or Bootstrap.

- **Ask the user for the namespace.** It must have at least 3 characters, contain only letters, digits and `_`,
  and not be a JavaScript keyword. Names starting with `Xojo` (any case) and the name `Example` are reserved by Xojo.
- If the user will sell or share the control with other developers, they should register the namespace with Xojo
  by email (WebNamespace@xojo.com). Registration covers that exact root only: `Acme` doesn't cover `AcmeControls`.
- Use the same namespace in both places:
  - **Xojo class name:** namespace, underscore, control name: `Acme_Rating`;
  - **JavaScript class:** `Acme.Rating`, which is also what `JavaScriptClassName` returns.
- JavaScript is case-sensitive. Pick one spelling of the namespace and use it everywhere.
