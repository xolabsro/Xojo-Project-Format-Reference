# WebSDK typings

`XojoWebSDK.d.ts` is an unmodified copy of the TypeScript declarations for the browser side of the Xojo Web
framework (WebSDK 2.0), as shipped with Xojo's WebSDK examples. It declares the `XojoWeb` namespace:
`XojoControl`, `XojoVisualControl`, `JSONItem`, the uploader engine and the WebListBox cell renderer API.

- Don't edit it.
- You need it only when writing the browser code in TypeScript: copy it into the TypeScript folder of your working
  folder (see [JavaScriptDelivery.md](../JavaScriptDelivery.md#typescript-optional)).
- Plain JavaScript needs nothing from this folder. [BrowserSide.md](../BrowserSide.md) lists the same API.
