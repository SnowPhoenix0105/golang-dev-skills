---
name: mockey-cn
description: |
  mockey 运行时函数/变量 mock 库。当需要在 Go 单元测试中对函数、方法或变量进行 Patch 时使用此技能。
  触发场景：提到 mock、mockey、打桩、Patch、单元测试 mock、函数替换、Mock、PatchConvey、PatchRun、MockValue 等。
---

# mockey 单元测试 Mock 指南

mockey（`github.com/bytedance/mockey`）是一个 Go 运行时 mock 库，通过重写函数指令实现函数和变量的替换。不需要将所有依赖指定为接口类型即可进行 mock。

> **前提**：必须在编译时禁用内联和编译优化，使用 `go test -gcflags="all=-l -N"`。

## 适用场景

- 单元测试中需要 mock 外部函数、方法或变量，且无法或不便使用接口注入
- 需要按条件返回不同值（条件 mock）或按顺序返回不同值（序列 mock）
- 需要在 mock 的同时执行原始逻辑（装饰器模式）

## Mock 策略优先级

选择 mock 方式时，按以下优先级：

1. **库/框架提供的专门 mock 能力** — 如果依赖的库或类型本身提供了 mock 接口或测试工具（如 `httptest`、`redismock` 等），优先使用。
2. **mockey 作为兜底** — 当依赖库没有提供 mock 能力，或不方便注入接口时，使用 mockey 进行运行时 Patch。

完整 API 参考见 `references/api-reference.md`。

## 生命周期管理

mockey 提供两种自动管理 mock 生命周期的函数，在作用域结束时自动解除该作用域内的所有 mock：

### PatchConvey

包装 `goconvey.Convey`，在 Convey 块结束时自动释放 mock。**适用于项目中已使用 goconvey 框架的场景**，可以替换原 `Convey` 调用。

```go
import (
    "testing"
    "github.com/bytedance/mockey"
    c "github.com/smartystreets/goconvey/convey"
)

func TestExample(t *testing.T) {
    mockey.PatchConvey("描述", t, func() {
        mockey.Mock(targetFunc).Return(mockValue).Build()
        // mock 在此处生效
        c.So(result, c.ShouldEqual, expected)
    })
    // mock 已自动释放
}
```

### PatchRun

轻量级作用域，**不依赖 goconvey**，仅在函数结束后自动释放 mock。

```go
func TestExample(t *testing.T) {
    mockey.PatchRun(func() {
        mockey.Mock(targetFunc).Return(mockValue).Build()
        // mock 在此处生效
    })
    // mock 已自动释放
}
```

### 选择建议

| 场景 | 推荐 |
| :--- | :--- |
| 项目已使用 goconvey（`Convey` + `So` 断言） | `PatchConvey` |
| 项目使用标准 testing 或其他断言库 | `PatchRun` |
| 需手动控制 mock 生命周期 | `Build()` + `defer mocker.UnPatch()` |

两种方式都支持嵌套使用，每层只释放自己内部的 mock。

### 手动释放

如果不使用 `PatchConvey` / `PatchRun`，mockey 创建的 patch **不会自动解除**，必须手动 `UnPatch`：

```go
mocker := mockey.Mock(targetFunc).Return(mockValue).Build()
defer mocker.UnPatch()
```

**无论是否使用 mockey 的生命周期函数，非 mockey 创建的 mock/patch（如其它 mock 库、手动替换的变量等）都必须自行释放。**

## 编译参数

使用 mockey 必须禁用内联和编译优化：

```bash
go test -gcflags="all=-l -N" ./...
```

- **Goland**：Run/Debug Configurations > Go tool arguments 中添加 `-gcflags="all=-l -N"`
- **VSCode**：`settings.json` 中添加 `"go.buildFlags": ["-gcflags=all=-N -l"]`

## 代码风格

**禁止使用 dot import**。mockey 自身的 README 大量使用 `import . "github.com/bytedance/mockey"`，但在本技能生成的代码中，统一使用 `mockey.` 前缀；goconvey 使用别名 `c`：

```go
import (
    "github.com/bytedance/mockey"
    c "github.com/smartystreets/goconvey/convey"
)
```

以上是本技能的默认风格。如果用户有明确要求、其它 skill 有明确约束，或仓库中已有单元测试有既定风格，以那些要求为准。

## 排障

常见问题速查，详见 `references/troubleshooting.md`：

| 现象 | 常见原因 |
| :--- | :--- |
| mock 不生效 | 未加 `-gcflags="all=-l -N"`；未调用 `Build()`；目标函数签名不匹配 |
| "re-mock" panic | 同一函数在同一个 PatchConvey/PatchRun 中重复 mock |
| "function is too short to patch" | 函数太短或未禁用内联，尝试 `MockUnsafe` |
| "signal SIGBUS" (M 系列 Mac) | arm64 偶发问题，重试或升级到最新版 |

完整排障指南见 `references/troubleshooting.md`。

## 兜底

当本技能和参考资料无法解决问题时，**向用户报告具体情况并寻求帮助**，不得静默猜测。报告内容：
- 遇到的错误信息和堆栈
- mock 的目标函数/变量
- 使用的 mockey 版本和 Go 版本
