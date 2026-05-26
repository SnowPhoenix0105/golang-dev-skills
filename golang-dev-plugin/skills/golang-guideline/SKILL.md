---
name: golang-guideline
description: Go language development guide. Use this skill when editing, reviewing, debugging, or understanding Go code. Covers development workflow, code style, toolchain usage, and troubleshooting.

---

# Go Language Development Guide

This skill supplements the model's general Go knowledge rather than replacing it. Project-level Go conventions take priority when present. Explicit user instructions also take priority.

## When to Use

- Writing new Go packages or modules
- Reviewing Go code (your own or code review)
- Deciding where to place a helper function
- Handling errors, panic recovery, goroutine lifecycles
- Type assertions, `if` short variable declarations, and other coding style choices
- Encountering Go version mismatch errors

Not applicable: basic Go language questions.

---

## 1. Development Workflow

### 1.1 Clarify Before Coding

Before writing code, ensure requirements are fully understood. In the following situations, **you must ask the user for confirmation** — never guess or fabricate:

**Ambiguous or unclear intent** — the user's description may map to multiple implementations, or the requirements may contain contradictions. List the ambiguities and let the user choose or clarify.

**Missing context** — when environment variables, config files, dependent services, project conventions, or other information is unclear, first check existing skills and memory for relevant records. If still not found, ask the user. Never fabricate missing information.

**Problems beyond your ability** — when the issue exceeds the skill's coverage, involves unknown domains, or requires user permissions or decisions, explain the situation to the user and ask for help rather than attempting potentially destructive operations on your own.

### 1.2 Write and Pass Unit Tests After Completing a Module

**A module is only considered complete after unit tests are written and passing.**

```bash
go test ./path/to/package/...
```

Tests should cover the package's public API. Default to table-driven tests. Focus on meaningful input boundaries and error paths rather than chasing 100% line coverage.

This is the **default behavior** and can be overridden by explicit instructions. If the user says "quick prototype, skip tests," follow that instruction.

### 1.3 Format Code Before Committing

After each round of code changes, format **the files you edited**:

```bash
go fmt path/to/edited_file.go
goimports -w path/to/edited_file.go
```

`go fmt` handles standard formatting (indentation, line breaks, spacing, etc.). `goimports` extends `go fmt` with automatic import management (add missing imports, remove unused imports, sort and group).

Notes:
- **Only format the files you edited.** Do not run formatting globally on the entire project to avoid generating unrelated diffs.
- If the user, other skills, or other constraints specify different formatting requirements, those requirements take priority and this rule is ignored.

---

## 2. Code Style

### 2.1 Modular Design: Prefer Methods Over Functions

If a helper is tightly coupled to a specific type and not generally useful within the package, define it as a **method** on that type rather than a package-level function.

```go
// Recommended: helper tightly coupled to Server, written as a method
func (s *Server) resolveAddr(host string) string { ... }

// Avoid: package-level function that only serves Server
func resolveServerAddr(host string) string { ... }
```

Rationale: package-level helper functions pollute the package namespace, easily conflict with other files in the same package, or get called incorrectly. Methods explicitly retain ownership, with narrower and clearer scope.

Only promote to a package-level function when one of these conditions is met:
- Serves multiple types within the package, or
- Is a pure utility function with no coupling to any type, or
- Is intentionally exposed as part of the package's public API

### 2.2 Restrained Use of `if` Short Variable Declarations

The `if xxx := Func(); xxx.IsXXX() { ... }` pattern is legal but easily abused. Follow these rules:

**Allowed** when `Func()` exists solely or primarily to produce a value used for the condition:
```go
// Allowed: map lookup, returns value + bool, no side effects
if v, ok := m[key]; ok { ... }

// Allowed: type assertion, no side effects
if rc, ok := body.(io.ReadCloser); ok { ... }
```

