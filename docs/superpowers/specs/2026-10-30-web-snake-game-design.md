# Web 端贪吃蛇小游戏 — 设计文档（Spec）

- 日期：2026-10-30
- 阶段：需求澄清（brainstorming）
- 状态：设计已定稿，待实现计划（writing-plans）
- 产出范围：本文档仅为设计说明，不包含源码实现

---

## 1. 背景与目标

### 1.1 需求原文
> 帮我开发一个 web 端的贪吃蛇小游戏

### 1.2 仓库现状（已验证事实）
| 项 | 结论 |
| --- | --- |
| 仓库内容 | 仅 `README.md`（内容 `# snake2`）与 1 个初始提交 `c0ef60c` |
| 既有代码/规范 | 无源码、无构建配置、无测试、无代码风格约定 |
| Node/npm/npx | **不可用**（`command -v` 仅命中 `/usr/bin/python3`） |
| Python3 | 可用，可用于静态文件服务与冒烟验证 |

### 1.3 目标（Goals）
1. 打开浏览器即可玩：**零构建、零依赖、零网络请求**（除静态资源外）。
2. 核心手感正确：方向输入不丢帧、不能 180° 反向自杀、撞墙/撞自己即结束。
3. 完整可玩闭环：开始 → 暂停 → 结束 → 重开；计分与最高分持久化。
4. 桌面键盘与移动端触控均可操作，窄屏可用。
5. 游戏规则逻辑与渲染/输入解耦，可在浏览器中独立测试。

### 1.4 非目标（Non-Goals，YAGNI）
- 不做后端、账号、在线排行榜、多人对战。
- 不做关卡编辑器、皮肤商城、音效音乐资源。
- 不做框架级工程化（TypeScript / Vite / 打包器）——受环境约束，且对单页小游戏无收益。
- 不做 IE 兼容与 SSR。

---

## 2. 需求澄清：默认决策（全自动模式静默接管）

流水线为全自动模式，不阻塞等待人工答复。以下为原本需确认的问题，按
「① 上下文已验证事实 > ② 风险最低/改动最小/符合架构惯例 > ③ 行业最佳实践」
逐条给出默认结论；实现阶段如需调整，仅需改本文档对应条目。

| # | 待澄清问题 | 默认决策 | 依据 |
| --- | --- | --- | --- |
| Q1 | 交付形态：单文件 HTML 还是多文件工程？ | **多文件静态站点**（`index.html` + `src/` 模块），可直接双击 `index.html` 打开，也可用 `python3 -m http.server` 托管 | 多文件便于逻辑与视图分离、便于测试；无构建依赖，仍保持“零安装” |
| Q2 | 技术栈：原生 JS / React / 游戏引擎？ | **原生 ES Module + Canvas 2D** | 无 Node 工具链，无法安装依赖或跑打包；贪吃蛇对渲染要求极低，引擎属过度设计 |
| Q3 | 视觉风格 | **深色霓虹极简**（深底 + 单色高亮蛇身 + 高对比食物），栅格化像素感，无外部字体/图片 | 无外链资源，避免离线失效；对比度满足可读性 |
| Q4 | 网格与画布尺寸 | 逻辑网格 **20 × 20**，单元格 24px 逻辑像素，画布 480 × 480，CSS 自适应缩放（`max-width: 100%`） | 20×20 是经典难度与可玩性平衡点；固定逻辑坐标 + 缩放避免像素偏移 |
| Q5 | 初始速度与提速 | 初始 **每秒 6 格**；每吃 5 个食物提速 1 格/秒；上限 **每秒 18 格** | 上限防止高分段不可操作 |
| Q6 | 撞墙行为 | **撞墙即死亡**（不做穿墙） | 规则直观、判定简单、无争议；穿墙作为可选扩展在 §9 记录 |
| Q7 | 允许 180° 反向吗 | **不允许**：反向输入被忽略（不改变当前方向，也不致死） | 避免“手滑即死”的挫败感；输入在每步 tick 结算，队列只保留最后一个有效方向 |
| Q8 | 食物生成规则 | 在**空格子集合**中随机取一格；胜利条件为蛇占满全部 400 格 | 避免出生在蛇身上导致的死锁 |
| Q9 | 暂停/继续 | `Space` 或 `P` 切换暂停；窗口失焦（`blur`）与页面隐藏（`visibilitychange`）**自动暂停** | 防止切走页面时“盲死” |
| Q10 | 移动端操作 | Canvas 上**滑动手势**（阈值 24px，取主轴方向）+ 屏幕方向键（D-pad） | 触控无键盘，双通道覆盖单手/双手习惯 |
| Q11 | 最高分存储 | `localStorage` 键 `snake2.bestScore`；读写异常时回退为内存值，不报错 | 隐私模式 / 禁用存储场景不应破坏游戏 |
| Q12 | 语言与文案 | 界面中文，代码标识符英文 | 与需求语言一致 |
| Q13 | 关卡/障碍 | MVP 不做障碍物与关卡 | YAGNI，保证单计划可完成 |
| Q14 | 声音 | MVP 不做 | 无音频资源授权与体积负担；预留钩子见 §9 |
| Q15 | 测试方式 | 纯逻辑引擎用**浏览器内断言页** `tests/engine-test.html` 验证；站点可达性用 `python3 -m http.server` + HTTP 200 冒烟 | 环境无 Node，无法跑 Jest/Vitest；断言页对逻辑覆盖率等价且零依赖 |

