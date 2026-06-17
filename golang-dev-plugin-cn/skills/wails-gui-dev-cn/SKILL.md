---
name: wails-gui-dev-cn
description: Wails 跨平台桌面应用开发框架。Go 后端 + Web 前端（任意框架），通过 CGO 桥接系统原生 WebView 与 Go 运行时，打包为单一二进制。当用户需要创建、修改、调试 Wails 桌面应用时使用此技能。触发场景：提到 wails、Go 桌面应用、WebView 桌面开发、Go + React/Vue/Svelte 桌面应用、wails2/wails3、前后端桥接。即使用户没有明确说"wails"，只要在做 Go 桌面 GUI 开发且涉及 Web 技术就应该考虑此技能。
---

# Wails 桌面应用开发

Wails 将 **Go 运行时 + 系统原生 WebView**（不嵌入浏览器）通过 CGO 桥接层打包为单一二进制。前端用任何 Web 框架，后端用 Go，两者通过消息通信。

## 版本：v2（稳定）与 v3（Alpha）

| 版本 | 状态 | CLI | 安装 |
|------|------|-----|------|
| v2 | 稳定 | `wails` | `go install github.com/wailsapp/wails/v2/cmd/wails@latest` |
| v3 | Alpha | `wails3` | `go install github.com/wailsapp/wails/v3/cmd/wails3@latest` |

**关键差异速查**：

| 方面 | v2 | v3 |
|------|-----|-----|
| 绑定 | `Bind: []interface{}{}` | `Services: []Service`（泛型） |
| 日志 | 自定义 `logger.Logger` | 标准库 `log/slog` |
| IPC | 固定 fetch + JS eval | 可插拔 `Transport` 接口 |
| 运行时 API | `context.Context` 注入 | `app.Get()` 全局单例 |
| 事件 | `runtime.EventsOn/Emit(ctx, ...)` | `app.Event.On/Emit()` + Hook |
| 窗口 | 单窗口 | 多窗口原生支持 |
| 菜单 | 函数式 `menu.Text/Checkbox/...` | 链式 `menu.Add().OnClick()` |
| 对话框 | `runtime.OpenFileDialog(ctx, opts)` | `app.Dialog.OpenFile().SetTitle(...).Show()` |
| 移动端 | ❌ | iOS + Android |
| 服务器模式 | ❌ | `-tags server` 无头部署 |

**选择建议**：新项目优先 v3；需要生产稳定性或维护已有项目选 v2。

## 整体架构

```
┌──────────────────────────────────────────────────────────┐
│                    操作系统原生窗口                        │
│  ┌────────────────────────────────────────────────────┐  │
│  │  原生 WebView（macOS: WKWebView / Win: WebView2    │  │
│  │  Linux: WebKitGTK / iOS: WKWebView / Android: WV） │  │
│  │  ┌──────────────────────────────────────────────┐  │  │
│  │  │  前端 (HTML/JS/CSS) — 任意 Web 框架          │  │  │
│  │  │  由 asset server 通过自定义 scheme 加载      │  │  │
│  │  └──────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────┘  │
├──────────────────────────────────────────────────────────┤
│  CGO 桥接层：消息路由 + Asset Server + JS 执行           │
├──────────────────────────────────────────────────────────┤
│  Go 运行时：App / Services / Events / Window / 原生 API  │
└──────────────────────────────────────────────────────────┘
```

### 通信机制

1. **JS → Go**：v2 通过 `window.wails.Call()`，v3 通过 `fetch("/wails/runtime")` 发送 JSON 消息，MessageProcessor 路由到绑定的 Go 方法
2. **Go → JS**：`win.ExecJS(js)` 在 WebView 中执行 JS；或通过事件系统推送
3. **资产加载**：前端资源通过自定义 URL scheme（`wails://`）由 Asset Server 从 `//go:embed` 嵌入的文件系统提供服务，**无需网络端口**
4. **事件系统**：Go 和 JS 共享统一事件机制，v3 支持 Hook（可在事件前拦截并取消）

详细架构（CGO 桥接、各平台 WebView 后端、Transport 架构、服务器模式）见 `references/architecture.md`。

## 核心原则

1. **服务即后端 API** — Go struct 的公开方法自动暴露给前端，无需手动定义路由
2. **Go 管逻辑，JS 管界面** — 清晰的职责边界（见下一节）
3. **前后端完全分离** — 前端可以是 React/Vue/Svelte/Lit/Vanilla，构建产物通过 `//go:embed` 嵌入
4. **主线程序列化（v3）** — UI 操作通过 `InvokeSync`/`InvokeAsync` 调度到主线程；v2 内部已透明处理
5. **原生能力不妥协** — 菜单、对话框、系统托盘、通知均使用平台原生 API

