# Lip Gloss Styling & Layout

Lip Gloss (`charm.land/lipgloss/v2`) is the terminal styling library in the Bubble Tea ecosystem, providing a CSS-like declarative API.

## Table of Contents

- [Style Basics](#style-basics)
- [Colors](#colors)
- [Borders](#borders)
- [Padding & Margin](#padding--margin)
- [Size Control](#size-control)
- [Text Alignment](#text-alignment)
- [Layout Composition](#layout-composition)
- [Canvas & Layer](#canvas--layer)
- [Performance Optimization](#performance-optimization)

## Style Basics

```go
import "charm.land/lipgloss/v2"

style := lipgloss.NewStyle().
    Bold(true).
    Italic(true).
    Underline(true).
    Strikethrough(true).
    Reverse(true).        // Reverse colors
    Blink(true).          // Blink
    Faint(true).          // Dim
    Foreground(lipgloss.Color("#FF0000")).
    Background(lipgloss.Color("#0000FF"))

fmt.Println(style.Render("Styled text"))
```

### Copy & Inherit

```go
baseStyle := lipgloss.NewStyle().
    Foreground(lipgloss.Color("#FFF")).
    Padding(1)

// Copy() creates an independent copy; modifications don't affect the original
accentStyle := baseStyle.Copy().
    Foreground(lipgloss.Color("#7D56F4"))

// Inherit() creates an inherited copy
inherited := baseStyle.Inherit(accentStyle)
```

## Colors

### Color Value Formats

```go
// Hexadecimal
lipgloss.Color("#FF0000")
lipgloss.Color("#F00")        // Short form

// ANSI 256 color
lipgloss.Color("202")         // Indexed color
lipgloss.Color("5")           // 4-bit ANSI

// Adaptive color (auto-switches for dark/light backgrounds)
lipgloss.AdaptiveColor{Light: "#333", Dark: "#FFF"}
lipgloss.AdaptiveColor{Light: "#FF0000", Dark: "#FF6666"}

// No color (terminal default)
lipgloss.NoColor{}
```

### Color Profile

Lip Gloss auto-detects terminal color capabilities and degrades gracefully:
- **TrueColor** (16M colors): Modern terminals
- **ANSI256** (256 colors): Older terminals
- **ANSI** (16 colors): Basic terminals
- **Ascii**: No color

```go
// Get current terminal color profile
profile := lipgloss.DefaultRenderer().ColorProfile()

// Manually specify
renderer := lipgloss.NewRenderer(
    os.Stdout,
    lipgloss.WithColorProfile(colorprofile.ANSI256),
)
```

## Borders

### Built-in Border Styles

```go
lipgloss.NormalBorder()   // ─ │ ┌ ┐ └ ┘ ├ ┤ ┬ ┴ ┼
lipgloss.RoundedBorder()  // ─ │ ╭ ╮ ╰ ╯ ├ ┤ ┬ ┴ ┼
lipgloss.DoubleBorder()   // ═ ║ ╔ ╗ ╚ ╝ ╠ ╣ ╦ ╩ ╬
lipgloss.ThickBorder()    // ━ ┃ ┏ ┓ ┗ ┛ ┣ ┫ ┳ ┻ ╋
lipgloss.HiddenBorder()   // Transparent border (takes space but invisible)
```

### Custom Border

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
    BorderTop(true).BorderBottom(true).   // Show only certain sides
    BorderLeft(true).BorderRight(true).
    BorderForeground(lipgloss.Color("#7D56F4")).
    BorderBackground(lipgloss.Color("#1A1B26"))  // v2 new
```

## Padding & Margin

```go
style := lipgloss.NewStyle().
    Padding(1).            // All sides 1
    Padding(1, 2).         // vertical 1, horizontal 2
    Padding(1, 2, 3).      // top 1, horizontal 2, bottom 3
    Padding(1, 2, 3, 4).   // top 1, right 2, bottom 3, left 4

    PaddingTop(1).
    PaddingRight(2).
    PaddingBottom(3).
    PaddingLeft(4)

// Margin works the same way
    Margin(1).
    MarginTop(1).MarginRight(2).MarginBottom(3).MarginLeft(4)
```

## Size Control

```go
style := lipgloss.NewStyle().
    Width(40).         // Fixed width
    Height(10).        // Fixed height
    MaxWidth(80).      // Maximum width
    MaxHeight(20).     // Maximum height

// Get rendered dimensions
w := style.GetWidth()
h := style.GetHeight()

// Get box model dimensions
style.GetFrameSize()  // (horizontal, vertical) margin+border+padding
```

## Text Alignment

```go
style := lipgloss.NewStyle().
    Align(lipgloss.Left).     // Horizontal left
    Align(lipgloss.Center).   // Horizontal center
    Align(lipgloss.Right).    // Horizontal right

    AlignVertical(lipgloss.Top).     // Vertical top
    AlignVertical(lipgloss.Center).  // Vertical center
    AlignVertical(lipgloss.Bottom)   // Vertical bottom
```

## Layout Composition

### JoinHorizontal / JoinVertical

```go
left := style1.Render("left panel")
right := style2.Render("right panel")
top := style3.Render("top")
bottom := style4.Render("bottom")

// Horizontal join (specify vertical alignment)
row := lipgloss.JoinHorizontal(lipgloss.Top, left, right)
row := lipgloss.JoinHorizontal(lipgloss.Center, left, right)
row := lipgloss.JoinHorizontal(lipgloss.Bottom, left, right)

// Vertical join (specify horizontal alignment)
col := lipgloss.JoinVertical(lipgloss.Left, top, bottom)
col := lipgloss.JoinVertical(lipgloss.Center, top, bottom)
```

### Position

```go
// Position content within a specified area
positioned := lipgloss.Place(
    width, height,
    lipgloss.Center, lipgloss.Center,  // horizontal and vertical alignment
    content,
)

// Overlay positioning (content overlaid on background)
overlay := lipgloss.PlaceOverlay(x, y, overlay, background)
```

### Practical Layout Example

```go
// Three-column layout
func (m model) View() tea.View {
    sidebar := m.renderSidebar()
    main := m.renderMain()
    status := m.renderStatusBar()

    // Left-right split
    body := lipgloss.JoinHorizontal(lipgloss.Top, sidebar, main)

    // Top-bottom composition
    content := lipgloss.JoinVertical(lipgloss.Left, body, status)

    return tea.NewView(content)
}

// Use Size to ensure alignment
func (m model) renderSidebar() string {
    return lipgloss.NewStyle().
        Width(20).
        Height(m.height - 2).     // Subtract status bar
        Padding(1).
        BorderRight(true).
        Render(listContent)
}
```

## Canvas & Layer

v2 supports advanced Canvas and Layer APIs:

```go
// Canvas — pixel-level rendering
canvas := lipgloss.NewCanvas(80, 24)
canvas.Set(5, 5, lipgloss.NewStyle().
    Foreground(lipgloss.Color("#FF0000")).
    Render("X"))
canvas.SetString(10, 3, "Hello from canvas")

// Layer — compositing (supports transparency)
layer := lipgloss.NewLayer(80, 24)
layer.Place(lipgloss.Position{X: 5, Y: 5}, overlayContent)
layer.Render()
```

## Performance Optimization

### Pre-allocate Styles

```go
// ❌ Creating styles in every View call
func (m model) View() tea.View {
    return tea.NewView(lipgloss.NewStyle().Bold(true).Render("text"))
}

// ✅ Pre-allocate globals
var boldStyle = lipgloss.NewStyle().Bold(true)

func (m model) View() tea.View {
    return tea.NewView(boldStyle.Render("text"))
}
```

### Use strings.Builder

```go
func (m model) renderLargeList(items []string) string {
    var b strings.Builder
    b.Grow(len(items) * 50)  // Pre-allocate
    for _, item := range items {
        b.WriteString(style.Render(item))
        b.WriteByte('\n')
    }
    return b.String()
}
```

### Cache Render Results

Skip re-rendering when content hasn't changed:

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

### Avoid Deeply Nested Style Calls

```go
// ❌ Creating new styles on each nesting
style.Copy().Width(20).Copy().Padding(1).Render(text)

// ✅ Set all properties at once
style.Width(20).Padding(1).Render(text)
```
