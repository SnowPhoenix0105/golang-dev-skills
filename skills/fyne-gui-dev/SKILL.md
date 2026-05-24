---
name: fyne-gui-dev
description: |
  Fyne cross-platform GUI application development. Use this skill when creating, modifying, debugging,
  or understanding Fyne GUI applications.
  Trigger scenarios: mentions of fyne, GUI desktop apps, cross-platform interfaces, custom Widget,
  fyne layouts, data binding, widget.New*/container.New*/app.New* and other Fyne APIs,
  Go GUI development. Even if the user doesn't explicitly say "fyne", consider this skill
  for any Go GUI development.
---

# Fyne GUI Application Development

Fyne is a pure Go cross-platform GUI framework with OpenGL ES 2.0 hardware-accelerated rendering.
Core design: **declarative Widgets + reactive data binding**, one API covering Windows/macOS/Linux/iOS/Android/WebAssembly.

## Fyne Ecosystem

The core framework `fyne.io/fyne/v2` provides cross-platform GUI fundamentals. Peripheral repos for optional use:

| Repo | Purpose | go module |
|------|---------|-----------|
| **fyne-x** | Community extensions: extra Widgets (FileTree, CompletionEntry, AnimatedGif, Map, HexWidget, NumericalEntry), ResponsiveLayout, About dialog, Adwaita theme | `fyne.io/x/fyne` |
| **fyne-cross** | Docker cross-compilation: one command to package Fyne apps for Windows/macOS/Linux/Android/iOS/FreeBSD | `github.com/fyne-io/fyne-cross` |
| **terminal** | Terminal emulator Widget: ANSI escape sequences, mouse, selection/copy, alternate screen buffer | `github.com/fyne-io/terminal` |
| **systray** | Cross-platform system tray (no GTK dependency): icon, menu, click events | `fyne.io/systray` |
| **tools** | fyne CLI: `go run fyne.io/tools/v2/cmd/fyne` — build, package, resource embedding | `fyne.io/tools/v2` |
| **refyne** | UI serialization: JSON import/export, Go code generation, property editor | `github.com/fyne-io/refyne` |
| **gl-js** | OpenGL ES 2 bindings (WebGL backend + Windows support) — internal dependency, rarely used directly | `github.com/fyne-io/gl-js` |
| **defyne** | Experimental IDE (superseded by Apptrix.ai), JSON serialization work spun out to refyne | `github.com/fyne-io/defyne` |

To browse local source, use `go list -m -json fyne.io/fyne/v2` to find the module path.

## Core Principles

1. **Container composition > absolute positioning** — use `container.New*` functions to build layouts
2. **Data binding first** — prefer the `binding` package for automatic UI updates over manual Widget manipulation
3. **Thread safety** — all UI operations must be on the main goroutine, dispatched via `fyne.Do()` / `fyne.DoAndWait()`
4. **Widget/Renderer separation** — Widget holds state, Renderer handles drawing
5. **Test-first** — use `fyne.io/fyne/v2/test` for in-memory UI testing

## Minimal App

```go
package main

import (
    "fyne.io/fyne/v2"
    "fyne.io/fyne/v2/app"
    "fyne.io/fyne/v2/container"
    "fyne.io/fyne/v2/widget"
)

func main() {
    a := app.New()              // or app.NewWithID("com.example.app")
    w := a.NewWindow("Title")
    w.SetContent(container.NewVBox(
        widget.NewLabel("Hello"),
        widget.NewButton("Click", func() {
            // handle click
        }),
    ))
    w.Resize(fyne.NewSize(400, 300))
    w.ShowAndRun()              // show + app.Run()
}
```

## Framework Interfaces (understand and possibly implement)

Fyne's core is a set of interface contracts. Built-in components all implement these interfaces; custom components need to implement them too. Understanding these interfaces is the foundation for using Fyne correctly.

