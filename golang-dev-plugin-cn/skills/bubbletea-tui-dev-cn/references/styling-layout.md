# Lip Gloss 样式与布局

Lip Gloss (`charm.land/lipgloss/v2`) 是 Bubble Tea 生态的终端样式库，提供类 CSS 的声明式 API。

## 目录

- [Style 基础](#style-基础)
- [颜色](#颜色)
- [边框](#边框)
- [Padding 与 Margin](#padding-与-margin)
- [尺寸控制](#尺寸控制)
- [文本对齐](#文本对齐)
- [组合布局](#组合布局)
- [Canvas 与 Layer](#canvas-与-layer)
- [性能优化](#性能优化)

## Style 基础

```go
import "charm.land/lipgloss/v2"

style := lipgloss.NewStyle().
    Bold(true).
    Italic(true).
    Underline(true).
    Strikethrough(true).
    Reverse(true).        // 反色
    Blink(true).          // 闪烁
    Faint(true).          // 淡化
    Foreground(lipgloss.Color("#FF0000")).
    Background(lipgloss.Color("#0000FF"))

fmt.Println(style.Render("Styled text"))
```

### 复制与继承

```go
baseStyle := lipgloss.NewStyle().
    Foreground(lipgloss.Color("#FFF")).
    Padding(1)

// Copy() 创建独立副本，修改不影响原样式
accentStyle := baseStyle.Copy().
    Foreground(lipgloss.Color("#7D56F4"))

// Inherit() 创建继承副本
inherited := baseStyle.Inherit(accentStyle)
```

## 颜色

### 颜色值格式

```go
// 十六进制
lipgloss.Color("#FF0000")
lipgloss.Color("#F00")        // 短格式

// ANSI 256 色
lipgloss.Color("202")         // 索引色
lipgloss.Color("5")           // 4-bit ANSI

// 自适应颜色（深色/浅色背景自动切换）
lipgloss.AdaptiveColor{Light: "#333", Dark: "#FFF"}
lipgloss.AdaptiveColor{Light: "#FF0000", Dark: "#FF6666"}

// 无颜色（终端默认）
lipgloss.NoColor{}
```

### 颜色 Profile

Lip Gloss 自动检测终端的颜色能力并降级：
- **TrueColor** (16M 色)：现代终端
- **ANSI256** (256 色)：较旧终端
- **ANSI** (16 色)：基本终端
- **Ascii**：无颜色

```go
// 获取当前终端颜色 profile
profile := lipgloss.DefaultRenderer().ColorProfile()

// 手动指定
renderer := lipgloss.NewRenderer(
    os.Stdout,
    lipgloss.WithColorProfile(colorprofile.ANSI256),
)
```

## 边框

### 内置边框样式

```go
lipgloss.NormalBorder()   // ─ │ ┌ ┐ └ ┘ ├ ┤ ┬ ┴ ┼
lipgloss.RoundedBorder()  // ─ │ ╭ ╮ ╰ ╯ ├ ┤ ┬ ┴ ┼
lipgloss.DoubleBorder()   // ═ ║ ╔ ╗ ╚ ╝ ╠ ╣ ╦ ╩ ╬
lipgloss.ThickBorder()    // ━ ┃ ┏ ┓ ┗ ┛ ┣ ┫ ┳ ┻ ╋
lipgloss.HiddenBorder()   // 透明边框（占位但不显示）
```

### 自定义边框

```go
style := lipgloss.NewStyle().
    Border(lipgloss.Border{
        Top:         "─",
        Bottom:      "─",
        Left:        "│",
        Right:       "│",
        TopLeft:     "┌",
        TopRight:    "┐",
        BottomLeft:  "└",
        BottomRight: "┘",
    }).
    BorderTop(true).BorderBottom(true).   // 只显示部分边
    BorderLeft(true).BorderRight(true).
    BorderForeground(lipgloss.Color("#7D56F4")).
    BorderBackground(lipgloss.Color("#1A1B26"))  // v2 新增
```

## Padding 与 Margin

```go
style := lipgloss.NewStyle().
    Padding(1).            // 四边 1
    Padding(1, 2).         // 上下1, 左右2
    Padding(1, 2, 3).      // 上1, 左右2, 下3
    Padding(1, 2, 3, 4).   // 上1, 右2, 下3, 左4

    PaddingTop(1).
    PaddingRight(2).
    PaddingBottom(3).
    PaddingLeft(4)

// Margin 同理
    Margin(1).
    MarginTop(1).MarginRight(2).MarginBottom(3).MarginLeft(4)
```

## 尺寸控制

```go
style := lipgloss.NewStyle().
    Width(40).         // 固定宽度
    Height(10).        // 固定高度
    MaxWidth(80).      // 最大宽度
    MaxHeight(20).     // 最大高度

// 获取渲染后的实际尺寸
w := style.GetWidth()
h := style.GetHeight()

// 获取框模型尺寸
style.GetFrameSize()  // (horizontal, vertical) margin+border+padding
```

## 文本对齐

```go
style := lipgloss.NewStyle().
    Align(lipgloss.Left).     // 水平左对齐
    Align(lipgloss.Center).   // 水平居中
    Align(lipgloss.Right).    // 水平右对齐

    AlignVertical(lipgloss.Top).     // 垂直顶部
    AlignVertical(lipgloss.Center).  // 垂直居中
    AlignVertical(lipgloss.Bottom)   // 垂直底部
```

## 组合布局

### JoinHorizontal / JoinVertical

```go
left := style1.Render("left panel")
right := style2.Render("right panel")
top := style3.Render("top")
bottom := style4.Render("bottom")

// 水平拼接（指定垂直对齐方式）
row := lipgloss.JoinHorizontal(lipgloss.Top, left, right)
row := lipgloss.JoinHorizontal(lipgloss.Center, left, right)
row := lipgloss.JoinHorizontal(lipgloss.Bottom, left, right)

// 垂直拼接（指定水平对齐方式）
col := lipgloss.JoinVertical(lipgloss.Left, top, bottom)
col := lipgloss.JoinVertical(lipgloss.Center, top, bottom)
```

### Position

```go
// 在指定区域中定位内容
positioned := lipgloss.Place(
    width, height,
    lipgloss.Center, lipgloss.Center,  // 水平和垂直对齐
    content,
)

// 叠加定位（内容叠加在背景上）
overlay := lipgloss.PlaceOverlay(x, y, overlay, background)
```

### 实用布局示例

```go
// 三栏布局
func (m model) View() tea.View {
    sidebar := m.renderSidebar()
    main := m.renderMain()
    status := m.renderStatusBar()

    // 左右分栏
    body := lipgloss.JoinHorizontal(lipgloss.Top, sidebar, main)

    // 上下组合
    content := lipgloss.JoinVertical(lipgloss.Left, body, status)

    return tea.NewView(content)
}

// 使用 Size 确保对齐
func (m model) renderSidebar() string {
    return lipgloss.NewStyle().
        Width(20).
        Height(m.height - 2).     // 减去状态栏
        Padding(1).
        BorderRight(true).
        Render(listContent)
}
```

## Canvas 与 Layer

v2 支持高级的 Canvas 和 Layer API：

```go
// Canvas — 像素级渲染
canvas := lipgloss.NewCanvas(80, 24)
canvas.Set(5, 5, lipgloss.NewStyle().
    Foreground(lipgloss.Color("#FF0000")).
    Render("X"))
canvas.SetString(10, 3, "Hello from canvas")

// Layer — 叠加渲染（支持透明度）
layer := lipgloss.NewLayer(80, 24)
layer.Place(lipgloss.Position{X: 5, Y: 5}, overlayContent)
layer.Render()
```

## 性能优化

### 预分配样式

```go
// ❌ 每次 View 中创建样式
func (m model) View() tea.View {
    return tea.NewView(lipgloss.NewStyle().Bold(true).Render("text"))
}

// ✅ 预分配
var boldStyle = lipgloss.NewStyle().Bold(true)

func (m model) View() tea.View {
    return tea.NewView(boldStyle.Render("text"))
}
```

### 使用 strings.Builder

```go
func (m model) renderLargeList(items []string) string {
    var b strings.Builder
    b.Grow(len(items) * 50)  // 预分配
    for _, item := range items {
        b.WriteString(style.Render(item))
        b.WriteByte('\n')
    }
    return b.String()
}
```

### 缓存渲染结果

当内容不变时不重新渲染：

```go
type model struct {
    lastItems    []Item
    cachedRender string
}

func (m model) View() tea.View {
    if m.itemsChanged {
        m.cachedRender = m.render(m.lastItems)
        m.itemsChanged = false
    }
    return tea.NewView(m.cachedRender)
}
```

### 避免深度嵌套调用

```go
// ❌ 每次嵌套都创建新样式
style.Copy().Width(20).Copy().Padding(1).Render(text)

// ✅ 一次性设置
style.Width(20).Padding(1).Render(text)
```
