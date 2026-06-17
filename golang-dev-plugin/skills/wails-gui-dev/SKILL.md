---
name: wails-gui-dev
description: Wails cross-platform desktop application framework. Go backend + Web frontend (any framework), bridging system-native WebView with Go runtime via CGO, packaged as a single binary. Use this skill when creating, modifying, or debugging Wails desktop applications. Trigger: mentions wails, Go desktop app, WebView desktop development, Go + React/Vue/Svelte desktop app, wails2/wails3, frontend-backend bridge. Even without explicitly saying "wails", consider this skill when doing Go desktop GUI development involving web technologies.
---

# Wails Desktop Application Development

Wails combines a **Go runtime + system-native WebView** (no embedded browser) via a CGO bridge layer into a single binary. The frontend uses any web framework; the backend uses Go. The two communicate through a messaging layer.

## Versions: v2 (Stable) vs v3 (Alpha)

| Version | Status | CLI | Install |
|---------|--------|-----|---------|
| v2 | Stable | `wails` | `go install github.com/wailsapp/wails/v2/cmd/wails@latest` |
| v3 | Alpha | `wails3` | `go install github.com/wailsapp/wails/v3/cmd/wails3@latest` |

**Key differences at a glance**:

| Aspect | v2 | v3 |
|--------|-----|-----|
| Binding | `Bind: []interface{}{}` | `Services: []Service` (generics) |
| Logging | Custom `logger.Logger` | Standard `log/slog` |
| IPC | Fixed fetch + JS eval | Pluggable `Transport` interface |
| Runtime API | `context.Context` injection | `app.Get()` global singleton |
| Events | `runtime.EventsOn/Emit(ctx, ...)` | `app.Event.On/Emit()` + Hooks |
| Windows | Single window | Multi-window, native support |
| Menus | Functional `menu.Text/Checkbox/...` | Chainable `menu.Add().OnClick()` |
| Dialogs | `runtime.OpenFileDialog(ctx, opts)` | `app.Dialog.OpenFile().SetTitle(...).Show()` |
| Mobile | ❌ | iOS + Android |
| Server mode | ❌ | `-tags server` headless deployment |

**Guidance**: Choose v3 for new projects; use v2 when you need production stability or are maintaining existing apps.

## Overall Architecture

```
┌──────────────────────────────────────────────────────────┐
│                   OS Native Window                        │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Native WebView (macOS: WKWebView / Win: WebView2  │  │
│  │  Linux: WebKitGTK / iOS: WKWebView / Android: WV)  │  │
│  │  ┌──────────────────────────────────────────────┐  │  │
│  │  │  Frontend (HTML/JS/CSS) — any web framework  │  │  │
│  │  │  Served by asset server via custom scheme    │  │  │
│  │  └──────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────┤
│  CGO Bridge: Message Routing + Asset Server + JS Exec    │
├──────────────────────────────────────────────────────────┤
│  Go Runtime: App / Services / Events / Window / Native   │
└──────────────────────────────────────────────────────────┘
```

### Communication Mechanisms

1. **JS → Go**: v2 via `window.wails.Call()`, v3 via `fetch("/wails/runtime")` — sends JSON messages; MessageProcessor routes to bound Go methods
2. **Go → JS**: `win.ExecJS(js)` executes JS in the WebView; or push via the event system
3. **Asset Loading**: Frontend resources served through a custom URL scheme (`wails://`) by the Asset Server from `//go:embed`-embedded filesystem — **no network port needed**
4. **Event System**: Go and JS share a unified event mechanism; v3 supports Hooks (intercept and cancel events before propagation)

See `references/architecture.md` for detailed architecture (CGO bridge, per-platform WebView backends, Transport architecture, server mode).

## Core Principles