```
Layer:                   What you do:
──────────────────────────────────────────
fyne.App          — App entry point          Call app.New()
fyne.Window       — Window management        Call w.SetContent() / w.Show()
fyne.Canvas       — Canvas (focus, scale)    Call c.Focus() / c.SetOnTypedKey()
fyne.Layout       — Layout algorithm         Use built-in layouts directly; implement this for custom layouts
fyne.Container    — Container (object group) Call container.New*(); rarely used directly
fyne.Widget       — Widget interface         Extend BaseWidget, implement CreateRenderer()
fyne.WidgetRenderer — Renderer interface     Implement Destroy/Layout/MinSize/Objects/Refresh
fyne.CanvasObject — Most basic interface     All drawable objects must implement (MinSize/Position/Size/Visible...)
```

**Functional interfaces a Widget can implement** (opt-in, not all required):
`Tappable` → tap | `DoubleTappable` → double-tap | `SecondaryTappable` → right-click
`Focusable` → keyboard input | `Scrollable` → scroll wheel | `Draggable` → drag
`Disableable` → enable/disable | `Shortcutable` → shortcut response

## Built-in Components (ready to use)

All provided by the framework — just call the constructor. They internally implement the framework interfaces above.

### Basic Widgets (full list in `references/api-reference.md`)

```go
// Top 10 most common
widget.NewLabel("text")                    // .Selectable = true for selectable text (v2.6+)
widget.NewButton("text", func() {})
widget.NewEntry()                          // .Text, .OnChanged
widget.NewCheck("label", func(bool) {})    // .Partial = true for indeterminate (v2.6+)
widget.NewSelect([]string{"a","b"}, func(string) {})
widget.NewSlider(min, max)
widget.NewProgressBar()
widget.NewRichText(segments...)            // multi-segment rich text
widget.NewRichTextFromMarkdown(content)    // Markdown → RichText
widget.NewCard(title, subtitle, content)
```

Forms (`widget.NewForm`), Toolbars (`widget.NewToolbar`), Calendar (`NewCalendar` v2.6+), Markdown rendering (`NewMarkdown`), and other Widgets — see `references/api-reference.md`.

### Virtual Scrolling (required for large datasets)

```go
list := widget.NewList(
    func() int { return rowCount },                     // row count
    func() fyne.CanvasObject { return widget.NewLabel("template") }, // template
    func(id widget.ListItemID, obj fyne.CanvasObject) { // update row
        obj.(*widget.Label).SetText(fmt.Sprintf("Row %d", id))
    })
// Similar: widget.NewTable / widget.NewTree
```

### Data-Bound Widgets (reactive UI)

```go
data := binding.NewString()
widget.NewEntryWithData(data)       // input ↔ data
widget.NewLabelWithData(data)       // data → auto-update
widget.NewCheckWithData("label", binding.NewBool())
widget.NewSliderWithData(min, max, binding.NewFloat())
widget.NewListWithData(dataList, createFn, updateFn)
```

## Containers & Layouts

```go
import "fyne.io/fyne/v2/container"

// Most common containers (full list in references/api-reference.md)
container.NewVBox(objs...)                   // vertical
container.NewHBox(objs...)                   // horizontal
container.NewBorder(top, bottom, left, right, center...) // border
container.NewGridWithColumns(cols, objs...)  // grid by columns
container.NewAdaptiveGrid(rowcols, objs...)  // adaptive grid
container.NewStack(objs...)                  // stacked
container.NewCenter(objs...)                 // centered
container.NewPadded(objs...)                 // standard padding

// Special
container.NewScroll(content)                 // scrollable
container.NewHSplit(left, right)             // horizontal split
container.NewVSplit(top, bottom)             // vertical split
container.NewAppTabs(items...)               // tabbed
container.NewNavigation(content)             // navigation (back button) (v2.7+)
container.NewClip(content)                   // clip overflow (v2.7+)
```

Spacer:
```go
container.NewHBox(layout.NewSpacer(), widget.NewButton("OK", fn))
```

