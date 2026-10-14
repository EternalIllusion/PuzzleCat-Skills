# 交互题 SDK 使用文档（创作者版）

引入 SDK 只需一行，**无需下载或申请**：

```html
<script src="/hunt-sandbox-sdk.js"></script>
```

加载后即可使用全局对象 `window.PuzzleHuntSandbox`。

---

## 运行环境

| 项目 | 说明 |
|------|------|
| 高度 | 平台自动测量内容高度并调整 iframe，**无需自行处理**；容器有最小高度，让内容自然撑开即可 |
| 网络 | 交互包没有登录凭据，也**不应直接请求平台后端**；服务端读写一律通过 SDK 完成 |
| 作用域 | 仅在 Hunt 题目页提供完整能力；独立谜题页、资源库试玩中**跨题状态通道不可用** |

> 外链交互题：iframe 直接指向你的域名，`/hunt-sandbox-sdk.js` 会解析到**你自己的域名**，这种情况下需自带该文件。如果你有外站运营需求，请联系平台沟通对接方式。

## 快速开始

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <script src="/hunt-sandbox-sdk.js"></script>
  <style>body { font-family: sans-serif; margin: 16px; }</style>
</head>
<body>
  <button id="next">完成第 1 步</button>
  <script>
    document.getElementById('next').onclick = function () {
      PuzzleHuntSandbox.submitProgress({ step: 1 });
    };
  </script>