1. **Services = Backend API** — Public methods on Go structs are automatically exposed to the frontend; no manual route definitions needed
2. **Go owns logic, JS owns UI** — Clear responsibility boundaries (see next section)
3. **Full frontend-backend separation** — Frontend can be React/Vue/Svelte/Lit/Vanilla; build output embedded via `//go:embed`
4. **Main thread serialization (v3)** — UI operations dispatched to main thread via `InvokeSync`/`InvokeAsync`; v2 handles this transparently
5. **Native capabilities, no compromises** — Menus, dialogs, system trays, notifications all use platform-native APIs

## Go vs JS Responsibility Boundary

In a Wails application, Go and JS have distinct responsibilities. Mixing them up leads to architectural confusion, performance issues, or security risks.

### Put in Go (Backend)

```
✅ Business logic and rules
✅ Database operations (SQLite, Postgres, etc.)
✅ File system read/write (local files, config)
✅ System calls and OS integration (process management, registry, permissions)
✅ Native dialogs, menus, system tray
✅ Encryption, authentication, sensitive data handling
✅ Performance-intensive computation (large data processing, image encoding)
✅ External API calls (those requiring secret protection)
✅ Clipboard read/write
✅ Multi-window management
✅ Hardware access (camera, microphone, serial port)
```

**Rule**: If it needs OS capabilities, involves security, or is core business logic — put it in Go.

### Put in JS (Frontend)

```
✅ UI rendering and DOM manipulation
✅ User interaction (clicks, drags, input validation)
✅ Animations and transitions
✅ Responsive layout
✅ Theme switching and styling
✅ Client-side form validation (instant feedback)
✅ Charts and visualization
✅ Routing (frontend page navigation)
✅ Loading states and skeleton screens
```

**Rule**: If it's purely visual, needs instant feedback, and requires no OS capabilities — put it in JS.

### Collaboration Patterns

| Scenario | Go does | JS does |
|----------|---------|---------|
| Open file | Call native file dialog, read file contents, return to JS | Show file-picker button, receive and render file contents |
| Save settings | Receive settings data from JS, write to local file/DB | Collect form data, call Go method to save, show success toast |
| Search | Execute search logic, query DB, return results | Show search box, render result list |
| File drag-and-drop | Receive file path list, process files | Render drop zone, show drag feedback animation |
| System theme | Detect system theme change, notify JS | Switch CSS variables/styles |
| Notifications | Call system notification API | Provide UI button to trigger notification |

### Decision Flow

```
Does this feature...
├─ Need OS permissions / native API? → Go
├─ Pure UI display / animation?      → JS
├─ Involve secrets / sensitive data? → Go
├─ Need instant user feedback?       → JS (call Go when data is needed)
├─ Involve heavy computation?        → Go (push results to JS via events)
└─ Not sure?                         → Default to Go (safer side), expose to JS via methods/events
```

## Minimal Application

### v2

```go
package main

import (
    "embed"
    "context"
    "github.com/wailsapp/wails/v2"
    "github.com/wailsapp/wails/v2/pkg/options"
    "github.com/wailsapp/wails/v2/pkg/options/assetserver"
    "github.com/wailsapp/wails/v2/pkg/runtime"
)

//go:embed all:frontend/dist
var assets embed.FS

func main() {
    err := wails.Run(&options.App{
        Title:  "My App",
        Width:  1024, Height: 768,
        AssetServer: &assetserver.Options{Assets: assets},
        Bind:    []interface{}{&GreetService{}},
        OnStartup: func(ctx context.Context) {
            runtime.LogInfo(ctx, "App started!")
        },
    })
    if err != nil {
        println("Error:", err.Error())
    }
}

type GreetService struct{}
func (s *GreetService) Greet(name string) string { return "Hello, " + name }
```

### v3

