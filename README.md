# PuzzleCat 创作者 Skill 集合

[![skills.sh](https://skills.sh/b/EternalIllusion/PuzzleCat-Skills)](https://skills.sh/EternalIllusion/PuzzleCat-Skills)

面向 [PuzzleCat](https://puzzle.cat) 平台创作者（出题人）的 AI 编码代理 skill 集合。每个 skill 是一个独立文件夹，在非商用情况下可直接复制到其它仓库（请注意开源协议）、或装入 Trae / Cursor 等支持 skill 格式的 AI 工具，让 AI 在创作过程中按平台规范行事。

> [!WARNING]
> 此项目为 [PuzzleCat 平台](https://puzzle.cat) 彼时的开发者 [EterIll](https://github.com/EternalIllusion) 个人维护的项目，不与 PuzzleCat 平台官方相关。PuzzleCat 平台不为本项目担保，本项目也不承诺与平台服务的更新同步。本项目的更新时间戳等不能作为针对 PuzzleCat 平台服务的相关依据。本仓库内容可能与平台内部实现存在差异，请以平台当前公布的规则与配额为准。

## Skill 一览

| Skill | 用途 | 触发场景 |
|-------|------|----------|
| [puzzlecat-interactive](./puzzlecat-interactive/) | 开发可上传的交互题包（HTML / ZIP） | 制作交互题、使用 `PuzzleHuntSandbox`、跨题状态联动、沙盒 iframe 题包打包 |
| [puzzlecat-content](./puzzlecat-content/) | 撰写题面富文本、答案 JSON、解锁规则与成就条件 | 编写题面 HTML、配置中间答案与 reveal 解锁、编辑解锁规则 AST |
| [puzzlecat-security-review](./puzzlecat-security-review/) | 对交互题材料做安全性自查并输出结构化自查报告 | 安全自查、投稿前检查、排查答案泄露与交互包风险 |

三个 skill 覆盖一条完整创作链路：**写题面（puzzlecat-content）→ 做交互（puzzlecat-interactive）→ 安全性自查（puzzlecat-security-review）**。

## 安装

### 方式一：让 AI 代理代装

无法运行 npx 时，对 AI 代理说一句话即可：

```text
请从 https://github.com/EternalIllusion/PuzzleCat-Skills 安装全部 skill。
```

### 方式二：CLI 一键安装（推荐）

使用 [skills.sh](https://skills.sh/) 生态的 `skills` CLI（支持 Trae / Trae CN / Claude Code / Cursor 等 80+ 编码代理，会自动检测你已安装的代理）：

```bash
# 安装全部三个 skill（默认装到当前项目）
npx skills add EternalIllusion/PuzzleCat-Skills

# 只装一个、装到全局用户目录（-g）、免确认（-y）
npx skills add EternalIllusion/PuzzleCat-Skills --skill puzzlecat-interactive -g -y

# 先看看仓库里有哪些 skill
npx skills add EternalIllusion/PuzzleCat-Skills --list
```

后续管理：`npx skills list` 查看、`npx skills update` 更新、`npx skills remove` 卸载。

## 目录格式

遵循通用 skill 约定：

```text
<skill-name>/
├── SKILL.md
└── references/
```

- `name` 与文件夹名一致，kebab-case。
- `description` 含「做什么 + 何时触发 + 不适用场景」，是路由触发依据。
- `license` 声明授权协议（本仓库为 `CC-BY-NC-SA`）。
- 正文为命令式指令，只写模型不知道的平台特定规则（配额、白名单、输出格式等）。

## 相关链接

- [PuzzleCat 平台](https://puzzle.cat)——平台主站。
- [交互题 SDK](https://puzzle.cat/hunt-sandbox-sdk.js)——`puzzlecat-interactive` 要求包内引入的脚本，可直接下载阅读。

## 授权协议

本仓库（PuzzleCat 创作者 Skill 集合，作者 EterIll，来源 <https://github.com/EternalIllusion/PuzzleCat-Skills>）采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 授权，完整法律文本见 [LICENSE](./LICENSE)。

你可以自由复制、分发、改编本仓库内容，但须同时满足三个条件：

- **署名（BY）**：注明作者 EterIll 并附上协议链接。
- **非商业性使用（NC）**：不得用于商业目的。
- **相同方式共享（SA）**：改编作品须以相同协议分发。