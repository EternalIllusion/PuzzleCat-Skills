---
name: puzzlecat-interactive
description: 按 PuzzleCat 交互题 SDK 规范开发可上传的互动题包（HTML 或 ZIP）。当用户要为 PuzzleCat 制作交互题、使用 PuzzleHuntSandbox、做跨题状态联动、或打包沙盒 iframe 题包时使用。不适用于与 PuzzleCat 平台无关的普通网页开发。
license: CC-BY-NC-SA
---

# PuzzleCat 交互题包开发

> **免责声明**：本 skill 为社区整理，非平台官方文档；规则以平台当前公布为准，可能与平台内部实现存在差异。

为 PuzzleCat 平台开发「交互题」——以沙盒 iframe 运行的互动题包。完整文档见 [references/INTERACTIVE_SDK.md](references/INTERACTIVE_SDK.md)，动手前先通读。更多有关 API 格式等详细信息，请下载并阅读 [hunt-sandbox-sdk.js](https://puzzle.cat/hunt-sandbox-sdk.js)。

## 硬性规则

1. **引入 SDK**：包内加一行 `<script src="/hunt-sandbox-sdk.js"></script>`，之后用全局对象 `PuzzleHuntSandbox`。服务端读写一律通过 SDK 完成；**禁止**在包内直连平台后端、携带登录凭据或把玩家数据发往第三方。
2. **包内无机密**：包文件对已解锁题目的玩家完全可见。不得写入正确答案明文、未购买提示正文。需要在包内本地校验答案时，用**哈希存储**——把用户输入哈希后与存储的哈希比对，不落明文。
3. **打包**：ZIP 根目录必须含 `index.html`（或仅一层文件夹包裹）；交互包 ≤15MB、图片 ≤10MB；macOS 打包排除 `.DS_Store`、`__MACOSX`。双击本地打开 `index.html` 应无白屏、控制台无报错。
4. **三种宽度自适应（作者责任）**：同一份包会在沉浸阅读（近全宽）、并排阅读（约半宽）、手机（≈375px 及更窄）下运行，平台不注入 viewport、不缩放画布。必须：
   - `<meta name="viewport" content="width=device-width, initial-scale=1">`
   - 流体布局（`%` / `vw` / `max-width: 100%`），禁止写死 `800px`/`1024px` 画布
   - 点击与拖拽同时支持触摸（不能只监听 `mousedown`/`mousemove`）；可点击目标 ≥44px
   - 三种宽度 + 横竖屏下均可操作与阅读、无内部滚动条
5. **高度**：平台自动测量内容高度并调整 iframe，一般无需处理；仅自动测量失效时手动调 `reportHeight()`。

## API 速查（全局对象 PuzzleHuntSandbox）

| 方法 | 用途 |
|------|------|
| `submitProgress(data)` | 上报一次进度，玩家端弹「交互题进度已记录」提示；**仅前端反馈，不保存到服务端** |
| `getContext()` | 读比赛上下文：Hunt 结构（已按解锁过滤）、队伍进度、已解题目答案、跨题状态快照 |
| `onContext(fn)` | 订阅上下文注入；已注入则立即回调 |
| `useTeamState(key)` | 全队共享的跨题状态句柄 |
| `usePlayerState(key)` | 队员个人的跨题状态句柄 |
| `getAnswerHistory(puzzleId, cb)` | 本队作答历史；键为标准化答案，值含 `resp`（判题回复）与 `hit`（`'canonical'`/`'intermediate'`，省略即未命中） |
| `log(msg)` | 输出日志到父页面（题面页）控制台，保存失败的错误也打在这里 |

`getContext()` 典型用途：按 `hunt.progress.solvedPuzzles` 决定交互走向（跨题联动）、用 `hunt.solvedAnswers` 做「回顾已解答案」类谜题。

## 跨题状态铁律

状态句柄均有 `get` / `set` / `reload` / `subscribe`（注册即回调一次，返回退订函数）。

- **先读后写**：状态按 key **整值覆盖**，不是合并写入。写前必须先读最新值（`subscribe` 回调或 `reload()` 后的值），本地合并后再 `set`，否则会用旧值覆盖队友刚写的数据。
- **无实时推送**：队友写入后，其它成员要刷新或进入题面才能读到。按「下一题见」式传递设计，勿做实时协作白板。
- **配额**：key 须以字母数字开头、只含字母数字与 `_` `.` `:` `-`；值须可 JSON 序列化。**建议把单条值与单个存储空间的累计用量都控制在 128KB 量级以内**，单次写入条数宜少不宜多。**超限静默失败**——页面不报错、玩家无感知，只在控制台留下日志。设计阶段就按这个量级预算，每 key 只存一类数据、减小单值体积。
- 独立谜题页、资源库试玩中跨题状态通道不可用；完整能力仅 Hunt 题目页提供。

## 工作方式

1. 先读 `references/INTERACTIVE_SDK.md`，再根据用户需求确定谜题机制用到哪些 API。
2. 生成包代码时直接产出完整可运行的 `index.html`（含 viewport meta、SDK 引入、流体布局样式）。
3. 用到跨题状态时，默认按「先读后写 + 整值覆盖」模式写代码，并在注释里标明 key 的用途与预期体积。
4. 交付前逐项过 references 文档末尾的「上线前自测清单」，并明确提示用户：三种宽度自测、testsolver 试玩、包内无机密。

## 配套 skill

同一创作链路中还有两个 skill，如已一并安装，按需加载：

- `puzzlecat-content`——题面 HTML 白名单、答案 JSON、解锁规则 AST。
- `puzzlecat-security-review`——交互题材料安全性自查（答案泄露 / 隐私凭据 / 沙盒风险），输出结构化自查报告。

## 建议（非强制，以用户意愿为准）

建议帮助用户进行本地进度保存相关功能的适配，如果不是Hunt交互题（无法使用跨题状态等），优先帮助用户实现基于本地存储的数据持久化，并询问用户是否需要追加存档导入导出的功能。
