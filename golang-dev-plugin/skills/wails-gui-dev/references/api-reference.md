# Wails API Reference (v2 + v3)

This document provides the complete API listing for Wails v2 and v3. Use for quick lookup.

## Table of Contents

- [App Creation Options](#app-creation-options)
  - [v2 options.App](#v2-optionsapp)
  - [v3 application.Options](#v3-applicationoptions)
- [Services & Binding](#services--binding)
  - [v2 Bind Mode](#v2-bind-mode)
  - [v3 Service Mode](#v3-service-mode)
- [Window Options (v3)](#window-options-v3)
  - [WebviewWindowOptions](#webviewwindowoptions)
  - [MacWindow](#macwindow)
  - [WindowsWindow](#windowswindow)
  - [LinuxWindow](#linuxwindow)
- [Platform Options](#platform-options)
  - [v3 MacOptions](#v3-macoptions)
  - [v3 WindowsOptions](#v3-windowsoptions)
  - [v3 LinuxOptions](#v3-linuxoptions)
  - [v3 IOSOptions](#v3-iosoptions)
  - [v3 AndroidOptions](#v3-androidoptions)
- [v2 Runtime API Full List](#v2-runtime-api-full-list)
- [v3 App/Window Methods Full List](#v3-appwindow-methods-full-list)
- [v3 Dialog API](#v3-dialog-api)
- [v3 Event API Full List](#v3-event-api-full-list)
- [v3 Menu API](#v3-menu-api)
- [JS Frontend Runtime API](#js-frontend-runtime-api)

---

## App Creation Options

### v2 options.App

```go
type App struct {
    Title             string          // Window title
    Width             int             // Initial width
    Height            int             // Initial height
    DisableResize     bool            // Disable resizing
    Fullscreen        bool            // Start fullscreen
    Frameless         bool            // No window frame
    MinWidth          int             // Minimum width
    MinHeight         int             // Minimum height
    MaxWidth          int             // Maximum width (0 = unlimited)
    MaxHeight         int             // Maximum height (0 = unlimited)
    StartHidden       bool            // Hidden at startup
    HideWindowOnClose bool            // Hide instead of quit on close
    AlwaysOnTop       bool            // Keep window on top
    BackgroundColour  *RGBA           // Background color (NewRGBA/NewRGB)
    Assets            fs.FS           // [Deprecated] Use AssetServer.Assets
    AssetsHandler     http.Handler    // [Deprecated] Use AssetServer.Handler
    AssetServer       *assetserver.Options
    Menu              *menu.Menu      // Application menu
    Logger            logger.Logger   // Logger instance
    LogLevel          logger.LogLevel // Log level
    LogLevelProduction logger.LogLevel
    OnStartup         func(ctx context.Context)
    OnDomReady        func(ctx context.Context)
    OnShutdown        func(ctx context.Context)
    OnBeforeClose     func(ctx context.Context) (prevent bool)
    Bind              []interface{}   // Bound structs
    EnumBind          []interface{}   // Enum values exposed
    WindowStartState  WindowStartState // Normal/Maximised/Minimised/Fullscreen
    ErrorFormatter    ErrorFormatter  // func(error) any
    CSSDragProperty   string          // Default "--wails-draggable"
    CSSDragValue      string          // Default "drag"
    EnableDefaultContextMenu bool
    EnableFraudulentWebsiteDetection bool
    SingleInstanceLock *SingleInstanceLock
    Windows *windows.Options
    Mac     *mac.Options
    Linux   *linux.Options
    Experimental    *Experimental
    Debug           Debug
    DragAndDrop     *DragAndDrop
    DisablePanicRecovery bool
    BindingsAllowedOrigins string
}
```

#### v2 Platform-Specific Options

**windows.Options**:
```go
type Options struct {
    WebviewIsTransparent              bool
    WindowIsTranslucent               bool
    DisableWindowIcon                 bool
    DisableFramelessWindowDecorations bool
    Theme                             Theme // SystemDefault/Dark/Light
    CustomTheme                       *ThemeSettings
    WebviewUserDataPath               string
    WebviewBrowserPath                string
    WebviewGpuIsDisabled              bool
    ResizeDebounceMS                  uint16
    OnSuspend  func()
    OnResume   func()
    WndProcInterceptor func(hwnd uintptr, msg uint32, wparam, lparam uintptr) (shouldReturn bool, returnCode uintptr)
}
```

**mac.Options**:
```go
type Options struct {
    TitleBar                    *TitleBar
    Appearance                  AppearanceType
    OnFileOpen                  func(filePath string)
    OnUrlOpen                   func(url string)
    OnReopen                    func()
    WebviewIsTransparent        bool
    WindowIsTranslucent         bool
    About                       *About
    OnSuspend  func()
    OnResume   func()
    Preferences                 *Preferences
}
```

**linux.Options**:
```go
type Options struct {
    Icon                []byte
    WindowIsTranslucent bool
    ProgramName         string
    DisableQuitOnLastWindowClosed bool
}
```

### v3 application.Options

```go
type Options struct {
    Name        string        // Application name
    Description string        // Application description
    Icon        []byte        // Application icon
    Mac         MacOptions
    Windows     WindowsOptions
    Linux       LinuxOptions
    IOS         IOSOptions
    Android     AndroidOptions
    Services    []Service     // Go services
    MarshalError func(error) []byte
    BindAliases map[uint32]uint32
    Logger      *slog.Logger
    LogLevel    slog.Level
    Assets      AssetOptions
    Flags       map[string]any
    PanicHandler              func(*PanicDetails)
    DisableDefaultSignalHandler bool
    KeyBindings               map[string]func(window Window)
    OnShutdown                func()
    PostShutdown              func()
    ShouldQuit                func() bool
    RawMessageHandler         func(window Window, message string, originInfo *OriginInfo)
    WarningHandler            func(string)
    ErrorHandler              func(err error)
    FileAssociations          []string
    SingleInstance            *SingleInstanceOptions
    Transport                 Transport
    Server                    ServerOptions
}
```

---

## Services & Binding

### v2 Bind Mode

```go
// Method signature constraints
// 1. Method name must be exported (capitalized)
// 2. Parameters and return types must support encoding/json
// 3. context.Context as first parameter is optional
// 4. Returning error as last return value becomes rejected Promise in frontend
// 5. Maximum 2 return values (excluding error)

type GreetService struct{}

// ✅ Valid signatures
func (s *GreetService) Greet(name string) string
func (s *GreetService) GreetWithContext(ctx context.Context, name string) string
func (s *GreetService) GetData(id int) (Data, error)
func (s *GreetService) NoReturn()
func (s *GreetService) OnlyError() error

// ❌ Invalid signatures
func (s *GreetService) greet(name string) string     // Not exported
func (s *GreetService) GetData(id int) (Data, string, error) // >2 return values
func (s *GreetService) GetChannel() chan int         // Unsupported type
```

**v2 Frontend invocation** (auto-generated wailsjs):
```js
import {Greet} from "../wailsjs/go/main/GreetService";
Greet("world").then(result => console.log(result)).catch(err => console.error(err));
```

### v3 Service Mode

```go
// Generic registration
application.NewService(&MyService{})
application.NewServiceWithOptions(&MyService{}, application.ServiceOptions{
    Name:         "CustomName", // Override service name
    Route:        "/api",       // If implementing http.Handler, mount at this route
    MarshalError: func(err error) []byte { ... },
})

// Optional interfaces
type ServiceName interface { ServiceName() string }
type ServiceStartup interface { ServiceStartup(ctx context.Context, options ServiceOptions) error }
type ServiceShutdown interface { ServiceShutdown() error }

// Startup order: Services list order + RegisterService order
// Shutdown order: reverse order (last registered, first shutdown)
```

---

## Window Options (v3)

### WebviewWindowOptions

```go
type WebviewWindowOptions struct {
    Name             string           // Unique name
    Title            string           // Window title
    Width            int              // Initial width
    Height           int              // Initial height
    AlwaysOnTop      bool
    URL              string           // Page path (relative to asset server)
    DisableResize    bool
    Frameless        bool
    MinWidth         int
    MinHeight        int
    MaxWidth         int
    MaxHeight        int
    StartState       WindowState      // Normal/Minimised/Maximised/Fullscreen
    BackgroundType   BackgroundType   // BackgroundTypeSolid, etc.
    BackgroundColour RGBA
    HTML             string           // Direct HTML content
    JS               string           // JS injected on page load
    CSS              string           // CSS injected on page load
    AllowSimpleEventEmit bool         // ⚠️ Security-sensitive: allow postMessage events
    InitialPosition  WindowStartPosition // WindowCentered / WindowXY
    X, Y             int              // Initial position
    Screen           *Screen          // Target screen
    Hidden           bool             // Hidden at startup
    Zoom             float64
    ZoomControlEnabled bool
    EnableFileDrop   bool
    Permissions      map[PermissionType]Permission
    OpenInspectorOnStartup bool
    Mac              MacWindow
    Windows          WindowsWindow
    Linux            LinuxWindow
    MinimiseButtonState  ButtonState  // ButtonEnabled/Disabled/Hidden
    MaximiseButtonState  ButtonState
    CloseButtonState     ButtonState
    FullscreenButtonState ButtonState
    DevToolsEnabled  bool
    DefaultContextMenuDisabled bool
    KeyBindings      map[string]func(window Window)
    IgnoreMouseEvents bool
    ContentProtectionEnabled bool
    HideOnFocusLost  bool
    UseApplicationMenu bool          // Windows/Linux: inherit app menu
}
```

### MacWindow

```go
type MacWindow struct {
    TitleBar                MacTitleBar
    Backdrop                MacBackdrop
    InvisibleTitleBarHeight int
    EventMapping            map[events.WindowEventType]events.WindowEventType
    DisableZoom             bool
    DisableFullscreen       bool
    DisableTrafficLights    bool
    EnableTitleBarAccessories bool
}

// MacTitleBar constants
MacTitleBarDefault
MacTitleBarHiddenInset
MacTitleBarHiddenInsetUnified

// MacBackdrop constants
MacBackdropNormal
MacBackdropTransparent
MacBackdropTranslucent
```

### WindowsWindow

```go
type WindowsWindow struct {
    Menu                *Menu
    EventMapping        map[events.WindowEventType]events.WindowEventType
    DisableIcon         bool
    DisableFramelessWindowDecorations bool
    Theme               Theme // SystemDefault/Dark/Light
    WndProcInterceptor  func(hwnd uintptr, msg uint32, wparam, lparam uintptr) (bool, uintptr)
}
```

### LinuxWindow

```go
type LinuxWindow struct {
    Menu         *Menu
    EventMapping map[events.WindowEventType]events.WindowEventType
}
```

---

## Platform Options

### v3 MacOptions

```go
type MacOptions struct {
    ActivationPolicy                                ActivationPolicy
    ApplicationShouldTerminateAfterLastWindowClosed bool
}
// ActivationPolicy: Regular / Accessory / Prohibited
```

### v3 WindowsOptions

```go
type WindowsOptions struct {
    Webview2Config            *Webview2Config
    WndProcInterceptor        func(hwnd uintptr, msg uint32, wparam, lparam uintptr) (bool, uintptr)
    DisableFramelessWindowDecorations bool
    Theme                     Theme
    DisableIcon               bool
    EnableFraudulentWebsiteDetection bool
    WebviewBrowserPath        string
    WebviewUserDataPath       string
    CustomTheme               *ThemeSettings
}

type Webview2Config struct {
    BrowserPath    string
    UserDataPath   string
    BrowserFlags   []string
    BrowserFeatures []string
}
```

### v3 LinuxOptions

```go
type LinuxOptions struct {
    ProgramName                   string
    DisableQuitOnLastWindowClosed bool
}
```

> **Note**: v3 Linux defaults to GTK4 + WebKitGTK 6.0. For GTK3 + WebKit2GTK 4.1, use `-tags gtk3`.

### v3 IOSOptions

```go
type IOSOptions struct {
    ScrollEnabled                bool
    BounceEnabled                bool
    ZoomEnabled                  bool
    NavigationGesturesEnabled    bool
    LinkPreviewEnabled           bool
    InlineMediaPlaybackEnabled   bool
    UserAgent                    string
    NativeTabBar                 *NativeTabBarConfig
    BackgroundColour             RGBA
    InputAccessoryView           *InputAccessoryViewConfig
}
```

### v3 AndroidOptions

```go
type AndroidOptions struct {
    ScrollEnabled            bool
    OverScrollEnabled        bool
    ZoomEnabled              bool
    UserAgent                string
    BackgroundColour         RGBA
    HardwareAccelerationEnabled bool
}
```

---

## v2 Runtime API Full List

All functions take a `context.Context` (obtained from `OnStartup`/`OnDomReady`/`OnShutdown`/`OnBeforeClose`).

### Window Operations (`runtime.Window*`)

| Function | Description |
|----------|-------------|
| `WindowSetTitle(ctx, title)` | Set window title |
| `WindowFullscreen(ctx)` | Enter fullscreen |
| `WindowUnfullscreen(ctx)` | Exit fullscreen |
| `WindowMaximise(ctx)` | Maximize |
| `WindowUnmaximise(ctx)` | Restore from maximize |
| `WindowToggleMaximise(ctx)` | Toggle maximize |
| `WindowMinimise(ctx)` | Minimize |
| `WindowUnminimise(ctx)` | Restore from minimize |
| `WindowShow(ctx)` | Show window |
| `WindowHide(ctx)` | Hide window |
| `WindowSetSize(ctx, width, height)` | Set size |
| `WindowGetSize(ctx) (int, int)` | Get size |
| `WindowSetMinSize(ctx, width, height)` | Set minimum size |
| `WindowSetMaxSize(ctx, width, height)` | Set maximum size |
| `WindowSetPosition(ctx, x, y)` | Set position |
| `WindowGetPosition(ctx) (int, int)` | Get position |
| `WindowCenter(ctx)` | Center on screen |
| `WindowSetAlwaysOnTop(ctx, bool)` | Always on top |
| `WindowReload(ctx)` | Reload page |
| `WindowReloadApp(ctx)` | Reload application |
| `WindowExecJS(ctx, js)` | Execute JS |
| `WindowSetBackgroundColour(ctx, R, G, B, A)` | Set background color |
| `WindowIsFullscreen(ctx) bool` | Is fullscreen? |
| `WindowIsMaximised(ctx) bool` | Is maximized? |
| `WindowIsMinimised(ctx) bool` | Is minimized? |
| `WindowIsNormal(ctx) bool` | Is normal state? |
| `WindowPrint(ctx)` | Print |
| `WindowSetSystemDefaultTheme(ctx)` | System default theme |
| `WindowSetLightTheme(ctx)` | Light theme |
| `WindowSetDarkTheme(ctx)` | Dark theme |

### Dialogs (`runtime.*Dialog`)

| Function | Description |
|----------|-------------|
| `OpenFileDialog(ctx, OpenDialogOptions) (string, error)` | Open file |
| `OpenMultipleFilesDialog(ctx, OpenDialogOptions) ([]string, error)` | Multiple files |
| `OpenDirectoryDialog(ctx, OpenDialogOptions) (string, error)` | Choose directory |
| `SaveFileDialog(ctx, SaveDialogOptions) (string, error)` | Save file |
| `MessageDialog(ctx, MessageDialogOptions) (string, error)` | Message dialog |

### Events

| Function | Description |
|----------|-------------|
| `EventsOn(ctx, name, callback) func()` | Register listener, returns cancel func |
| `EventsOnce(ctx, name, callback) func()` | One-shot listener |
| `EventsOnMultiple(ctx, name, callback, counter) func()` | Limited-count listener |
| `EventsEmit(ctx, name, data...)` | Emit event |
| `EventsOff(ctx, name, additionalNames...)` | Unregister listener(s) |
| `EventsOffAll(ctx)` | Unregister all listeners |

### Logging

| Function | Description |
|----------|-------------|
| `LogPrint(ctx, msg)` / `LogPrintf(ctx, fmt, args...)` | Print level |
| `LogTrace(ctx, msg)` / `LogTracef(ctx, fmt, args...)` | Trace level |
| `LogDebug(ctx, msg)` / `LogDebugf(ctx, fmt, args...)` | Debug level |
| `LogInfo(ctx, msg)` / `LogInfof(ctx, fmt, args...)` | Info level |
| `LogWarning(ctx, msg)` / `LogWarningf(ctx, fmt, args...)` | Warning level |
| `LogError(ctx, msg)` / `LogErrorf(ctx, fmt, args...)` | Error level |
| `LogFatal(ctx, msg)` / `LogFatalf(ctx, fmt, args...)` | Fatal level |
| `LogSetLogLevel(ctx, level)` | Set log level |

### Other

| Function | Description |
|----------|-------------|
| `Quit(ctx)` | Quit application |
| `Hide(ctx)` | Hide application |
| `Show(ctx)` | Show application |
| `Environment(ctx) EnvironmentInfo` | Environment info (BuildType/Platform/Arch) |
| `BrowserOpenURL(ctx, url)` | Open URL in browser |
| `ClipboardGetText(ctx) (string, error)` | Read clipboard |
| `ClipboardSetText(ctx, text) error` | Write clipboard |
| `ScreenGetAll(ctx) ([]Screen, error)` | Get all screens |

---

## v3 App/Window Methods Full List

Obtain the global App instance via `app := application.Get()`.

### WebviewWindow Methods

```go
// Basic properties
win.SetTitle(title string)
win.Title() string
win.Name() string
win.ID() uint
win.SetSize(width, height int)
win.Size() (int, int)
win.Width() int
win.Height() int
win.SetMinSize(width, height int)
win.SetMaxSize(width, height int)
win.SetPosition(x, y int)
win.Position() (int, int)
win.RelativePosition() (int, int)
win.SetRelativePosition(x, y int)
win.Center()
win.CenterOnScreen(screen *Screen)
win.SetAlwaysOnTop(bool)
win.SetResizable(bool)
win.SetFrameless(bool)
win.IsFrameless() bool
win.SetBackgroundColour(colour RGBA)

// Window state
win.Minimise()
win.Unminimise()
win.IsMinimised() bool
win.Maximise()
win.Unmaximise()
win.IsMaximised() bool
win.Fullscreen()
win.Unfullscreen()
win.IsFullscreen() bool
win.IsNormal() bool
win.Zoom()

// Visibility
win.Show()
win.Hide()
win.IsVisible() bool
win.IsFocused() bool
win.Focus()
win.Close()
win.Destroy()

// Button states
win.SetMinimiseButtonState(ButtonState)
win.SetMaximiseButtonState(ButtonState)
win.SetCloseButtonState(ButtonState)
win.SetFullscreenButtonState(ButtonState)

// JS execution
win.ExecJS(js string)

// Zoom
win.ZoomIn()
win.ZoomOut()
win.ZoomReset()
win.GetZoom() float64
win.SetZoom(zoom float64)

// Page
win.Reload()
win.ForceReload()
win.SetURL(url string)
win.SetHTML(html string)

// Other
win.Flash(enabled bool)
win.Print() error
win.OpenDevTools()
win.GetScreen() (*Screen, error)
win.StartDrag() error
win.StartResize(border string) error  // "left"/"right"/"top"/"bottom", etc.
win.IsEnabled() bool
win.SetEnabled(enabled bool)
win.SetIgnoreMouseEvents(ignore bool)
win.IsIgnoreMouseEvents() bool
win.SetContentProtection(enabled bool)

// Menus
win.SetMenu(menu *Menu)
win.OpenContextMenu(menu *Menu, data *ContextMenuData)

// Edit operations (v3.0+)
win.Cut()
win.Copy()
win.Paste()
win.Undo()
win.Redo()
win.Delete()
win.SelectAll()

// Native window handle
win.NativeWindow() unsafe.Pointer

// Events
win.EmitEvent(name string, data ...any) bool
win.RegisterHook(eventType events.WindowEventType, callback func(*WindowEvent))
win.OnWindowEvent(eventType events.WindowEventType, callback func(*WindowEvent))
```

### App Methods

```go
app.Quit()
app.CurrentWindow() *WebviewWindow
app.Window.GetByID(id uint) *WebviewWindow
app.Window.GetByName(name string) *WebviewWindow
app.Context() context.Context  // Application-level context
```

---

## v3 Dialog API

### MessageDialog

```go
dialog := app.Dialog.Info()    // or .Question() / .Warning() / .Error()
dialog.SetTitle("Title")
dialog.SetMessage("Message")
dialog.SetIcon(iconBytes)
dialog.AttachToWindow(win)     // Modal attachment
dialog.SetDefaultButton(btn)
dialog.SetCancelButton(btn)
dialog.AddButton("Label") *Button  // Returns Button, chainable .OnClick(fn) / .SetAsDefault() / .SetAsCancel()
dialog.Show()
```

### OpenFileDialog

```go
dialog := app.Dialog.OpenFile()
dialog.SetTitle("Title")
dialog.SetMessage("Message")        // Shown as prompt on macOS
dialog.SetButtonText("Choose")
dialog.SetDirectory("/path")
dialog.CanChooseFiles(true)
dialog.CanChooseDirectories(true)
dialog.CanCreateDirectories(true)
dialog.ShowHiddenFiles(true)
dialog.ResolvesAliases(true)
dialog.AllowsOtherFileTypes(true)
dialog.TreatsFilePackagesAsDirectories(true)
dialog.CanSelectHiddenExtension(true)
dialog.AddFilter("Text Files", "*.txt; *.md")
dialog.AttachToWindow(win)

// Three invocation modes
result, err := dialog.PromptForSingleSelection()
results, err := dialog.PromptForMultipleSelection()
```

### SaveFileDialog

```go
dialog := app.Dialog.SaveFile()
dialog.SetTitle("Title")
dialog.SetMessage("Message")        // Shown as prompt on macOS
dialog.SetButtonText("Save")
dialog.SetDirectory("/path")
dialog.SetFilename("README.md")
dialog.HideExtension(true)
dialog.ShowHiddenFiles(true)
dialog.CanCreateDirectories(true)
dialog.AllowsOtherFileTypes(true)
dialog.AddFilter("Text Files", "*.txt; *.md")
dialog.AttachToWindow(win)

result, err := dialog.PromptForSingleSelection()
```

---

## v3 Event API Full List

### Predefined Events (events.Common)

#### Application-Level

| Event | Description |
|-------|-------------|
| `ApplicationStarted` | Application finished launching |
| `ThemeChanged` | System theme changed |
| `SystemDidWake` | System woke from sleep |
| `SystemWillSleep` | System about to sleep |

#### Window-Level

| Event | Description |
|-------|-------------|
| `WindowClosing` | Window about to close (cancelable) |
| `WindowDidMove` | Window moved |
| `WindowDidResize` | Window resized |
| `WindowDPIChanged` | DPI changed |
| `WindowFilesDropped` | Files dropped on window |
| `WindowFocus` | Window gained focus |
| `WindowLostFocus` | Window lost focus |
| `WindowFullscreen` | Window entered fullscreen |
| `WindowHide` | Window hidden |
| `WindowMaximise` | Window maximized |
| `WindowMinimise` | Window minimized |
| `WindowRestore` | Window restored |
| `WindowRuntimeReady` | Frontend runtime ready |
| `WindowShow` | Window shown |
| `WindowUnFullscreen` | Window exited fullscreen |
| `WindowUnMaximise` | Window restored from maximize |
| `WindowUnMinimise` | Window restored from minimize |
| `WindowZoom` / `WindowZoomIn` / `WindowZoomOut` / `WindowZoomReset` | Zoom events |

#### Platform-specific (200+ macOS/Linux/Windows system events)

Full list in `v3/pkg/events/known_events.go`.

### Custom Events

```go
// Register listeners
app.Event.On(name, func(e *application.CustomEvent) { ... })  // Returns cancel function
app.Event.OnApplicationEvent(eventType, func(e *application.ApplicationEvent) { ... })

// Emit
app.Event.Emit(name, data)
app.Event.EmitEvent(event) // Emit CustomEvent directly
```

### Event Modes

v3 event system supports two modes:
- **Strict mode** (default): Events must be pre-registered; emitting unregistered events reports an error
- **Loose mode**: Allows emitting unregistered events

---

## v3 Menu API

### Menu Methods

```go
menu := app.NewMenu()
menu.Add(label string) *MenuItem
menu.AddSeparator()
menu.AddCheckbox(label string, enabled bool) *MenuItem
menu.AddRadio(label string, enabled bool) *MenuItem
menu.AddSubmenu(label string) *Menu
menu.AddRole(role Role) *Menu  // AppMenu/FileMenu/EditMenu/WindowMenu/ServicesMenu/HelpMenu
menu.Update()                   // Apply to native layer
menu.Clear()
menu.Destroy()
menu.SetLabel(label string)
menu.FindByLabel(label string) *MenuItem
menu.FindByRole(role Role) *MenuItem
menu.ItemAt(index int) *MenuItem
menu.RemoveMenuItem(item *MenuItem)
menu.Clone() *Menu
menu.Append(in *Menu)
menu.Prepend(in *Menu)
```

### MenuItem Methods

```go
item.OnClick(callback func(ctx *Context)) *MenuItem
item.SetEnabled(enabled bool)
item.SetChecked(checked bool)
item.SetAccelerator(accel string)  // "CmdOrCtrl+S"
item.SetTooltip(tooltip string)
item.SetHidden(hidden bool)
item.Destroy()
```

### ContextMenu

```go
ctxMenu := application.NewContextMenu("name")
ctxMenu.Add("Copy").OnClick(func(ctx *application.Context) { ... })
ctxMenu.AddSeparator()
ctxMenu.Update()  // Register with app
ctxMenu.Destroy()
```

---

## JS Frontend Runtime API

### v2 JS Runtime

```js
// Call Go methods (auto-generated)
import {Greet} from "../wailsjs/go/main/GreetService"
const result = await Greet("world")

// Events
window.wails.EventsOn("event-name", (data) => { ... })
window.wails.EventsOnce("event-name", (data) => { ... })
window.wails.EventsOnMultiple("event-name", callback, maxCalls)
window.wails.EventsEmit("event-name", data)
window.wails.EventsOff("event-name")

// Logging
window.wails.LogLevel("debug")
window.wails.LogPrint("msg")
window.wails.LogDebug("msg")

// System
window.wails.Quit()
window.wails.BrowserOpenURL("https://...")
```

### v3 JS Runtime

v3 communicates through HTTP transport. JS-side APIs are generated in the `frontend/wailsjs/` directory. The runtime script is automatically served at `/wails/runtime.js` and injected by `BundledAssetFileServer`.

```js
// Call Go methods (via fetch through HTTP transport)
// Auto-generated binding code handles serialization and transport

// Events (via runtime script)
window._wails.invoke("wails:event:emit:eventName")
window._wails.on("eventName", callback)
```
