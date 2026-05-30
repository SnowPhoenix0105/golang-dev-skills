# Compiling, Packaging & Distribution

## Contents

- [1. Build Tags](#1-build-tags)
- [2. Cross Compilation](#2-cross-compilation)
- [3. Desktop Packaging](#3-desktop-packaging)
- [4. Mobile Packaging](#4-mobile-packaging)
- [5. Web Packaging](#5-web-packaging)
- [6. App Store Distribution](#6-app-store-distribution)

---

## 1. Build Tags

Fyne provides several build tags to control behavior and driver selection:

| Tag | Description |
|-----|-------------|
| `debug` | Show debug info including visual layout bounds to help understand layout |
| `flatpak` | Optimize for Linux Flatpak sandbox |
| `gles` | Force embedded OpenGL (GLES) instead of full OpenGL. Normally auto-selected by target device |
| `hints` | Display developer hints for improvements. Logs when your app doesn't follow Material Design |
| `mobile` | Simulate mobile window on desktop — useful for previewing mobile layout |
| `no_animations` | Disable non-essential animations in built-in widgets |
| `no_emoji` | Don't include the embedded emoji font, reducing binary size |
| `no_metadata` | Disable runtime lookup of metadata from FyneApp.toml (always disabled for release builds) |
| `no_native_menus` | macOS only: don't use native menus. Menus display inside the app window (useful for testing Windows/Linux behavior) |
| `wayland` | Linux/BSD: use Wayland window protocol instead of X11 |

Usage:

```bash
go run -tags mobile main.go              # simulate mobile on desktop
go run -tags "hints,debug" main.go       # multiple tags
go test -tags ci ./...                   # CI environment testing
```

---

## 2. Cross Compilation

Fyne uses CGo for graphics rendering, so cross-compilation requires a C compiler for the target platform.

### 2.1 Direct Compilation

Requires `CGO_ENABLED=1` and the correct CC compiler:

| Target GOOS | CC Compiler | Source |
|-------------|-------------|--------|
| `darwin` | `o32-clang` | [osxcross](https://github.com/tpoechtrager/osxcross) (requires macOS SDK) |
| `windows` | `x86_64-w64-mingw64-gcc` | mingw64 (macOS: `brew install mingw-w64`) |
| `linux` | `gcc` or `x86_64-linux-musl-gcc` | musl-cross (macOS: `brew install musl-cross`), also needs X11/mesa headers |

### 2.2 Using fyne-cross (Recommended)

`fyne-cross` uses Docker images with all cross-compilation toolchains pre-installed:

```bash
# Install
go install github.com/fyne-io/fyne-cross@latest

# Build for each platform
fyne-cross linux
fyne-cross windows -arch=*
fyne-cross darwin
fyne-cross android -arch=arm64
fyne-cross ios           # macOS host only
fyne-cross freebsd -arch=amd64

# Specify output name and sub-package
fyne-cross linux -output myapp ./cmd/myapp
```

Supported platforms: darwin(amd64/386), linux(amd64/386/arm64/arm), windows(amd64/386), android(amd64/386/arm64/arm), ios, freebsd(amd64/arm64)

---

## 3. Desktop Packaging

Use the `fyne` CLI tool to automatically handle icon conversion, metadata embedding, and platform-specific formats:

```bash
# Install CLI tool
go install fyne.io/tools/cmd/fyne@latest

# macOS → .app bundle
fyne package -os darwin -icon myapp.png

# Linux → .tar.gz (contains usr/local/ directory structure)
fyne package -os linux -icon myapp.png

# Windows → .exe (embedded icon and metadata)
fyne package -os windows -icon myapp.png

# Release mode (strip debug symbols, reduce size)
fyne package -os windows -icon myapp.png -release

# Install directly to local system
fyne install -icon myapp.png
```

All commands support a default icon file named `Icon.png` in the project root, allowing you to omit the `-icon` parameter.

### 3.1 FyneApp.toml Configuration (v2.1+)

```toml
[Details]
  ID = "com.example.myapp"
  Name = "My Application"
  Version = "1.0.0"
  Build = 1
  Icon = "Icon.png"

[Migrations]
  fyneDo = true   # v2.6.0+: enable new threading model
```

With `FyneApp.toml`, `fyne package` can omit many parameters.

---

## 4. Mobile Packaging

Fyne code compiles directly to Android/iOS apps, but requires additional SDKs and tools:

**Requirements:**
- Android: Android SDK + NDK, `adb` on PATH
- iOS: macOS + Xcode + Command Line Tools

**Packaging commands:**

```bash
# Android (.apk)
fyne package -os android -app-id com.example.myapp -icon mobileIcon.png

# iOS (.app)
fyne package -os ios -app-id com.example.myapp -icon mobileIcon.png

# iOS Simulator
fyne package -os iossimulator -app-id com.example.myapp -icon mobileIcon.png
```

**Installation commands:**

```bash
# Android — install to device
adb install myapp.apk

# iOS — install to simulator
xcrun simctl install booted myapp.app
```

For iOS device installation, open Xcode → Window → Devices and Simulators → drag `.app` onto your device list.

---

## 5. Web Packaging

Fyne apps can run in browsers via WebAssembly:

```bash
# Local development preview
fyne serve                    # starts http://localhost:8080

# Package for web distribution
fyne package -os web          # generates wasm and supporting files
```

**Known limitations (v2.5.0):**
- File open/save dialogs not yet supported
- Documents storage not yet supported

Live demo: [demo.fyne.io](https://demo.fyne.io/)

---

## 6. App Store Distribution

`fyne release` handles signing and store preparation steps after packaging.

### 6.1 macOS App Store

Prerequisites: macOS + Xcode, Apple Developer account, Mac App Store certificates, Apple Transporter

```bash
fyne release -appID com.example.myapp -appVersion 1.0 -appBuild 1 -category games
# Drag .pkg into Transporter, submit for review on AppStore Connect
```

### 6.2 iOS App Store

Prerequisites: macOS + Xcode, Apple Developer account, iOS distribution certificate, Apple Transporter

```bash
fyne release -os ios -appID com.example.myapp -appVersion 1.0 -appBuild 1
# Drag .ipa into Transporter, submit for review on AppStore Connect
```

### 6.3 Google Play Store (Android)

Prerequisites: Google Play Console account, distribution keystore

```bash
fyne release -os android -appID com.example.myapp -appVersion 1.0 -appBuild 1
# Disable "Play app signing" in Play Console, manually upload .apk
```
