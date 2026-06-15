# 中文版 Skill 审计报告

> 审计时间：2025-07  
> 审计范围：`golang-dev-plugin-cn/skills/` 下全部 5 个技能  

## 一、总览

| 技能 | SKILL.md 大小 | references 文件数 | references 总大小 |
| :--- | :--- | :--- | :--- |
| fyne-gui-dev-cn | ~20 KB | 6 | ~100 KB |
| bubbletea-tui-dev-cn | ~17 KB | 7 | ~66 KB |
| golang-guideline-cn | ~12 KB | 0 | 0 |
| gorm-sql-dev-cn | ~10 KB | 8 | ~66 KB |
| mockey-ut-cn | ~5 KB | 2 | ~11 KB |

---

## 二、按技能逐一分析

### 2.1 golang-guideline-cn — 结构不一致，缺少 references 分层

**问题 1：与其余 4 个技能结构不一致**

其余 4 个技能全部采用"SKILL.md（精简索引）+ references/（详细内容）"的渐进式披露模式。golang-guideline-cn 是唯一把所有内容塞进单个 SKILL.md 的技能，没有 `references/` 目录。

**严重程度**：中。12KB 不至于无法加载，但违背了 CLAUDE.md 中规定的"SKILL.md 保持精简，只保留核心信息和索引，较长内容放到 references/ 文件中"原则。

**问题 2：排障覆盖面极窄**

故障排查只有"3.1 Go 版本不匹配"一个场景。相比之下，其余技能都有独立的 `troubleshooting.md`，覆盖 8-15 个常见问题。golang-guideline 作为"打底"通用技能，理应覆盖更多场景（如 module 冲突、cgo 编译问题、race detector 误报、GC 调优等）。

**严重程度**：中高。排障指南是技能的核心价值之一，当前过于单薄。

**问题 3：缺少测试指导**

虽然 1.2 节提到了"完成模块后编写单元测试"，但只是流程性要求，缺少怎么写、用什么库、常见测试模式（table-driven 之外还有 subtest、golden file、mock 策略选择等）的指导。

**严重程度**：低。其他技能（fyne、bubbletea、mockey）各自覆盖了测试，golang-guideline 的定位是通用指南，测试覆盖可以保持简洁。

---

### 2.2 mockey-ut-cn — 引用链断裂 + 接口 Mock 内容缺失

**问题 1：接口 Mock（`mockey/exp/iface`）内容缺失——这是最严重的问题**

`references/troubleshooting.md` 第 157-163 行列出了 mock 接口的三种方式，推荐使用 `mockey/exp/iface` 包，但：

- `references/api-reference.md` 完全没有覆盖 `mockey/exp/iface`
- `SKILL.md` 也没有接口 mock 的内容
- troubleshooting 末尾说"详见 SKILL.md 中的 API 参考"，但 SKILL.md 第 27 行只说"完整 API 参考见 `references/api-reference.md`"

形成断裂链：**troubleshooting → SKILL.md → api-reference.md →（内容不存在）**

**严重程度**：高。用户按排障指南的提示去查 API 参考，但找不到任何关于接口 mock 的 API 文档。这是一个真正的功能缺口。

**问题 2：内容覆盖度过低**

mockey 只有 2 个 reference 文件（api-reference + troubleshooting），总计约 11KB，是 5 个技能中最少的。相比之下，同等复杂度的技能（bubbletea、gorm）都有 6-8 个 reference 文件。缺少：

- 最佳实践 / 使用模式（如 mock 策略选择决策树、多层 mock 的组织方式）
- 与 `golang-guideline-cn` 中测试要求（2.1 节"优先使用库/框架专门 mock 能力"）的交叉引用

**严重程度**：中低。可能是有意保持精简（mockey 本身 API 面不大），但 troubleshooting 中暴露的内容缺口需要修复。

**问题 3：`references/troubleshooting.md` 指向 SKILL.md 而非直接指向 api-reference.md**

第 163 行 `详见 SKILL.md 中的 API 参考` 应直接指向 `references/api-reference.md`，减少一次跳转。

