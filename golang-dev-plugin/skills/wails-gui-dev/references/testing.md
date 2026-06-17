# Wails Testing & Debugging

This document covers testing methods, debugging toolchains, and common troubleshooting for Wails v2 and v3.

## Table of Contents

- [Dev Debugging](#dev-debugging)
  - [Dev Mode](#dev-mode)
  - [WebView DevTools](#webview-devtools)
  - [Log Debugging](#log-debugging)
- [Unit Testing](#unit-testing)
  - [v2 Testing](#v2-testing)
  - [v3 Testing](#v3-testing)
- [Build Verification Tests](#build-verification-tests)
- [Panic Recovery & Error Handling](#panic-recovery--error-handling)
- [Performance Profiling](#performance-profiling)
- [Common Debugging Issues](#common-debugging-issues)

---

## Dev Debugging

### Dev Mode

```bash
wails dev         # v2
wails3 dev        # v3
```

Dev mode features:
- **Frontend hot reload**: Vite/webpack HMR instant updates
- **Go hot reload**: Go code auto-recompile and restart on changes
- **DevTools auto-enabled**: Right-click to open browser developer tools
- **Console logging**: Go and JS logs output to terminal

### WebView DevTools

How to open on each platform:

| Platform | Method |
|----------|--------|
| macOS | Safari → Develop → [app name] → select page |
| Windows | Right-click WebView → Inspect (or F12) |
| Linux | WebKit inspector (`webkit_web_inspector_show()`) |

**v3 auto-open devtools**:
```go
app.Window.NewWithOptions(application.WebviewWindowOptions{
    OpenInspectorOnStartup: true,
})
```

**v3 trigger from code**:
```go
win.OpenDevTools()
```

**v2 trigger from code**:
```go
// Via frontend JS
window.wails.OpenInspector()
```

### Log Debugging

**v2 logging**:
```go
// Set log level
runtime.LogSetLogLevel(ctx, logger.DEBUG)
runtime.LogDebug(ctx, "debug message")
runtime.LogDebugf(ctx, "count: %d", count)

// Frontend
window.wails.LogLevel("debug")
window.wails.LogDebug("frontend debug")
```

**v3 logging**:
```go
// Create leveled logger
app := application.New(application.Options{
    Logger:  application.DefaultLogger(slog.LevelDebug),
    LogLevel: slog.LevelDebug,
})

// Usage
app.Logger.Info("message", "key", value)
app.Logger.Debug("debug", "count", 42)
app.Logger.Error("error", "err", err)
```

**v3 conditional-compilation logging**:
```go
// Only enabled in non-production builds
// Uses isDebugMode from application_debug.go
```

**v3 GTK debugging** (Linux):
```bash
# Enable GTK debug output at build time
CGO_CFLAGS="-DWAILS_GTK_DEBUG" go build ...
```

---

## Unit Testing

### v2 Testing

**Testing runtime API**:
```go
// Runtime API depends on frontend/logger/events in context.Context
// To test, construct a context with mock implementations

import (
    "context"
    "testing"
)

func TestMyBoundMethod(t *testing.T) {
    service := &MyService{}
    result := service.Greet("world")
    if result != "Hello, world" {
        t.Errorf("expected 'Hello, world', got '%s'", result)
    }
}
```

**Testing bound methods**:
```go
// Test Go struct methods directly, without going through Wails bridge layer
// Wails-bound Go methods are pure functions and can be unit tested directly
```

**v2 internal tests**:
- `v2/internal/binding/binding_test/` — Binding system tests
- `v2/pkg/options/options_test.go` — Options merge tests
- `v2/pkg/commands/buildtags/buildtags_test.go` — Build tag tests

### v3 Testing

**Testing Service methods**:
```go
// Service methods are pure Go functions — test directly
func TestGreetService_Greet(t *testing.T) {
    s := &GreetService{}
    result := s.Greet("world")
    assert.Equal(t, "Hello, world", result)
}

// Test ServiceStartup/ServiceShutdown
func TestServiceLifecycle(t *testing.T) {
    s := &MyService{}
    ctx := context.Background()
    err := s.ServiceStartup(ctx, application.DefaultServiceOptions)
    require.NoError(t, err)

    // Use the service...

    err = s.ServiceShutdown()
    require.NoError(t, err)
}
```

**v3 internal tests**:
- `v3/pkg/application/services_test.go` — Service registration/lifecycle
- `v3/pkg/application/bindings_test.go` — Binding invocation
- `v3/pkg/application/events_test.go` — Event system
- `v3/pkg/application/options_test.go` — Options handling
- `v3/pkg/application/webview_window_test.go` — Window management
- `v3/pkg/application/internal/tests/services/` — Service lifecycle integration tests

**v3 Benchmark tests**:
```bash
go test -bench=. ./v3/pkg/application/...
```
Key benchmark files:
- `bindings_bench_test.go` — Binding call performance
- `bindings_optimized_bench_test.go` — Optimized path performance
- `events_bench_test.go` — Event emit/consume performance
- `json_libs_bench_test.go` / `json_v2_bench_test.go` — JSON serialization performance

---

## Build Verification Tests

v3 provides comprehensive cross-platform build verification (`v3/Taskfile.yaml`):

```bash
# All platforms (43 examples × 3 platforms = 129 builds)
task test:examples:all

# Current platform
task test:examples

# CLI code tests
task test:cli

# Single example
task test:example:darwin DIR=badge
task test:example:windows DIR=badge
task test:example:linux DIR=badge
task test:example:linux:docker DIR=badge
```

Output binary naming:
- macOS: `testbuild-{name}-darwin`
- Windows: `testbuild-{name}-windows.exe`
- Linux: `testbuild-{name}-linux`

---

## Panic Recovery & Error Handling

### v2

Panics in message processing are caught by default to prevent the entire app from crashing.
```go
options.App{
    DisablePanicRecovery: false,  // Default false = recovery enabled
}
```

### v3

Custom panic handling:
```go
application.Options{
    PanicHandler: func(details *application.PanicDetails) {
        // details.Error  — panic value
        // details.Stack  — stack trace
        // details.Window — window where panic occurred (may be nil)
        log.Printf("Panic in window %v: %v\nStack:\n%s",
            details.Window, details.Error, details.Stack)
    },
}
```

Error handling:
```go
application.Options{
    ErrorHandler: func(err error) {
        app.Logger.Error("application error", "err", err)
    },
    WarningHandler: func(msg string) {
        app.Logger.Warn("warning", "msg", msg)
    },
}
```

---

## Performance Profiling

### Runtime Performance

- Wails itself has minimal overhead: Go method call ≈ direct reflection call + JSON serialization
- WebView performance = underlying browser engine performance (Chromium/WebKit)
- Frontend rendering performance depends on the web framework (React/Vue/Svelte, etc.)

### Optimization Tips

1. **Reduce Go ↔ JS call frequency**: Batch data, avoid frequent small messages
2. **Use appropriate data structures**: Pagination or virtual scrolling for large lists
3. **v3 Transport selection**: WebSocket Transport for high-frequency data push
4. **JSON serialization**: v3 provides multiple JSON library benchmarks; choose the fastest

### Running Benchmarks

```bash
# v3 JSON serialization comparison
go test -bench=JSON -benchmem ./v3/pkg/application/

# v3 binding calls
go test -bench=Binding -benchmem ./v3/pkg/application/

# v3 event system
go test -bench=Event -benchmem ./v3/pkg/application/
```

---

## Common Debugging Issues

### 1. Blank Screen, No Errors

**Troubleshooting steps**:
```bash
# 1. Check embed path
# Confirm //go:embed all:frontend/dist path is correct
# Confirm dist/ directory exists and contains index.html

# 2. Check asset server
# In dev mode, use curl to test
curl http://localhost:34115/

# 3. Check console
# Open WebView DevTools, check Console and Network tabs

# 4. Check build
wails build -debug  # Keep devtools
```

### 2. Go Method Not Responding in Frontend

**Troubleshooting steps**:
```go
// 1. Confirm method is exported
func (s *Service) Greet(name string) string  // ✅
func (s *Service) greet(name string) string  // ❌

// 2. Confirm parameter types support JSON
// ✅ string, int, bool, float, struct (with json tags), slice, map
// ❌ chan, func, complex, unsafe.Pointer

// 3. v2: confirm struct is registered in Bind
options.App{Bind: []interface{}{&MyService{}}}

// 4. v3: confirm Service is in Services list
application.Options{Services: []application.Service{
    application.NewService(&MyService{}),
}}

// 5. Check frontend invocation path
// v2: import {Method} from "../wailsjs/go/main/MyService"
// Confirm wailsjs has been generated (run wails dev or wails build)
```

### 3. Hot Reload Not Working

```bash
# 1. Confirm dev server is running
npm run dev    # In frontend/ directory

# 2. Check wails.json config
"frontend:dev:serverUrl": "auto"

# 3. Manually specify URL
"frontend:dev:serverUrl": "http://localhost:5173"

# 4. Use environment variable
FRONTEND_DEVSERVER_URL=http://localhost:5173 wails dev
```

### 4. Linux Build Errors

```bash
# Check dependencies
wails3 doctor

# Ubuntu/Debian v3 (GTK4 default)
sudo apt install libgtk-4-dev libwebkitgtk-6.0-dev

# Ubuntu/Debian v3 (GTK3 legacy)
sudo apt install libgtk-3-dev libwebkit2gtk-4.1-dev

# Build with GTK3
go build -tags gtk3
```

### 5. Windows WebView2 Errors

```bash
# Install WebView2 Runtime
# https://developer.microsoft.com/en-us/microsoft-edge/webview2/

# Embed WebView2 Bootstrapper
wails build -webview2 embed

# Check WebView2 version
wails doctor
```

### 6. macOS Code Signing

```bash
# Development signing
wails build -sign

# Specify certificate
wails build -sign -signid "Developer ID Application: ..."

# Skip signing (development only)
wails build -skip-sign
```