**Forbidden** when `Func()` has side effects — especially calls that return errors:
```go
// Not OK: Func() has side effects (disk write, network, state change)
if result := saveToDisk(data); result.IsZero() { ... }

// Not OK: the most common abuse — error checking in if-init
if err := doSomething(); err != nil {
    return err
}
```

Problems with error checking in if-init:
- Hides the function call in the `if` header, reducing code scanability
- Conflates error handling with conditional branching semantics
- Inconsistent with the Go community's mainstream habit of placing error checks on their own line

Split into two separate lines:
```go
err := doSomething()
if err != nil {
    return err
}
```

Simple rule: `(T, bool)` (without side effects) can go in if-init; everything else (including `(T, error)`) should be assigned in a separate statement first, then checked.

### 2.3 Function Signatures: `ctx` as the Default First Parameter

Unless the function involves no I/O, database, RPC, or cancellable operations (i.e., pure math, string manipulation, simple data structure operations, and other fundamental utility functions), the **first parameter must be `context.Context`**.

```go
// Recommended: involves I/O or error paths, ctx goes first
func (s *Server) FetchUser(ctx context.Context, id int64) (*User, error) { ... }
func ProcessBatch(ctx context.Context, items []Item) (Result, error) { ... }

// Allowed to omit ctx: pure, side-effect-free utility functions
func add(a, b int) int { return a + b }
func trimSpace(s string) string { return strings.TrimSpace(s) }
func contains[T comparable](slice []T, val T) bool { ... }
```

Rationale: `context.Context` is the standard way to propagate timeouts, cancellation signals, and tracing information in Go. Placing it first is a universal Go community convention and a hard requirement of many frameworks. Adding it to the parameter list upfront is far easier than refactoring later.

Rule of thumb: if a function might call any library function, might return an error, might need to log, or might extend its behavior in the future, **default to adding ctx**.

### 2.4 Goroutines Must Have Recover

When creating a new goroutine, **you must ensure there is a panic recovery mechanism inside the goroutine** to prevent a single goroutine crash from bringing down the entire process.

If the project already has a packaged recover utility or concurrency utility (e.g., `errgroup`, safe `go` launch wrapper, etc.), prefer using the packaged utility over raw `go func()`.

```go
// Recommended: use the project's safe launch wrapper
util.SafeGo(ctx, func() { ... })

// Acceptable: manual launch with explicit defer recover
go func() {
    defer func() {
        if r := recover(); r != nil {
            log.Errorf("goroutine panic: %v\n%s", r, debug.Stack())
        }
    }()
    // ...
}()

// Forbidden: bare launch without recover
go func() {
    doSomething() // if this panics, the entire process crashes
}()
```

Exception: the user explicitly requests no recover, or the code already has an explicit comment explaining why recover is not needed. If the user explicitly requests no recover, add a comment in the code explaining the reason to prevent future code reviews from requesting recover again.

### 2.5 Other Return Values Are Meaningless When Error Is Returned

Go convention: when a function returns an `error`, the other return values are meaningless by default and callers should not rely on them. Implementations should zero out other return values when returning an error.

```go
// Recommended: zero out other return values when returning an error
func Lookup(key string) (*Item, error) {
    if key == "" {
        return nil, errors.New("key must not be empty")
    }
    // ...
}

// Not OK: returning a half-finished value alongside an error, callers may misuse it
func Lookup(key string) (*Item, error) {
    item, err := queryDB(key)
    if err != nil {
        return item, err // item may be partially constructed, callers should not rely on it
    }
}
```

Key points:
- **Implementer**: `return nil, err` / `return "", err` / `return 0, err` — use explicit zero values, never pass back partial results.
- **Caller**: check `err != nil` first; do not read other return values before that check.
- The `(T, bool)` pattern (e.g., map lookup, type assertion) is not subject to this rule.

### 2.6 Never Silently Ignore Errors

Errors returned by functions **must be handled** and must not be silently discarded. In order of preference:

1. **Propagate upward** — most recommended; let the upper call chain handle errors uniformly
2. **Log it** — don't propagate, but record the error for troubleshooting
3. **Explicitly ignore** — assign the error to a variable, then write `_ = err`, with a **mandatory comment** explaining why it can be ignored