**严重程度**：低。可用性问题，非功能性缺陷。

---

### 2.3 fyne-gui-dev-cn — SKILL.md 过大，section 编号引用脆弱

**问题 1：SKILL.md 约 20KB，接近"不够精简"的临界点**

虽然结构清晰、渐进式披露做得不错，但仍然包含了大量可直接移入 references 的细节：

- 完整的对话框 API 列表（14 行代码块 + 列表）
- 完整的菜单与快捷键示例（~15 行）
- 完整的系统托盘代码（~10 行）
- 完整的编译部署命令（~40 行，含桌面/移动/Web/交叉编译 4 个平台）

这些内容在 SKILL.md 中属于"完整细节"而非"精简索引"。建议将对话框 API 列表、系统托盘生命周期管理、编译部署细节分别移入对应的 references，SKILL.md 只保留 2-3 行提示 + 引用。

**严重程度**：中。不影响功能，但影响加载效率和 agent 的注意力分配。

**问题 2：对 `references/best-practices.md` 的章节编号引用是隐式且脆弱的**

SKILL.md 中多处写"见 `references/best-practices.md` 第 15 节"、"第 19 节"等。`best-practices.md` 确实有编号章节（"15. i18n 翻译"、"19. 系统托盘生命周期"），但这些编号只是 markdown 标题中的数字，没有锚点机制。如果将来在 best-practices.md 中增删章节，所有编号引用都会静默断裂。

**建议**：改为使用章节名称引用（如"见 `references/best-practices.md` i18n 翻译章节"），或给每个章节加显式锚点。

**严重程度**：中。当前引用是正确的，但维护风险高。

**问题 3：references 之间的交叉引用使用相对路径**

`references/troubleshooting.md` 和 `references/best-practices.md` 中都存在对同目录下其他 reference 文件的引用（如"见 `references/deployment.md`"），这是好的做法。但需确保引用链不形成循环。

**严重程度**：低。当前没有发现循环引用。

---

### 2.4 gorm-sql-dev-cn — 整体质量最好，小问题

**问题 1：缺少独立的 testing.md**

GORM 有丰富的测试支持（`gorm.io/gorm/tests` 包、`AssertEqual` 辅助、SQLite 内存数据库测试模式），但当前技能没有独立的测试指导。`model-crud.md` 中涉及 CRUD 但不涉及如何写 GORM 的单元测试。

**严重程度**：中低。gorm 的测试主要是常规 Go 测试 + SQLite 内存库，不一定需要独立 reference，但作为一个"全功能 ORM"技能，测试指导是有价值的补充。

**问题 2：缺少 best-practices.md**

gorm 有大量最佳实践话题（N+1 预防、零值处理策略、连接池配置、Preload vs Joins 选择、软删除设计模式等），目前这些分散在 `model-crud.md`、`advanced-queries.md`、`troubleshooting.md` 中，缺少一个集中式的"最佳实践"reference。

**严重程度**：低。分散覆盖不等于没有覆盖，但集中式 best-practices 会更方便检索。

**问题 3：SKILL.md 中的链式 API 和终结 API 表格是纯方法名列表**

两个大表格列出了方法名和一句话用途，但没有参数签名。用户看完表格仍然需要跳转到 `references/model-crud.md` 才能知道具体用法。如果表格改为保留最常用的 5-8 个方法 + "完整 API 见 references"，可以显著缩短 SKILL.md。

**严重程度**：低。当前做法也可以理解为"索引密度合理"。

---

### 2.5 bubbletea-tui-dev-cn — 结构合理，细节可优化

**问题 1：SKILL.md 约 17KB，可以更精简**

Program 选项列表（~15 行）、完整的鼠标处理代码（~15 行）、按键处理三种方式（~25 行）都在 SKILL.md 中。这些可以部分或全部移入 `references/core-concepts.md`。

**严重程度**：中低。与 fyne 类似的问题但程度稍轻。

**问题 2：Bubbles 组件表缺少用法提示**

