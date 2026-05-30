# Best Practices & Usage Patterns

## Table of Contents

- [1. Project Structure](#1-project-structure)
- [2. Layout Patterns](#2-layout-patterns)
- [3. Data Binding Patterns](#3-data-binding-patterns)
- [4. Async Operation Patterns](#4-async-operation-patterns)
- [5. Custom Widget In-Depth](#5-custom-widget-in-depth)
- [6. Performance Optimization](#6-performance-optimization)
- [7. Platform Adaptation](#7-platform-adaptation)
- [8. Window Management](#8-window-management)
- [9. Resource Management](#9-resource-management)
- [10. Dialog Patterns](#10-dialog-patterns)
- [11. Navigation Pattern](#11-navigation-pattern)
- [12. Canvas Styling Patterns](#12-canvas-styling-patterns)
- [13. Label Selectable Text](#13-label-selectable-text)
- [14. RowWrap Layout](#14-rowwrap-layout)
- [15. i18n Translations](#15-i18n-translations)
- [16. Custom Layouts](#16-custom-layouts)
- [17. Extending Existing Widgets](#17-extending-existing-widgets)
- [18. Advanced Preferences](#18-advanced-preferences)
- [19. System Tray Lifecycle](#19-system-tray-lifecycle)
- [20. Advanced Multi-Window](#20-advanced-multi-window)
- [21. Animation](#21-animation)
- [22. Scaling & Device Adaptation](#22-scaling--device-adaptation)

---

## 1. Project Structure

Recommended Fyne project structure:

```
myapp/
├── main.go              # entry point, app init
├── FyneApp.toml         # metadata (ID, version, icon, build settings)
├── ui/
│   ├── views/           # pages/screens
│   ├── widgets/         # custom reusable components
│   └── theme/           # custom theme
├── model/               # data models and business logic
├── bindings/            # reactive data bindings
└── go.mod
```

### FyneApp.toml

```toml
[Details]
  ID = "com.example.myapp"
  Name = "My Application"
  Version = "1.0.0"
  Build = 1
  Icon = "Icon.png"

[Migrations]
  fyneDo = true   # v2.6.0+: enable new threading model
```

## 2. Layout Patterns

### 2.1 Basic Page Structure

```go
func mainPage() fyne.CanvasObject {
    header := widget.NewLabel("App Title")
    sidebar := widget.NewList(/* ... */)
    body := widget.NewLabel("Content")
    footer := widget.NewLabel("Status: Ready")

    return container.NewBorder(
        header,               // top
        footer,               // bottom
        sidebar,              // left
        nil,                  // right (empty)
        body,                 // center
    )
}
```

### 2.2 Form Layout

```go
func formPage() fyne.CanvasObject {
    nameEntry := widget.NewEntry()
    emailEntry := widget.NewEntry()
    passwordEntry := widget.NewPasswordEntry()

    form := &widget.Form{
        Items: []*widget.FormItem{
            {Text: "Name", Widget: nameEntry},
            {Text: "Email", Widget: emailEntry},
            {Text: "Password", Widget: passwordEntry},
        },
        OnSubmit: func() {
            // validate and submit
        },
    }
    return form
}
```

### 2.3 Adaptive Grid (Responsive)

```go
// rowcols is the column count in landscape; becomes row count in portrait
grid := container.NewAdaptiveGrid(4,
    widget.NewButton("A", nil),
    widget.NewButton("B", nil),
    // ...
)
```

### 2.4 Flexible Space

```go
// Use layout.Spacer to push buttons to the right
toolbar := container.NewHBox(
    layout.NewSpacer(),
    widget.NewButton("Save", saveFn),
    widget.NewButton("Cancel", cancelFn),
)
```

### 2.5 Fixed-Width Sidebar

```go
sidebar := widget.NewList(/* ... */)
content := widget.NewLabel("Body")

// Left pane fixed width, right pane fills remaining
split := container.NewHSplit(sidebar, content)
split.Offset = 0.2  // left pane takes 20%
```

## 3. Data Binding Patterns

### 3.1 Two-Way Binding (Input ↔ Display)

```go
data := binding.NewString()
data.Set("initial")

// Three Widgets share the same data source
input := widget.NewEntryWithData(data)
preview := widget.NewLabelWithData(data)
count := widget.NewLabelWithData(
    binding.SPrintf("Length: %d", binding.StringToInt(data)))
```

### 3.2 List Binding

```go
items := binding.NewStringList()

list := widget.NewListWithData(items,
    func() fyne.CanvasObject {
        return widget.NewLabel("template")
    },
    func(item binding.DataItem, obj fyne.CanvasObject) {
        label := obj.(*widget.Label)
        label.Bind(item.(binding.String))
    })

// Update data from background thread (safe)
go func() {
    results := fetchData()
    items.Set(results)  // safe — auto-dispatched via fyne.Do
}()
```

### 3.3 External Binding (bind to existing variables)

```go
type Model struct {
    Name  string
    Count int
}
m := &Model{Name: "Alice", Count: 5}

nameBinding := binding.BindString(&m.Name)
countBinding := binding.BindInt(&m.Count)

widget.NewLabelWithData(nameBinding)
widget.NewLabelWithData(binding.IntToString(countBinding))

// After external code modifies m, notify:
m.Name = "Bob"
nameBinding.Reload()  // important!
```

### 3.4 Preference Binding

```go
a := app.NewWithID("com.example.app")
prefs := a.Preferences()

// Auto two-way sync with Preferences
themeBinding := binding.BindPreferenceString("theme", prefs)
widget.NewSelectWithData(
    []string{"light", "dark"},
    themeBinding,
    func(val string) {
        // switch theme
    })
```

## 4. Async Operation Patterns

### 4.1 Background Task + UI Update

```go
func loadData(progress *widget.ProgressBar, label *widget.Label) {
    go func() {
        for i := 0; i <= 100; i += 10 {
            time.Sleep(100 * time.Millisecond)
            fyne.Do(func() {
                progress.SetValue(float64(i) / 100.0)
                label.SetText(fmt.Sprintf("Loading... %d%%", i))
            })
        }

        result := computeResult()
        fyne.Do(func() {
            label.SetText(result)
        })
    }()
}
```

### 4.2 Use Data Binding Instead of Manual Thread Dispatch

```go
// Cleaner approach: use data binding
progress := binding.NewFloat()
status := binding.NewString()

go func() {
    for i := 0; i <= 100; i += 10 {
        time.Sleep(100 * time.Millisecond)
        progress.Set(float64(i) / 100.0)
        status.Set(fmt.Sprintf("Loading... %d%%", i))
    }
    status.Set("Done!")
}()

// UI components auto-respond
widget.NewProgressBarWithData(progress)
widget.NewLabelWithData(status)
```

## 5. Custom Widget In-Depth

### 5.1 Data-Binding-Aware Custom Widget

```go
type ToggleWidget struct {
    widget.BaseWidget
    Label  string
    Active bool
    OnChanged func(bool)
}

func (w *ToggleWidget) CreateRenderer() fyne.WidgetRenderer {
    return newToggleRenderer(w)
}

func (w *ToggleWidget) Tapped(ev *fyne.PointEvent) {
    w.Active = !w.Active
    if w.OnChanged != nil {
        w.OnChanged(w.Active)
    }
    w.Refresh()  // trigger renderer update
}
```

### 5.2 Theme-Aware Widget

```go
func (r *myRenderer) MinSize() fyne.Size {
    th := r.w.Theme()   // get current Theme (may come from ThemeOverride container)
    pad := th.Size(theme.SizeNamePadding)
    textSize := th.Size(theme.SizeNameText)
    return fyne.NewSize(textSize*10+pad*2, textSize+pad*2)
}

func (r *myRenderer) Refresh() {
    th := r.w.Theme()
    r.bg.FillColor = th.Color(theme.ColorNameBackground, fyne.ThemeVariantLight)
    r.bg.Refresh()
    canvas.Refresh(r.w)
}
```

### 5.3 Using cache.Renderer for Child Widget Renderers

```go
import "fyne.io/fyne/v2/internal/cache"

func (r *myRenderer) Layout(size fyne.Size) {
    childRenderer := cache.Renderer(r.w.childWidget)
    childRenderer.Layout(size)
}
```

## 6. Performance Optimization

### 6.1 Virtual Scrolling

Large datasets (>100 rows) **must use virtual scrolling**:

```go
// ✅ O(visible rows) complexity
list := widget.NewList(dataLen, createTemplate, updateRow)

// ❌ O(total rows) — creates all Widgets at once
for _, item := range items {
    container.Add(widget.NewLabel(item))
}
```

### 6.2 Batch Updates

```go
// ✅ Batch then single Refresh
data := binding.NewStringList()
newItems := fetchItems()
data.Set(newItems)  // triggers once

// ❌ Triggers per-item in loop
for _, item := range items {
    data.Append(item)  // triggers Refresh each time
}
```

### 6.3 Avoid Unnecessary Rendering

```go
// Widget.Resize already does equality check
func (w *BaseWidget) Resize(size fyne.Size) {
    if size == w.Size() {
        return  // size unchanged, skip
    }
    // ...
}
```

## 7. Platform Adaptation

### 7.1 Detecting Device Type

```go
if fyne.CurrentDevice().IsMobile() {
    // mobile-specific layout
    w.SetContent(container.NewVBox(/* ... */))
} else {
    // desktop
    w.SetContent(container.NewHSplit(sidebar, content))
}

if fyne.CurrentDevice().IsBrowser() {
    // WebAssembly-specific handling
}
```

### 7.2 Keyboard Availability Check

```go
if fyne.CurrentDevice().HasKeyboard() {
    // register keyboard shortcuts
    w.Canvas().SetOnTypedKey(func(ev *fyne.KeyEvent) {
        // ...
    })
}
```

## 8. Window Management

### 8.1 Multiple Windows

```go
a := app.New()
mainWin := a.NewWindow("Main")
settingsWin := a.NewWindow("Settings")

// Default: all windows closed → quit
// Custom: intercept close
settingsWin.SetCloseIntercept(func() {
    if hasUnsavedChanges {
        dialog.ShowConfirm("Quit?", "Unsaved changes!", func(ok bool) {
            if ok { settingsWin.Close() }
        }, settingsWin)
    } else {
        settingsWin.Close()
    }
})
```

### 8.2 System Tray (Desktop Feature)

```go
// If driver supports desktop features
if desk, ok := a.(desktop.App); ok {
    menu := fyne.NewMenu("Tray",
        fyne.NewMenuItem("Show", func() { w.Show() }),
        fyne.NewMenuItem("Quit", func() { a.Quit() }),
    )
    desk.SetSystemTrayMenu(menu)
}
```

## 9. Resource Management

### 9.1 Resource Embedding

```bash
# CLI
fyne bundle -o bundled.go image.png font.ttf

# Or go:generate
//go:generate fyne bundle -o bundled.go assets/*.png
```

```go
img := canvas.NewImageFromResource(resourceImagePng)
```

### 9.2 Custom Theme

```go
type CustomTheme struct{}

func (t CustomTheme) Color(name fyne.ThemeColorName, variant fyne.ThemeVariant) color.Color {
    switch name {
    case theme.ColorNamePrimary:
        return color.NRGBA{R: 70, G: 130, B: 255, A: 255}
    default:
        return theme.DefaultTheme().Color(name, variant)
    }
}

func (t CustomTheme) Font(style fyne.TextStyle) fyne.Resource {
    if style.Monospace {
        return resourceMonospaceFont
    }
    return theme.DefaultTheme().Font(style)
}

func (t CustomTheme) Icon(name fyne.ThemeIconName) fyne.Resource {
    return theme.DefaultTheme().Icon(name)
}

func (t CustomTheme) Size(name fyne.ThemeSizeName) float32 {
    return theme.DefaultTheme().Size(name)
}

// Apply
a.Settings().SetTheme(&CustomTheme{})

// Local theme override
container.NewThemeOverride(&CustomTheme{}, content)
```

## 10. Dialog Patterns

### 10.1 Confirm Action

```go
func deleteItem(id string, w fyne.Window) {
    dialog.ShowConfirm("Delete Item",
        fmt.Sprintf("Delete item #%s?", id),
        func(confirmed bool) {
            if confirmed {
                doDelete(id)
            }
        }, w)
}
```

### 10.2 Progress Dialog

```go
func processWithProgress(w fyne.Window) {
    prog := dialog.NewProgress("Processing", "Please wait...", w)

    go func() {
        for i := 0; i <= 100; i += 10 {
            time.Sleep(100 * time.Millisecond)
            fyne.Do(func() { prog.SetValue(float64(i) / 100.0) })
        }
        fyne.Do(func() {
            prog.Hide()
            dialog.ShowInformation("Done", "Processing complete", w)
        })
    }()

    prog.Show()
}
```

## 11. Navigation Pattern

`container.NewNavigation` (v2.7+) provides a page navigation stack with a back button, suitable for settings pages, wizards, and other multi-level interfaces.

```go
func settingsPage() fyne.CanvasObject {
    general := widget.NewLabel("General Settings")
    advanced := widget.NewLabel("Advanced Settings")

    // Navigation: pushing a new page auto-shows back button
    nav := container.NewNavigationWithTitle(general, "Settings")

    general.OnTapped = func() {
        nav.Push(advanced)  // push new page, show back button
    }
    return nav
}
```

## 12. Canvas Styling Patterns

### 12.1 Rectangle Corner Radius & Aspect Ratio

```go
// Pill-shaped button background
bg := canvas.NewRectangle(theme.Color(theme.ColorNamePrimary))
bg.CornerRadius = 20  // set to half height ≈ pill

// Top-only rounded corners
bg.TopLeftCornerRadius = 8
bg.TopRightCornerRadius = 8

// Fixed aspect ratio
bg.Aspect = 1.6  // width auto = height × 1.6
```

### 12.2 Image Fill Modes

```go
// Avatar: Cover crop to fill
avatar := canvas.NewImageFromFile("avatar.jpg")
avatar.FillMode = canvas.ImageFillCover
avatar.SetMinSize(fyne.NewSquareSize(48))

// Thumbnail: Contain to fit without cropping
thumb := canvas.NewImageFromFile("photo.jpg")
thumb.FillMode = canvas.ImageFillContain
thumb.SetMinSize(fyne.NewSize(200, 150))
```

## 13. Label Selectable Text

v2.6+ allows users to select and copy Label text:

```go
label := widget.NewLabel("Selectable text")
label.Selectable = true  // users can select and Ctrl+C to copy

// Get selected text
selected := label.SelectedText()
```

## 14. RowWrap Layout

v2.7+ RowWrap arranges children horizontally and auto-wraps when space is insufficient, similar to CSS flex-wrap:

```go
// Via GridWrap container (uses RowWrap internally)
tags := container.NewGridWrap(fyne.NewSize(80, 30),
    widget.NewButton("Go", nil),
    widget.NewButton("Rust", nil),
    widget.NewButton("Python", nil),
    widget.NewButton("TypeScript", nil),
)

// Manual RowWrap layout
c := container.New(layout.NewRowWrapLayout(), objs...)
// Or with custom padding
c := container.New(layout.NewRowWrapLayoutWithCustomPadding(8, 4), objs...)
```

## 15. i18n Translations

v2.5.0+ includes built-in translation support via the `fyne.io/fyne/v2/lang` package.

### 15.1 Marking Translatable Strings

```go
import "fyne.io/fyne/v2/lang"

// L — the default value serves as the key; falls back to it when no translation is found
title := widget.NewLabel(lang.L("My App Title"))

// X (LocalizeKey) — explicit key + default value (for disambiguation)
title := widget.NewLabel(lang.X("window.title", "My App Window Title"))
```

### 15.2 Translation Files

One JSON file per language, placed in a `translation/` directory. Naming: `[prefix][language subtag].json`.

```json
// translation/en.json
{
  "My App Title": "My App Title"
}

// translation/zh.json
{
  "My App Title": "我的应用标题"
}

// translation/fr.json
{
  "My App Title": "Titre de mon application"
}
```

### 15.3 Embedding & Loading Translations

```go
import "fyne.io/fyne/v2/lang"

//go:embed translation
var translations embed.FS

func main() {
    a := app.New()
    lang.AddTranslationsFS(translations, "translation")
    // ...
}
```

### 15.4 Plurals

```go
// N (LocalizePlural) — selects singular/plural based on count
age := widget.NewLabel(lang.N("{% raw %}{{.Years}}{% endraw %} years old", years,
    map[string]any{"Years": years}))
```

Translation file distinguishes "one" and "other":

```json
{
  "{% raw %}{{.Years}}{% endraw %} years old": {
    "one": "1 year old",
    "other": "{% raw %}{{.Years}}{% endraw %} years old"
  }
}
```

## 16. Custom Layouts

Implement the `fyne.Layout` interface to create custom layouts. Only two methods are required:

```go
type Layout interface {
    Layout([]CanvasObject, Size)         // arrange children
    MinSize(objects []CanvasObject) Size // calculate minimum size
}
```

### 16.1 Complete Example: Diagonal Layout

```go
import "fyne.io/fyne/v2"

type diagonal struct{}

func (d *diagonal) MinSize(objects []fyne.CanvasObject) fyne.Size {
    w, h := float32(0), float32(0)
    for _, o := range objects {
        childSize := o.MinSize()
        w += childSize.Width
        h += childSize.Height
    }
    return fyne.NewSize(w, h)
}

func (d *diagonal) Layout(objects []fyne.CanvasObject, containerSize fyne.Size) {
    // Start from bottom-left, offset each child rightwards and downwards
    pos := fyne.NewPos(0, containerSize.Height-d.MinSize(objects).Height)
    for _, o := range objects {
        size := o.MinSize()
        o.Resize(size)
        o.Move(pos)
        pos = pos.Add(fyne.NewPos(size.Width, size.Height))
    }
}
```

Usage:

```go
w.SetContent(container.New(&diagonal{},
    widget.NewLabel("topleft"),
    widget.NewLabel("Middle Label"),
    widget.NewLabel("bottomright"),
))
```

## 17. Extending Existing Widgets

Extend existing Widget behavior via Go type embedding — no need to create from scratch.

### 17.1 Adding Tap to an Icon

```go
type tappableIcon struct {
    widget.Icon
}

func newTappableIcon(res fyne.Resource) *tappableIcon {
    icon := &tappableIcon{}
    icon.ExtendBaseWidget(icon)  // Note: cannot use widget.NewIcon — it already calls ExtendBaseWidget
    icon.SetResource(res)
    return icon
}

func (t *tappableIcon) Tapped(_ *fyne.PointEvent) {
    log.Println("Icon tapped")
}
```

### 17.2 Numerical Entry (Complete Extension Example)

Override `TypedRune`, `TypedShortcut`, and `Keyboard` to create an entry that only accepts numbers:

```go
import (
    "strconv"
    "fyne.io/fyne/v2/driver/mobile"
    "fyne.io/fyne/v2/widget"
)

type numericalEntry struct {
    widget.Entry
}

func newNumericalEntry() *numericalEntry {
    entry := &numericalEntry{}
    entry.ExtendBaseWidget(entry)
    return entry
}

// Filter keyboard input — only allow digits and decimal separators
func (e *numericalEntry) TypedRune(r rune) {
    if (r >= '0' && r <= '9') || r == '.' || r == ',' {
        e.Entry.TypedRune(r)
    }
}

// Filter paste — only allow if clipboard content parses as a number
func (e *numericalEntry) TypedShortcut(shortcut fyne.Shortcut) {
    paste, ok := shortcut.(*fyne.ShortcutPaste)
    if !ok {
        e.Entry.TypedShortcut(shortcut)
        return
    }
    content := paste.Clipboard.Content()
    if _, err := strconv.ParseFloat(content, 64); err == nil {
        e.Entry.TypedShortcut(shortcut)
    }
}

// Show numeric keyboard on mobile
func (e *numericalEntry) Keyboard() mobile.KeyboardType {
    return mobile.NumberKeyboard
}
```

**Key points for extending Widgets:**
- Call `ExtendBaseWidget(self)` in the constructor, not the original Widget constructor
- Override `TypedRune` to filter keyboard input, `TypedShortcut` to filter shortcuts/paste
- Delegate unhandled input to the embedded Widget's method of the same name

## 18. Advanced Preferences

The `Preferences` API persists user settings, supporting `Bool`, `Float`, `Int`, `String` and their list types.

### 18.1 Basic Usage

```go
// Must use a unique app ID to ensure isolated storage
a := app.NewWithID("com.example.tutorial.preferences")

// Each type has three methods: Get, GetWithFallback, Set
a.Preferences().SetBool("darkMode", true)
a.Preferences().SetFloatList("recentScores", []float64{95.5, 88.0})

value := a.Preferences().IntWithFallback("luckyNumber", 21)
name := a.Preferences().String("username")  // returns "" if not found
```

### 18.2 Complete Example: Remembering User Choice

```go
a := app.NewWithID("com.example.app")
w := a.NewWindow("Settings")

var timeout time.Duration

selector := widget.NewSelect([]string{"10 seconds", "30 seconds", "1 minute"},
    func(selected string) {
        switch selected {
        case "10 seconds":
            timeout = 10 * time.Second
        case "30 seconds":
            timeout = 30 * time.Second
        case "1 minute":
            timeout = time.Minute
        }
        a.Preferences().SetString("AppTimeout", selected)
    })

// Load previous selection, fall back to default
selector.SetSelected(a.Preferences().StringWithFallback("AppTimeout", "10 seconds"))
```

### 18.3 Preference Data Binding

```go
prefs := a.Preferences()
themeBinding := binding.BindPreferenceString("theme", prefs)
// Auto-syncs to Preferences on selection
widget.NewSelectWithData([]string{"light", "dark"}, themeBinding, func(s string) {})
```

## 19. System Tray Lifecycle

A complete system tray app requires managing the window lifecycle, not simply closing.

### 19.1 Close Intercept + Hide/Show

```go
func main() {
    a := app.New()
    w := a.NewWindow("SysTray")

    // Intercept close → hide instead of destroy
    w.SetCloseIntercept(func() {
        w.Hide()
    })

    // Only set up tray on desktop
    if desk, ok := a.(desktop.App); ok {
        m := fyne.NewMenu("MyApp",
            fyne.NewMenuItem("Show", func() {
                w.Show()  // re-show the hidden window
            }))
        desk.SetSystemTrayMenu(m)
        desk.SetSystemTrayIcon(theme.FyneLogo())  // optional: custom icon
    }

    w.SetContent(widget.NewLabel("Fyne System Tray"))
    w.ShowAndRun()
}
```

**Key points:**
- `SetCloseIntercept` overrides the default close behavior (hide instead of exit)
- No need to recreate the window; `Show()` is more efficient than creating a new one
- Type-assert `a.(desktop.App)` to check for desktop feature support
- `app.NewWithID` + `FyneApp.toml` `[Details].Icon` can set the tray icon

## 20. Advanced Multi-Window

### 20.1 Open Multiple Windows on Startup

```go
func main() {
    a := app.New()
    mainWin := a.NewWindow("Main")
    mainWin.SetContent(widget.NewLabel("Main Window"))
    mainWin.Show()

    settingsWin := a.NewWindow("Settings")
    settingsWin.SetContent(widget.NewLabel("Settings"))
    settingsWin.Resize(fyne.NewSize(300, 200))
    settingsWin.Show()

    a.Run()  // exits when all windows are closed
}
```

### 20.2 Setting a Master Window

By default the app exits when all windows close. Use `SetMaster()` to designate a primary window:

```go
mainWin.SetMaster()  // closing the master window → app exits
```

### 20.3 Opening Windows at Runtime

```go
openNew := widget.NewButton("Open Window", func() {
    w3 := a.NewWindow("Third")
    w3.SetContent(widget.NewLabel("Third Window"))
    w3.Show()
})
```

### 20.4 Close Intercept + Unsaved Changes

```go
settingsWin.SetCloseIntercept(func() {
    if hasUnsavedChanges {
        dialog.ShowConfirm("Unsaved Changes",
            "Save before closing?",
            func(ok bool) {
                if ok { save(); settingsWin.Close() }
                // if not saved, do nothing — window stays open
            }, settingsWin)
    } else {
        settingsWin.Close()
    }
})
```

## 21. Animation

v2.0+ Fyne includes an animation framework for smooth transitions of colors, positions, and sizes.

### 21.1 Color Animation

```go
obj := canvas.NewRectangle(color.Black)
red := color.NRGBA{R: 0xff, A: 0xff}
blue := color.NRGBA{B: 0xff, A: 0xff}

canvas.NewColorRGBAAnimation(red, blue, time.Second*2, func(c color.Color) {
    obj.FillColor = c
    canvas.Refresh(obj)  // trigger redraw each frame
}).Start()
```

### 21.2 Position Animation + Auto-Reverse

```go
// When an object's method signature matches the animation callback, pass the method directly
move := canvas.NewPositionAnimation(
    fyne.NewPos(0, 0), fyne.NewPos(200, 0),
    time.Second,
    obj.Move,  // Move(Position) matches the callback signature
)
move.AutoReverse = true  // automatically reverse after reaching the end
move.Start()
```

### 21.3 Size Animation

```go
sizeAnim := canvas.NewSizeAnimation(
    fyne.NewSize(50, 50), fyne.NewSize(200, 100),
    time.Second,
    func(s fyne.Size) { obj.Resize(s); canvas.Refresh(obj) },
)
sizeAnim.Start()
```

### 21.4 Combining Multiple Animations

Multiple animations can run simultaneously without interference:

```go
// Run color transition and position move at the same time
canvas.NewColorRGBAAnimation(red, blue, time.Second*2, func(c color.Color) {
    obj.FillColor = c
    obj.Refresh()
}).Start()

canvas.NewPositionAnimation(
    fyne.NewPos(0, 0), fyne.NewPos(200, 0), time.Second, obj.Move,
).Start()
```

### 21.5 Key Points

- Start animations before `ShowAndRun()`, or dispatch via `fyne.Do`
- Use `container.NewWithoutLayout()` for manually animated objects
- Default curve is `AnimationEaseInOut`; alternatives: `AnimationLinear`, `AnimationEaseIn`, `AnimationEaseOut` (see `references/api-reference.md` Animation section)

## 22. Scaling & Device Adaptation

### 22.1 Coordinate System

Fyne uses device-independent coordinates with a 120DPI baseline (vs. Material Design's 160DPI). This means:
- On a 120DPI monitor, 1 unit ≈ 1 pixel
- On HiDPI (e.g. Retina, ~240DPI), 1 unit ≈ 2 pixels
- On mobile devices, the ratio can be even higher

All `Size` and `Position` values use device-independent coordinates. The framework handles scaling automatically.

### 22.2 Adjusting Scale

```bash
# Environment variable
FYNE_SCALE=1.5 ./myapp    # 50% larger
FYNE_SCALE=0.8 ./myapp    # 20% smaller

# Query current scale
scale := app.Settings().Scale()
```

### 22.3 Adapting Pixel Images

```go
// Images default to MinSize 0 and get squeezed by layouts
// Use SetMinSize to set a device-independent minimum size
img := canvas.NewImageFromFile("photo.jpg")
img.SetMinSize(fyne.NewSize(200, 150))  // consistent size across all screens

// Combine with FillMode for scaling behavior
img.FillMode = canvas.ImageFillCover   // crop to fill
img.FillMode = canvas.ImageFillContain // fit proportionally
```

### 22.4 Mobile Detection

```go
if fyne.CurrentDevice().IsMobile() {
    // Portrait: stack vertically
    w.SetContent(container.NewVBox(items...))
} else {
    // Wide screen: split horizontally
    w.SetContent(container.NewHSplit(sidebar, content))
}
```

`fyne.CurrentDevice().Orientation()` returns `fyne.OrientationVertical` or `fyne.OrientationHorizontal`.
