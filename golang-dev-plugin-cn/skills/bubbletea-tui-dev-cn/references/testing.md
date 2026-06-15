# 测试

## 目录

- [Model 单元测试](#model-单元测试)
- [teatest 集成测试](#teatest-集成测试)
- [模拟终端输入](#模拟终端输入)
- [测试 Cmd 执行](#测试-cmd-执行)
- [CI 测试](#ci-测试)

## Model 单元测试

Model 的 Update 和 View 是纯函数（或接近纯函数），适合直接单元测试：

```go
func TestModelUpdate(t *testing.T) {
    m := initialModel()

    // 发送按键消息
    newModel, cmd := m.Update(tea.KeyPressMsg{
        Code: 'q',
    })

    // 验证 Model 正确更新
    if newModel.(model).count != 0 {
        t.Errorf("expected count 0, got %d", newModel.(model).count)
    }

    // 验证 cmd 不为 nil（如 Quit）
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

### 测试消息处理

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

### 测试自定义消息

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
    // 验证触发后续 cmd
    if cmd == nil {
        t.Error("expected follow-up cmd")
    }
}
```

## teatest 集成测试

`teatest` 是 Bubble Tea 生态的集成测试库（`github.com/charmbracelet/x/exp/teatest`），支持完整的 Program 生命周期测试。

> ⚠️ **实验性 API**：`teatest` 位于 `x/exp` 下，API 可能在版本间变化，生产环境中需注意兼容性。

### 基础用法

```go
import "github.com/charmbracelet/x/exp/teatest"

func TestApp(t *testing.T) {
    // 创建测试模型
    tm := teatest.NewTestModel(t, initialModel(),
        teatest.WithDefaultTerminalSize(80, 24),
    )

    // 发送按键
    tm.Send(tea.KeyPressMsg{Code: 'q'})

    // 等待程序退出
    tm.WaitFinished(t)

    // 验证最终状态
    finalModel := tm.FinalModel(t)
    m := finalModel.(model)
    if m.exitCode != 0 {
        t.Errorf("expected exit code 0, got %d", m.exitCode)
    }
}
```

### 验证渲染输出

```go
func TestRendering(t *testing.T) {
    tm := teatest.NewTestModel(t, initialModel())

    // 等待特定帧（包含特定内容）
    tm.WaitForFrame(t, func(frame string) bool {
        return strings.Contains(frame, "Count: 5")
    })

    // 获取最后一帧
    lastFrame := tm.LastFrame(t)
    if !strings.Contains(lastFrame, "Press q to quit") {
        t.Error("help text not shown")
    }
}
```

### 测试交互序列

```go
func TestNavigationFlow(t *testing.T) {
    tm := teatest.NewTestModel(t, initialModel())

    // 按下箭头键导航
    tm.Send(tea.KeyPressMsg{Code: tea.KeyDown})
    tm.Send(tea.KeyPressMsg{Code: tea.KeyDown})

    // 按 Enter 选择
    tm.Send(tea.KeyPressMsg{Code: tea.KeyEnter})

    // 等待渲染更新
    tm.WaitForFrame(t, func(frame string) bool {
        return strings.Contains(frame, "Selected: item 2")
    })

    // 退出
    tm.Send(tea.KeyPressMsg{Code: 'q'})
    tm.WaitFinished(t)
}
```

### 测试 Mouse 交互

```go
func TestMouseClick(t *testing.T) {
    m := model{items: []string{"a", "b", "c"}}
    tm := teatest.NewTestModel(t, m)

    // 模拟鼠标点击
    tm.Send(tea.MouseClickMsg{
        X: 5, Y: 2,
        Button: tea.MouseLeft,
    })

    tm.WaitFinished(t)
}
```

### teatest 常用方法

```go
tm.Send(msg tea.Msg)               // 发送消息
tm.WaitFinished(t)                  // 等待 Program 退出
tm.FinalModel(t) tea.Model         // 获取最终 Model
tm.LastFrame(t) string             // 获取最后一帧渲染输出
tm.WaitForFrame(t, match func(string) bool)  // 等待特定帧
```

## 模拟终端输入

如果不想依赖 teatest，可以直接测试 Cmd 逻辑：

```go
func TestLoadDataCmd(t *testing.T) {
    // 执行 Cmd，获取返回的消息
    cmd := loadData()
    msg := cmd()

    // 验证消息类型和内容
    dataMsg, ok := msg.(dataLoadedMsg)
    if !ok {
        t.Fatalf("expected dataLoadedMsg, got %T", msg)
    }
    if len(dataMsg.items) != expectedCount {
        t.Errorf("expected %d items, got %d", expectedCount, len(dataMsg.items))
    }
}
```

### 使用 mock 测试异步 Cmd

```go
func TestFetchDataCmd(t *testing.T) {
    // 使用 mock client
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

## CI 测试

### 非 TTY 模式测试

```go
func TestNonTTY(t *testing.T) {
    // Bubble Tea 应用应在非 TTY 下也能正常工作
    // 使用 WithoutRenderer 选项模拟
    p := tea.NewProgram(model{},
        tea.WithoutRenderer(),
        tea.WithInput(nil),
    )
    finalModel, err := p.Run()
    if err != nil {
        t.Fatalf("program failed: %v", err)
    }
    // 验证最终状态
    _ = finalModel.(model)
}
```

### GitHub Actions CI 配置

```yaml
# .github/workflows/test.yml
- name: Run tests
  run: go test ./...
  env:
    TERM: xterm-256color
    NO_COLOR: "1"
```

### CI 注意事项

- 在 CI 中通常没有真实的 TTY，可能需要对 TUI 执行做特殊处理
- 使用 `tea.WithOutput(&buf)` 和 `tea.WithInput(strings.NewReader("q\n"))` 控制输入输出
- 设置 `TERM=xterm-256color` 确保颜色 profile 可被检测
- `NO_COLOR=1` 可以避免颜色输出影响测试断言
