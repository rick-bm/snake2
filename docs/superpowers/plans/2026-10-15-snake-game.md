# 贪吃蛇（Web 端）实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans or superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现一个单文件、零依赖、浏览器直接打开的经典贪吃蛇小游戏。

**Architecture:** 单个 `index.html`，内联 CSS 与 JavaScript，使用 `<canvas>` 2D 网格渲染 + `requestAnimationFrame` 时间累积驱动固定节奏游戏循环。逻辑分区：配置常量 / 状态管理 / 输入处理 / 游戏逻辑 / 渲染 / 主循环。

**Tech Stack:** 原生 HTML5 + CSS3 + ES6 JavaScript（无框架、无构建、无依赖）。

**执行方式（已决策）:** subagent-driven-development（子代理驱动执行）——每个任务派发独立实现子代理，任务间由评审子代理做 spec 合规 + 代码质量检查。实现由后续实现阶段节点执行；本澄清节点只产出设计 + 实现计划。

## Global Constraints

- 交付物为单个文件 `index.html`，所有 CSS 与 JS 内联，禁止引入任何第三方库或 CDN。
- 网格：20×20，单元格 24px，画布 480×480。
- 移动节奏：每格 150ms（`MOVE_INTERVAL_MS = 150`）。
- 仅键盘方向键（↑↓←→）移动；`Space` 暂停/继续；结束后按方向键或 `Enter` 重新开始。
- 禁止 180° 直接反向（用方向队列确保正确性）。
- 视觉风格：简洁极简风，浅色背景 + 单一深色蛇身 + 红色圆点食物 + 充足留白。
- 初始蛇长 3，位于棋盘中央，初始向右移动；吃食物分数 +1、蛇身 +1。
- 撞墙或撞自身结束；棋盘满格判胜利（并入「已结束」状态展示）。
- 本任务 Git 只读：所有步骤**禁止**执行 `git add/commit/push` 等写操作，用「验证检查点」代替提交。git 仅允许 `status/log/diff/show` 等只读命令。
- 本项目无测试框架，验证采用：浏览器手动冒烟 + `node` 对抽取出的脚本做语法检查。
- 验证环境降级：若执行环境无 `node` 二进制，则语法检查降级为——抽取 `<script>` 内容与 Task 2 代码块拼接为完整 HTML，用 `lightpanda fetch --dump html` 无头加载，确认无致命 JS 报错（无 node 环境已实测：拼接产物可正常加载、无脚本错误）。

---

## 文件结构

- 创建 `index.html`：全部 HTML/CSS/JS 逻辑（唯一交付物）。
- 修改（可选）`README.md`：加一行说明如何运行。若任务未要求则跳过。

---

### Task 1: 页面骨架 + 静态渲染（棋盘、蛇、食物）

**Files:**
- Create: `index.html`

**Interfaces:**
- Produces: 常量 `GRID_SIZE, CELL_SIZE, CANVAS_SIZE, MOVE_INTERVAL_MS, INITIAL_LENGTH, COLORS`、`State`；全局状态 `snake, direction, directionQueue, food, score, state`；函数 `reset(), spawnFood(), render(), roundRect()`

- [ ] **Step 1: 写入完整 `index.html`（骨架 + 静态渲染）**

创建 `index.html`，内容如下：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>贪吃蛇</title>
<style>
  :root {
    --bg: #f7f7f5;
    --text: #333333;
    --board: #ffffff;
    --snake: #111111;
    --food: #ff5a5a;
    --grid: #f0f0ee;
    --border: #e5e5e2;
    --muted: #888888;
  }
  * { box-sizing: border-box; margin: 0; padding: 0; }
  body {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 16px;
    background: var(--bg);
    color: var(--text);
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", "PingFang SC", "Microsoft YaHei", sans-serif;
  }
  .score { font-size: 20px; letter-spacing: 0.5px; }
  .score span { font-weight: 700; }
  canvas {
    background: var(--board);
    border: 1px solid var(--border);
    border-radius: 6px;
    box-shadow: 0 2px 12px rgba(0,0,0,0.06);
    display: block;
  }
  .hint { font-size: 13px; color: var(--muted); }
</style>
</head>
<body>
<div class="score">分数 <span id="score">0</span></div>
<canvas id="game" width="480" height="480"></canvas>
<div class="hint" id="hint">按方向键开始 · Space 暂停</div>
<script>
"use strict";

/* ===== 配置常量 ===== */
const GRID_SIZE = 20;
const CELL_SIZE = 24;
const CANVAS_SIZE = GRID_SIZE * CELL_SIZE; // 480
const MOVE_INTERVAL_MS = 150;
const INITIAL_LENGTH = 3;

