# Testing

## Table of Contents

- [Model Unit Tests](#model-unit-tests)
- [teatest Integration Tests](#teatest-integration-tests)
- [Simulating Terminal Input](#simulating-terminal-input)
- [Testing Cmd Execution](#testing-cmd-execution)
- [CI Testing](#ci-testing)

## Model Unit Tests

The Model's Update and View are pure (or near-pure) functions, suitable for direct unit testing:

```go
func TestModelUpdate(t *testing.T) {
    m := initialModel()

    // Send a key message
    newModel, cmd := m.Update(tea.KeyPressMsg{
        Code: 'q',
    })

    // Verify Model updated correctly
    if newModel.(model).count != 0 {
        t.Errorf("expected count 0, got %d", newModel.(model).count)
    }

    // Verify cmd is not nil (e.g., Quit)
    if cmd == nil {
        t.Error("expected quit cmd, got nil")
    }
}

func TestModelView(t *testing.T) {
    m := model{count: 5}
    v := m.View()

    if !strings.Contains(v.Content, "Count: 5") {
        t.Errorf("view doesn't contain count: %s", v.Content)
    }
}
```

### Testing Message Handling

```go
func TestHandleWindowResize(t *testing.T) {
    m := initialModel()

    m, _ = m.Update(tea.WindowSizeMsg{Width: 120, Height: 40})

    finalModel := m.(model)
    if finalModel.width != 120 {
        t.Errorf("expected width 120, got %d", finalModel.width)
    }
    if finalModel.height != 40 {
        t.Errorf("expected height 40, got %d", finalModel.height)
    }
}
```

### Testing Custom Messages

```go
func TestDataLoadedHandler(t *testing.T) {
    m := initialModel()

    m, cmd := m.Update(dataLoadedMsg{
        items: []string{"a", "b", "c"},
    })

    finalModel := m.(model)
    if len(finalModel.items) != 3 {
        t.Errorf("expected 3 items, got %d", len(finalModel.items))
    }
    // Verify follow-up cmd was triggered
    if cmd == nil {
        t.Error("expected follow-up cmd")
    }
}
```

## teatest Integration Tests

`teatest` is the official Bubble Tea integration test library (`github.com/charmbracelet/x/exp/teatest`), supporting full Program lifecycle testing.

### Basic Usage

```go
import "github.com/charmbracelet/x/exp/teatest"

func TestApp(t *testing.T) {
    // Create test model
    tm := teatest.NewTestModel(t, initialModel(),
        teatest.WithDefaultTerminalSize(80, 24),
    )

    // Send key press
    tm.Send(tea.KeyPressMsg{Code: 'q'})

    // Wait for program exit
    tm.WaitFinished(t)

    // Verify final state
    finalModel := tm.FinalModel(t)
    m := finalModel.(model)
    if m.exitCode != 0 {
        t.Errorf("expected exit code 0, got %d", m.exitCode)
    }
}
```

### Verifying Render Output

```go
func TestRendering(t *testing.T) {
    tm := teatest.NewTestModel(t, initialModel())

    // Wait for a specific frame (containing specific content)
    tm.WaitForFrame(t, func(frame string) bool {
        return strings.Contains(frame, "Count: 5")
    })

    // Get last frame
    lastFrame := tm.LastFrame(t)
    if !strings.Contains(lastFrame, "Press q to quit") {
        t.Error("help text not shown")
    }
}
```

### Testing Interaction Sequences

```go
func TestNavigationFlow(t *testing.T) {
    tm := teatest.NewTestModel(t, initialModel())

    // Press arrow keys to navigate
    tm.Send(tea.KeyPressMsg{Code: tea.KeyDown})
    tm.Send(tea.KeyPressMsg{Code: tea.KeyDown})

    // Press Enter to select
    tm.Send(tea.KeyPressMsg{Code: tea.KeyEnter})

    // Wait for render update
    tm.WaitForFrame(t, func(frame string) bool {
        return strings.Contains(frame, "Selected: item 2")
    })

    // Quit
    tm.Send(tea.KeyPressMsg{Code: 'q'})
    tm.WaitFinished(t)
}
```

### Testing Mouse Interaction

```go
func TestMouseClick(t *testing.T) {
    m := model{items: []string{"a", "b", "c"}}
    tm := teatest.NewTestModel(t, m)

    // Simulate mouse click
    tm.Send(tea.MouseClickMsg{
        X: 5, Y: 2,
        Button: tea.MouseLeft,
    })

    tm.WaitFinished(t)
}
```

### teatest Common Methods

```go
tm.Send(msg tea.Msg)               // Send a message
tm.WaitFinished(t)                  // Wait for Program exit
tm.FinalModel(t) tea.Model         // Get final Model
tm.LastFrame(t) string             // Get last rendered frame
tm.WaitForFrame(t, match func(string) bool)  // Wait for a specific frame
```

## Simulating Terminal Input

If you don't want to depend on teatest, you can test Cmd logic directly:

```go
func TestLoadDataCmd(t *testing.T) {
    // Execute Cmd, get returned message
    cmd := loadData()
    msg := cmd()

    // Verify message type and content
    dataMsg, ok := msg.(dataLoadedMsg)
    if !ok {
        t.Fatalf("expected dataLoadedMsg, got %T", msg)
    }
    if len(dataMsg.items) != expectedCount {
        t.Errorf("expected %d items, got %d", expectedCount, len(dataMsg.items))
    }
}
```

### Testing Async Cmd with Mocks

```go
func TestFetchDataCmd(t *testing.T) {
    // Use mock client
    mockClient := &MockAPIClient{
        data: []Item{{ID: 1, Name: "test"}},
    }

    cmd := fetchDataWithClient(mockClient, 1)
    msg := cmd()

    switch msg := msg.(type) {
    case errMsg:
        t.Fatalf("unexpected error: %v", msg.err)
    case dataLoadedMsg:
        if len(msg.data) != 1 {
            t.Errorf("expected 1 item, got %d", len(msg.data))
        }
    }
}
```

## CI Testing

### Non-TTY Mode Testing

```go
func TestNonTTY(t *testing.T) {
    // Bubble Tea apps should work in non-TTY environments
    // Use WithoutRenderer option to simulate
    p := tea.NewProgram(model{},
        tea.WithoutRenderer(),
        tea.WithInput(nil),
    )
    finalModel, err := p.Run()
    if err != nil {
        t.Fatalf("program failed: %v", err)
    }
    // Verify final state
    _ = finalModel.(model)
}
```

### GitHub Actions CI Configuration

```yaml
# .github/workflows/test.yml
- name: Run tests
  run: go test ./...
  env:
    TERM: xterm-256color
    NO_COLOR: "1"
```

### CI Considerations

- CI typically lacks a real TTY; may need special handling for TUI execution
- Use `tea.WithOutput(&buf)` and `tea.WithInput(strings.NewReader("q\n"))` to control I/O
- Set `TERM=xterm-256color` to ensure color profile detection
- `NO_COLOR=1` prevents color output from interfering with test assertions
