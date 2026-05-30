# Custom Widget In-Depth Guide

## Table of Contents

- [Widget/Renderer Separation Pattern](#widgetrenderer-separation-pattern)
- [Complete Implementation Template](#complete-implementation-template)
- [Key Rules Explained](#key-rules-explained)
- [Extending Existing Widgets](#extending-existing-widgets)
- [Supporting Data Binding](#supporting-data-binding)
- [Debugging Custom Widgets](#debugging-custom-widgets)
- [Common Mistakes](#common-mistakes)
- [Content Padding Guidelines](#content-padding-guidelines)

---

## Widget/Renderer Separation Pattern

Fyne uses a declarative-renderer separation pattern:
- **Widget**: holds state (text, selection state, data bindings, etc.) — a declarative object
- **WidgetRenderer**: each Widget instance has one renderer, responsible for the actual CanvasObject drawing

`CreateRenderer()` is called once by the framework when rendering is needed, and the result is cached. Multiple `Refresh()` calls do not re-create the Renderer.

## Complete Implementation Template

```go
package mywidget

import (
    "image/color"

    "fyne.io/fyne/v2"
    "fyne.io/fyne/v2/canvas"
    "fyne.io/fyne/v2/theme"
    "fyne.io/fyne/v2/widget"
)

// 1. Declarative Widget
type MyWidget struct {
    widget.BaseWidget

    Text  string
    Icon  fyne.Resource
    Color color.Color

    OnTapped func()
}

// 2. Constructor
func NewMyWidget(text string, tapped func()) *MyWidget {
    w := &MyWidget{
        Text:     text,
        OnTapped: tapped,
        Color:    theme.Color(theme.ColorNamePrimary),
    }
    w.ExtendBaseWidget(w) // 🔴 MUST call
    return w
}

// 3. CreateRenderer
func (w *MyWidget) CreateRenderer() fyne.WidgetRenderer {
    bg := canvas.NewRectangle(w.Color)
    icon := canvas.NewImageFromResource(w.Icon)
    label := canvas.NewText(w.Text, theme.Color(theme.ColorNameForeground))

    r := &myRenderer{
        bg:    bg,
        icon:  icon,
        label: label,
        w:     w,
    }
    r.objects = []fyne.CanvasObject{bg, icon, label}
    return r
}

// 4. Optional: implement functional interfaces
func (w *MyWidget) Tapped(ev *fyne.PointEvent) {
    if w.OnTapped != nil {
        w.OnTapped()
    }
}

// 5. Setters
func (w *MyWidget) SetText(text string) {
    w.Text = text
    w.Refresh()
}

func (w *MyWidget) SetColor(c color.Color) {
    w.Color = c
    w.Refresh()
}
```

```go
// 6. Renderer
type myRenderer struct {
    bg       *canvas.Rectangle
    icon     *canvas.Image
    label    *canvas.Text
    w        *MyWidget
    objects  []fyne.CanvasObject
}

func (r *myRenderer) Destroy() {
    // Clean up resources (animations, callbacks, etc.)
}

func (r *myRenderer) Layout(size fyne.Size) {
    // 🔴 Never call r.w.Refresh() in this method
    pad := theme.Padding()

    r.bg.Resize(size)

    // icon top-left
    iconSize := fyne.NewSquareSize(size.Height - pad*2)
    r.icon.Resize(iconSize)
    r.icon.Move(fyne.NewPos(pad, pad))

    // label to the right of icon
    labelX := pad*2 + iconSize.Width
    r.label.Move(fyne.NewPos(labelX, pad))
    r.label.Resize(fyne.NewSize(size.Width-labelX-pad, size.Height-pad*2))
}

func (r *myRenderer) MinSize() fyne.Size {
    // 🔴 Compute from child objects — don't hardcode
    iconMin := r.icon.MinSize()
    labelMin := r.label.MinSize()
    pad := theme.Padding()

    w := iconMin.Width + labelMin.Width + pad*3
    h := fyne.Max(iconMin.Height, labelMin.Height) + pad*2
    return fyne.NewSize(w, h)
}

func (r *myRenderer) Objects() []fyne.CanvasObject {
    // 🔴 MUST return a non-empty slice with no nil elements
    return r.objects
}

func (r *myRenderer) Refresh() {
    // 🔴 Sync Widget state to draw objects
    r.bg.FillColor = r.w.Color
    r.label.Text = r.w.Text
    r.icon.Resource = r.w.Icon

    // Refresh draw objects
    r.bg.Refresh()
    r.label.Refresh()
    r.icon.Refresh()
}
```

## Key Rules Explained

### Rule 1: Must Call ExtendBaseWidget

```go
func NewMyWidget() *MyWidget {
    w := &MyWidget{}
    w.ExtendBaseWidget(w)  // Sets internal impl pointer
    return w
}
```

Not calling it causes:
- `Refresh()` to be ineffective (can't find Renderer)
- Data binding listeners not firing
- `Theme()` returning nil
- Broken event handling

### Rule 2: Objects() Must Not Return Empty/Nil

```go
// ❌ Wrong
func (r *myRenderer) Objects() []fyne.CanvasObject {
    return []fyne.CanvasObject{}  // Widget invisible
}

// ❌ Wrong
func (r *myRenderer) Objects() []fyne.CanvasObject {
    return []fyne.CanvasObject{nil}  // causes panic
}

// ✅ Correct
func (r *myRenderer) Objects() []fyne.CanvasObject {
    return r.objects  // at least 1 valid object
}
```

### Rule 3: Use Pointer Receivers

```go
// ❌ canvas.Line value type does not satisfy CanvasObject interface
line := canvas.Line{}

// ✅ Must use pointer
line := &canvas.Line{}
```

Same applies to `canvas.Text`, `canvas.Circle`, `canvas.Rectangle`, etc.

### Rule 4: Don't Call Refresh in Layout

```go
// ❌ Wrong — may cause infinite recursion
func (r *myRenderer) Layout(size fyne.Size) {
    r.bg.Resize(size)
    r.w.Refresh()  // Refresh may trigger Layout
}

// ✅ Correct
func (r *myRenderer) Layout(size fyne.Size) {
    r.bg.Resize(size)
    // Only adjust geometry, don't trigger Refresh
}
```

### Rule 5: MinSize Based on Children

```go
// ❌ Hardcoded
func (r *myRenderer) MinSize() fyne.Size {
    return fyne.NewSize(100, 30)  // Breaks adaptive layout
}

// ✅ Based on child minSize + padding
func (r *myRenderer) MinSize() fyne.Size {
    labelMin := r.label.MinSize()
    pad := theme.Padding()
    return fyne.NewSize(labelMin.Width+pad*2, labelMin.Height+pad*2)
}
```

## Extending Existing Widgets

### Extending Label

```go
type MyLabel struct {
    widget.Label
    Priority int  // custom field
}

func NewMyLabel(text string, priority int) *MyLabel {
    l := &MyLabel{Priority: priority}
    l.ExtendBaseWidget(l)
    l.SetText(text)
    return l
}
```

Accessing extended fields in custom Layout:
```go
func (l *myLayout) Layout(objects []fyne.CanvasObject, size fyne.Size) {
    for _, o := range objects {
        if myLabel, ok := o.(*MyLabel); ok {
            // access myLabel.Priority
        }
    }
}
```

## Supporting Data Binding

```go
type BoundWidget struct {
    widget.BaseWidget

    data     binding.String
    listener binding.DataListener
}

func NewBoundWidget(data binding.String) *BoundWidget {
    w := &BoundWidget{data: data}
    w.ExtendBaseWidget(w)

    w.listener = binding.NewDataListener(func() {
        w.Refresh()  // data changed → redraw
    })
    data.AddListener(w.listener)
    return w
}
```

## Debugging Custom Widgets

1. Check Objects() is non-empty: `test.WidgetRenderer(w).Objects()`
2. Verify MinSize > 0
3. Use `test.NewCanvas()` to create a test environment
4. Use `test.Tap(w)` to verify interaction
5. Use `test.RenderObjectToMarkup(w)` to view rendering snapshot

## Common Mistakes

| Error | Consequence | Fix |
|-------|-------------|-----|
| Forgot `ExtendBaseWidget` | Widget completely non-functional | Call at end of constructor |
| `Objects()` returns empty | Widget invisible | Return at least 1 object |
| `Objects()` contains nil | panic | Ensure all elements non-nil |
| `canvas.Line` value type | Compilation error | Use `*canvas.Line` |
| `MinSize` hardcoded | Adaptive layout broken | Compute from children |
| `Refresh` in `Layout` | Infinite recursion | Remove Refresh call |
| Multiple Widgets share Renderer | State chaos | Each Widget has its own Renderer |

## Content Padding Guidelines

Align your custom Widget visually with built-in components by following Fyne's padding standards:

```go
func (r *myRenderer) MinSize() fyne.Size {
    th := r.w.Theme()
    pad := th.Size(theme.SizeNamePadding)          // standard spacing 4
    innerPad := th.Size(theme.SizeNameInnerPadding) // content inset 8

    // content area = text size + innerPad on both sides
    textSize := fyne.MeasureText("Sample", th.Size(theme.SizeNameText), fyne.TextStyle{})
    return fyne.NewSize(textSize.Width+innerPad*2, textSize.Height+innerPad*2)
}

func (r *myRenderer) Layout(size fyne.Size) {
    th := r.w.Theme()
    innerPad := th.Size(theme.SizeNameInnerPadding)

    // Position text at left edge + innerPad, aligning with Label/Entry
    r.text.Move(fyne.NewPos(innerPad, innerPad))
    r.text.Resize(fyne.NewSize(size.Width-innerPad*2, size.Height-innerPad*2))
}
```

**Key values**:
- `theme.SizeNamePadding` (default 4) — spacing between elements in containers
- `theme.SizeNameInnerPadding` (default 8) — content inset within a control

Built-in components (Label, Entry, Button, etc.) consistently use these values. Custom Widgets that follow the same standard ensure visual consistency.
