# Wails API 参考（v2 + v3）

本文档提供 Wails v2 和 v3 的完整 API 列表。快速查找用。

## 目录

- [App 创建选项](#app-创建选项)
  - [v2 options.App](#v2-optionsapp)
  - [v3 application.Options](#v3-applicationoptions)
- [服务与绑定](#服务与绑定)
  - [v2 Bind 模式](#v2-bind-模式)
  - [v3 Service 模式](#v3-service-模式)
- [窗口选项（v3）](#窗口选项v3)
  - [WebviewWindowOptions](#webviewwindowoptions)
  - [MacWindow](#macwindow)
  - [WindowsWindow](#windowswindow)
  - [LinuxWindow](#linuxwindow)
- [平台选项](#平台选项)
  - [v3 MacOptions](#v3-macoptions)
  - [v3 WindowsOptions](#v3-windowsoptions)
  - [v3 LinuxOptions](#v3-linuxoptions)
  - [v3 IOSOptions](#v3-iosoptions)
  - [v3 AndroidOptions](#v3-androidoptions)
- [v2 运行时 API 完整列表](#v2-运行时-api-完整列表)
- [v3 App/Window 方法完整列表](#v3-appwindow-方法完整列表)
- [v3 对话框 API](#v3-对话框-api)
- [v3 事件 API 完整列表](#v3-事件-api-完整列表)
- [v3 菜单 API](#v3-菜单-api)
- [JS 前端运行时 API](#js-前端运行时-api)

---

## App 创建选项

### v2 options.App

```go
type App struct {
    Title             string          // 窗口标题
    Width             int             // 初始宽度
    Height            int             // 初始高度
    DisableResize     bool            // 禁止调整大小
    Fullscreen        bool            // 全屏启动
    Frameless         bool            // 无边框
    MinWidth          int             // 最小宽度
    MinHeight         int             // 最小高度
    MaxWidth          int             // 最大宽度（0 = 无限制）
    MaxHeight         int             // 最大高度（0 = 无限制）
    StartHidden       bool            // 启动时隐藏
    HideWindowOnClose bool            // 关闭时隐藏窗口
    AlwaysOnTop       bool            // 窗口置顶
    BackgroundColour  *RGBA           // 背景色（NewRGBA/NewRGB）
    Assets            fs.FS           // [已废弃] 用 AssetServer.Assets
    AssetsHandler     http.Handler    // [已废弃] 用 AssetServer.Handler
    AssetServer       *assetserver.Options
    Menu              *menu.Menu      // 应用菜单
    Logger            logger.Logger   // 日志记录器
    LogLevel          logger.LogLevel // 日志级别
    LogLevelProduction logger.LogLevel
    OnStartup         func(ctx context.Context)
    OnDomReady        func(ctx context.Context)
    OnShutdown        func(ctx context.Context)
    OnBeforeClose     func(ctx context.Context) (prevent bool)
    Bind              []interface{}   // 绑定 struct
    EnumBind          []interface{}   // 枚举值暴露
    WindowStartState  WindowStartState // Normal/Maximised/Minimised/Fullscreen
    ErrorFormatter    ErrorFormatter  // func(error) any
    CSSDragProperty   string          // 默认 "--wails-draggable"
    CSSDragValue      string          // 默认 "drag"
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

#### v2 平台特定选项

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
    Name        string        // 应用名称
    Description string        // 应用描述
    Icon        []byte        // 应用图标
    Mac         MacOptions
    Windows     WindowsOptions
    Linux       LinuxOptions
    IOS         IOSOptions
    Android     AndroidOptions
    Services    []Service     // Go 服务
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

## 服务与绑定

### v2 Bind 模式

```go
// 方法签名约束
// 1. 方法名首字母大写
// 2. 参数和返回类型需支持 encoding/json
// 3. 支持 context.Context 作为第一个参数（可选）
// 4. 返回 error 作为最后一个返回值会在前端转为 rejected Promise
// 5. 最多 2 个返回值（不含 error）

type GreetService struct{}

// ✅ 有效签名
func (s *GreetService) Greet(name string) string
func (s *GreetService) GreetWithContext(ctx context.Context, name string) string
func (s *GreetService) GetData(id int) (Data, error)
func (s *GreetService) NoReturn()
func (s *GreetService) OnlyError() error

// ❌ 无效签名
func (s *GreetService) greet(name string) string  // 小写
func (s *GreetService) GetData(id int) (Data, string, error) // 超过2个返回值
func (s *GreetService) GetChannel() chan int  // 不支持的类型
```

**v2 前端调用**（自动生成的 wailsjs）：
```js
import {Greet} from "../wailsjs/go/main/GreetService";
Greet("world").then(result => console.log(result)).catch(err => console.error(err));
```

### v3 Service 模式

```go
// 泛型注册
application.NewService(&MyService{})
application.NewServiceWithOptions(&MyService{}, application.ServiceOptions{
    Name:         "CustomName", // 覆盖服务名
    Route:        "/api",       // 如果实现 http.Handler，挂载到此路由
    MarshalError: func(err error) []byte { ... },
})

// 可选接口
type ServiceName interface { ServiceName() string }
type ServiceStartup interface { ServiceStartup(ctx context.Context, options ServiceOptions) error }
type ServiceShutdown interface { ServiceShutdown() error }

// 启动顺序：按 Services 列表顺序 + RegisterService 注册顺序
// 关闭顺序：逆向序（后注册的先关闭）
```

---

## 窗口选项（v3）

### WebviewWindowOptions

```go
type WebviewWindowOptions struct {
    Name             string           // 唯一名称
    Title            string           // 标题
    Width            int              // 初始宽度
    Height           int              // 初始高度
    AlwaysOnTop      bool
    URL              string           // 加载页面路径（相对于 asset server）
    DisableResize    bool
    Frameless        bool
    MinWidth         int
    MinHeight        int
    MaxWidth         int
    MaxHeight        int
    StartState       WindowState      // Normal/Minimised/Maximised/Fullscreen
    BackgroundType   BackgroundType   // BackgroundTypeSolid 等
    BackgroundColour RGBA
    HTML             string           // 直接设置 HTML 内容
    JS               string           // 页面加载时注入的 JS
    CSS              string           // 页面加载时注入的 CSS
    AllowSimpleEventEmit bool         // ⚠️ 安全敏感：允许 postMessage 发送事件
    InitialPosition  WindowStartPosition // WindowCentered / WindowXY
    X, Y             int              // 初始位置
    Screen           *Screen          // 目标屏幕
    Hidden           bool             // 启动时隐藏
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
    UseApplicationMenu bool          // Windows/Linux 继承 app 菜单
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

// MacTitleBar 枚举
MacTitleBarDefault
MacTitleBarHiddenInset
MacTitleBarHiddenInsetUnified

// MacBackdrop 枚举
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

## 平台选项

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

> **注意**：v3 Linux 默认使用 GTK4 + WebKitGTK 6.0。如需 GTK3 + WebKit2GTK 4.1，使用 `-tags gtk3`。

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

## v2 运行时 API 完整列表

所有函数接受 `context.Context`（从 `OnStartup`/`OnDomReady`/`OnShutdown`/`OnBeforeClose` 获取）。

### 窗口操作（`runtime.Window*`）

| 函数 | 说明 |
|------|------|
| `WindowSetTitle(ctx, title)` | 设置窗口标题 |
| `WindowFullscreen(ctx)` | 全屏 |
| `WindowUnfullscreen(ctx)` | 退出全屏 |
| `WindowMaximise(ctx)` | 最大化 |
| `WindowUnmaximise(ctx)` | 取消最大化 |
| `WindowToggleMaximise(ctx)` | 切换最大化 |
| `WindowMinimise(ctx)` | 最小化 |
| `WindowUnminimise(ctx)` | 取消最小化 |
| `WindowShow(ctx)` | 显示窗口 |
| `WindowHide(ctx)` | 隐藏窗口 |
| `WindowSetSize(ctx, width, height)` | 设置尺寸 |
| `WindowGetSize(ctx) (int, int)` | 获取尺寸 |
| `WindowSetMinSize(ctx, width, height)` | 最小尺寸 |
| `WindowSetMaxSize(ctx, width, height)` | 最大尺寸 |
| `WindowSetPosition(ctx, x, y)` | 设置位置 |
| `WindowGetPosition(ctx) (int, int)` | 获取位置 |
| `WindowCenter(ctx)` | 居中 |
| `WindowSetAlwaysOnTop(ctx, bool)` | 置顶 |
| `WindowReload(ctx)` | 重载页面 |
| `WindowReloadApp(ctx)` | 重载应用 |
| `WindowExecJS(ctx, js)` | 执行 JS |
| `WindowSetBackgroundColour(ctx, R, G, B, A)` | 背景色 |
| `WindowIsFullscreen(ctx) bool` | 是否全屏 |
| `WindowIsMaximised(ctx) bool` | 是否最大化 |
| `WindowIsMinimised(ctx) bool` | 是否最小化 |
| `WindowIsNormal(ctx) bool` | 是否普通状态 |
| `WindowPrint(ctx)` | 打印 |
| `WindowSetSystemDefaultTheme(ctx)` | 系统默认主题 |
| `WindowSetLightTheme(ctx)` | 浅色主题 |
| `WindowSetDarkTheme(ctx)` | 深色主题 |

### 对话框（`runtime.*Dialog`）

| 函数 | 说明 |
|------|------|
| `OpenFileDialog(ctx, OpenDialogOptions) (string, error)` | 打开文件 |
| `OpenMultipleFilesDialog(ctx, OpenDialogOptions) ([]string, error)` | 多文件选择 |
| `OpenDirectoryDialog(ctx, OpenDialogOptions) (string, error)` | 选择目录 |
| `SaveFileDialog(ctx, SaveDialogOptions) (string, error)` | 保存文件 |
| `MessageDialog(ctx, MessageDialogOptions) (string, error)` | 消息对话框 |

### 事件

| 函数 | 说明 |
|------|------|
| `EventsOn(ctx, name, callback) func()` | 注册监听，返回取消函数 |
| `EventsOnce(ctx, name, callback) func()` | 单次监听 |
| `EventsOnMultiple(ctx, name, callback, counter) func()` | 有限次监听 |
| `EventsEmit(ctx, name, data...)` | 发射事件 |
| `EventsOff(ctx, name, additionalNames...)` | 取消监听 |
| `EventsOffAll(ctx)` | 取消所有监听 |

### 日志

| 函数 | 说明 |
|------|------|
| `LogPrint(ctx, msg)` / `LogPrintf(ctx, fmt, args...)` | Print 级别 |
| `LogTrace(ctx, msg)` / `LogTracef(ctx, fmt, args...)` | Trace 级别 |
| `LogDebug(ctx, msg)` / `LogDebugf(ctx, fmt, args...)` | Debug 级别 |
| `LogInfo(ctx, msg)` / `LogInfof(ctx, fmt, args...)` | Info 级别 |
| `LogWarning(ctx, msg)` / `LogWarningf(ctx, fmt, args...)` | Warning 级别 |
| `LogError(ctx, msg)` / `LogErrorf(ctx, fmt, args...)` | Error 级别 |
| `LogFatal(ctx, msg)` / `LogFatalf(ctx, fmt, args...)` | Fatal 级别 |
| `LogSetLogLevel(ctx, level)` | 设置日志级别 |

### 其他

| 函数 | 说明 |
|------|------|
| `Quit(ctx)` | 退出应用 |
| `Hide(ctx)` | 隐藏应用 |
| `Show(ctx)` | 显示应用 |
| `Environment(ctx) EnvironmentInfo` | 环境信息（BuildType/Platform/Arch） |
| `BrowserOpenURL(ctx, url)` | 浏览器打开 URL |
| `ClipboardGetText(ctx) (string, error)` | 读取剪贴板 |
| `ClipboardSetText(ctx, text) error` | 写入剪贴板 |
| `ScreenGetAll(ctx) ([]Screen, error)` | 获取所有屏幕 |

---

## v3 App/Window 方法完整列表

通过 `app := application.Get()` 获取全局 App 实例。

### WebviewWindow 方法

```go
// 基本属性
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

// 窗口状态
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

// 可见性
win.Show()
win.Hide()
win.IsVisible() bool
win.IsFocused() bool
win.Focus()
win.Close()
win.Destroy()

// 按钮状态
win.SetMinimiseButtonState(ButtonState)
win.SetMaximiseButtonState(ButtonState)
win.SetCloseButtonState(ButtonState)
win.SetFullscreenButtonState(ButtonState)

// JS 执行
win.ExecJS(js string)

// 缩放
win.ZoomIn()
win.ZoomOut()
win.ZoomReset()
win.GetZoom() float64
win.SetZoom(zoom float64)

// 页面
win.Reload()
win.ForceReload()
win.SetURL(url string)
win.SetHTML(html string)

// 其他
win.Flash(enabled bool)
win.Print() error
win.OpenDevTools()
win.GetScreen() (*Screen, error)
win.StartDrag() error
win.StartResize(border string) error  // "left"/"right"/"top"/"bottom" 等
win.IsEnabled() bool
win.SetEnabled(enabled bool)
win.SetIgnoreMouseEvents(ignore bool)
win.IsIgnoreMouseEvents() bool
win.SetContentProtection(enabled bool)

// 菜单
win.SetMenu(menu *Menu)
win.OpenContextMenu(menu *Menu, data *ContextMenuData)

// 编辑操作 (v3.0+)
win.Cut()
win.Copy()
win.Paste()
win.Undo()
win.Redo()
win.Delete()
win.SelectAll()

// 原生窗口句柄
win.NativeWindow() unsafe.Pointer

// 事件
win.EmitEvent(name string, data ...any) bool
win.RegisterHook(eventType events.WindowEventType, callback func(*WindowEvent))
win.OnWindowEvent(eventType events.WindowEventType, callback func(*WindowEvent))
```

### App 方法

```go
app.Quit()
app.CurrentWindow() *WebviewWindow
app.Window.GetByID(id uint) *WebviewWindow
app.Window.GetByName(name string) *WebviewWindow
app.Context() context.Context  // 应用级 context
```

---

## v3 对话框 API

### MessageDialog

```go
dialog := app.Dialog.Info()    // 或 .Question() / .Warning() / .Error()
dialog.SetTitle("Title")
dialog.SetMessage("Message")
dialog.SetIcon(iconBytes)
dialog.AttachToWindow(win)     // 模态附加
dialog.SetDefaultButton(btn)
dialog.SetCancelButton(btn)
dialog.AddButton("Label") *Button  // 返回 Button，可链式 .OnClick(fn) / .SetAsDefault() / .SetAsCancel()
dialog.Show()
```

### OpenFileDialog

```go
dialog := app.Dialog.OpenFile()
dialog.SetTitle("Title")
dialog.SetMessage("Message")        // macOS 上显示为提示
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

// 三种调用方式
result, err := dialog.PromptForSingleSelection()
results, err := dialog.PromptForMultipleSelection()
```

### SaveFileDialog

```go
dialog := app.Dialog.SaveFile()
dialog.SetTitle("Title")
dialog.SetMessage("Message")        // macOS 上显示为提示
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

## v3 事件 API 完整列表

### 预定义事件（events.Common）

#### 应用级

| 事件 | 说明 |
|------|------|
| `ApplicationStarted` | 应用启动完成 |
| `ThemeChanged` | 系统主题变更 |
| `SystemDidWake` | 系统从睡眠唤醒 |
| `SystemWillSleep` | 系统即将休眠 |

#### 窗口级

| 事件 | 说明 |
|------|------|
| `WindowClosing` | 窗口即将关闭（可取消） |
| `WindowDidMove` | 窗口已移动 |
| `WindowDidResize` | 窗口已调整大小 |
| `WindowDPIChanged` | DPI 变更 |
| `WindowFilesDropped` | 文件拖放 |
| `WindowFocus` | 窗口获得焦点 |
| `WindowLostFocus` | 窗口失去焦点 |
| `WindowFullscreen` | 窗口进入全屏 |
| `WindowHide` | 窗口隐藏 |
| `WindowMaximise` | 窗口最大化 |
| `WindowMinimise` | 窗口最小化 |
| `WindowRestore` | 窗口恢复 |
| `WindowRuntimeReady` | 前端运行时就绪 |
| `WindowShow` | 窗口显示 |
| `WindowUnFullscreen` | 窗口退出全屏 |
| `WindowUnMaximise` | 窗口取消最大化 |
| `WindowUnMinimise` | 窗口取消最小化 |
| `WindowZoom` / `WindowZoomIn` / `WindowZoomOut` / `WindowZoomReset` | 缩放事件 |

#### 平台特定（macOS/Linux/Windows 各自的系统事件）

完整列表见 `v3/pkg/events/known_events.go`（200+ 事件）。

### 自定义事件

```go
// 注册监听
app.Event.On(name, func(e *application.CustomEvent) { ... })  // 返回取消函数
app.Event.OnApplicationEvent(eventType, func(e *application.ApplicationEvent) { ... })

// 发射
app.Event.Emit(name, data)
app.Event.EmitEvent(event) // 直接发射 CustomEvent
```

### Event 模式

v3 事件系统支持两种模式：
- **严格模式**（默认）：事件必须预先注册，否则发射时报错
- **宽松模式**：允许发射未注册的事件

---

## v3 菜单 API

### Menu 方法

```go
menu := app.NewMenu()
menu.Add(label string) *MenuItem
menu.AddSeparator()
menu.AddCheckbox(label string, enabled bool) *MenuItem
menu.AddRadio(label string, enabled bool) *MenuItem
menu.AddSubmenu(label string) *Menu
menu.AddRole(role Role) *Menu  // AppMenu/FileMenu/EditMenu/WindowMenu/ServicesMenu/HelpMenu
menu.Update()                   // 应用菜单到原生层
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

### MenuItem 方法

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
ctxMenu.Update()  // 注册到 app
ctxMenu.Destroy()
```

---

## JS 前端运行时 API

### v2 JS 运行时

```js
// 调用 Go 方法（自动生成）
import {Greet} from "../wailsjs/go/main/GreetService"
const result = await Greet("world")

// 事件
window.wails.EventsOn("event-name", (data) => { ... })
window.wails.EventsOnce("event-name", (data) => { ... })
window.wails.EventsOnMultiple("event-name", callback, maxCalls)
window.wails.EventsEmit("event-name", data)
window.wails.EventsOff("event-name")

// 日志
window.wails.LogLevel("debug")
window.wails.LogPrint("msg")
window.wails.LogDebug("msg")

// 系统
window.wails.Quit()
window.wails.BrowserOpenURL("https://...")
```

### v3 JS 运行时

v3 通过 HTTP transport 通信。JS 端 API 生成在 `frontend/wailsjs/` 目录下。运行时脚本自动通过 `/wails/runtime.js` 提供，由 `BundledAssetFileServer` 自动注入。

```js
// 调用 Go 方法（使用 fetch 通过 HTTP transport）
// 自动生成的绑定代码处理序列化和传输

// 事件（通过运行时脚本）
window._wails.invoke("wails:event:emit:eventName")
window._wails.on("eventName", callback)
```
