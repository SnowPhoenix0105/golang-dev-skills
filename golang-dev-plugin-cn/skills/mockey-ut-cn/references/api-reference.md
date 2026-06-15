# mockey API 参考

## 导入

```go
import "github.com/bytedance/mockey"
import c "github.com/smartystreets/goconvey/convey"  // 仅在需要 goconvey 断言时
```

## 目录

- [函数 Mock](#函数-mock)
- [方法 Mock](#方法-mock)
- [变量 Mock](#变量-mock)
- [钩子函数](#钩子函数)
- [条件 Mock](#条件-mock)
- [序列返回](#序列返回)
- [装饰器模式](#装饰器模式)
- [接口 Mock (`exp/iface`)](#接口-mock-expiface)
- [泛型 Mock](#泛型-mock)
- [GetMethod](#getmethod)
- [Goroutine 过滤](#goroutine-过滤)
- [Mocker 高级操作](#mocker-高级操作)
- [全局工具函数](#全局工具函数)

---

## 函数 Mock

### 基本用法

```go
// mock 函数，指定返回值
mockey.Mock(targetFunc).Return(returnValue).Build()

// mock 无返回值的函数，仍需调用空的 Return()
mockey.Mock(voidFunc).Return().Build()
```

### Mock + Return

```go
import "github.com/bytedance/mockey"

func TestFunc(t *testing.T) {
    mockey.PatchRun(func() {
        mockey.Mock(strconv.Itoa).Return("mocked").Build()
        result := strconv.Itoa(42) // "mocked"
    })
}
```

### Mock + To（自定义钩子）

钩子函数必须与原函数签名一致：

```go
mockey.Mock(targetFunc).To(func(in string) string {
    return "hooked: " + in
}).Build()
```

---

## 方法 Mock

### 值接收器

```go
type A struct{}

func (a A) Foo(in string) string { return in }

// mock 值接收器方法
mockey.Mock(A.Foo).Return("mocked").Build()
```

### 指针接收器

```go
type B struct{}

func (b *B) Foo(in string) string { return in }

// mock 指针接收器方法，注意使用 (*B).Foo
mockey.Mock((*B).Foo).Return("mocked").Build()
```

### 方法 mock 钩子函数

钩子函数可以**选择性包含** receiver 作为第一个参数：

```go
// 包含 receiver
mockey.Mock(A.Foo).To(func(a A, in string) string {
    return a.prefix + ":hooked"
}).Build()

// 不包含 receiver 也可以
mockey.Mock(A.Foo).To(func(in string) string {
    return "hooked"
}).Build()
```

---

## 变量 Mock

使用 `MockValue` mock 变量，传入变量指针：

```go
var globalConfig = "original"

mockey.MockValue(&globalConfig).To("mocked")
defer mockey.MockValue(&globalConfig).UnPatch() // 或使用 PatchRun
```

`MockValue` 返回 `*MockerVar`，也支持 `UnPatch()`。

---

## 钩子函数

`To` 用于指定自定义的替换逻辑。钩子函数签名必须与目标函数一致（参数和返回值）：

```go
func Fibonacci(n int) int {
    if n <= 1 { return n }
    return Fibonacci(n-1) + Fibonacci(n-2)
}

// hook 函数签名为 func(int) int
mockey.Mock(Fibonacci).To(func(n int) int {
    return n * 2  // 自定义逻辑
}).Build()
```

---

## 条件 Mock

使用 `When` 链式定义多个条件，按定义的先后顺序匹配：

```go
mockey.Mock(targetFunc).
    When(func(in string) bool { return len(in) == 0 }).Return("EMPTY").
    When(func(in string) bool { return len(in) <= 3 }).Return("SHORT").
    When(func(in string) bool { return len(in) <= 10 }).Return("MEDIUM").
    Build()
```

条件函数的入参与目标函数一致，返回 `bool`。如果所有条件都不满足，执行原始函数逻辑。

---

## 序列返回

使用 `Sequence` 实现每次调用返回不同值：

```go
mockey.Mock(targetFunc).Return(
    mockey.Sequence("first").Then("second").Times(2).Then("third"),
).Build()

// 第 1 次调用: "first"
// 第 2 次调用: "second"
// 第 3 次调用: "second"
// 第 4 次及以后: "third"（循环）
```

---

## 装饰器模式

使用 `Origin` 在 mock 的同时保留原始函数逻辑：

```go
origin := targetFunc // 仅用于获取类型

decorator := func(in string) string {
    log.Println("before call")
    result := origin(in) // 调用原始逻辑
    log.Println("after call")
    return result
}

mockey.Mock(targetFunc).Origin(&origin).To(decorator).Build()
```

---

## 接口 Mock (`exp/iface`)

通过 `mockey/exp/iface` 包，mock 一个接口方法即可对所有实现了该接口的类型生效，无需逐一 mock 具体类型。

> **注意**：这是实验性功能，需要 Go 1.20+ 且 < 1.26。

### 导入

```go
import "github.com/bytedance/mockey/exp/iface"
```

### 基本用法

mock 接口方法，对所有实现类型生效：

```go
// mock io.Reader.Read，所有 io.Reader 的实现都被 mock
m := iface.Mock(io.Reader.Read).Return(1, io.EOF).Build()
defer m.UnPatch()

// 验证：bytes.Reader、bytes.Buffer、net.TCPConn、os.File 等全部被 mock
data, _ := io.ReadAll(bytes.NewReader([]byte("hello"))) // [0] <nil>
```

### 自定义接口

```go
type MyInterface interface {
    Foo(s string) string
    Bar()
}

// mock 所有 MyInterface 实现
m := iface.Mock(MyInterface.Foo).Return("mocked!").Build()
defer m.UnPatch()

// 无论哪种实现，Foo 都返回 "mocked!"
```

### To（自定义钩子）

钩子签名与原方法一致，可选包含 `unsafe.Pointer` 作为第一个参数（指向 receiver）：

```go
// 不含 receiver — 对所有实现用同一逻辑
iface.Mock(MyInterface.Foo).To(func(s string) string {
    return "MOCKED: " + s
}).Build()

// 含 receiver — 根据具体实现类型分发
iface.Mock(MyInterface.Foo).To(func(r unsafe.Pointer, s string) string {
    switch r {
    case unsafe.Pointer(impl1):
        return "from impl1: " + s
    case unsafe.Pointer(impl2):
        return "from impl2: " + s
    default:
        return "other: " + s
    }
}).Build()
```

### When（条件 Mock）

条件函数签名同样可选包含 receiver：

```go
iface.Mock(MyInterface.Foo).
    When(func(s string) bool { return s == "hello" }).
    Return("hi").Build()

// 含 receiver 的条件
iface.Mock(MyInterface.Foo).
    When(func(r unsafe.Pointer, s string) bool {
        return (*MyImpl)(r).flag && s == "hello"
    }).
    Return("matched").Build()
```

### SelectType / SelectPkg — 限定 mock 范围

如果不希望 mock 所有实现，可以用 selector 精确限定：

```go
// 仅 mock *MyIImpl1 和 *MyIImpl2（按类型名匹配）
iface.Mock(MyInterface.Foo,
    iface.SelectType("MyIImpl1", "MyIImpl2"),
).Return("only two types").Build()

// 仅 mock 特定包下的实现（按完整包路径）
iface.Mock(MyInterface.Foo,
    iface.SelectPkg("example.com/myapp/service"),
).Return("only this pkg").Build()

// 组合使用：AND 语义（类型名 + 包路径同时匹配）
iface.Mock(MyInterface.Foo,
    iface.SelectType("MyIImpl1"),
    iface.SelectPkg("example.com/myapp/service"),
).Return("both conditions").Build()
```

### Mocker 方法

`Build()` 返回的 `*iface.Mocker` 提供：

| 方法 | 作用 |
| :--- | :--- |
| `m.UnPatch()` | 解除所有关联的 mock |
| `m.Times()` | 底层方法的调用总次数 |
| `m.MockTimes()` | mock 实际生效的次数 |

```go
m := iface.Mock(io.Reader.Read).Return(1, nil).Build()
// ... 执行测试 ...
fmt.Println(m.Times())     // 5 = 共调用了 5 次
fmt.Println(m.MockTimes()) // 4 = 其中 4 次被 mock 拦截
m.UnPatch()
```

### 与 PatchConvey / PatchRun 集成

接口 mock 的 `*iface.Mocker` 可以与 `mockey.PatchConvey` / `mockey.PatchRun` 配合使用，作用域结束后自动释放：

```go
mockey.PatchConvey("test with iface mock", t, func() {
    iface.Mock(io.Reader.Read).Return(1, io.EOF).Build()
    // 此处所有 io.Reader 实现都被 mock
    // PatchConvey 结束后自动恢复
})
```

---

## 泛型 Mock

Go 1.20+ 时 `Mock` 可自动识别泛型，无需额外操作：

```go
func FooGeneric[T any](t T) T { return t }

// Go 1.20+：直接使用 Mock
mockey.Mock(FooGeneric[string]).Return("mocked").Build()
```

低于 Go 1.20 时使用 `MockGeneric`：

```go
mockey.MockGeneric(FooGeneric[string]).Return("mocked").Build()
```

**注意**：手动 UnPatch 同一泛型函数的不同类型实例时，必须按"后入先出"顺序释放。

---

## GetMethod

`GetMethod` 用于从实例获取方法引用，适用于以下特殊情况：

### 通过实例 mock 方法

```go
a := &MyStruct{}
// Mock(a.Foo) 不起作用，因为 a 是实例不是类型
mockey.Mock(mockey.GetMethod(a, "Foo")).Return("mocked").Build()
```

### mock 未导出类型的方法

```go
// sha256.New() 返回未导出的 *digest 类型
mockey.Mock(mockey.GetMethod(sha256.New(), "Sum")).Return([]byte{0}).Build()
```

### mock 嵌套结构体中的方法

```go
type Wrapper struct{ inner }
type inner struct{}

// Mock(Wrapper.Foo) 不起作用，因为是 inner 的方法
mockey.Mock(mockey.GetMethod(Wrapper{}, "Foo")).Return("mocked").Build()
```

### OptUnexportedTargetType

当编译器抹除了未导出方法的类型信息时：

```go
var fnType func() bool
mockey.Mock(mockey.GetMethod(instance, "method", mockey.OptUnexportedTargetType(fnType))).
    Return(true).Build()
```

---

## Goroutine 过滤

默认 mock 在所有 goroutine 生效，可以限制范围：

```go
// 仅在当前 goroutine 生效
mockey.Mock(targetFunc).IncludeCurrentGoRoutine().Return("mocked").Build()

// 当前 goroutine 除外
mockey.Mock(targetFunc).ExcludeCurrentGoRoutine().Return("mocked").Build()

// 指定 goroutine ID
gid := mockey.GetGoroutineId()
mockey.Mock(targetFunc).FilterGoRoutine(mockey.Include, gid).Return("mocked").Build()
```

---

## Mocker 高级操作

`Build()` 返回 `*mockey.Mocker`，支持：

### 查询调用次数

```go
mocker := mockey.Mock(targetFunc).Return("mocked").Build()
// ... 执行测试 ...
fmt.Println(mocker.Times())     // targetFunc 被调用的总次数
fmt.Println(mocker.MockTimes()) // mock 实际生效的次数
```

### 重新 mock

```go
mocker.Return("new value")  // 修改返回值
mocker.To(newHookFunc)       // 替换钩子函数
mocker.When(newCondition)    // 替换条件
```

### 释放

```go
mocker.UnPatch()  // 解除 mock
mocker.Release()  // 解除 mock 并重置 builder，可重新 Build()
```

---

## 全局工具函数

| 函数 | 作用 |
| :--- | :--- |
| `mockey.PatchConvey(items...)` | 包装 goconvey.Convey，自动释放 mock |
| `mockey.PatchRun(f func())` | 创建 mock 作用域，函数结束后自动释放 |
| `mockey.UnPatchAll()` | 释放当前作用域内的所有 mock |
| `mockey.MockUnsafe(target)` | 带有较少安全限制的 Mock，用于 Mock 无法工作的情况 |
| `mockey.GetGoroutineId()` | 获取当前 goroutine ID |
| `mockey.GetMethod(instance, name)` | 从实例解析方法引用 |
| `mockey.MockValue(targetPtr)` | mock 变量值 |
| `mockey.OptUnexportedTargetType(t)` | 为 GetMethod 指定未导出方法的类型 |
| `mockey.OptUnsafe` | Mock 选项：解除安全限制 |
