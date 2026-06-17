# Wails Build & Deploy

This document covers compilation, packaging, cross-compilation, and platform distribution for Wails v2 and v3.

## Table of Contents

- [System Dependencies](#system-dependencies)
- [Dev Mode](#dev-mode)
- [Production Builds](#production-builds)
- [Cross-Compilation](#cross-compilation)
- [Platform-Specific Packaging](#platform-specific-packaging)
- [Docker Builds](#docker-builds)
- [v2 NSIS Installer (Windows)](#v2-nsis-installer-windows)
- [Build Tags](#build-tags)
- [Resource File Structure](#resource-file-structure)

---

## System Dependencies

### Install Wails CLI

```bash
# v2
go install github.com/wailsapp/wails/v2/cmd/wails@latest

# v3
go install github.com/wailsapp/wails/v3/cmd/wails3@latest
```

### Platform Dependency Check

```bash
wails doctor      # v2
wails3 doctor     # v3
```

### Required Dependencies by Platform

**macOS**:
- Xcode Command Line Tools (`xcode-select --install`)

**Windows**:
- [WebView2 Runtime](https://developer.microsoft.com/en-us/microsoft-edge/webview2/) (pre-installed on Windows 10/11)
- GCC (e.g., [TDM-GCC](https://jmeubank.github.io/tdm-gcc/) or MinGW-w64)

**Linux**:
```bash
# Ubuntu/Debian (v3 GTK4 default)
sudo apt install libgtk-4-dev libwebkitgtk-6.0-dev

# Ubuntu/Debian (v3 GTK3 legacy mode)
sudo apt install libgtk-3-dev libwebkit2gtk-4.1-dev

# Ubuntu/Debian (v2)
sudo apt install libgtk-3-dev libwebkit2gtk-4.0-dev

# Fedora
sudo dnf install gtk4-devel webkitgtk-6.0-devel

# Arch
sudo pacman -S gtk4 webkitgtk-6.0
```

---

## Dev Mode

```bash
# v2 — start Vite dev server + Go app
wails dev

# v3 — same
wails3 dev

# v3 with extra flags
wails3 dev -tags server       # Server mode dev
wails3 dev -f                 # Force rebuild
```

Dev mode features:
- Frontend hot reload (HMR via Vite/webpack dev server)
- Go code auto-recompile and restart on changes
- DevTools automatically enabled
- `frontend:dev:serverUrl: "auto"` auto-detects dev server URL
- v3 supports `FRONTEND_DEVSERVER_URL` environment variable override

### Frontend Dev Server Configuration (wails.json)

```json
{
  "frontend:dev:watcher": "npm run dev",
  "frontend:dev:serverUrl": "auto"
}
```

- `"auto"` — auto-detect Vite/webpack dev server output URL
- `"http://localhost:5173"` — fixed URL
- Environment variable `FRONTEND_DEVSERVER_URL` overrides config

---

## Production Builds

```bash
# === v2 ===
wails build                          # Current platform, production mode
wails build -platform windows/amd64  # Specific platform
wails build -platform darwin/arm64,darwin/amd64
wails build -platform linux/amd64
wails build -o myapp                 # Custom output name
wails build -clean                   # Clean before build
wails build -nsis                    # + NSIS installer
wails build -webview2 embed          # Embed WebView2 Bootstrapper
wails build -upx                     # UPX compression
wails build -debug                   # Debug build (keep devtools)

# === v3 ===
wails3 build
wails3 build -platform windows/amd64,darwin/arm64,darwin/amd64,linux/amd64
wails3 build -f                      # Force rebuild
wails3 build -tags server            # Server mode
wails3 build -ldflags "-s -w"        # Custom linker flags
wails3 build -o myapp                # Custom output name
```

### v3 Taskfile Builds (Advanced)

```bash
# Build all examples (all platforms)
task test:examples:all

# Build all examples (current platform)
task test:examples

# Single example
task test:example:darwin DIR=badge
task test:example:windows DIR=badge
task test:example:linux DIR=badge

# Docker builds (Linux)
task test:examples:linux:docker
task test:examples:linux:docker:arm64
task test:examples:linux:docker:x86_64
```

---

## Cross-Compilation

### Principles

- **macOS → Windows/Linux**: Can cross-compile directly
- **Windows → macOS**: Cannot cross-compile directly (requires macOS + Xcode)
- **Linux → Windows**: Can cross-compile directly
- Target platform C toolchain required

### Common Commands

```bash
# From macOS to Windows
wails build -platform windows/amd64

# From macOS to Linux
wails build -platform linux/amd64

# Multi-target from any platform
wails build -platform darwin/amd64,darwin/arm64,windows/amd64,linux/amd64
```

### Environment Variables

```bash
CGO_ENABLED=1
CC=x86_64-w64-mingw32-gcc      # Windows cross-compilation
CXX=x86_64-w64-mingw32-g++
```

---

## Platform-Specific Packaging

### macOS

Build output: `build/bin/myapp.app` (Application Bundle)

```
myapp.app/
└── Contents/
    ├── Info.plist     # Application metadata
    ├── MacOS/
    │   └── myapp      # Binary
    └── Resources/
        └── icon.icns  # Application icon
```

Packaging options:
- `Info.plist` customized at `build/darwin/Info.plist`
- Code signing: `wails build -sign` (requires Apple Developer certificate)
- Notarization: `wails build -notarize`

### Windows

Build output: `build/bin/myapp.exe`

Optional NSIS installer:
```bash
wails build -nsis
# → build/bin/myapp-amd64-installer.exe
```

NSIS config at `build/windows/installer/project.nsi`.

### Linux

Build output: `build/bin/myapp`

Additional files:
- `build/bin/myapp.desktop` — desktop entry file
- Optional AppImage packaging

### v3 Server Mode

```bash
go build -tags server -o myapp-server

# Docker deployment
FROM alpine:latest
COPY myapp-server /app/
COPY frontend/dist /app/frontend/dist/
EXPOSE 8080
CMD ["/app/myapp-server"]
```

---

## Docker Builds

Wails v3 provides Docker build support:

```bash
# Docker cross-compile for Linux
task test:examples:linux:docker
task test:examples:linux:docker:arm64
task test:examples:linux:docker:x86_64
```

Docker files at `v3/test/docker/`:
- `Dockerfile.linux-arm64`
- `Dockerfile.linux-x86_64`

---

## v2 NSIS Installer (Windows)

```bash
wails build -nsis
```

NSIS template files:
- `build/windows/installer/project.nsi` — installer script
- `build/windows/installer/wails_tools.nsh` — tool macros

Custom install options:
- Install path
- Start menu shortcut
- Desktop shortcut
- File associations

---

## Build Tags

### v3 Build Tags

| Tag | Description |
|-----|-------------|
| `production` | Production build: disable devtools, enable optimizations |
| `server` | Server mode: no native window, pure HTTP service |
| `gtk3` | Linux: use GTK3 + WebKit2GTK 4.1 (legacy support) |
| `debug` | Debug build |

### v2 Build Tags

| Tag | Description |
|-----|-------------|
| `production` | Production build |
| `debug` | Debug build |
| `dev` | Dev build |

---

## Resource File Structure

### Application Icon

```
build/
├── appicon.png        # Main icon (≥1024x1024)
├── darwin/
│   └── Info.plist     # macOS Bundle config
└── windows/
    ├── icon.ico       # Windows icon
    └── installer/
        └── project.nsi  # NSIS install script (optional)
```

### macOS Info.plist Example

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" ...>
<plist version="1.0">
<dict>
    <key>CFBundleName</key>
    <string>MyApp</string>
    <key>CFBundleDisplayName</key>
    <string>My App</string>
    <key>CFBundleIdentifier</key>
    <string>com.example.myapp</string>
    <key>CFBundleVersion</key>
    <string>1.0.0</string>
    <key>CFBundlePackageType</key>
    <string>APPL</string>
    <key>CFBundleSignature</key>
    <string>????</string>
    <key>LSMinimumSystemVersion</key>
    <string>10.15</string>
    <key>CFBundleIconFile</key>
    <string>icon.icns</string>
</dict>
</plist>
```

### Frontend Build Output

```
frontend/
├── dist/               # Build output → Go embed
│   ├── index.html
│   ├── assets/
│   │   ├── index-xxx.js
│   │   └── index-xxx.css
│   └── ...
├── src/                # Source
├── package.json
└── vite.config.js
```

Build flow:
1. `wails build` runs `frontend:install` (e.g., `npm install`)
2. Runs `frontend:build` (e.g., `npm run build`)
3. Frontend output goes to `frontend/dist/`
4. Go code `//go:embed all:frontend/dist` embeds the output into the binary
5. Asset Server serves these resources via custom URL scheme
