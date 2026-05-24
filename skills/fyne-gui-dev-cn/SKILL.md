---
name: fyne-gui-dev-cn
description: |
  Fyne 跨平台 GUI 应用开发。当用户需要创建、修改、调试、或理解 Fyne GUI 应用时使用此技能。
  触发场景：提到 fyne、GUI 桌面应用、跨平台界面、自定义 Widget、fyne 布局、数据绑定、
  widget.New*/container.New*/app.New* 等 Fyne API、Go 语言图形界面开发。
  即使用户没有明确说"fyne"，只要在做 Go GUI 开发就应该考虑此技能。
---

# Fyne GUI 应用开发

Fyne 是纯 Go 实现的跨平台 GUI 框架，基于 OpenGL ES 2.0 硬件加速渲染。
核心设计：**声明式 Widget + 响应式数据绑定**，同一套 API 覆盖 Windows/macOS/Linux/iOS/Android/WebAssembly。

## 核心原则

1. **容器组合 > 绝对定位** — 使用 `container.New*` 系列函数构建布局
2. **数据绑定优先** — 优先用 `binding` 包实现 UI 自动更新，而非手动操作 Widget
3. **线程安全** — 所有 UI 操作必须在主 goroutine 中，通过 `fyne.Do()` / `fyne.DoAndWait()` 调度
4. **Widget 与 Renderer 分离** — Widget 持有状态，Renderer 负责绘制
5. **测试优先** — 使用 `fyne.io/fyne/v2/test` 包进行内存中的 UI 测试

## 最小应用

```go
package main

import (
    "fyne.io/fyne/v2"
    "fyne.io/fyne/v2/app"
    "fyne.io/fyne/v2/container"
    "fyne.io/fyne/v2/widget"
)

func main() {
    a := app.New()              // 或 app.NewWithID("com.example.app")
    w := a.NewWindow("Title")
    w.SetContent(container.NewVBox(
        widget.NewLabel("Hello"),
        widget.NewButton("Click", func() {
            // handle click
        }),
    ))
    w.Resize(fyne.NewSize(400, 300))
    w.ShowAndRun()              // show + app.Run()
}
```

## 框架接口（理解并可能实现）

Fyne 的核心是一套接口合约。内置组件全部实现了这些接口，自定义组件也需要实现它们。理解这些接口是正确使用 Fyne 的基础。

```
层次:                    你做什么:
──────────────────────────────────────────
fyne.App          — 应用程序入口         调用 app.New()
fyne.Window       — 窗口管理             调用 w.SetContent() / w.Show()
fyne.Canvas       — 画布（焦点、缩放）    调用 c.Focus() / c.SetOnTypedKey()
fyne.Layout       — 布局算法             内置布局直接用，自定义布局实现此接口
fyne.Container    — 容器（对象的集合）    调用 container.New*()，通常不直接使用
fyne.Widget       — 组件接口             继承 BaseWidget，实现 CreateRenderer()
fyne.WidgetRenderer — 渲染器接口         实现 Destroy/Layout/MinSize/Objects/Refresh
fyne.CanvasObject — 最底层基础接口       所有可绘制对象必须实现（MinSize/Position/Size/Visible...）
```

**Widget 必须实现的功能接口**（按需选择，不是都要实现）：
`Tappable` → 点击 | `DoubleTappable` → 双击 | `SecondaryTappable` → 右键
`Focusable` → 键盘输入 | `Scrollable` → 滚轮 | `Draggable` → 拖拽
`Disableable` → 启用/禁用 | `Shortcutable` → 快捷键响应

## 内置组件（开箱即用）

以下均由框架提供，直接调用构造函数即可。它们内部实现了上述框架接口。

### 基础 Widget

```go
widget.NewLabel("text")
widget.NewButton("text", func() {})
widget.NewEntry()                       // .Text 获取内容, .OnChanged = func(string)
widget.NewMultiLineEntry()
widget.NewPasswordEntry()
widget.NewEntryWithValidation(placeholder, validator) // 带验证
widget.NewCheck("label", func(bool) {})
widget.NewCheckGroup([]string{"a","b"}, func([]string) {})
widget.NewRadioGroup([]string{"a","b"}, func(string) {})
widget.NewSelect([]string{"a","b"}, func(string) {})  // .PlaceHolder
widget.NewSelectEntry([]string{"a","b"})              // 可输入的下拉
widget.NewSlider(min, max)                            // .Step 设置步长
widget.NewProgressBar()
widget.NewProgressBarInfinite()
widget.NewHyperlink("text", url)
widget.NewIcon(resource)
widget.NewSeparator()
widget.NewCard(title, subtitle, content)
widget.NewCalendar(callback)
widget.NewFileIcon(uri)
widget.NewActivity()
```

