# Troubleshooting & Debugging Tips

## Table of Contents

- [Quick Diagnosis Table](#quick-diagnosis-table)
- [1. Widget Not Visible](#1-widget-not-visible)
- [2. Renderer Creation Failure](#2-renderer-creation-failure)
- [3. panic: fyne.Canvas is nil](#3-panic-fynecancas-is-nil)
- [4. panic: fyne.Layout is nil](#4-panic-fynelayout-is-nil)
- [5. Data Binding Not Updating](#5-data-binding-not-updating)
- [6. Layout Issues](#6-layout-issues)
- [7. Threading Issues](#7-threading-issues)
- [8. Build & Packaging Issues](#8-build--packaging-issues)
- [9. Debugging Tools](#9-debugging-tools)
- [10. Common Misuse Patterns](#10-common-misuse-patterns)
- [11. Getting Help](#11-getting-help)

---

## Quick Diagnosis Table

| Symptom | First Check |
|---------|------------|
| Widget not showing | `Objects()` non-empty? `ExtendBaseWidget` called? |
| Tap not triggering | Widget size > 0? `Tappable` interface implemented? |
| Layout broken | `MinSize()` reasonable? `Layout()` computing positions correctly? |
| panic | nil Layout? nil Canvas? `fyne.Do` called on main goroutine? |
| Data binding not updating | External binding calling `Reload()`? |
| Packaged resources failing | File paths used instead of embedded resources? |
| Frame rate drop / UI freeze | Refresh in tight loop? Refresh inside Layout? |
| Infinite recursion / StackOverflow | SetRowHeight called inside UpdateCell? |
| Memory leak | Repeated SetContent without cleaning old Widgets? |
| Theme colors not working | Custom Theme not covering all Color/Size methods? |
| v2.6+ `fyne.Do called from main goroutine` | `fyne.Do` used in event callback |

## 1. Widget Not Visible

**The most common issue.** Check in this order:

```
□ CreateRenderer() returning nil?
□ Objects() returning empty slice or containing nil?
□ ExtendBaseWidget(w) called in constructor?
□ Value receiver used instead of pointer (canvas.Line vs *canvas.Line)?
□ Manual Resize() missing when used without a Layout container?
```

### Debugging

```go
// Check Renderer
r := test.WidgetRenderer(myWidget)
if r == nil {
    fmt.Println("Renderer is nil — CreateRenderer returned nil")
}

// Check Objects
objs := r.Objects()
fmt.Printf("Objects count: %d\n", len(objs))
for i, obj := range objs {
    fmt.Printf("  [%d] %T, visible=%v\n", i, obj, obj.Visible())
}

// Check MinSize
fmt.Printf("MinSize: %v, Size: %v\n", myWidget.MinSize(), myWidget.Size())
```

## 2. Renderer Creation Failure

```
panic: runtime error: invalid memory address or nil pointer dereference
```

Check:
- Did `CreateRenderer()` return nil?
- Was `cache.Renderer(impl)` called when `impl` is nil? (check `ExtendBaseWidget` was called)
- Did the Widget call methods requiring a Renderer during construction?

## 3. panic: fyne.Canvas is nil

Called an operation requiring Canvas before the Widget was added to a Window/Canvas.

```go
// ❌ Wrong
selectWidget := widget.NewSelect(options, callback)
selectWidget.Tapped(ev)  // internally needs Canvas, which is nil

// ✅ Correct: put in Canvas first
window.SetContent(selectWidget)
// now Canvas is available
```

## 4. panic: fyne.Layout is nil

Widget Renderer not yet initialized, internal Layout is nil.

```go
// ❌ Wrong
list := widget.NewList(lenFn, createFn, updateFn)
list.Resize(fyne.NewSize(100, 100))  // Renderer not created

// ✅ Correct: put in container, then Resize the container
c := container.NewVBox(list)
window.SetContent(c)
c.Resize(fyne.NewSize(400, 300))
```

## 5. Data Binding Not Updating

### Checklist

```go
// 1. External binding using Reload correctly?
var count int
b := binding.BindInt(&count)
count = 10       // ❌ direct modification doesn't notify
b.Reload()       // ✅ must call explicitly

// 2. Correct WithData constructor?
label := widget.NewLabelWithData(data)  // ✅ auto-bound

// 3. Listener registered?
data.AddListener(binding.NewDataListener(func() {
    fmt.Println("Changed!")  // verify it fires
}))

// 4. List binding updating correctly?
list := binding.NewStringList()
list.Set(append(existing, "new"))  // ✅ triggers notification
```

## 6. Layout Issues

### 6.1 Children Overlapping or Clipped

- Is the parent container large enough? `parent.Resize(largerSize)`
- Is `MinSize` returning too small a value?

### 6.2 Gaps in Grid

Hidden objects still occupy grid cells. Fix:
```go
// ❌ hidden but occupies space
obj.Hide()

// ✅ remove from container
container.Remove(obj)
```

### 6.3 Form Layout Alignment

`layout.FormLayout` requires paired objects per row (label + input):
```go
formLayout := layout.NewFormLayout()
// Objects must be ordered [label1, input1, label2, input2, ...]
```

## 7. Threading Issues

### 7.1 Race Conditions

```go
// ❌ Wrong
go func() {
    label.SetText("updated")  // race risk
}()

// ✅ Correct
go func() {
    fyne.Do(func() {
        label.SetText("updated")
    })
}()
```

### 7.2 Deadlocks

Common deadlock scenarios:
- Operating on OverlayStack or modifying focus inside `FocusGained/FocusLost`
- Calling `Refresh` inside `Layout`
- Calling `fyne.DoAndWait` recursively from within a `fyne.DoAndWait` callback

## 8. Build & Packaging Issues

### 8.1 Resources 404 After Packaging

```go
// ❌ Filesystem path unavailable after packaging
img := canvas.NewImageFromFile("/Users/me/project/icon.png")

// ✅ Embed resources
//go:generate fyne bundle -o bundled.go icon.png
img := canvas.NewImageFromResource(resourceIconPng)
```

### 8.2 Cross-Compilation

```bash
# fyne-cross as a tool
# Official Docker images support Windows/macOS/Linux cross-compilation
```

## 9. Debugging Tools

### 9.1 Debug Rendering

```bash
# Show layout bounding boxes and interactive areas
go run -tags debug main.go
```

### 9.2 RenderToMarkup Debugging

```go
import "fyne.io/fyne/v2/test"

// Get text-based structure of Widget
markup := test.RenderObjectToMarkup(widget)
fmt.Println(markup)
```

### 9.3 Traversing the Render Tree

```go
func printTree(obj fyne.CanvasObject, indent string) {
    fmt.Printf("%s%T pos=%v size=%v visible=%v\n",
        indent, obj, obj.Position(), obj.Size(), obj.Visible())

    if w, ok := obj.(fyne.Widget); ok {
        r := test.WidgetRenderer(w)
        for _, child := range r.Objects() {
            printTree(child, indent+"  ")
        }
    }
    if c, ok := obj.(*fyne.Container); ok {
        for _, child := range c.Objects {
            printTree(child, indent+"  ")
        }
    }
}
```

### 9.4 Checking Theme

```go
app := fyne.CurrentApp()
th := app.Settings().Theme()
fmt.Printf("Variant: %v, Scale: %.2f\n",
    app.Settings().ThemeVariant(), app.Settings().Scale())

// Reset to default theme
app.Settings().SetTheme(theme.DefaultTheme())
```

## 10. Common Misuse Patterns

### 10.1 Refresh Abuse — Calling Refresh in Layout or Resize callbacks

**The most insidious performance issue.** `Resize()` and `Layout()` are called frequently; calling `Refresh()` inside them causes exponential redraws.

```go
// ❌ Wrong — Refresh in Layout
func (r *myRenderer) Layout(size fyne.Size) {
    r.bg.Resize(size)
    r.w.Refresh()  // triggers new Layout → infinite loop or performance crash
}

// ❌ Wrong — Refresh in MinSize
func (r *myRenderer) MinSize() fyne.Size {
    r.w.Refresh()  // MinSize is queried frequently, redraws each time
    return fyne.NewSize(100, 30)
}

// ✅ Correct — Layout only adjusts geometry, no Refresh
func (r *myRenderer) Layout(size fyne.Size) {
    r.bg.Resize(size)
}
```

### 10.2 Refresh vs Resize Confusion — Frequent Refresh Crashes Frame Rate

`Resize()` only changes geometry (cheap), `Refresh()` does a full redraw (expensive).

```go
// ❌ Frequent Refresh — frame rate drops from 60fps to 10fps
for i := 0; i < 1000; i++ {
    progress.SetValue(float64(i) / 1000)
    progress.Refresh()  // 1000 redraws per second!
}

// ✅ Batch update — single Refresh at the end
for i := 0; i < 1000; i++ {
    progress.SetValue(float64(i) / 1000)
}
progress.Refresh()  // done once
```

**Better approach**: for continuous changes, use data binding — the framework auto-optimizes refreshes:

```go
data := binding.NewFloat()
widget.NewProgressBarWithData(data)

go func() {
    for i := 0; i <= 100; i += 10 {
        time.Sleep(100 * time.Millisecond)
        data.Set(float64(i) / 100)  // framework auto-optimizes refresh rate
    }
}()
```

### 10.3 Adding/Removing Container Children One by One in a Loop

```go
// ❌ Wrong — each Add/Remove triggers layout
for _, item := range items {
    container.Add(widget.NewLabel(item))
}

// ✅ Correct — batch replace
widgets := make([]fyne.CanvasObject, len(items))
for i, item := range items {
    widgets[i] = widget.NewLabel(item)
}
container.Objects = widgets
container.Refresh()
```

### 10.4 Modifying Layout State in UpdateCell/UpdateRow Callbacks

`widget.List` and `widget.Table` update callbacks are called inside the render loop; calling `SetRowHeight` or `Resize` inside them causes infinite recursion (stack overflow).

```go
// ❌ Wrong — SetRowHeight in UpdateCell
table := widget.NewTable(
    func() (int, int) { return 10, 3 },
    func() fyne.CanvasObject { return widget.NewLabel("") },
    func(id widget.TableCellID, obj fyne.CanvasObject) {
        table.SetRowHeight(id.Row, 50)  // → triggers re-layout → re-enters UpdateCell → infinite recursion
    })

// ✅ Correct — pre-set row heights outside UpdateCell
table.SetRowHeight(0, 50)
```

### 10.5 Calling fyne.Do on the Main Goroutine (v2.6.0+)

v2.6.0 introduced strict thread checking; calling `fyne.Do` on the main goroutine produces an error:

```
*** Error in Fyne call thread, fyne.Do[AndWait] called from main goroutine ***
```

```go
// ❌ Fyne callbacks are already on the main goroutine, fyne.Do not needed
btn.OnTapped = func() {
    fyne.Do(func() {
        label.SetText("clicked")  // v2.6.0+ error!
    })
}

// ✅ Fyne callbacks can operate directly
btn.OnTapped = func() {
    label.SetText("clicked")
}
```

**Rule**: `fyne.Do` is only needed inside manually created `go func()` background goroutines. Fyne event callbacks (`OnTapped`, `OnChanged`, `CreateRenderer`, Layout, etc.) are already on the main goroutine.

### 10.6 Background Goroutine Operating on UI Without fyne.Do

```go
// ❌ Race risk (v2.6.0+ directly errors)
go func() {
    time.Sleep(10 * time.Millisecond)
    canvas.Refresh(myWidget)  // not on main goroutine
}()

// ✅ Correct
go func() {
    time.Sleep(10 * time.Millisecond)
    fyne.Do(func() {
        canvas.Refresh(myWidget)
    })
}()
```

### 10.7 Large Image Synchronous Decode Blocking UI

`canvas.NewImageFromFile()` decodes images synchronously. Large files (>2MB) can block the main thread for 30ms+.

```go
// ❌ Blocks main goroutine
img := canvas.NewImageFromFile("large-photo.jpg")

// ✅ Decode in background, return to main goroutine
go func() {
    file, _ := os.Open("large-photo.jpg")
    decoded, _, _ := image.Decode(file)
    file.Close()
    fyne.Do(func() {
        img := canvas.NewImageFromImage(decoded)
        container.Add(img)
    })
}()
```

### 10.8 Using Hide() Instead of Remove() Causing Gaps

Hidden objects don't display but **still occupy layout space**, especially noticeable in Grid/FixedGridLayout.

```go
// ❌ Hidden but still occupies cell
obj.Hide()

// ✅ Completely remove from container
container.Remove(obj)
```

### 10.9 CanvasForObject Return Value Not Nil-Checked

When the Widget hasn't been added to a Canvas yet, `CanvasForObject()` returns nil.

```go
// ❌ Potential nil panic
c := fyne.CurrentApp().Driver().CanvasForObject(myWidget)
c.Refresh(myWidget)

// ✅ Check first, or just use w.Refresh() (handled internally)
if c != nil {
    c.Refresh(myWidget)
}
// Simpler:
myWidget.Refresh()  // BaseWidget.Refresh already nil-checks
```

### 10.10 Non-Visible Widget MinSize May Be 0

Non-visible Widgets have no renderer; `MinSize()` may return {0, 0}. Tests must put Widget in Canvas first.

```go
// ❌ Direct MinSize test may be inaccurate
w := NewMyWidget()
min := w.MinSize()  // may be {0, 0}

// ✅ Put in test Canvas first
c := test.NewCanvas()
c.SetContent(w)
c.Resize(fyne.NewSize(400, 300))
min = w.MinSize()  // now correct
```

### 10.11 SetContent Without Cleaning Up Old Content Causes Memory Leaks

Repeated `SetContent()` leaves old Widget trees in memory. If the old tree holds data binding listeners or event callbacks, GC is prevented.

```go
// ❌ Memory leak
w.SetContent(page1)
w.SetContent(page2)  // page1 and all its child Widgets leak

// ✅ Option 1: Use Stack container, Hide old page
pages := container.NewStack(page1, page2)
w.SetContent(pages)
page1.Hide()
page2.Show()
// page1 stays in memory but doesn't leak, can be Show()n again

// ✅ Option 2: Clean up listeners before switching
func switchPage(w fyne.Window, newPage fyne.CanvasObject) {
    old := w.Content()
    if cleanable, ok := old.(interface{ Cleanup() }); ok {
        cleanable.Cleanup()
    }
    w.SetContent(newPage)
}
```

### 10.12 Ignoring the Dialog parent Parameter

Dialogs need a parent Window to determine position and lifecycle. Passing nil causes display position issues or event propagation problems.

```go
// ❌ parent is nil
dialog.ShowInformation("Title", "Msg", nil)

// ✅ Always pass a valid Window
dialog.ShowInformation("Title", "Msg", myWindow)
```

## 11. Getting Help

- Official docs: https://docs.fyne.io
- API docs: https://pkg.go.dev/fyne.io/fyne/v2
- Forum: https://github.com/fyne-io/fyne/discussions
- Issue Tracker: https://github.com/fyne-io/fyne/issues

### Finding Fyne Source Code

```bash
# Method 1: go doc (no path needed, direct API lookup)
go doc fyne.io/fyne/v2/widget
go doc fyne.io/fyne/v2/widget.Label

# Method 2: Find Fyne module path
FYNE_DIR=$(go list -m -json fyne.io/fyne/v2 | grep '"Dir"' | cut -d'"' -f4)
echo "$FYNE_DIR/widget/"

# Method 3: GOMODCACHE fallback
ls "$(go env GOMODCACHE)/fyne.io/fyne/v2@v2.5.0"

# Method 4: Search for specific symbols
rg "func.*CreateRenderer" "$(go env GOMODCACHE)/fyne.io/fyne/v2@*/widget/"
```

### Fyne Core Package Structure (by import path)

| import | Description |
|--------|-------------|
| `fyne.io/fyne/v2` | Core interfaces: App, Window, Canvas, CanvasObject, Widget, Layout |
| `fyne.io/fyne/v2/app` | App implementation |
| `fyne.io/fyne/v2/widget` | Built-in Widgets (Button, Label, Entry, List, ...) |
| `fyne.io/fyne/v2/container` | Container constructors (NewVBox, NewBorder, ...) |
| `fyne.io/fyne/v2/layout` | Layout algorithms (HBoxLayout, VBoxLayout, GridLayout, ...) |
| `fyne.io/fyne/v2/canvas` | Primitive graphics objects (Rectangle, Text, Image, Line, ...) |
| `fyne.io/fyne/v2/dialog` | Dialogs |
| `fyne.io/fyne/v2/data/binding` | Data binding |
| `fyne.io/fyne/v2/test` | Test toolkit |
| `fyne.io/fyne/v2/theme` | Theme: colors, fonts, icons, sizes |
| `fyne.io/fyne/v2/driver/desktop` | Desktop-specific features (Hoverable, CustomShortcut) |
| `fyne.io/fyne/v2/storage` | Storage abstraction |
| `fyne.io/fyne/v2/internal/cache` | Internal cache (Renderer cache, theme cache) |
