# 核心概念与架构

## 目录

- [Elm Architecture 与 Bubble Tea](#elm-architecture-与-bubble-tea)
- [Model 接口详解](#model-接口详解)
- [View 结构体字段完整列表](#view-结构体字段完整列表)
- [消息系统](#消息系统)
- [Program 生命周期](#program-生命周期)
- [嵌套 Model 模式](#嵌套-model-模式)
- [焦点管理](#焦点管理)
- [响应式布局](#响应式布局)

## Elm Architecture 与 Bubble Tea

Bubble Tea 严格遵循 Elm Architecture 的单向数据流：

```
                    ┌──────────┐
         ┌─────────→│   Init   │
         │          └────┬─────┘
         │               │ Cmd (可选)
         │               ↓
         │          ┌──────────┐        ┌──────────┐
  View ←─┤          │  Update  │←───────│   Cmd    │
  (渲染)  │          └──────────┘  Msg   │ (异步IO) │
         │               │               └──────────┘
         │               │ Cmd (可选)
         │               ↓
         │          ┌──────────┐
         └──────────│   View   │
                    └──────────┘
```

**关键原则**：
1. **单向数据流** — 数据只沿一个方向流动：Init → Update → View → Render
2. **消息驱动** — 所有状态变更都通过 Msg 触发 Update
3. **Update 必须快** — Update 函数应在 <1ms 内返回。耗时 I/O 包装为 Cmd
4. **View 是纯函数** — 给定相同 Model，View 应输出相同结果

## Model 接口详解

```go
type Model interface {
    Init() Cmd
    Update(Msg) (Model, Cmd)
    View() View
}
```

### Init() Cmd

- 在 `Program.Run()` 启动时调用一次
- 返回初始异步操作，如：加载数据、启动定时器、查询终端能力
- 无初始 I/O 时返回 `nil`
- 常用模式：

```go
func (m model) Init() tea.Cmd {
    return tea.Batch(
        loadData,           // 加载初始数据
        tickEvery(),        // 启动周期性 tick
        queryTerminalCap,   // 查询终端能力
    )
}
```

### Update(Msg) (Model, Cmd)

- 每次收到消息时调用
- 返回更新后的 Model 和可选的新 Cmd
- **必须快速返回**：耗时操作（网络请求、文件读写、长时间计算）包装为 Cmd
- 典型模式：先处理全局消息（quit、窗口大小），再转发给当前焦点子组件

```go
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    // 第一步：处理全局消息
    switch msg := msg.(type) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "ctrl+c":
            return m, tea.Quit
        }
    case tea.WindowSizeMsg:
        m.width = msg.Width
        m.height = msg.Height
    }

    // 第二步：根据当前状态/焦点转发
    var cmd tea.Cmd
    switch m.focus {
    case focusList:
        m.list, cmd = m.list.Update(msg)
    case focusInput:
        m.textinput, cmd = m.textinput.Update(msg)
    }
    return m, cmd
}
```

### View() View

- 每次 Update 后自动调用
- 返回 `tea.View` 结构体，描述 UI 内容和终端特性
- v2 声明式：不再通过程序选项命令式控制终端状态
- View 应渲染**整个**屏幕，Bubble Tea 负责 diff 和重绘优化

## View 结构体字段完整列表

```go
type View struct {
    Content string              // UI 文本内容（ANSI 转义序列编码样式）
    AltScreen bool              // 交替屏幕缓冲（全屏模式）
    MouseMode MouseMode         // 鼠标模式：MouseModeNone / MouseModeCellMotion / MouseModeAllMotion
    ReportFocus bool            // 启用焦点报告（收到 FocusMsg/BlurMsg）
    DisableBracketedPasteMode bool  // 禁用括号粘贴模式
    WindowTitle string          // 终端窗口标题
    Cursor *Cursor              // 光标状态（位置、样式、颜色、闪烁）
    BackgroundColor color.Color // 终端背景色
    ForegroundColor color.Color // 终端前景色
    ProgressBar *ProgressBar    // 原生进度条（部分终端支持）
    KeyboardEnhancements KeyboardEnhancements  // 键盘增强功能请求
    OnMouse func(msg MouseMsg) Cmd  // View 级别的鼠标处理（依赖上次渲染内容）
}
```

### MouseMode

```go
const (
    MouseModeNone       MouseMode = iota  // 禁用鼠标
    MouseModeCellMotion                    // 点击+释放+滚轮+拖拽（推荐，兼容性好）
    MouseModeAllMotion                     // 所有事件，包括无按键的移动
)
```

### Cursor

```go
cursor := &tea.Cursor{
    Position: tea.Position{X: 5, Y: 2},  // 相对于帧左上角
    Color:    color.RGBA{255, 0, 0, 255},
    Shape:    tea.CursorBlock,           // CursorBlock / CursorUnderline / CursorBar
    Blink:    true,
}
```

### ProgressBar

```go
// 需要在 View 中设置
v.ProgressBar = tea.NewProgressBar(tea.ProgressBarDefault, 50)       // 50%
v.ProgressBar = tea.NewProgressBar(tea.ProgressBarIndeterminate, 0)   // 不确定
v.ProgressBar = tea.NewProgressBar(tea.ProgressBarError, 0)           // 错误状态
// 状态值: ProgressBarNone / ProgressBarDefault / ProgressBarError
//         ProgressBarIndeterminate / ProgressBarWarning
```

### KeyboardEnhancements

```go
v.KeyboardEnhancements = tea.KeyboardEnhancements{
    ReportEventTypes:      true,  // 接收 KeyReleaseMsg 和 Key.IsRepeat
    ReportAlternateKeys:   false, // 报告备用键码
    ReportAllKeysAsEscapeCodes: false,  // 所有按键以转义序列形式报告
    ReportAssociatedText:  false, // 报告关联文本
}
```

终端支持的能力通过 `tea.KeyboardEnhancementsMsg` 返回。

## 消息系统

### 框架自动发送的消息

| 消息类型 | 触发条件 | 关键字段 |
| :--- | :--- | :--- |
| `tea.KeyPressMsg` | 按键按下 | 见 [Key 结构体](#key-结构体) |
| `tea.KeyReleaseMsg` | 按键释放 | 需 `ReportEventTypes = true`，同上 |
| `tea.MouseClickMsg` | 鼠标点击 | `X, Y int; Button MouseButton; Mod KeyMod` |
| `tea.MouseReleaseMsg` | 鼠标释放 | 同上 |
| `tea.MouseWheelMsg` | 鼠标滚轮 | 同上，`Button` 为 `MouseWheelUp/Down/Left/Right` |
| `tea.MouseMotionMsg` | 鼠标移动 | 同上 |
| `tea.WindowSizeMsg` | 终端窗口大小变化 | `Width int; Height int` |
| `tea.FocusMsg` | 终端获取焦点 | 需 `ReportFocus = true` |
| `tea.BlurMsg` | 终端失去焦点 | 需 `ReportFocus = true` |
| `tea.ColorProfileMsg` | 颜色 profile 信息 | `Profile colorprofile.Profile` |
| `tea.EnvMsg` | 环境变量 | `[]string` 类型 |
| `tea.KeyboardEnhancementsMsg` | 键盘增强响应 | `Flags int` |
| `tea.QuitMsg` | `tea.Quit()` 被调用 | — |
| `tea.SuspendMsg` | `tea.Suspend()` 被调用 | — |
| `tea.ResumeMsg` | 从挂起恢复 | — |

### Key 结构体

```go
type Key struct {
    Text        string  // 可打印字符（如 "a", "A", "1", "!"），特殊键为空
    Mod         KeyMod  // 修饰键：ModCtrl, ModAlt, ModShift, ModMeta, ModSuper, ModHyper
    Code        rune    // 键码：特殊键用常量（KeyEnter, KeyTab），可打印键用 rune
    ShiftedCode rune    // 带 Shift 的实际字符（仅 Kitty 协议/Windows）
    BaseCode    rune    // PC-101 布局基准键码（仅 Kitty 协议/Windows）
    IsRepeat    bool    // 是否重复按键（仅 Kitty 协议/Windows）
}
```

### 常用键码常量

```go
// 特殊键
tea.KeyUp, tea.KeyDown, tea.KeyRight, tea.KeyLeft
tea.KeyEnter, tea.KeyReturn, tea.KeyTab
tea.KeyEscape, tea.KeyEsc
tea.KeyBackspace, tea.KeySpace
tea.KeyDelete, tea.KeyInsert
tea.KeyHome, tea.KeyEnd, tea.KeyPgUp, tea.KeyPgDown

// 功能键
tea.KeyF1 ... tea.KeyF63

// 修饰键
tea.KeyLeftCtrl, tea.KeyRightCtrl, tea.KeyLeftAlt, tea.KeyRightAlt
tea.KeyLeftShift, tea.KeyRightShift, tea.KeyLeftSuper, tea.KeyRightSuper

// 媒体键
tea.KeyMediaPlay, tea.KeyMediaPause, tea.KeyMediaNext, tea.KeyMediaPrev
tea.KeyLowerVol, tea.KeyRaiseVol, tea.KeyMute

// 修饰键常量
tea.ModCtrl, tea.ModAlt, tea.ModShift, tea.ModMeta, tea.ModSuper, tea.ModHyper
```

### 匹配按键的三种方式

```go
switch msg := msg.(type) {
case tea.KeyPressMsg:
    // 方式一：String() — 最常用
    switch msg.String() {
    case "ctrl+c", "q":
        return m, tea.Quit
    case "enter":
        // ...
    case "space":
        // 注意：v2 中空格键的 String() 返回 "space"，不是 " "
    }

    // 方式二：Code — 类型安全
    switch msg.Code {
    case tea.KeyEnter:
        // ...
    default:
        switch msg.Text {
        case "y", "Y":
            // ...
        }
    }

    // 方式三：修饰键检查
    if msg.Mod&tea.ModCtrl != 0 {
        switch msg.Code {
        case 'c':
            return m, tea.Quit
        case 's':
            // Ctrl+S
        }
    }
}
```

### 自定义消息

任何类型都可以作为 Msg：

```go
type tickMsg time.Time

type statusMsg int

type errMsg struct{ err error }

func (e errMsg) Error() string { return e.err.Error() }

type DataLoadedMsg struct {
    Items []Item
}
```

## Program 生命周期

### 创建与运行

```go
p := tea.NewProgram(initialModel, options...)
finalModel, err := p.Run()
```

### 生命周期事件顺序

1. `NewProgram` — 创建 Program 实例
2. `p.Run()` — 初始化终端（raw mode、隐藏光标等）
3. 发送初始消息：`WindowSizeMsg` → `ColorProfileMsg` → `EnvMsg`
4. 调用 `model.Init()` 获取初始 Cmd
5. 渲染初始 View
6. **事件循环**：等待 Msg → Update → View → 渲染
7. 收到 `QuitMsg` 或 `InterruptMsg` 时退出循环
8. 渲染最终帧，恢复终端状态
9. 返回 finalModel 和 error

### 退出方式

```go
// 方式一：从 Update 中返回 tea.Quit
return m, tea.Quit

// 方式二：从外部调用
p.Quit()

// 方式三：系统信号
// SIGINT  → InterruptMsg → ErrInterrupted
// SIGTERM → QuitMsg → 正常退出

// 方式四：context 取消
ctx, cancel := context.WithCancel(context.Background())
p := tea.NewProgram(model, tea.WithContext(ctx))
// 其他 goroutine 中: cancel()
// p.Run() 返回 ErrProgramKilled
```

### External Message Injection

```go
p := tea.NewProgram(model)

go func() {
    // 后台操作完成后注入消息
    result := doSomething()
    p.Send(resultMsg{data: result})
}()

p.Run()
```

### ReleaseTerminal / RestoreTerminal

在需要临时恢复终端（如执行外部命令）时使用：

```go
// 释放终端
if err := p.ReleaseTerminal(); err != nil {
    return err
}

// 执行外部命令（使用标准 stdin/stdout）
execCmd := exec.Command("vim", "file.txt")
execCmd.Stdin = os.Stdin
execCmd.Stdout = os.Stdout
execCmd.Run()

// 恢复终端
if err := p.RestoreTerminal(); err != nil {
    return err
}
```

## 嵌套 Model 模式

### 模式一：直接嵌套（组件在当前布局中）

最常用的模式，适用于子组件始终是当前视图的一部分：

```go
type mainModel struct {
    list      list.Model       // Bubbles list 组件
    textinput textinput.Model  // Bubbles textinput 组件
    width     int
    height    int
}

func (m mainModel) Init() tea.Cmd {
    // Bubbles 组件无 Init 方法，直接返回 nil 或自己的初始 Cmd
    return nil
}

func (m mainModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmds []tea.Cmd

    switch msg := msg.(type) {
    case tea.WindowSizeMsg:
        m.width = msg.Width
        m.height = msg.Height
        // 调整子组件大小
        m.list.SetSize(msg.Width/2, msg.Height)
        m.textinput.SetWidth(msg.Width / 2)
    }

    // 根据焦点转发
    var cmd tea.Cmd
    switch m.focus {
    case focusList:
        m.list, cmd = m.list.Update(msg)
    case focusInput:
        m.textinput, cmd = m.textinput.Update(msg)
    }
    cmds = append(cmds, cmd)

    return m, tea.Batch(cmds...)
}

func (m mainModel) View() tea.View {
    left := m.list.View()
    right := m.textinput.View()
    content := lipgloss.JoinHorizontal(lipgloss.Top, left, right)
    return tea.NewView(content)
}
```

### 模式二：页面切换（子 Model 独占全屏）

适用于独立的"页面"之间切换：

```go
type pageState int

const (
    pageList pageState = iota
    pageDetail
    pageSettings
)

type mainModel struct {
    state    pageState
    list     list.Model
    detail   DetailModel
    settings SettingsModel
    width    int
    height   int
}

func (m mainModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    // 全局导航
    switch msg := msg.(type) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "ctrl+c":
            return m, tea.Quit
        case "esc":
            m.state = pageList  // 统一返回列表
        }
    }

    // 根据当前页面转发
    var cmd tea.Cmd
    switch m.state {
    case pageList:
        m.list, cmd = m.list.Update(msg)
    case pageDetail:
        m.detail, cmd = m.detail.Update(msg)
    case pageSettings:
        m.settings, cmd = m.settings.Update(msg)
    }
    return m, cmd
}

func (m mainModel) View() tea.View {
    switch m.state {
    case pageList:
        return tea.NewView(m.list.View())
    case pageDetail:
        return tea.NewView(m.detail.View())
    case pageSettings:
        return tea.NewView(m.settings.View())
    }
    return tea.NewView("")
}
```

### 模式三：对话框/覆盖层

```go
type mainModel struct {
    base    BaseModel
    dialog  *DialogModel  // nil 表示无对话框
}

func (m mainModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    // 对话框优先处理
    if m.dialog != nil {
        var cmd tea.Cmd
        m.dialog, cmd = m.dialog.Update(msg)
        // 检查对话框是否请求关闭
        if m.dialog.Done {
            m.dialog = nil
        }
        return m, cmd
    }

    // 正常流程
    var cmd tea.Cmd
    m.base, cmd = m.base.Update(msg)
    return m, cmd
}
```

## 焦点管理

### Focus 状态机

```go
type FocusState int

const (
    FocusNone FocusState = iota
    FocusList
    FocusInput
    FocusDetail
)

// Tab 键轮转焦点
case tea.KeyPressMsg:
    switch msg.String() {
    case "tab":
        m.focus = (m.focus + 1) % numFocusStates
    case "shift+tab":
        m.focus = (m.focus - 1 + numFocusStates) % numFocusStates
    }
```

### 焦点恢复模式（对话框场景）

```go
// 打开对话框前保存焦点
func (m *model) openDialog() {
    m.previousFocus = m.focus
    m.dialog = newDialog()
    m.focus = FocusDialog
}

// 关闭对话框后恢复
func (m *model) closeDialog() {
    m.dialog = nil
    m.focus = m.previousFocus
}
```

## 响应式布局

### 处理窗口大小变化

```go
case tea.WindowSizeMsg:
    m.width = msg.Width
    m.height = msg.Height

    // 自适应布局断点
    if m.width < 80 {
        m.layout = compactLayout
    } else if m.width < 120 {
        m.layout = normalLayout
    } else {
        m.layout = wideLayout
    }

    // 通知子组件
    m.list.SetSize(m.listWidth(), m.height)
```

### 最小终端尺寸检查

```go
const (
    minWidth  = 60
    minHeight = 20
)

func (m model) View() tea.View {
    if m.width < minWidth || m.height < minHeight {
        return tea.NewView("Terminal too small. Please resize.")
    }
    return tea.NewView(m.renderContent())
}
```
