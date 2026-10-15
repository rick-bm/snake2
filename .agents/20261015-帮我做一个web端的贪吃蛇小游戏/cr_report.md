# 代码评审报告 — Web 端贪吃蛇小游戏

- 日期：2026-10-15
- 阶段 / 节点：review / 代码评审
- 采用技能：code-review-skill
- 需求：帮我做一个 web 端的贪吃蛇小游戏
- 评审对象（本次编码实现变更，git `0e5ba7a → b0b637d`）：
  - `index.html`（+271 行，唯一交付物）
  - `README.md`（+19 行）
- 上游依据文档：
  - `docs/superpowers/specs/2026-10-15-snake-game-design.md`（设计）
  - `docs/superpowers/plans/2026-10-15-snake-game.md`（实施计划）

## 0. 知识库检索留痕

- 执行命令：`mind content-search -b V_Qg6pUe1CQLEgNGtbNV3 --query "web 贪吃蛇 小游戏" --json --limit 5`
- 结果：`matchedDocuments: 0`，`fallbackReason: "index-empty"`，`total: 0`。
- 结论：**已检索、未命中**（知识库当前索引为空，无可用参考内容）。

## 1. 结论（Decision）

**✅ Approve with comments（放行，附带建议）**

- blocker 数量：**0**（无 🔴 阻断合并问题）
- 🟡 important：1 条
- 🟢 nit / 💡 suggestion：5 条

整体质量较高：单文件零依赖、逻辑分区清晰、命名规范、与设计文档和实现计划**逐条一致**。核心游戏逻辑（移动、反向保护、方向队列、进食增分、撞墙/撞自身判负、棋盘满判胜、状态机、RAF 固定节奏循环）经逐行核对正确，无阻断性缺陷。

## 2. 高层评估（架构 / 设计）

- 技术方案与设计一致：原生 HTML5 `index.html`，内联 CSS/JS，`<canvas>` 2D 网格渲染 + `requestAnimationFrame` 时间累积固定节奏（150ms/格），无第三方依赖、无构建步骤。
- 逻辑分区（配置常量 / DOM / 状态 / 工具 / 初始化 / 食物生成 / 渲染 / 输入 / 游戏逻辑 / 主循环）边界清晰，符合 spec 第 4 节。
- 状态机 `READY → RUNNING ↔ PAUSED → OVER` 与设计文档第 7 节一致，胜利并入 `OVER`（与计划 Task 4 决策一致）。
- 反向保护采用「方向队列 + `isOpposite` 收尾方向判定」，正确处理快速连按导致的方向翻转抖动，与计划 Task 2 一致。
- 食物生成用「占用集合 + 全盘扫描空位」，`free.length === 0` 时返回 `null`，天然无死循环，与 spec 第 8 节「安全上限」一致（此处实现优于单纯的 while 重试）。
- 复杂度：`spawnFood` 最坏 O(400)（20×20），`step`/`render` 均 O(蛇长) 且蛇长 ≤ 400，无性能隐患；`render` 每帧 38 条网格线，代价可忽略。

## 3. 逐文件发现

### 3.1 `index.html`

#### 🟡 [important] 暂停状态下方向键会被写入方向队列
`keydown` 处理中，方向键分支对 `READY`/`OVER` 走 `reset()` 分支，其余情况统一进入入队逻辑（`index.html:180-186`）。`State.PAUSED` 未单独拦截，因此暂停时按方向键虽不移动蛇，但会 `directionQueue.push(dir)`，恢复后第 1 个 tick 会立即改向。若设计意图为「暂停 = 完全冻结」，应让 PAUSED 分支直接 `return`（仅保留 Space 恢复）。
- 影响：行为与「已暂停」提示的直觉略有出入（半冻结）。
- 修复建议：在方向键分支开头加上 `if (state === State.PAUSED) return;`（在 `preventDefault` 之后），或显式声明「暂停允许预置方向」为非目标。

#### 🟢 [nit] 初始加载（READY）画面为空网格，无蛇无食物
`index.html` 未在加载时调用 `reset()`，初始 `snake = []`、`food = null`，因此打开页面仅见 480×480 网格与「按方向键开始」提示，需按键后蛇才出现。与 spec 状态机的 READY 语义一致（非 bug），但对「打开即玩」的成功标准略有观感落差。
- 建议：可在 `loop` 首次渲染前对 READY 态静态绘制一条初始蛇（不进入 RUNNING），或保持现状并在文案中明确。

