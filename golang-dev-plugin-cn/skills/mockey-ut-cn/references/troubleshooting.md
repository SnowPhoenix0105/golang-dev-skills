# mockey 排障指南

## 目录

- [mock 不生效](#mock-不生效)
- ["re-mock" panic](#re-mock-panic)
- ["function is too short to patch"](#function-is-too-short-to-patch)
- [参数/返回值不匹配](#参数返回值不匹配)
- [signal SIGBUS (M 系列 Mac)](#signal-sigbus-m-系列-mac)
- [signal SIGSEGV (低版本 macOS)](#signal-sigsegv-低版本-macos)
- [mock init() 中的函数](#mock-init-中的函数)
- [mock 在 goroutine 中失效](#mock-在-goroutine-中失效)
- [接口 mock](#接口-mock-排障)

---

## mock 不生效

按以下顺序排查：

### 1. 未禁用内联和编译优化

检查终端输出是否有：
```
Mockey check failed, please add -gcflags="all=-N -l".
```

解决：使用 `go test -gcflags="all=-l -N" ./...`

### 2. 未调用 Build() 或未指定返回值

即使函数无返回值，也必须调用空的 `Return()` 或使用 `To`：

```go
// 正确：无返回值也要 Return()
mockey.Mock(voidFunc).Return().Build()

// 错误：没有 Return 也没有 To
mockey.Mock(voidFunc).Build() // 不生效
```

### 3. mock 目标不完全匹配

- 值接收器方法用 `A.Foo`，不是 `(*A).Foo`
- 指针接收器方法用 `(*B).Foo`，不是 `B.Foo`
- 实例 mock 方法需要用 `GetMethod`

### 4. 释放时机问题

在 `PatchConvey` / `PatchRun` 中创建的 mock，在作用域结束后自动释放。如果 goroutine 在 mock 释放后才执行目标函数，mock 不会生效。

### 5. 调用先于 mock 执行

常见于 `init()` 函数。打断点到原函数第一行，如果堆栈不在测试代码的 mock 之后，则是时序问题。

### 6. 泛型函数使用了非泛型 mock

低于 Go 1.20 时需使用 `MockGeneric`。

---

## "re-mock" panic

错误信息如：`re-mock xxx, previous mock at: xxx`

**原因**：同一个函数在最小单元（同一个 PatchConvey/PatchRun 或无包裹的测试函数）中被重复 mock。

**解决**：
- 使用嵌套的 `PatchConvey`/`PatchRun` 隔离不同 mock
- 或获取 `Mocker` 后使用 `mocker.Return()` / `mocker.To()` 修改已有 mock
- 或使用 `mocker.Release()` 释放后重新 mock

```go
// 正确：嵌套隔离
mockey.PatchRun(func() {
    mockey.Mock(fn).Return("first").Build()
})
mockey.PatchRun(func() {
    mockey.Mock(fn).Return("second").Build()
})
```

---

## "function is too short to patch"

1. **未禁用内联** — 先检查 `-gcflags="all=-l -N"`
2. **函数确实太短** — 两行以上的函数通常不会，可以尝试 `mockey.MockUnsafe`
3. **已被其他工具 mock** — 检查是否和 monkey/gomonkey 等冲突

---

## 参数/返回值不匹配

错误信息如：`args not match` / `Return Num of Func does not match`

- `Return` 参数数量/类型必须和目标函数一致
- `To` 钩子函数签名必须和目标函数一致
- `When` 条件函数入参必须和目标函数一致
- 如果签名看起来一样，检查 import 路径是否一致

---

## signal SIGBUS (M 系列 Mac)

```
fatal error: unexpected signal during runtime execution
[signal SIGBUS: bus error code=0x1 ...]
```

M 系列 Mac (darwin/arm64) 偶发问题。解决：
- 重试测试
- 升级 mockey 到最新版

---

## signal SIGSEGV (低版本 macOS)

低版本 macOS (10.x / 11.x) 可能出现：

```
[signal SIGSEGV: segmentation violation ...]
```

解决：`go env -w CGO_ENABLED=0` 关闭 cgo

---

## mock init() 中的函数

init() 执行早于单元测试，无法直接 mock。方案：利用 Go init 字典序：

1. 新建一个包（如 `testmock`），在其中 `init()` 里 mock 目标函数，加上环境判断（如 `os.Getenv("CI") == "true"`）
2. 在被测包的第一个 go 文件中最前面额外引用 `testmock`
3. 运行测试时注入 `CI=true`

---

## mock 在 goroutine 中失效

```go
mockey.PatchConvey("test", t, func() {
    mockey.Mock(fn).Return("mocked").Build()
    go func() {
        fn() // 执行时机不确定，可能 mock 已释放
    }()
})
// 主 goroutine 到这里时 mock 可能已释放
```

解决：在 goroutine 内部等待或使用 `IncludeCurrentGoRoutine` / `ExcludeCurrentGoRoutine` 控制作用范围。

---

## 接口 mock 排障

mock 接口的三种方式，按推荐程度：

1. **接口 mock（推荐）** — `mockey/exp/iface` 包，mock 接口方法对所有实现生效
2. **GetMethod** — 从实例获取方法引用后 mock
3. **dummy 实现** — 创建接口的 dummy 实现类型并 mock 构造函数

详见 SKILL.md 中的 API 参考。
