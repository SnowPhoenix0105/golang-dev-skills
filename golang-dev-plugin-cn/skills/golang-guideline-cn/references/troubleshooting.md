# Go 故障排查指南

## 目录

- [Go 版本不匹配](#go-版本不匹配)
- [Module 依赖冲突](#module-依赖冲突)
- [CGO / 交叉编译问题](#cgo--交叉编译问题)
- [Race Detector](#race-detector)
- [性能与 GC 调优](#性能与-gc-调优)
- [编译错误速查](#编译错误速查)
- [IDE / gopls 问题](#ide--gopls-问题)
- [依赖校验失败](#依赖校验失败)

---

## Go 版本不匹配

当项目要求的 Go 版本与当前激活的版本不一致时，按以下顺序处理：

### 步骤一：检查本地是否已有预装版本

```bash
ls ~/sdk/
```

如果所需版本（如 `go1.22.8`）已存在于 `~/sdk/` 下，直接使用版本化二进制即可：

```bash
go1.22.8 version
go1.22.8 build ./...
```

### 步骤二：通过 `golang.org/dl` 安装

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

### 步骤三：同步 `go.mod`（按需）

```bash
go1.22.8 mod edit -go=1.22.8
go1.22.8 mod tidy
```

在检查和修改 `go.mod` 时应使用版本化的二进制（`go1.22.8`）而非系统默认 `go`，以确保使用正确的工具链。

---

## Module 依赖冲突

### `go mod tidy` 报错

```bash
go mod tidy
```

常见原因与解决：

| 错误信息 | 可能原因 | 解决 |
| :--- | :--- | :--- |
| `module ... found ... but does not contain package ...` | 依赖的路径或版本不正确 | 检查 import 路径，确认包的 module 名称和版本 |
| `no matching versions for query ...` | 版本号不存在或 tag 未打 | 用 `go list -m -versions <module>` 查看可用版本 |
| `invalid version: unknown revision` | commit hash 不存在或拼写错误 | 检查 `go.mod` 中 replace 指令的版本号/commit |
| `ambiguous import` | 多个 module 提供同一个包 | 用 `replace` 指令锁定到其中一个 |

### replace 指令冲突

```bash
# 查看当前模块图
go mod graph

# 查看为何某个模块被引入
go mod why -m <module_path>
```

如果 `replace` 指令导致循环依赖或版本冲突，优先使用 `exclude` 排除有问题的版本，再 `replace` 到可用版本。

### 多个依赖要求不同主版本 (v1 vs v2)

Go module 约定：主版本 ≥2 时必须体现在 module path 中（如 `module/v2`）。如果两个间接依赖分别要求 v1 和 v2，不会冲突——它们被视为不同的 module。

如果确实冲突（非主版本不同），用 `go mod graph | grep <name>` 找到引入方，考虑升级或 replace 其中一方。

---

## CGO / 交叉编译问题

### CGO 编译失败

```bash
# 设置 CGO_ENABLED=0 尝试禁用 CGO
CGO_ENABLED=0 go build .

# 如果必须启用 CGO，确保系统有 C 编译器
# macOS: xcode-select --install
# Ubuntu: sudo apt-get install build-essential
# Windows: 安装 MinGW-w64 或 TDM-GCC
```

### 交叉编译时 CGO 失败

CGO 交叉编译需要目标平台的 C 交叉编译工具链。如果不依赖 C 库，最简单的方式是禁用 CGO：

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build .
```

如果必须用 CGO 交叉编译，使用 `zig` 作为 C 编译器（自动处理交叉编译）：

```bash
CC="zig cc -target x86_64-linux-musl" GOOS=linux GOARCH=amd64 go build .
```

---

## Race Detector

### 启用竞态检测

```bash
go test -race ./...
go build -race ./...
```

### race detector 报告但难以定位

```bash
# 加大 history size（默认 64K，复杂程序可能不够）
go test -race -gcflags="-l" -ldflags="-race" ./...

# 只跑出问题的测试，减少干扰
go test -race -run TestSpecificCase -count=1 ./...
```

### race detector 误报

race detector 不会产生误报（false positive），但有可能是程序逻辑正确但确实存在 data race（如有意不使用同步的 lock-free 结构）。

如果确认是预期行为（如 `sync/atomic` 操作的 race），可以标记为 false positive：

```go
// 注意：此处有意不使用锁，race detector 报告可忽略
```

但仍建议优先使用 `sync/atomic` 或加锁来消除 race。

---

## 性能与 GC 调优

### 排查内存泄漏

```bash
# 运行 benchmark 并输出内存 profile
go test -bench=. -benchmem -memprofile=mem.out ./...

# 查看内存分配
go tool pprof -alloc_space mem.out
```

### GC 相关问题

```bash
# 查看 GC 日志
GODEBUG=gctrace=1 go run main.go

# 设置 GC 目标百分比（默认 100）
# 增大 → 内存换性能（GC 频率降低）
# 减小 → 性能换内存（GC 频率升高）
GOGC=200 go run main.go
```

### 性能分析

```bash
# CPU profile
go test -bench=. -cpuprofile=cpu.out ./...
go tool pprof -http=:8080 cpu.out

# 或对运行中的程序采样
import _ "net/http/pprof"  # 在 main 中 import
# 然后：go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

### 常见性能问题

| 现象 | 排查方向 |
| :--- | :--- |
| 内存持续增长 | 检查是否有 goroutine 泄漏（`runtime.NumGoroutine()`），是否有未关闭的 response body |
| CPU 高 | `go tool pprof` 看热点函数，检查是否有不必要的内存分配（`-benchmem`） |
| GC 停顿长 | 减少堆分配，使用 `sync.Pool` 复用对象 |
| 大量小对象分配 | 考虑用 `sync.Pool` 或预分配 slice |

---

## 编译错误速查

| 错误 | 含义 | 解决 |
| :--- | :--- | :--- |
| `imported and not used: "xxx"` | 导入了未使用的包 | 删除 import 或用 `_ "xxx"` 匿名导入 |
| `xxx declared and not used` | 声明了未使用的变量 | 删除或用 `_` 丢弃 |
| `cannot use xxx as type ...` | 类型不匹配 | 检查类型转换，确认泛型约束 |
| `method has pointer receiver` | 值类型无法调用指针方法 | 改用 `&obj` 指针 |
| `assignment to entry in nil map` | 未初始化的 map 写入 | 先 `make(map[K]V)` |
| `cannot assign to struct field xxx in map` | map 中的 struct 字段不可赋值 | 取指针 `m[key].field = val` → 用指针 map：`map[K]*V` |
| `undefined: xxx` | 符号未定义 | 检查 import 路径、包名、或是否引用了未导出的符号 |
| `main redeclared in this block` | 同一包内有多个 `main()` | Go 每个目录只有一个 package，检查文件 package 声明 |

---

## IDE / gopls 问题

### gopls 不工作或跳转失效

```bash
# 重启 gopls
pkill gopls

# 查看 gopls 状态
gopls version

# 清除 gopls 缓存后重启
rm -rf ~/.cache/gopls
```

### `gopls: not a Go module`

在非 Go module 项目中使用 IDE 功能：

```bash
# 若项目根无 go.mod，初始化 module
go mod init <module-name>
```

### go.mod 变更后 IDE 不更新

```bash
# 重新同步依赖
go mod tidy
go mod download
```

然后重启 gopls（IDE 中执行 "Restart Language Server" 或 `pkill gopls`）。

---

## 依赖校验失败

### `checksum mismatch` / `verifying module: checksum mismatch`

```bash
# 清除本地缓存中的问题模块
go clean -modcache

# 重新下载
go mod download
```

如果仍有问题，检查 `GONOSUMCHECK` 或 `GOPRIVATE` 是否覆盖了私有仓库：

```bash
go env -w GONOSUMCHECK=private.repo.com
go env -w GOPRIVATE=private.repo.com
```

### `go.sum` 文件冲突（git merge 后）

```bash
# 删除 go.sum 后重新生成
rm go.sum
go mod tidy
```
