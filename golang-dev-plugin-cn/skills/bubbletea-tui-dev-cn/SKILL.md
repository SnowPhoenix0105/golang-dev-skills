---
name: bubbletea-tui-dev-cn
description: |
  Bubble Tea 终端 UI (TUI) 应用开发。基于 Elm Architecture 的 Go TUI 框架。
  触发场景：提到 bubbletea、bubble tea、TUI、终端界面、命令行界面、tea.Model、tea.Cmd、
  tea.NewProgram、tea.KeyPressMsg、Bubbles 组件（spinner/textinput/textarea/table/list/viewport）、
  Lip Gloss 样式、Elm Architecture、终端渲染等。
  即使用户没有明确说"bubbletea"，只要在做 Go 终端界面开发就应该考虑此技能。
---

# Bubble Tea TUI 应用开发

Bubble Tea（`charm.land/bubbletea/v2`）是基于 [The Elm Architecture][elm] 的 Go 语言 TUI 框架，提供声明式 View、消息驱动的单向数据流和高速 Cell-based 渲染器。

[elm]: https://guide.elm-lang.org/architecture/

## Bubble Tea 生态系统

| 仓库 | 用途 | go module |
| :--- | :--- | :--- |
| **bubbletea** | 核心 TUI 框架：Model/Update/View 生命周期、消息系统、渲染器、输入处理 | `charm.land/bubbletea/v2` |
| **bubbles** | 官方组件库：spinner、textinput、textarea、table、list、viewport、paginator、progress、filepicker、help、key、timer、stopwatch 等 | `charm.land/bubbles/v2` |
| **lipgloss** | 终端样式与布局：Style 链式 API、Color、Border、Padding/Margin、Align、Join、Canvas | `charm.land/lipgloss/v2` |
| **ultraviolet** | 底层终端抽象：TerminalReader、ScreenBuffer、Key/Mouse 事件解析、Cell-buffer 渲染 | `github.com/charmbracelet/ultraviolet` |
| **catwalk** | 高级布局引擎：Flexbox 布局、组件树、声明式 UI，构建在 ultraviolet 之上 | `charm.land/catwalk` |
| **harmonica** | 弹簧动画库：基于物理的平滑动画 | `github.com/charmbracelet/harmonica` |
| **huh** | 交互式表单和提示工具包 | `github.com/charmbracelet/huh` |
| **glamour** | Markdown 终端渲染 | `github.com/charmbracelet/glamour` |
| **wish** | SSH 服务端中间件（让 Bubble Tea 应用通过 SSH 访问） | `github.com/charmbracelet/wish` |
| **bubblezone** | 鼠标事件区域追踪辅助库 | `github.com/lrstanley/bubblezone` |

