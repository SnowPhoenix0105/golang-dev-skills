# 命令 (Commands)

## 目录

- [Cmd 基础](#cmd-基础)
- [Batch — 并行执行](#batch--并行执行)
- [Sequence — 串行执行](#sequence--串行执行)
- [Tick — 独立时钟周期](#tick--独立时钟周期)
- [Every — 系统时钟同步](#every--系统时钟同步)
- [WindowSize — 查询窗口尺寸](#windowsize--查询窗口尺寸)
- [Quit / Suspend / Interrupt](#quit--suspend--interrupt)
- [自定义 Cmd 模式](#自定义-cmd-模式)
- [Clipboard 操作](#clipboard-操作)
- [Exec — 执行外部命令](#exec--执行外部命令)

## Cmd 基础

```go
type Cmd func() Msg
```

Cmd 是一个返回消息的函数，在后台 goroutine 中异步执行。执行结果（消息）自动发送到 Update。

**关键规则**：
- Cmd 在 goroutine 中执行，不要直接操作 Model
- Cmd 的结果通过 Msg 回到 Update 更新状态
- `nil` Cmd 表示"无操作"，框架会忽略
- 永远不要在 Cmd 中向其他组件发送消息 — 消息自然流转到 Update

## Batch — 并行执行

`tea.Batch` 并行执行多个 Cmd，结果不保证顺序：

```go
func (m model) Init() tea.Cmd {
    return tea.Batch(
        loadUserData,       // 并行加载
        loadConfig,
        startTicker(),
    )
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "enter":
            // 同时执行保存和发送
            return m, tea.Batch(saveData, sendRequest)
        }
    }
    return m, nil
}
```

## Sequence — 串行执行

`tea.Sequence` 按顺序逐个执行 Cmd。前一个 Cmd 的消息返回后，才执行下一个：

```go
func (m model) Init() tea.Cmd {
    return tea.Sequence(
        startLoading,       // 1. 显示加载中
        loadData,           // 2. 加载数据
        finishLoading,      // 3. 隐藏加载
    )
}
```

**注意**：`Sequence` 也是异步的——它不会阻塞 Update，而是按顺序逐个触发 Cmd。如果某个 Cmd 返回 `BatchMsg` 或 `sequenceMsg`，会被自动展开。

## Tick — 独立时钟周期

`tea.Tick` 创建一个定时器，从调用时刻开始计时，执行指定时长后触发消息：

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
        // 继续下一轮 tick
        return m, doTick()
    }
    return m, nil
}
```

**注意**：`Tick` 每次只触发一次。要持续触发，必须在收到 tick 消息后返回新的 `Tick` Cmd。

## Every — 系统时钟同步

`tea.Every` 与系统时钟对齐触发，适合需要整秒、整分钟触发的场景：

```go
func tickEveryMinute() tea.Cmd {
    return tea.Every(time.Minute, func(t time.Time) tea.Msg {
        return tickMsg(t)
    })
}
```

与 `Tick` 的区别：如果当前时间是 `12:34:20`，`Every(time.Minute, ...)` 在 `12:35:00` 触发（40 秒后），而 `Tick(time.Minute, ...)` 在 `12:35:20` 触发（60 秒后）。

## WindowSize — 查询窗口尺寸

```go
func queryWindowSize() tea.Msg {
    return tea.RequestWindowSize()
}
```

通常不需要手动调用——框架在启动和窗口大小变化时自动发送 `WindowSizeMsg`。但在恢复终端（`RestoreTerminal`）后，可能需要检查是否发生大小变化。

## Quit / Suspend / Interrupt

```go
// 退出程序
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "q", "ctrl+c":
            return m, tea.Quit   // 返回特殊命令，触发正常退出
        }
}

// 挂起程序（类似 ctrl+z）
case "ctrl+z":
    return m, tea.Suspend

// 中断程序
case "esc":
    return m, tea.Interrupt
```

`tea.Quit`、`tea.Suspend`、`tea.Interrupt` 都是返回对应 Msg 的函数，对应 `QuitMsg`、`SuspendMsg`、`InterruptMsg`。

## 自定义 Cmd 模式

### 基本模式

```go
// 定义一个返回 Msg 的函数
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

// Init 中启动
func (m model) Init() tea.Cmd {
    return checkServer("https://example.com")
}

// Update 中处理结果
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

### 带参数的 Cmd

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

### Cmd 组合技巧

```go
// 在 Update 中动态决定执行哪个 Cmd
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
        // 加载完成后自动开始下一个操作
        cmds = append(cmds, processData(m.data))
    }

    return m, tea.Batch(cmds...)
}
```

## Clipboard 操作

Bubble Tea 支持原生系统剪贴板操作：

```go
// 读取剪贴板 — ReadClipboard() 返回 Msg，需包装为 Cmd
func readClipboard() tea.Cmd {
    return func() tea.Msg {
        return tea.ReadClipboard()
    }
}

// 写入剪贴板 — SetClipboard 直接返回 Cmd
func writeClipboard(text string) tea.Cmd {
    return tea.SetClipboard(text)
}

// 读取主选择（Linux primary selection）
func readPrimaryClipboard() tea.Cmd {
    return func() tea.Msg {
        return tea.ReadPrimaryClipboard()
    }
}

// 写入主选择
func writePrimaryClipboard(text string) tea.Cmd {
    return tea.SetPrimaryClipboard(text)
}

// 在 Update 中处理剪贴板内容
case tea.ClipboardMsg:
    m.clipboardContent = string(msg)
```

## Exec — 执行外部命令

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

`tea.ExecProcess` 会临时释放终端，执行外部命令，完成后恢复终端并将结果消息发送到 Update。
