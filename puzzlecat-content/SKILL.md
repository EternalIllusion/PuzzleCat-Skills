---
name: puzzlecat-content
description: 按 PuzzleCat 题目编辑器规范撰写题面富文本、答案 JSON、解锁规则与成就条件。当用户编写 PuzzleCat 题面 HTML、配置中间答案与 reveal 解锁、编辑解锁规则 AST 或成就触发条件时使用。不用于与题面内容无关的前端开发。
license: CC-BY-NC-SA
---

# PuzzleCat 题面与答案配置规范

> **免责声明**：本 skill 为社区整理，非平台官方文档；规则以平台当前公布为准，可能与平台内部实现存在差异。

为 PuzzleCat 平台撰写题面内容与答案/解锁配置。产出必须能通过平台的html净化与高级 JSON 校验。

## 题面载体（高级模式）

- **HTML 富文本**：`{ "type": "html", "data": "..." }`；正文与 HTML 源码双模式，保存时经平台 HTML 净化。
- **交互题（iframe）**：`{ "type": "interactive", "data": "https://..." }` 或 `asset:资源ID`；仅高级模式可配置；资源从创作者中心资源库选（ZIP ≤15MB、图片 ≤10MB）；**剧情题不可用**此载体。

## HTML 白名单

允许标签：

> p, br, strong, b, em, i, u, s, strike, del, ul, ol, li, h1–h6, blockquote, pre, code, span, div, a, img, audio, video, source, figure, table, thead, tbody, tr, th, td, font, details, summary

**禁止**（保存时会被净化掉，生成时不要写）：`script`、事件属性（`onclick` 等）、一切未列入白名单的标签与 `style` 内联样式依赖。

针对有较复杂排版需求的题目，建议用户使用图片格式。

## 答案 JSON

| 字段 | 说明 |
|------|------|
| `answers[0].answer` | 主答案 |
| `answers[0].aliases` | 同义答案数组 |
| `answers[0].message` | 答对文案 |
| `intermediateAnswers[]` | 中间答案（`matchType`；可选 `revealKey` + `revealHtml` 解锁追加题面） |
| `intermediateAnswers[].message` | 命中反馈文案；可为**字符串数组** = 多条候选，命中时随机展示一条 |
| `wrongMessage` | 答错文案 |

```json
{
  "answers": [{ "answer": "CAT", "aliases": ["猫"], "message": "正确！" }],
  "intermediateAnswers": [{
    "revealKey": "stage-1-uuid",
    "answer": "A",
    "message": "阶段一通过",
    "revealHtml": "<p>下一段题面…</p>"
  }, {
    "answer": "抽卡",
    "message": ["提示一", "提示二", "提示三"]
  }],
  "wrongMessage": "再想想"
}
```

- 命中带 `revealHtml` 的中间答案后，题面追加显示该 HTML（`unlockedReveals`）；进度按独立题用户 / Hunt 队伍持久化，刷新与跨设备保留；未解锁段不会出现在玩家响应里。
- **Hunt 支线**：同字段写在 `branches[].intermediateAnswers[]`，`revealKey` / `revealHtml` 语义与主轨相同。基础编辑器支线区只配终答，**支线中间答案与 reveal 必须在高级 JSON 中编辑**。

## 解锁规则 AST

高级模式直接编辑 AST；常用 op：`always`、`allSolved`、`anySolved`、`kOfSolved`、`sectionSolvedCount`、`timeAfterStart`、`manual`。

```json
{ "op": "and", "rules": [
  { "op": "allSolved", "puzzleIds": ["p-1"] },
  { "op": "kOfSolved", "k": 2, "puzzleIds": ["p-2","p-3"] }
]}
```

**注意**：Hunt支线、解锁逻辑等仅在Puzzle Hunt题目中有效，需要向用户确认使用场景。

## 成就条件与勋章

成就条件存于 `triggerConfig.rule`，与解锁同款 AST。常用 op：`finishRank`、`huntCleared`、`firstBlood`、`noHintsClear`、`teamSolveCount`、`solveMeta`/`solveFM`、`wrongAnswerUnder`、`timeWindow`、`manual`。

平台勋章每等级 `levels[n].rule`，可选全局 `timeWindow` 与各等级 AND 组合。常用 op：`metricAtLeast`、`metricAtMost`、`huntAchievementEarned`、`activityMetric`、`manual`。

**注意**：平台勋章仅在 Puzzle Hunt 题目中有效，且需先向平台申请。使用前先向用户确认使用场景与申请结果；未通过时提示用户通过平台的意见反馈 / QQ 群咨询。

## 时间窗口 timeWindow

```json
{
  "op": "timeWindow",
  "timezone": "Asia/Shanghai",
  "startAt": "2026-05-01T00:00:00+08:00",
  "endAt": "2026-05-07T23:59:59+08:00",
  "hoursAfterHuntStart": 24,
  "hoursBeforeHuntEnd": 2
}
```

## 工作方式

1. 生成题面 HTML 时只用白名单标签；需要折叠/分段用 `details`+`summary`，不要引入白名单外结构。
2. 生成答案 JSON 时确认主答案唯一、同义答案覆盖常见变体（含大小写与全半角差异由平台归一化处理，不必枚举纯格式变体）。
3. 中间答案用于分阶段推进或里程碑反馈；需要解锁新题面才加 `revealKey`/`revealHtml`，否则只用 `message`。
4. 生成 AST 时先和用户确认题目 ID 的对应关系，不要臆造 `puzzleIds`。

## 配套 skill

同一创作链路中还有两个 skill，如已一并安装，按需加载：

- `puzzlecat-interactive`——交互题包（沙盒 iframe）开发与打包。
- `puzzlecat-security-review`——交互题材料安全性自查（答案泄露 / 隐私凭据 / 沙盒风险），输出结构化自查报告。
