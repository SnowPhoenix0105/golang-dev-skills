---
name: bubbletea-tui-dev
description: Bubble Tea terminal UI (TUI) application development. A Go TUI framework based on The Elm Architecture. Covers bubbletea, bubble tea, TUI, terminal interface, command-line interface, tea.Model, tea.Cmd, tea.NewProgram, tea.KeyPressMsg, Bubbles components (spinner/textinput/textarea/table/list/viewport), Lip Gloss styling, Elm Architecture, terminal rendering, and more. Even if the user does not explicitly say "bubbletea", consider this skill for Go terminal UI development.
---

# Bubble Tea TUI Application Development

Bubble Tea (`charm.land/bubbletea/v2`) is a Go TUI framework based on [The Elm Architecture][elm], featuring declarative Views, message-driven unidirectional data flow, and a high-performance cell-based renderer.

[elm]: https://guide.elm-lang.org/architecture/

## Bubble Tea Ecosystem

| Repository | Purpose | go module |
| :--- | :--- | :--- |
| **bubbletea** | Core TUI framework: Model/Update/View lifecycle, message system, renderer, input handling | `charm.land/bubbletea/v2` |
| **bubbles** | Official component library: spinner, textinput, textarea, table, list, viewport, paginator, progress, filepicker, help, key, timer, stopwatch, etc. | `charm.land/bubbles/v2` |
| **lipgloss** | Terminal styling and layout: Style chain API, Color, Border, Padding/Margin, Align, Join, Canvas | `charm.land/lipgloss/v2` |
| **ultraviolet** | Low-level terminal abstraction: TerminalReader, ScreenBuffer, Key/Mouse event parsing, cell-buffer rendering | `github.com/charmbracelet/ultraviolet` |
| **catwalk** | Advanced layout engine: Flexbox layout, component tree, declarative UI, built on ultraviolet | `charm.land/catwalk` |
| **harmonica** | Spring animation library: physics-based smooth animations | `github.com/charmbracelet/harmonica` |
| **huh** | Interactive form and prompt toolkit | `github.com/charmbracelet/huh` |
| **glamour** | Markdown terminal rendering | `github.com/charmbracelet/glamour` |
| **wish** | SSH server middleware (expose Bubble Tea apps over SSH) | `github.com/charmbracelet/wish` |
| **bubblezone** | Mouse event zone tracking helper | `github.com/lrstanley/bubblezone` |