GridWrap / RowWrap / Max / DocTabs and more — see `references/api-reference.md`.

## Data Binding

```go
import "fyne.io/fyne/v2/data/binding"

// goroutine-safe — readable/writable from any goroutine
s := binding.NewString()               // .Get() / .Set(val)
b := binding.NewBool()
i := binding.NewInt()

// External binding: bind to existing variables (call .Reload() after modifying)
binding.BindString(&myVar)

// Conversions
binding.IntToString(intBinding)
binding.SPrintf("Count: %d", intBinding)
```

Full API (collection bindings, list bindings, tree bindings, URI bindings, Preference bindings, etc.) — see `references/api-reference.md` Data Binding API section.

## Canvas Primitives

```go
import "fyne.io/fyne/v2/canvas"

canvas.NewRectangle(color)              // rectangle (common background)
canvas.NewCircle(color)
canvas.NewLine(color)                   // MUST use *canvas.Line (pointer!)
canvas.NewText("text", color)           // MUST use *canvas.Text (pointer!)
canvas.NewImageFromFile(path)
canvas.NewLinearGradient(start, end)
```

**v2.7+ additions**: `NewSquare`, `NewArc`/`NewPieArc`/`NewDoughnutArc`, `NewPolygon`, Rectangle corner radius (`CornerRadius`/`Aspect`), Image fill modes (`FillMode`/`CornerRadius`). Full list and properties — see `references/api-reference.md` Canvas Primitives + Rectangle Properties + ImageFill Modes sections.

## Custom Widgets

```go
type MyWidget struct {
    widget.BaseWidget
    Text string
}

func NewMyWidget(text string) *MyWidget {
    w := &MyWidget{Text: text}
    w.ExtendBaseWidget(w)  // REQUIRED! Sets internal impl pointer
    return w
}

func (w *MyWidget) CreateRenderer() fyne.WidgetRenderer {
    bg := canvas.NewRectangle(theme.Color(theme.ColorNameButton))
    label := canvas.NewText(w.Text, theme.Color(theme.ColorNameForeground))
    return &myRenderer{bg: bg, label: label, w: w,
        objs: []fyne.CanvasObject{bg, label}}
}

type myRenderer struct {
    bg, label fyne.CanvasObject
    w         *MyWidget
    objs      []fyne.CanvasObject
}

func (r *myRenderer) Destroy() {}
func (r *myRenderer) Layout(size fyne.Size) { r.bg.Resize(size) }
func (r *myRenderer) MinSize() fyne.Size    { return fyne.NewSize(100, 30) }
func (r *myRenderer) Objects() []fyne.CanvasObject { return r.objs }
func (r *myRenderer) Refresh() {
    r.label.(*canvas.Text).Text = r.w.Text
    r.label.Refresh()
}

// Optional: implement Tappable etc.
func (w *MyWidget) Tapped(ev *fyne.PointEvent) { /* ... */ }
```

**5 Key Rules**:
1. Constructor **MUST** call `ExtendBaseWidget(w)`
2. `Objects()` **MUST** return a non-empty slice with no nil elements
3. All Canvas objects use **pointers** (`*canvas.Line` not `canvas.Line`)
4. **Never** call `Refresh()` inside `Layout()`
5. `MinSize()` should be computed from child objects — **don't hardcode**

## Threading Model

```go
// Update UI safely from background goroutine
go func() {
    result := expensiveWork()
    fyne.Do(func() {
        label.SetText(result)     // ✅ safe
    })
}()

// Wait for execution to complete
var title string
fyne.DoAndWait(func() {
    title = window.Title()
})
```

Data binding APIs are safe from any goroutine; callbacks are automatically dispatched via `fyne.Do`.

## Dialogs

