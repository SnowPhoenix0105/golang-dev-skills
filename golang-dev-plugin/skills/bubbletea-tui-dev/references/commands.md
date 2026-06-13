# Commands

## Table of Contents

- [Cmd Basics](#cmd-basics)
- [Batch — Parallel Execution](#batch--parallel-execution)
- [Sequence — Sequential Execution](#sequence--sequential-execution)
- [Tick — Independent Clock Interval](#tick--independent-clock-interval)
- [Every — System Clock Synchronized](#every--system-clock-synchronized)
- [WindowSize — Query Window Size](#windowsize--query-window-size)
- [Quit / Suspend / Interrupt](#quit--suspend--interrupt)
- [Custom Cmd Patterns](#custom-cmd-patterns)
- [Clipboard Operations](#clipboard-operations)
- [Exec — Run External Commands](#exec--run-external-commands)

## Cmd Basics

```go
type Cmd func() Msg
```

A Cmd is a function that returns a message, executed asynchronously in a background goroutine. The result (message) is automatically sent to Update.

**Key rules**:
- Cmds execute in goroutines; do not manipulate the Model directly
- Cmd results come back to Update as Msg to update state
- `nil` Cmd means "no operation" and is ignored by the framework
- Never send messages to other components from within a Cmd — messages naturally flow to Update

## Batch — Parallel Execution

`tea.Batch` executes multiple Cmds concurrently with no ordering guarantees:

```go
func (m model) Init() tea.Cmd {
    return tea.Batch(
        loadUserData,       // run in parallel
        loadConfig,
        startTicker(),
    )
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "enter":
            // Save and send simultaneously
            return m, tea.Batch(saveData, sendRequest)
        }
    }
    return m, nil
}
```

## Sequence — Sequential Execution

`tea.Sequence` executes Cmds one at a time, in order. The next Cmd only runs after the previous Cmd's message is returned:

```go
func (m model) Init() tea.Cmd {
    return tea.Sequence(
        startLoading,       // 1. Show loading
        loadData,           // 2. Load data
        finishLoading,      // 3. Hide loading
    )
}
```

**Note**: `Sequence` is also asynchronous — it doesn't block Update, but triggers Cmds one after another. If a Cmd returns a `BatchMsg` or `sequenceMsg`, it is automatically unwrapped.

## Tick — Independent Clock Interval

`tea.Tick` creates a timer that starts counting from the moment it's invoked, triggering a message after the specified duration:

```go
type tickMsg time.Time

func doTick() tea.Cmd {
    return tea.Tick(time.Second, func(t time.Time) tea.Msg {
        return tickMsg(t)
    })
}

func (m model) Init() tea.Cmd {
    return doTick()
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg.(type) {
    case tickMsg:
        m.count++
        // Continue the next tick
        return m, doTick()
    }
    return m, nil
}
```

**Note**: `Tick` only fires once. To fire continuously, return a new `Tick` Cmd when receiving the tick message.

## Every — System Clock Synchronized

`tea.Every` triggers aligned with the system clock, useful for ticking on round seconds/minutes:

```go
func tickEveryMinute() tea.Cmd {
    return tea.Every(time.Minute, func(t time.Time) tea.Msg {
        return tickMsg(t)
    })
}
```

Difference from `Tick`: if the current time is `12:34:20`, `Every(time.Minute, ...)` fires at `12:35:00` (40 seconds later), while `Tick(time.Minute, ...)` fires at `12:35:20` (60 seconds later).

## WindowSize — Query Window Size

```go
func queryWindowSize() tea.Msg {
    return tea.RequestWindowSize()
}
```

Usually not needed — the framework automatically sends `WindowSizeMsg` at startup and on window resize. However, after restoring the terminal (`RestoreTerminal`), you may want to check if the size changed.

## Quit / Suspend / Interrupt

```go
// Quit the program
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "q", "ctrl+c":
            return m, tea.Quit   // Returns special command, triggers normal exit
        }
}

// Suspend the program (like ctrl+z)
case "ctrl+z":
    return m, tea.Suspend

// Interrupt the program
case "esc":
    return m, tea.Interrupt
```

`tea.Quit`, `tea.Suspend`, `tea.Interrupt` are all functions that return their corresponding Msg: `QuitMsg`, `SuspendMsg`, `InterruptMsg`.

## Custom Cmd Patterns

### Basic Pattern

```go
// Define a function returning a Cmd
func checkServer(url string) tea.Cmd {
    return func() tea.Msg {
        c := &http.Client{Timeout: 10 * time.Second}
        res, err := c.Get(url)
        if err != nil {
            return errMsg{err}
        }
        defer res.Body.Close()
        return statusMsg(res.StatusCode)
    }
}

// Start in Init
func (m model) Init() tea.Cmd {
    return checkServer("https://example.com")
}

// Handle result in Update
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case statusMsg:
        m.status = int(msg)
        return m, tea.Quit
    case errMsg:
        m.err = msg.err
        return m, tea.Quit
    }
    return m, nil
}
```

### Parameterized Cmd

```go
func fetchData(id int) tea.Cmd {
    return func() tea.Msg {
        data, err := api.GetItem(id)
        if err != nil {
            return errMsg{err}
        }
        return dataLoadedMsg{data}
    }
}
```

### Cmd Composition Tips

```go
// Dynamically decide which Cmd to execute in Update
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmds []tea.Cmd

    switch msg := msg.(type) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "enter":
            if m.needsSave {
                cmds = append(cmds, saveData(m.data))
            }
            cmds = append(cmds, tea.Quit)
        }
    case dataLoadedMsg:
        m.data = msg.data
        // Automatically start the next operation after loading
        cmds = append(cmds, processData(m.data))
    }

    return m, tea.Batch(cmds...)
}
```

## Clipboard Operations

Bubble Tea supports native system clipboard operations:

```go
// Read clipboard — ReadClipboard() returns Msg, wrap as Cmd
func readClipboard() tea.Cmd {
    return func() tea.Msg {
        return tea.ReadClipboard()
    }
}

// Write clipboard — SetClipboard returns Cmd directly
func writeClipboard(text string) tea.Cmd {
    return tea.SetClipboard(text)
}

// Read primary selection (Linux primary selection)
func readPrimaryClipboard() tea.Cmd {
    return func() tea.Msg {
        return tea.ReadPrimaryClipboard()
    }
}

// Write primary selection
func writePrimaryClipboard(text string) tea.Cmd {
    return tea.SetPrimaryClipboard(text)
}

// Handle clipboard content in Update
case tea.ClipboardMsg:
    m.clipboardContent = string(msg)
```

## Exec — Run External Commands

```go
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "e":
            cmd := exec.Command("vim", "file.txt")
            return m, tea.ExecProcess(cmd, func(err error) tea.Msg {
                return editorFinishedMsg{err}
            })
        }
    case editorFinishedMsg:
        if msg.err != nil {
            m.err = msg.err
        }
        return m, nil
}
```

`tea.ExecProcess` temporarily releases the terminal, runs the external command, then restores the terminal and sends the result message to Update.
