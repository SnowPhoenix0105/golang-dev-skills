# Complete API Quick Reference

## Table of Contents

**Framework Layer (interfaces/contracts — must understand for custom Widgets):**
- [CanvasObject Interface](#canvasobject-interface)
- [Widget Interface & WidgetRenderer](#widget-interface--widgetrenderer)
- [BaseWidget](#basewidget)
- [Layout List](#layout-list)
- [Event Interfaces](#event-interfaces)
- [Data Binding API](#data-binding-api)
- [App Interface](#app-interface)
- [Window Interface](#window-interface)
- [Canvas Interface](#canvas-interface)
- [OverlayStack](#overlaystack)
- [Theme & Styling — Theme Interface](#theme--styling)
- [Geometry](#geometry) / [Device](#device) / [Resources](#resources) / [Storage](#storage) / [Preferences](#preferences) / [Animations](#animations)

**Component Layer (ready-made constructors):**
- [Widget List](#widget-list)
- [Container List](#container-list)
- [Layout List](#layout-list)
- [Canvas Primitives](#canvas-primitives)
- [Dialog API](#dialog-api)
- [Menu API](#menu-api)
- [Shortcuts](#shortcuts)

**Utility Functions:**
- [Key Functions](#key-functions)

---

## CanvasObject Interface

```go
type CanvasObject interface {
    MinSize() Size
    Move(Position)
    Position() Position
    Resize(Size)
    Size() Size
    Hide()
    Show()
    Visible() bool
    Refresh()
}
```

## Widget Interface & WidgetRenderer

```go
type Widget interface {
    CanvasObject
    CreateRenderer() WidgetRenderer
}

type WidgetRenderer interface {
    Destroy()
    Layout(Size)
    MinSize() Size
    Objects() []CanvasObject
    Refresh()
}
```

`widget.NewSimpleRenderer(object)` can be used for Widgets with only one CanvasObject.

## BaseWidget

Located in the `widget` package, provides standard implementations for position/size/visibility/Refresh.

```go
type BaseWidget struct {
    size     fyne.Size
    position fyne.Position
    Hidden   bool
    // impl       fyne.Widget   // internal field
    // themeCache fyne.Theme    // internal field
}
```

Key method: `ExtendBaseWidget(wid)` — must be called by the constructor of any Widget extending BaseWidget.

`DisableableWidget` extends BaseWidget, adding `Enable()/Disable()/Disabled()`.

---

## Built-in Components (Component API)

All provided by the framework — ready to instantiate. They internally implement the framework interfaces above.

### Widget List

| Widget | Constructor | Data-Binding Constructor | Description |
|--------|------------|--------------------------|-------------|
| Label | `NewLabel(text)` | `NewLabelWithData(data)` | Text label |
| Button | `NewButton(text, fn)` | `NewButtonWithData(name, icon, data, fn)` | Button |
| Entry | `NewEntry()` | `NewEntryWithData(data)` | Single-line input |
| MultiLineEntry | `NewMultiLineEntry()` | - | Multi-line input |
| PasswordEntry | `NewPasswordEntry()` | - | Password input |
| EntryValidation | `NewEntryWithValidation(placeholder, validate)` | - | Validated input, `.AlwaysShowValidationError` (v2.7+) |
| Select | `NewSelect(opts, fn)` | `NewSelectWithData(opts, data, fn)` | Dropdown select |
| SelectEntry | `NewSelectEntry(opts)` | - | Editable dropdown |
| Check | `NewCheck(label, fn)` | `NewCheckWithData(label, data)` | Checkbox |
| CheckGroup | `NewCheckGroup(opts, fn)` | - | Checkbox group |
| RadioGroup | `NewRadioGroup(opts, fn)` | - | Radio group |
| Slider | `NewSlider(min, max)` | `NewSliderWithData(min, max, data)` | Slider |
| ProgressBar | `NewProgressBar()` | - | Progress bar |
| ProgressBarInfinite | `NewProgressBarInfinite()` | - | Infinite progress |
| Toolbar | `NewToolbar(items...)` | - | Toolbar |
| Icon | `NewIcon(res)` | - | Icon |
| Hyperlink | `NewHyperlink(text, url)` | - | Hyperlink |
| Separator | `NewSeparator()` | - | Separator |
| Card | `NewCard(title, sub, content)` | - | Card |
| Accordion | `NewAccordion(items...)` | - | Accordion |
| AccordionItem | `NewAccordionItem(title, detail)` | - | Accordion item |
| Form | `NewForm(items...)` | - | Form |
| FormItem | `NewFormItem(text, widget)` | - | Form item |
| List | `NewList(len, create, update)` | `NewListWithData(data, create, update)` | Virtual list |
| Table | `NewTable(len, create, update)` | - | Virtual table |
| Tree | `NewTree(childUIDs, isBranch, create, update)` | - | Tree |
| FileIcon | `NewFileIcon(uri)` | - | File icon |
| TextGrid | `NewTextGrid()` | - | Monospace text grid |
| Calendar | `NewCalendar(callback)` | - | Calendar |
| DateEntry | `NewDateEntry()` | - | Date input |
| Activity | `NewActivity()` | - | Activity indicator |
| Markdown | `NewMarkdown(content, onHyperlink)` | - | Markdown rendering |
| RichText | `NewRichText(segments...)` | - | Multi-segment rich text |
| RichTextFromMarkdown | `NewRichTextFromMarkdown(content)` | - | Markdown → RichText |
| PopUp | `NewPopUp(content, canvas)` | - | Pop-up layer |
| ModalPopUp | `NewModalPopUp(content, canvas)` | - | Modal pop-up (with overlay) |
| PopUpMenu | `NewPopUpMenu(menu, canvas)` | - | Pop-up menu |
| Selectable | - | - | `label.Selectable = true` selectable text (v2.6+) |
| PartialCheck | - | - | `check.Partial = true` indeterminate state (v2.6+) |

### Container List

```go
// Basic containers
container.New(layout, objects...)
container.NewWithoutLayout(objects...)

// Layout containers
container.NewVBox(objects...)
container.NewHBox(objects...)
container.NewBorder(top, bottom, left, right, center...)
container.NewGridWithColumns(cols, objects...)
container.NewGridWithRows(rows, objects...)
container.NewGridWrap(size, objects...)
container.NewAdaptiveGrid(rowcols, objects...)
container.NewStack(objects...)
container.NewCenter(objects...)
container.NewPadded(objects...)
container.NewMax(obj1, obj2...)                   // v2.7+ largest MinSize

// Special containers
container.NewScroll(content)
container.NewHSplit(left, right)          // offset adjustable
container.NewVSplit(top, bottom)
container.NewAppTabs(items...)
container.NewDocTabs(items...)
container.NewNavigation(content)          // v2.7+ navigation (back button)
container.NewNavigationWithTitle(content, title) // v2.7+ with title
container.NewClip(content)                // v2.7+ clip overflow
container.NewInnerWindow(title, content)

// TabItem
container.NewTabItem(text, content)
container.NewTabItemWithIcon(text, icon, content)
```

### Layout List

```go
layout.NewHBoxLayout()
layout.NewVBoxLayout()
layout.NewBorderLayout(top, bottom, left, right)
layout.NewGridLayout(cols)
layout.NewGridLayoutWithColumns(cols)
layout.NewGridLayoutWithRows(rows)
layout.NewGridWrapLayout(size)
layout.NewAdaptiveGridLayout(rowcols)
layout.NewFormLayout()
layout.NewPaddedLayout()
layout.NewCenterLayout()
layout.NewStackLayout()
layout.NewRowWrapLayout()                                // v2.7+ horizontal wrap
layout.NewRowWrapLayoutWithCustomPadding(hPad, vPad)     // v2.7+ with custom padding
layout.NewMaxLayout()                                    // v2.7+ max child MinSize
layout.NewSpacer()              // flexible spacer (not a Layout, is a CanvasObject)
```

### Canvas Primitives

```go
canvas.NewRectangle(color)
canvas.NewSquare(color)                                    // v2.7+ square
canvas.NewCircle(color)
canvas.NewLine(color)                          // MUST use *canvas.Line
canvas.NewText("text", color)                  // MUST use *canvas.Text
canvas.NewImageFromResource(res)
canvas.NewImageFromFile(path)
canvas.NewImageFromImage(img)                  // from image.Image
canvas.NewImageFromReader(reader, name)
canvas.NewRaster(width, height, drawFn)
canvas.NewRasterWithDetails(w, h, pxColorFn)
canvas.NewLinearGradient(start, end)
canvas.NewHorizontalGradient(start, end)
canvas.NewVerticalGradient(start, end)
canvas.NewRadialGradient(center, radius)
canvas.NewCircleGradient(center, radius)
canvas.NewPolygon(sides, color)                            // v2.7+ regular polygon
canvas.NewRegularPolygon(sides, color)                     // v2.7+ regular polygon (centered)
canvas.NewArbitraryPolygon(points...)
canvas.NewArc(startAngle, endAngle, cutoutRatio, color)    // v2.7+ arc (0=pie, 0.5=doughnut, 1=point)
canvas.NewPieArc(startAngle, endAngle, color)              // v2.7+ pie slice (cutout=0)
canvas.NewDoughnutArc(startAngle, endAngle, color)         // v2.7+ doughnut (cutout=0.5)
canvas.NewBezierCurve(p0, p1, p2)             // bezier curve
canvas.NewBlur(target)                         // blur effect (requires GPU)
```

### Rectangle Properties (v2.7+)

```go
rect := canvas.NewRectangle(color)
rect.CornerRadius = 10              // uniform radius
rect.TopLeftCornerRadius = 8        // per-corner radius
rect.TopRightCornerRadius = 8
rect.BottomLeftCornerRadius = 4
rect.BottomRightCornerRadius = 4
rect.Aspect = 1.6                   // aspect ratio (constrains size when non-zero)
// CornerRadius = min(width,height)/2 → "pill" shape
```

### ImageFill Modes (v2.7+)

```go
img := canvas.NewImageFromFile("photo.jpg")
img.FillMode = canvas.ImageFillStretch    // stretch to fill (default)
img.FillMode = canvas.ImageFillContain   // fit (maintain aspect, may leave gaps)
img.FillMode = canvas.ImageFillOriginal  // original size (container fits image)
img.FillMode = canvas.ImageFillCover     // cover (crop to fill while maintaining aspect)
img.CornerRadius = 8                     // image corner radius
```

## Data Binding API

### Value Types

```go
binding.NewBool()              → Bool
binding.NewFloat()             → Float
binding.NewInt()               → Int
binding.NewString()            → String
binding.NewURI()               → URI
binding.NewUntyped()           → Untyped (any)

// Methods: Get() / Set(val) / AddListener(l) / RemoveListener(l)
```

### Collection Types

```go
binding.NewBoolList()          → BoolList
binding.NewFloatList()         → FloatList
binding.NewIntList()           → IntList
binding.NewStringList()        → StringList

// Methods: Get() / Set([]T) / GetValue(index) / SetValue(index, val) / Append / Length
```

### External Bindings

```go
binding.BindBool(&var)         → ExternalBool
binding.BindFloat(&var)        → ExternalFloat
binding.BindInt(&var)          → ExternalInt
binding.BindString(&var)       → ExternalString
binding.BindUntyped(&var)      → ExternalUntyped

// Note: call .Reload() after modifying the external variable to notify listeners
```

### Collection External Bindings

```go
binding.BindBoolList(&var)
binding.BindFloatList(&var)
binding.BindIntList(&var)
binding.BindStringList(&var)
```

### Converters

```go
binding.IntToString(Int)            → String
binding.FloatToString(Float)        → String
binding.BoolToString(Bool)          → String
binding.StringToInt(String)         → Int
binding.StringToFloat(String)       → Float
binding.StringToBool(String)        → Bool
binding.URIToString(URI)            → String
binding.StringToURI(String)         → URI
binding.FormatFloat(Float, format)  → String
binding.FormatInt(Int, format)      → String
binding.SPrintf(format, args...)    → String (generic formatting)
```

### Preferences Bindings

```go
binding.BindPreferenceBool(key, prefs)  → Bool (two-way sync with Preferences)
binding.BindPreferenceFloat(key, prefs) → Float
binding.BindPreferenceInt(key, prefs)   → Int
binding.BindPreferenceString(key, prefs)→ String
```

### Tree/Map Bindings

```go
binding.NewStringTree()
binding.NewStringMap()
```

## Event Interfaces

```go
// Pointer events
Tappable          interface { Tapped(*PointEvent) }
DoubleTappable    interface { DoubleTapped(*PointEvent) }
SecondaryTappable interface { TappedSecondary(*PointEvent) }

// Keyboard events
Focusable         interface { FocusGained(); FocusLost(); TypedRune(rune); TypedKey(*KeyEvent) }
Tabbable          interface { AcceptsTab() bool }

// Other
Scrollable        interface { Scrolled(*ScrollEvent) }
Draggable         interface { Dragged(*DragEvent); DragEnd() }
Disableable       interface { Enable(); Disable(); Disabled() bool }
Shortcutable      interface { TypedShortcut(Shortcut) }
```

### Event Structs

```go
type PointEvent struct {
    AbsolutePosition Position
    Position         Position    // relative to the triggered object
}

type KeyEvent struct {
    Name     KeyName            // cross-platform key name
    Physical HardwareKey
}

type ScrollEvent struct {
    PointEvent
    Scrolled Delta
}

type DragEvent struct {
    PointEvent
    Dragged Delta
}
```

### KeyName Constants

Direction: `KeyUp, KeyDown, KeyLeft, KeyRight`
Function: `KeyF1..KeyF12, KeyEscape, KeyReturn, KeyTab, KeyBackspace, KeyInsert, KeyDelete`
Navigation: `KeyPageUp, KeyPageDown, KeyHome, KeyEnd, KeySpace, KeyEnter`
Letters: `KeyA..KeyZ`
Digits: `Key0..Key9`
Symbols: `KeyApostrophe, KeyComma, KeyMinus, KeyPeriod, KeySlash, KeyBackslash, KeyLeftBracket, KeyRightBracket, KeySemicolon, KeyEqual, KeyBackTick`

### KeyModifier

```go
KeyModifierShift   = 1 << iota
KeyModifierControl
KeyModifierAlt
KeyModifierSuper
```

## Shortcuts

```go
type Shortcut interface { ShortcutName() string }
type KeyboardShortcut interface {
    Shortcut
    Key() KeyName
    Mod() KeyModifier
}

// Built-in shortcuts
&ShortcutCopy{Clipboard}
&ShortcutPaste{Clipboard}
&ShortcutCut{Clipboard}
&ShortcutSelectAll{}
&ShortcutUndo{}
&ShortcutRedo{}

// Canvas-level shortcuts
canvas.AddShortcut(shortcut, handler)
canvas.RemoveShortcut(shortcut)
```

## Dialog API

```go
// Information
dialog.ShowInformation(title, msg, parent)
dialog.ShowError(err, parent)
dialog.ShowConfirm(title, msg, callback, parent)

// Input
dialog.ShowEntry(title, msg, parent)
dialog.ShowText(title, msg, parent)
dialog.ShowForm(title, submit, cancel, items, callback, parent)
dialog.ShowCustom(title, dismiss, content, parent)

// File
dialog.ShowFileOpen(callback, parent)
dialog.ShowFileSave(callback, parent)
dialog.ShowFolderOpen(callback, parent)

// Color
dialog.ShowColorPicker(title, msg, callback, parent)

// Progress
dialog.NewProgress(title, msg, parent)
dialog.NewProgressInfinite(title, msg, parent)
```

## Menu API

```go
// Menu items
fyne.NewMenuItem(label, action)
fyne.NewMenuItemWithIcon(label, icon, action)
fyne.NewMenuItemSeparator()

// MenuItem properties
item.ChildMenu = subMenu
item.Icon = iconResource
item.Shortcut = shortcut
item.Disabled = true
item.Checked = true
item.IsQuit = true

// Menus
fyne.NewMenu(label, items...)
menu.Refresh()

// Main menu bar
mainMenu := fyne.NewMainMenu(menus...)
window.SetMainMenu(mainMenu)
mainMenu.Refresh()
```

## Menu/Dialog Close Signal (v2.6+)

```go
menuItem.Action = func() { ... }
// If no Action is set, menu items with Shortcuts automatically trigger the Shortcut
```

## App Interface

```go
app.New()                                    // create app
app.NewWithID("com.example.app")            // with unique ID

// App methods
app.NewWindow(title) Window
app.Run()                                    // blocking, starts event loop
app.Quit()
app.Driver() Driver
app.Settings() Settings
app.Preferences() Preferences
app.Storage() Storage
app.Lifecycle() Lifecycle
app.Clipboard() Clipboard
app.Cache() Cache
app.OpenURL(url) error
app.SendNotification(*Notification)
app.ScheduleNotification(*Notification, time.Time) (*ScheduledNotification, error)
app.CancelScheduledNotification(id string) error
app.SetIcon(Resource)
app.Icon() Resource
app.CloudProvider() CloudProvider
app.SetCloudProvider(CloudProvider)
app.Metadata() AppMetadata
app.UniqueID() string
```

## Window Interface

```go
window.Title() string
window.SetTitle(string)
window.FullScreen() bool
window.SetFullScreen(bool)
window.Resize(Size)
window.RequestFocus()
window.FixedSize() bool
window.SetFixedSize(bool)
window.CenterOnScreen()
window.Padded() bool
window.SetPadded(bool)
window.Icon() Resource
window.SetIcon(Resource)
window.SetMaster()
window.MainMenu() *MainMenu
window.SetMainMenu(*MainMenu)
window.SetOnClosed(func())
window.SetCloseIntercept(func())          // intercept close, manually call window.Close()
window.SetOnDropped(func(Position, []URI))  // drop callback
window.Show()
window.Hide()
window.Close()
window.ShowAndRun()                       // Show() + Run()
window.Content() CanvasObject
window.SetContent(CanvasObject)
window.Canvas() Canvas
window.Clipboard() Clipboard              // Deprecated: use App.Clipboard()
```

## Canvas Interface

```go
canvas.Content() CanvasObject
canvas.SetContent(CanvasObject)
canvas.Refresh(CanvasObject)
canvas.Focus(Focusable)
canvas.FocusNext()
canvas.FocusPrevious()
canvas.Unfocus()
canvas.Focused() Focusable
canvas.Size() Size
canvas.Scale() float32                       // current display scale factor (120DPI baseline)
canvas.Overlays() OverlayStack
canvas.OnTypedRune() func(rune)
canvas.SetOnTypedRune(func(rune))
canvas.OnTypedKey() func(*KeyEvent)
canvas.SetOnTypedKey(func(*KeyEvent))
canvas.AddShortcut(shortcut, handler)
canvas.RemoveShortcut(shortcut)
canvas.Capture() image.Image                 // capture Canvas as image
canvas.PixelCoordinateForPosition(Position) (int, int) // physical pixel conversion
canvas.InteractiveArea() (Position, Size)   // v1.4+
```

## OverlayStack

```go
type OverlayStack interface {
    Add(overlay CanvasObject)
    List() []CanvasObject
    Remove(overlay CanvasObject)
    Top() CanvasObject
}

canvas.Overlays().Add(widget.NewPopUp(...))
```

### PopUp Components

```go
// Modal pop-up (overlay + auto-close on outside click)
pop := widget.NewModalPopUp(content, canvas)
pop.Show()

// Plain PopUp (no overlay, manual Hide)
pop := widget.NewPopUp(content, canvas)
pop.Show()
pop.Hide()

// Pop-up menu (shown at a position)
menu := fyne.NewMenu("", fyne.NewMenuItem("Action", func() {}))
pop := widget.NewPopUpMenu(menu, canvas)
pop.ShowAtPosition(fyne.NewPos(100, 200))
```

### Desktop Extensions (driver/desktop)

```go
import "fyne.io/fyne/v2/driver/desktop"

// Hoverable interface (MouseIn / MouseOut / MouseMoved)
// Custom Widgets implement this to respond to mouse hover
func (w *MyWidget) MouseIn(ev *desktop.MouseEvent)   { /* mouse enter */ }
func (w *MyWidget) MouseOut()                         { /* mouse leave */ }
func (w *MyWidget) MouseMoved(ev *desktop.MouseEvent) { /* mouse move */ }

// Custom desktop shortcuts
canvas.AddShortcut(&desktop.CustomShortcut{
    KeyName:  fyne.KeyS,
    Modifier: desktop.KeyModifierControl | desktop.KeyModifierShift,
}, func(shortcut fyne.Shortcut) { /* Save As... */ })

// System tray
if desk, ok := app.(desktop.App); ok {
    desk.SetSystemTrayMenu(fyne.NewMenu("Tray",
        fyne.NewMenuItem("Show", func() { w.Show() }),
        fyne.NewMenuItem("Quit", func() { app.Quit() }),
    ))
}
```

### Migration Checks

```bash
# Scan for deprecated APIs
rg "NewContainer\b" .                  # → container.New / NewWithoutLayout
rg "NewContainerWithLayout\b" .        # → container.New
rg "DarkTheme\|LightTheme" .           # → theme.DefaultTheme() (removed in v3.0)
rg "fyne\.Do\b" .                      # v2.6+ check for misuse on main goroutine
```

## Geometry

```go
type Position struct { X, Y float32 }
func NewPos(x, y float32) Position
func NewSquareOffsetPos(length float32) Position  // v2.4+
(p Position) Add(Vector2) Position
(p Position) AddXY(x, y float32) Position
(p Position) Subtract(Vector2) Position
(p Position) SubtractXY(x, y float32) Position
(p Position) Components() (float32, float32)
(p Position) IsZero() bool

type Size struct { Width, Height float32 }
func NewSize(w, h float32) Size
func NewSquareSize(side float32) Size            // v2.4+
(s Size) Max(Vector2) Size
(s Size) Min(Vector2) Size
(s Size) Add(Vector2) Size
(s Size) Subtract(Vector2) Size
(s Size) AddWidthHeight(w, h float32) Size
(s Size) SubtractWidthHeight(w, h float32) Size
(s Size) Components() (float32, float32)
(s Size) IsZero() bool

type Delta struct { DX, DY float32 }
```

## Device

```go
type Device interface {
    Orientation() DeviceOrientation
    IsMobile() bool
    IsBrowser() bool
    HasKeyboard() bool
    SystemScaleForWindow(Window) float32
    Locale() Locale
}

fyne.CurrentDevice()

// DeviceOrientation
OrientationVertical / OrientationVerticalUpsideDown
OrientationHorizontalLeft / OrientationHorizontalRight

fyne.IsVertical(orient) bool
fyne.IsHorizontal(orient) bool
```

## Resources

```go
type Resource interface {
    Name() string
    Content() []byte
}

type ThemedResource interface {      // v2.5+
    Resource
    ThemeColorName() ThemeColorName
}

// Creating resources
fyne.NewStaticResource(name, content []byte)
fyne.LoadResourceFromPath(path)
fyne.LoadResourceFromURLString(url)
fyne.CacheResourceFromURLString(url) // v2.8+ auto-cache

// Theme-adapted resources (icon variants for different theme colors)
theme.NewThemedResource(res)           // foreground color
theme.NewDisabledThemedResource(res)   // disabled color
theme.NewErrorThemedResource(res)      // error color
theme.NewInvertedThemedResource(res)   // inverted (light foreground)
theme.NewPrimaryThemedResource(res)    // primary color

// Embedding resources
//go:generate fyne bundle -o bundled.go icon.png
```

## Preferences

```go
prefs := app.Preferences()
prefs.Bool(key) / SetBool(key, val) / BoolWithFallback(key, fallback)
prefs.Int(key) / SetInt(key, val) / IntWithFallback(key, fallback)
prefs.Float(key) / SetFloat(key, val) / FloatWithFallback(key, fallback)
prefs.String(key) / SetString(key, val) / StringWithFallback(key, fallback)
// v2.4+ lists
prefs.BoolList(key) / SetBoolList(key, val) / BoolListWithFallback(key, fallback)
prefs.FloatList(key) / SetFloatList(key, val) / FloatListWithFallback(key, fallback)
prefs.IntList(key) / SetIntList(key, val) / IntListWithFallback(key, fallback)
prefs.StringList(key) / SetStringList(key, val) / StringListWithFallback(key, fallback)
prefs.RemoveValue(key)
prefs.AddChangeListener(func())
prefs.ChangeListeners() []func()
```

## Storage

```go
type Storage interface {
    RootURI() URI
    Create(name string) (URIWriteCloser, error)
    Open(name string) (URIReadCloser, error)
    Save(name string) (URIWriteCloser, error)
    Remove(name string) error
    RemoveAll(name string) error  // v2.7+ recursive delete
    List() []string
}
```

## Animations

```go
type Animation struct {
    AutoReverse bool
    Curve       AnimationCurve
    Duration    time.Duration
    RepeatCount int
    Tick        func(float32)
}

fyne.NewAnimation(duration, tickFn)
anim.Start() / anim.Stop()

// Built-in curves
fyne.AnimationLinear
fyne.AnimationEaseIn
fyne.AnimationEaseOut
fyne.AnimationEaseInOut  // default
fyne.AnimationRepeatForever // = -1
```

## Key Functions

```go
// Thread dispatch
fyne.Do(fn)              // non-blocking, dispatch to main goroutine
fyne.DoAndWait(fn)       // blocking, wait for fn to complete

// App
fyne.CurrentApp() App
fyne.CurrentDevice() Device

// Logging
fyne.LogError(reason, err)

// Notifications
fyne.NewNotification(title, content)

// Math
fyne.Min(a, b float32) float32
fyne.Max(a, b float32) float32
```

## Theme & Styling

### TextStyle

```go
type TextStyle struct {
    Bold          bool  // bold
    Italic        bool  // italic
    Monospace     bool  // monospace font
    Symbol        bool  // symbol font (v2.2+)
    TabWidth      int   // tab width in spaces (v2.1+)
    Underline     bool  // underline (v2.5+)
    Strikethrough bool  // strikethrough (v2.8+)
}
```

### TextAlign

```go
TextAlignLeading  / TextAlignCenter / TextAlignTrailing
```

### TextWrap

```go
TextWrapOff      // no wrap, expand width
TextWrapBreak    // wrap by character
TextWrapWord     // wrap by word
```

### TextTruncation (v2.4+)

```go
TextTruncateOff       // no truncation (default)
TextTruncateClip      // clip
TextTruncateEllipsis  // add ellipsis at end
```

### fyne.MeasureText

```go
size := fyne.MeasureText("Hello", 14, fyne.TextStyle{Bold: true})
// returns fyne.Size{Width, Height}
```

### Theme Interface

```go
type Theme interface {
    Color(ThemeColorName, ThemeVariant) color.Color
    Font(TextStyle) Resource
    Icon(ThemeIconName) Resource
    Size(ThemeSizeName) float32
}
```

### ThemeVariant

```go
theme.VariantLight  // light
theme.VariantDark   // dark
```

### Getting Current Theme

```go
theme.Current()                // get current app theme
theme.CurrentForWidget(w)      // get theme for a specific Widget (considers ThemeOverride)
```

### Theme Convenience Functions (v2.5+, use directly without Theme object)

```go
theme.Color(name)              // current theme color
theme.ColorForWidget(name, w)  // Widget-specific theme color
theme.Size(name)               // current theme size
theme.SizeForWidget(name, w)   // Widget-specific theme size
theme.Font(style)              // current theme font
```

### Predefined Themes

```go
theme.DefaultTheme()    // adaptive light/dark (recommended)
theme.LightTheme()      // light (Deprecated, removed in v3.0)
theme.DarkTheme()       // dark (Deprecated, removed in v3.0)
```

### Primary Color Name Constants

```go
theme.ColorRed     theme.ColorOrange   theme.ColorYellow
theme.ColorGreen   theme.ColorBlue     theme.ColorPurple
theme.ColorBrown   theme.ColorGray

// Get all available color names
theme.PrimaryColorNames()  // []string
```

Use `app.Settings().PrimaryColor()` to get/set the user's preferred primary color.

### Complete ThemeColorName Constants

```go
// Basics
theme.ColorNamePrimary              // primary color
theme.ColorNameBackground           // background
theme.ColorNameForeground           // foreground (text/icons)
theme.ColorNameForegroundOnPrimary  // contrast on primary (v2.5+)
theme.ColorNameForegroundOnError    // contrast on error (v2.5+)
theme.ColorNameForegroundOnSuccess  // contrast on success (v2.5+)
theme.ColorNameForegroundOnWarning  // contrast on warning (v2.5+)

// Interactive states
theme.ColorNameButton               // button color
theme.ColorNameDisabledButton       // disabled button
theme.ColorNameDisabled             // disabled foreground
theme.ColorNameHover                // hover highlight
theme.ColorNamePressed              // press highlight
theme.ColorNameFocus                // focus highlight
theme.ColorNameSelection            // selection highlight (v2.1+)

// Input components
theme.ColorNameInputBackground      // input background
theme.ColorNameInputBorder          // input border (v2.3+)
theme.ColorNamePlaceHolder          // placeholder text

// Semantic colors
theme.ColorNameError                // error
theme.ColorNameSuccess              // success (v2.3+)
theme.ColorNameWarning              // warning (v2.3+)
theme.ColorNameHyperlink            // hyperlink (v2.4+)

// Component colors
theme.ColorNameScrollBar            // scroll bar
theme.ColorNameScrollBarBackground  // scroll bar background (v2.6+)
theme.ColorNameShadow               // shadow
theme.ColorNameSeparator            // separator (v2.3+)
theme.ColorNameHeaderBackground     // table/list header background (v2.4+)
theme.ColorNameMenuBackground       // menu background (v2.3+)
theme.ColorNameOverlayBackground    // overlay background (dialogs etc.) (v2.3+)

// Inner windows (v2.8+)
theme.ColorNameInnerWindowBorder           // inner window border
theme.ColorNameInnerWindowBorderInactive   // inactive inner window border
```

### Complete ThemeSizeName Constants

```go
// Text sizes
theme.SizeNameText            // body 14px
theme.SizeNameHeadingText     // heading 24px (v2.1+)
theme.SizeNameSubHeadingText  // subheading 18px (v2.1+)
theme.SizeNameCaptionText     // caption 11px

// Spacing
theme.SizeNamePadding              // outer padding 4px
theme.SizeNameInnerPadding         // inner padding 8px (v2.3+)
theme.SizeNameLineSpacing          // line spacing 4px (v2.3+)
theme.SizeNameInlineIcon           // inline icon 20px
theme.SizeNameSeparatorThickness   // separator thickness 1px

// Input components
theme.SizeNameInputBorder          // input border 1px
theme.SizeNameInputRadius          // input radius 5px (v2.4+)
theme.SizeNameSelectionRadius      // selection radius 3px (v2.4+)

// Scroll bars
theme.SizeNameScrollBar            // scroll bar 12px
theme.SizeNameScrollBarSmall       // small scroll bar 3px
theme.SizeNameScrollBarRadius      // scroll bar radius 3px (v2.5+)

// Buttons (v2.8+)
theme.SizeNameButtonRadius         // button radius 5px

// Inner windows (v2.6+)
theme.SizeNameWindowButtonHeight   // window button height 16px
theme.SizeNameWindowButtonRadius   // window button radius 8px
theme.SizeNameWindowButtonIcon     // window button icon 14px
theme.SizeNameWindowTitleBarHeight // title bar height 26px
theme.SizeNameInnerWindowRadius    // window radius 5px (v2.8+)

// Other
theme.SizeNameSplitThickness       // split panel thickness (v2.8+)
theme.SizeNameModalBlurRadius      // modal blur radius (v2.7+)
```

### Size Convenience Functions

```go
theme.Padding()              // outer padding
theme.InnerPadding()         // inner padding
theme.TextSize()             // body text size
theme.TextHeadingSize()      // heading text size
theme.TextSubHeadingSize()   // subheading text size
theme.CaptionTextSize()      // caption text size
theme.IconInlineSize()       // inline icon size
theme.ScrollBarSize()        // scroll bar size
theme.ScrollBarSmallSize()   // small scroll bar size
theme.InputBorderSize()      // input border size
theme.InputRadiusSize()      // input radius size
theme.SelectionRadiusSize()  // selection radius size
theme.LineSpacing()          // line spacing
theme.SeparatorThicknessSize() // separator thickness
```

### Complete ThemeIconName Constants

```go
// Navigation / actions
theme.IconNameCancel        theme.IconNameConfirm      theme.IconNameDelete
theme.IconNameSearch        theme.IconNameSearchReplace theme.IconNameMenu
theme.IconNameMenuExpand

// Form controls
theme.IconNameCheckButton              // unchecked
theme.IconNameCheckButtonChecked       // checked
theme.IconNameCheckButtonFill          // filled (v2.5+)
theme.IconNameCheckButtonPartial       // partial (v2.6+)
theme.IconNameRadioButton              // unchecked
theme.IconNameRadioButtonChecked       // checked
theme.IconNameRadioButtonFill          // filled (v2.5+)

// Color
theme.IconNameColorAchromatic   theme.IconNameColorChromatic   theme.IconNameColorPalette

// Content operations
theme.IconNameContentAdd     theme.IconNameContentRemove    theme.IconNameContentCut
theme.IconNameContentCopy    theme.IconNameContentPaste     theme.IconNameContentClear
theme.IconNameContentRedo    theme.IconNameContentUndo

// Status
theme.IconNameInfo          theme.IconNameQuestion       theme.IconNameWarning
theme.IconNameError         theme.IconNameBrokenImage    // v2.4+

// Document
theme.IconNameDocument        theme.IconNameDocumentCreate
theme.IconNameDocumentPrint   theme.IconNameDocumentSave

// Mail
theme.IconNameMailAttachment  theme.IconNameMailCompose   theme.IconNameMailForward
theme.IconNameMailReply       theme.IconNameMailReplyAll  theme.IconNameMailSend

// Media
theme.IconNameMediaMusic       theme.IconNameMediaPhoto       theme.IconNameMediaVideo
theme.IconNameMediaFastForward theme.IconNameMediaFastRewind
theme.IconNameMediaPause       theme.IconNameMediaPlay
theme.IconNameMediaSkipNext    theme.IconNameMediaSkipPrevious
theme.IconNameMediaRecord      theme.IconNameMediaReplay
theme.IconNameMediaStop

// Navigation
theme.IconNameNavigateBack     theme.IconNameNavigateNext
theme.IconNameArrowDropDown    theme.IconNameArrowDropUp
theme.IconNameMoreHorizontal   theme.IconNameMoreVertical

// File
theme.IconNameFile             theme.IconNameFileApplication
theme.IconNameFileAudio        theme.IconNameFileImage
theme.IconNameFileVideo        theme.IconNameFileText

// View
theme.IconNameViewFullScreen   theme.IconNameViewRefresh
theme.IconNameViewZoomFit      theme.IconNameViewZoomIn
theme.IconNameViewZoomOut

// Account
theme.IconNameAccount          theme.IconNameLogin
theme.IconNameLogout

// List
theme.IconNameList             theme.IconNameGrid

// Inner windows (v2.5+)
theme.IconNameDragCornerIndicator

// Other
theme.IconNameComputer         theme.IconNameDownload
theme.IconNameFavorite         theme.IconNameHelp
theme.IconNameHome             theme.IconNameMove
theme.IconNameSettings         theme.IconNameStorage
theme.IconNameUpload           theme.IconNameVisibility
theme.IconNameVisibilityOff    theme.IconNameVolumeDown
theme.IconNameVolumeMute       theme.IconNameVolumeUp
```

### Icon Convenience Functions (return fyne.Resource)

```go
theme.CancelIcon()               theme.ConfirmIcon()
theme.DeleteIcon()               theme.SearchIcon()
theme.MenuIcon()                 theme.MenuExpandIcon()
theme.CheckButtonIcon()          theme.CheckButtonCheckedIcon()
theme.RadioButtonIcon()          theme.RadioButtonCheckedIcon()
theme.InfoIcon()                 theme.QuestionIcon()
theme.WarningIcon()              theme.ErrorIcon()
theme.DocumentIcon()             theme.DocumentCreateIcon()
theme.DocumentPrintIcon()        theme.DocumentSaveIcon()
theme.MailComposeIcon()          theme.MailSendIcon()
theme.MediaPlayIcon()            theme.MediaPauseIcon()
theme.MediaRecordIcon()          theme.MediaStopIcon()
theme.MediaFastForwardIcon()     theme.MediaFastRewindIcon()
theme.ContentAddIcon()           theme.ContentRemoveIcon()
theme.ContentCutIcon()           theme.ContentCopyIcon()
theme.ContentPasteIcon()         theme.ContentUndoIcon()
theme.ContentRedoIcon()          theme.NavigateBackIcon()
theme.NavigateNextIcon()         theme.ViewFullScreenIcon()
theme.ViewRefreshIcon()          theme.AccountIcon()
theme.LoginIcon()                theme.LogoutIcon()
theme.SettingsIcon()             theme.HomeIcon()
theme.HelpIcon()                 theme.DownloadIcon()
theme.UploadIcon()               theme.ComputerIcon()
theme.StorageIcon()              theme.FileIcon()
theme.FileApplicationIcon()      theme.FileAudioIcon()
theme.FileImageIcon()            theme.FileVideoIcon()
theme.FileTextIcon()             theme.FavoriteIcon()
theme.ListIcon()                 theme.GridIcon()
theme.VisibilityIcon()           theme.VisibilityOffIcon()
```

### Font Convenience Functions

```go
theme.TextFont()                // regular font
theme.TextBoldFont()            // bold
theme.TextItalicFont()          // italic
theme.TextBoldItalicFont()      // bold italic
theme.TextMonospaceFont()       // monospace
theme.SymbolFont()              // symbol font
theme.DefaultEmojiFont()        // emoji font

// Default font resources
theme.DefaultTextFont()
theme.DefaultTextBoldFont()
theme.DefaultTextBoldItalicFont()
theme.DefaultTextItalicFont()
theme.DefaultTextMonospaceFont()
theme.DefaultSymbolFont()
```

### Custom Fonts (Environment Variables)

```bash
FYNE_FONT=/path/to/font.ttf         # main font
FYNE_FONT_MONOSPACE=/path/to/mono.ttf  # monospace font
FYNE_FONT_SYMBOL=/path/to/symbol.ttf   # symbol font
```

### Full Custom Theme Template

```go
type MyTheme struct{}

func (t MyTheme) Color(name fyne.ThemeColorName, v fyne.ThemeVariant) color.Color {
    switch name {
    case theme.ColorNamePrimary:
        return color.NRGBA{R: 70, G: 130, B: 255, A: 255}
    case theme.ColorNameBackground:
        if v == theme.VariantLight {
            return color.White
        }
        return color.NRGBA{R: 0x17, G: 0x17, B: 0x18, A: 255}
    default:
        return theme.DefaultTheme().Color(name, v) // fallback to default
    }
}

func (t MyTheme) Font(style fyne.TextStyle) fyne.Resource {
    if style.Monospace {
        return resourceMyMonoFont  // custom monospace font
    }
    return theme.DefaultTheme().Font(style)
}

func (t MyTheme) Icon(name fyne.ThemeIconName) fyne.Resource {
    return theme.DefaultTheme().Icon(name)
}

func (t MyTheme) Size(name fyne.ThemeSizeName) float32 {
    switch name {
    case theme.SizeNameText:
        return 16 // custom body text size
    case theme.SizeNamePadding:
        return 6
    default:
        return theme.DefaultTheme().Size(name)
    }
}
```

### Applying Themes

```go
// Global theme
a.Settings().SetTheme(&MyTheme{})

// Local theme override (only affects subtree within container)
container.NewThemeOverride(&MyTheme{}, content)
```
