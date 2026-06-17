# Wails 编译与部署

本文档覆盖 Wails v2 和 v3 的编译、打包、交叉编译和平台分发。

## 目录

- [系统依赖](#系统依赖)
- [开发模式](#开发模式)
- [生产构建](#生产构建)
- [交叉编译](#交叉编译)
- [平台特定打包](#平台特定打包)
- [Docker 构建](#docker-构建)
- [v2 NSIS 安装包（Windows）](#v2-nsis-安装包windows)
- [编译标签](#编译标签)
- [资源文件结构](#资源文件结构)

---

## 系统依赖

### 安装 Wails CLI

```bash
# v2
go install github.com/wailsapp/wails/v2/cmd/wails@latest

# v3
go install github.com/wailsapp/wails/v3/cmd/wails3@latest
```

### 平台依赖检查

```bash
wails doctor      # v2
wails3 doctor     # v3
```

### 各平台必需依赖

**macOS**：
- Xcode Command Line Tools（`xcode-select --install`）

**Windows**：
- [WebView2 Runtime](https://developer.microsoft.com/en-us/microsoft-edge/webview2/)（Windows 10/11 通常已预装）
- GCC（如 [TDM-GCC](https://jmeubank.github.io/tdm-gcc/) 或 MinGW-w64）

**Linux**：
```bash
# Ubuntu/Debian（v3 默认 GTK4）
sudo apt install libgtk-4-dev libwebkitgtk-6.0-dev

# Ubuntu/Debian（v3 GTK3 遗留模式）
sudo apt install libgtk-3-dev libwebkit2gtk-4.1-dev

# Ubuntu/Debian（v2）
sudo apt install libgtk-3-dev libwebkit2gtk-4.0-dev

# Fedora
sudo dnf install gtk4-devel webkitgtk-6.0-devel

# Arch
sudo pacman -S gtk4 webkitgtk-6.0
```

---

## 开发模式

```bash
# v2 — 启动 Vite dev server + Go app
wails dev

# v3 — 同上
wails3 dev

# v3 带额外参数
wails3 dev -tags server       # 服务器模式开发
wails3 dev -f                 # 强制重建
```

开发模式特性：
- 前端热重载（HMR，通过 Vite/webpack dev server）
- Go 代码修改后自动重新编译和重启
- DevTools 自动启用
- `frontend:dev:serverUrl: "auto"` 自动检测 dev server URL
- v3 支持 `FRONTEND_DEVSERVER_URL` 环境变量覆盖

### 前端 dev server 配置（wails.json）

```json
{
  "frontend:dev:watcher": "npm run dev",
  "frontend:dev:serverUrl": "auto"
}
```

- `"auto"` — 自动检测 Vite/webpack dev server 的输出 URL
- `"http://localhost:5173"` — 固定 URL
- 环境变量 `FRONTEND_DEVSERVER_URL` 可覆盖配置

---

## 生产构建

```bash
# === v2 ===
wails build                          # 当前平台，生产模式
wails build -platform windows/amd64  # 指定平台
wails build -platform darwin/arm64,darwin/amd64
wails build -platform linux/amd64
wails build -o myapp                 # 自定义输出名称
wails build -clean                   # 构建前清理
wails build -nsis                    # + NSIS 安装包
wails build -webview2 embed          # 嵌入 WebView2 Bootstrapper
wails build -upx                     # 使用 UPX 压缩
wails build -debug                   # Debug 构建（保留 devtools）

# === v3 ===
wails3 build
wails3 build -platform windows/amd64,darwin/arm64,darwin/amd64,linux/amd64
wails3 build -f                      # 强制重建
wails3 build -tags server            # 服务器模式
wails3 build -ldflags "-s -w"        # 自定义链接标志
wails3 build -o myapp                # 自定义输出名
```

### v3 Taskfile 构建（高级）

```bash
# 构建所有示例（所有平台）
task test:examples:all

# 构建所有示例（当前平台）
task test:examples

# 单个示例
task test:example:darwin DIR=badge
task test:example:windows DIR=badge
task test:example:linux DIR=badge

# Docker 构建（Linux）
task test:examples:linux:docker
task test:examples:linux:docker:arm64
task test:examples:linux:docker:x86_64
```

---

## 交叉编译

### 原则

- **macOS → Windows/Linux**：可直接交叉编译
- **Windows → macOS**：不可直接交叉编译（需 macOS + Xcode）
- **Linux → Windows**：可直接交叉编译
- 需要目标平台的 C 工具链

### 常用命令

```bash
# 从 macOS 编 Windows
wails build -platform windows/amd64

# 从 macOS 编 Linux  
wails build -platform linux/amd64

# 从任何平台编多目标
wails build -platform darwin/amd64,darwin/arm64,windows/amd64,linux/amd64
```

### 环境变量

```bash
CGO_ENABLED=1
CC=x86_64-w64-mingw32-gcc      # Windows 交叉编译
CXX=x86_64-w64-mingw32-g++
```

---

## 平台特定打包

### macOS

构建输出：`build/bin/myapp.app`（应用 Bundle）

```
myapp.app/
└── Contents/
    ├── Info.plist     # 应用元数据
    ├── MacOS/
    │   └── myapp      # 二进制
    └── Resources/
        └── icon.icns  # 应用图标
```

打包选项：
- `Info.plist` 在 `build/darwin/Info.plist` 自定义
- 代码签名：`wails build -sign`（需要 Apple Developer 证书）
- 公证：`wails build -notarize`

### Windows

构建输出：`build/bin/myapp.exe`

可选 NSIS 安装包：
```bash
wails build -nsis
# → build/bin/myapp-amd64-installer.exe
```

NSIS 配置在 `build/windows/installer/project.nsi`。

### Linux

构建输出：`build/bin/myapp`

附加文件：
- `build/bin/myapp.desktop` — 桌面入口文件
- 可选 AppImage 打包

### v3 服务器模式

```bash
go build -tags server -o myapp-server

# Docker 部署
FROM alpine:latest
COPY myapp-server /app/
COPY frontend/dist /app/frontend/dist/
EXPOSE 8080
CMD ["/app/myapp-server"]
```

---

## Docker 构建

Wails v3 提供 Docker 构建支持：

```bash
# Docker 交叉编译 Linux
task test:examples:linux:docker
task test:examples:linux:docker:arm64
task test:examples:linux:docker:x86_64
```

Docker 文件位于 `v3/test/docker/`：
- `Dockerfile.linux-arm64`
- `Dockerfile.linux-x86_64`

---

## v2 NSIS 安装包（Windows）

```bash
wails build -nsis
```

NSIS 模板文件：
- `build/windows/installer/project.nsi` — 安装程序脚本
- `build/windows/installer/wails_tools.nsh` — 工具宏

自定义安装选项：
- 安装路径
- 开始菜单快捷方式
- 桌面快捷方式
- 文件关联

---

## 编译标签

### v3 构建标签

| 标签 | 说明 |
|------|------|
| `production` | 生产构建：禁用 devtools、启用优化 |
| `server` | 服务器模式：无原生窗口，纯 HTTP 服务 |
| `gtk3` | Linux：使用 GTK3 + WebKit2GTK 4.1（遗留支持） |
| `debug` | Debug 构建 |

### v2 构建标签

| 标签 | 说明 |
|------|------|
| `production` | 生产构建 |
| `debug` | Debug 构建 |
| `dev` | 开发构建 |

---

## 资源文件结构

### 应用图标

```
build/
├── appicon.png        # 主图标（≥1024x1024）
├── darwin/
│   └── Info.plist     # macOS Bundle 配置
└── windows/
    ├── icon.ico       # Windows 图标
    └── installer/
        └── project.nsi  # NSIS 安装脚本（可选）
```

### macOS Info.plist 示例

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

### 前端构建产物

```
frontend/
├── dist/               # 构建输出 → Go embed 打包
│   ├── index.html
│   ├── assets/
│   │   ├── index-xxx.js
│   │   └── index-xxx.css
│   └── ...
├── src/                # 源码
├── package.json
└── vite.config.js
```

构建流程：
1. `wails build` 执行 `frontend:install`（如 `npm install`）
2. 执行 `frontend:build`（如 `npm run build`）
3. 前端产物输出到 `frontend/dist/`
4. Go 代码中 `//go:embed all:frontend/dist` 将产物嵌入二进制
5. Asset Server 通过自定义 URL scheme 提供这些资源
