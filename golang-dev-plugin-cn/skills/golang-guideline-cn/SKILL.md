---
name: golang-guideline-cn
description: Go语言开发指南。当编辑、审查、调试、理解Go语言代码时使用此技能，包含开发流程、代码风格、工具链使用、故障排查等。

---

# Go 语言开发指南

作为模型通用 Go 知识的补充而非替代，如果有明确的项目级别 Go 开发约定，应优先遵循项目级别的约定，如果有用户明确要求，也应遵循用户要求。

## 适用场景

- 编写新的 Go 包或模块
- 审查 Go 代码（自己写的或 code review）
- 决定一个辅助函数应该放在哪里
- 处理 error、panic recover、goroutine 生命周期
- 类型断言、`if` 短变量声明等编码风格选择
- 遇到 Go 版本不匹配的错误

不适用：Go 语言本身的基础问题。

---

## 1. 开发流程

### 1.1 先澄清，再动手

编写代码前，确保需求理解到位。遇到以下情况时，**必须向用户提问确认**，不得自行猜测或编撰：

**意图不明确或有歧义** — 用户的描述可能对应多种实现方式，或需求本身存在矛盾。将歧义点列出，让用户选择或澄清。

**上下文信息缺失** — 需要的环境变量、配置文件、依赖服务、项目约定等信息不明确时，先检查已有 skills 和 memory 中是否有记录，仍找不到则向用户询问。不得编造缺失的信息。

**遇到无法自行处理的问题** — 超出技能覆盖范围、涉及未知领域、或需要用户权限/决策的事项，向用户说明情况并求助，而非自行尝试可能造成破坏的操作。

### 1.2 完成一个模块的开发后，应为其编写单元测试并跑通

**完成一个模块的实现后，应为其编写单元测试并跑通，才算模块完成。**

```bash
go test ./path/to/package/...
```

测试覆盖包的公开 API。默认使用 table-driven tests。关注有意义的输入边界和错误路径，而非追求 100% 行覆盖率。

这是**默认行为**，可以被显式指令覆盖。如果用户说"快速原型，不用写测试"，遵从即可。

### 1.3 提交前格式化代码

每轮代码编写完成后，对**本次编辑的文件**执行格式化：

```bash
go fmt path/to/edited_file.go
goimports -w path/to/edited_file.go
```

`go fmt` 处理标准格式化（缩进、换行、空格等）。`goimports` 在 fmt 的基础上自动管理 import 语句（添加缺失的包、删除未使用的包、排序分组）。

注意：
- **只格式化编辑过的文件**，不要对整个项目全局执行，避免产生无关的 diff。
- 如果用户、其它 skill 或其它约束对格式化有不同要求，以那些要求为准，忽略本条规则。

---

## 2. 代码风格

### 2.1 模块化设计：优先用方法而非函数

如果一个辅助逻辑与某个类型强耦合、且不具备包内通用性，应定义为该类型的**方法**，而非包级函数。

```go
// 推荐：与 Server 强耦合的辅助逻辑，写成方法
func (s *Server) resolveAddr(host string) string { ... }

// 避免：只服务于 Server 的包级函数
func resolveServerAddr(host string) string { ... }
```

理由：包级辅助函数会污染包的命名空间，容易在同包的其它文件中冲突，或被错误调用。方法则显式保留了归属关系，作用域更窄、更清晰。

只有满足以下条件之一时，才提升为包级函数：
- 服务于包内的多个类型，或
- 是一个与任何类型无耦合的纯工具函数，或
- 刻意作为包的公开 API 暴露

### 2.2 克制使用 `if` 短变量声明

`if xxx := Func(); xxx.IsXXX() { ... }` 这种写法虽然合法，但容易滥用。遵循以下规则：

**允许**当 `Func()` 的唯一或主要目的就是产生用于判断的值时：
```go
// 允许：map 取值，返回值 + 布尔，无副作用
if v, ok := m[key]; ok { ... }

// 允许：类型断言，无副作用
if rc, ok := body.(io.ReadCloser); ok { ... }
```

