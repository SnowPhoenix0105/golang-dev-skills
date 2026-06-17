# Wails Architecture & Internals

This document dives deep into Wails' internal architecture, communication mechanisms, and platform differences.

## Table of Contents

- [Layered Architecture](#layered-architecture)
- [Go ↔ JS Communication](#go--js-communication)
  - [v2 Communication Path](#v2-communication-path)
  - [v3 Communication Path](#v3-communication-path)
- [Asset Server](#asset-server)
- [Platform WebView Backends](#platform-webview-backends)
- [CGO Bridge Layer](#cgo-bridge-layer)
- [v3 Pluggable Transport Architecture](#v3-pluggable-transport-architecture)
- [v3 Server Mode](#v3-server-mode)
- [Platform Differences & Constraints](#platform-differences--constraints)

---

## Layered Architecture

```
┌────────────────────────────────────────────────────────┐
│          Application Layer                              │
│  v2: wails.Run(&options.App{})                        │
│  v3: application.New(opts) → app.Run()                │
├────────────────────────────────────────────────────────┤
│          API Layer (platform-agnostic Go code)          │
│  v2: pkg/runtime/    pkg/options/   pkg/menu/         │
│  v3: pkg/application/  pkg/events/  pkg/services/     │
├────────────────────────────────────────────────────────┤
│          Internal Layer                                 │
│  v2: internal/frontend/  internal/binding/            │
│      internal/assetserver/  internal/menumanager/     │
│  v3: internal/assetserver/  internal/capabilities/    │
│      internal/operatingsystem/                         │
├────────────────────────────────────────────────────────┤
│          CGO Bridge Layer (platform-specific C/ObjC)    │
│  v2: internal/frontend/desktop/darwin/  (ObjC)        │
│      internal/frontend/desktop/linux/   (C + GTK)     │
│      internal/frontend/desktop/windows/ (C + Win32)    │
│  v3: pkg/application/linux_cgo.go      (C + GTK4)     │
│      pkg/application/*_darwin.m        (ObjC)         │
│      pkg/application/*_windows.go      (C + Win32)    │
├────────────────────────────────────────────────────────┤
│          Native Layer                                   │
│  macOS:   Cocoa + WKWebView                           │
│  Linux:   GTK4 + WebKitGTK 6.0 (default)              │
│           GTK3 + WebKit2GTK 4.1 (-tags gtk3)          │
│  Windows: Win32 + WebView2 (Edge/Chromium)            │
│  iOS:     UIKit + WKWebView                           │
│  Android: Android WebView                             │
└────────────────────────────────────────────────────────┘
```

---

## Go ↔ JS Communication

### v2 Communication Path

```
JS calling Go methods:
  window.wails.Call(methodName, args...)
    ↓
  WebView message handler receives
    ↓
  CGO exported function (processMessage)
    ↓
  channel → Go goroutine processing
    ↓
  MessageProcessor.ProcessMessage()
    ↓
  binding.DB.Call() → reflection-invoked Go method
    ↓
  Result serialized to JSON
    ↓
  CGO → window.wails.Callback(id, result)
    ↓
  JS Promise resolve/reject

Go executing JS:
  frontend.ExecJS(js)
    ↓
  CGO → WebView.evaluateJavaScript()
    ↓
  JS executed in WebView
```

Key files:
- `v2/internal/frontend/desktop/darwin/frontend.go` — macOS frontend implementation
- `v2/internal/frontend/desktop/linux/frontend.go` — Linux frontend implementation
- `v2/internal/frontend/desktop/windows/frontend.go` — Windows frontend implementation
- `v2/internal/binding/binding.go` — Binding reflection core (10015 lines)
- `v2/internal/frontend/dispatcher/dispatcher.go` — Message routing

### v3 Communication Path

```
JS calling Go methods:
  fetch("/wails/runtime", {
    method: "POST",
    body: JSON.stringify({methodID: xxx, args: [...]})
  })
    ↓
  HTTP Transport receives (or custom Transport)
    ↓
  MessageProcessor.ProcessMessage()
    ↓
  bindings.go → reflection-invoked Service method
    ↓
  JSON response returned
    ↓
  JS Promise resolve/reject

Go executing JS:
  win.ExecJS(js)
    ↓
  Platform WebView implementation → evaluateJavaScript()
    ↓
  JS executed in WebView

Event flow (Go → JS):
  app.Event.Emit(...)
    ↓
  WailsEventListener.DispatchWailsEvent()
    ↓
  Transport layer broadcasts (HTTP/WebSocket/IPC)
    ↓
  JS receives event
```

Key files:
- `v3/pkg/application/bindings.go` — Service binding reflection
- `v3/pkg/application/messageprocessor.go` — Message routing
- `v3/pkg/application/transport.go` — Transport interface definition
- `v3/pkg/application/transport_http.go` — Default HTTP Transport
- `v3/pkg/application/websocket_server.go` — WebSocket Transport

---

## Asset Server

The Asset Server serves embedded frontend resources to the WebView through a custom URL scheme.

### v2 Asset Server

- Custom scheme: `wails://wails/`
- Middleware support: `assetserver.Options.Middleware`
- Dev mode: Auto-detect dev server URL, proxy to Vite/webpack
- Key files: `v2/pkg/assetserver/assetserver.go`, `v2/internal/assetserver/webview/`

### v3 Asset Server

- Custom scheme: `wails://`
- Fully based on standard `http.Handler`:
  ```go
  application.BundledAssetFileServer(assets)  // Embedded FS + auto-injected runtime.js
  application.AssetFileServerFS(assets)       // Embedded FS only
  ```
- Middleware support: `AssetOptions.Middleware` (standard `func(http.Handler) http.Handler`)
- Chainable: `application.ChainMiddleware(m1, m2, m3)`
- Service routing: If a Service implements `http.Handler`, mount at `ServiceOptions.Route`
- `/wails/runtime.js` auto-served (injected by `BundledAssetFileServer`)
- Key files: `v3/internal/assetserver/`

---

## Platform WebView Backends

### macOS (WKWebView)

- **Engine**: System-built-in WKWebView (WebKit)
- **Language**: Go + CGO → Objective-C
- **Message bridge**: `WKScriptMessageHandler` protocol
- **Asset loading**: `WKURLSchemeHandler` protocol intercepting `wails://` requests
- **Window management**: Cocoa NSWindow + NSApplication
- **Key files**:
  - `v3/pkg/application/application_darwin.go`
  - `v3/pkg/application/webview_window_darwin.go`
  - `v3/pkg/application/application_darwin.m` (ObjC implementation)

### Linux (GTK4 + WebKitGTK 6.0)

- **Engine**: WebKitGTK 6.0 (requires `libwebkitgtk-6.0-dev`)
- **Language**: Go + CGO → C (GTK4/WebKitGTK API)
- **Message bridge**: `webkit_user_content_manager_register_script_message_handler()`
- **Asset loading**: `webkit_web_context_register_uri_scheme()` intercepting `wails://`
- **Window management**: GTK4 GtkWindow + GtkApplication
- **Build tags**:
  - Default: GTK4 + WebKitGTK 6.0
  - `-tags gtk3`: GTK3 + WebKit2GTK 4.1 (legacy support, removed in v3.1)
- **Key files**:
  - `v3/pkg/application/linux_cgo.go` (~55723 lines, GTK4 default)
  - `v3/pkg/application/linux_cgo_gtk3.go` (~74534 lines, GTK3 legacy)
  - `v3/pkg/application/linux_cgo.c` (C helper functions)
  - `v3/pkg/application/linux_cgo.h`
- **Known limitations**:
  - Window positioning is a no-op under Wayland (`SetPosition`/`Center`)
  - File dialog `ShowHiddenFiles`/`CanCreateDirectories` controlled by Portal, not the app

### Windows (WebView2)

- **Engine**: Microsoft Edge WebView2 (Chromium-based)
- **Language**: Go + CGO → Win32 API + WebView2 COM
- **Message bridge**: `ICoreWebView2::add_WebMessageReceived()`
- **Asset loading**: Custom URL scheme or virtual host name mapping
- **Window management**: Win32 CreateWindow + WndProc
- **Key files**:
  - `v3/pkg/application/application_windows.go`
  - `v3/pkg/application/webview_window_windows.go` (~86829 lines)
  - `v3/pkg/w32/` (Win32 API bindings)

### iOS (WKWebView)

- **Engine**: System-built-in WKWebView
- **Language**: Go + CGO → Objective-C
- **Key files**:
  - `v3/pkg/application/application_ios.go`
  - `v3/pkg/application/application_ios.m`
  - `v3/pkg/application/mobile_features_ios.m`
- **Reference**: `IOS_ARCHITECTURE.md`

### Android (Android WebView)

- **Engine**: Android System WebView
- **Key files**:
  - `v3/pkg/application/application_android.go`
  - `v3/pkg/application/mobile_features_android.go`
- **Reference**: `ANDROID.md`

---

## CGO Bridge Layer

CGO is Wails' lowest-level bridge mechanism, responsible for Go runtime ↔ native platform interop.

### Communication Pattern

```
Go goroutine                    C/ObjC thread (main thread)
     │                                │
     │  InvokeSync(fn)                │
     ├──────────→  channel  ─────────→│  Execute fn
     │          (block)               │
     │←──────────  done  ────────────┤
     │                                │
     │  InvokeAsync(fn)               │
     ├──────────→  channel  ─────────→│  Execute fn
     │  (return immediately)          │
```

### v3 Main Thread Dispatch

```go
// Synchronous execution (blocks until done)
application.InvokeSync(func() {
    // Runs on UI main thread
})

// Asynchronous execution
application.InvokeAsync(func() {
    // Runs on UI main thread
})
```

### CGO Export Pattern (Linux example)

```go
//export onProcessRequest
func onProcessRequest(request *C.WebKitURISchemeRequest, userData C.gpointer) {
    // Receive resource request from WebKit
}

//export sendMessageToBackend
func sendMessageToBackend(manager *C.WebKitUserContentManager, jsResult *C.WebKitJavascriptResult, userData C.gpointer) {
    // Receive message from JS
}
```

---

## v3 Pluggable Transport Architecture

v3 abstracts IPC communication into a `Transport` interface:

```go
type Transport interface {
    Start(ctx context.Context, processor *MessageProcessor) error
    Stop() error
}
```

### Built-in Transports

1. **HTTPTransport** (default):
   - Frontend sends JSON messages via `fetch("/wails/runtime")` POST
   - Go pushes events via WebView `ExecJS`

2. **WebSocketTransport**:
   - Frontend connects via WebSocket
   - Bidirectional real-time communication
   - Suitable for high-frequency data push scenarios

3. **Custom Transport**:
   - Implement the `Transport` interface
   - Can be used for: custom protocols, inter-process communication, message queues, etc.

### WailsEventListener Interface

Transports can implement `WailsEventListener` to receive event broadcasts:

```go
type WailsEventListener interface {
    DispatchWailsEvent(event *CustomEvent)
}
```

---

## v3 Server Mode

Build with `-tags server` to run the application as a headless HTTP server with no native window dependency.

```go
application.Options{
    Server: application.ServerOptions{
        Host:           "0.0.0.0",
        Port:           8080,
        ReadTimeout:    30 * time.Second,
        WriteTimeout:   30 * time.Second,
        IdleTimeout:    120 * time.Second,
        ShutdownTimeout: 30 * time.Second,
        TLS: &application.TLSOptions{
            CertFile: "/path/to/cert.pem",
            KeyFile:  "/path/to/key.pem",
        },
    },
}
```

- Same codebase, desktop mode or server mode
- Suitable for Docker/container deployment, server-side rendering, web-only access

---

## Platform Differences & Constraints

| Feature | macOS | Windows | Linux (GTK4) | Linux (GTK3) |
|---------|-------|---------|-------------|-------------|
| Absolute window positioning | ✅ | ✅ | ⚠️ Wayland NO-OP | ⚠️ Wayland NO-OP |
| Transparent background | ✅ | ✅ | ✅ | ✅ |
| Frosted glass effect | ✅ | ❌ | ❌ | ❌ |
| Frameless window | ✅ | ✅ | ✅ | ✅ |
| Content protection (anti-screenshot) | ✅ | ✅ | ❌ | ❌ |
| Ignore mouse events | ✅ | ✅ | ❌ | ❌ |
| File dialog `ShowHiddenFiles` | ✅ | ✅ | ⚠️ Portal-controlled | ⚠️ Portal-controlled |
| File dialog `CanCreateDirectories` | ✅ | ✅ | ⚠️ Portal-controlled | ✅ |
| System tray | ✅ | ✅ | ✅ | ✅ |
| Notifications | ✅ | ✅ | ✅ | ✅ |
| Menu bar | ✅ (native) | ✅ (in-window) | ✅ (in-window) | ✅ (in-window) |
| WebView version | System WKWebView | Edge WebView2 | WebKitGTK 6.0 | WebKit2GTK 4.1 |
| WebView auto-update | OS updates | Automatic | Package manager | Package manager |
| Cross-compilation | ❌ (requires macOS) | ✅ | ✅ (Docker) | ✅ (Docker) |