---

## 3. 方案对比与选型

### 方案 A：原生 ES Module + Canvas 2D（推荐 ✅）
- 结构：`index.html` 承载 DOM 外壳（计分板、遮罩层），`src/` 下按职责拆分模块，`src/engine.js` 为纯逻辑。
- 优点：零安装、双击可跑；逻辑可单测；渲染性能充裕；代码量可控（约 600–800 行）。
- 缺点：需自行实现循环、输入与状态机（本需求规模下成本很低）。

### 方案 B：DOM 网格（400 个 `div` + CSS 过渡）
- 优点：无 Canvas，易调试样式。
- 缺点：每 tick 增删节点/改 class 引起布局抖动；60fps 下性能与体验劣于 Canvas；模块边界更浑浊。
- 结论：不采用。

### 方案 C：引入游戏引擎/框架（Phaser、PixiJS、React）
- 优点：生态成熟，扩展性强。
- 缺点：需要 npm 安装与打包（**环境无 Node，直接不可行**）；对 20×20 网格属过度设计。
- 结论：不采用。

**选型：方案 A。** 与“无 Node 工具链”的已验证事实一致，改动面最小，符合静态站点惯例。

---

## 4. 架构设计

### 4.1 分层
```
┌─────────────────────────────────────────────┐
│ main.js          组合根：装配、状态机、循环  │
├───────────────┬───────────────┬─────────────┤
│ input.js      │ renderer.js   │ storage.js  │
│ 键盘/触控 →    │ 状态 → 像素    │ 最高分读写  │
│ 方向意图       │ (Canvas/DOM)  │ (容错)      │
├───────────────┴───────────────┴─────────────┤
│ engine.js     纯逻辑：tick / 碰撞 / 计分     │
│               （无 DOM、无计时器、无全局）    │
├─────────────────────────────────────────────┤
│ config.js     常量：网格、速度、颜色、键位   │
└─────────────────────────────────────────────┘
```
依赖方向单向向下：`main` → (`renderer` / `input` / `storage` / `engine`) → `config`。
`engine` 与 `config` **不得**引用 DOM 或 `window`（`storage` 除外，它只碰 `localStorage`）。

### 4.2 模块职责与接口
| 模块 | 职责 | 关键接口（示意） |
| --- | --- | --- |
| `src/config.js` | 全部可调常量 | `GRID_COLS=20`, `GRID_ROWS=20`, `CELL=24`, `BASE_STEPS_PER_SEC=6`, `MAX_STEPS_PER_SEC=18`, `SPEED_UP_EVERY=5`, `STORAGE_KEY` |
| `src/engine.js` | 纯状态推进与判定 | `createGame(opts) → state`；`queueDirection(state, dir)`；`step(state) → {state, event}`；`isWin(state)`；`isGameOver(state)` |
| `src/renderer.js` | 绘制盘面/HUD/遮罩 | `createRenderer(canvas) → {draw(state, ui), resize()}` |
| `src/input.js` | 采集意图并归一化 | `bindInput({onDirection, onTogglePause, onRestart})` |
| `src/storage.js` | 最高分持久化 | `loadBestScore() → number`；`saveBestScore(n)` |
| `src/main.js` | 状态机 + 固定步长循环 + 事件绑定 | `start()`, `pause()`, `resume()`, `restart()` |
| `index.html` | DOM 外壳 + 样式引用 | HUD、开始/暂停/结束遮罩、D-pad |
| `styles.css` | 布局与主题 | 响应式、深色霓虹、`prefers-reduced-motion` |

