# golang-dev-skills

Go 语言开发相关的 Agent Skills 集合，遵循 `agentskills.io` 开放标准，可被多种 Agent 工具加载使用。

## 可用 Skills

| Skill | 说明 |
| :--- | :--- |
| `golang-guideline` | Go 语言开发指南（英文版）— 开发流程、代码风格、故障排查 |
| `golang-guideline-cn` | Go 语言开发指南（中文版） |
| `fyne-gui-dev` | Fyne 跨平台 GUI 应用开发（英文版） |
| `fyne-gui-dev-cn` | Fyne 跨平台 GUI 应用开发（中文版） |
| `mockey-ut` | mockey 单元测试 mock 库（英文版） |
| `mockey-ut-cn` | mockey 单元测试 mock 库（中文版） |

## 接入你的 Agent

选择你使用的 Agent 工具，按对应步骤操作。

仓库地址：`https://github.com/SnowPhoenix0105/golang-dev-skills`

---

### Claude Code

**添加仓库：**

在 Claude Code 终端运行：
```
/plugin marketplace add SnowPhoenix0105/golang-dev-skills
```
等待确认 "Marketplace added"。

**安装 Skills：**

```bash
# 安装单个 skill
/plugin install <skill-name>@SnowPhoenix0105-golang-dev-skills

# 例如
/plugin install golang-guideline-cn@SnowPhoenix0105-golang-dev-skills
```

**更新 Skills：**

```bash
# 检查更新
/plugin marketplace update SnowPhoenix0105/golang-dev-skills --check

# 升级特定 skill
/plugin upgrade --plugin <skill-name>

# 升级全部
/plugin upgrade --all
```

---

### OpenAI Codex

**添加仓库：**

```bash
codex marketplace add https://github.com/SnowPhoenix0105/golang-dev-skills
```

验证：`codex marketplace list`

**安装 Skills：**

```bash
# 安装单个 skill
codex plugins install <skill-name>

# 例如
codex plugins install golang-guideline-cn
```

如需批量安装，可从 `marketplace.json` 读取列表后循环执行。

**更新 Skills：**

```bash
# 升级特定 skill
codex plugins update <skill-name>

# 升级全部
codex plugins update --all
```

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

# 安装全部 skills
openclaw plugins install ./golang-dev-skills/skills/golang-guideline
openclaw plugins install ./golang-dev-skills/skills/golang-guideline-cn
openclaw plugins install ./golang-dev-skills/skills/fyne-gui-dev
openclaw plugins install ./golang-dev-skills/skills/fyne-gui-dev-cn

# 或只安装需要的 skill
openclaw plugins install ./golang-dev-skills/skills/<skill-name>
```

**更新 Skills：**

```bash
# 拉取最新代码
cd golang-dev-skills && git pull

# 新 skill 需要手动添加
openclaw plugins install ./skills/<new-skill>
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
