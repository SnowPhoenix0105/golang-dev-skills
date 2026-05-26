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

## 接入你的 Agent

选择你使用的 Agent 工具，按对应步骤操作。

仓库地址：`https://github.com/SnowPhoenix0105/golang-dev-skills`

---

### Claude Code

**添加市场源（CLI）：**

```bash
claude plugin marketplace add SnowPhoenix0105/golang-dev-skills
```

**或在交互式会话中添加：**

```
/plugin marketplace add SnowPhoenix0105/golang-dev-skills
```

**安装 Skills（CLI）：**

```bash
# 安装英文版插件（包含所有英文 skill）
claude plugin install "golang-dev-plugin@golang-dev-skills"

# 安装中文版插件
claude plugin install "golang-dev-plugin-cn@golang-dev-skills"
```

**或在交互式会话中安装：**

```
/plugin install golang-dev-plugin@SnowPhoenix0105-golang-dev-skills
/plugin install golang-dev-plugin-cn@SnowPhoenix0105-golang-dev-skills
```

**更新 Skills：**

```bash
# CLI 模式
claude plugin marketplace update SnowPhoenix0105/golang-dev-skills --check
claude plugin upgrade --all

# 或交互模式
/plugin marketplace update SnowPhoenix0105/golang-dev-skills --check
/plugin upgrade --all
```

---

### OpenAI Codex

**添加市场源：**

```bash
codex plugin marketplace add SnowPhoenix0105/golang-dev-skills
```

验证：`codex plugin marketplace list`

**安装 Skills：**

```bash
# 安装英文版插件（包含所有英文 skill）
codex plugin install "golang-dev-plugin@golang-dev-skills"

# 安装中文版插件
codex plugin install "golang-dev-plugin-cn@golang-dev-skills"
```

**更新 Skills：**

```bash
# 升级全部
codex plugin update --all

# 升级特定插件
codex plugin update "golang-dev-plugin-cn@golang-dev-skills"
```

> Marketplace name `golang-dev-skills` 对应 `.agents/plugins/marketplace.json` 中顶层的 `"name"` 字段。

---

### CC Switch

纯 GUI 操作，无需命令行。

**添加仓库并安装：**

1. 启动应用，导航至 **Skills** 管理页面
2. 选择 **"Add Custom Repository"**
3. 输入仓库 URL：`https://github.com/SnowPhoenix0105/golang-dev-skills`
4. 扫描完成后，勾选需要安装的 Skill，点击 **"Install"**

**更新 Skills：**

- 在 Skills 页面点击 **"Check for Updates"** 或 **"Update All"**
- 也可以单击某个 Skill 进行单独更新

---

### OpenClaw

OpenClaw 主要通过 ClawHub 或本地路径安装。如果仓库尚未发布到 ClawHub，先克隆到本地：

```bash
# 克隆仓库
git clone https://github.com/SnowPhoenix0105/golang-dev-skills.git

# 安装全部 skills（OpenClaw 直接指向插件内的 skills/ 目录）
openclaw plugins install ./golang-dev-skills/golang-dev-plugin/skills/golang-guideline
openclaw plugins install ./golang-dev-skills/golang-dev-plugin/skills/fyne-gui-dev
openclaw plugins install ./golang-dev-skills/golang-dev-plugin/skills/mockey-ut
openclaw plugins install ./golang-dev-skills/golang-dev-plugin-cn/skills/golang-guideline-cn
openclaw plugins install ./golang-dev-skills/golang-dev-plugin-cn/skills/fyne-gui-dev-cn
openclaw plugins install ./golang-dev-skills/golang-dev-plugin-cn/skills/mockey-ut-cn

# 或只安装需要的 skill
openclaw plugins install ./golang-dev-skills/<plugin-name>/skills/<skill-name>
```

**更新 Skills：**

```bash
# 拉取最新代码
cd golang-dev-skills && git pull

# 新 skill 需要手动添加
openclaw plugins install ./<plugin-name>/skills/<new-skill>
```

---

### Hermes

**全部导入：**

```bash
hermes skill import --git https://github.com/SnowPhoenix0105/golang-dev-skills.git --namespace golang-dev
```

**部分导入：**

```bash
# Hermes 不支持从远程仓库挑选单个 skill，需先克隆再按路径导入
git clone https://github.com/SnowPhoenix0105/golang-dev-skills.git temp-skills
hermes skill import --from ./temp-skills/skills/<skill-name> --namespace golang-dev
```

**更新 Skills：**

```bash
# 更新全部
hermes skills update

# 更新单个
hermes skills update <skill-name>

# 重新导入远程仓库（覆盖旧版本）
hermes skill import --git https://github.com/SnowPhoenix0105/golang-dev-skills.git --namespace golang-dev --force
```

---

## 友情链接

- eino skills: [eino-ext](https://github.com/cloudwego/eino-ext)