const COLORS = {
  bg: "#ffffff",
  snake: "#111111",
  food: "#ff5a5a",
  grid: "#f0f0ee"
};

/* ===== DOM ===== */
const canvas = document.getElementById("game");
const scoreEl = document.getElementById("score");
const hintEl = document.getElementById("hint");
const ctx = canvas.getContext("2d");
if (!ctx) {
  document.body.innerHTML = "<p>当前浏览器不支持 canvas，无法运行。</p>";
  throw new Error("canvas 2d context unavailable");
}

/* ===== 状态 ===== */
const State = { READY: "ready", RUNNING: "running", PAUSED: "paused", OVER: "over" };

let state = State.READY;
let snake = [];
let direction = { x: 1, y: 0 };
let directionQueue = [];
let food = null;
let score = 0;

/* ===== 工具 ===== */
function roundRect(ctx, x, y, w, h, r) {
  ctx.beginPath();
  ctx.moveTo(x + r, y);
  ctx.arcTo(x + w, y, x + w, y + h, r);
  ctx.arcTo(x + w, y + h, x, y + h, r);
  ctx.arcTo(x, y + h, x, y, r);
  ctx.arcTo(x, y, x + w, y, r);
  ctx.closePath();
  ctx.fill();
}

/* ===== 初始化 / 复位 ===== */
function reset() {
  const mid = Math.floor(GRID_SIZE / 2);
  snake = [];
  for (let i = 0; i < INITIAL_LENGTH; i++) {
    snake.push({ x: mid - i, y: mid });
  }
  direction = { x: 1, y: 0 };
  directionQueue = [];
  score = 0;
  scoreEl.textContent = "0";
  state = State.RUNNING;
  spawnFood();
}

/* ===== 食物生成 ===== */
function spawnFood() {
  const occupied = new Set(snake.map(s => s.x + "," + s.y));
  const free = [];
  for (let y = 0; y < GRID_SIZE; y++) {
    for (let x = 0; x < GRID_SIZE; x++) {
      if (!occupied.has(x + "," + y)) free.push({ x, y });
    }
  }
  food = free.length === 0 ? null : free[Math.floor(Math.random() * free.length)];
}

/* ===== 渲染 ===== */
function render() {
  ctx.fillStyle = COLORS.bg;
  ctx.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);

  ctx.strokeStyle = COLORS.grid;
  ctx.lineWidth = 1;
  for (let i = 1; i < GRID_SIZE; i++) {
    ctx.beginPath();
    ctx.moveTo(i * CELL_SIZE + 0.5, 0);
    ctx.lineTo(i * CELL_SIZE + 0.5, CANVAS_SIZE);
    ctx.stroke();
    ctx.beginPath();
    ctx.moveTo(0, i * CELL_SIZE + 0.5);
    ctx.lineTo(CANVAS_SIZE, i * CELL_SIZE + 0.5);
    ctx.stroke();
  }

  if (food) {
    ctx.fillStyle = COLORS.food;
    ctx.beginPath();
    ctx.arc(food.x * CELL_SIZE + CELL_SIZE / 2, food.y * CELL_SIZE + CELL_SIZE / 2, CELL_SIZE / 2 - 2, 0, Math.PI * 2);
    ctx.fill();
  }

  snake.forEach(seg => {
    ctx.fillStyle = COLORS.snake;
    roundRect(ctx, seg.x * CELL_SIZE + 1, seg.y * CELL_SIZE + 1, CELL_SIZE - 2, CELL_SIZE - 2, 3);
  });
}
</script>
</body>
</html>
```

> 说明：本步先交付骨架与静态渲染；`step`/`gameOver`/`win`/输入处理/主循环在后续步骤补齐。为避免语法错误，本步暂不引用未定义函数。

- [ ] **Step 2: 语法检查（抽取脚本）**

```bash
awk '/<script>/{flag=1;next}/<\/script>/{flag=0}flag' index.html > /tmp/_snake_check.js && node --check /tmp/_snake_check.js
```

Expected: 无输出，退出码 0。

- [ ] **Step 3: 浏览器冒烟**

用浏览器打开 `index.html`，Expected: 显示 480×480 棋盘（淡色网格线），无蛇无食物（因为本步尚未调用 `reset()`）。

> Git 只读：不执行 commit。

---

### Task 2: 键盘输入 + 方向队列 + 游戏循环（移动）

**Files:**
- Modify: `index.html`（在 `render` 函数之后、`</script>` 之前新增代码）

**Interfaces:**
- Consumes: `State, state, direction, directionQueue, snake, reset`（Task 1）
- Produces: `isOpposite(a,b)`, `KEY_DIRS`, 键盘事件监听, `step()`, `gameOver()`, `win()`, `loop(timestamp)`

- [ ] **Step 1: 新增输入处理 + 游戏逻辑 + 主循环**

在 `render` 函数定义之后、`</script>` 之前，插入以下代码（`render()` 结尾 `}` 后的位置）：

```javascript