SKILL.md 中 Bubbles 组件表列出了 12 个组件及其包路径和一句话用途，但缺少"什么时候用哪个"的选择指南。这点 `references/components.md` 可能覆盖了，但 SKILL.md 中的表格本身可以加一列"适用场景"。

**严重程度**：低。功能上没问题，属于锦上添花。

**问题 3：测试部分引用 `github.com/charmbracelet/x/exp/teatest`**

SKILL.md 第 373 行提到 `// 可在 go.mod 中添加：github.com/charmbracelet/x/exp/teatest`。这个包在 `x/exp` 下，属于实验性包，API 可能变化。建议加一个稳定性提示。

**严重程度**：低。实验性包的引用在 TUI 生态中很常见，但应提醒用户注意 API 稳定性。

---

## 三、跨技能共性问题

### 3.1 结构模式不统一

| 参考文件类型 | fyne | gorm | bubbletea | mockey | golang |
| :--- | :--- | :--- | :--- | :--- | :--- |
| SKILL.md + references | ✅ | ✅ | ✅ | ✅ | ❌ |
| best-practices.md | ✅ | ❌ | ✅ | ❌ | ❌ |
| troubleshooting.md | ✅ | ✅ | ✅ | ✅ | ❌ |
| testing.md | ✅ | ❌ | ✅ | ❌ | ❌ |
| api-reference.md | ✅ | 分散在多个文件 | 分散在多个文件 | ✅ | N/A |

**建议**：不强求所有技能有相同结构，但应确保每个技能至少有：
1. `SKILL.md`（精简索引 + 核心原则 + 最小示例 + 兜底）
2. `references/troubleshooting.md`（排障指南）

目前 golang-guideline 缺少第 2 项。

### 3.2 测试指导覆盖不一致

- fyne 和 bubbletea 有独立的 `testing.md`，覆盖框架特定的测试 API
- gorm 的测试依赖标准 Go 测试 + SQLite，内容分散在 `model-crud.md` 中
- mockey 本身就是测试工具，不需要独立 testing.md
- golang-guideline 作为通用指南，其测试要求（1.2 节）过于简略

**建议**：golang-guideline 可以增加一个简洁的测试模式 reference（table-driven、subtest、golden file、mock 策略优先级），并与 mockey 技能交叉引用。

### 3.3 兜底格式基本统一，但 golang-guideline 缺少错误报告模板

其余 4 个技能都在末尾有明确的兜底指令，且给出了应向用户报告的具体内容清单（版本号、错误信息、代码片段等）。golang-guideline 的兜底（第 4 节）只有行为要求，没有具体的报告模板。

**严重程度**：低。行为要求本身是完整的。

---

## 四、修复优先级建议

### 🔴 高优先级

1. **mockey-ut-cn：补充 `mockey/exp/iface` 接口 mock 文档**  
   在 `references/api-reference.md` 中增加"接口 Mock"章节，或新建 `references/interface-mock.md`，并修复 troubleshooting → SKILL.md → api-reference.md 的断裂引用链。

### 🟡 中优先级

2. **golang-guideline-cn：建立 references/ 目录，拆分内容**  
   至少增加 `references/troubleshooting.md`，覆盖更多 Go 开发常见故障场景（module 冲突、cgo 编译、race detector、GC 调优等）。

3. **fyne-gui-dev-cn：精简 SKILL.md**  
   将对话框 API 完整列表、部署命令细节等内容移入 references，将 SKILL.md 控制在 12KB 以内。

4. **所有技能：改用章节名称引用替代隐式编号引用**  
   fyne 的 `best-practices.md` 按编号命名的章节引用有维护风险。

### 🟢 低优先级

5. **gorm-sql-dev-cn：考虑新增 `best-practices.md`**  
   将分散在多个文件中的最佳实践话题集中整理。

6. **mockey-ut-cn：修复 troubleshooting 中指向 SKILL.md 而非直接指向 api-reference.md 的引用**

7. **bubbletea-tui-dev-cn：给 teatest 包引用加实验性提示**

8. **统一内部引用风格**：所有 reference 之间的交叉引用统一使用 `` `references/xxx.md` `` 格式（当前已基本一致，仅 mockey 有例外）。
