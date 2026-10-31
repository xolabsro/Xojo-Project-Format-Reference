# Pragma Directives — Format Reference

Write `#Pragma` directives inside method or event bodies, alongside ordinary
code. See `CancelClosing` in [Window1](DesktopAppStructure/Window1.xojo_window).

## Unused parameters and variables

```xojo
#Pragma Unused appQuitting
```

Suppress unused-variable analysis for one declared variable or parameter per
directive.

## TODO reminders

```xojo
#Pragma Warning "TODO: Implement validation"
```

The message appears when analyzing the project. Use `#Pragma Error "Message"`
when the issue must prevent compilation.

## Execution and debugging

| Directive | Values | Purpose |
| --------- | ------ | ------- |
| `BackgroundTasks` | `True`, `False` | Control automatic background-task yielding |
| `BoundsChecking` | `True`, `False` | Control array bounds checks |
| `NilObjectChecking` | `True`, `False` | Control automatic nil checks |
| `StackOverflowChecking` | `True`, `False` | Control stack overflow checks |
| `BreakOnExceptions` | `True`, `False`, `Default` | Override debugger exception breaking |
| `Debug` | `True`, `False` | Enable or disable method debugging |
| `X86CallingConvention` | `StdCall`, `CDecl` | Set a method's x86 calling convention |

`DisableBackgroundTasks` and `DisableBoundsChecking` disable their respective
settings. Do not disable background tasks in Web applications: other sessions
can stall or disconnect.

Toggleable checks and background-task settings follow their enclosing scope
and do not propagate into called methods. Restore them explicitly when needed
within that scope.

Android does not support `BreakOnExceptions` or `Debug`.

See [Xojo's pragma documentation](https://documentation.xojo.com/api/language/pragma_directives.html).