## Go 与 JS 分工边界

Wails 应用中 Go 和 JS 各有明确职责。错误的分配会导致架构混乱、性能下降或安全风险。

### 放在 Go 里（后端）

```
✅ 业务逻辑和规则
✅ 数据库操作（SQLite、Postgres 等）
✅ 文件系统读写（本地文件、配置）
✅ 系统调用和 OS 集成（进程管理、注册表、权限）
✅ 原生对话框、菜单、系统托盘
✅ 加密、认证、敏感数据处理
✅ 性能密集型计算（大数据处理、图像编码）
✅ 外部 API 调用（需要密钥保护的）
✅ 剪贴板读写
✅ 多窗口管理
✅ 硬件访问（摄像头、麦克风、串口）
```

**原则**：需要 OS 能力、涉及安全、或属于业务核心的，放 Go。

### 放在 JS 里（前端）

```
✅ UI 渲染和 DOM 操作
✅ 用户交互（点击、拖拽、输入验证）
✅ 动画和过渡效果
✅ 响应式布局
✅ 主题切换和样式
✅ 客户端表单验证（即时反馈）
✅ 图表和可视化
✅ 路由（前端页面切换）
✅ 加载状态和骨架屏
```

**原则**：纯视觉、即时反馈、无需 OS 能力的，放 JS。

### 需要协作的场景

| 场景 | Go 负责 | JS 负责 |
|------|---------|---------|
| 打开文件 | 调用原生文件对话框，读取文件内容，返回给 JS | 展示文件选择按钮，接收并渲染文件内容 |
| 保存设置 | 接收 JS 传来的设置数据，写入本地文件/数据库 | 收集表单数据，调用 Go 方法保存，显示成功提示 |
| 搜索 | 执行搜索逻辑，查询数据库，返回结果 | 显示搜索框，展示搜索结果列表 |
| 文件拖放 | 接收文件路径列表，处理文件 | 渲染拖放区域，显示拖放反馈动画 |
| 系统主题 | 检测系统主题变化，通知 JS | 切换 CSS 变量/样式 |
| 通知 | 调用系统通知 API | 提供触发通知的 UI 按钮 |

### 决策流程

```
这个功能...
├─ 需要 OS 权限/原生 API？ → Go
├─ 纯 UI 展示/动画？       → JS
├─ 涉及密钥/敏感数据？     → Go
├─ 用户即时交互反馈？       → JS（需要数据时调 Go）
├─ 大量计算？              → Go（通过事件将结果推给 JS）
└─ 不确定？                → 默认放 Go（安全侧），通过事件/方法暴露给 JS
```

## 最小应用

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

## 项目结构

```
myapp/
├── frontend/              # 前端（Vite 项目，任意框架）
│   ├── src/               # 源码（React/Vue/Svelte/Lit/Vanilla）
│   ├── dist/              # 构建产物 → Go embed 打包
│   ├── package.json
│   ├── vite.config.js
│   └── wailsjs/           # ⚡ 自动生成的 JS 绑定（运行 wails dev/build 后）
│       ├── go/             #    Go 方法的 JS 包装 + TypeScript 类型
│       └── runtime/        #    运行时类型定义
├── app.go                 # Go 后端入口
├── main.go                # main() 调用 app.go
├── wails.json             # Wails 配置
├── go.mod
└── build/                 # 平台资源（图标、Info.plist、NSIS 脚本）
```

## 核心 API 速览

以下为最常用的 API 入口，完整列表见 `references/api-reference.md`。

### App 创建（v3）

```go
app := application.New(application.Options{
    Name:        "My App",
    Description: "描述",
    Services:    []application.Service{application.NewService(&MyService{})},
    Assets:      application.AssetOptions{Handler: application.BundledAssetFileServer(assets)},
    Logger:      application.DefaultLogger(slog.LevelDebug),
    // 生命周期
    OnShutdown:  func() { /* 清理 */ },
    ShouldQuit:  func() bool { return true },
    // 平台选项
    Mac:     application.MacOptions{ApplicationShouldTerminateAfterLastWindowClosed: true},
    Windows: application.WindowsOptions{},
    Linux:   application.LinuxOptions{},
})
```

完整选项（`KeyBindings`、`PanicHandler`、`SingleInstance`、`Transport`、`Server`、`IOS`/`Android` 等）见 `references/api-reference.md`。

### 服务绑定

