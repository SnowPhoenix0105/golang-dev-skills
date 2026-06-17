# Wails 架构与实现原理

本文档深入解析 Wails 的内部架构、通信机制和平台差异。

## 目录

- [整体架构分层](#整体架构分层)
- [Go ↔ JS 通信机制](#go--js-通信机制)
  - [v2 通信路径](#v2-通信路径)
  - [v3 通信路径](#v3-通信路径)
- [Asset Server（资源服务）](#asset-server资源服务)
- [平台 WebView 后端](#平台-webview-后端)
- [CGO 桥接层](#cgo-桥接层)
- [v3 可插拔 Transport 架构](#v3-可插拔-transport-架构)
- [v3 服务器模式](#v3-服务器模式)
- [平台差异与约束](#平台差异与约束)

---

## 整体架构分层

```
┌────────────────────────────────────────────────────────┐
│          应用层（Application Layer）                    │
│  v2: wails.Run(&options.App{})                        │
│  v3: application.New(opts) → app.Run()                │
├────────────────────────────────────────────────────────┤
│          API 层（非平台特定 Go 代码）                    │
│  v2: pkg/runtime/    pkg/options/   pkg/menu/         │
│  v3: pkg/application/  pkg/events/  pkg/services/     │
├────────────────────────────────────────────────────────┤
│          内部层（Internal Layer）                       │
│  v2: internal/frontend/  internal/binding/            │
│      internal/assetserver/  internal/menumanager/     │
│  v3: internal/assetserver/  internal/capabilities/    │
│      internal/operatingsystem/                         │
├────────────────────────────────────────────────────────┤
│          CGO 桥接层（平台特定 C/ObjC 代码）              │
│  v2: internal/frontend/desktop/darwin/  (ObjC)        │
│      internal/frontend/desktop/linux/   (C + GTK)     │
│      internal/frontend/desktop/windows/ (C + Win32)    │
│  v3: pkg/application/linux_cgo.go      (C + GTK4)     │
│      pkg/application/*_darwin.m        (ObjC)         │
│      pkg/application/*_windows.go      (C + Win32)    │
├────────────────────────────────────────────────────────┤
│          原生层（Native Layer）                          │
│  macOS:   Cocoa + WKWebView                           │
│  Linux:   GTK4 + WebKitGTK 6.0 (默认)                 │
│           GTK3 + WebKit2GTK 4.1 (-tags gtk3)          │
│  Windows: Win32 + WebView2 (Edge/Chromium)            │
│  iOS:     UIKit + WKWebView                           │
│  Android: Android WebView                             │
└────────────────────────────────────────────────────────┘
```

---

## Go ↔ JS 通信机制

### v2 通信路径

```
JS 端调用 Go 方法:
  window.wails.Call(methodName, args...)
    ↓
  WebView message handler 接收
    ↓
  CGO 导出函数 (processMessage)
    ↓
  channel → Go goroutine 处理
    ↓
  MessageProcessor.ProcessMessage()
    ↓
  binding.DB.Call() → 反射调用 Go 方法
    ↓
  返回结果序列化为 JSON
    ↓
  CGO → window.wails.Callback(id, result)
    ↓
  JS Promise resolve/reject

Go 端执行 JS:
  frontend.ExecJS(js)
    ↓
  CGO → WebView.evaluateJavaScript()
    ↓
  JS 在 WebView 中执行
```

关键文件：
- `v2/internal/frontend/desktop/darwin/frontend.go` — macOS 前端实现
- `v2/internal/frontend/desktop/linux/frontend.go` — Linux 前端实现
- `v2/internal/frontend/desktop/windows/frontend.go` — Windows 前端实现
- `v2/internal/binding/binding.go` — 绑定反射调用核心（10015 行）
- `v2/internal/frontend/dispatcher/dispatcher.go` — 消息路由

### v3 通信路径

```
JS 端调用 Go 方法:
  fetch("/wails/runtime", {
    method: "POST",
    body: JSON.stringify({methodID: xxx, args: [...]})
  })
    ↓
  HTTP Transport 接收（或自定义 Transport）
    ↓
  MessageProcessor.ProcessMessage()
    ↓
  bindings.go → 反射调用 Service 方法
    ↓
  返回 JSON response
    ↓
  JS Promise resolve/reject

Go 端执行 JS:
  win.ExecJS(js)
    ↓
  平台 WebView 实现 → evaluateJavaScript()
    ↓
  JS 在 WebView 中执行

事件流（Go → JS）:
  app.Event.Emit(...)
    ↓
  WailsEventListener.DispatchWailsEvent()
    ↓
  Transport 层广播（HTTP/WebSocket/IPC）
    ↓
  JS 端接收事件
```

关键文件：
- `v3/pkg/application/bindings.go` — 服务绑定反射调用
- `v3/pkg/application/messageprocessor.go` — 消息路由
- `v3/pkg/application/transport.go` — Transport 接口定义
- `v3/pkg/application/transport_http.go` — 默认 HTTP Transport
- `v3/pkg/application/websocket_server.go` — WebSocket Transport

---

## Asset Server（资源服务）

Asset Server 将嵌入的前端资源通过自定义 URL scheme 提供给 WebView。

### v2 Asset Server

- 自定义 scheme：`wails://wails/`
- 中间件支持：`assetserver.Options.Middleware`
- 开发模式：Auto-detect dev server URL，代理到 Vite/webpack
- 关键文件：`v2/pkg/assetserver/assetserver.go`、`v2/internal/assetserver/webview/`

### v3 Asset Server

- 自定义 scheme：`wails://`
- 完全基于标准 `http.Handler`：
  ```go
  application.BundledAssetFileServer(assets)  // 内嵌 FS + 自动注入 runtime.js
  application.AssetFileServerFS(assets)       // 仅内嵌 FS
  ```
- 中间件支持：`AssetOptions.Middleware`（标准 `func(http.Handler) http.Handler`）
- 可链式组合：`application.ChainMiddleware(m1, m2, m3)`
- 服务路由支持：如果 Service 实现了 `http.Handler`，可挂载到 `ServiceOptions.Route`
- `/wails/runtime.js` 自动提供（由 `BundledAssetFileServer` 注入）
- 关键文件：`v3/internal/assetserver/`

---

## 平台 WebView 后端

### macOS（WKWebView）

- **引擎**：系统内置 WKWebView（WebKit）
- **语言**：Go + CGO → Objective-C
- **消息桥接**：`WKScriptMessageHandler` 协议
- **资源加载**：`WKURLSchemeHandler` 协议拦截 `wails://` 请求
- **窗口管理**：Cocoa NSWindow + NSApplication
- **关键文件**：
  - `v3/pkg/application/application_darwin.go`
  - `v3/pkg/application/webview_window_darwin.go`
  - `v3/pkg/application/application_darwin.m`（ObjC 实现）

### Linux（GTK4 + WebKitGTK 6.0）

- **引擎**：WebKitGTK 6.0（需要 `libwebkitgtk-6.0-dev`）
- **语言**：Go + CGO → C（GTK4/WebKitGTK API）
- **消息桥接**：`webkit_user_content_manager_register_script_message_handler()`
- **资源加载**：`webkit_web_context_register_uri_scheme()` 拦截 `wails://`
- **窗口管理**：GTK4 GtkWindow + GtkApplication
- **构建标签**：
  - 默认：GTK4 + WebKitGTK 6.0
  - `-tags gtk3`：GTK3 + WebKit2GTK 4.1（遗留支持，v3.1 将移除）
- **关键文件**：
  - `v3/pkg/application/linux_cgo.go`（~55723 行，GTK4 默认）
  - `v3/pkg/application/linux_cgo_gtk3.go`（~74534 行，GTK3 遗留）
  - `v3/pkg/application/linux_cgo.c`（C 辅助函数）
  - `v3/pkg/application/linux_cgo.h`
- **已知限制**：
  - Wayland 下窗口定位无效（`SetPosition`/`Center` 为 NO-OP）
  - 文件对话框的 `ShowHiddenFiles`/`CanCreateDirectories` 由 Portal 控制，应用无法覆写

### Windows（WebView2）

- **引擎**：Microsoft Edge WebView2（Chromium 内核）
- **语言**：Go + CGO → Win32 API + WebView2 COM
- **消息桥接**：`ICoreWebView2::add_WebMessageReceived()`
- **资源加载**：自定义 URL scheme 或虚拟主机名映射
- **窗口管理**：Win32 CreateWindow + WndProc
- **关键文件**：
  - `v3/pkg/application/application_windows.go`
  - `v3/pkg/application/webview_window_windows.go`（~86829 行）
  - `v3/pkg/w32/`（Win32 API 绑定）

### iOS（WKWebView）

- **引擎**：系统内置 WKWebView
- **语言**：Go + CGO → Objective-C
- **关键文件**：
  - `v3/pkg/application/application_ios.go`
  - `v3/pkg/application/application_ios.m`
  - `v3/pkg/application/mobile_features_ios.m`
- **参考**：`IOS_ARCHITECTURE.md`

### Android（Android WebView）

- **引擎**：Android System WebView
- **关键文件**：
  - `v3/pkg/application/application_android.go`
  - `v3/pkg/application/mobile_features_android.go`
- **参考**：`ANDROID.md`

---

## CGO 桥接层

CGO 是 Wails 最底层的桥接机制，负责 Go 运行时与原生平台的互操作。

### 通信模式

```
Go goroutine                    C/ObjC thread (main thread)
     │                                │
     │  InvokeSync(fn)                │
     ├──────────→  channel  ─────────→│  执行 fn
     │          (block)               │
     │←──────────  done  ────────────┤
     │                                │
     │  InvokeAsync(fn)               │
     ├──────────→  channel  ─────────→│  执行 fn
     │  (return immediately)          │
```

### v3 主线程调度

```go
// 同步执行（阻塞等待结果）
application.InvokeSync(func() {
    // 在 UI 主线程执行
})

// 异步执行
application.InvokeAsync(func() {
    // 在 UI 主线程执行
})
```

### CGO 导出函数模式（以 Linux 为例）

```go
//export onProcessRequest
func onProcessRequest(request *C.WebKitURISchemeRequest, userData C.gpointer) {
    // 从 WebKit 接收资源请求
}

//export sendMessageToBackend
func sendMessageToBackend(manager *C.WebKitUserContentManager, jsResult *C.WebKitJavascriptResult, userData C.gpointer) {
    // 从 JS 接收消息
}
```

---

## v3 可插拔 Transport 架构

v3 将 IPC 通信抽象为 `Transport` 接口：

```go
type Transport interface {
    Start(ctx context.Context, processor *MessageProcessor) error
    Stop() error
}
```

### 内置 Transport

1. **HTTPTransport**（默认）：
   - 前端通过 `fetch("/wails/runtime")` POST JSON 消息
   - Go 端通过 WebView `ExecJS` 推送事件

2. **WebSocketTransport**：
   - 前端通过 WebSocket 连接
   - 双向实时通信
   - 适合高频率数据推送场景

3. **自定义 Transport**：
   - 实现 `Transport` 接口
   - 可用于：自定义协议、进程间通信、消息队列等

### WailsEventListener 接口

Transport 可以实现 `WailsEventListener` 来接收事件广播：

```go
type WailsEventListener interface {
    DispatchWailsEvent(event *CustomEvent)
}
```

---

## v3 服务器模式

通过 `-tags server` 构建，应用作为无头 HTTP 服务器运行，无原生窗口依赖。

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

- 同一份代码，桌面模式或服务器模式
- 适用于 Docker/容器部署、Server-Side Rendering、Web-only 访问

---

## 平台差异与约束

| 特性 | macOS | Windows | Linux (GTK4) | Linux (GTK3) |
|------|-------|---------|-------------|-------------|
| 窗口绝对定位 | ✅ | ✅ | ⚠️ Wayland NO-OP | ⚠️ Wayland NO-OP |
| 透明背景 | ✅ | ✅ | ✅ | ✅ |
| 毛玻璃效果 | ✅ | ❌ | ❌ | ❌ |
| 无边框窗口 | ✅ | ✅ | ✅ | ✅ |
| 内容保护（防截屏） | ✅ | ✅ | ❌ | ❌ |
| 忽略鼠标事件 | ✅ | ✅ | ❌ | ❌ |
| 文件对话框 `ShowHiddenFiles` | ✅ | ✅ | ⚠️ Portal 控制 | ⚠️ Portal 控制 |
| 文件对话框 `CanCreateDirectories` | ✅ | ✅ | ⚠️ Portal 控制 | ✅ |
| 系统托盘 | ✅ | ✅ | ✅ | ✅ |
| 通知 | ✅ | ✅ | ✅ | ✅ |
| 菜单栏 | ✅（原生） | ✅（窗口内） | ✅（窗口内） | ✅（窗口内） |
| WebView 版本 | 系统 WKWebView | Edge WebView2 | WebKitGTK 6.0 | WebKit2GTK 4.1 |
| WebView 自动更新 | 系统更新 | 自动 | 包管理器 | 包管理器 |
| 跨平台构建 | ❌（需 macOS） | ✅ | ✅（Docker） | ✅（Docker） |
