# Go Troubleshooting Guide

## Table of Contents

- [Go Version Mismatch](#go-version-mismatch)
- [Module Dependency Conflicts](#module-dependency-conflicts)
- [CGO / Cross-Compilation Issues](#cgo--cross-compilation-issues)
- [Race Detector](#race-detector)
- [Performance & GC Tuning](#performance--gc-tuning)
- [Compilation Error Quick Reference](#compilation-error-quick-reference)
- [IDE / gopls Issues](#ide--gopls-issues)
- [Dependency Checksum Failures](#dependency-checksum-failures)

---

## Go Version Mismatch

When the version of Go required by the project does not match the currently active version, handle it in the following order:

### Step 1: Check if the required version is already installed locally

```bash
ls ~/sdk/
```

If the required version (e.g., `go1.22.8`) already exists under `~/sdk/`, use the versioned binary directly:

```bash
go1.22.8 version
go1.22.8 build ./...
```

### Step 2: Install via `golang.org/dl`

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

### Step 3: Sync `go.mod` (as needed)

```bash
go1.22.8 mod edit -go=1.22.8
go1.22.8 mod tidy
```

When inspecting or modifying `go.mod`, use the versioned binary (`go1.22.8`) rather than the system default `go` to ensure the correct toolchain is used.

---

## Module Dependency Conflicts

### `go mod tidy` Errors

```bash
go mod tidy
```

Common causes and fixes:

| Error | Likely Cause | Fix |
| :--- | :--- | :--- |
| `module ... found ... but does not contain package ...` | Incorrect import path or version | Verify import paths; confirm module name and version |
| `no matching versions for query ...` | Version doesn't exist or no tag | Use `go list -m -versions <module>` to list available versions |
| `invalid version: unknown revision` | Commit hash doesn't exist or typo | Check replace directive version/hash in `go.mod` |
| `ambiguous import` | Multiple modules provide the same package | Use `replace` to pin to one |

### replace Directive Conflicts

```bash
# View current module graph
go mod graph

# Check why a module is being pulled in
go mod why -m <module_path>
```

If `replace` causes circular dependencies or version conflicts, prefer `exclude` to block problematic versions first, then `replace` with a working version.

### Multiple Dependencies Require Different Major Versions (v1 vs v2)

Go module convention: major version ≥2 must appear in the module path (e.g., `module/v2`). If two transitive dependencies require v1 and v2 respectively, they won't conflict — they are treated as different modules.

If there is a real conflict (not a major version difference), use `go mod graph | grep <name>` to find the importer, then consider upgrading or replacing one side.

---

## CGO / Cross-Compilation Issues

### CGO Compilation Failure

```bash
# Try disabling CGO first
CGO_ENABLED=0 go build .

# If CGO is required, ensure a C compiler is available
# macOS: xcode-select --install
# Ubuntu: sudo apt-get install build-essential
# Windows: install MinGW-w64 or TDM-GCC
```

### CGO Cross-Compilation Failure

CGO cross-compilation requires the target platform's C cross-compilation toolchain. If you don't depend on C libraries, the simplest approach is to disable CGO:

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build .
```

If CGO cross-compilation is required, use `zig` as the C compiler (handles cross-compilation automatically):

```bash
CC="zig cc -target x86_64-linux-musl" GOOS=linux GOARCH=amd64 go build .
```

---

## Race Detector

### Enabling Race Detection

```bash
go test -race ./...
go build -race ./...
```

### Race Detector Reports but Hard to Pinpoint

```bash
# Increase history size (default 64K; complex programs may need more)
go test -race -gcflags="-l" -ldflags="-race" ./...

# Run only the problematic test to reduce noise
go test -race -run TestSpecificCase -count=1 ./...
```

### Race Detector False Positives

The race detector does not produce false positives, but the program may have intentional data races (e.g., lock-free structures that deliberately avoid synchronization).

If confirmed to be intentional (e.g., `sync/atomic` operations), mark as intentional:

```go
// Note: intentionally unsynchronized; race detector report can be ignored
```

However, it's still recommended to prefer `sync/atomic` or locks to eliminate the race.

---

## Performance & GC Tuning

### Investigating Memory Leaks

```bash
# Run benchmarks with memory profiling
go test -bench=. -benchmem -memprofile=mem.out ./...

# View memory allocations
go tool pprof -alloc_space mem.out
```

### GC-Related Issues

```bash
# View GC logs
GODEBUG=gctrace=1 go run main.go

# Set GC target percentage (default 100)
# Increase → trade memory for performance (less frequent GC)
# Decrease → trade performance for memory (more frequent GC)
GOGC=200 go run main.go
```

### Performance Profiling

```bash
# CPU profile
go test -bench=. -cpuprofile=cpu.out ./...
go tool pprof -http=:8080 cpu.out

# Or sample a running process
import _ "net/http/pprof"  # in main
# Then: go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

### Common Performance Issues

| Symptom | Investigation Direction |
| :--- | :--- |
| Memory grows continuously | Check for goroutine leaks (`runtime.NumGoroutine()`), unclosed response bodies |
| High CPU | `go tool pprof` for hot functions; check for unnecessary allocations (`-benchmem`) |
| Long GC pauses | Reduce heap allocations; use `sync.Pool` for object reuse |
| Many small allocations | Consider `sync.Pool` or pre-allocated slices |

---

## Compilation Error Quick Reference

| Error | Meaning | Fix |
| :--- | :--- | :--- |
| `imported and not used: "xxx"` | Imported but unused package | Remove import or use `_ "xxx"` |
| `xxx declared and not used` | Declared but unused variable | Remove or discard with `_` |
| `cannot use xxx as type ...` | Type mismatch | Check type conversion; confirm generic constraints |
| `method has pointer receiver` | Value type can't call pointer method | Use `&obj` pointer |
| `assignment to entry in nil map` | Writing to uninitialized map | `make(map[K]V)` first |
| `cannot assign to struct field xxx in map` | Can't assign to struct field in map | Use pointer map: `map[K]*V` |
| `undefined: xxx` | Symbol not defined | Check import path, package name, or if referencing unexported symbol |
| `main redeclared in this block` | Multiple `main()` in same package | Go has one package per directory; check file package declarations |

---

## IDE / gopls Issues

### gopls Not Working or Navigation Broken

```bash
# Restart gopls
pkill gopls

# Check gopls status
gopls version

# Clear gopls cache then restart
rm -rf ~/.cache/gopls
```

### `gopls: not a Go module`

Using IDE features in a non-Go-module project:

```bash
# If no go.mod at project root, initialize a module
go mod init <module-name>
```

### IDE Not Updating After go.mod Changes

```bash
# Re-sync dependencies
go mod tidy
go mod download
```

Then restart gopls (in IDE: "Restart Language Server" or `pkill gopls`).

---

## Dependency Checksum Failures

### `checksum mismatch` / `verifying module: checksum mismatch`

```bash
# Clear the problematic module from local cache
go clean -modcache

# Re-download
go mod download
```

If the issue persists, check whether `GONOSUMCHECK` or `GOPRIVATE` covers the private repository:

```bash
go env -w GONOSUMCHECK=private.repo.com
go env -w GOPRIVATE=private.repo.com
```

### `go.sum` File Conflicts (after git merge)

```bash
# Remove go.sum and regenerate
rm go.sum
go mod tidy
```