表单、工具栏、富文本等高级 Widget 用法见 `references/api-reference.md` Widget 列表和 `references/best-practices.md`。

### 虚拟滚动（大数据集，必须使用）

```go
list := widget.NewList(
    func() int { return rowCount },                     // 行数
    func() fyne.CanvasObject { return widget.NewLabel("template") }, // 模板
    func(id widget.ListItemID, obj fyne.CanvasObject) { // 更新行
        obj.(*widget.Label).SetText(fmt.Sprintf("Row %d", id))
    })
// 类似：widget.NewTable / widget.NewTree
```

### 数据绑定 Widget（响应式 UI）

```go
data := binding.NewString()
widget.NewEntryWithData(data)       // 输入 ↔ data
widget.NewLabelWithData(data)       // data → 自动更新
widget.NewCheckWithData("label", binding.NewBool())
widget.NewSliderWithData(min, max, binding.NewFloat())
widget.NewListWithData(dataList, createFn, updateFn)
```

## 容器与布局

```go
import "fyne.io/fyne/v2/container"

// 常用容器
container.NewVBox(objs...)                         // 垂直（按 MinSize 高度）
container.NewHBox(objs...)                         // 水平（按 MinSize 宽度）
container.NewBorder(top, bottom, left, right, center...) // 边框（nil=不填）
container.NewGridWithColumns(cols, objs...)        // 按列网格
container.NewGridWithRows(rows, objs...)           // 按行网格
container.NewGridWrap(size, objs...)               // 固定大小自动换行
container.NewAdaptiveGrid(rowcols, objs...)        // 自适应（移动端响应旋转）
container.NewStack(objs...)                        // 堆叠（后添加在上层）
container.NewCenter(objs...)                       // 居中
container.NewPadded(objs...)                       // 标准 padding

// 特殊容器
container.NewScroll(content)                       // 滚动
container.NewHSplit(left, right)                   // 水平分割
container.NewVSplit(top, bottom)                   // 垂直分割
container.NewAppTabs(items...)                     // 标签页
container.NewDocTabs(items...)                     // 文档标签页
container.NewWithoutLayout(objs...)                // 无布局（需手动 Resize/Move）
```

弹性占位符：
```go
container.NewHBox(layout.NewSpacer(), widget.NewButton("OK", fn))
```

## 数据绑定

```go
import "fyne.io/fyne/v2/data/binding"

// 基础绑定（goroutine-safe）
s := binding.NewString()               // 值绑定
b := binding.NewBool()                 // 获取/设置：.Get() / .Set()
i := binding.NewInt()
f := binding.NewFloat()

// 集合绑定
binding.NewStringList()                // list 绑定
binding.NewStringTree()                // 树绑定

// 外部绑定（绑定到已有变量，修改变量后需调用 .Reload()）
binding.BindString(&myVar)
binding.BindInt(&myCount)

// 转换
binding.IntToString(intBinding)        // Int → String
binding.StringToInt(strBinding)        // String → Int
binding.SPrintf("%d/%d", cur, total)   // 格式化
```

## Canvas 原始图形

```go
import "fyne.io/fyne/v2/canvas"

canvas.NewRectangle(color)              // 矩形（常用作背景）
canvas.NewCircle(color)
canvas.NewLine(color)                   // 必须用 *canvas.Line
canvas.NewText("text", color)           // 必须用 *canvas.Text
canvas.NewImageFromResource(resource)
canvas.NewImageFromFile(path)
canvas.NewLinearGradient(start, end)
canvas.NewRadialGradient(center, radius)
```

## 自定义 Widget

```go
type MyWidget struct {
    widget.BaseWidget
    Text string
}

func NewMyWidget(text string) *MyWidget {
    w := &MyWidget{Text: text}
    w.ExtendBaseWidget(w)  // 必须！设置内部 impl 指针
    return w
}

func (w *MyWidget) CreateRenderer() fyne.WidgetRenderer {
    bg := canvas.NewRectangle(theme.Color(theme.ColorNameButton))
    label := canvas.NewText(w.Text, theme.Color(theme.ColorNameForeground))
    return &myRenderer{bg: bg, label: label, w: w,
        objs: []fyne.CanvasObject{bg, label}}
}

type myRenderer struct {
    bg, label fyne.CanvasObject
    w         *MyWidget
    objs      []fyne.CanvasObject
}

func (r *myRenderer) Destroy() {}
func (r *myRenderer) Layout(size fyne.Size) { r.bg.Resize(size) }
func (r *myRenderer) MinSize() fyne.Size    { return fyne.NewSize(100, 30) }
func (r *myRenderer) Objects() []fyne.CanvasObject { return r.objs }
func (r *myRenderer) Refresh() {
    r.label.(*canvas.Text).Text = r.w.Text
    r.label.Refresh()
}

// 可选实现 Tappable 等接口
func (w *MyWidget) Tapped(ev *fyne.PointEvent) { /* ... */ }
```

