# Control Containment — Format Reference

Nest the child's `Begin … End` block inside the parent control's block and
set the child's parent property:

```text
Begin ParentControl Parent1
   Left = 20
   Top = 20

   Begin ChildControl Child1
      <ParentProperty> = "Parent1"
      Left = 40
      Top = 40
   End
End
```

`Parent1` is the name of the parent control. Replace it with the actual control name in both the `Begin` line and the child's parent property.

Replace the control types and `<ParentProperty>` with the values for the
project type:

| Project type | Parent property |
| ------------ | --------------- |
| Desktop | `InitialParent = "Parent1"` |
| Web | `Parent = "Parent1"` |
| Android | `Parent = "Parent1"` |
| iOS | `Parent = "Parent1"` |

## Coordinates

> Save `Left` and `Top` relative to the window, page, or screen, not relative to the parent control.

In the example above, the parent is at `(20, 20)` and the child is at
`(40, 40)`. The child is 20 units from the parent's left and top edges. If the
parent moves to `(100, 20)`, update the child to `(120, 40)` to preserve that
offset.

For iOS, also set the child's `AutoLayout` constraints relative to its parent.