</body>
</html>
```

## API 参考

全局对象 `PuzzleHuntSandbox` 提供以下方法：

| 方法 | 说明 |
|------|------|
| `ready()` | 通知父页面「已就绪」；由 SDK 自行处理，一般无需手动调 |
| `reportHeight()` | 手动上报内容高度。iframe高度一般自动同步，仅在极少数自动测量失效的场景兜底 |
| `submitProgress(data)` | 上报一次进度，玩家端会弹出「交互题进度已记录」提示。**仅作交互反馈，不会保存到服务端**；需要持久化请用跨题状态（见「跨题状态通道」一节） |
| `getContext()` | 读取比赛上下文（见「跨题状态通道」一节） |
| `onContext(fn)` | 订阅上下文注入：已注入则立即回调，否则注入时回调 |
| `useTeamState(key)` | 取一个**全队共享**的跨题状态句柄（见「跨题状态通道」一节） |
| `usePlayerState(key)` | 取一个**队员个人**的跨题状态句柄（见「跨题状态通道」一节） |
| `getAnswerHistory(puzzleId, cb)` | 读取本队在某题的作答历史（见「作答历史」一节） |

### 比赛上下文 getContext()

返回结构：

```js
{
  ctx: {
    huntId: '…',     // 当前 Hunt 的 hashid
    puzzleId: '…',   // 当前题目 ID（可能为空）
  },
  hunt: {
    // 本队当前可见的结构与题目（已按解锁状态过滤，锁住的题不会出现）
    structure: { sections: [...], puzzles: [...] },
    // 本队进度：solvedPuzzles 已解题 ID 列表、partialPuzzles 部分解、
    // gatePassed 已通过 Answer Gate 的题、freeTokensRemaining 剩余提示币
    progress: { solvedPuzzles: [...], partialPuzzles: [...], gatePassed: [...], freeTokensRemaining: 0 },
    // 仅包含**本队已解出题目**的答案；未解出的题不会泄露
    solvedAnswers: { 'puzzle-id': '答案原文' },
  },
  states: {
    team:   { key: value, ... },  // 该队全部 team 状态快照
    player: { key: value, ... },  // 当前队员全部 player 状态快照
  },
}
```

典型用途：根据 `progress.solvedPuzzles` 决定交互内容走向（跨题联动）、利用 `solvedAnswers` 做「回顾已解答案」类谜题。

```js
PuzzleHuntSandbox.onContext(function (data) {
  if (!data) return;
  var solved = data.hunt.progress.solvedPuzzles || [];
  if (solved.indexOf('stage-1') >= 0) showStage2();
});
```

## 作答历史 getAnswerHistory(puzzleId, cb)

读取**本队**在指定题目的全部提交记录，按标准化答案去重（同一答案多次提交时，后一次覆盖前一次）：

```js
PuzzleHuntSandbox.getAnswerHistory('stage-1', function (answers, error) {
  if (error) return;   // 不可用时返回错误
  // answers: Record<标准化答案, { resp, hit? }>
  //   resp: 判题回复消息（中间答案的消息 / 正确 / 错误提示）
  //   hit: 命中答案类型，'canonical' 主答案 / 'intermediate' 中间答案（里程碑）；
  //        省略 = 未命中任何答案
  if (answers['warm-up'] && answers['warm-up'].hit === 'intermediate') {
    showMilestone();
  }
});
```

- 仅含本队自身的答案提交，不含提示购买与反馈消息；其他队伍不可见。
- 键为平台标准化后的答案（默认小写、去空格等，按题目归一化设置），`resp` 为判题回复消息（作者自定义消息优先，否则为平台默认文案）。
- 测试者草稿试玩（testsolver）同样可用，读取的是试玩进度里的记录。

## 跨题状态通道

同一 Hunt 内的交互题可以互相传递 JSON 数据（例：题 A 的涂鸦笔记给题 B 读）。

**两种作用域**：

- `useTeamState(key)` — 全队共享：队友 A 写入，队友 B 打开同一 Hunt 任意题目即可读到。
- `usePlayerState(key)` — 队员个人：每个队员各存一份，互不可见。

两者都绑定队伍，队伍解散即随之清理。

**每个句柄的方法**：

| 方法 | 说明 |
|------|------|
| `get()` | 取本地缓存值。尚未拿到数据时为 `undefined`；`reload()` 后服务端无该 key 时为 `null` |
| `set(value)` | 写入并保存；保存失败仅写日志，不影响页面运行 |
| `reload()` | 从服务端重新拉取该 key |
| `subscribe(fn)` | 订阅变化：注册时立即用当前值回调一次，之后每次值更新（含队友刷新页面带来的快照）都会回调；返回退订函数 |

```js
var notes = PuzzleHuntSandbox.useTeamState('notes');
notes.subscribe(function (v) { render(v); });   // 渲染最新值
notes.set({ strokes: '...' });                  // 写回
```

### ⚠️ 先读后写：状态是整值覆盖

状态按 key **整值覆盖**存储，不是合并写入。如果队友或另一个页面刚写了新值，你未读取就直接 `set()`，会**用旧值覆盖掉对方的数据**。

推荐流程：**订阅/读最新值 → 本地合并 → 写回**：

```js
var canvas = PuzzleHuntSandbox.useTeamState('canvas');
canvas.reload();
canvas.subscribe(function (v) {
  v = v || {};
  v.lastEditor = 'playerB';
  canvas.set(v);   // 在最新值基础上合并后再写
});
```

同一 key 的写入频率也会影响冲突概率，高频场景建议：每 key 只存一类数据、减小单值体积、写前 `reload()`。

### 无实时推送

队友写入后，其它成员**不会立即收到推送**；他们刷新或进入题面时才能读到最新值。设计谜题时请以此为准（例如「下一题见」式传递，而非实时协作白板）。如有实时性需求，请参考hunt状态同步功能（预计2026.11上线）。

## 配额与限制

- **key 命名**：仅字母数字与 `_` `.` `:` `-`，且须以字母数字开头；长度有上限。
- **状态值**：必须可 JSON 序列化。
- **用量预算**：单条值、单次写入条数与每个存储空间的总量，都按 **128KB 量级**规划。这是保守取值——留足余量比贴着上限更稳妥，超了反而难排查。
- **超限行为**：保存**静默失败**——页面不报错、玩家无感知，只在题面页控制台留下日志。

### 每个存储空间总量

把服务端存储想象成一批**互相独立的网盘**：每场比赛给**每支队伍**开两类网盘——

- **队伍盘**（`useTeamState`）：每队一个，全队读写同一份；
- **个人盘**（`usePlayerState`）：每名队员各一个，只属于他自己。

队伍盘之间、个人盘之间、队伍盘与个人盘之间**互不挤占**，每个盘独立享有对应大小的空间。

**举例**：一场 Hunt 共 10 支队、50 名玩家（每队 5 人）——

- 队伍盘：10 个（A 队、B 队……各一个）；
- 个人盘：50 个（每名玩家一个）；
- 合计 **60 个独立的空间**，但对你设计谜题有意义的是**单个盘的配额**。

所以：A 队把队伍盘写满了，不影响 B 队；某玩家的个人盘满了，也不影响他队友。你不用担心 50 人加起来爆容量——只需保证**任意一个盘内**你规划的所有 key 加起来不超配额即可。

`set()` 保存失败时**页面不会报错、玩家也看不到提示**，只在题面页浏览器控制台留下日志。因此请在设计阶段就按 128KB 量级预算好用量，别指望玩家反馈。

## 6. 注意事项与常见问题

- **SDK 不能代替答案提交**。玩家最终仍须在题页下方的答案框输入并提交；交互内容只是过程体验。
- **勿在交互包内写入正确答案或未购买提示的正文**。包文件对已解锁题目的玩家完全可见，机密内容一律放服务端（答案、提示由平台下发，交互题后端脚本预计2026.11上线）。**我们强烈推荐使用哈希存储答案，避免明文存储。验证时对用户输入的哈希值与存储的哈希值进行对比。**
- **高度自适应但请留好流式布局**：玩家可在沉浸阅读、并排阅读与手机窄屏间切换，iframe 可用宽度差异很大。请用流体布局并自测三种宽度，不要写死固定像素画布。
- **调试**：用 `PuzzleHuntSandbox.log('...')` 输出日志，在父页面（题面页）浏览器控制台查看；保存失败的错误也会打在这里。
- **zip 打包**：确保 `index.html` 位于包根目录（或仅有一层文件夹包裹）；macOS 打包注意排除 `.DS_Store`、`__MACOSX` 等杂物。

## 7. 上线前自测清单

- [ ] 包内已引入 `/hunt-sandbox-sdk.js`
- [ ] zip 根目录含 `index.html`，双击本地打开无白屏、控制台无报错
- [ ] 沉浸 / 并排 / 手机三种宽度下布局正常、无内部滚动条
- [ ] 用 testsolver 试玩：跨题状态写入后，刷新或换题能读到；`getContext()` 数据符合预期
- [ ] 如使用 `getAnswerHistory`，确认 testsolver 试玩下返回符合预期
- [ ] 同 key 双端写场景下，确认使用了「先读后写」合并流程
- [ ] 包内不含答案、未购提示正文等机密内容
