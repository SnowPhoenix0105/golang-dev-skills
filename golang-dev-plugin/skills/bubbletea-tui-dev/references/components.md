# Bubbles Component Library

Bubbles is the official Bubble Tea component library. All components follow the same Update/View pattern as `tea.Model` (note: their `View()` returns `string` not `tea.View`, and they have no `Init()` method), and can be directly nested.

## Table of Contents

- [Common Patterns](#common-patterns)
- [spinner — Loading Indicator](#spinner--loading-indicator)
- [textinput — Single-line Text Input](#textinput--single-line-text-input)
- [textarea — Multi-line Text Input](#textarea--multi-line-text-input)
- [table — Table](#table--table)
- [list — Selectable List](#list--selectable-list)
- [viewport — Scrollable Viewport](#viewport--scrollable-viewport)
- [paginator — Page Navigation](#paginator--page-navigation)
- [progress — Progress Bar](#progress--progress-bar)
- [filepicker — File Picker](#filepicker--file-picker)
- [help — Help Bar](#help--help-bar)
- [key — Key Bindings](#key--key-bindings)
- [timer / stopwatch](#timer--stopwatch)

## Common Patterns

All Bubbles components follow a consistent usage pattern:

```go
// 1. Create component
component := component.New()

// 2. Embed in main Model
type model struct {
    component component.Model
}

// 3. Forward Update (Bubbles components have no Init, no forwarding needed)
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmd tea.Cmd
    m.component, cmd = m.component.Update(msg)
    return m, cmd
}

// 4. Compose View (Bubbles View() returns string, wrap with tea.NewView)
func (m model) View() tea.View {
    return tea.NewView(m.component.View())
}
```

> **Note**: Bubbles components have no `Init()` method and their `View()` returns `string` (not `tea.View`). Wrap the component's View string with `tea.NewView()` in the parent Model's View.

## spinner — Loading Indicator

```go
import "charm.land/bubbles/v2/spinner"

// Basic usage
s := spinner.New()
s.Style = lipgloss.NewStyle().Foreground(lipgloss.Color("205"))

// Custom spinner style
s := spinner.New(spinner.WithSpinner(spinner.Dot))

// Start animation
func (m model) Init() tea.Cmd {
    return s.Tick
}

// Render in View
func (m model) View() tea.View {
    return tea.NewView(fmt.Sprintf("%s Loading...", s.View()))
}
```

Built-in spinner types: `spinner.Line`, `spinner.Dot`, `spinner.MiniDot`, `spinner.Jump`, `spinner.Pulse`, `spinner.Points`, `spinner.Globe`, `spinner.Moon`, `spinner.Monkey`, `spinner.Meter`, `spinner.Hamburger`.

## textinput — Single-line Text Input

```go
import "charm.land/bubbles/v2/textinput"

ti := textinput.New()
ti.Placeholder = "Enter your name"
ti.Focus()           // Gain focus
ti.CharLimit = 50    // Character limit
ti.Width = 30        // Display width

// Style
ti.PromptStyle = lipgloss.NewStyle().Foreground(lipgloss.Color("99"))
ti.TextStyle = lipgloss.NewStyle().Foreground(lipgloss.Color("255"))

// Common methods
ti.SetValue("initial")
ti.Reset()
val := ti.Value()
```

### EchoMode (Password Input)

```go
ti.EchoMode = textinput.EchoPassword       // Mask as *
ti.EchoMode = textinput.EchoNone           // Show nothing
```

### Validation

```go
ti.Validate = func(s string) error {
    if len(s) < 3 {
        return errors.New("too short")
    }
    return nil
}
```

## textarea — Multi-line Text Input

```go
import "charm.land/bubbles/v2/textarea"

ta := textarea.New()
ta.Placeholder = "Type your message..."
ta.Focus()
ta.CharLimit = 500        // -1 = unlimited
ta.ShowLineNumbers = false
ta.SetHeight(6)
ta.SetWidth(60)

// Dynamic height
ta.DynamicHeight = true   // Auto-adjust height based on content
ta.MinHeight = 3
ta.MaxHeight = 10

// Style
ta.FocusedStyle.CursorLine = lipgloss.NewStyle().
    Background(lipgloss.Color("57"))
ta.BlurredStyle.CursorLine = lipgloss.NewStyle().
    Background(lipgloss.Color("240"))

// Get/set content
content := ta.Value()
ta.SetValue("initial text")
ta.Reset()
```

### Full Style Configuration

```go
ta.SetStyles(textarea.Styles{
    Base:       lipgloss.NewStyle().Padding(1),
    CursorLine: lipgloss.NewStyle().Background(lipgloss.Color("57")),
    Placeholder: lipgloss.NewStyle().Foreground(lipgloss.Color("240")),
    Text:       lipgloss.NewStyle().Foreground(lipgloss.Color("255")),
    Prompt:     lipgloss.NewStyle().Foreground(lipgloss.Color("99")),
})
```

## table — Table

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

// Style
t.SetStyles(table.Styles{
    Header: lipgloss.NewStyle().Bold(true).Foreground(lipgloss.Color("99")),
    Cell:   lipgloss.NewStyle().Padding(0, 1),
    Selected: lipgloss.NewStyle().
        Foreground(lipgloss.Color("#FFF")).
        Background(lipgloss.Color("#7D56F4")),
})

// Selection
t.MoveUp(1)
t.MoveDown(1)
selectedRow := t.SelectedRow()
```

## list — Selectable List

```go
import "charm.land/bubbles/v2/list"

// Define list items
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

// Create list
delegate := list.NewDefaultDelegate()
l := list.New(items, delegate, 0, 0)  // width, height
l.Title = "Team Members"
l.SetShowStatusBar(true)
l.SetFilteringEnabled(true)     // Enable filtering

// Style
l.Styles.Title = lipgloss.NewStyle().
    Bold(true).Foreground(lipgloss.Color("99"))
```

### Custom Delegate

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

## viewport — Scrollable Viewport

```go
import "charm.land/bubbles/v2/viewport"

vp := viewport.New(width, height)
vp.SetContent("Long content that exceeds the viewport...")
vp.YOffset              // Current vertical offset
vp.TotalLineCount()     // Total line count
vp.ScrollPercent()      // Scroll percentage

// Scrolling
vp.LineDown(1)
vp.LineUp(1)
vp.ViewDown()
vp.ViewUp()
vp.GotoTop()
vp.GotoBottom()

// Mouse wheel
vp.MouseWheelEnabled = true
vp.MouseWheelDelta = 3
```

## paginator — Page Navigation

```go
import "charm.land/bubbles/v2/paginator"

p := paginator.New()
p.Type = paginator.Dots          // Dot style
p.PerPage = 10
p.TotalPages = len(items) / 10
p.Page = 0                       // Current page

// Navigate pages
p.PrevPage()
p.NextPage()

// Get range
start, end := p.GetSliceBounds(len(items))
pageItems := items[start:end]

// Use with viewport or table
func (m model) updatePage() {
    start, end := m.paginator.GetSliceBounds(len(m.allRows))
    m.table.SetRows(m.allRows[start:end])
}
```

## progress — Progress Bar

```go
import "charm.land/bubbles/v2/progress"

prog := progress.New()
prog.Width = 40
prog.Percent = 0.5  // 50%

// Gradient fill
prog.FullGradient = "#7D56F4,#5A56F4"
prog.EmptyColor = "#333333"

// Animation (with harmonica)
prog := progress.New(progress.WithSpringOptions(harmonica.Options{
    Damping:   12,
    Frequency: 10,
}))

// Style
prog.Style = lipgloss.NewStyle().
    BorderStyle(lipgloss.RoundedBorder()).
    BorderForeground(lipgloss.Color("63")).
    PaddingRight(2)
```

## filepicker — File Picker

```go
import "charm.land/bubbles/v2/filepicker"

fp := filepicker.New()
fp.CurrentDirectory, _ = os.Getwd()
fp.AllowedTypes = []string{".go", ".mod", ".sum"}  // File type filter
fp.ShowHidden = false

// Handle file selection in Update
case tea.KeyPressMsg:
    switch msg.String() {
    case "enter":
        if path, err := fp.DidSelectFile(msg); err == nil {
            m.selectedFile = path
        }
    }
```

## help — Help Bar

```go
import (
    "charm.land/bubbles/v2/help"
    "charm.land/bubbles/v2/key"
)

// Define key bindings
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

// Implement help.KeyMap interface
func (k keyMap) ShortHelp() []key.Binding {
    return []key.Binding{k.Up, k.Down, k.Enter, k.Quit}
}

func (k keyMap) FullHelp() [][]key.Binding {
    return [][]key.Binding{
        {k.Up, k.Down},
        {k.Enter, k.Quit},
    }
}

// Render in View
func (m model) View() tea.View {
    helpView := m.help.ShortHelpView(m.keys.ShortHelp())
    content := m.list.View() + "\n" + helpView
    return tea.NewView(content)
}
```

`key.Binding` methods:
- `key.NewBinding(key.WithKeys("up", "k"), key.WithHelp("↑/k", "up"))`
- `Binding.SetHelp(key, description string)` — Dynamically change help text
- `Binding.SetEnabled(bool)` — Disable binding
- `Binding.Enabled() bool` — Check if enabled

## timer / stopwatch

```go
import (
    "charm.land/bubbles/v2/timer"
    "charm.land/bubbles/v2/stopwatch"
)

// Timer — countdown
tm := timer.New(time.Minute)
tm.Start()    // Start countdown
tm.Stop()     // Pause
tm.Reset()    // Reset

// Receive in Update
case timer.TickMsg:
    // Fires every second
case timer.TimeoutMsg:
    // Time's up

// Stopwatch — count up
sw := stopwatch.New()
sw.Start()
sw.Stop()
sw.Reset()
```
