# 最佳实践

以下模式来自生产级 Bubble Tea 应用，可直接套用到自己的项目中。

## 目录

- [1. 集中式样式管理](#1-集中式样式管理)
- [2. 分层的 KeyMap 模式](#2-分层的-keymap-模式)
- [3. 对话框覆盖层栈](#3-对话框覆盖层栈)
- [4. 带 TTL 的状态消息](#4-带-ttl-的状态消息)
- [5. 两阶段布局重算](#5-两阶段布局重算)
- [6. canvas 渲染模式（可选）](#6-canvas-渲染模式可选)
- [7. 事件通道桥接外部异步事件](#7-事件通道桥接外部异步事件)
- [8. 语义化渲染辅助函数](#8-语义化渲染辅助函数)

## 1. 集中式样式管理

将所有 Lip Gloss 样式定义在一个 `Styles` 结构体中，启动时一次性分配，View 中只引用不创建：

```go
type Styles struct {
    Header   lipgloss.Style
    List     lipgloss.Style
    Selected lipgloss.Style
    Status   struct {
        Info    lipgloss.Style
        Error   lipgloss.Style
    }
}

func NewStyles() *Styles {
    s := &Styles{}
    s.Header = lipgloss.NewStyle().Bold(true).Padding(0, 1)
    s.List = lipgloss.NewStyle().Padding(0, 1)
    s.Selected = lipgloss.NewStyle().Foreground(lipgloss.Color("#FFF")).Background(lipgloss.Color("#7D56F4"))
    s.Status.Info = lipgloss.NewStyle().Foreground(lipgloss.Color("#99"))
    s.Status.Error = lipgloss.NewStyle().Foreground(lipgloss.Color("#F00"))
    return s
}

// 通过 Common 结构体注入所有组件
type Common struct {
    Styles *Styles
    Config *Config
}

// 在 NewProgram 之前初始化一次
com := &Common{Styles: NewStyles(), Config: LoadConfig()}
p := tea.NewProgram(NewModel(com))
```

**要点**：
- 所有样式在 `NewStyles()` 中一次性分配，View 中零分配
- 通过 `Common` 结构体注入，避免全局变量
- 子组件通过构造函数接收 `*Common`，不自己创建样式

**进阶：双色板主题系统（深色/浅色）**

为每个语义色同时定义 hex 和 xterm 回退值，在启动时检测终端色彩能力、`NO_COLOR` 环境变量、`TERM=dumb`，一次性构建主题调色板：

```go
type cliColor struct {
    hex   string  // TrueColor 终端使用
    xterm int     // 256 色终端回退
}

type cliPalette struct {
    accent    cliColor  // 品牌主色
    muted     cliColor  // 正文
    faint     cliColor  // 辅助文字
    success   cliColor  // 成功
    warn      cliColor  // 警告
    err       cliColor  // 错误
    border    cliColor  // 边框
    selection cliColor  // 选中高亮
}

var darkTheme = cliPalette{
    accent:    cliColor{"#d97757", 173},
    muted:     cliColor{"#c0c4cc", 251},
    faint:     cliColor{"#858b96", 245},
    success:   cliColor{"#74b87a", 108},
    warn:      cliColor{"#d9a441", 179},
    err:       cliColor{"#e0696a", 167},
    border:    cliColor{"#343945", 237},
    selection: cliColor{"#d97757", 173},
}

var lightTheme = cliPalette{ /* 浅色对应的色值 */ }

// 颜色选择：TrueColor 用 hex，回退到 xterm
func themeLipColor(c cliColor) color.Color {
    if supportsTrueColor && c.hex != "" {
        return lipgloss.Color(c.hex)
    }
    return lipgloss.Color(strconv.Itoa(c.xterm))
}

// 检测是否应启用颜色
var colorEnabled = func() bool {
    if os.Getenv("NO_COLOR") != "" || os.Getenv("TERM") == "dumb" {
        return false
    }
    return term.IsTerminal(int(os.Stdout.Fd()))
}()

// 样式工厂：用调色板构建样式，colorEnabled=false 时跳过颜色
func themeStyle(c cliColor) lipgloss.Style {
    if !colorEnabled { return lipgloss.NewStyle() }
    return lipgloss.NewStyle().Foreground(themeLipColor(c))
}
```

之后用 `dim(s) = themeStyle(palette.faint).Render(s)`、`accent(s) = themeStyle(palette.accent).Render(s)` 等语义函数包裹文字，避免散落原始 lipgloss 调用。

## 2. 分层的 KeyMap 模式

将按键绑定按功能区域组织为嵌套结构体，统一在 `DefaultKeyMap()` 中定义默认值：

```go
type KeyMap struct {
    Global struct {
        Quit  key.Binding
        Help  key.Binding
    }
    List struct {
        Up     key.Binding
        Down   key.Binding
        Select key.Binding
    }
    Editor struct {
        Send    key.Binding
        Newline key.Binding
    }
}

func DefaultKeyMap() KeyMap {
    km := KeyMap{}
    km.Global.Quit = key.NewBinding(
        key.WithKeys("ctrl+c", "q"),
        key.WithHelp("q", "quit"),
    )
    km.Global.Help = key.NewBinding(
        key.WithKeys("?"),
        key.WithHelp("?", "help"),
    )
    km.List.Up = key.NewBinding(
        key.WithKeys("up", "k"),
        key.WithHelp("↑/k", "up"),
    )
    km.List.Down = key.NewBinding(
        key.WithKeys("down", "j"),
        key.WithHelp("↓/j", "down"),
    )
    km.Editor.Send = key.NewBinding(
        key.WithKeys("enter"),
        key.WithHelp("enter", "send"),
    )
    km.Editor.Newline = key.NewBinding(
        key.WithKeys("shift+enter", "ctrl+j"),
        key.WithHelp("ctrl+j", "newline"),
    )
    return km
}

// 通过实现 help.KeyMap 接口来支持 help 组件
func (km KeyMap) ShortHelp() []key.Binding {
    return []key.Binding{km.Global.Quit, km.List.Up, km.List.Down}
}
func (km KeyMap) FullHelp() [][]key.Binding { /* 返回完整分组 */ }
```

**要点**：
- 嵌套结构体的字段名（`Global`、`List`、`Editor`）即表示焦点区域
- 可用 `key.Binding.SetHelp()` 在运行时动态修改帮助文本（如根据上下文切换提示）
- 实现 `help.KeyMap` 接口后直接对接 `help` 组件

## 3. 对话框覆盖层栈

用接口 + 栈管理多层对话框（确认框 → 表单 → 详情），上层优先处理消息：

```go
// Dialog 接口
type Dialog interface {
    ID() string
    HandleMsg(msg tea.Msg) any   // 返回 Action，由调用方处理
}

// Overlay 管理对话框栈
type Overlay struct {
    stack []Dialog
}

func (o *Overlay) OpenDialog(d Dialog) {
    o.stack = append(o.stack, d)
}

func (o *Overlay) CloseFrontDialog() {
    if len(o.stack) > 0 {
        o.stack = o.stack[:len(o.stack)-1]
    }
}

func (o *Overlay) HasDialogs() bool {
    return len(o.stack) > 0
}

// Update 中：对话框优先消费消息
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    if m.overlay.HasDialogs() {
        return m.handleDialog(msg)
    }
    // 正常流程...
}
```

**Grace Period（防误操作）**：打开异步对话框（如权限请求）时，前一个组件可能还有飞行中的按键。用 200ms 静默期 + 1500ms 绝对超时吸收这些残余按键：

```go
func (o *Overlay) OpenDialogWithGrace(d Dialog) {
    now := time.Now()
    o.stack = append(o.stack, d)
    o.graceOpenedAt = now
    o.graceLastInputAt = now
}

// Update 中：Grace Period 内吸收按键
if _, ok := msg.(tea.KeyPressMsg); ok && m.overlay.inGracePeriod() {
    m.overlay.graceLastInputAt = time.Now()
    return m, nil  // 丢弃按键
}
```

**进阶：阻塞式审批（pendingApproval）**：当对话框需要阻塞后台 goroutine 等待用户决策时（如工具调用审批），在 Model 中设置 `pendingApproval` 字段，按键和鼠标输入被拦截用于"批准/拒绝"操作。后台 goroutine 通过 channel 等待审批结果：

```go
// Model 中
pendingApproval *Approval  // nil = 无待审批项

// Update 中：有待审批项时，按键走审批逻辑
if m.pendingApproval != nil {
    switch msg.(type) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "y":
            m.ctrl.Approve(m.pendingApproval)
            m.pendingApproval = nil
        case "n":
            m.ctrl.Deny(m.pendingApproval)
            m.pendingApproval = nil
        }
    }
    return m, nil  // 拦截所有其他消息
}
```

## 4. 带 TTL 的状态消息

状态栏消息（成功/错误/警告）自动在指定时间后消失：

```go
type InfoType int
const (
    InfoTypeError   InfoType = iota
    InfoTypeWarn
    InfoTypeSuccess
)

type InfoMsg struct {
    Type InfoType
    Msg  string
}

// 设置消息并启动自动清除
func (m *model) setInfoMsg(msg InfoMsg) tea.Cmd {
    m.statusMsg = msg
    return tea.Tick(5*time.Second, func(time.Time) tea.Msg {
        return clearStatusMsg{}
    })
}
```

**要点**：
- 每种 `InfoType` 使用不同的 Lip Gloss 样式（如红色/黄色/绿色）
- View 中根据 `statusMsg.IsEmpty()` 决定是否渲染
- 调用 `setInfoMsg()` 时会自动取消上一个 TTL 定时器（因为旧消息被覆盖）

## 5. 两阶段布局重算

文本区域的 `SetWidth()` 可能因自动换行导致高度变化，用两阶段重算处理：

```go
func (m *model) updateLayoutAndSize() {
    // 第一阶段：基于当前 textarea 高度计算布局
    prevHeight := m.textarea.Height()
    m.layout = m.calculateLayout(m.width, m.height)
    m.textarea.SetWidth(m.layout.editorWidth)

    // 第二阶段：如果 SetWidth 改变了 textarea 高度，再做一次调整
    if m.textarea.Height() != prevHeight {
        m.layout = m.calculateLayout(m.width, m.height)
        m.textarea.SetWidth(m.layout.editorWidth)
    }
}

// 监听导致布局变化的操作
case tea.WindowSizeMsg:
    m.width, m.height = msg.Width, msg.Height
    m.updateLayoutAndSize()
```

**要点**：
- 需要重新计算布局的场景：窗口大小变化、textarea 内容变化、状态切换、焦点切换
- 使用 `handleXxxChange` 模式：比较变化前后的值，只在确实变化时才触发重算
- 对 textarea，封装一个 `updateTextareaWithPrevHeight()` 方法来统一处理"修改前记录高度 → Update → 检查是否需要重算"

**进阶：动态底部高度计算**：不要硬编码底部区域高度。底部可能有审批横幅、todo 面板、选择器、补全菜单等多个面板，它们随状态出现/消失。每次渲染时动态计算：

```go
func (m chatTUI) bottomRows() int {
    rows := 0
    for _, s := range []string{
        m.renderTodoPanel(),
        m.renderApprovalBanner(),
        m.renderChooser(),
        m.renderCompletion(),
    } {
        if s != "" {
            rows += strings.Count(s, "\n") + 1
        }
    }
    if !m.hideComposer() {
        rows += m.input.Height() + 2   // textarea 高度 + 边框
    }
    return rows + m.statusLineCount    // 底部固定状态行
}

// 在 View 中：viewport 高度 = 窗口高度 - 底部行数
vpHeight := m.height - m.bottomRows()
if vpHeight < 1 { vpHeight = 1 }
m.viewport.SetHeight(vpHeight)
```

每个 panel 的 render 方法在无内容时返回 `""`，有内容时返回渲染好的字符串。`bottomRows()` 只数字符串行数——不需要手动为每种状态组合维护高度常量。

## 6. canvas 渲染模式（可选）

对于复杂布局（多栏、需精确控制光标位置），使用 `ultraviolet` 的 `ScreenBuffer` 逐个区域绘制：

```go
func (m model) View() tea.View {
    canvas := uv.NewScreenBuffer(m.width, m.height)

    // 各子组件绘制到 canvas 的不同区域
    m.sidebar.Draw(canvas, m.layout.sidebar)
    m.main.Draw(canvas, m.layout.main)
    m.statusBar.Draw(canvas, m.layout.status)

    // 光标位置由组件内部决定（如 textarea 的光标位置）
    v := tea.NewView(canvas.Render())
    v.Cursor = m.textarea.Cursor()  // 从组件获取光标
    return v
}
```

**要点**：
- 每个子组件实现 `Draw(scr uv.Screen, area uv.Rectangle)` 方法
- 布局信息（每个区域的 `image.Rectangle`）在 `calculateLayout` 中预先计算
- 简单应用直接用 `tea.NewView(string)` + `lipgloss.Join*` 拼接即可，无需引入 ultraviolet
- ultraviolet 引入的额外依赖较重，仅在需要精确坐标控制或 canvas 级别操作时才使用

## 7. 事件通道桥接外部异步事件

当 Model 需要消费来自后台 goroutine（如 AI Agent 的 stream 事件、WebSocket 消息）的异步事件时，用 Go channel + `tea.Cmd` 桥接：

```go
// 定义一个 Cmd，阻塞在 channel 上等待事件
func waitForAgentEvent(ch chan event.Event) tea.Cmd {
    return func() tea.Msg {
        return agentEventMsg(<-ch)  // 阻塞直到事件到达
    }
}

// Init 中启动事件监听
func (m chatTUI) Init() tea.Cmd {
    return tea.Batch(
        textarea.Blink,
        waitForAgentEvent(m.eventCh),  // 开始监听 agent 事件
    )
}

// Update 中：处理完一个事件后，立即重新注册监听
case agentEventMsg:
    m.handleAgentEvent(event.Event(msg))
    // 重新注册，形成"监听 → 处理 → 再监听"的循环
    cmds = append(cmds, waitForAgentEvent(m.eventCh))
```

**calmdown 防抖**：当后台事件洪泛时（如大量 stream token），限制单次 Update 处理的事件数，超过阈值后先 yield 渲染再继续：

```go
const maxEventDrain = 512  // 单次 Update 最多处理 512 个事件

drained := 0
for drained < maxEventDrain {
    select {
    case ev := <-m.eventCh:
        m.handleAgentEvent(ev)
        drained++
    default:
        break  // channel 已空，yield 渲染
    }
}
```

> 这比 `tea.Batch` 并行执行更好，因为 agent 事件的顺序必须保留。

## 8. 语义化渲染辅助函数

不要在各处散落原始 `lipgloss.NewStyle()` 调用，而是封装一层语义化的 helper 函数：

```go
// 语义化的渲染 helpers，内部使用预构建的调色板
func viewHeader(format string, args ...any) string {
    return accent(fmt.Sprintf(format, args...))
}

func viewSubhead(s string) string {
    return dim("  " + s)
}

func viewMeta(s string) string {
    return dim(s)
}

func viewHint(s string) string {
    return dim("  " + s)
}

func viewMore(n int, noun string) string {
    if n <= 0 { return "" }
    return dim(fmt.Sprintf("  +%d more %s", n, noun))
}

// 文本裁剪（保持终端宽度安全）
func viewCompactPath(path string, width int) string {
    return compactMiddle(oneLineText(path), max(1, width))
}

func viewCompactText(s string, width int) string {
    return compactEnd(oneLineText(s), max(1, width))
}
```

**要点**：
- `accent()`、`dim()`、`bold()` 等底层函数封装了对调色板的引用，换主题只需改 palette 变量
- `viewHeader`/`viewSubhead`/`viewHint` 等语义命名让调用方代码自文档化
- 文本裁剪函数确保渲染不会超出终端宽度
- 这些 helpers 不依赖 `tea.Model`，可以在任何地方使用（包括子面板的 render 方法）
