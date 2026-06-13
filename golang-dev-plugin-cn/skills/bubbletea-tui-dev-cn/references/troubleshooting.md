# 排障指南

## 目录

- [快速诊断表](#快速诊断表)
- [程序启动问题](#程序启动问题)
- [渲染问题](#渲染问题)
- [输入问题](#输入问题)
- [性能问题](#性能问题)
- [终端兼容性问题](#终端兼容性问题)
- [调试技巧](#调试技巧)

## 快速诊断表

| 症状 | 最可能原因 | 首先检查 |
| :--- | :--- | :--- |
| 无输出，程序卡住 | 未在 Update 中返回 Quit | 是否正确匹配了退出按键 |
| 输出混乱/重叠 | View 渲染了整个屏幕但内容长度变化 | 确保 View 返回完整内容，清除旧内容 |
| 按键无反应 | 错误匹配了 v1 的 `tea.KeyMsg` | 改用 `tea.KeyPressMsg` |
| 鼠标不工作 | View 未启用 MouseMode | 在 View 中设置 `v.MouseMode` |
| 闪烁 | 大量字符串拼接 | 使用 strings.Builder，预分配样式 |
| 窗口调整后布局乱 | 未处理 WindowSizeMsg | 添加 WindowSizeMsg 处理 |
| panic 后终端乱码 | panic 在 Cmd goroutine 中 | 设置 TEA_DEBUG=1 获取 stack trace |
| Cmd 不执行 | 返回了 nil 但期望有行为 | 检查 Cmd 是否正确返回 |
| 子组件不更新 | 消息未转发给子组件 | 检查 Update 中的转发逻辑 |
| 颜色不正确 | 终端不支持 TrueColor | 降级到 ANSI256 或自适应颜色 |

## 程序启动问题

### 程序立即退出

```go
// ❌ 错误：Init 返回了 Quit
func (m model) Init() tea.Cmd {
    return tea.Quit  // Init 中返回 Quit → 程序立即退出
}

// ✅ 正确：Init 只返回初始操作或 nil
func (m model) Init() tea.Cmd {
    return nil
}
```

### "panic: InitialModel cannot be nil"

```go
// ❌ 错误
p := tea.NewProgram(nil)

// ✅ 正确
p := tea.NewProgram(initialModel())
```

### 非 TTY 环境

```go
// 检查是否为 TTY
if !term.IsTerminal(os.Stdout.Fd()) {
    fmt.Println("Not a terminal, falling back to plain mode")
    // 使用 WithoutRenderer 运行
    p := tea.NewProgram(model, tea.WithoutRenderer())
    p.Run()
}
```

## 渲染问题

### 渲染闪烁

最常见原因是每次 View 都创建大量新样式或字符串：

```go
// ❌ 每次创建样式
func (m model) View() tea.View {
    style := lipgloss.NewStyle().Bold(true).Padding(1)
    return tea.NewView(style.Render("content"))
}

// ✅ 全局预分配
var contentStyle = lipgloss.NewStyle().Bold(true).Padding(1)

func (m model) View() tea.View {
    return tea.NewView(contentStyle.Render("content"))
}
```

### 内容未更新

检查 Update 是否返回了正确的 Model：

```go
// ⚠️ 值接收者可以工作（因为 return 了修改后的副本），但大 struct 会带来拷贝开销
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    m.count++
    return m, nil  // 返回的是修改后的副本，修改会生效
}

// ✅ 推荐：使用指针接收者，避免每次 Update 拷贝整个 struct
func (m *model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    m.count++
    return m, nil
}
```

### 部分内容消失

View 应该渲染**整个**屏幕。如果内容行数变化，确保新内容覆盖旧内容的每一行：

```go
// ❌ 可能留有残留
func (m model) View() tea.View {
    return tea.NewView("Short content")  // 如果之前有 20 行，剩余行不会清除
}

// ✅ 填充到完整高度或用空格填充
func (m model) View() tea.View {
    lines := []string{"Header", "Content", "Footer"}
    for len(lines) < m.height {
        lines = append(lines, "")  // 用空行填充
    }
    return tea.NewView(strings.Join(lines, "\n"))
}
```

### AltScreen 内容消失

退出 AltScreen 后所有内容消失是正常行为。如需保留内容：

```go
// 使用 Println 在 TUI 上方打印（持久输出）
p.Println("Important output that stays after exit")
```

## 输入问题

### 按键无反应

v2 中按键消息类型变更：

```go
// ❌ v1 代码（v2 无效）
case tea.KeyMsg:
    switch msg.Type {
    case tea.KeyRunes:  // v1 API
    }

// ✅ v2 代码
case tea.KeyPressMsg:
    switch msg.String() {
    case "enter":
    // ...
    }
```

### 空格键匹配

v2 中空格键的 `String()` 返回 `"space"`，不是 `" "`：

```go
// ❌ v1
case " ":

// ✅ v2
case "space":
```

### 修饰键组合

```go
case tea.KeyPressMsg:
    // 检查 ctrl+c
    if msg.Mod&tea.ModCtrl != 0 && msg.Code == 'c' {
        return m, tea.Quit
    }
    // 检查 ctrl+s
    if msg.Mod&tea.ModCtrl != 0 && msg.Code == 's' {
        // save
    }
```

### 部分终端按键不识别

某些终端不支持全部按键。确保处理常用按键的 fallback：

```go
// 同时支持多种退出方式
case tea.KeyPressMsg:
    switch msg.String() {
    case "ctrl+c", "ctrl+d", "q", "esc":
        return m, tea.Quit
    }
```

## 性能问题

### Update 太慢

Update 耗时超过 1ms 会导致输入延迟：

```go
// ❌ 阻塞 Update
func (m *model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    data := httpGet("https://api.example.com")  // 阻塞！
    m.data = data
    return m, nil
}

// ✅ 将耗时操作包装为 Cmd
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

### View 字符串拼接

```go
// ❌ 循环中反复 +=
func (m model) View() tea.View {
    s := ""
    for _, item := range m.items {
        s += renderItem(item) + "\n"  // 频繁内存分配
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

### 渲染帧率过高

默认 60 FPS，可通过 `WithFPS` 调整：

```go
p := tea.NewProgram(model, tea.WithFPS(30))  // 降到 30 FPS
```

### 大量列表项

超过 1000 项时应使用虚拟滚动或限制渲染范围：

```go
// 只渲染可见范围的项
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

## 终端兼容性问题

### 颜色降级

```go
// 使用自适应颜色，自动适配深色/浅色背景
var color = lipgloss.AdaptiveColor{Light: "#333", Dark: "#FFF"}

// 避免纯十六进制颜色在不支持 TrueColor 的终端中显示异常
// 提供 fallback 颜色
var color = lipgloss.Color("#7D56F4")  // 在不支持的终端会降级
```

### 最小终端尺寸

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

### SSH 环境

```go
// 通过环境变量检测 SSH
if _, ok := os.LookupEnv("SSH_TTY"); ok {
    // SSH 会话中的特殊处理：
    // - 降低帧率
    p := tea.NewProgram(model, tea.WithFPS(15))
    // - 避免使用某些终端特性
    // - 使用 WithEnvironment 传递远程环境变量
}
```

### 不同平台差异

```go
// Windows 特殊处理
import "runtime"

if runtime.GOOS == "windows" {
    // 启用 Windows 终端虚拟处理
}
```

## 调试技巧

### 日志调试

```go
// 设置日志文件
f, err := tea.LogToFile("debug.log", "debug")
if err != nil {
    log.Fatal(err)
}
defer f.Close()

// 在代码中打日志
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    log.Printf("Received message: %T %+v", msg, msg)  // 会写入 debug.log
    // ...
}
```

使用另一个终端窗口 `tail -f debug.log` 实时查看。

### TEA_TRACE 环境变量

```bash
# 获取 Bubble Tea 内部事件追踪
TEA_TRACE=trace.log go run .
# 查看所有输入/渲染事件
tail -f trace.log
```

### Panic 调试

```bash
# TEA_DEBUG=1 会在 panic 时输出详细日志文件
TEA_DEBUG=1 go run .
# 会生成 bubbletea-panic-<timestamp>.log
```

### Delve 调试器

```bash
# 启动 headless delve（因为 TUI 占用 stdin/stdout）
dlv debug --headless --api-version=2 --listen=127.0.0.1:43000 .

# 另一个终端连接
dlv connect 127.0.0.1:43000

# 设置断点
(dlv) break main.go:45
(dlv) continue
```

### 已知问题排查

1. **tmux 中鼠标不工作** — 确保 tmux 配置了 `set -g mouse on`
2. **tmux 中焦点事件不可用** — 需要 tmux 配置 focus-events
3. **iTerm2 中某些颜色显示异常** — 检查 iTerm2 的 minimum contrast 设置
4. **Kitty 中部分按键码不同** — 启用 KeyboardEnhancements 以获取精确的键码
