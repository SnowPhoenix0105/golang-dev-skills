# Bubbles 组件库

Bubbles 是 Bubble Tea 官方组件库，所有组件遵循与 `tea.Model` 相同的 Update/View 模式（注意其 `View()` 返回 `string` 而非 `tea.View`，且无 `Init()` 方法），可直接嵌套使用。

## 目录

- [通用模式](#通用模式)
- [spinner — 加载指示器](#spinner--加载指示器)
- [textinput — 单行文本输入](#textinput--单行文本输入)
- [textarea — 多行文本输入](#textarea--多行文本输入)
- [table — 表格](#table--表格)
- [list — 可选择列表](#list--可选择列表)
- [viewport — 可滚动视口](#viewport--可滚动视口)
- [paginator — 分页导航](#paginator--分页导航)
- [progress — 进度条](#progress--进度条)
- [filepicker — 文件选择器](#filepicker--文件选择器)
- [help — 帮助栏](#help--帮助栏)
- [key — 按键绑定](#key--按键绑定)
- [timer / stopwatch](#timer--stopwatch)

## 通用模式

所有 Bubbles 组件遵循一致的使用模式：

```go
// 1. 创建组件
component := component.New()

// 2. 嵌入主 Model
type model struct {
    component component.Model
}

// 3. 转发 Update（Bubbles 组件无 Init，不需要转发）
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmd tea.Cmd
    m.component, cmd = m.component.Update(msg)
    return m, cmd
}

// 4. 组合 View（Bubbles View() 返回 string，用 tea.NewView 包裹）
func (m model) View() tea.View {
    return tea.NewView(m.component.View())
}
```

> **注意**：Bubbles 组件无 `Init()` 方法，`View()` 返回 `string`（非 `tea.View`）。组件的 View 字符串需在父 Model 中用 `tea.NewView()` 包裹。

## spinner — 加载指示器

```go
import "charm.land/bubbles/v2/spinner"

// 基础用法
s := spinner.New()
s.Style = lipgloss.NewStyle().Foreground(lipgloss.Color("205"))

// 自定义 spinner 样式
s := spinner.New(spinner.WithSpinner(spinner.Dot))

// 启动
func (m model) Init() tea.Cmd {
    return s.Tick  // 启动动画
}

// View 中渲染
func (m model) View() tea.View {
    return tea.NewView(fmt.Sprintf("%s Loading...", s.View()))
}
```

内置 Spinner 类型：`spinner.Line`、`spinner.Dot`、`spinner.MiniDot`、`spinner.Jump`、`spinner.Pulse`、`spinner.Points`、`spinner.Globe`、`spinner.Moon`、`spinner.Monkey`、`spinner.Meter`、`spinner.Hamburger`。

## textinput — 单行文本输入

```go
import "charm.land/bubbles/v2/textinput"

ti := textinput.New()
ti.Placeholder = "Enter your name"
ti.Focus()           // 获取焦点
ti.CharLimit = 50    // 字符数限制
ti.Width = 30        // 显示宽度

// 设置样式
ti.PromptStyle = lipgloss.NewStyle().Foreground(lipgloss.Color("99"))
ti.TextStyle = lipgloss.NewStyle().Foreground(lipgloss.Color("255"))

// 常用方法
ti.SetValue("initial")
ti.Reset()
val := ti.Value()
```

### EchoMode（密码输入）

```go
ti.EchoMode = textinput.EchoPassword       // 遮蔽为 *
ti.EchoMode = textinput.EchoNone           // 完全不显示
```

### 数据验证

```go
ti.Validate = func(s string) error {
    if len(s) < 3 {
        return errors.New("too short")
    }
    return nil
}
```

## textarea — 多行文本输入

```go
import "charm.land/bubbles/v2/textarea"

ta := textarea.New()
ta.Placeholder = "Type your message..."
ta.Focus()
ta.CharLimit = 500        // -1 = 无限制
ta.ShowLineNumbers = false
ta.SetHeight(6)
ta.SetWidth(60)

// 动态高度
ta.DynamicHeight = true   // 根据内容自动调整高度
ta.MinHeight = 3
ta.MaxHeight = 10

// 样式
ta.FocusedStyle.CursorLine = lipgloss.NewStyle().
    Background(lipgloss.Color("57"))
ta.BlurredStyle.CursorLine = lipgloss.NewStyle().
    Background(lipgloss.Color("240"))

// 获取/设置内容
content := ta.Value()
ta.SetValue("initial text")
ta.Reset()
```

### 完整样式设置

```go
ta.SetStyles(textarea.Styles{
    Base:       lipgloss.NewStyle().Padding(1),
    CursorLine: lipgloss.NewStyle().Background(lipgloss.Color("57")),
    Placeholder: lipgloss.NewStyle().Foreground(lipgloss.Color("240")),
    Text:       lipgloss.NewStyle().Foreground(lipgloss.Color("255")),
    Prompt:     lipgloss.NewStyle().Foreground(lipgloss.Color("99")),
})
```

## table — 表格

```go
import "charm.land/bubbles/v2/table"

columns := []table.Column{
    {Title: "Name", Width: 20},
    {Title: "Age", Width: 5},
    {Title: "City", Width: 15},
}

rows := []table.Row{
    {"Alice", "30", "New York"},
    {"Bob", "25", "San Francisco"},
}

t := table.New(
    table.WithColumns(columns),
    table.WithRows(rows),
    table.WithFocused(true),
    table.WithHeight(10),
)

// 样式
t.SetStyles(table.Styles{
    Header: lipgloss.NewStyle().Bold(true).Foreground(lipgloss.Color("99")),
    Cell:   lipgloss.NewStyle().Padding(0, 1),
    Selected: lipgloss.NewStyle().
        Foreground(lipgloss.Color("#FFF")).
        Background(lipgloss.Color("#7D56F4")),
})

// 选择
t.MoveUp(1)
t.MoveDown(1)
selectedRow := t.SelectedRow()
```

## list — 可选择列表

```go
import "charm.land/bubbles/v2/list"

// 定义列表项
type item struct {
    title, desc string
}

func (i item) Title() string       { return i.title }
func (i item) Description() string { return i.desc }
func (i item) FilterValue() string { return i.title }

items := []list.Item{
    item{title: "Alice", desc: "Engineer"},
    item{title: "Bob", desc: "Designer"},
}

// 创建列表
delegate := list.NewDefaultDelegate()
l := list.New(items, delegate, 0, 0)  // width, height
l.Title = "Team Members"
l.SetShowStatusBar(true)
l.SetFilteringEnabled(true)     // 启用过滤

// 样式
l.Styles.Title = lipgloss.NewStyle().
    Bold(true).Foreground(lipgloss.Color("99"))
```

### 自定义 Delegate

```go
type customDelegate struct{}

func (d customDelegate) Height() int                             { return 2 }
func (d customDelegate) Spacing() int                            { return 0 }
func (d customDelegate) Update(msg tea.Msg, m *list.Model) tea.Cmd { return nil }
func (d customDelegate) Render(w io.Writer, m list.Model, index int, item list.Item) {
    i, ok := item.(myItem)
    if !ok { return }
    str := fmt.Sprintf("%s\n  %s", i.Title(), i.Description())
    if index == m.Index() {
        str = lipgloss.NewStyle().Foreground(lipgloss.Color("205")).Render(str)
    }
    fmt.Fprint(w, str)
}
```

## viewport — 可滚动视口

```go
import "charm.land/bubbles/v2/viewport"

vp := viewport.New(width, height)
vp.SetContent("Long content that exceeds the viewport...")
vp.YOffset        // 当前垂直偏移
vp.TotalLineCount()  // 总行数
vp.ScrollPercent()   // 滚动百分比

// 滚动
vp.LineDown(1)
vp.LineUp(1)
vp.ViewDown()
vp.ViewUp()
vp.GotoTop()
vp.GotoBottom()

// 鼠标滚轮
vp.MouseWheelEnabled = true
vp.MouseWheelDelta = 3
```

## paginator — 分页导航

```go
import "charm.land/bubbles/v2/paginator"

p := paginator.New()
p.Type = paginator.Dots          // 圆点样式
p.PerPage = 10
p.TotalPages = len(items) / 10
p.Page = 0                       // 当前页

// 翻页
p.PrevPage()
p.NextPage()

// 获取范围
start, end := p.GetSliceBounds(len(items))
pageItems := items[start:end]

// 配合 viewport 或 table 使用
func (m model) updatePage() {
    start, end := m.paginator.GetSliceBounds(len(m.allRows))
    m.table.SetRows(m.allRows[start:end])
}
```

## progress — 进度条

```go
import "charm.land/bubbles/v2/progress"

prog := progress.New()
prog.Width = 40
prog.Percent = 0.5  // 50%

// 渐变填充
prog.FullGradient = "#7D56F4,#5A56F4"
prog.EmptyColor = "#333333"

// 动画（配合 harmonica）
prog := progress.New(progress.WithSpringOptions(harmonica.Options{
    Damping:   12,
    Frequency: 10,
}))

// 样式
prog.Style = lipgloss.NewStyle().
    BorderStyle(lipgloss.RoundedBorder()).
    BorderForeground(lipgloss.Color("63")).
    PaddingRight(2)
```

## filepicker — 文件选择器

```go
import "charm.land/bubbles/v2/filepicker"

fp := filepicker.New()
fp.CurrentDirectory, _ = os.Getwd()
fp.AllowedTypes = []string{".go", ".mod", ".sum"}  // 文件类型过滤
fp.ShowHidden = false

// 在 Update 中处理选中的文件
case tea.KeyPressMsg:
    switch msg.String() {
    case "enter":
        if path, err := fp.DidSelectFile(msg); err == nil {
            m.selectedFile = path
        }
    }
```

## help — 帮助栏

```go
import (
    "charm.land/bubbles/v2/help"
    "charm.land/bubbles/v2/key"
)

// 定义按键绑定
type keyMap struct {
    Up    key.Binding
    Down  key.Binding
    Quit  key.Binding
    Enter key.Binding
}

var keys = keyMap{
    Up:    key.NewBinding(key.WithKeys("up", "k"), key.WithHelp("↑/k", "up")),
    Down:  key.NewBinding(key.WithKeys("down", "j"), key.WithHelp("↓/j", "down")),
    Quit:  key.NewBinding(key.WithKeys("q", "ctrl+c"), key.WithHelp("q", "quit")),
    Enter: key.NewBinding(key.WithKeys("enter"), key.WithHelp("enter", "select")),
}

// 实现 help.KeyMap 接口
func (k keyMap) ShortHelp() []key.Binding {
    return []key.Binding{k.Up, k.Down, k.Enter, k.Quit}
}

func (k keyMap) FullHelp() [][]key.Binding {
    return [][]key.Binding{
        {k.Up, k.Down},
        {k.Enter, k.Quit},
    }
}

// 在 View 中渲染
func (m model) View() tea.View {
    helpView := m.help.ShortHelpView(m.keys.ShortHelp())
    content := m.list.View() + "\n" + helpView
    return tea.NewView(content)
}
```

`key.Binding` 支持的方法：
- `key.NewBinding(key.WithKeys("up", "k"), key.WithHelp("↑/k", "up"))`
- `Binding.SetHelp(key, description string)` — 动态修改帮助文本
- `Binding.SetEnabled(bool)` — 禁用绑定
- `Binding.Enabled() bool` — 检查是否启用

## timer / stopwatch

```go
import (
    "charm.land/bubbles/v2/timer"
    "charm.land/bubbles/v2/stopwatch"
)

// Timer — 倒计时
tm := timer.New(time.Minute)
tm.Start()    // 开始倒计时
tm.Stop()     // 暂停
tm.Reset()    // 重置

// 在 Update 中接收超时消息
case timer.TickMsg:
    // 每秒触发一次
case timer.TimeoutMsg:
    // 计时结束

// Stopwatch — 正计时
sw := stopwatch.New()
sw.Start()
sw.Stop()
sw.Reset()
```
