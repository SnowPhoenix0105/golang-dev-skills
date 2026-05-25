# mockey Troubleshooting Guide

## Table of Contents

- [Mock Not Taking Effect](#mock-not-taking-effect)
- ["re-mock" Panic](#re-mock-panic)
- ["function is too short to patch"](#function-is-too-short-to-patch)
- [Parameter/Return Value Mismatch](#parameterreturn-value-mismatch)
- [signal SIGBUS (Apple Silicon Mac)](#signal-sigbus-apple-silicon-mac)
- [signal SIGSEGV (Older macOS)](#signal-sigsegv-older-macos)
- [Mocking Functions in init()](#mocking-functions-in-init)
- [Mock Fails in Goroutines](#mock-fails-in-goroutines)
- [Interface Mock Troubleshooting](#interface-mock-troubleshooting)

---

## Mock Not Taking Effect

Check in order:

### 1. Inlining / Compiler Optimizations Not Disabled

Check terminal output for:
```
Mockey check failed, please add -gcflags="all=-N -l".
```

Fix: Use `go test -gcflags="all=-l -N" ./...`

### 2. Missing Build() or Return Value

Even for void functions, you must call empty `Return()` or use `To`:

```go
// Correct: call Return() even for void functions
mockey.Mock(voidFunc).Return().Build()

// Wrong: no Return and no To
mockey.Mock(voidFunc).Build() // won't work
```

### 3. Mock Target Mismatch

- Value receiver methods use `A.Foo`, not `(*A).Foo`
- Pointer receiver methods use `(*B).Foo`, not `B.Foo`
- Mocking methods on instances requires `GetMethod`

### 4. Release Timing Issue

Mocks created inside `PatchConvey` / `PatchRun` are auto-released when the scope ends. If a goroutine executes the target function after mock release, it won't be mocked.

### 5. Call Happens Before Mock

Common in `init()` functions. Set a breakpoint on the first line of the original function — if the stack doesn't show test code after the mock, it's a timing issue.

### 6. Generic Function with Non-Generic Mock

Use `MockGeneric` for Go versions below 1.20.

---

## "re-mock" Panic

Error message: `re-mock xxx, previous mock at: xxx`

**Cause**: The same function is mocked twice within the same minimal scope (same PatchConvey/PatchRun, or unwrapped test function).

**Fix**:
- Use nested `PatchConvey`/`PatchRun` to isolate different mocks
- Or use `mocker.Return()` / `mocker.To()` to modify an existing mock
- Or use `mocker.Release()` before re-mocking

```go
// Correct: nested isolation
mockey.PatchRun(func() {
    mockey.Mock(fn).Return("first").Build()
})
mockey.PatchRun(func() {
    mockey.Mock(fn).Return("second").Build()
})
```

---

## "function is too short to patch"

1. **Inlining not disabled** — check `-gcflags="all=-l -N"` first
2. **Function genuinely too short** — functions with 2+ lines usually don't have this issue; try `mockey.MockUnsafe`
3. **Already mocked by another tool** — check for conflicts with monkey/gomonkey

---

## Parameter/Return Value Mismatch

Error messages: `args not match` / `Return Num of Func does not match`

- `Return` parameter count/type must match the target function
- `To` hook function signature must match the target function
- `When` condition function parameters must match the target function
- If signatures appear identical, check import paths match

---

## signal SIGBUS (Apple Silicon Mac)

```
fatal error: unexpected signal during runtime execution
[signal SIGBUS: bus error code=0x1 ...]
```

Occasional issue on Apple Silicon Mac (darwin/arm64). Fix:
- Retry the test
- Upgrade mockey to the latest version

---

## signal SIGSEGV (Older macOS)

Older macOS (10.x / 11.x) may encounter:

```
[signal SIGSEGV: segmentation violation ...]
```

Fix: `go env -w CGO_ENABLED=0` to disable cgo

---

## Mocking Functions in init()

init() runs before unit tests, so direct mocking doesn't work. Workaround using Go's init ordering (lexicographic):

1. Create a new package (e.g., `testmock`) with an `init()` that mocks the target function, guarded by an env check (e.g., `os.Getenv("CI") == "true"`)
2. Import `testmock` as the first import in the first Go file (lexicographically) of the package under test
3. Run tests with `CI=true`

---

## Mock Fails in Goroutines

```go
mockey.PatchConvey("test", t, func() {
    mockey.Mock(fn).Return("mocked").Build()
    go func() {
        fn() // execution timing is uncertain — mock may already be released
    }()
})
// By the time the main goroutine reaches here, mocks may be released
```

Fix: Wait inside the goroutine, or use `IncludeCurrentGoRoutine` / `ExcludeCurrentGoRoutine` to control scope.

---

## Interface Mock Troubleshooting

Three approaches to mock interfaces, in order of preference:

1. **Interface mock (recommended)** — `mockey/exp/iface` package, mocks interface methods for all implementations
2. **GetMethod** — resolve method reference from instance, then mock
3. **Dummy implementation** — create a dummy implementation type and mock the constructor

See the API reference in SKILL.md for details.