> **v1 → v2 Migration**: The biggest change in v2 is View returning `tea.View` struct instead of `string` (declarative), and key messages changing from `tea.KeyMsg` to `tea.KeyPressMsg`/`tea.KeyReleaseMsg`. Full migration guide: [UPGRADE_GUIDE_V2.md](https://github.com/charmbracelet/bubbletea/blob/main/UPGRADE_GUIDE_V2.md).

To browse local source code, use `go list -m -json charm.land/bubbletea/v2` to find the module path.

## Core Architecture: The Elm Architecture

Bubble Tea enforces unidirectional data flow:

```
User Input → Update(msg) → New Model → View() → Render
                  ↑                      ↓
                Cmd (async IO) → Msg ────┘
```

Three core methods, all on one Model:

| Method | Signature | Responsibility |
| :--- | :--- | :--- |
| **Init** | `Init() Cmd` | Returns initial commands (e.g., start timer, HTTP request). Return `nil` if no initial I/O. |
| **Update** | `Update(Msg) (Model, Cmd)` | Receives messages, returns updated Model and optional new Cmd. **Must return fast (<1ms)**; wrap time-consuming operations in Cmd. |
| **View** | `View() View` | Renders UI based on current Model state. v2 returns `tea.View` struct instead of string. |

**Model** is any type implementing the three methods above, typically a struct. Msg can be any type (`tea.Msg = uv.Event`).

## Core Principles

1. **Message-driven, not mutex** — All state changes go through `tea.Msg`; avoid `sync.Mutex` in Update/View
2. **Update must be fast** — Return within <1ms; wrap time-consuming I/O in `tea.Cmd` (async goroutine)
3. **Declarative View** — The `tea.View` struct returned by View declares terminal features (AltScreen, MouseMode, Cursor, WindowTitle, etc.) instead of using imperative program options
4. **Cmd composition** — Use `tea.Batch` for parallel, `tea.Sequence` for sequential Cmd execution
5. **Nested Models** — Complex apps embed sub-component Models in the parent Model and forward messages in Update

## Minimal Application

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

## Core Types

### Model Interface

```go
type Model interface {
    Init() Cmd
    Update(Msg) (Model, Cmd)
    View() View
}
```

### Msg

`tea.Msg` is an alias for `uv.Event` and can be any type. The framework automatically sends these message types:

| Message Type | Trigger |
| :--- | :--- |
| `tea.KeyPressMsg` | Key pressed |
| `tea.KeyReleaseMsg` | Key released (requires keyboard enhancements) |
| `tea.MouseClickMsg` / `tea.MouseReleaseMsg` / `tea.MouseWheelMsg` / `tea.MouseMotionMsg` | Mouse events (requires MouseMode in View) |
| `tea.WindowSizeMsg` | Terminal window size change |
| `tea.FocusMsg` / `tea.BlurMsg` | Terminal gained/lost focus (requires `ReportFocus = true` in View) |
| `tea.QuitMsg` | `tea.Quit()` called |
| `tea.SuspendMsg` | `tea.Suspend()` called |
| `tea.ResumeMsg` | Resumed from suspension |
| `tea.ColorProfileMsg` | Terminal color profile info |
| `tea.EnvMsg` | Environment variables |
| `tea.KeyboardEnhancementsMsg` | Keyboard enhancement capability response |

### Cmd

```go
type Cmd func() Msg
```

`Cmd` is a function that returns a message, executed asynchronously in a goroutine. Full command API: see `references/commands.md`.

### View

In v2, View returns a `tea.View` struct, declaratively controlling terminal features:

```go
func (m model) View() tea.View {
    var v tea.View
    v.SetContent("Hello, World!")                // or tea.NewView("Hello!")
    v.AltScreen = true                            // fullscreen mode
    v.MouseMode = tea.MouseModeCellMotion         // mouse support
    v.ReportFocus = true                          // focus events
    v.WindowTitle = "My App"                      // window title
    v.Cursor = tea.NewCursor(5, 2)                // cursor position
    v.BackgroundColor = color.RGBA{...}           // terminal background
    return v
}
```

Complete `tea.View` field list: see `references/core-concepts.md`.

## Program & Options

```go
p := tea.NewProgram(model,
    tea.WithContext(ctx),             // external context control
    tea.WithInput(inputReader),       // custom input (nil = disabled)
    tea.WithOutput(outputWriter),     // custom output
    tea.WithEnvironment(env),         // custom environment variables
    tea.WithFPS(60),                  // max frame rate (default 60, max 120)
    tea.WithColorProfile(profile),    // force color profile
    tea.WithWindowSize(120, 40),      // initial window size (for testing)
    tea.WithFilter(filterFn),         // event filter
    tea.WithoutSignalHandler(),       // disable signal handling
    tea.WithoutCatchPanics(),         // disable panic catching
    tea.WithoutRenderer(),            // disable renderer (plain CLI mode)
)
```

Program methods:
- `p.Run() (Model, error)` — Start the program, block until exit
- `p.Send(msg)` — Inject a message from outside
- `p.Quit()` — Quit the program from outside
- `p.Kill()` — Terminate immediately
- `p.Println(args...)` / `p.Printf(format, args...)` — Print log above the TUI
- `p.ReleaseTerminal()` / `p.RestoreTerminal()` — Temporarily release/restore terminal (e.g., for external commands)

## Key Handling

Full key code list: see `references/core-concepts.md`.

```go
// Method 1: switch on msg.String() — concise
case tea.KeyPressMsg:
    switch msg.String() {
    case "ctrl+c", "q":
        return m, tea.Quit
    case "enter":
        // confirm
    case "up", "down":
        // navigate
    }

// Method 2: switch on key.Code — more type-safe
case tea.KeyPressMsg:
    switch msg.Code {
    case tea.KeyEnter:
        // confirm
    case tea.KeyRunes:
        switch msg.Text {
        case "y":
            // pressed y
        }
    }

// Method 3: check modifier keys
case tea.KeyPressMsg:
    if msg.Mod&tea.ModCtrl != 0 && msg.Code == 'c' {
        return m, tea.Quit
    }
```

`tea.Key` struct fields: `Text string` (printable characters), `Code rune` (key code), `Mod KeyMod` (modifier keys), `ShiftedCode rune`, `BaseCode rune`, `IsRepeat bool`.

## Mouse Handling

```go
// Enable mouse in View
func (m model) View() tea.View {
    v := tea.NewView("Click me!")
    v.MouseMode = tea.MouseModeCellMotion  // or MouseModeAllMotion
    return v
}

// Handle mouse events in Update
case tea.MouseClickMsg:
    x, y := msg.X, msg.Y
    // handle click

case tea.MouseWheelMsg:
    delta := 1  // scroll up
    if msg.Button == tea.MouseWheelDown {
        delta = -1
    }
```

Mouse modes: `MouseModeNone` (disabled), `MouseModeCellMotion` (click+drag+wheel, recommended), `MouseModeAllMotion` (all movement events).

## Debugging & Logging

```go
// Write to file (stdout is occupied by TUI)
if f, err := tea.LogToFile("debug.log", "debug"); err == nil {
    defer f.Close()
}

// Or use TEA_TRACE environment variable
// TEA_TRACE=trace.log to get Bubble Tea internal event tracing
```

Debugging tips:
- Use `tail -f debug.log` to watch logs in real time
- Delve requires headless mode: `dlv debug --headless --api-version=2 --listen=127.0.0.1:43000 .` then connect from another terminal with `dlv connect`
- `TEA_DEBUG=1` environment variable outputs detailed logs on panic

## Bubbles Component Library

| Component | Purpose | go package |
| :--- | :--- | :--- |
| `spinner` | Loading/wait indicator | `charm.land/bubbles/v2/spinner` |
| `textinput` | Single-line text input | `charm.land/bubbles/v2/textinput` |
| `textarea` | Multi-line text input | `charm.land/bubbles/v2/textarea` |
| `table` | Tabular data display | `charm.land/bubbles/v2/table` |
| `list` | Selectable list | `charm.land/bubbles/v2/list` |
| `viewport` | Scrollable viewport | `charm.land/bubbles/v2/viewport` |
| `paginator` | Page navigation | `charm.land/bubbles/v2/paginator` |
| `progress` | Progress bar | `charm.land/bubbles/v2/progress` |
| `filepicker` | File picker | `charm.land/bubbles/v2/filepicker` |
| `help` | Help/keybinding bar | `charm.land/bubbles/v2/help` |
| `key` | Key binding definitions | `charm.land/bubbles/v2/key` |
| `timer` / `stopwatch` | Timer/stopwatch | `charm.land/bubbles/v2/timer` |

Bubbles components follow the same Update/View pattern as `tea.Model` and can be directly nested in the main Model (note: their `View()` returns `string`, wrap with `tea.NewView()` in the parent's View). Full usage: see `references/components.md`.

## Lip Gloss Styling

```go
import "charm.land/lipgloss/v2"

var style = lipgloss.NewStyle().
    Bold(true).
    Foreground(lipgloss.Color("#FAFAFA")).
    Background(lipgloss.Color("#7D56F4")).
    Padding(1, 2).           // vertical 1, horizontal 2
    Margin(0, 1).            // vertical 0, horizontal 1
    Border(lipgloss.RoundedBorder()).
    BorderForeground(lipgloss.Color("#7D56F4")).
    Width(40).
    Align(lipgloss.Center)

// Usage
fmt.Println(style.Render("Hello, kitty"))

// Layout composition
left := style1.Render("left")
right := style2.Render("right")
fmt.Println(lipgloss.JoinHorizontal(lipgloss.Top, left, right))

// Adaptive color
lipgloss.AdaptiveColor{Light: "#333", Dark: "#FFF"}
```

Full styling API and layout capabilities: see `references/styling-layout.md`.

## Nested Model Pattern

The standard architecture for complex TUI apps nests sub-components in the parent Model:

```go
type mainModel struct {
    state     viewState               // current view
    list      list.Model              // Bubbles component
    textinput textinput.Model
    // custom sub-model
    dialog    *DialogModel
}

func (m mainModel) Init() tea.Cmd {
    // Bubbles components have no Init method; start your own initial Cmds here
    return nil
}

func (m mainModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    // 1. Handle global messages
    switch msg := msg.(type) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "ctrl+c":
            return m, tea.Quit
        case "tab":
            m.state = nextState(m.state)  // switch focus
        }
    case tea.WindowSizeMsg:
        m.width, m.height = msg.Width, msg.Height
    }

    // 2. Forward to focused sub-component
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
    leftPanel := m.list.View()
    rightPanel := m.textinput.View()
    content := lipgloss.JoinHorizontal(lipgloss.Top, leftPanel, rightPanel)
    return tea.NewView(content)
}
```

Detailed architecture patterns (Model Stack, focus management, responsive layout, etc.): see `references/core-concepts.md`.

## Testing

```go
// teatest — official Bubble Tea test library
// Add to go.mod: github.com/charmbracelet/x/exp/teatest
// ⚠️ Experimental API under x/exp — interfaces may change between versions

func TestMyApp(t *testing.T) {
    tm := teatest.NewTestModel(t, initialModel())

    tm.Send(tea.KeyPressMsg{Code: 'q'})

    tm.WaitFinished(t)

    finalModel := tm.FinalModel(t)
}
```

Complete testing solutions (Model unit tests, integration tests, simulated terminal input): see `references/testing.md`.

## Common Errors Quick Reference

| Problem | Cause | Solution |
| :--- | :--- | :--- |
| No output after launch | `p.Run()` not called or Model is nil | Ensure `tea.NewProgram(model).Run()` |
| `View()` returns stale content | Update returned wrong Model | Ensure Update returns the updated Model |
| Keys not responding | Wrong message type matched | Use `case tea.KeyPressMsg` (v2), not `tea.KeyMsg` |
| Mouse not working | MouseMode not enabled in View | Set `v.MouseMode = tea.MouseModeCellMotion` in View |
| Flickering | Heavy string construction in View | Use `strings.Builder`, pre-allocate styles |
| Layout broken after resize | `tea.WindowSizeMsg` not handled | Listen for WindowSizeMsg and update width/height |
| Terminal garbled after panic | Panic caught but terminal not restored | Set `TEA_DEBUG=1` for detailed logs |
| Cmd not executing | Cmd not returned from Update or nil ignored | Use `tea.Batch` to compose multiple Cmds |
| Content disappears after AltScreen exit | Normal behavior | Use `p.Println()` for persistent output above TUI |
| Cannot update UI from background goroutine | Cross-goroutine Model manipulation | Use `p.Send(msg)` to send message back to Update |

## Getting Unstuck: Where to Look First

When references don't cover your case, use these methods to explore Bubble Tea source code.

### Package Path → Filesystem Mapping

```bash
# Method 1: go list (precise, recommended)
BT_DIR=$(go list -m -json charm.land/bubbletea/v2 | grep '"Dir"' | cut -d'"' -f4)

# Method 2: GOMODCACHE
BT_DIR=$(echo $(go env GOMODCACHE)/charm.land/bubbletea/v2@*)
```

Once you have `$BT_DIR`, the package path `charm.land/bubbletea/v2` corresponds to `$BT_DIR/`.

For Bubbles: `BB_DIR=$(go list -m -json charm.land/bubbles/v2 | grep '"Dir"' | cut -d'"' -f4)`

For Lip Gloss: `LG_DIR=$(go list -m -json charm.land/lipgloss/v2 | grep '"Dir"' | cut -d'"' -f4)`

### Finding APIs

```bash
go doc charm.land/bubbletea/v2
go doc charm.land/bubbletea/v2.KeyPressMsg
rg "func.*" "$BT_DIR/"
```

## Reference Documents (by Scenario)

| Scenario | Document | Key Sections |
| :--- | :--- | :--- |
| Don't understand Model/Update/View relationship | `references/core-concepts.md` | Elm Architecture, Lifecycle |
| Need to perform async operations | `references/commands.md` | Cmd/Batch/Sequence/Tick/Every |
| Need to choose/use a component | `references/components.md` | Relevant component section |
| Styling/layout issues | `references/styling-layout.md` | Style chain API, Join, Canvas |
| Layout broken/performance issues/crashes | `references/troubleshooting.md` | Diagnostic table first, then specific sections |
| How to organize code in larger projects | `references/best-practices.md` | Style management, layered KeyMap, dialog stack, layout recalculation |
| How to write tests | `references/testing.md` | Unit tests, teatest integration tests |
| Need navigation between nested Models | `references/core-concepts.md` | Nested Model pattern, Focus management |
| Upgrading from v1 to v2 | Official [UPGRADE_GUIDE_V2.md](https://github.com/charmbracelet/bubbletea/blob/main/UPGRADE_GUIDE_V2.md) | — |

If none of the above references cover your issue, explore the source code using the methods in the "Getting Unstuck" section above. If you still cannot resolve the issue after exploring the source code, **report the specific situation to the user and ask for help — never silently guess.**
