# 最佳实践与使用模式

## 目录

- [1. 项目结构](#1-项目结构)
- [2. 布局模式](#2-布局模式)
- [3. 数据绑定模式](#3-数据绑定模式)
- [4. 异步操作模式](#4-异步操作模式)
- [5. 自定义 Widget 深入](#5-自定义-widget-深入)
- [6. 性能优化](#6-性能优化)
- [7. 平台适配](#7-平台适配)
- [8. 窗口管理](#8-窗口管理)
- [9. 资源管理](#9-资源管理)
- [10. 对话框模式](#10-对话框模式)
- [11. Navigation 导航模式](#11-navigation-导航模式)
- [12. Canvas 样式模式](#12-canvas-样式模式)
- [13. Label 可选文本](#13-label-可选文本)
- [14. RowWrap 布局](#14-rowwrap-布局)

---

## 1. 项目结构

推荐的 Fyne 项目结构：

```
myapp/
├── main.go              # 入口，app 初始化
├── FyneApp.toml         # 元数据（ID、版本、图标、构建设置）
├── ui/
│   ├── views/           # 页面/屏幕
│   ├── widgets/         # 自定义可复用组件
│   └── theme/           # 自定义主题
├── model/               # 数据模型和业务逻辑
├── bindings/            # 响应式数据绑定
└── go.mod
```

### FyneApp.toml

```toml
[Details]
  ID = "com.example.myapp"
  Name = "My Application"
  Version = "1.0.0"
  Build = 1
  Icon = "Icon.png"

[Migrations]
  fyneDo = true   # v2.6.0+：启用新线程模型
```

## 2. 布局模式

### 2.1 基本页面结构

```go
func mainPage() fyne.CanvasObject {
    header := widget.NewLabel("App Title")
    sidebar := widget.NewList(/* ... */)
    body := widget.NewLabel("Content")
    footer := widget.NewLabel("Status: Ready")

    return container.NewBorder(
        header,               // top
        footer,               // bottom
        sidebar,              // left
        nil,                  // right (不填)
        body,                 // center
    )
}
```

### 2.2 表单布局

```go
func formPage() fyne.CanvasObject {
    nameEntry := widget.NewEntry()
    emailEntry := widget.NewEntry()
    passwordEntry := widget.NewPasswordEntry()

    form := &widget.Form{
        Items: []*widget.FormItem{
            {Text: "Name", Widget: nameEntry},
            {Text: "Email", Widget: emailEntry},
            {Text: "Password", Widget: passwordEntry},
        },
        OnSubmit: func() {
            // 验证并提交
        },
    }
    return form
}
```

### 2.3 自适应网格（响应式）

```go
// rowcols 是横屏时的列数，竖屏时自动变为行数
grid := container.NewAdaptiveGrid(4,
    widget.NewButton("A", nil),
    widget.NewButton("B", nil),
    // ...
)
```

### 2.4 弹性空间

```go
// 利用 layout.Spacer 将按钮推到右侧
toolbar := container.NewHBox(
    layout.NewSpacer(),
    widget.NewButton("Save", saveFn),
    widget.NewButton("Cancel", cancelFn),
)
```

### 2.5 固定大小的 sidebar

```go
sidebar := widget.NewList(/* ... */)
content := widget.NewLabel("Body")

// 左栏固定宽度，右栏占据剩余
split := container.NewHSplit(sidebar, content)
split.Offset = 0.2  // 左栏占 20%
```

## 3. 数据绑定模式

### 3.1 双向绑定（输入 ↔ 显示）

```go
data := binding.NewString()
data.Set("initial")

// 三个 Widget 共享同一个 data source
input := widget.NewEntryWithData(data)
preview := widget.NewLabelWithData(data)
count := widget.NewLabelWithData(
    binding.SPrintf("Length: %d", binding.StringToInt(data)))
```

### 3.2 列表绑定

```go
items := binding.NewStringList()

list := widget.NewListWithData(items,
    func() fyne.CanvasObject {
        return widget.NewLabel("template")
    },
    func(item binding.DataItem, obj fyne.CanvasObject) {
        label := obj.(*widget.Label)
        label.Bind(item.(binding.String))
    })

// 在后台线程更新数据（安全）
go func() {
    results := fetchData()
    items.Set(results)  // 安全——自动通过 fyne.Do 通知
}()
```

### 3.3 外部绑定（绑定到已有变量）

```go
type Model struct {
    Name  string
    Count int
}
m := &Model{Name: "Alice", Count: 5}

nameBinding := binding.BindString(&m.Name)
countBinding := binding.BindInt(&m.Count)

widget.NewLabelWithData(nameBinding)
widget.NewLabelWithData(binding.IntToString(countBinding))

// 外部代码修改 m 后，需要通知：
m.Name = "Bob"
nameBinding.Reload()  // 重要！
```

### 3.4 偏好绑定

```go
a := app.NewWithID("com.example.app")
prefs := a.Preferences()

// 与 Preferences 自动双向同步
themeBinding := binding.BindPreferenceString("theme", prefs)
widget.NewSelectWithData(
    []string{"light", "dark"},
    themeBinding,
    func(val string) {
        // switch theme
    })
```

## 4. 异步操作模式

### 4.1 后台任务 + UI 更新

```go
func loadData(progress *widget.ProgressBar, label *widget.Label) {
    go func() {
        for i := 0; i <= 100; i += 10 {
            time.Sleep(100 * time.Millisecond)
            fyne.Do(func() {
                progress.SetValue(float64(i) / 100.0)
                label.SetText(fmt.Sprintf("Loading... %d%%", i))
            })
        }

        result := computeResult()
        fyne.Do(func() {
            label.SetText(result)
        })
    }()
}
```

### 4.2 使用数据绑定代替手动线程调度

```go
// 更简洁的方式：用数据绑定
progress := binding.NewFloat()
status := binding.NewString()

go func() {
    for i := 0; i <= 100; i += 10 {
        time.Sleep(100 * time.Millisecond)
        progress.Set(float64(i) / 100.0)
        status.Set(fmt.Sprintf("Loading... %d%%", i))
    }
    status.Set("Done!")
}()

// UI 组件自动响应
widget.NewProgressBarWithData(progress)
widget.NewLabelWithData(status)
```

## 5. 自定义 Widget 深入

### 5.1 支持数据绑定的自定义 Widget

```go
type ToggleWidget struct {
    widget.BaseWidget
    Label  string
    Active bool
    OnChanged func(bool)
}

func (w *ToggleWidget) CreateRenderer() fyne.WidgetRenderer {
    return newToggleRenderer(w)
}

func (w *ToggleWidget) Tapped(ev *fyne.PointEvent) {
    w.Active = !w.Active
    if w.OnChanged != nil {
        w.OnChanged(w.Active)
    }
    w.Refresh()  // 触发渲染器更新
}
```

### 5.2 支持自定义 Theme 的 Widget

```go
func (r *myRenderer) MinSize() fyne.Size {
    th := r.w.Theme()   // 获取当前 Theme（可能来自 ThemeOverride 容器）
    pad := th.Size(theme.SizeNamePadding)
    textSize := th.Size(theme.SizeNameText)
    return fyne.NewSize(textSize*10+pad*2, textSize+pad*2)
}

func (r *myRenderer) Refresh() {
    th := r.w.Theme()
    r.bg.FillColor = th.Color(theme.ColorNameBackground, fyne.ThemeVariantLight)
    r.bg.Refresh()
    canvas.Refresh(r.w)
}
```

### 5.3 使用 cache.Renderer 获取子 Widget 渲染器

```go
import "fyne.io/fyne/v2/internal/cache"

func (r *myRenderer) Layout(size fyne.Size) {
    childRenderer := cache.Renderer(r.w.childWidget)
    childRenderer.Layout(size)
}
```

## 6. 性能优化

### 6.1 虚拟滚动

大数据集（>100 行）**必须使用虚拟滚动**：

```go
// ✅ O(可见行数) 复杂度
list := widget.NewList(dataLen, createTemplate, updateRow)

// ❌ O(总行数) — 会一次性创建所有 Widget
for _, item := range items {
    container.Add(widget.NewLabel(item))
}
```

### 6.2 批量更新

```go
// ✅ 批量操纵后一次性 Refresh
data := binding.NewStringList()
newItems := fetchItems()
data.Set(newItems)  // 一次触发

// ❌ 循环内逐个触发
for _, item := range items {
    data.Append(item)  // 每次触发一次 Refresh
}
```

### 6.3 避免不必要的渲染

```go
// Widget 的 Resize 已经做了相等性检查
func (w *BaseWidget) Resize(size fyne.Size) {
    if size == w.Size() {
        return  // 尺寸没变，跳过
    }
    // ...
}
```

## 7. 平台适配

### 7.1 检测设备类型

```go
if fyne.CurrentDevice().IsMobile() {
    // 移动端特定布局
    w.SetContent(container.NewVBox(/* ... */))
} else {
    // 桌面端
    w.SetContent(container.NewHSplit(sidebar, content))
}

if fyne.CurrentDevice().IsBrowser() {
    // WebAssembly 特定处理
}
```

### 7.2 键盘设备判断

```go
if fyne.CurrentDevice().HasKeyboard() {
    // 注册键盘快捷键
    w.Canvas().SetOnTypedKey(func(ev *fyne.KeyEvent) {
        // ...
    })
}
```

## 8. 窗口管理

### 8.1 多窗口

```go
a := app.New()
mainWin := a.NewWindow("Main")
settingsWin := a.NewWindow("Settings")

// 默认：所有窗口关闭 → 退出
// 自定义：拦截关闭
settingsWin.SetCloseIntercept(func() {
    if hasUnsavedChanges {
        dialog.ShowConfirm("Quit?", "Unsaved changes!", func(ok bool) {
            if ok { settingsWin.Close() }
        }, settingsWin)
    } else {
        settingsWin.Close()
    }
})
```

### 8.2 System Tray（桌面功能）

```go
// 如果 driver 支持 desktop 特性
if desk, ok := a.(desktop.App); ok {
    menu := fyne.NewMenu("Tray",
        fyne.NewMenuItem("Show", func() { w.Show() }),
        fyne.NewMenuItem("Quit", func() { a.Quit() }),
    )
    desk.SetSystemTrayMenu(menu)
}
```

## 9. 资源管理

### 9.1 资源嵌入

```bash
# 命令行
fyne bundle -o bundled.go image.png font.ttf

# 或 go:generate
//go:generate fyne bundle -o bundled.go assets/*.png
```

```go
img := canvas.NewImageFromResource(resourceImagePng)
```

### 9.2 自定义主题

```go
type CustomTheme struct{}

func (t CustomTheme) Color(name fyne.ThemeColorName, variant fyne.ThemeVariant) color.Color {
    switch name {
    case theme.ColorNamePrimary:
        return color.NRGBA{R: 70, G: 130, B: 255, A: 255}
    default:
        return theme.DefaultTheme().Color(name, variant)
    }
}

func (t CustomTheme) Font(style fyne.TextStyle) fyne.Resource {
    if style.Monospace {
        return resourceMonospaceFont
    }
    return theme.DefaultTheme().Font(style)
}

func (t CustomTheme) Icon(name fyne.ThemeIconName) fyne.Resource {
    return theme.DefaultTheme().Icon(name)
}

func (t CustomTheme) Size(name fyne.ThemeSizeName) float32 {
    return theme.DefaultTheme().Size(name)
}

// 应用
a.Settings().SetTheme(&CustomTheme{})

// 局部主题覆写
container.NewThemeOverride(&CustomTheme{}, content)
```

## 10. 对话框模式

### 10.1 确认操作

```go
func deleteItem(id string, w fyne.Window) {
    dialog.ShowConfirm("Delete Item",
        fmt.Sprintf("Delete item #%s?", id),
        func(confirmed bool) {
            if confirmed {
                doDelete(id)
            }
        }, w)
}
```

### 10.2 进度对话框

```go
func processWithProgress(w fyne.Window) {
    prog := dialog.NewProgress("Processing", "Please wait...", w)

    go func() {
        for i := 0; i <= 100; i += 10 {
            time.Sleep(100 * time.Millisecond)
            fyne.Do(func() { prog.SetValue(float64(i) / 100.0) })
        }
        fyne.Do(func() {
            prog.Hide()
            dialog.ShowInformation("Done", "Processing complete", w)
        })
    }()

    prog.Show()
}
```

## 11. Navigation 导航模式

`container.NewNavigation` (v2.7+) 提供带返回按钮的页面导航栈，适合设置页、向导等多层级界面。

```go
func settingsPage() fyne.CanvasObject {
    general := widget.NewLabel("General Settings")
    advanced := widget.NewLabel("Advanced Settings")

    // 导航：push 新页面时自动出现返回按钮
    nav := container.NewNavigationWithTitle(general, "Settings")

    general.OnTapped = func() {
        nav.Push(advanced)  // 推入新页面，显示返回按钮
    }
    return nav
}
```

## 12. Canvas 样式模式

### 12.1 Rectangle 圆角与 aspect ratio

```go
// 胶囊形（pill）按钮背景
bg := canvas.NewRectangle(theme.Color(theme.ColorNamePrimary))
bg.CornerRadius = 20  // 设为高度一半 ≈ pill

// 仅顶部圆角
bg.TopLeftCornerRadius = 8
bg.TopRightCornerRadius = 8

// 固定宽高比
bg.Aspect = 1.6  // 宽度自动 = 高度 × 1.6
```

### 12.2 Image 填充模式

```go
// 头像：Cover 裁剪填满
avatar := canvas.NewImageFromFile("avatar.jpg")
avatar.FillMode = canvas.ImageFillCover
avatar.SetMinSize(fyne.NewSquareSize(48))

// 缩略图：Contain 适配不裁剪
thumb := canvas.NewImageFromFile("photo.jpg")
thumb.FillMode = canvas.ImageFillContain
thumb.SetMinSize(fyne.NewSize(200, 150))
```

## 13. Label 可选文本

v2.6+ 支持用户选中和复制 Label 文字：

```go
label := widget.NewLabel("Selectable text")
label.Selectable = true  // 用户可框选、Ctrl+C 复制

// 获取选中文本
selected := label.SelectedText()
```

## 14. RowWrap 布局

v2.7+ RowWrap 水平排列子元素，空间不够时自动换行，类似 CSS flex-wrap：

```go
// 配合 GridWrap container 使用（内部已用 RowWrap）
tags := container.NewGridWrap(fyne.NewSize(80, 30),
    widget.NewButton("Go", nil),
    widget.NewButton("Rust", nil),
    widget.NewButton("Python", nil),
    widget.NewButton("TypeScript", nil),
)

// 手动使用 RowWrap layout
c := container.New(layout.NewRowWrapLayout(), objs...)
// 或带自定义间距
c := container.New(layout.NewRowWrapLayoutWithCustomPadding(8, 4), objs...)
```