**禁止**当 `Func()` 带有副作用时 —— 尤其是带 error 返回值的调用：
```go
// 不当：Func() 有副作用（写盘、网络、状态变更）
if result := saveToDisk(data); result.IsZero() { ... }

// 不当：最常见的滥用 —— 在 if 初始化中做错误检查
if err := doSomething(); err != nil {
    return err
}
```

错误检查放在 if-init 中的问题在于：
- 将函数调用藏在 `if` 头部，降低代码的可扫描性
- 将错误处理与条件分支语义混为一谈
- 与 Go 社区将错误检查单列一行的主流习惯不一致

应拆分为独立的两行：
```go
err := doSomething()
if err != nil {
    return err
}
```

简单判断标准：`(T, bool)`（且无副作用）可以放 if-init；其余情况（包括 `(T, error)`）一律先用独立语句赋值，再对变量做判断。

### 2.3 函数签名：默认第一个参数是 `ctx`

除非函数内部不涉及任何 I/O、数据库、RPC、或可取消的操作（即纯粹的数学计算、字符串处理、简单数据结构操作等基础工具函数），否则函数的**第一个参数必须为 `context.Context`**。

```go
// 推荐：涉及 I/O 或错误路径，ctx 放第一位
func (s *Server) FetchUser(ctx context.Context, id int64) (*User, error) { ... }
func ProcessBatch(ctx context.Context, items []Item) (Result, error) { ... }

// 允许省略 ctx：纯粹的、无副作用的工具函数
func add(a, b int) int { return a + b }
func trimSpace(s string) string { return strings.TrimSpace(s) }
func contains[T comparable](slice []T, val T) bool { ... }
```

理由：`context.Context` 是 Go 中传递超时、取消信号、追踪信息的标准方式。将其放在第一位是 Go 社区的通行惯例，也是多种框架的强制要求。提前列入参数列表比后续重构加上去要容易得多。

判断标准：如果一个函数可能调用任何库函数、可能有错误返回、可能需要记录日志、或可能在未来扩展额外行为，**默认加上 ctx**。

### 2.4 goroutine 必须有 recover

创建新的 goroutine 时，**必须确保 goroutine 内部有 panic recover 机制**，避免单个 goroutine 崩溃导致整个进程退出。

如果项目已有封装好的 recover 工具或并发工具（如 `errgroup`、安全 `go` 启动包装函数等），优先使用封装好的，而非裸写 `go func()`。

```go
// 推荐：使用项目封装的安全启动函数
util.SafeGo(ctx, func() { ... })

// 可接受：自行启动，但显式 defer recover
go func() {
    defer func() {
        if r := recover(); r != nil {
            log.Errorf("goroutine panic: %v\n%s", r, debug.Stack())
        }
    }()
    // ...
}()

// 禁止：裸启动，无 recover
go func() {
    doSomething() // 一旦 panic，整个进程崩溃
}()
```

例外：用户明确要求可以不加 recover，或代码中已有显式注释说明不需要 recover 的理由。如果用户明确要求不 recover，应在代码中加注释说明原因，避免后续 code review 时再次要求补充 recover。

### 2.5 返回 error 时其余返回值无意义

Go 惯例：函数返回 `error` 时，其余返回值默认无意义，调用方不应依赖其值。实现方在返回 error 时应将其他返回值置为零值。

```go
// 推荐：返回 error 时，其余返回值置零值
func Lookup(key string) (*Item, error) {
    if key == "" {
        return nil, errors.New("key must not be empty")
    }
    // ...
}

// 不当：返回 error 同时返回了半成品值，调用方可能误用
func Lookup(key string) (*Item, error) {
    item, err := queryDB(key)
    if err != nil {
        return item, err // item 可能是半成品，调用方不应依赖
    }
}
```

要点：
- **实现方**：`return nil, err` / `return "", err` / `return 0, err`，用显式零值，不传半成品。
- **调用方**：先检查 `err != nil`，在此之前不读取其他返回值。
- `(T, bool)` 模式（如 map 取值、类型断言）不受此规则约束。

### 2.6 不要静默忽略 error

函数返回的 error **必须处理**，不得直接丢弃。按照推荐程度递减：