### 4.3 状态模型（纯数据）
```js
state = {
  cols: 20, rows: 20,
  snake: [{x, y}, ...],      // 索引 0 为蛇头
  direction: 'right',         // 'up'|'down'|'left'|'right'
  pendingDirection: 'right',  // 下一步生效方向，最多 1 个
  food: {x, y},
  score: 0,
  eaten: 0,
  status: 'ready',            // 'ready'|'running'|'paused'|'over'|'won'
  stepsPerSec: 6,
  rng: fn                     // 可注入，测试用确定性随机
}
```
不可变性策略：`step()` 返回**新对象**（浅拷贝 + 新数组），便于测试断言与渲染对比；单帧开销可忽略。

### 4.4 数据流（一次 tick）
```
keydown/swipe → input.js 归一化方向 → main 调用 engine.queueDirection(state, dir)
                                                 │（忽略 180° 反向、忽略非 running 状态）
rAF(now) → accumulator += dt → 每满 1/stepsPerSec 执行一次 engine.step(state)
                                                 │
        step 内部：1) direction = pendingDirection
                  2) 计算新头 = 头 + delta
                  3) 越界 → status='over'，event='hit-wall'
                  4) 撞自身（排除将被移走的尾格，未吃食物时） → status='over'，event='hit-self'
                  5) 命中食物 → 不弹尾，score+10，eaten+1，按需提速，重生成食物
                     若蛇长 == 400 → status='won'
                  6) 否则弹尾
                  返回 {state, event}
                                                 │
main 依 event 更新最高分、HUD、遮罩 → renderer.draw(state, ui)
```

### 4.5 游戏循环
- 使用 `requestAnimationFrame` + **固定步长累加器**（`dt` 上限钳制 250ms，防止后台标签页返回时爆步）。
- 渲染每帧执行（用于插值/高亮动画），逻辑推进按 `stepsPerSec` 节流；`paused`/`over`/`won` 时不累加。
- 循环只由 `main.js` 持有，`engine` 内不存在计时器（可测性关键约束）。

### 4.6 输入映射
| 输入 | 动作 |
| --- | --- |
| `ArrowUp/W` | 上 |
| `ArrowDown/S` | 下 |
| `ArrowLeft/A` | 左 |
| `ArrowRight/D` | 右 |
| `Space` / `P` | 暂停 ⇄ 继续（`ready` 时等同开始） |
| `Enter` / `R` | 结束态重开 |
| 滑动（阈值 24px，主轴判定） | 对应方向 |
| D-pad 按钮（`pointerdown`） | 对应方向 |
| `blur` / `visibilitychange(hidden)` | 自动暂停 |

方向键需 `preventDefault()` 阻止页面滚动；D-pad 与 Canvas 容器设 `touch-action: none`。

---

## 5. 视觉与布局

- 画布：480×480 逻辑像素；容器 `width: min(92vw, 480px)`，画布 `width/height` 属性固定，CSS 缩放，`devicePixelRatio` 适配（按 DPR 设置 backing store 并 `ctx.scale`）。
- 主题（深色霓虹）：
  - 背景 `#0b0f14`，棋盘 `#0f1620`，网格线 `rgba(255,255,255,0.04)`
  - 蛇身 `#2ee6a8`（头更亮 `#7dffcd`），食物 `#ff5f7a`
  - 文本 `#e6edf3`，次要 `#8b98a5`
- HUD：`得分 / 最高分 / 速度` 三栏，等宽数字（`font-variant-numeric: tabular-nums`）。
- 遮罩层（居中卡片）三态：`ready`（标题 + 「开始游戏」 + 操作说明）、`paused`（「已暂停」+ 继续）、`over`（「游戏结束」+ 得分 + 重开）、`won`（「通关！」）。
- 移动端：HUD 在画布上方，D-pad 在下方，仅触控设备显示（`@media (hover: none)`）。
- 无障碍：遮罩/按钮可键盘聚焦；分数变化写入 `aria-live="polite"` 区域；尊重 `prefers-reduced-motion`（关闭脉冲动画）。

---