/* ===== 输入处理 ===== */
const KEY_DIRS = {
  ArrowUp:    { x: 0, y: -1 },
  ArrowDown:  { x: 0, y: 1 },
  ArrowLeft:  { x: -1, y: 0 },
  ArrowRight: { x: 1, y: 0 }
};

function isOpposite(a, b) {
  return a.x + b.x === 0 && a.y + b.y === 0;
}

function updateHint(text) {
  hintEl.textContent = text;
}

document.addEventListener("keydown", (e) => {
  const dir = KEY_DIRS[e.key];
  if (dir) {
    e.preventDefault();
    if (state === State.READY || state === State.OVER) {
      reset();
      updateHint("Space 暂停");
      if (!isOpposite(direction, dir)) directionQueue.push(dir);
      return;
    }
    const last = directionQueue.length
      ? directionQueue[directionQueue.length - 1]
      : direction;
    if (!isOpposite(last, dir) && directionQueue.length < 3) {
      directionQueue.push(dir);
    }
    return;
  }

  if (e.code === "Space") {
    e.preventDefault();
    if (state === State.RUNNING) {
      state = State.PAUSED;
      updateHint("已暂停 · Space 继续");
    } else if (state === State.PAUSED) {
      state = State.RUNNING;
      updateHint("Space 暂停");
    }
    return;
  }

  if (e.key === "Enter" && state === State.OVER) {
    e.preventDefault();
    reset();
    updateHint("Space 暂停");
  }
});

/* ===== 游戏逻辑 ===== */
function step() {
  if (state !== State.RUNNING) return;
  if (directionQueue.length) direction = directionQueue.shift();

  const head = snake[0];
  const newHead = { x: head.x + direction.x, y: head.y + direction.y };

  if (newHead.x < 0 || newHead.x >= GRID_SIZE || newHead.y < 0 || newHead.y >= GRID_SIZE) {
    return gameOver();
  }

  const willEat = food && newHead.x === food.x && newHead.y === food.y;
  const bodyToCheck = willEat ? snake : snake.slice(0, -1);
  if (bodyToCheck.some(s => s.x === newHead.x && s.y === newHead.y)) {
    return gameOver();
  }

  snake.unshift(newHead);
  if (willEat) {
    score++;
    scoreEl.textContent = String(score);
    spawnFood();
    if (!food) return win();
  } else {
    snake.pop();
  }
}

function gameOver() {
  state = State.OVER;
  updateHint(`游戏结束 · 分数 ${score} · 按方向键或 Enter 重新开始`);
}

function win() {
  state = State.OVER;
  updateHint(`你赢了！分数 ${score} · 按方向键或 Enter 重新开始`);
}

/* ===== 主循环 ===== */
let lastTime = 0;
let acc = 0;

function loop(timestamp) {
  requestAnimationFrame(loop);
  if (lastTime === 0) lastTime = timestamp;
  const delta = timestamp - lastTime;
  lastTime = timestamp;

  if (state === State.RUNNING) {
    acc += delta;
    while (acc >= MOVE_INTERVAL_MS) {
      acc -= MOVE_INTERVAL_MS;
      step();
      if (state !== State.RUNNING) break;
    }
  }
  render();
}