**5 条关键规则**：
1. 构造函数中 **必须** 调用 `ExtendBaseWidget(w)`
2. `Objects()` **必须** 返回非空、元素非 nil 的切片
3. 所有 Canvas 对象使用 **指针**（`*canvas.Line` 不是 `canvas.Line`）
4. **禁止** 在 `Layout()` 中调用 `Refresh()`
5. `MinSize()` 基于子对象计算，**不要硬编码**

## 线程模型

```go
// 后台 goroutine 安全更新 UI
go func() {
    result := expensiveWork()
    fyne.Do(func() {
        label.SetText(result)     // ✅ 安全
    })
}()

// 等待执行完成
var title string
fyne.DoAndWait(func() {
    title = window.Title()
})
```

数据绑定 API 在任何 goroutine 中安全，回调自动通过 `fyne.Do` 调度。

## 对话框

```go
dialog.ShowInformation("Title", "Message", parent)
dialog.ShowConfirm("Title", "Msg", func(ok bool) {}, parent)
dialog.ShowError(err, parent)
dialog.ShowFileOpen(callback, parent)
dialog.ShowFileSave(callback, parent)
dialog.ShowFolderOpen(callback, parent)
dialog.ShowCustom("Title", "OK", content, parent)
dialog.ShowForm("Title", "Submit", "Cancel", formItems, callback, parent)
dialog.ShowEntry("Title", "Msg", parent)
dialog.ShowColorPicker("Title", "Msg", callback, parent)
dialog.NewProgress("Title", "Msg", parent)       // 返回 ProgressDialog
```

## 菜单与快捷键

```go
// 窗口菜单栏
item := fyne.NewMenuItem("Copy", nil)
item.Shortcut = &fyne.ShortcutCopy{Clipboard: w.Clipboard()}
fileMenu := fyne.NewMenu("File",
    fyne.NewMenuItem("New", newFile),
    fyne.NewMenuItemSeparator(),
    fyne.NewMenuItem("Quit", func() { a.Quit() }))
w.SetMainMenu(fyne.NewMainMenu(fileMenu, editMenu))

// 画布级自定义快捷键
w.Canvas().AddShortcut(&desktop.CustomShortcut{
    KeyName: fyne.KeyS, Modifier: fyne.KeyModifierControl | fyne.KeyModifierShift,
}, func(shortcut fyne.Shortcut) { /* Save As */ })
```

## 主题与样式

```go
// TextStyle — 文本样式
fyne.TextStyle{Bold: true, Italic: false, Monospace: false,
    Underline: false, Strikethrough: false, Symbol: false, TabWidth: 4}
```

### 自定义主题（完整模板见 references/api-reference.md）

```go
type myTheme struct{}
func (t myTheme) Color(name fyne.ThemeColorName, v fyne.ThemeVariant) color.Color {
    switch name {
    case theme.ColorNamePrimary:
        return color.NRGBA{R: 70, G: 130, B: 255, A: 255}
    default:
        return theme.DefaultTheme().Color(name, v) // 回退到默认
    }
}
func (t myTheme) Font(s fyne.TextStyle) fyne.Resource { return theme.DefaultTheme().Font(s) }
func (t myTheme) Icon(name fyne.ThemeIconName) fyne.Resource { return theme.DefaultTheme().Icon(name) }
func (t myTheme) Size(name fyne.ThemeSizeName) float32 { return theme.DefaultTheme().Size(name) }

app.Settings().SetTheme(&myTheme{})
container.NewThemeOverride(&myTheme{}, content)  // 局部覆写
```

常用颜色/尺寸/字体/图标查询见 `references/api-reference.md` 样式与主题部分。

## 测试

```go
import "fyne.io/fyne/v2/test"

// 模拟交互
test.Tap(btn)
test.TapCanvas(c, pos)
test.Type(entry, "text")                // 先 FocusGained，再逐 rune 输入
test.Drag(c, pos, deltaX, deltaY)
test.Scroll(c, pos, deltaX, deltaY)
test.MoveMouse(c, pos)
test.FocusNext(c) / test.FocusPrevious(c)

// 验证渲染
r := test.WidgetRenderer(widget)
objs := r.Objects()                     // 检查渲染对象
markup := test.RenderObjectToMarkup(o)  // 快照 (v2.6+)

// 测试环境
c := test.NewCanvas()
a := test.NewApp()
// CI: go test -tags ci ./...
```