```go
package main

import (
    "embed"
    "log"
    "github.com/wailsapp/wails/v3/pkg/application"
)

//go:embed all:frontend/dist
var assets embed.FS

func main() {
    app := application.New(application.Options{
        Name: "My App",
        Services: []application.Service{
            application.NewService(&GreetService{}),
        },
        Assets: application.AssetOptions{
            Handler: application.BundledAssetFileServer(assets),
        },
        Mac: application.MacOptions{
            ApplicationShouldTerminateAfterLastWindowClosed: true,
        },
    })
    app.Window.NewWithOptions(application.WebviewWindowOptions{
        Title: "My App", Width: 1024, Height: 768,
    })
    if err := app.Run(); err != nil {
        log.Fatal(err)
    }
}

type GreetService struct{}
func (s *GreetService) Greet(name string) string { return "Hello, " + name }
```

## Project Structure

```
myapp/
├── frontend/              # Frontend (Vite project, any framework)
│   ├── src/               # Source (React/Vue/Svelte/Lit/Vanilla)
│   ├── dist/              # Build output → embedded by Go
│   ├── package.json
│   ├── vite.config.js
│   └── wailsjs/           # ⚡ Auto-generated JS bindings (after wails dev/build)
│       ├── go/             #    JS wrappers for Go methods + TypeScript types
│       └── runtime/        #    Runtime type definitions
├── app.go                 # Go backend entry point
├── main.go                # main() calls app.go
├── wails.json             # Wails config
├── go.mod
└── build/                 # Platform resources (icons, Info.plist, NSIS scripts)
```

## Core API Quick Reference

Below are the most commonly used API entry points. Full listings in `references/api-reference.md`.

### App Creation (v3)

```go
app := application.New(application.Options{
    Name:        "My App",
    Description: "Description",
    Services:    []application.Service{application.NewService(&MyService{})},
    Assets:      application.AssetOptions{Handler: application.BundledAssetFileServer(assets)},
    Logger:      application.DefaultLogger(slog.LevelDebug),
    // Lifecycle
    OnShutdown:  func() { /* cleanup */ },
    ShouldQuit:  func() bool { return true },
    // Platform options
    Mac:     application.MacOptions{ApplicationShouldTerminateAfterLastWindowClosed: true},
    Windows: application.WindowsOptions{},
    Linux:   application.LinuxOptions{},
})
```

For full options (`KeyBindings`, `PanicHandler`, `SingleInstance`, `Transport`, `Server`, `IOS`/`Android`, etc.), see `references/api-reference.md`.

### Service Binding

**v2**: `Bind: []interface{}{&MyService{}}` registers structs; all exported methods are exposed to the frontend.
**v3**: `application.NewService(&MyService{})` generic registration, optional `ServiceStartup`/`ServiceShutdown`/`ServiceName` interfaces.

Method signature constraints: name must be exported (capitalized), parameters and return values must support JSON serialization. See `references/api-reference.md` — Services & Binding section for details.

### Window (v3)

```go
win := app.Window.NewWithOptions(application.WebviewWindowOptions{
    Title: "Window Title", Width: 1024, Height: 768,
    MinWidth: 400, MinHeight: 300,
    Frameless: false, AlwaysOnTop: false,
    DevToolsEnabled: true,                  // Default on in non-production builds
    OpenInspectorOnStartup: false,
    Mac:     application.MacWindow{TitleBar: application.MacTitleBarHiddenInsetUnified},
    Windows: application.WindowsWindow{},
    Linux:   application.LinuxWindow{},
})
```

Window operations: `win.SetTitle()`, `win.Center()`, `win.Maximise()`, `win.Minimise()`, `win.Fullscreen()`, `win.SetSize()`, `win.Show()`/`win.Hide()`, `win.Close()`, `win.ExecJS()`, `win.Reload()`, `win.SetAlwaysOnTop()`, `win.ZoomIn()`/`win.ZoomOut()`, and 60+ more methods. Full list in `references/api-reference.md`.

### Dialogs (v3)

Chainable API — same Go code across all platforms:

