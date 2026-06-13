# Core Concepts & Architecture

## Table of Contents

- [The Elm Architecture & Bubble Tea](#the-elm-architecture--bubble-tea)
- [Model Interface Details](#model-interface-details)
- [View Struct Field Reference](#view-struct-field-reference)
- [Message System](#message-system)
- [Program Lifecycle](#program-lifecycle)
- [Nested Model Patterns](#nested-model-patterns)
- [Focus Management](#focus-management)
- [Responsive Layout](#responsive-layout)

## The Elm Architecture & Bubble Tea

Bubble Tea strictly follows the Elm Architecture's unidirectional data flow:

```
                    ┌──────────┐
         ┌─────────→│   Init   │
         │          └────┬─────┘
         │               │ Cmd (optional)
         │               ↓
         │          ┌──────────┐        ┌──────────┐
  View ←─┤          │  Update  │←───────│   Cmd    │
  (render)│         └──────────┘  Msg   │ (async IO)│
         │               │               └──────────┘
         │               │ Cmd (optional)
         │               ↓
         │          ┌──────────┐
         └──────────│   View   │
                    └──────────┘
```

**Key Principles**:
1. **Unidirectional data flow** — Data only flows one way: Init → Update → View → Render
2. **Message-driven** — All state changes are triggered by Msg in Update
3. **Update must be fast** — Return within <1ms. Time-consuming I/O is wrapped in Cmd
4. **View is a pure function** — Given the same Model, View should produce the same output

## Model Interface Details

```go
type Model interface {
    Init() Cmd
    Update(Msg) (Model, Cmd)
    View() View
}
```

### Init() Cmd

- Called once when `Program.Run()` starts
- Returns initial async operations: loading data, starting timers, querying terminal capabilities
- Return `nil` when no initial I/O is needed
- Common patterns:

```go
func (m model) Init() tea.Cmd {
    return tea.Batch(
        loadData,           // load initial data
        tickEvery(),        // start periodic tick
        queryTerminalCap,   // query terminal capabilities
    )
}
```

### Update(Msg) (Model, Cmd)

- Called every time a message is received
- Returns updated Model and optional new Cmd
- **Must return fast**: time-consuming operations (network requests, file I/O, heavy computation) should be wrapped in Cmd
- Typical pattern: handle global messages first (quit, window size), then forward to the currently focused sub-component

```go
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    // Step 1: Handle global messages
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

    // Step 2: Forward based on current state/focus
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

- Automatically called after every Update
- Returns a `tea.View` struct describing UI content and terminal features
- v2 is declarative: terminal state is no longer controlled imperatively via program options
- View should render the **entire** screen; Bubble Tea handles diff and repaint optimization

## View Struct Field Reference

```go
type View struct {
    Content string              // UI text content (styles encoded as ANSI escape sequences)
    AltScreen bool              // Alternate screen buffer (fullscreen mode)
    MouseMode MouseMode         // MouseModeNone / MouseModeCellMotion / MouseModeAllMotion
    ReportFocus bool            // Enable focus reporting (receive FocusMsg/BlurMsg)
    DisableBracketedPasteMode bool  // Disable bracketed paste mode
    WindowTitle string          // Terminal window title
    Cursor *Cursor              // Cursor state (position, style, color, blink)
    BackgroundColor color.Color // Terminal background color
    ForegroundColor color.Color // Terminal foreground color
    ProgressBar *ProgressBar    // Native progress bar (supported by some terminals)
    KeyboardEnhancements KeyboardEnhancements  // Requested keyboard enhancement features
    OnMouse func(msg MouseMsg) Cmd  // View-level mouse handler (depends on last render)
}
```

### MouseMode

```go
const (
    MouseModeNone       MouseMode = iota  // Mouse disabled
    MouseModeCellMotion                    // Click+release+wheel+drag (recommended, good compatibility)
    MouseModeAllMotion                     // All events including motion without buttons
)
```

### Cursor

```go
cursor := &tea.Cursor{
    Position: tea.Position{X: 5, Y: 2},  // Relative to frame top-left
    Color:    color.RGBA{255, 0, 0, 255},
    Shape:    tea.CursorBlock,           // CursorBlock / CursorUnderline / CursorBar
    Blink:    true,
}
```

### ProgressBar

```go
// Set in View
v.ProgressBar = tea.NewProgressBar(tea.ProgressBarDefault, 50)       // 50%
v.ProgressBar = tea.NewProgressBar(tea.ProgressBarIndeterminate, 0)   // indeterminate
v.ProgressBar = tea.NewProgressBar(tea.ProgressBarError, 0)           // error state
// States: ProgressBarNone / ProgressBarDefault / ProgressBarError
//         ProgressBarIndeterminate / ProgressBarWarning
```

### KeyboardEnhancements

```go
v.KeyboardEnhancements = tea.KeyboardEnhancements{
    ReportEventTypes:      true,  // Receive KeyReleaseMsg and Key.IsRepeat
    ReportAlternateKeys:   false, // Report alternate key codes
    ReportAllKeysAsEscapeCodes: false,  // Report all keys as escape sequences
    ReportAssociatedText:  false, // Report associated text
}
```

Terminal-supported capabilities are returned via `tea.KeyboardEnhancementsMsg`.

## Message System

### Framework-Auto-Sent Messages

| Message Type | Trigger Condition | Key Fields |
| :--- | :--- | :--- |
| `tea.KeyPressMsg` | Key pressed | See [Key Struct](#key-struct) |
| `tea.KeyReleaseMsg` | Key released | Requires `ReportEventTypes = true`, same as above |
| `tea.MouseClickMsg` | Mouse click | `X, Y int; Button MouseButton; Mod KeyMod` |
| `tea.MouseReleaseMsg` | Mouse release | Same |
| `tea.MouseWheelMsg` | Mouse wheel | Same; `Button` is `MouseWheelUp/Down/Left/Right` |
| `tea.MouseMotionMsg` | Mouse movement | Same |
| `tea.WindowSizeMsg` | Terminal window resize | `Width int; Height int` |
| `tea.FocusMsg` | Terminal gained focus | Requires `ReportFocus = true` |
| `tea.BlurMsg` | Terminal lost focus | Requires `ReportFocus = true` |
| `tea.ColorProfileMsg` | Color profile info | `Profile colorprofile.Profile` |
| `tea.EnvMsg` | Environment variables | `[]string` type |
| `tea.KeyboardEnhancementsMsg` | Keyboard enhancement response | `Flags int` |
| `tea.QuitMsg` | `tea.Quit()` called | — |
| `tea.SuspendMsg` | `tea.Suspend()` called | — |
| `tea.ResumeMsg` | Resumed from suspension | — |

### Key Struct

```go
type Key struct {
    Text        string  // Printable characters (e.g., "a", "A", "1", "!"). Empty for special keys.
    Mod         KeyMod  // Modifier: ModCtrl, ModAlt, ModShift, ModMeta, ModSuper, ModHyper
    Code        rune    // Key code: special keys use constants (KeyEnter, KeyTab), printable use rune
    ShiftedCode rune    // Actual shifted character (Kitty protocol / Windows only)
    BaseCode    rune    // PC-101 base layout key code (Kitty protocol / Windows only)
    IsRepeat    bool    // Key repeat event (Kitty protocol / Windows only)
}
```

### Common Key Code Constants

```go
// Special keys
tea.KeyUp, tea.KeyDown, tea.KeyRight, tea.KeyLeft
tea.KeyEnter, tea.KeyReturn, tea.KeyTab
tea.KeyEscape, tea.KeyEsc
tea.KeyBackspace, tea.KeySpace
tea.KeyDelete, tea.KeyInsert
tea.KeyHome, tea.KeyEnd, tea.KeyPgUp, tea.KeyPgDown

// Function keys
tea.KeyF1 ... tea.KeyF63

// Modifier keys
tea.KeyLeftCtrl, tea.KeyRightCtrl, tea.KeyLeftAlt, tea.KeyRightAlt
tea.KeyLeftShift, tea.KeyRightShift, tea.KeyLeftSuper, tea.KeyRightSuper

// Media keys
tea.KeyMediaPlay, tea.KeyMediaPause, tea.KeyMediaNext, tea.KeyMediaPrev
tea.KeyLowerVol, tea.KeyRaiseVol, tea.KeyMute

// Modifier constants
tea.ModCtrl, tea.ModAlt, tea.ModShift, tea.ModMeta, tea.ModSuper, tea.ModHyper
```

### Three Ways to Match Keys

```go
switch msg := msg.(type) {
case tea.KeyPressMsg:
    // Method 1: String() — most common
    switch msg.String() {
    case "ctrl+c", "q":
        return m, tea.Quit
    case "enter":
        // ...
    case "space":
        // Note: in v2, space key String() returns "space", not " "
    }

    // Method 2: Code — type-safe
    switch msg.Code {
    case tea.KeyEnter:
        // ...
    default:
        switch msg.Text {
        case "y", "Y":
            // ...
        }
    }

    // Method 3: Check modifiers
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

### Custom Messages

Any type can serve as a Msg:

```go
type tickMsg time.Time

type statusMsg int

type errMsg struct{ err error }

func (e errMsg) Error() string { return e.err.Error() }

type DataLoadedMsg struct {
    Items []Item
}
```

## Program Lifecycle

### Creating and Running

```go
p := tea.NewProgram(initialModel, options...)
finalModel, err := p.Run()
```

### Lifecycle Event Order

1. `NewProgram` — Creates Program instance
2. `p.Run()` — Initializes terminal (raw mode, hide cursor, etc.)
3. Sends initial messages: `WindowSizeMsg` → `ColorProfileMsg` → `EnvMsg`
4. Calls `model.Init()` for initial Cmd
5. Renders initial View
6. **Event loop**: Wait for Msg → Update → View → Render
7. Exits loop on `QuitMsg` or `InterruptMsg`
8. Renders final frame, restores terminal state
9. Returns finalModel and error

### Exit Methods

```go
// Method 1: Return tea.Quit from Update
return m, tea.Quit

// Method 2: Call from outside
p.Quit()

// Method 3: System signals
// SIGINT  → InterruptMsg → ErrInterrupted
// SIGTERM → QuitMsg → normal exit

// Method 4: Context cancellation
ctx, cancel := context.WithCancel(context.Background())
p := tea.NewProgram(model, tea.WithContext(ctx))
// In another goroutine: cancel()
// p.Run() returns ErrProgramKilled
```

### External Message Injection

```go
p := tea.NewProgram(model)

go func() {
    // Inject message after background operation completes
    result := doSomething()
    p.Send(resultMsg{data: result})
}()

p.Run()
```

### ReleaseTerminal / RestoreTerminal

Use when you need to temporarily restore the terminal (e.g., to run an external command):

```go
// Release terminal
if err := p.ReleaseTerminal(); err != nil {
    return err
}

// Run external command (using standard stdin/stdout)
execCmd := exec.Command("vim", "file.txt")
execCmd.Stdin = os.Stdin
execCmd.Stdout = os.Stdout
execCmd.Run()

// Restore terminal
if err := p.RestoreTerminal(); err != nil {
    return err
}
```

## Nested Model Patterns

### Pattern 1: Direct Nesting (component in current layout)

Most common pattern, suitable when sub-components are always part of the current view:

```go
type mainModel struct {
    list      list.Model       // Bubbles list component
    textinput textinput.Model  // Bubbles textinput component
    width     int
    height    int
}

func (m mainModel) Init() tea.Cmd {
    // Bubbles components have no Init method; return nil or your own initial Cmds
    return nil
}

func (m mainModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmds []tea.Cmd

    switch msg := msg.(type) {
    case tea.WindowSizeMsg:
        m.width = msg.Width
        m.height = msg.Height
        // Adjust sub-component sizes
        m.list.SetSize(msg.Width/2, msg.Height)
        m.textinput.SetWidth(msg.Width / 2)
    }

    // Forward based on focus
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

### Pattern 2: Page Switching (sub-Model takes full screen)

Suitable for switching between independent "pages":

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
    // Global navigation
    switch msg := msg.(type) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "ctrl+c":
            return m, tea.Quit
        case "esc":
            m.state = pageList  // always return to list
        }
    }

    // Forward based on current page
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

### Pattern 3: Dialog/Overlay

```go
type mainModel struct {
    base    BaseModel
    dialog  *DialogModel  // nil means no dialog
}

func (m mainModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    // Dialog takes priority
    if m.dialog != nil {
        var cmd tea.Cmd
        m.dialog, cmd = m.dialog.Update(msg)
        // Check if dialog requested close
        if m.dialog.Done {
            m.dialog = nil
        }
        return m, cmd
    }

    // Normal flow
    var cmd tea.Cmd
    m.base, cmd = m.base.Update(msg)
    return m, cmd
}
```

## Focus Management

### Focus State Machine

```go
type FocusState int

const (
    FocusNone FocusState = iota
    FocusList
    FocusInput
    FocusDetail
)

// Tab key rotates focus
case tea.KeyPressMsg:
    switch msg.String() {
    case "tab":
        m.focus = (m.focus + 1) % numFocusStates
    case "shift+tab":
        m.focus = (m.focus - 1 + numFocusStates) % numFocusStates
    }
```

### Focus Restore Pattern (dialog scenario)

```go
// Save focus before opening dialog
func (m *model) openDialog() {
    m.previousFocus = m.focus
    m.dialog = newDialog()
    m.focus = FocusDialog
}

// Restore after closing
func (m *model) closeDialog() {
    m.dialog = nil
    m.focus = m.previousFocus
}
```

## Responsive Layout

### Handling Window Size Changes

```go
case tea.WindowSizeMsg:
    m.width = msg.Width
    m.height = msg.Height

    // Adaptive layout breakpoints
    if m.width < 80 {
        m.layout = compactLayout
    } else if m.width < 120 {
        m.layout = normalLayout
    } else {
        m.layout = wideLayout
    }

    // Notify sub-components
    m.list.SetSize(m.listWidth(), m.height)
```

### Minimum Terminal Size Check

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