**v2**：`Bind: []interface{}{&MyService{}}` 注册 struct，所有公开方法暴露给前端。
**v3**：`application.NewService(&MyService{})` 泛型注册，可选实现 `ServiceStartup`/`ServiceShutdown`/`ServiceName` 接口。

方法签名约束：首字母大写，参数/返回值需支持 JSON 序列化。详见 `references/api-reference.md` 服务与绑定章节。

### 窗口（v3）

```go
win := app.Window.NewWithOptions(application.WebviewWindowOptions{
    Title: "窗口标题", Width: 1024, Height: 768,
    MinWidth: 400, MinHeight: 300,
    Frameless: false, AlwaysOnTop: false,
    DevToolsEnabled: true,                  // 非 production 构建默认开启
    OpenInspectorOnStartup: false,
    Mac:     application.MacWindow{TitleBar: application.MacTitleBarHiddenInsetUnified},
    Windows: application.WindowsWindow{},
    Linux:   application.LinuxWindow{},
})
```

窗口操作：`win.SetTitle()`、`win.Center()`、`win.Maximise()`、`win.Minimise()`、`win.Fullscreen()`、`win.SetSize()`、`win.Show()`/`win.Hide()`、`win.Close()`、`win.ExecJS()`、`win.Reload()`、`win.SetAlwaysOnTop()`、`win.ZoomIn()`/`win.ZoomOut()` 等 60+ 方法。完整列表见 `references/api-reference.md`。

### 对话框（v3）

链式 API，所有平台使用相同的 Go 代码：

```go
// 消息对话框
app.Dialog.Info().SetTitle("提示").SetMessage("操作完成").Show()
app.Dialog.Question().SetTitle("确认").SetMessage("确定删除？").
    AddButton("确定").OnClick(func() { /* ... */ }).
    AddButton("取消").Show()
app.Dialog.Warning().SetMessage("警告内容").Show()
app.Dialog.Error().SetMessage("错误内容").Show()

// 文件对话框
path, _ := app.Dialog.OpenFile().
    AddFilter("文本文件", "*.txt;*.md").
    PromptForSingleSelection()
paths, _ := app.Dialog.OpenFile().PromptForMultipleSelection()
dir, _ := app.Dialog.OpenFile().CanChooseDirectories(true).CanChooseFiles(false).PromptForSingleSelection()
savePath, _ := app.Dialog.SaveFile().SetFilename("README.md").PromptForSingleSelection()
```

完整对话框 API（`SetButtonText`、`ShowHiddenFiles`、`AttachToWindow`、`SetDirectory` 等）见 `references/api-reference.md`。

### 菜单（v3）

```go
menu := app.NewMenu()
menu.AddRole(application.AppMenu)    // macOS 应用菜单
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

// 上下文菜单
ctxMenu := application.NewContextMenu("my-menu")
ctxMenu.Add("Copy").OnClick(func(ctx *application.Context) { /* ... */ })
ctxMenu.Update()
```

### 事件系统（v3）

```go
// 自定义事件
app.Event.On("myevent", func(e *application.CustomEvent) {
    app.Logger.Info("收到事件", "name", e.Name, "data", e.Data)
    e.Cancel()  // 可取消
})
app.Event.Emit("myevent", "data")

// 系统级 Hook（可拦截取消）
win.RegisterHook(events.Common.WindowClosing, func(e *application.WindowEvent) {
    if unsaved { e.Cancel() }  // 阻止关闭
})

// 窗口事件监听
win.OnWindowEvent(events.Common.WindowFocus, func(e *application.WindowEvent) {
    app.Logger.Info("窗口获得焦点")
})
```

预定义事件 30+（`WindowClosing`、`WindowDidMove`、`WindowFilesDropped`、`ThemeChanged` 等），完整列表见 `references/api-reference.md`。

### v2 运行时 API 速查

v2 所有运行时操作通过 `context.Context` 注入，从 `OnStartup`/`OnDomReady`/`OnBeforeClose` 回调获取：

```go
// 窗口
runtime.WindowSetTitle(ctx, "title")
runtime.WindowFullscreen(ctx) / runtime.WindowUnfullscreen(ctx)
runtime.WindowMaximise(ctx) / runtime.WindowMinimise(ctx)
runtime.WindowSetSize(ctx, 800, 600)
runtime.WindowCenter(ctx)
runtime.WindowExecJS(ctx, "console.log('hello')")

// 对话框
file, _ := runtime.OpenFileDialog(ctx, runtime.OpenDialogOptions{...})
runtime.SaveFileDialog(ctx, runtime.SaveDialogOptions{...})
runtime.MessageDialog(ctx, runtime.MessageDialogOptions{...})

// 事件
runtime.EventsOn(ctx, "event", func(data ...interface{}) {})
runtime.EventsEmit(ctx, "event", data)

// 日志
runtime.LogInfo(ctx, "msg") / runtime.LogInfof(ctx, "count: %d", n)
runtime.LogDebug(ctx, "msg") / runtime.LogError(ctx, "msg")
```