```go
dialog.ShowInformation("Title", "Message", parent)
dialog.ShowConfirm("Title", "Msg", func(ok bool) {}, parent)
dialog.ShowError(err, parent)
dialog.ShowFileOpen(callback, parent)
dialog.ShowFileSave(callback, parent)
dialog.ShowFolderOpen(callback, parent)
dialog.ShowCustom("Title", "OK", content, parent)
dialog.ShowForm("Title", "Submit", "Cancel", formItems, callback, parent)
dialog.ShowEntry("Title", "Msg", parent)
dialog.ShowColorPicker("Title", "Msg", callback, parent)
dialog.NewProgress("Title", "Msg", parent)       // returns ProgressDialog
```

## Menus & Shortcuts

```go
// Window menu bar
item := fyne.NewMenuItem("Copy", nil)
item.Shortcut = &fyne.ShortcutCopy{Clipboard: w.Clipboard()}
fileMenu := fyne.NewMenu("File",
    fyne.NewMenuItem("New", newFile),
    fyne.NewMenuItemSeparator(),
    fyne.NewMenuItem("Quit", func() { a.Quit() }))
w.SetMainMenu(fyne.NewMainMenu(fileMenu, editMenu))

// Canvas-level custom shortcuts
w.Canvas().AddShortcut(&desktop.CustomShortcut{
    KeyName: fyne.KeyS, Modifier: fyne.KeyModifierControl | fyne.KeyModifierShift,
}, func(shortcut fyne.Shortcut) { /* Save As */ })
```

## Theme & Styling

```go
// TextStyle
fyne.TextStyle{Bold: true, Italic: false, Monospace: false,
    Underline: false, Strikethrough: false, Symbol: false, TabWidth: 4}
```

### Custom Theme (full template in references/api-reference.md)

```go
type myTheme struct{}
func (t myTheme) Color(name fyne.ThemeColorName, v fyne.ThemeVariant) color.Color {
    switch name {
    case theme.ColorNamePrimary:
        return color.NRGBA{R: 70, G: 130, B: 255, A: 255}
    default:
        return theme.DefaultTheme().Color(name, v) // fallback to default
    }
}
func (t myTheme) Font(s fyne.TextStyle) fyne.Resource { return theme.DefaultTheme().Font(s) }
func (t myTheme) Icon(name fyne.ThemeIconName) fyne.Resource { return theme.DefaultTheme().Icon(name) }
func (t myTheme) Size(name fyne.ThemeSizeName) float32 { return theme.DefaultTheme().Size(name) }

app.Settings().SetTheme(&myTheme{})
container.NewThemeOverride(&myTheme{}, content)  // local override
```

For common colors/sizes/fonts/icons, see `references/api-reference.md` Theme & Styling section.

## Testing

```go
import "fyne.io/fyne/v2/test"

// Simulate interaction
test.Tap(btn)
test.TapCanvas(c, pos)
test.Type(entry, "text")                // FocusGained first, then rune by rune
test.Drag(c, pos, deltaX, deltaY)
test.Scroll(c, pos, deltaX, deltaY)
test.MoveMouse(c, pos)
test.FocusNext(c) / test.FocusPrevious(c)

// Verify rendering
r := test.WidgetRenderer(widget)
objs := r.Objects()                     // inspect rendered objects
markup := test.RenderObjectToMarkup(o)  // snapshot (v2.6+)

// Test environment
c := test.NewCanvas()
a := test.NewApp()
// CI: go test -tags ci ./...
```

## Common Errors Quick Reference

| Problem | Cause | Fix |
|---------|-------|-----|
| Widget invisible | `Objects()` empty/nil, forgot `ExtendBaseWidget` | Check CreateRenderer and constructor |
| No tap response | Size is 0, or `Tappable` not implemented | Manually Resize or implement `Tapped` |
| `panic: Layout is nil` | `Resize()` called before Renderer init | Put in container first, then Resize |
| `panic: Canvas is nil` | Widget not in window hierarchy | Call `SetContent` / `Show` first |
| Packaged resources 404 | File system paths used | `fyne bundle` to embed as Go code |
| Grid has gaps | `Hide()` objects still occupy cells | `Remove` from `Objects` |
| Data binding not updating | External var modified without `Reload()` | Call `b.Reload()` |
| Non-visible Widget MinSize is 0 | Framework optimization: no renderer for invisible widgets | Put in Canvas before testing |
| v2.6+ `fyne.Do called from main goroutine` | `fyne.Do` used inside event callback (already on main goroutine) | Remove `fyne.Do` from event callbacks |

