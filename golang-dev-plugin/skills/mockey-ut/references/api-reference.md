# mockey API Reference

## Import

```go
import "github.com/bytedance/mockey"
import c "github.com/smartystreets/goconvey/convey"  // only when goconvey assertions are needed
```

## Table of Contents

- [Function Mock](#function-mock)
- [Method Mock](#method-mock)
- [Variable Mock](#variable-mock)
- [Hook Functions](#hook-functions)
- [Conditional Mock](#conditional-mock)
- [Sequence Return](#sequence-return)
- [Decorator Pattern](#decorator-pattern)
- [Generic Mock](#generic-mock)
- [GetMethod](#getmethod)
- [Goroutine Filtering](#goroutine-filtering)
- [Mocker Advanced Operations](#mocker-advanced-operations)
- [Global Utility Functions](#global-utility-functions)

---

## Function Mock

### Basic Usage

```go
// mock a function with a fixed return value
mockey.Mock(targetFunc).Return(returnValue).Build()

// mock a void function — still need empty Return()
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

### Mock + To (Custom Hook)

The hook function must have the same signature as the target:

```go
mockey.Mock(targetFunc).To(func(in string) string {
    return "hooked: " + in
}).Build()
```

---

## Method Mock

### Value Receiver

```go
type A struct{}

func (a A) Foo(in string) string { return in }

// mock value receiver method
mockey.Mock(A.Foo).Return("mocked").Build()
```

### Pointer Receiver

```go
type B struct{}

func (b *B) Foo(in string) string { return in }

// mock pointer receiver method, use (*B).Foo
mockey.Mock((*B).Foo).Return("mocked").Build()
```

### Method Mock Hook Functions

Hook functions can optionally include the receiver as the first argument:

```go
// with receiver
mockey.Mock(A.Foo).To(func(a A, in string) string {
    return a.prefix + ":hooked"
}).Build()

// without receiver also works
mockey.Mock(A.Foo).To(func(in string) string {
    return "hooked"
}).Build()
```

---

## Variable Mock

Use `MockValue` to mock variables, passing a pointer:

```go
var globalConfig = "original"

mockey.MockValue(&globalConfig).To("mocked")
defer mockey.MockValue(&globalConfig).UnPatch() // or use PatchRun
```

`MockValue` returns `*MockerVar`, which also supports `UnPatch()`.

---

## Hook Functions

`To` specifies custom replacement logic. The hook function signature must match the target function (parameters and return values):

```go
func Fibonacci(n int) int {
    if n <= 1 { return n }
    return Fibonacci(n-1) + Fibonacci(n-2)
}

// hook signature must be func(int) int
mockey.Mock(Fibonacci).To(func(n int) int {
    return n * 2  // custom logic
}).Build()
```

---

## Conditional Mock

Use `When` to chain multiple conditions, matched in order of definition:

```go
mockey.Mock(targetFunc).
    When(func(in string) bool { return len(in) == 0 }).Return("EMPTY").
    When(func(in string) bool { return len(in) <= 3 }).Return("SHORT").
    When(func(in string) bool { return len(in) <= 10 }).Return("MEDIUM").
    Build()
```

The condition function's parameters must match the target function, and it must return `bool`. If no condition matches, the original function logic executes.

---

## Sequence Return

Use `Sequence` to return different values on each call:

```go
mockey.Mock(targetFunc).Return(
    mockey.Sequence("first").Then("second").Times(2).Then("third"),
).Build()

// 1st call: "first"
// 2nd call: "second"
// 3rd call: "second"
// 4th call onwards: "third" (cycles)
```

---

## Decorator Pattern

Use `Origin` to preserve the original function logic while mocking:

```go
origin := targetFunc // only used to get the type

decorator := func(in string) string {
    log.Println("before call")
    result := origin(in) // call original logic
    log.Println("after call")
    return result
}

mockey.Mock(targetFunc).Origin(&origin).To(decorator).Build()
```

---

## Generic Mock

Go 1.20+ `Mock` automatically detects generics — no extra steps needed:

```go
func FooGeneric[T any](t T) T { return t }

// Go 1.20+: use Mock directly
mockey.Mock(FooGeneric[string]).Return("mocked").Build()
```

For Go < 1.20, use `MockGeneric`:

```go
mockey.MockGeneric(FooGeneric[string]).Return("mocked").Build()
```

**Note**: When manually UnPatching different type instances of the same generic function, release in LIFO order.

---

## GetMethod

`GetMethod` resolves a method reference from an instance. Useful for these special cases:

### Mock Method via Instance

```go
a := &MyStruct{}
// Mock(a.Foo) won't work — a is an instance, not a type
mockey.Mock(mockey.GetMethod(a, "Foo")).Return("mocked").Build()
```

### Mock Unexported Type Methods

```go
// sha256.New() returns unexported *digest type
mockey.Mock(mockey.GetMethod(sha256.New(), "Sum")).Return([]byte{0}).Build()
```

### Mock Methods in Nested Structs

```go
type Wrapper struct{ inner }
type inner struct{}

// Mock(Wrapper.Foo) won't work — it's inner's method
mockey.Mock(mockey.GetMethod(Wrapper{}, "Foo")).Return("mocked").Build()
```

### OptUnexportedTargetType

When the compiler strips type information for unexported methods:

```go
var fnType func() bool
mockey.Mock(mockey.GetMethod(instance, "method", mockey.OptUnexportedTargetType(fnType))).
    Return(true).Build()
```

---

## Goroutine Filtering

By default mocks apply in all goroutines. You can limit the scope:

```go
// Only in the current goroutine
mockey.Mock(targetFunc).IncludeCurrentGoRoutine().Return("mocked").Build()

// Excluding the current goroutine
mockey.Mock(targetFunc).ExcludeCurrentGoRoutine().Return("mocked").Build()

// By goroutine ID
gid := mockey.GetGoroutineId()
mockey.Mock(targetFunc).FilterGoRoutine(mockey.Include, gid).Return("mocked").Build()
```

---

## Mocker Advanced Operations

`Build()` returns `*mockey.Mocker`, which supports:

### Query Call Counts

```go
mocker := mockey.Mock(targetFunc).Return("mocked").Build()
// ... run tests ...
fmt.Println(mocker.Times())     // total times targetFunc was called
fmt.Println(mocker.MockTimes()) // times mock actually took effect
```

### Re-mock

```go
mocker.Return("new value")  // change return value
mocker.To(newHookFunc)       // replace hook function
mocker.When(newCondition)    // replace condition
```

### Release

```go
mocker.UnPatch()  // remove mock
mocker.Release()  // remove mock and reset builder for re-Build()
```

---

## Global Utility Functions

| Function | Purpose |
| :--- | :--- |
| `mockey.PatchConvey(items...)` | Wraps goconvey.Convey with auto mock release |
| `mockey.PatchRun(f func())` | Creates a mock scope, auto-released after function ends |
| `mockey.UnPatchAll()` | Releases all mocks in the current scope |
| `mockey.MockUnsafe(target)` | Mock with reduced safety restrictions, for when Mock fails |
| `mockey.GetGoroutineId()` | Gets the current goroutine ID |
| `mockey.GetMethod(instance, name)` | Resolves a method reference from an instance |
| `mockey.MockValue(targetPtr)` | Mocks a variable value |
| `mockey.OptUnexportedTargetType(t)` | Specifies unexported method type for GetMethod |
| `mockey.OptUnsafe` | Mock option: disable safety restrictions |
