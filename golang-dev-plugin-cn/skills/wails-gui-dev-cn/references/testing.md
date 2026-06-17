# Wails 测试与调试

本文档覆盖 Wails v2 和 v3 的测试方法、调试工具链和常见问题排查。

## 目录

- [开发调试](#开发调试)
  - [dev 模式](#dev-模式)
  - [WebView DevTools](#webview-devtools)
  - [日志调试](#日志调试)
- [单元测试](#单元测试)
  - [v2 测试](#v2-测试)
  - [v3 测试](#v3-测试)
- [构建验证测试](#构建验证测试)
- [Panic 恢复与错误处理](#panic-恢复与错误处理)
- [性能分析](#性能分析)
- [常见调试问题](#常见调试问题)

---

## 开发调试

### dev 模式

```bash
wails dev         # v2
wails3 dev        # v3
```

dev 模式特性：
- **前端热重载**：Vite/webpack HMR 即时更新
- **Go 热重载**：Go 代码修改后自动重新编译和重启
- **DevTools 自动启用**：可右键打开浏览器开发者工具
- **控制台日志**：Go 和 JS 日志输出到终端

### WebView DevTools

各平台打开方式：

| 平台 | 方式 |
|------|------|
| macOS | Safari → Develop → [你的应用名] → 选择页面 |
| Windows | 右键 WebView → Inspect（或 F12） |
| Linux | WebKit inspector（需 `webkit_web_inspector_show()`） |

**v3 自动打开 devtools**：
```go
app.Window.NewWithOptions(application.WebviewWindowOptions{
    OpenInspectorOnStartup: true,
})
```

**v3 代码中触发**：
```go
win.OpenDevTools()
```

**v2 代码中触发**：
```go
// 通过前端 JS 方式
window.wails.OpenInspector()
```

### 日志调试

**v2 日志**：
```go
// 设置日志级别
runtime.LogSetLogLevel(ctx, logger.DEBUG)
runtime.LogDebug(ctx, "debug message")
runtime.LogDebugf(ctx, "count: %d", count)

// 前端
window.wails.LogLevel("debug")
window.wails.LogDebug("frontend debug")
```

**v3 日志**：
```go
// 创建带级别的 Logger
app := application.New(application.Options{
    Logger:  application.DefaultLogger(slog.LevelDebug),
    LogLevel: slog.LevelDebug,
})

// 使用
app.Logger.Info("message", "key", value)
app.Logger.Debug("debug", "count", 42)
app.Logger.Error("error", "err", err)
```

**v3 条件编译日志**：
```go
// 仅在非 production 构建中启用
// 使用 application_debug.go 中的 isDebugMode
```

**v3 GTK 调试**（Linux）：
```bash
# 编译时启用 GTK 调试输出
CGO_CFLAGS="-DWAILS_GTK_DEBUG" go build ...
```

---

## 单元测试

### v2 测试

**测试 runtime API**：
```go
// runtime API 依赖 context.Context 中的 frontend/logger/events
// 测试时需要构造带有 mock 实现的 context

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

**测试绑定方法**：
```go
// 直接测试 Go struct 的方法，不通过 Wails 桥接层
// Wails 绑定的 Go 方法是纯函数，可以直接单元测试
```

**v2 内部测试**：
- `v2/internal/binding/binding_test/` — 绑定系统测试
- `v2/pkg/options/options_test.go` — 选项合并测试
- `v2/pkg/commands/buildtags/buildtags_test.go` — 构建标签测试

### v3 测试

**测试 Service 方法**：
```go
// Service 方法是纯 Go 函数，直接测试即可
func TestGreetService_Greet(t *testing.T) {
    s := &GreetService{}
    result := s.Greet("world")
    assert.Equal(t, "Hello, world", result)
}

// 测试 ServiceStartup/ServiceShutdown
func TestServiceLifecycle(t *testing.T) {
    s := &MyService{}
    ctx := context.Background()
    err := s.ServiceStartup(ctx, application.DefaultServiceOptions)
    require.NoError(t, err)
    
    // 使用服务...
    
    err = s.ServiceShutdown()
    require.NoError(t, err)
}
```

**v3 内部测试**：
- `v3/pkg/application/services_test.go` — 服务注册/生命周期
- `v3/pkg/application/bindings_test.go` — 绑定调用
- `v3/pkg/application/events_test.go` — 事件系统
- `v3/pkg/application/options_test.go` — 选项处理
- `v3/pkg/application/webview_window_test.go` — 窗口管理
- `v3/pkg/application/internal/tests/services/` — 服务生命周期集成测试

**v3 Benchmark 测试**：
```bash
go test -bench=. ./v3/pkg/application/...
```
关键 benchmark 文件：
- `bindings_bench_test.go` — 绑定调用性能
- `bindings_optimized_bench_test.go` — 优化路径性能
- `events_bench_test.go` — 事件发射/消费性能
- `json_libs_bench_test.go` / `json_v2_bench_test.go` — JSON 序列化性能

---

## 构建验证测试

v3 提供全面的跨平台构建验证（`v3/Taskfile.yaml`）：

```bash
# 所有平台（43 示例 × 3 平台 = 129 次构建）
task test:examples:all

# 当前平台
task test:examples

# CLI 代码测试
task test:cli

# 单示例
task test:example:darwin DIR=badge
task test:example:windows DIR=badge
task test:example:linux DIR=badge
task test:example:linux:docker DIR=badge
```

输出二进制命名规则：
- macOS：`testbuild-{name}-darwin`
- Windows：`testbuild-{name}-windows.exe`
- Linux：`testbuild-{name}-linux`

---

## Panic 恢复与错误处理

### v2

默认在消息处理中捕获 panic，防止整个应用崩溃。
```go
options.App{
    DisablePanicRecovery: false,  // 默认 false = 启用恢复
}
```

### v3

自定义 Panic 处理：
```go
application.Options{
    PanicHandler: func(details *application.PanicDetails) {
        // details.Error  — panic 值
        // details.Stack  — 堆栈跟踪
        // details.Window — 发生 panic 的窗口（可能 nil）
        log.Printf("Panic in window %v: %v\nStack:\n%s",
            details.Window, details.Error, details.Stack)
    },
}
```

错误处理：
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

## 性能分析

### 运行时性能

- Wails 本身开销极小：Go 方法调用 ≈ 直接反射调用 + JSON 序列化
- WebView 性能 = 底层浏览器引擎性能（Chromium/WebKit）
- 前端渲染性能取决于 Web 框架（React/Vue/Svelte 等）

### 优化要点

1. **减少 Go ↔ JS 调用频率**：批量处理数据，避免频繁小消息
2. **使用合适的数据结构**：大列表考虑分页或虚拟滚动
3. **v3 Transport 选择**：高频数据推送考虑 WebSocket Transport
4. **JSON 序列化**：v3 提供多个 JSON 库 benchmark，选择最快方案

### Benchmark 运行

```bash
# v3 JSON 序列化对比
go test -bench=JSON -benchmem ./v3/pkg/application/

# v3 绑定调用
go test -bench=Binding -benchmem ./v3/pkg/application/

# v3 事件系统
go test -bench=Event -benchmem ./v3/pkg/application/
```

---

## 常见调试问题

### 1. 前端白屏，无错误信息

**排查步骤**：
```bash
# 1. 检查 embed 路径
# 确认 //go:embed all:frontend/dist 路径正确
# 确认 dist/ 目录存在且包含 index.html

# 2. 检查 asset server
# 使用 curl 测试（需先在 dev 模式运行）
curl http://localhost:34115/

# 3. 检查控制台
# 打开 WebView DevTools，查看 Console 和 Network 标签

# 4. 检查构建
wails build -debug  # 保留 devtools
```

### 2. Go 方法前端调用失败

**排查步骤**：
```go
// 1. 确认方法名大写
func (s *Service) Greet(name string) string  // ✅
func (s *Service) greet(name string) string  // ❌

// 2. 确认参数类型支持 JSON
// ✅ string, int, bool, float, struct（有 json tag）, slice, map
// ❌ chan, func, complex, unsafe.Pointer

// 3. v2 确认 struct 在 Bind 中注册
options.App{Bind: []interface{}{&MyService{}}}

// 4. v3 确认 Service 在 Services 列表中
application.Options{Services: []application.Service{
    application.NewService(&MyService{}),
}}

// 5. 检查前端调用路径
// v2: import {Method} from "../wailsjs/go/main/MyService"
// 确认 wailsjs 已生成（运行 wails dev 或 wails build）
```

### 3. 热重载不工作

```bash
# 1. 确认 dev server 运行
npm run dev    # 在 frontend/ 目录下

# 2. 检查 wails.json 配置
"frontend:dev:serverUrl": "auto"

# 3. 手动指定 URL
"frontend:dev:serverUrl": "http://localhost:5173"

# 4. 使用环境变量
FRONTEND_DEVSERVER_URL=http://localhost:5173 wails dev
```

### 4. Linux 编译错误

```bash
# 检查依赖
wails3 doctor

# Ubuntu/Debian v3（GTK4 默认）
sudo apt install libgtk-4-dev libwebkitgtk-6.0-dev

# Ubuntu/Debian v3（GTK3 遗留）
sudo apt install libgtk-3-dev libwebkit2gtk-4.1-dev

# 构建时指定 GTK3
go build -tags gtk3
```

### 5. Windows WebView2 错误

```bash
# 安装 WebView2 Runtime
# https://developer.microsoft.com/en-us/microsoft-edge/webview2/

# 嵌入 WebView2 Bootstrapper
wails build -webview2 embed

# 检查 WebView2 版本
wails doctor
```

### 6. macOS 代码签名

```bash
# 开发签名
wails build -sign

# 指定证书
wails build -sign -signid "Developer ID Application: ..."

# 跳过签名（仅开发）
wails build -skip-sign
```