```go
// Message dialogs
app.Dialog.Info().SetTitle("Info").SetMessage("Operation complete").Show()
app.Dialog.Question().SetTitle("Confirm").SetMessage("Delete?").
    AddButton("OK").OnClick(func() { /* ... */ }).
    AddButton("Cancel").Show()
app.Dialog.Warning().SetMessage("Warning content").Show()
app.Dialog.Error().SetMessage("Error content").Show()

// File dialogs
path, _ := app.Dialog.OpenFile().
    AddFilter("Text Files", "*.txt;*.md").
    PromptForSingleSelection()
paths, _ := app.Dialog.OpenFile().PromptForMultipleSelection()
dir, _ := app.Dialog.OpenFile().CanChooseDirectories(true).CanChooseFiles(false).PromptForSingleSelection()
savePath, _ := app.Dialog.SaveFile().SetFilename("README.md").PromptForSingleSelection()
```

Full dialog API (`SetButtonText`, `ShowHiddenFiles`, `AttachToWindow`, `SetDirectory`, etc.) in `references/api-reference.md`.

### Menus (v3)

```go
menu := app.NewMenu()
menu.AddRole(application.AppMenu)    // macOS application menu
menu.AddRole(application.FileMenu)
menu.AddRole(application.EditMenu)

fileMenu := menu.AddSubmenu("File")
fileMenu.Add("Open").OnClick(func(ctx *application.Context) {
    app.Dialog.OpenFile().PromptForSingleSelection()
})
fileMenu.AddSeparator()
fileMenu.AddCheckbox("Enabled", true)
fileMenu.AddRadio("Option A", true)

app.Menu.Set(menu)

// Context menu
ctxMenu := application.NewContextMenu("my-menu")
ctxMenu.Add("Copy").OnClick(func(ctx *application.Context) { /* ... */ })
ctxMenu.Update()
```

### Event System (v3)

```go
// Custom events
app.Event.On("myevent", func(e *application.CustomEvent) {
    app.Logger.Info("Event received", "name", e.Name, "data", e.Data)
    e.Cancel()  // Cancelable
})
app.Event.Emit("myevent", "data")

// System-level Hooks (can intercept and cancel)
win.RegisterHook(events.Common.WindowClosing, func(e *application.WindowEvent) {
    if unsaved { e.Cancel() }  // Prevent closing
})

// Window event listeners
win.OnWindowEvent(events.Common.WindowFocus, func(e *application.WindowEvent) {
    app.Logger.Info("Window focused")
})
```

30+ predefined events (`WindowClosing`, `WindowDidMove`, `WindowFilesDropped`, `ThemeChanged`, etc.). Full list in `references/api-reference.md`.

### v2 Runtime API Quick Reference

All v2 runtime operations use `context.Context` obtained from `OnStartup`/`OnDomReady`/`OnBeforeClose` callbacks:

```go
// Window
runtime.WindowSetTitle(ctx, "title")
runtime.WindowFullscreen(ctx) / runtime.WindowUnfullscreen(ctx)
runtime.WindowMaximise(ctx) / runtime.WindowMinimise(ctx)
runtime.WindowSetSize(ctx, 800, 600)
runtime.WindowCenter(ctx)
runtime.WindowExecJS(ctx, "console.log('hello')")

// Dialogs
file, _ := runtime.OpenFileDialog(ctx, runtime.OpenDialogOptions{...})
runtime.SaveFileDialog(ctx, runtime.SaveDialogOptions{...})
runtime.MessageDialog(ctx, runtime.MessageDialogOptions{...})

// Events
runtime.EventsOn(ctx, "event", func(data ...interface{}) {})
runtime.EventsEmit(ctx, "event", data)

// Logging
runtime.LogInfo(ctx, "msg") / runtime.LogInfof(ctx, "count: %d", n)
runtime.LogDebug(ctx, "msg") / runtime.LogError(ctx, "msg")
```

Full v2 runtime API (30+ functions) in `references/api-reference.md`.

## Build & Deploy