完整 v2 runtime API（30+ 函数）见 `references/api-reference.md`。

## 编译与部署

```bash
# 系统依赖检查
wails doctor      # v2
wails3 doctor     # v3

# 开发模式（热重载 + devtools）
wails dev         # v2
wails3 dev        # v3

# 生产构建
wails build                          # v2，当前平台
wails build -platform windows/amd64  # v2，指定平台
wails build -nsis                    # v2，+ NSIS 安装包
wails3 build                         # v3
wails3 build -platform darwin/arm64,windows/amd64,linux/amd64
wails3 build -f                      # v3，强制重建
wails3 build -tags server            # v3，服务器模式
```

平台打包产物：
- **macOS**：`.app` bundle（含 `Info.plist`）
- **Windows**：`.exe`，可选 NSIS 安装程序
- **Linux**：二进制 + `.desktop` 文件

交叉编译、Docker 构建、编译标签（`production`/`server`/`gtk3`）详见 `references/build-deploy.md`。

## 测试与调试

### 开发调试

- `wails dev` / `wails3 dev` — 自动启用 devtools
- **WebView DevTools**：macOS（Safari → Develop → 应用名）、Windows（右键 → Inspect）、Linux（WebKit inspector）
- v3 `OpenInspectorOnStartup: true` 自动打开

### 单元测试

- Go Service 方法为纯 Go 函数，直接 `go test` 即可
- v3 提供全面的构建验证：`task test:examples:all`（43 示例 × 3 平台 = 129 次构建）
- Benchmark：`go test -bench=. ./v3/pkg/application/...`

### 常见问题速查

| 问题 | 平台 | 原因 | 解决 |
|------|------|------|------|
| 前端白屏 | 全部 | 资源加载失败 | 检查 `//go:embed` 路径、`frontend:build` 产出目录 |
| Go 方法前端调用无响应 | 全部 | 方法未导出或签名不兼容 | 方法名首字母大写，参数/返回类型支持 JSON |
| v2 `context.Context` 为空 | v2 | 未从生命周期回调获取 | 使用 `OnStartup`/`OnDomReady` 传入的 ctx |
| 编译报 cgo 错误 | Linux | 缺少 GTK/WebKit | `wails doctor`；Ubuntu 需 `apt install libgtk-4-dev libwebkitgtk-6.0-dev`（v3） |
| v3 事件不触发 | v3 | 严格模式未注册 | 在代码中预先注册事件名，或使用宽松模式 |
| 窗口位置无效 | Linux | Wayland 协议限制 | Wayland 下 `SetPosition`/`Center` 可能无效果 |
| 热重载不工作 | 全部 | dev server 未运行 | 确认 `npm run dev` 运行中，检查 `frontend:dev:serverUrl` |
| 拖拽无边框窗口无效 | 全部 | CSS 属性未设置 | HTML 元素上加 `style="--wails-draggable: drag"` |

更多调试技巧、Panic 恢复、Benchmark 指南见 `references/testing.md`。

## 无从下手？先读这些

当 references 覆盖不到时，探索 Wails 源码：

```bash
# 定位源码路径
WAILS_DIR=$(go list -m -json github.com/wailsapp/wails/v3 | grep '"Dir"' | cut -d'"' -f4)

# 查找 API
go doc github.com/wailsapp/wails/v3/pkg/application

# 学习 43 个内置示例
cd $WAILS_DIR/examples && ls
# 关键示例：binding/ dialogs/ events/ menu/ window/ systray-basic/ clipboard/ screen/ plain/
```

v3 代码约定：`New(opts)` → `Run()` 标准入口；`app.Window.NewWithOptions(opts)` 创建窗口；对话框/菜单用链式 API；平台文件按 `*_darwin.go`/`*_windows.go`/`*_linux.go` 命名。

## 参考资料

| 场景 | 查阅文件 |
|------|---------|
| 完整的 API 参数/选项列表 | `references/api-reference.md` |
| 架构原理、通信机制、平台差异 | `references/architecture.md` |
| 编译/交叉编译/打包/Docker | `references/build-deploy.md` |
| 测试/调试/Benchmark/排查 | `references/testing.md` |

上述 references 都覆盖不到时，按照上一节方法探索源码。如果探索源码后仍然无法解决，**向用户报告具体情况并寻求帮助，不得静默猜测。**
