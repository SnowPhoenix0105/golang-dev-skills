# Complete Testing Guide

## Table of Contents

- [Creating a Test Environment](#creating-a-test-environment)
- [Simulating User Interaction](#simulating-user-interaction)
- [Verifying Rendering Results](#verifying-rendering-results)
- [Complete Test Examples](#complete-test-examples)
- [CI Integration](#ci-integration)
- [Testing Best Practices](#testing-best-practices)
- [Feature-Specific Test Examples](#feature-specific-test-examples)

---

The `fyne.io/fyne/v2/test` package provides pure in-memory UI testing — no graphics driver required.

## Creating a Test Environment

```go
import "fyne.io/fyne/v2/test"

// Create an in-memory Canvas
c := test.NewCanvas()
c.SetContent(myWidget)
c.Resize(fyne.NewSize(400, 300))

// Create an in-memory App (includes Preferences, Storage, Clipboard)
a := test.NewApp()
w := a.NewWindow("Test")
```

## Simulating User Interaction

### Taps

```go
// Tap a Tappable object
test.Tap(btn)

// Tap at a specific position
test.TapAt(widget, fyne.NewPos(10, 10))

// Tap at absolute canvas coordinates
test.TapCanvas(canvas, fyne.NewPos(100, 50))

// Double-tap
test.DoubleTap(btn)

// Right-click
test.TapSecondary(widget)
test.TapSecondaryAt(widget, fyne.NewPos(5, 5))
```

### Keyboard Input

```go
// FocusGained first, then rune by rune
test.Type(entry, "Hello World")

// Canvas-level input (triggers OnTypedRune)
test.TypeOnCanvas(canvas, "text")
```

### Drag & Scroll

```go
// Drag from pos by deltaX, deltaY
test.Drag(canvas, fyne.NewPos(50, 50), 30, 10)

// Scroll
test.Scroll(canvas, fyne.NewPos(100, 100), 0, -50) // scroll up
```

### Mouse Movement

```go
// Simulate mouse movement (triggers Hoverable.MouseIn/MouseMoved/MouseOut)
test.MoveMouse(canvas, fyne.NewPos(100, 100))
```

### Focus Control

```go
test.FocusNext(canvas)
test.FocusPrevious(canvas)
```

## Verifying Rendering Results

### Via WidgetRenderer

```go
func TestLabelText(t *testing.T) {
    label := widget.NewLabel("Hello")
    r := test.WidgetRenderer(label)

    // Check render object count
    objs := r.Objects()
    if len(objs) != 1 {
        t.Fatalf("expected 1 object, got %d", len(objs))
    }

    // Check render object content
    text := objs[0].(*canvas.Text)
    if text.Text != "Hello" {
        t.Errorf("expected 'Hello', got '%s'", text.Text)
    }
}
```

### RenderToMarkup (v2.6.0+)

```go
// Object-level snapshot
markup := test.RenderObjectToMarkup(myWidget)
if !strings.Contains(markup, "expected text") {
    t.Error("text not found in markup")
}

// Canvas-level snapshot
snapshot := test.RenderToMarkup(canvas)
```

### Checking Layout Results

```go
// LaidOutObjects returns all children after Layout
objects := test.LaidOutObjects(container)
for _, obj := range objects {
    t.Logf("pos=%v size=%v visible=%v", obj.Position(), obj.Size(), obj.Visible())
}
```

## Complete Test Examples

### Widget Function Test

```go
func TestButton_Tap(t *testing.T) {
    clicked := false
    btn := widget.NewButton("Click me", func() {
        clicked = true
    })

    test.Tap(btn)

    if !clicked {
        t.Error("callback should have been triggered")
    }
}
```

### Widget Rendering Test

```go
func TestButton_Layout(t *testing.T) {
    btn := widget.NewButton("Submit", nil)
    r := test.WidgetRenderer(btn)

    // Verify minimum size
    min := r.MinSize()
    if min.Width <= 0 || min.Height <= 0 {
        t.Error("MinSize should be positive")
    }

    // Verify after setting size
    r.Layout(fyne.NewSize(200, 40))
    objs := r.Objects()
    // objs[0] is background, objs[1] is text
    if objs[0].Size().Width != 200 {
        t.Error("background should fill widget")
    }
}
```

### Custom Widget Test

```go
func TestMyWidget_Rendering(t *testing.T) {
    w := NewMyWidget("test", nil)

    c := test.NewCanvas()
    c.SetContent(w)
    c.Resize(fyne.NewSize(200, 100))

    r := test.WidgetRenderer(w)
    objs := r.Objects()

    // Verify background
    bg := objs[0].(*canvas.Rectangle)
    if bg.Size().Width != 200 {
        t.Error("background should fill")
    }

    // Verify text
    for _, obj := range objs {
        if text, ok := obj.(*canvas.Text); ok {
            if text.Text != "test" {
                t.Errorf("expected 'test', got '%s'", text.Text)
            }
        }
    }
}
```

### Data Binding Test

```go
func TestEntryWithData(t *testing.T) {
    data := binding.NewString()
    data.Set("initial")

    entry := widget.NewEntryWithData(data)

    test.Type(entry, "new value")

    got, _ := data.Get()
    if got != "new value" {
        t.Errorf("data should be 'new value', got '%s'", got)
    }
}
```

### Focus Chain Test

```go
func TestFocusChain(t *testing.T) {
    e1 := widget.NewEntry()
    e2 := widget.NewEntry()

    c := test.NewCanvas()
    c.SetContent(container.NewVBox(e1, e2))

    test.FocusNext(c)
    first := c.Focused()
    test.FocusNext(c)
    second := c.Focused()

    if first == second {
        t.Error("focus should have moved")
    }
    if first.(fyne.Focusable) != e1 {
        t.Error("first focused should be e1")
    }
}
```

### Layout Test

```go
func TestVBoxLayout(t *testing.T) {
    l1 := widget.NewLabel("Item 1")
    l2 := widget.NewLabel("Item 2")

    box := container.NewVBox(l1, l2)

    c := test.NewCanvas()
    c.SetContent(box)
    c.Resize(fyne.NewSize(400, 200))

    // Verify l1 is above l2
    if l1.Position().Y >= l2.Position().Y {
        t.Error("l1 should be above l2")
    }
}
```

### Hide/Show Test

```go
func TestHideShow(t *testing.T) {
    label := widget.NewLabel("visible")
    if !label.Visible() {
        t.Fatal("should be visible by default")
    }

    label.Hide()
    if label.Visible() {
        t.Fatal("should be hidden")
    }

    label.Show()
    if !label.Visible() {
        t.Fatal("should be visible again")
    }
}
```

### Disable Test

```go
func TestDisable(t *testing.T) {
    btn := widget.NewButton("Click", func() { t.Log("clicked") })
    btn.Disable()

    if !btn.Disabled() {
        t.Fatal("should be disabled")
    }

    test.Tap(btn) // disabled button should not trigger callback

    btn.Enable()
    if btn.Disabled() {
        t.Fatal("should be enabled")
    }
}
```

## CI Integration

```bash
# Use ci tag to run tests in CI (no graphics driver needed)
go test -tags ci ./...

# Or run only Fyne-related tests
go test -tags ci -run "TestWidget" ./...
```

## Testing Best Practices

1. **Use test.NewCanvas()** — much faster than a real window, no graphics environment needed
2. **Check via WidgetRenderer** — verify render object content rather than manually screenshotting
3. **Test MinSize** — ensure reasonable dimensions in various layouts
4. **Test layout behavior** — put Widget in a container, verify position and size after Layout
5. **Don't use time.Sleep** — all operations in the test framework are synchronous
6. **Tag tests for CI** — `//go:build ci` ensures they also run in CI
7. **Write at least 3 kinds of tests per Widget**: construction/rendering, interaction (Tap/Type), state changes (Show/Hide/Enable)

## Feature-Specific Test Examples

### Label.Selectable (v2.6+)

```go
func TestLabel_Selectable(t *testing.T) {
    label := widget.NewLabel("select me")
    label.Selectable = true

    c := test.NewCanvas()
    c.SetContent(label)
    c.Resize(fyne.NewSize(200, 50))

    selected := label.SelectedText()
    if selected != "" {
        t.Error("no text should be selected initially")
    }

    // Simulate selection, then verify
    // Note: Selection operations require full focus and drag interaction
}
```

### Check.Partial (v2.6+)

```go
func TestCheck_Partial(t *testing.T) {
    check := widget.NewCheck("Select All", nil)
    
    // Set to partial (indeterminate) state
    check.Partial = true
    check.Refresh()

    // Verify state
    if !check.Partial {
        t.Error("check should be in Partial state")
    }

    // Tapping should clear Partial and switch to Checked
    test.Tap(check)
    if check.Partial {
        t.Error("tapping should clear Partial state")
    }
    if !check.Checked {
        t.Error("check should be checked after tap")
    }
}
```