```bash
# System dependency check
wails doctor      # v2
wails3 doctor     # v3

# Dev mode (hot reload + devtools)
wails dev         # v2
wails3 dev        # v3

# Production build
wails build                          # v2, current platform
wails build -platform windows/amd64  # v2, specific platform
wails build -nsis                    # v2, + NSIS installer
wails3 build                         # v3
wails3 build -platform darwin/arm64,windows/amd64,linux/amd64
wails3 build -f                      # v3, force rebuild
wails3 build -tags server            # v3, server mode
```

Platform packaging outputs:
- **macOS**: `.app` bundle (with `Info.plist`)
- **Windows**: `.exe`, optional NSIS installer
- **Linux**: binary + `.desktop` file

Cross-compilation, Docker builds, build tags (`production`/`server`/`gtk3`), see `references/build-deploy.md`.

## Testing & Debugging

### Dev Debugging

- `wails dev` / `wails3 dev` — devtools enabled automatically
- **WebView DevTools**: macOS (Safari → Develop → app name), Windows (right-click → Inspect), Linux (WebKit inspector)
- v3 `OpenInspectorOnStartup: true` opens devtools automatically

### Unit Testing

- Go Service methods are plain Go functions — just run `go test`
- v3 comprehensive build verification: `task test:examples:all` (43 examples × 3 platforms = 129 builds)
- Benchmark: `go test -bench=. ./v3/pkg/application/...`

### Common Issues Quick Reference

| Issue | Platform | Cause | Solution |
|-------|----------|-------|----------|
| Blank screen | All | Asset loading failure | Check `//go:embed` path and `frontend:build` output directory |
| Go method not responding in frontend | All | Method not exported or signature incompatible | Method name must be capitalized; params/returns must be JSON-compatible |
| v2 `context.Context` is nil | v2 | Not obtained from lifecycle callback | Use ctx from `OnStartup`/`OnDomReady` |
| cgo build error | Linux | Missing GTK/WebKit | `wails doctor`; Ubuntu needs `apt install libgtk-4-dev libwebkitgtk-6.0-dev` (v3) |
| v3 events not firing | v3 | Event name not registered in strict mode | Pre-register event names, or use loose mode |
| Window position ignored | Linux | Wayland protocol limitation | `SetPosition`/`Center` may be no-ops under Wayland |
| Hot reload not working | All | Dev server not running | Ensure `npm run dev` is running; check `frontend:dev:serverUrl` |
| Frameless window not draggable | All | CSS property not set | Add `style="--wails-draggable: drag"` to HTML elements |

More debugging tips, panic recovery, and benchmark guides in `references/testing.md`.

## Stuck? Start Here

When references don't cover your need, explore the Wails source:

```bash
# Locate source path
WAILS_DIR=$(go list -m -json github.com/wailsapp/wails/v3 | grep '"Dir"' | cut -d'"' -f4)

# Look up APIs
go doc github.com/wailsapp/wails/v3/pkg/application

# Study 43 built-in examples
cd $WAILS_DIR/examples && ls
# Key examples: binding/ dialogs/ events/ menu/ window/ systray-basic/ clipboard/ screen/ plain/
```

v3 code conventions: `New(opts)` → `Run()` is the standard entry point; `app.Window.NewWithOptions(opts)` creates windows; dialogs and menus use chainable APIs; platform files follow `*_darwin.go`/`*_windows.go`/`*_linux.go` naming.

## Reference Files

| Scenario | Read |
|----------|------|
| Complete API parameters/options listing | `references/api-reference.md` |
| Architecture internals, communication, platform differences | `references/architecture.md` |
| Build, cross-compile, package, Docker | `references/build-deploy.md` |
| Testing, debugging, benchmarks, troubleshooting | `references/testing.md` |

If the above references still don't cover your issue, explore the source code as described above. If you still can't resolve the problem after exploring the source, **report the specific situation to the user and ask for help — do not silently guess.**
