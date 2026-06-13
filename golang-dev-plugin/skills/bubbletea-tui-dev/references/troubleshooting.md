# Troubleshooting Guide

## Table of Contents

- [Quick Diagnostic Table](#quick-diagnostic-table)
- [Program Startup Issues](#program-startup-issues)
- [Rendering Issues](#rendering-issues)
- [Input Issues](#input-issues)
- [Performance Issues](#performance-issues)
- [Terminal Compatibility](#terminal-compatibility)
- [Debugging Tips](#debugging-tips)

## Quick Diagnostic Table

| Symptom | Most Likely Cause | Check First |
| :--- | :--- | :--- |
| No output, program stuck | Quit not returned in Update | Check if exit key is properly matched |
| Garbled/overlapping output | View renders entire screen but content length varies | Ensure View returns complete content, clear old content |
| Keys don't respond | Wrongly matched v1 `tea.KeyMsg` | Use `tea.KeyPressMsg` instead |
| Mouse doesn't work | MouseMode not enabled in View | Set `v.MouseMode` in View |
| Flickering | Heavy string concatenation | Use strings.Builder, pre-allocate styles |
| Layout broken after resize | WindowSizeMsg not handled | Add WindowSizeMsg handling |
| Terminal garbled after panic | Panic in Cmd goroutine | Set TEA_DEBUG=1 for stack trace |
| Cmd not executing | Returned nil but expected behavior | Check if Cmd is correctly returned |
| Sub-component not updating | Message not forwarded to sub-component | Check forwarding logic in Update |
| Colors incorrect | Terminal doesn't support TrueColor | Fall back to ANSI256 or adaptive colors |

## Program Startup Issues

### Program Exits Immediately

```go
// ❌ Wrong: Init returns Quit
func (m model) Init() tea.Cmd {
    return tea.Quit  // Returning Quit in Init → program exits immediately
}

// ✅ Correct: Init should return initial operations or nil
func (m model) Init() tea.Cmd {
    return nil
}
```

### "panic: InitialModel cannot be nil"

```go
// ❌ Wrong
p := tea.NewProgram(nil)

// ✅ Correct
p := tea.NewProgram(initialModel())
```

### Non-TTY Environment

```go
// Check if running in a TTY
if !term.IsTerminal(os.Stdout.Fd()) {
    fmt.Println("Not a terminal, falling back to plain mode")
    // Run with WithoutRenderer
    p := tea.NewProgram(model, tea.WithoutRenderer())
    p.Run()
}
```

## Rendering Issues

### Flickering

Most common cause: creating new styles or strings in every View call:

```go
// ❌ Creating styles every call
func (m model) View() tea.View {
    style := lipgloss.NewStyle().Bold(true).Padding(1)
    return tea.NewView(style.Render("content"))
}

// ✅ Pre-allocate globals
var contentStyle = lipgloss.NewStyle().Bold(true).Padding(1)

func (m model) View() tea.View {
    return tea.NewView(contentStyle.Render("content"))
}
```

### Content Not Updating

Check if Update returns the correct Model:

```go
// ⚠️ Value receiver works (the modified copy is returned), but large structs incur copy overhead
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    m.count++
    return m, nil  // Returns the modified copy; changes take effect
}

// ✅ Recommended: use pointer receiver to avoid copying the entire struct on every Update
func (m *model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    m.count++
    return m, nil
}
```

### Partial Content Disappears

View should render the **entire** screen. If content line count changes, ensure new content covers every line of the old content:

```go
// ❌ May leave residue
func (m model) View() tea.View {
    return tea.NewView("Short content")  // If previously 20 lines, remaining lines not cleared
}

// ✅ Fill to full height or pad with blank lines
func (m model) View() tea.View {
    lines := []string{"Header", "Content", "Footer"}
    for len(lines) < m.height {
        lines = append(lines, "")  // Pad with empty lines
    }
    return tea.NewView(strings.Join(lines, "\n"))
}
```

### AltScreen Content Disappears

Content disappearing after exiting AltScreen is normal behavior. To preserve content:

```go
// Use Println to print above the TUI (persistent output)
p.Println("Important output that stays after exit")
```

## Input Issues

### Keys Don't Respond

Key message types changed in v2:

```go
// ❌ v1 code (doesn't work in v2)
case tea.KeyMsg:
    switch msg.Type {
    case tea.KeyRunes:  // v1 API
    }

// ✅ v2 code
case tea.KeyPressMsg:
    switch msg.String() {
    case "enter":
    // ...
    }
```

### Space Key Matching

In v2, space key's `String()` returns `"space"`, not `" "`:

```go
// ❌ v1
case " ":

// ✅ v2
case "space":
```

### Modifier Key Combinations

```go
case tea.KeyPressMsg:
    // Check ctrl+c
    if msg.Mod&tea.ModCtrl != 0 && msg.Code == 'c' {
        return m, tea.Quit
    }
    // Check ctrl+s
    if msg.Mod&tea.ModCtrl != 0 && msg.Code == 's' {
        // save
    }
```

### Some Terminal Keys Not Recognized

Some terminals don't support all keys. Always handle fallbacks for common keys:

```go
// Support multiple exit methods
case tea.KeyPressMsg:
    switch msg.String() {
    case "ctrl+c", "ctrl+d", "q", "esc":
        return m, tea.Quit
    }
```

## Performance Issues

### Update Too Slow

Update taking over 1ms causes input latency:

```go
// ❌ Blocking Update
func (m *model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    data := httpGet("https://api.example.com")  // Blocks!
    m.data = data
    return m, nil
}

// ✅ Wrap time-consuming operations as Cmd
func loadData() tea.Cmd {
    return func() tea.Msg {
        data, err := httpGet("https://api.example.com")
        if err != nil {
            return errMsg{err}
        }
        return dataLoadedMsg{data}
    }
}
```

### View String Concatenation

```go
// ❌ Repeated += in loop
func (m model) View() tea.View {
    s := ""
    for _, item := range m.items {
        s += renderItem(item) + "\n"  // Frequent memory allocation
    }
    return tea.NewView(s)
}

// ✅ strings.Builder
func (m model) View() tea.View {
    var b strings.Builder
    b.Grow(len(m.items) * 50)
    for _, item := range m.items {
        b.WriteString(renderItem(item))
        b.WriteByte('\n')
    }
    return tea.NewView(b.String())
}
```

### Frame Rate Too High

Default 60 FPS. Adjust with `WithFPS`:

```go
p := tea.NewProgram(model, tea.WithFPS(30))  // Lower to 30 FPS
```

### Many List Items

With over 1000 items, use virtual scrolling or limit render range:

```go
// Only render visible items
func (m model) renderList() string {
    start := m.scrollOffset
    end := min(start+m.height, len(m.items))
    var b strings.Builder
    for i := start; i < end; i++ {
        b.WriteString(renderItem(m.items[i]))
        b.WriteByte('\n')
    }
    return b.String()
}
```

## Terminal Compatibility

### Color Degradation

```go
// Use adaptive colors for dark/light background support
var color = lipgloss.AdaptiveColor{Light: "#333", Dark: "#FFF"}

// Pure hex colors may display incorrectly on non-TrueColor terminals
// Use fallback colors where possible
var color = lipgloss.Color("#7D56F4")  // Will degrade on unsupported terminals
```

### Minimum Terminal Size

```go
const minWidth, minHeight = 60, 20

func (m model) View() tea.View {
    if m.width < minWidth || m.height < minHeight {
        return tea.NewView(
            "Terminal too small. " +
            fmt.Sprintf("Current: %dx%d, Required: %dx%d",
                m.width, m.height, minWidth, minHeight))
    }
    return tea.NewView(m.renderContent())
}
```

### SSH Environments

```go
// Detect SSH via environment variables
if _, ok := os.LookupEnv("SSH_TTY"); ok {
    // SSH session special handling:
    // - Lower frame rate
    p := tea.NewProgram(model, tea.WithFPS(15))
    // - Avoid certain terminal features
    // - Use WithEnvironment to pass remote environment variables
}
```

### Platform Differences

```go
// Windows special handling
import "runtime"

if runtime.GOOS == "windows" {
    // Enable Windows terminal virtual processing
}
```

## Debugging Tips

### Logging

```go
// Set up log file
f, err := tea.LogToFile("debug.log", "debug")
if err != nil {
    log.Fatal(err)
}
defer f.Close()

// Log in code
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    log.Printf("Received message: %T %+v", msg, msg)  // Writes to debug.log
    // ...
}
```

Use another terminal window with `tail -f debug.log` for real-time viewing.

### TEA_TRACE Environment Variable

```bash
# Get Bubble Tea internal event tracing
TEA_TRACE=trace.log go run .
# View all input/render events
tail -f trace.log
```

### Panic Debugging

```bash
# TEA_DEBUG=1 outputs detailed log on panic
TEA_DEBUG=1 go run .
# Generates bubbletea-panic-<timestamp>.log
```

### Delve Debugger

```bash
# Start headless delve (TUI occupies stdin/stdout)
dlv debug --headless --api-version=2 --listen=127.0.0.1:43000 .

# Connect from another terminal
dlv connect 127.0.0.1:43000

# Set breakpoint
(dlv) break main.go:45
(dlv) continue
```

### Known Issues

1. **Mouse doesn't work in tmux** — Ensure tmux is configured with `set -g mouse on`
2. **Focus events not available in tmux** — Requires tmux focus-events configuration
3. **Certain colors display incorrectly in iTerm2** — Check iTerm2's minimum contrast setting
4. **Different key codes in Kitty** — Enable KeyboardEnhancements for precise key codes