## Stuck? Read These First

When references don't cover your case and you need to explore Fyne source code, use these methods.

### Package Path → Filesystem Mapping

Find the filesystem path for a Go package path like `fyne.io/fyne/v2/widget` using one of two methods:

```bash
# Method 1: go list (precise, recommended)
FYNE_DIR=$(go list -m -json fyne.io/fyne/v2 | grep '"Dir"' | cut -d'"' -f4)

# Method 2: GOMODCACHE (simple, supports wildcards)
FYNE_DIR=$(echo $(go env GOMODCACHE)/fyne.io/fyne/v2@*)
# If multiple versions exist, pick one or specify the version:
FYNE_DIR=$(go env GOMODCACHE)/fyne.io/fyne/v2@v2.5.0
```

Once you have `$FYNE_DIR`, the package subpath maps directly: `fyne.io/fyne/v2/widget` → `$FYNE_DIR/widget/`, `fyne.io/fyne/v2/canvas` → `$FYNE_DIR/canvas/`.

### Finding APIs

```bash
# go doc (preferred, no source path needed)
go doc fyne.io/fyne/v2/widget
go doc fyne.io/fyne/v2/widget.Button

# rg search
rg "func.*New.*Pop" "$FYNE_DIR/widget/"
rg "type.*Renderer" "$FYNE_DIR/widget/"
```

### Learning from Built-in Widgets

After finding `$FYNE_DIR`, reading built-in Widget source is the best way to learn. For example, to see how components in the `widget` package implement `CreateRenderer`:

```bash
rg "ExtendBaseWidget" "$FYNE_DIR/widget/"
rg "CreateRenderer" "$FYNE_DIR/widget/" | head -20
```

### Fyne Code Conventions

Understanding these conventions lets you infer undocumented APIs from built-in Widget source:

- **Widget constructor**: `New<Name>(requiredParams...) *<Name>`, internally calls `ExtendBaseWidget(self)`
- **Declarative Widget/Renderer separation**: Widget struct holds state, Renderer struct holds CanvasObjects
- **Renderer created in `CreateRenderer()`** and cached in `internal/cache`
- **Naming convention**: `On<Event>` is a callback field, `Set<Property>` is a setter, `<Property>()` is a getter
- **Refresh trigger chain**: `widget.Refresh()` → `cache.Renderer(w).Refresh()` → update CanvasObject state → `canvas.Refresh(w)`
- **Layout trigger chain**: `widget.Resize(size)` → `cache.Renderer(w).Layout(size)` → arrange children

## References (Look Up by Scenario)

| Scenario | Read | Key Sections |
|----------|------|--------------|
| Unsure which Widget/API to use | `references/api-reference.md` | Complete Widget list, Container list |
| Need full color/icon/font lists | `references/api-reference.md` | Theme & Styling section |
| Layout issues / responsive design | `references/best-practices.md` | Section 2 "Layout Patterns" |
| Data binding not working | `references/best-practices.md` | Section 3 "Data Binding Patterns" |
| Widget not showing / crash / perf | `references/troubleshooting.md` | Start with quick diagnosis table, then sections |
| Writing custom Widgets | `references/custom-widget.md` | Full template + 5 key rules |
| How to write unit tests | `references/testing.md` | Interaction simulation, rendering verification |

If the references above don't cover your case, follow the "Stuck? Read These First" section to explore the source. If exploring the source still doesn't resolve the issue, **report the specifics to the user and ask for help — do not silently guess.**