## 6. 规则与数值（验收基线）
| 规则 | 取值 |
| --- | --- |
| 网格 | 20 × 20（共 400 格） |
| 初始蛇 | 长度 3，水平居中，头朝右 |
| 初始方向 | 右 |
| 食物 | 每步生成于空格集合；吃后立即重生 |
| 计分 | 每个食物 +10 分 |
| 提速 | 每累计 5 个食物 +1 格/秒，上限 18 格/秒 |
| 死亡 | 越界、撞自身（不含即将移走的尾格） |
| 胜利 | 蛇占满 400 格 |
| 重开 | 分数归零、速度复位、蛇复位；最高分保留 |

**尾格例外**：未吃食物时，尾格会在本步被移走，因此新头进入“当前尾格”合法；吃掉食物时尾格不移走，此时该格视为蛇身。

---

## 7. 错误处理与边界
| 场景 | 处理 |
| --- | --- |
| `localStorage` 抛错/不可用 | `storage` 内部 `try/catch`，降级为内存变量；同一会话内仍生效 |
| 损坏的最高分（非数字/负数） | `Number.isFinite` + `>= 0` 校验，否则按 0 处理 |
| 后台标签页返回 | 累加器 `dt` 钳制 250ms；`visibilitychange` 已自动暂停 |
| Canvas 上下文获取失败 | 顶部显示可读错误提示，不白屏 |
| 极窄屏（< 320px） | 画布按比例缩小，D-pad 换行不溢出 |
| 快速连续同帧多键 | `pendingDirection` 只保留最后一次 **有效** 方向；单 tick 内不做 180° 反转（含“上→左→下”同帧合谋反向的防护：校验对象是 `direction` 而非 `pendingDirection`） |
| 蛇占满后仍收到食物请求 | 先判胜利再生成食物，避免死循环 |

---

## 8. 测试策略（与 §2 Q15 一致，受“无 Node”约束）
1. **纯逻辑断言页** `tests/engine-test.html`：直接 `import` `src/engine.js`，用注入的 `rng` 构造确定性场景，覆盖：
   - 初始状态（长度 3、方向右、status=ready）
   - 正常前进、吃到食物后长度 +1 且分数 +10
   - 反向输入被忽略（方向仍为原方向）
   - 撞墙 → `status='over'`，`event='hit-wall'`
   - 撞自身 → `status='over'`，`event='hit-self'`
   - 尾格例外合法通行
   - 提速阈值与上限
   - 食物从不生成在蛇身
   - 占满 400 格 → `status='won'`
   - 页面顶部输出 `PASS n / FAIL n` 汇总。
2. **站点冒烟**：`python3 -m http.server` 起服务，`curl -s -o /dev/null -w '%{http_code}'` 校验 `index.html`、`styles.css`、各 `src/*.js`、测试页均 200。
3. **手工验收清单**：开始/暂停/重开、键盘与滑动、移动端窄屏、刷新后最高分保留。
4. 环境无 headless 浏览器，**不声明**已完成像素级视觉自动化校验；视觉检查在实现阶段以人工/截图方式记录。

---

## 9. 可选扩展（不在本次范围）
穿墙模式开关、障碍关卡、音效（预留 `engine` 返回 `event` 已为音效提供钩子）、速度曲线可配置、排行榜后端。

---

## 10. 验收标准（Definition of Done）
1. 双击 `index.html` 或用任意静态服务器打开即可游玩，控制台无报错。
2. 键盘（方向键/WASD）、触控滑动、D-pad 三种输入均生效；方向键不引起页面滚动。
3. 撞墙、撞自己立即结束并显示得分与「再来一局」。
4. 暂停/继续可用；切走标签页自动暂停。
5. 得分与最高分正确；刷新页面后最高分保留；存储不可用时不报错。
6. `tests/engine-test.html` 全部用例 PASS。
7. 375px 宽视口下无横向滚动、D-pad 可点。
8. 代码按 §4.2 模块边界组织，`engine.js` 不含 DOM 引用。

---

## 11. 自审记录（Spec Self-Review）
- 占位符扫描：无 `TBD`/`TODO`；所有规则均有确定数值（§6）。
- 内部一致性：`4.3` 状态字段与 `4.4` 判定顺序、§6 数值、§8 用例一一对应；模块表与 §4.1 分层图一致。
- 范围检查：单一实现计划可覆盖（静态站点 7 个文件 + 1 个测试页），无需拆分。
- 歧义检查：已明确“尾格例外”“反向输入忽略而非致死”“同帧多键以 `direction` 为校验基准”“撞墙即死不做穿墙”。