> **v1 → v2 迁移**：v2 最大的变化是 View 从返回 `string` 改为返回 `tea.View` 结构体（声明式），按键消息从 `tea.KeyMsg` 改为 `tea.KeyPressMsg`/`tea.KeyReleaseMsg`。完整迁移指南见 [UPGRADE_GUIDE_V2.md](https://github.com/charmbracelet/bubbletea/blob/main/UPGRADE_GUIDE_V2.md)。

如需浏览本地源码，用 `go list -m -json charm.land/bubbletea/v2` 查找模块路径。

## 核心架构：Elm Architecture

Bubble Tea 强制执行单向数据流：

```
用户输入 → Update(msg) → 新Model → View() → 渲染
                ↑                        ↓
              Cmd (异步IO) → Msg ────────┘
```

三个核心方法，全在一个 Model 上：

| 方法 | 签名 | 职责 |
| :--- | :--- | :--- |
| **Init** | `Init() Cmd` | 返回初始命令（如启动定时器、发起 HTTP 请求）。无初始 I/O 则返回 `nil` |
| **Update** | `Update(Msg) (Model, Cmd)` | 接收消息，返回更新后的 Model 和可选的新 Cmd。**必须快速返回（<1ms），耗时操作用 Cmd** |
| **View** | `View() View` | 根据 Model 当前状态渲染 UI。v2 返回 `tea.View` 结构体而非字符串 |

**Model** 是任意实现了上述三个接口的类型，通常是一个 struct。Msg 可以是任意类型（`tea.Msg = uv.Event`）。

## 核心原则

1. **消息驱动，非互斥锁** — 所有状态变更通过 `tea.Msg` 传递，避免在 Update/View 中使用 `sync.Mutex`
2. **Update 必须快** — Update 函数应在 <1ms 内返回，耗时 I/O 放入 `tea.Cmd`（异步 goroutine 执行）
3. **声明式 View** — View 返回的 `tea.View` 结构体声明 AltScreen、MouseMode、Cursor、WindowTitle 等终端特性，不再通过程序选项命令式控制
4. **Cmd 组合** — 用 `tea.Batch` 并行、`tea.Sequence` 串行组合多个 Cmd
5. **嵌套 Model** — 复杂应用将子组件 Model 嵌入父 Model，在 Update 中转发消息给子组件

## 最小应用

```go
package main

import (
    "fmt"
    "os"

    tea "charm.land/bubbletea/v2"
)

type model int

func (m model) Init() tea.Cmd { return nil }

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "ctrl+c", "q":
            return m, tea.Quit
        }
    }
    return m, nil
}

func (m model) View() tea.View {
    return tea.NewView(fmt.Sprintf("Count: %d\n\nPress q to quit.\n", m))
}

func main() {
    p := tea.NewProgram(model(0))
    if _, err := p.Run(); err != nil {
        fmt.Printf("Error: %v\n", err)
        os.Exit(1)
    }
}
```

## 核心类型

### Model 接口

```go
type Model interface {
    Init() Cmd
    Update(Msg) (Model, Cmd)
    View() View
}
```

### Msg

`tea.Msg` 是 `uv.Event` 的别名，可以是任意类型。框架自动发送以下消息类型：

| 消息类型 | 触发时机 |
| :--- | :--- |
| `tea.KeyPressMsg` | 按键按下 |
| `tea.KeyReleaseMsg` | 按键释放（需启用键盘增强） |
| `tea.MouseClickMsg` / `tea.MouseReleaseMsg` / `tea.MouseWheelMsg` / `tea.MouseMotionMsg` | 鼠标事件（需在 View 中启用 MouseMode） |
| `tea.WindowSizeMsg` | 终端窗口大小变化 |
| `tea.FocusMsg` / `tea.BlurMsg` | 终端获取/失去焦点（需在 View 中设置 `ReportFocus = true`） |
| `tea.QuitMsg` | 调用 `tea.Quit()` |
| `tea.SuspendMsg` | 调用 `tea.Suspend()` |
| `tea.ResumeMsg` | 从挂起恢复 |
| `tea.ColorProfileMsg` | 终端颜色 profile 信息 |
| `tea.EnvMsg` | 环境变量 |
| `tea.KeyboardEnhancementsMsg` | 键盘增强能力响应 |

### Cmd

```go
type Cmd func() Msg
```

`Cmd` 是返回消息的函数，在 goroutine 中异步执行。完整命令 API 见 `references/commands.md`。

### View

v2 中 View 返回 `tea.View` 结构体，声明式控制终端特性：

```go
func (m model) View() tea.View {
    var v tea.View
    v.SetContent("Hello, World!")  // 或 tea.NewView("Hello!")
    v.AltScreen = true              // 全屏模式
    v.MouseMode = tea.MouseModeCellMotion  // 鼠标支持
    v.ReportFocus = true            // 焦点事件
    v.WindowTitle = "My App"        // 窗口标题
    v.Cursor = tea.NewCursor(5, 2)  // 光标位置
    v.BackgroundColor = color.RGBA{...}  // 终端背景色
    return v
}
```

`tea.View` 完整字段列表见 `references/core-concepts.md`。

## Program 与选项

```go
p := tea.NewProgram(model,
    tea.WithContext(ctx),             // 外部 context 控制
    tea.WithInput(inputReader),       // 自定义输入（nil = 禁用）
    tea.WithOutput(outputWriter),     // 自定义输出
    tea.WithEnvironment(env),         // 自定义环境变量
    tea.WithFPS(60),                  // 最大帧率（默认60，最大120）
    tea.WithColorProfile(profile),    // 强制颜色 profile
    tea.WithWindowSize(120, 40),      // 初始窗口大小（测试用）
    tea.WithFilter(filterFn),         // 事件过滤器
    tea.WithoutSignalHandler(),       // 禁用信号处理
    tea.WithoutCatchPanics(),         // 禁用 panic 捕获
    tea.WithoutRenderer(),            // 禁用渲染器（纯 CLI 模式）
)
```

Program 方法：
- `p.Run() (Model, error)` — 启动程序，阻塞直到退出
- `p.Send(msg)` — 从外部注入消息
- `p.Quit()` — 从外部退出程序
- `p.Kill()` — 立即终止
- `p.Println(args...)` / `p.Printf(format, args...)` — 在 TUI 上方打印日志
- `p.ReleaseTerminal()` / `p.RestoreTerminal()` — 临时释放/恢复终端（如执行外部命令时）

## 按键处理

完整键码列表见 `references/core-concepts.md`。

```go
// 方式一：switch msg.String() — 简洁
case tea.KeyPressMsg:
    switch msg.String() {
    case "ctrl+c", "q":
        return m, tea.Quit
    case "enter":
        // 确认
    case "up", "down":
        // 导航
    }

// 方式二：switch key.Code — 更类型安全
case tea.KeyPressMsg:
    switch msg.Code {
    case tea.KeyEnter:
        // 确认
    case tea.KeyRunes:
        switch msg.Text {
        case "y":
            // 按了 y
        }
    }

// 方式三：检查修饰键
case tea.KeyPressMsg:
    if msg.Mod&tea.ModCtrl != 0 && msg.Code == 'c' {
        return m, tea.Quit
    }
```

`tea.Key` 结构体字段：`Text string`（可打印字符）、`Code rune`（键码）、`Mod KeyMod`（修饰键）、`ShiftedCode rune`、`BaseCode rune`、`IsRepeat bool`。

## 鼠标处理

```go
// 在 View 中启用鼠标
func (m model) View() tea.View {
    v := tea.NewView("Click me!")
    v.MouseMode = tea.MouseModeCellMotion  // 或 MouseModeAllMotion
    return v
}

// 在 Update 中处理鼠标事件
case tea.MouseClickMsg:
    x, y := msg.X, msg.Y
    // 处理点击

case tea.MouseWheelMsg:
    delta := 1  // 向上滚动
    if msg.Button == tea.MouseWheelDown {
        delta = -1
    }
```

鼠标模式：`MouseModeNone`（禁用）、`MouseModeCellMotion`（点击+拖拽+滚轮，推荐）、`MouseModeAllMotion`（所有移动事件）。

## 调试与日志

```go
// 写入文件（因为 stdout 被 TUI 占用）
if f, err := tea.LogToFile("debug.log", "debug"); err == nil {
    defer f.Close()
}

// 或使用 TEA_TRACE 环境变量
// TEA_TRACE=trace.log 可获取 Bubble Tea 内部事件追踪
```

Debug 技巧：
- 使用 `tail -f debug.log` 实时查看日志
- 用 Delve 调试需要 headless 模式：`dlv debug --headless --api-version=2 --listen=127.0.0.1:43000 .` 然后从另一个终端 `dlv connect`
- `TEA_DEBUG=1` 环境变量可在 panic 时输出详细日志文件

## Bubbles 组件库

| 组件 | 用途 | go package |
| :--- | :--- | :--- |
| `spinner` | 加载/等待指示器 | `charm.land/bubbles/v2/spinner` |
| `textinput` | 单行文本输入 | `charm.land/bubbles/v2/textinput` |
| `textarea` | 多行文本输入 | `charm.land/bubbles/v2/textarea` |
| `table` | 表格数据展示 | `charm.land/bubbles/v2/table` |
| `list` | 可选择列表 | `charm.land/bubbles/v2/list` |
| `viewport` | 可滚动视口 | `charm.land/bubbles/v2/viewport` |
| `paginator` | 分页导航 | `charm.land/bubbles/v2/paginator` |
| `progress` | 进度条 | `charm.land/bubbles/v2/progress` |
| `filepicker` | 文件选择器 | `charm.land/bubbles/v2/filepicker` |
| `help` | 帮助/快捷键栏 | `charm.land/bubbles/v2/help` |
| `key` | 按键绑定定义 | `charm.land/bubbles/v2/key` |
| `timer` / `stopwatch` | 计时器/秒表 | `charm.land/bubbles/v2/timer` |

Bubbles 组件遵循与 `tea.Model` 相同的 Update/View 模式，可直接嵌套到主 Model 中（注意其 `View()` 返回 `string`，需在父 Model 的 View 中用 `tea.NewView()` 包裹）。完整用法见 `references/components.md`。

## Lip Gloss 样式

```go
import "charm.land/lipgloss/v2"

var style = lipgloss.NewStyle().
    Bold(true).
    Foreground(lipgloss.Color("#FAFAFA")).
    Background(lipgloss.Color("#7D56F4")).
    Padding(1, 2).           // 上下1，左右2
    Margin(0, 1).            // 上下0，左右1
    Border(lipgloss.RoundedBorder()).
    BorderForeground(lipgloss.Color("#7D56F4")).
    Width(40).
    Align(lipgloss.Center)

// 使用
fmt.Println(style.Render("Hello, kitty"))

// 组合布局
left := style1.Render("left")
right := style2.Render("right")
fmt.Println(lipgloss.JoinHorizontal(lipgloss.Top, left, right))

// 自适应颜色
lipgloss.AdaptiveColor{Light: "#333", Dark: "#FFF"}
```

完整样式 API 和布局能力见 `references/styling-layout.md`。

## 嵌套 Model 模式

复杂 TUI 应用的标准架构是将子组件嵌入父 Model：

```go
type mainModel struct {
    state     viewState               // 当前视图
    list      list.Model              // Bubbles 组件
    textinput textinput.Model
    // 自定义子 model
    dialog    *DialogModel
}

func (m mainModel) Init() tea.Cmd {
    // Bubbles 组件无 Init 方法；这里可以启动自己的初始 Cmd
    return nil
}

func (m mainModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    // 1. 处理全局消息
    switch msg := msg.(type) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "ctrl+c":
            return m, tea.Quit
        case "tab":
            m.state = nextState(m.state)  // 切换焦点
        }
    case tea.WindowSizeMsg:
        m.width, m.height = msg.Width, msg.Height
    }

    // 2. 根据状态转发给子组件
    var cmd tea.Cmd
    switch m.state {
    case listView:
        m.list, cmd = m.list.Update(msg)
    case inputView:
        m.textinput, cmd = m.textinput.Update(msg)
    }
    return m, cmd
}

func (m mainModel) View() tea.View {
    // 使用 lipgloss 组合各子组件的 View
    leftPanel := m.list.View()
    rightPanel := m.textinput.View()
    content := lipgloss.JoinHorizontal(lipgloss.Top, leftPanel, rightPanel)
    return tea.NewView(content)
}
```

详细架构模式（Model Stack、焦点管理、响应式布局等）见 `references/core-concepts.md`。

## 测试

```go
// teatest — Bubble Tea 官方测试库
// 可在 go.mod 中添加：github.com/charmbracelet/x/exp/teatest

// 测试 Program 运行
func TestMyApp(t *testing.T) {
    tm := teatest.NewTestModel(t, initialModel())

    // 发送按键
    tm.Send(tea.KeyPressMsg{Code: 'q'})

    // 等待退出
    tm.WaitFinished(t)

    // 验证最终状态
    finalModel := tm.FinalModel(t)
}
```

完整测试方案（Model 单元测试、集成测试、模拟终端输入）见 `references/testing.md`。

## 常见错误速查

| 问题 | 原因 | 解决 |
| :--- | :--- | :--- |
| 程序启动后无输出 | 未调用 `p.Run()` 或 Model 为 nil | 确保 `tea.NewProgram(model).Run()` |
| `View()` 返回旧内容 | Update 返回了错误的 Model | 确保 Update 返回更新后的 Model |
| 按键无响应 | 未正确匹配消息类型 | 使用 `case tea.KeyPressMsg`（v2），不是 `tea.KeyMsg` |
| 鼠标不工作 | View 中未启用 MouseMode | 在 View 中设置 `v.MouseMode = tea.MouseModeCellMotion` |
| 渲染闪烁 | View 中频繁构建大字符串 | 使用 `strings.Builder`，预分配样式 |
| 窗口大小变化后布局错乱 | 未处理 `tea.WindowSizeMsg` | 监听并更新 width/height |
| `panic: runtime error` 后终端乱码 | panic 被捕获但终端未恢复 | 设置 `TEA_DEBUG=1` 获取详细日志 |
| Cmd 不执行 | Update 未返回 Cmd 或 Cmd 为 nil 被忽略 | 使用 `tea.Batch` 组合多个 Cmd |
| AltScreen 退出后内容消失 | 正常行为 | 如需持久内容，使用 `p.Println()` 在 TUI 上方输出 |
| 无法在后台 goroutine 中更新 UI | 跨 goroutine 直接操作 Model | 通过 `p.Send(msg)` 发送消息回到 Update |

## 无从下手？先读这些

当 references 覆盖不到、需要探索 Bubble Tea 源码时，使用以下方法自行查找答案。

### 包路径 → 文件系统映射

```bash
# 方式一：go list（精准，推荐）
BT_DIR=$(go list -m -json charm.land/bubbletea/v2 | grep '"Dir"' | cut -d'"' -f4)

# 方式二：GOMODCACHE
BT_DIR=$(echo $(go env GOMODCACHE)/charm.land/bubbletea/v2@*)
```

获取 `$BT_DIR` 后，包路径 `charm.land/bubbletea/v2` 即对应 `$BT_DIR/`。

对 Bubbles：`BB_DIR=$(go list -m -json charm.land/bubbles/v2 | grep '"Dir"' | cut -d'"' -f4)`

对 Lip Gloss：`LG_DIR=$(go list -m -json charm.land/lipgloss/v2 | grep '"Dir"' | cut -d'"' -f4)`

### 查找 API

```bash
go doc charm.land/bubbletea/v2
go doc charm.land/bubbletea/v2.KeyPressMsg
rg "func.*" "$BT_DIR/"
```

## 参考资料（按场景查阅）

| 场景 | 查阅文件 | 重点章节 |
| :--- | :--- | :--- |
| 不理解 Model/Update/View 的关系 | `references/core-concepts.md` | Elm 架构、生命周期 |
| 不知道怎么执行异步操作 | `references/commands.md` | Cmd/Batch/Sequence/Tick/Every |
| 需要选择组件/了解组件用法 | `references/components.md` | 对应组件章节 |
| 样式不对/布局不好看 | `references/styling-layout.md` | Style 链式 API、Join、Canvas |
| 布局错乱/性能差/程序崩溃 | `references/troubleshooting.md` | 先查诊断表，再查对应章节 |
| 不知道大型项目怎么组织代码 | `references/best-practices.md` | 样式管理、KeyMap 分层、对话框栈、布局重算 |
| 怎么写测试 | `references/testing.md` | 单元测试、teatest 集成测试 |
| 需要在嵌套 Model 之间导航 | `references/core-concepts.md` | 嵌套 Model 模式、焦点管理 |
| 从 v1 升级到 v2 | 官方 [UPGRADE_GUIDE_V2.md](https://github.com/charmbracelet/bubbletea/blob/main/UPGRADE_GUIDE_V2.md) | — |

上述 references 都覆盖不到时，按照上一节"无从下手？先读这些"的方法探索源码。如果探索源码后仍然无法解决，**向用户报告具体情况并寻求帮助，不得静默猜测。**