## 常见错误速查

| 问题 | 原因 | 解决 |
|------|------|------|
| Widget 不可见 | `Objects()` 空/nil，忘记 `ExtendBaseWidget` | 检查 CreateRenderer 和构造器 |
| 点击无响应 | 尺寸为 0，或未实现 `Tappable` | 手动 Resize 或实现 `Tapped` |
| `panic: Layout is nil` | `Resize()` 在 Renderer 初始化前调用 | 放入容器后再 Resize |
| `panic: Canvas is nil` | Widget 不在窗口层级中 | 先 `SetContent` / `Show` |
| 打包后资源 404 | 使用了文件系统路径 | `fyne bundle` 嵌入为 Go 代码 |
| Grid 有空白 | `Hide()` 的对象仍占格 | 从 `Objects` 中 `Remove` |
| 数据绑定不更新 | 外部变量修改后未 `Reload()` | 调用 `b.Reload()` |
| 非可见 Widget MinSize 为 0 | 框架优化：不可见 Widget 无渲染器 | 放入 Canvas 中再测试 |

## 无从下手？先读这些

当 references 覆盖不到、需要探索 Fyne 源码时，使用以下方法自行查找答案：

### 查找 API

```bash
# 在线文档（首选）
go doc fyne.io/fyne/v2/widget   # 查看 widget 包的所有导出符号
go doc fyne.io/fyne/v2/widget.Button  # 查看具体类型的方法

# 搜索源码中的模式
rg "func.*New.*Pop" $(go env GOMODCACHE)/fyne.io/fyne/v2@*/widget/
rg "type.*Renderer" $(go env GOMODCACHE)/fyne.io/fyne/v2@*/widget/
```

### 学习内置 Widget 的实现

Fyne 内置 Widget 源码是最好的参考。查找路径：`$(go env GOMODCACHE)/fyne.io/fyne/v2@<version>/`

```bash
# 找到 Fyne 安装路径
go list -m -json fyne.io/fyne/v2 | grep Dir

# 看现有 Widget 如何实现 CreateRenderer
cat $(go env GOMODCACHE)/fyne.io/fyne/v2@*/widget/button.go | head -100

# 搜索特定模式
rg "ExtendBaseWidget" $(go env GOMODCACHE)/fyne.io/fyne/v2@*/widget/
rg "CreateRenderer" $(go env GOMODCACHE)/fyne.io/fyne/v2@*/widget/ | head -20
```

### Fyne 代码约定

理解这些约定后，可以从内置 Widget 源码中自行推断未文档化的 API：

- **Widget 构造器**：`New<Name>(requiredParams...) *<Name>`，内部调用 `ExtendBaseWidget(self)`
- **声明式 Widget 和 Renderer 分离**：Widget struct 持有状态，Renderer struct 持有 CanvasObject
- **Renderer 在 `CreateRenderer()` 中创建**并缓存在 `internal/cache` 中
- **命名约定**：`On<Event>` 是回调字段，`Set<Property>` 是 setter，`<Property>()` 是 getter
- **Refresh 触发链**：`widget.Refresh()` → `cache.Renderer(w).Refresh()` → 更新 CanvasObject 状态 → `canvas.Refresh(w)`
- **布局触发链**：`widget.Resize(size)` → `cache.Renderer(w).Layout(size)` → 排列子对象

## 参考资料（按场景查阅）

| 场景 | 查阅文件 | 重点章节 |
|------|---------|---------|
| 不确定用哪个 Widget/API | `references/api-reference.md` | 完整 Widget 列表、Container 列表 |
| 需要完整颜色/图标/字体列表 | `references/api-reference.md` | 样式与主题部分（末尾 TOC） |
| 布局不对/需要响应式设计 | `references/best-practices.md` | 第2节"布局模式" |
| 数据绑定不工作 | `references/best-practices.md` | 第3节"数据绑定模式" |
| Widget 不显示/崩溃/性能差 | `references/troubleshooting.md` | 先查快速诊断表，再查对应章节 |
| 写自定义 Widget | `references/custom-widget.md` | 完整模板 + 5 条关键规则 |
| 单元测试怎么写 | `references/testing.md` | 模拟交互、渲染验证示例 |
| 需要完整架构理解 | `docs/01-核心概念与架构.md` | 分层架构、事件系统 |

上述 references 都覆盖不到时，按照上一节"无从下手？先读这些"的方法探索源码。
