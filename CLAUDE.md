# CLAUDE.md

本文件为 Claude Code（claude.ai/code）在本仓库中工作时提供指导。

## 仓库用途

本仓库收录 Agent Skills（主要面向 Go 开发），遵循 `agentskills.io` 开放标准。技能在此处编写，可被 Claude Code、Codex、CC Switch、OpenClaw、Hermes 等主流 Agent 工具加载。

`docs/` 存放进行中的调研和设计笔记，不随技能一起部署，是作者的草稿空间。

## 仓库结构

```
golang-dev-skills/
├── skills/                     # 标准 Skills 目录（agentskills.io 规范）
│   ├── <skill-name>/           # 一个目录 = 一个 skill，目录名必须与 SKILL.md 中 name 一致
│   │   ├── SKILL.md            # 核心文件，YAML frontmatter + Markdown
│   │   ├── references/         # 较长的参考资料（可选）
│   │   ├── scripts/            # 脚本（可选）
│   │   └── assets/             # 模板、静态资源（可选）
│   └── ...
├── .claude-plugin/             # Claude Code 适配层
│   ├── plugin.json
│   └── marketplace.json
├── .agents/plugins/            # OpenAI Codex 适配层
│   └── marketplace.json
├── .openclaw-plugin/           # OpenClaw 适配层
│   └── marketplace.json
├── .github/workflows/          # CI 校验
│   └── skills-validation.yml
└── CLAUDE.md                   # 本文件
```

> CC Switch 和 Hermes 无需仓库内适配文件，它们自动扫描 `skills/` 目录发现 skill。

## Skill 命名规则（agentskills.io）

- 1-64 字符，仅限小写字母、数字和连字符
- 不能以连字符开头或结尾
- 不能包含连续连字符（如 `my--skill` 无效）
- **必须与父目录名一致**

## 技能编写流程

创建或修改技能时，按以下顺序进行：

1. **调研** — 联网搜索相关经验分享、最佳实践和官方文档。如内容较多，将笔记保存到 `docs/`。
   - 如果是创建某个库的使用技能，调研阶段需要阅读该库的**源代码**：
     - 用户给出了本地源代码目录 → 直接阅读该目录下的代码
     - 未给出 → 优先从 `go mod cache` 中寻找（`go env GOMODCACHE`）
     - 仍找不到 → 引导用户提供源代码路径
2. **编写中文版** — 默认先处理中文版。SKILL.md 保持精简，只保留核心信息和索引，较长内容放到 `references/` 文件中。
   - 库的使用技能，按库的复杂度决定覆盖范围：
     - **API 较简单**：核心概念、核心 API、排障指南、注意事项
     - **API 较复杂**：核心概念与整体架构、核心类型与核心 API、示例、测试支持（如有）、排障指南
   - 以上是技能需要涵盖的**元素**，并非要一字不差地作为目录结构。注意渐进式披露，不要把全部内容堆在 SKILL.md 里。
3. **检查** — 写完后，逐项检查：
   - **渐进式披露**：SKILL.md 应只包含要点和索引，每个精简章节必须明确指向对应的 reference 文件（如"完整列表见 `references/xxx.md`"）。较长的 reference 文件必须有目录/索引。确认所有内部引用的文件（references、docs 等）实际存在。
   - **上下文独立**：假设技能加载时没有创建该技能时的任何上下文。不得硬编码绝对路径（如 `/Users/xxx/`）——应使用动态发现命令（`go list -m`、`GOMODCACHE`）。如需引用源码位置，在 SKILL.md 中建立一次"包路径 → 文件系统"映射规则，后续用包路径表示，不得在 reference 中重复硬编码路径。
   - **兜底与排障**：SKILL.md 末尾必须包含显式指令——当技能参考资料和源码探索都无法解决问题时，向用户报告具体情况并寻求帮助，不得静默猜测。
4. **更新 marketplace 清单** — 新增、删除、重命名 skill 时，**必须同步更新**以下所有 marketplace 清单文件：
   - `.claude-plugin/marketplace.json`
   - `.agents/plugins/marketplace.json`
   - `.openclaw-plugin/marketplace.json`

   各文件格式：
   | 文件 | 所属工具 | 格式要点 |
   | :--- | :--- | :--- |
   | `.claude-plugin/marketplace.json` | Claude Code | `plugins` 数组，每项含 `name`、`source`、`category` |
   | `.agents/plugins/marketplace.json` | OpenAI Codex | `plugins` 数组，每项含 `source`（含 `source`/`path`）、`policy`、`category` |
   | `.openclaw-plugin/marketplace.json` | OpenClaw | `skills` 数组，每项为 `./skills/<name>` 相对路径字符串 |
5. **翻译为英文** — 中文版定稿后，翻译为英文版（无 `-cn` 后缀）。尽量保持含义对应。用户未指定语言时，默认先处理中文版。

## 语言策略

- 技能同时存在中文版（`-cn` 后缀）和英文版（无后缀）。
- 用户未指定语言时，先创建/修改中文版，再同步英文版。
- 翻译应保持含义对应，而非逐字直译。技术术语保留英文（如 "Widget"、"Renderer"、"binding"）。

## Manifest 文件位置速查

| 工具 | 适配文件 | 发现机制 |
| :--- | :--- | :--- |
| Claude Code | `.claude-plugin/marketplace.json` | `/plugin marketplace add` |
| OpenAI Codex | `.agents/plugins/marketplace.json` | `codex marketplace add` |
| OpenClaw | `.openclaw-plugin/marketplace.json` | `openclaw plugins install` |
| CC Switch | 无需适配文件 | GitHub 仓库扫描 |
| Hermes | 无需适配文件 | `hermes skills install` |