1. **向上传递** — 最推荐，由调用链上层统一处理
2. **打日志** — 不向上传，但记录错误信息以便排查
3. **显式忽略** — 将 err 赋值出来，再写 `_ = err`，且**必须附带注释**说明可忽略的原因

```go
// 推荐：向上传递
result, err := doSomething()
if err != nil {
    return fmt.Errorf("doSomething: %w", err)
}

// 可接受：打日志后继续
result, err := doSomething()
if err != nil {
    log.Warnf("doSomething failed, continuing: %v", err)
}

// 可接受但不推荐：显式忽略 + 注释
err := doSomething()
_ = err // 此错误在本地文件系统中不会发生，忽略

// 禁止：静默丢弃
doSomething() // 编译不过
result, _ := doSomething() // 静默忽略 error
```

禁止使用 `result, _ := doSomething()` 静默忽略 error（仅 `(T, bool)` 模式的 map 取值、类型断言除外）。

### 2.7 类型断言必须用双返回值形式

类型断言默认使用 `a, ok := x.(T)` 双返回值形式，避免 panic。单返回值 `a := x.(T)` 在断言失败时会直接 panic，应避免使用。

```go
// 推荐：双返回值，安全
a, ok := x.(T)
if !ok {
    // 处理断言失败
}

// 可接受：明确接受断言失败时使用零值
a, _ := x.(T) // 断言失败时 a 为零值，后续逻辑已处理

// 禁止：单返回值，失败时 panic
a := x.(T)
```

即使断言失败在逻辑上不可能发生（如刚通过 switch-type 判断了类型），也必须使用 `a, _ := x.(T)` 形式，禁止使用单返回值 `a := x.(T)`。

### 2.8 禁止使用 dot import

除非用户明确要求或其它 skill 明确要求，**禁止使用 `import . "path/to/package"`**（dot import）。

```go
// 禁止：dot import
import . "fmt"

func main() {
    Println("hello") // 分不清是 fmt.Println 还是自定义的 Println
}
```

dot import 会将外部包的导出符号全部注入当前包的命名空间，带来以下问题：
- **命名冲突**：外部包新增的导出符号可能与当前包的标识符冲突
- **可读性差**：读者无法区分符号来自哪个包，代码审查和维护困难
- **工具链问题**：某些静态分析工具和 IDE 对 dot import 的支持不佳

应使用具名导入（`import "fmt"`）或别名导入（`import fmt_alias "fmt"`）。

例外：用户明确要求使用 dot import，或其它 skill 明确要求使用 dot import。

---

## 3. 故障排查

### 3.1 Go 版本不匹配

当项目要求的 Go 版本与当前激活的版本不一致时，按以下顺序处理：

**步骤一：检查本地是否已有预装版本**

```bash
ls ~/sdk/
```

如果所需版本（如 `go1.22.8`）已存在于 `~/sdk/` 下，直接使用版本化二进制即可：

```bash
go1.22.8 version
go1.22.8 build ./...
```

**步骤二：通过 `golang.org/dl` 安装**

如果 `~/sdk/` 中没有所需版本，使用 Go 官方版本管理器：

```bash
# 安装目标版本的 dl 封装
go install golang.org/dl/go1.22.8@latest

# 下载并安装该版本
go1.22.8 download

# 验证
go1.22.8 version
```

这会自动将版本安装到 `~/sdk/go1.22.8/`。之后直接用 `go1.22.8` 来执行命令即可，无需手动修改 `PATH` 或 `GOROOT`。

**步骤三：同步 `go.mod`（按需）**

如果需要将 `go.mod` 更新为刚安装的版本：

```bash
go1.22.8 mod edit -go=1.22.8
go1.22.8 mod tidy
```

在检查和修改 `go.mod` 时应使用版本化的二进制（`go1.22.8`）而非系统默认 `go`，以确保使用正确的工具链。

---

## 4. 兜底

当本技能的指南不足以覆盖当前场景时（例如与项目级约定冲突、涉及本技能未涵盖的 Go 特性、或用户有明确的风格偏好与本指南不同），**向用户确认具体要求**，不得静默猜测。向用户报告：

- 遇到了什么具体情况
- 本技能给出的默认建议是什么
- 当前场景的特殊之处在哪里

等待用户明确指示后再继续。