#### 🟢 [nit] 主循环状态变量声明位置分散
`acc` 在 `reset()`（`index.html:106`）中被读写，但声明在文件末尾主循环区（`index.html:249`），`lastTime` 亦在末尾。运行时无问题（`reset` 仅在事件回调中被调用，彼时 `let acc` 已初始化，无 TDZ 错误），但「声明在引用函数之后」降低可读性。
- 建议：将 `acc`、`lastTime` 上移至状态区或与 `reset()` 就近归组。

#### 🟢 [nit] 颜色/主题存在双处维护
`COLORS.bg/snake/food/grid`（`index.html:56-61`）与 CSS 变量 `--bg/--snake/--food/--grid`（`index.html:8-17`）各维护一份，且画布背景 `#ffffff` 同时硬编码于 CSS `canvas`（`--board`）与 `COLORS.bg`。改动主题需同步多处。
- 建议：可由 JS 通过 `getComputedStyle` 读取 CSS 变量，或明确只以 CSS 变量为唯一真源。

#### 🟢 [nit] 按键判断 API 混用
方向键/Enter 用 `e.key`，Space 用 `e.code`（`index.html:170-205`）。当前行为正确（避免输入法空格差异），但同一处理函数内混用两种标识略显不一致。
- 建议：统一约定（如键盘布局无关的 `code` 用于控制键）并加注释说明。

### 3.2 `README.md`

#### 🟢 [nit] 内容冗余
「操作」（第 11-13 行）与「说明」（第 17-18 行）两节内容基本重复。
- 建议：合并为一节，保留「运行 / 操作」即可。

#### 💡 [suggestion] 补充初始按键说明
README 未明示「首次需按方向键开始（初始画面无蛇）」，与 `index.html` 的 READY 行为配合时可补充一句，降低用户困惑。

## 4. 逐项核对的正确性结论（无问题项确认）

- ✅ 反向保护：`isOpposite(a,b)` 支持任意两方向；运行中判定取队列尾方向作为「当前有效方向」，避免「上→下」快速连按误判；READY/OVER 重启时判定相对初始 `direction`。逻辑正确。
- ✅ 撞自身判定：吃食物时 `bodyToCheck = snake`（尾节不移动），不吃食物时 `bodyToCheck = snake.slice(0,-1)`（尾节即将移除，可走到旧尾位置）；与计划 Task 3 说明完全一致。
- ✅ 进食/胜利：`step` 内 `newHead` 与 `food` 重合 → `score++` → `spawnFood()` → 无空位返回 `null` → `win()`，棋盘满判胜并入 `OVER`。
- ✅ 边界：撞墙先于撞自身判定，`newHead` 越界即 `gameOver()`。
- ✅ 主循环：`lastTime === 0` 首帧重置避免首帧超大 delta；`while (acc >= MOVE_INTERVAL_MS)` 处理掉帧期间的多步推进，节奏固定。
- ✅ 无占位符 / TODO / 缺失引用；`acc` 为唯一「后声明先引用」，已确认为运行时安全（见 3.1）。

## 5. 验证说明

- 测试框架：本项目 spec 明确「无测试框架，验收采用手动冒烟」，故不新增测试，按评审策略做静态审查 + 语法/可用性核对。
- 语法检查降级：环境无 `node`（`command -v node` 为空），仅存在 `lightpanda` 与 `python3`。
  - `lightpanda` 因沙箱网络隔离无法加载本地页面：`file://` 报 `UnsupportedProtocol`，`http://127.0.0.1` 报 `CouldntConnect`，浏览器冒烟**未能执行**。
  - 已通过脚本块语法人工核对 + `git show` 核对变更无语法/结构异常。
- 结论：本次以「静态代码审查」为主，逻辑分支、边界、类型一致性均已逐行核对；运行时浏览器冒烟受限于沙箱网络隔离，属环境限制，已如实标明，非代码缺陷。

## 6. 阻断小结

| 严重度 | 数量 | 说明 |
| --- | --- | --- |
| 🔴 blocking | 0 | 无阻断合并问题 |
| 🟡 important | 1 | 暂停时方向键入队（半冻结），建议拦截 |
| 🟢 nit | 5 | 初始无蛇渲染 / 状态变量声明位置 / 颜色双处维护 / 按键 API 混用 / README 冗余 |
| 💡 suggestion | 1 | README 补充首次按键说明 |

**blocker_count = 0**