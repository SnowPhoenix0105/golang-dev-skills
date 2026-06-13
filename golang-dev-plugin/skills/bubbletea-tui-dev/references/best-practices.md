# Best Practices

These patterns are derived from production-grade Bubble Tea applications. Use them directly in your own projects.

## Table of Contents

- [1. Centralized Style Management](#1-centralized-style-management)
- [2. Layered KeyMap Pattern](#2-layered-keymap-pattern)
- [3. Dialog Overlay Stack](#3-dialog-overlay-stack)
- [4. Status Messages with TTL](#4-status-messages-with-ttl)
- [5. Two-Pass Layout Recalculation](#5-two-pass-layout-recalculation)
- [6. Canvas-Based Rendering (Optional)](#6-canvas-based-rendering-optional)

## 1. Centralized Style Management

Define all Lip Gloss styles in a single `Styles` struct, allocated once at startup. Never create styles inside View:

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

// Inject via a Common struct into all components
type Common struct {
    Styles *Styles
    Config *Config
}

// Initialize once before NewProgram
com := &Common{Styles: NewStyles(), Config: LoadConfig()}
p := tea.NewProgram(NewModel(com))
```

**Key points**:
- All styles allocated once in `NewStyles()` — zero allocations in View
- Injected via `Common` struct to avoid global variables
- Sub-components receive `*Common` through their constructors, never create their own styles

## 2. Layered KeyMap Pattern

Organize key bindings by functional area in a nested struct. Define defaults in a single `DefaultKeyMap()` function:

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

// Implement help.KeyMap to integrate with the help component
func (km KeyMap) ShortHelp() []key.Binding {
    return []key.Binding{km.Global.Quit, km.List.Up, km.List.Down}
}
func (km KeyMap) FullHelp() [][]key.Binding { /* return full grouped bindings */ }
```

**Key points**:
- Nested struct field names (`Global`, `List`, `Editor`) represent focus areas
- Use `key.Binding.SetHelp()` to dynamically change help text at runtime (e.g., context-aware hints)
- Implementing `help.KeyMap` enables direct integration with the `help` component

## 3. Dialog Overlay Stack

Use an interface + stack to manage multiple dialog layers (confirm → form → detail). Upper layers consume messages first:

```go
// Dialog interface
type Dialog interface {
    ID() string
    HandleMsg(msg tea.Msg) any   // Returns an Action, handled by the caller
}

// Overlay manages a dialog stack
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

// In Update: dialogs consume messages with priority
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    if m.overlay.HasDialogs() {
        return m.handleDialog(msg)
    }
    // Normal flow...
}
```

**Grace Period (accidental input prevention)**: When opening async dialogs (e.g., permission prompts), the previous component may still have in-flight keystrokes. Use a 200ms quiet period + 1500ms absolute timeout to absorb them:

```go
func (o *Overlay) OpenDialogWithGrace(d Dialog) {
    now := time.Now()
    o.stack = append(o.stack, d)
    o.graceOpenedAt = now
    o.graceLastInputAt = now
}

// In Update: absorb keystrokes during the Grace Period
if _, ok := msg.(tea.KeyPressMsg); ok && m.overlay.inGracePeriod() {
    m.overlay.graceLastInputAt = time.Now()
    return m, nil  // Discard keystroke
}
```

## 4. Status Messages with TTL

Status bar messages (success/error/warning) auto-clear after a specified duration:

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

// Set message and schedule auto-clear
func (m *model) setInfoMsg(msg InfoMsg) tea.Cmd {
    m.statusMsg = msg
    return tea.Tick(5*time.Second, func(time.Time) tea.Msg {
        return clearStatusMsg{}
    })
}
```

**Key points**:
- Each `InfoType` uses a distinct Lip Gloss style (e.g., red/yellow/green)
- View checks `statusMsg.IsEmpty()` to decide whether to render
- Calling `setInfoMsg()` automatically cancels the previous TTL timer (old message is overwritten)

## 5. Two-Pass Layout Recalculation

Setting width on a textarea can change its height due to word wrapping. Handle with two-pass recalculation:

```go
func (m *model) updateLayoutAndSize() {
    // Pass 1: calculate layout based on current textarea height
    prevHeight := m.textarea.Height()
    m.layout = m.calculateLayout(m.width, m.height)
    m.textarea.SetWidth(m.layout.editorWidth)

    // Pass 2: if SetWidth changed the textarea height, reconcile
    if m.textarea.Height() != prevHeight {
        m.layout = m.calculateLayout(m.width, m.height)
        m.textarea.SetWidth(m.layout.editorWidth)
    }
}

// Watch for operations that affect layout
case tea.WindowSizeMsg:
    m.width, m.height = msg.Width, msg.Height
    m.updateLayoutAndSize()
```

**Key points**:
- Scenarios requiring layout recalculation: window resize, textarea content change, state switch, focus switch
- Use the `handleXxxChange` pattern: compare before/after values, only recalculate if something actually changed
- For textarea, wrap with `updateTextareaWithPrevHeight()` to unify "record height before change → Update → check if recalculation needed"

## 6. Canvas-Based Rendering (Optional)

For complex layouts (multi-column, precise cursor positioning), use ultraviolet's `ScreenBuffer` to draw per-region:

```go
func (m model) View() tea.View {
    canvas := uv.NewScreenBuffer(m.width, m.height)

    // Each sub-component draws onto its own region of the canvas
    m.sidebar.Draw(canvas, m.layout.sidebar)
    m.main.Draw(canvas, m.layout.main)
    m.statusBar.Draw(canvas, m.layout.status)

    // Cursor position determined by active component (e.g., textarea cursor)
    v := tea.NewView(canvas.Render())
    v.Cursor = m.textarea.Cursor()
    return v
}
```

**Key points**:
- Each sub-component implements a `Draw(scr uv.Screen, area uv.Rectangle)` method
- Layout information (each region's `image.Rectangle`) is pre-computed in `calculateLayout`
- For simple apps, stick with `tea.NewView(string)` + `lipgloss.Join*` — no need to introduce ultraviolet
- Ultraviolet brings significant extra dependencies; only use it when you need precise coordinate control or canvas-level operations
