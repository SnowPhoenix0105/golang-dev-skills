# golang-dev-skills

Go 语言开发相关的 Agent Skills 集合，遵循 `agentskills.io` 开放标准，可被多种 Agent 工具加载使用。

## 可用 Skills

| Plugin | Skill | 说明 |
| :--- | :--- | :--- |
| `golang-dev-plugin` | `golang-guideline` | Go 语言开发指南（英文版）— 开发流程、代码风格、故障排查 |
| `golang-dev-plugin-cn` | `golang-guideline-cn` | Go 语言开发指南（中文版） |
| `golang-dev-plugin` | `fyne-gui-dev` | Fyne 跨平台 GUI 应用开发（英文版） |
| `golang-dev-plugin-cn` | `fyne-gui-dev-cn` | Fyne 跨平台 GUI 应用开发（中文版） |
| `golang-dev-plugin` | `mockey-ut` | mockey 单元测试 mock 库（英文版） |
| `golang-dev-plugin-cn` | `mockey-ut-cn` | mockey 单元测试 mock 库（中文版） |
| `golang-dev-plugin` | `gorm-sql-dev` | GORM SQL 数据库 ORM 开发（英文版） |
| `golang-dev-plugin-cn` | `gorm-sql-dev-cn` | GORM SQL 数据库 ORM 开发（中文版） |
| `golang-dev-plugin` | `bubbletea-tui-dev` | Bubble Tea 终端 UI (TUI) 应用开发（英文版） |
| `golang-dev-plugin-cn` | `bubbletea-tui-dev-cn` | Bubble Tea 终端 UI (TUI) 应用开发（中文版） |

## 接入你的 Agent

本仓库遵循 `agentskills.io` 开放标准，可通过 **`npx skills`**（Vercel 维护的 Agent Skills 生态 CLI）一键安装，兼容 70+ 种 AI Agent 工具（Claude Code、Codex、Cursor、GitHub Copilot、Windsurf、Gemini CLI 等）。

> 前置要求：**Node.js ≥ 16**。无需全局安装，`npx` 自动拉取最新版 CLI。

仓库地址：`https://github.com/SnowPhoenix0105/golang-dev-skills`

---

### 整体导入（安装全部 skill）

```bash
# 非交互式，一键安装全部 10 个 skill 到所有检测到的 Agent
npx skills add SnowPhoenix0105/golang-dev-skills --all
```

也可以交互式选择：

```bash
npx skills add SnowPhoenix0105/golang-dev-skills
# → 进入交互界面，空格勾选需要的 skill，回车确认
```

---

### 导入单个 skill

```bash
# 安装指定 skill（英文版）
npx skills add SnowPhoenix0105/golang-dev-skills --skill golang-guideline
npx skills add SnowPhoenix0105/golang-dev-skills --skill fyne-gui-dev
npx skills add SnowPhoenix0105/golang-dev-skills --skill mockey-ut

# 中文版
npx skills add SnowPhoenix0105/golang-dev-skills --skill golang-guideline-cn
npx skills add SnowPhoenix0105/golang-dev-skills --skill fyne-gui-dev-cn
npx skills add SnowPhoenix0105/golang-dev-skills --skill mockey-ut-cn

# 或使用 @ 简写
npx skills add SnowPhoenix0105/golang-dev-skills@golang-guideline
```

安装到指定 Agent（如只需装给 Claude Code）：

```bash
npx skills add SnowPhoenix0105/golang-dev-skills --skill golang-guideline -a claude-code -y
```

---

### 整体更新

```bash
# 更新所有已安装 skill 到最新版本
npx skills update

# 跳过交互，自动检测范围
npx skills update -y
```

---

### 更新单个 skill

```bash
# 按名称更新指定 skill
npx skills update golang-guideline
npx skills update fyne-gui-dev-cn

# 检查是否有可用更新（不实际更新）
npx skills check
```

---

### 其他常用命令

```bash
npx skills list            # 查看已安装的全部 skill
npx skills list -g         # 仅查看全局安装的 skill
npx skills find golang     # 按关键词搜索 skill
npx skills remove <name>   # 移除指定 skill
```

---

### Claude Code Plugin 方式（可选）

如果你使用 Claude Code，也可以通过 Plugin 市场安装：

```bash
# 添加市场源
claude plugin marketplace add SnowPhoenix0105/golang-dev-skills

# 安装英文版插件（包含全部英文 skill）
claude plugin install "golang-dev-plugin@golang-dev-skills"

# 安装中文版插件
claude plugin install "golang-dev-plugin-cn@golang-dev-skills"

# 更新
claude plugin marketplace update SnowPhoenix0105/golang-dev-skills --check
claude plugin upgrade --all
```

## 友情链接

- eino skills: [eino-ext](https://github.com/cloudwego/eino-ext)