requestAnimationFrame(loop);
```

- [ ] **Step 2: 语法检查**

```bash
awk '/<script>/{flag=1;next}/<\/script>/{flag=0}flag' index.html > /tmp/_snake_check.js && node --check /tmp/_snake_check.js
```

Expected: 无输出，退出码 0。

- [ ] **Step 3: 浏览器冒烟**

打开 `index.html`，按方向键。Expected: 蛇开始向右移动；方向键可转向；无法 180° 反向（初始向右时按「左」应被忽略、继续向右；运行中快按「上→下」时「下」应被忽略、保持向上）；Space 可暂停/恢复；`hint` 文案随状态切换。

> Git 只读：不执行 commit。

---

### Task 3: 吃食物、计分、碰撞、结束/胜利

**Files:**
- Modify: `index.html`（逻辑已在 Task 2 写入，本任务为验证 + 边缘修正）

**Interfaces:**
- Consumes: `step(), spawnFood(), gameOver(), win()`（Task 2）

- [ ] **Step 1: 核对 `step()` 中计分与碰撞逻辑**（无需改代码，确认与设计一致）

确认点：
- `willEat` 判定：新蛇头与 `food` 坐标重合。
- 撞自身检测时吃食物场景用完整 `snake` 判定（尾节会被移除，等效空出一格），不吃食物场景用 `snake.slice(0,-1)`（尾节即将移除，不算碰撞）。
- 吃食物后：`score++` → `spawnFood()` → 若 `food === null` 判 `win()`。
- `spawnFood()` 在无空位时返回 `null`，有 `free.length` 兜底，不存在死循环。

- [ ] **Step 2: 语法检查**

```bash
awk '/<script>/{flag=1;next}/<\/script>/{flag=0}flag' index.html > /tmp/_snake_check.js && node --check /tmp/_snake_check.js
```

Expected: 无输出，退出码 0。

- [ ] **Step 3: 浏览器冒烟**

打开 `index.html`。Expected:
- 控制蛇头撞到食物：蛇变长、分数 +1、食物随机重生（不与蛇身重合）。
- 控制蛇头撞墙（边界）或撞自身：游戏结束，`hint` 显示「游戏结束 · 分数 N · 按方向键或 Enter 重新开始」。
- 结束后按方向键或 `Enter`：重新开始，分数归零。
- 控制台无报错。

> Git 只读：不执行 commit。

---

### Task 4: 暂停/继续与重开打磨 + 可选 README

**Files:**
- Modify: `index.html`
- Modify（可选）: `README.md`

**Interfaces:**
- Consumes: `State, updateHint, reset(), step(), render()`（Task 1/2）

- [ ] **Step 1: 核对状态机与提示文案一致性**

确认点：
- `READY`：初始 `hint` 为「按方向键开始 · Space 暂停」；首次按方向键进入 `RUNNING`。
- `RUNNING ↔ PAUSED`：`Space` 切换，`hint` 分别在「Space 暂停」/「已暂停 · Space 继续」。
- `OVER`：`gameOver()`/`win()` 写死结束文案；方向键与 `Enter` 均触发 `reset()` 并回到 `RUNNING`。
- 无「胜利」独立状态，「胜利」并入 `OVER`（与审阅建议一致）。

- [ ] **Step 2: 语法检查 + 全量冒烟回归**

```bash
awk '/<script>/{flag=1;next}/<\/script>/{flag=0}flag' index.html > /tmp/_snake_check.js && node --check /tmp/_snake_check.js
```

Expected: 无输出，退出码 0。

浏览器全量回归 Expected：
- [ ] 打开正常显示棋盘，按方向键开始移动。
- [ ] 方向键转向正常，无法 180° 反向。
- [ ] 吃食物蛇变长、分数 +1、食物重生。
- [ ] 撞墙/撞自身后结束并显示分数。
- [ ] `Space` 暂停/恢复正常。
- [ ] 结束后方向键/`Enter` 可重开。
- [ ] 长时间运行无卡顿，控制台无报错。

- [ ] **Step 3: 写 README**

将 `README.md` 替换为：

```markdown
# 贪吃蛇（Web 端）

单文件、零依赖的经典贪吃蛇小游戏。

## 运行

直接用浏览器打开 `index.html`。

## 操作

- 方向键（↑↓←→）：移动
- Space：暂停 / 继续
- 撞墙或撞自身即结束；结束后按方向键或 `Enter` 重新开始

## 说明

- 单文件零依赖，直接用浏览器打开 `index.html` 即玩。
- 方向键（↑↓←→）移动；`Space` 暂停 / 继续。
```

> Git 只读：不执行 commit。

---

## Self-Review 记录

- **Spec 覆盖**：技术方案（Task 1）、移动/输入/循环（Task 2）、进食计分碰撞结束胜利（Task 3）、暂停/重开/验收（Task 4）——全部要求均有对应任务。
- **占位符扫描**：无 TBD/TODO/「适当处理」类占位。
- **类型一致性**：`GRID_SIZE/CELL_SIZE/CANVAS_SIZE/MOVE_INTERVAL_MS/INITIAL_LENGTH/COLORS`、`snake/direction/directionQueue/food/score/state/State`、`reset/spawnFood/step/gameOver/win/render/loop/roundRect/isOpposite/updateHint/KEY_DIRS` 命名在各任务间一致。
- **Git 约束**：全部提交步骤以「验证检查点」替代，符合本任务 Git 只读要求。