```go
// Recommended: propagate upward
result, err := doSomething()
if err != nil {
    return fmt.Errorf("doSomething: %w", err)
}

// Acceptable: log and continue
result, err := doSomething()
if err != nil {
    log.Warnf("doSomething failed, continuing: %v", err)
}

// Acceptable but not recommended: explicit ignore + comment
err := doSomething()
_ = err // this error cannot occur on the local filesystem, ignoring

// Forbidden: silent discard
doSomething() // won't compile
result, _ := doSomething() // silently ignores error
```

Using `result, _ := doSomething()` to silently ignore errors is forbidden (only the `(T, bool)` pattern for map lookups and type assertions is exempt).

### 2.7 Type Assertions Must Use the Two-Value Form

Type assertions default to the `a, ok := x.(T)` two-value form to avoid panics. The single-value form `a := x.(T)` panics on assertion failure and should be avoided.

```go
// Recommended: two-value, safe
a, ok := x.(T)
if !ok {
    // handle assertion failure
}

// Acceptable: explicitly accepting zero value on assertion failure
a, _ := x.(T) // if assertion fails, a is the zero value; subsequent logic handles this

// Forbidden: single-value, panics on failure
a := x.(T)
```

Even when assertion failure is logically impossible (e.g., immediately after a switch-type check), you must use the `a, _ := x.(T)` form. The single-value `a := x.(T)` form is forbidden.

### 2.8 Dot Imports Are Forbidden

Unless explicitly required by the user or another skill, **dot imports (`import . "path/to/package"`) are forbidden**.

```go
// Forbidden: dot import
import . "fmt"

func main() {
    Println("hello") // unclear whether this is fmt.Println or a custom Println
}
```

Dot imports inject all exported symbols of the external package into the current package's namespace, causing the following problems:
- **Naming conflicts**: newly added exported symbols in the external package may clash with identifiers in the current package
- **Poor readability**: readers cannot tell which package a symbol comes from, making code review and maintenance difficult
- **Tooling issues**: some static analysis tools and IDEs have poor support for dot imports

Use named imports (`import "fmt"`) or alias imports (`import fmt_alias "fmt"`) instead.

Exception: the user or another skill explicitly requires dot imports.

---

## 3. Troubleshooting

### 3.1 Go Version Mismatch

When the version of Go required by the project does not match the currently active version, handle it in the following order:

**Step 1: Check if the required version is already installed locally**

```bash
ls ~/sdk/
```

If the required version (e.g., `go1.22.8`) already exists under `~/sdk/`, use the versioned binary directly:

```bash
go1.22.8 version
go1.22.8 build ./...
```

**Step 2: Install via `golang.org/dl`**

If the required version is not in `~/sdk/`, use Go's official version manager:

```bash
# Install the dl wrapper for the target version
go install golang.org/dl/go1.22.8@latest

# Download and install that version
go1.22.8 download

# Verify
go1.22.8 version
```

This automatically installs the version to `~/sdk/go1.22.8/`. After that, use `go1.22.8` directly to run commands — no need to manually modify `PATH` or `GOROOT`.

**Step 3: Sync `go.mod` (as needed)**

If you need to update `go.mod` to the newly installed version:

```bash
go1.22.8 mod edit -go=1.22.8
go1.22.8 mod tidy
```

When inspecting or modifying `go.mod`, use the versioned binary (`go1.22.8`) rather than the system default `go` to ensure the correct toolchain is used.

---

## 4. Fallback

When this skill's guidance is insufficient for the current situation (e.g., conflicting with project-level conventions, involving Go features not covered by this skill, or the user has explicit style preferences that differ from this guide), **confirm the specific requirements with the user** — do not silently guess. Report to the user:

- What specific situation was encountered
- What the default recommendation of this skill is
- What makes the current situation different

Wait for explicit user instructions before continuing.
