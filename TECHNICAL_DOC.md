# 移动端拼图游戏 — 技术实现文档

> **版本**: v1.0  
> **最后更新**: 2026-05-23  
> **适用文件**: `index.html`（单文件架构，含 HTML + CSS + JS）  
> **目标读者**: 未来进行页面改造或优化的 AI 智能体 / 开发者

---

## 目录

1. [项目概述与架构设计](#1-项目概述与架构设计)
2. [DOM 结构规范](#2-dom-结构规范)
3. [CSS 设计系统](#3-css-设计系统)
4. [CONFIG 配置项详解](#4-config-配置项详解)
5. [State 状态对象详解](#5-state-状态对象详解)
6. [功能模块实现细节](#6-功能模块实现细节)
   - 6.1 [图片加载与默认图生成](#61-图片加载与默认图生成)
   - 6.2 [布局计算引擎](#62-布局计算引擎)
   - 6.3 [网格创建与图片提示](#63-网格创建与图片提示)
   - 6.4 [拼图块创建与图片切割](#64-拼图块创建与图片切割)
   - 6.5 [散落位置生成算法](#65-散落位置生成算法)
   - 6.6 [触摸拖动与边界限制](#66-触摸拖动与边界限制)
   - 6.7 [精准吸附功能](#67-精准吸附功能)
   - 6.8 [可取下重拼机制](#68-可取下重拼机制)
   - 6.9 [接近提示系统](#69-接近提示系统)
   - 6.10 [完成检测与庆祝动画](#610-完成检测与庆祝动画)
   - 6.11 [参考图预览](#611-参考图预览)
   - 6.12 [游戏重置](#612-游戏重置)
7. [数据流与生命周期](#7-数据流与生命周期)
8. [事件绑定架构](#8-事件绑定架构)
9. [关键业务规则](#9-关键业务规则)
10. [已知风险点与改造约束](#10-已知风险点与改造约束)
11. [性能特征与优化记录](#11-性能特征与优化记录)
12. [历史 Bug 修复记录](#12-历史-bug-修复记录)

---

## 1. 项目概述与架构设计

### 1.1 核心定位

移动端 3×3 拼图游戏，适配微信内置浏览器，单文件 HTML 架构，零外部依赖。

### 1.2 核心功能规则（不可违反）

| 规则编号 | 规则描述 | 违反后果 |
|:---|:---|:---|
| R1 | 初始拼图散落在九宫格周围，不遮挡目标区域 | 用户看不到空位，无法开始游戏 |
| R2 | 全程限制碎片在屏幕内 | 碎片拖出屏幕后无法找回 |
| R3 | 仅拖到碎片自己的正确目标位置才吸附 | 放错位置不锁定，保证游戏正确性 |
| R4 | 吸附后可随时取下重拼 | 锁定后用户无法纠正错误 |
| R5 | 所有碎片吸附后触发完成 | 完成检测失效 |

### 1.3 技术架构

```
单文件架构 (index.html)
├── <style> — 全部 CSS（CSS 变量 + 响应式 + 动画）
├── <body> — DOM 结构（声明式，JS 动态填充）
└── <script> — IIFE 封装的全部 JS 逻辑
    ├── CONFIG   — 静态配置常量（只读）
    ├── state    — 运行时可变状态
    ├── DOM 引用  — getElementById 缓存
    ├── 工具函数  — clamp / dist / shuffle
    ├── 功能模块  — 各子系统函数
    └── init()   — 入口，绑定事件 + 启动加载
```

### 1.4 设计决策记录

| 决策 | 选择 | 原因 | 替代方案 |
|:---|:---|:---|:---|
| 图片切割方式 | `background-position` 偏移 | 无需 Canvas 切割，性能最优，无跨域问题 | Canvas drawImage 切割（需 CORS，生成 blob） |
| 网格边框 | `box-shadow: 0 0 0 1px` | 不影响 CSS Grid 布局计算 | `border: 1px`（会改变单元格尺寸，导致 1px 偏移） |
| 图片提示 | `::before` 伪元素 + 低透明度 | 提示与边框独立控制透明度 | 直接在 `.grid-cell` 上设背景（边框也变透明） |
| 坐标系统 | 基于 `#puzzle-area` 的绝对像素坐标 | 统一参考系，避免嵌套偏移 | 百分比定位（精度不足） |
| 事件模型 | document 级 move/end + piece 级 start | 防止手指滑出碎片后丢失事件 | 仅 piece 级事件（滑出后失效） |

---

## 2. DOM 结构规范

### 2.1 完整 DOM 树

```
body
├── #landscape-warning          — 横屏提示（CSS 媒体查询控制显隐）
└── #app                        — 主容器（flex column, 100dvh）
    ├── #loading-overlay        — 加载遮罩（z-index: 10000）
    │   ├── .loader             — 旋转动画
    │   └── <p>                 — "加载中..."
    ├── header                  — 标题区（flex-shrink: 0）
    │   └── h1                  — "拼图挑战"
    ├── main#puzzle-area        — 拼图区域（flex: 1, 相对定位基准）
    │   ├── #grid               — 九宫格目标区域（绝对定位）
    │   │   └── .grid-cell ×9   — 目标格子（CSS Grid 子项）
    │   ├── button#ref-toggle   — 参考图切换按钮（z-index: 100）
    │   ├── #ref-preview        — 参考图预览浮层（z-index: 100）
    │   └── .puzzle-piece ×9    — 拼图碎片（绝对定位，JS 动态创建）
    ├── footer                  — 底部操作区（flex-shrink: 0）
    │   ├── #btn-restart        — 重新开始按钮
    │   └── #progress           — 进度文本 "3 / 9"
    └── #completion-overlay     — 完成庆祝遮罩（z-index: 9000）
        └── .completion-content — 庆祝内容（动画容器）
            ├── .completion-icon — ✓ 图标
            ├── h2              — "恭喜完成!"
            └── #btn-play-again — 再来一局按钮
```

### 2.2 z-index 层级规范

| 层级 | z-index 值 | 用途 |
|:---|:---|:---|
| 拼图碎片（普通） | 10+ 递增 | `state.zCounter` 控制，每次触摸递增 |
| 拖动中碎片 | 9999 | `.dragging` 类强制置顶 |
| 参考图组件 | 100 | `#ref-toggle` + `#ref-preview` |
| 完成遮罩 | 9000 | `#completion-overlay` |
| 加载遮罩 | 10000 | `#loading-overlay` |
| 横屏提示 | 20000 | `#landscape-warning` |

> ⚠️ **改造约束**: 新增浮层组件的 z-index 必须在对应层级范围内，不得干扰拼图碎片的拖动层级（10~9999）。

---

## 3. CSS 设计系统

### 3.1 CSS 变量清单

```css
:root {
    --bg-primary: #0a0e1a;          /* 页面主背景 */
    --bg-surface: #141b2d;          /* 卡片/按钮背景 */
    --bg-elevated: #1e293b;         /* 悬浮/激活态背景 */
    --text-primary: #e2e8f0;        /* 主文本 */
    --text-secondary: #94a3b8;      /* 次要文本 */
    --accent: #e8a838;              /* 主强调色（琥珀金） */
    --accent-glow: rgba(232,168,56,0.25);
    --success: #4ade80;             /* 成功色（吸附/完成） */
    --success-glow: rgba(74,222,128,0.25);
    --border: rgba(255,255,255,0.08);       /* 通用边框 */
    --border-accent: rgba(232,168,56,0.35); /* 强调边框 */
    --radius: 6px;                  /* 统一圆角 */
    --shadow-piece: 0 4px 16px rgba(0,0,0,0.5);   /* 碎片默认阴影 */
    --shadow-drag: 0 12px 32px rgba(0,0,0,0.6);   /* 拖动时阴影 */
}
```

### 3.2 关键 CSS 类状态机

```
.puzzle-piece 状态转换:

  [初始] ──→ .entrance (淡入动画, 自动移除)
    │
    ├──→ .dragging (拖动中: scale(1.06), z-index:9999)
    │       │
    │       ├──→ [吸附成功] 移除 .dragging → 添加 .snapped → .snap-anim (脉冲)
    │       └──→ [松手未吸附] 移除 .dragging
    │
    └──→ .snapped (已吸附: 绿色边框阴影)
            │
            └──→ [再次触摸] 移除 .snapped → .dragging (可取下)
```

### 3.3 动画清单

| 动画名 | 持续时间 | 应用场景 | 关键帧 |
|:---|:---|:---|:---|
| `pieceFadeIn` | 0.4s | 碎片入场 | scale(0.7)→scale(1), opacity(0)→opacity(1) |
| `snapPulse` | 0.35s | 吸附成功脉冲 | box-shadow 从 0 扩展到 10px 消散 |
| `scaleIn` | 0.5s | 完成弹窗入场 | scale(0.5)→scale(1), 弹性曲线 |
| `spin` | 0.8s | 加载旋转 | rotate(0)→rotate(360deg) |

### 3.4 响应式策略

- **九宫格宽度**: `min(屏幕宽度 × 72%, 屏幕高度 × 48%)` — 保证上下留有散落空间
- **安全区域适配**: `env(safe-area-inset-top/bottom)` — 适配刘海屏/底部横条
- **横屏拦截**: `@media (orientation:landscape)` 隐藏 `#app`，显示 `#landscape-warning`
- **视口单位**: `100dvh` — 动态视口高度，避免移动端地址栏影响

---

## 4. CONFIG 配置项详解

```javascript
var CONFIG = {
    GRID_SIZE: 3,              // 网格维度（3 = 3×3 = 9块）
    GAP: 3,                    // 网格单元格间距（px）
    SNAP_THRESHOLD: 25,        // 吸附判定距离（px），碎片中心与目标中心距离 < 此值触发吸附
    GRID_WIDTH_RATIO: 0.72,    // 九宫格占屏幕宽度比例
    IMAGE_URL: null,           // 外部图片 URL，null 时使用 Canvas 生成默认图
    SCATTER_PADDING: 8,        // 散落区域边距（px），防止碎片贴边
    SNAP_TRANSITION_MS: 120,   // 吸附位移动画时长（ms）
    ENTRANCE_STAGGER_MS: 60,   // 入场动画逐个延迟（ms）
    HINT_DISTANCE: 50,         // 接近提示触发距离（px），应 > SNAP_THRESHOLD
    IMAGE_GEN_SIZE: 600        // 默认图生成尺寸（px），正方形
};
```

### 4.1 配置项依赖关系

```
GRID_SIZE ──→ 影响碎片数量、网格行列数、进度显示
GAP ──→ 影响 pieceSize 计算、gridWidth 计算、getTargetPos 计算
GRID_WIDTH_RATIO ──→ 影响 gridWidth 上限
SNAP_THRESHOLD ──→ 必须小于 HINT_DISTANCE（否则提示无意义）
SNAP_TRANSITION_MS ──→ 影响 checkSnap 中 setTimeout 的延迟
SCATTER_PADDING ──→ 影响散落区域的有效范围
```

### 4.2 修改配置项的注意事项

| 配置项 | 修改风险 | 说明 |
|:---|:---|:---|
| `GRID_SIZE` | 🔴 高 | 改为 4 或 5 需同步修改 CSS `grid-template-columns/rows`，且散落算法需重新验证空间是否足够 |
| `GAP` | 🟡 中 | 改大后 pieceSize 变小，需确保碎片仍可辨识；改小可能使缝隙不可见 |
| `SNAP_THRESHOLD` | 🟡 中 | 过大导致误吸附，过小导致难以吸附；建议范围 15~35px |
| `GRID_WIDTH_RATIO` | 🟡 中 | 过大导致上下散落空间不足，过小导致碎片看不清 |
| `IMAGE_URL` | 🟢 低 | 需确保图片为正方形、支持 CORS、建议 600×600px |

---

## 5. State 状态对象详解

```javascript
var state = {
    pieces: [],        // PieceData[] — 所有碎片数据对象数组
    pieceSize: 0,      // number — 单个碎片边长（px），由 calculateLayout 计算
    gridWidth: 0,      // number — 九宫格总宽度（px），含 GAP
    gridLeft: 0,       // number — 九宫格左上角 X 坐标（相对于 puzzle-area）
    gridTop: 0,        // number — 九宫格左上角 Y 坐标（相对于 puzzle-area）
    areaWidth: 0,      // number — puzzle-area 可用宽度
    areaHeight: 0,     // number — puzzle-area 可用高度
    completed: false,   // boolean — 游戏是否已完成
    activePiece: null,  // PieceData | null — 当前正在拖动的碎片
    offsetX: 0,        // number — 触摸点与碎片左上角的 X 偏移
    offsetY: 0,        // number — 触摸点与碎片左上角的 Y 偏移
    zCounter: 10,      // number — z-index 递增计数器
    imageUrl: null      // string — 最终使用的图片 URL（外部或 data:URL）
};
```

### 5.1 PieceData 数据结构

```javascript
{
    el: HTMLElement,     // 碎片 DOM 元素引用
    index: number,       // 原始索引（0~8），row * GRID_SIZE + col
    row: number,         // 在九宫格中的行号（0~2）
    col: number,         // 在九宫格中的列号（0~2）
    snapped: boolean,    // 是否已吸附到正确位置
    targetX: number,     // 正确目标位置的 X 坐标（px）
    targetY: number,     // 正确目标位置的 Y 坐标（px）
    currentX: number,    // 当前 X 坐标（px）
    currentY: number     // 当前 Y 坐标（px）
}
```

> ⚠️ **关键约束**: `targetX/targetY` 在 `createPieces` 时一次性计算，之后不再变化。`currentX/currentY` 在拖动和吸附时实时更新。两者的坐标系统一基于 `#puzzle-area` 的左上角原点。

---

## 6. 功能模块实现细节

### 6.1 图片加载与默认图生成

**函数**: `loadImage(callback)` / `generateDefaultImage(size)`

**数据流**:
```
CONFIG.IMAGE_URL 是否为 null?
├── 是 → setTimeout 100ms → generateDefaultImage(600) → data:image/jpeg;base64,...
│         （延迟确保 DOM 渲染完成）
└── 否 → new Image() → crossOrigin='anonymous'
         ├── onload → callback(IMAGE_URL)
         └── onerror → callback(generateDefaultImage(600))
             （外部图加载失败时降级为默认图）
```

**默认图生成细节**:
- Canvas 600×600px
- 对角线渐变：`#667eea → #764ba2 → #f093fb → #f5576c`
- 7 个半透明白色圆形装饰
- 6 条放射线（每 60° 旋转）
- 中心小圆点
- 输出为 JPEG quality=0.92 的 data URL

> ⚠️ **改造注意**: 若需支持自定义默认图，修改 `generateDefaultImage` 即可。若需支持多张图片轮换，在 `loadImage` 中添加图片数组逻辑。

### 6.2 布局计算引擎

**函数**: `calculateLayout()`

**计算流程**:
```
1. 获取 puzzle-area 的 getBoundingClientRect()
   → state.areaWidth, state.areaHeight

2. 计算九宫格最大宽度
   → maxGridW = areaWidth × GRID_WIDTH_RATIO (72%)
   → maxGridH = areaHeight × 0.48 (48%，保证上下有散落空间)
   → gridWidth = min(maxGridW, maxGridH)
   → gridWidth = Math.floor(gridWidth)  // 取整避免亚像素

3. 反向计算 pieceSize（扣缝隙）
   → pieceSize = floor((gridWidth - (GRID_SIZE-1) × GAP) / GRID_SIZE)

4. 重新计算精确 gridWidth（基于 pieceSize 反推）
   → gridWidth = pieceSize × GRID_SIZE + (GRID_SIZE-1) × GAP
   // 保证 pieceSize 是整数，gridWidth 可能比初始值小 1~2px

5. 计算九宫格居中位置
   → gridLeft = floor((areaWidth - gridWidth) / 2)
   → gridTop = floor((areaHeight - gridWidth) / 2)
```

> ⚠️ **关键公式**: `pieceSize = floor((gridWidth - (N-1) × GAP) / N)`  
> 此公式确保 N 个碎片 + (N-1) 个缝隙恰好填满 gridWidth，无余量。  
> **不可改为 ceil**，否则碎片总宽度会超出 gridWidth 导致溢出。

### 6.3 网格创建与图片提示

**函数**: `createGrid()`

**实现要点**:

1. **网格容器定位**: 绝对定位，`left/top` 由 `calculateLayout` 计算得出
2. **CSS Grid 布局**: `grid-template-columns/rows: repeat(3, 1fr)` + `gap: GAPpx`
3. **图片提示**: 每个 `.grid-cell` 通过 CSS 自定义属性传递图片信息

**图片提示的 CSS 实现原理**:
```css
.grid-cell::before {
    content: '';
    position: absolute;
    inset: 0;
    opacity: 0.08;                          /* 极低透明度，仅作暗示 */
    background-size: var(--bg-full);        /* = pieceSize × 3，整图尺寸 */
    background-image: var(--hint-img);      /* = url(图片地址) */
    background-position: var(--hint-pos);   /* = -col*pieceSize px -row*pieceSize px */
    pointer-events: none;                   /* 不拦截触摸事件 */
}
```

**为什么用伪元素而非直接背景**:
- `.grid-cell` 自身有 `border: 1.5px dashed`，若直接设背景+透明度，边框也会变透明
- 伪元素独立控制透明度，边框保持可见

**CSS 变量传递链**:
```
JS: cell.style.setProperty('--hint-img', 'url(...)')
JS: cell.style.setProperty('--hint-pos', '-col*ps px -row*ps px')
JS: gridEl.style.setProperty('--bg-full', 'ps*3 px')
CSS: .grid-cell::before 读取这三个变量
```

> ⚠️ **改造约束**: 若需修改提示透明度，调整 `.grid-cell::before` 的 `opacity` 值。不可改为直接在 `.grid-cell` 上设 `background`，否则边框可见性受损。

### 6.4 拼图块创建与图片切割

**函数**: `createPieces()`

**图片切割原理**（background-position 方案）:

每个碎片是一个 `div.puzzle-piece`，通过 CSS `background` 属性显示原图的对应区域：

```javascript
el.style.backgroundImage = 'url(' + state.imageUrl + ')';
el.style.backgroundSize = bgFull + ' ' + bgFull;     // = pieceSize×3 px
el.style.backgroundPosition = (-col * pieceSize) + 'px ' + (-row * pieceSize) + 'px';
```

**视觉原理**:
```
原图 (600×600) 被 backgroundSize 缩放到 (pieceSize×3)×(pieceSize×3)
碎片 div 尺寸 = pieceSize × pieceSize（只显示一部分）
backgroundPosition 负偏移 = 将原图的对应区域移入 div 可视范围

例: 第(1,2)块（第2行第3列）
  backgroundPosition = (-2 × pieceSize)px (-1 × pieceSize)px
  显示原图右下角区域
```

**碎片创建顺序**:
1. 生成索引数组 [0,1,2,...,8]，Fisher-Yates 洗牌
2. 按洗牌顺序创建碎片，z-index 递增
3. 每个碎片绑定 `touchstart` + `mousedown` 事件

> ⚠️ **改造注意**: 碎片的 `backgroundSize` 必须等于 `pieceSize × GRID_SIZE`，而非 `gridWidth`。因为 `gridWidth = pieceSize × GRID_SIZE + (GRID_SIZE-1) × GAP`，含缝隙。若用 `gridWidth` 会导致图片被轻微拉伸。

### 6.5 散落位置生成算法

**函数**: `calculateScatterPositions()`

**算法流程**:

```
1. 定义九宫格周围的安全散落区域（zones）
   ├── 上方区域: y ∈ [pad, gridTop - ps - margin]
   ├── 下方区域: y ∈ [gridBottom + margin, areaHeight - ps - pad]
   ├── 左侧区域: x ∈ [pad, gridLeft - ps - margin]
   └── 右侧区域: x ∈ [gridRight + margin, areaWidth - ps - pad]

2. 每个区域添加有效性验证（maxX > min, maxY > min）
   → 防止小屏设备出现负尺寸区域

3. 若所有区域均无效，降级为全屏散落
   → zones = [{ minX:pad, maxX:areaWidth-ps-pad, minY:pad, maxY:areaHeight-ps-pad }]

4. 对每个碎片，尝试最多 40 次随机放置:
   a. 轮询选择 zone（attempt % zones.length）
   b. 在 zone 内随机生成 (x, y)
   c. clamp 到屏幕安全范围
   d. 检查与已放置碎片的中心距离 < ps × 0.45 → 重叠，继续尝试
   e. 不重叠 → 采用此位置

5. 40 次尝试后仍未找到不重叠位置 → 使用最后一次的坐标（允许部分重叠）
```

**防重叠判定**:
```javascript
dist(cx1, cy1, cx2, cy2) < pieceSize × 0.45
// 两碎片中心距离 < 碎片边长 × 0.45
// 0.45 约等于对角线的一半略小，允许部分重叠但不完全遮挡
```

> ⚠️ **改造风险**: 若 `GRID_SIZE` 改为 4（16块）或 5（25块），碎片数量增加，散落空间可能不足。需考虑:
> - 降低 `GRID_WIDTH_RATIO` 使九宫格更小
> - 允许碎片部分遮挡九宫格
> - 缩小碎片尺寸

### 6.6 触摸拖动与边界限制

**函数**: `onPointerStart(e, piece)` / `onPointerMove(e)` / `onPointerEnd(e)`

**事件架构**:

| 事件 | 绑定位置 | passive | 说明 |
|:---|:---|:---|:---|
| `touchstart` | 每个 `.puzzle-piece` | `false` | 需 `preventDefault` 阻止 300ms 延迟 |
| `mousedown` | 每个 `.puzzle-piece` | 默认 | PC 兼容 |
| `touchmove` | `document` | `false` | 需 `preventDefault` 阻止页面滚动 |
| `mousemove` | `document` | 默认 | PC 兼容 |
| `touchend/touchcancel` | `document` | 默认 | 手指抬起 |
| `mouseup` | `document` | 默认 | PC 兼容 |

**拖动坐标计算**:

```
onPointerStart:
  offsetX = touchX - areaRect.left - piece.currentX
  offsetY = touchY - areaRect.top  - piece.currentY
  // 记录触摸点与碎片左上角的偏移，避免碎片瞬移到手指下方

onPointerMove:
  newX = touchX - areaRect.left - offsetX
  newY = touchY - areaRect.top  - offsetY
  newX = clamp(newX, 0, areaWidth - pieceSize)    // 左右边界
  newY = clamp(newY, 0, areaHeight - pieceSize)    // 上下边界
  piece.currentX = Math.round(newX)                // 取整避免亚像素抖动
  piece.currentY = Math.round(newY)
```

**边界震动反馈**:
```javascript
if (atBoundary && navigator.vibrate) {
    navigator.vibrate(8);  // 8ms 轻震，兼容性检查
}
```

**层级管理**:
```javascript
piece.el.style.zIndex = ++state.zCounter;  // 每次触摸递增，最后触摸的碎片在最上层
// .dragging 类强制 z-index: 9999，拖动时覆盖所有其他碎片
```

> ⚠️ **改造约束**: 
> - `touchmove` 的 `{ passive: false }` 不可移除，否则页面会在拖动时滚动
> - `onPointerMove` 绑定在 `document` 而非碎片上，这是为了手指滑出碎片后仍能追踪
> - 坐标计算基于 `puzzleArea.getBoundingClientRect()`，若改变容器结构需同步修改

### 6.7 精准吸附功能

**函数**: `checkSnap(piece)` → 返回 `boolean`

**判定逻辑**:
```javascript
var pcx = piece.currentX + pieceSize / 2;   // 碎片中心 X
var pcy = piece.currentY + pieceSize / 2;   // 碎片中心 Y
var tcx = piece.targetX + pieceSize / 2;    // 目标中心 X
var tcy = piece.targetY + pieceSize / 2;    // 目标中心 Y
var d = dist(pcx, pcy, tcx, tcy);           // 欧氏距离

if (d < SNAP_THRESHOLD) { ... }  // 仅判断碎片自己的目标位置
```

**吸附动画时序**:
```
T=0ms:   移除 .dragging 类（缩放还原开始）
         设置 transition: left/top/transform/box-shadow 120ms ease-out
         设置 left/top 为目标坐标（位移动画开始）
         ── 缩放还原与位移动画同步进行 ──

T=120ms: transition 结束，碎片到达目标位置
         切换 transition 为 transform/box-shadow 0.12s
         添加 .snap-anim 类（脉冲动画开始）

T=470ms: 移除 .snap-anim 类（350ms 脉冲动画结束）
```

> ⚠️ **关键实现细节**: `checkSnap` 返回布尔值，`onPointerEnd` 根据返回值决定是否移除 `.dragging` 类。吸附时在 `checkSnap` 内部移除 `.dragging`，确保缩放还原与位移动画同步。若在 `onPointerEnd` 中统一移除，会导致缩放瞬间还原而位移还在动画中，造成视觉跳跃。

### 6.8 可取下重拼机制

**实现位置**: `onPointerStart` 函数内

```javascript
if (piece.snapped) {
    piece.snapped = false;                    // 清除已吸附标记
    piece.el.classList.remove('snapped');      // 移除绿色边框
    updateProgress();                          // 更新进度（减少计数）
}
```

**设计原则**:
- ❌ 不移除触摸事件监听
- ❌ 不添加 `pointer-events: none`
- ❌ 不锁定碎片位置
- ✅ 仅修改 `snapped` 状态标记和视觉类
- ✅ 拖动时自动清除吸附状态

> ⚠️ **改造约束**: 绝对不可在吸附后对碎片添加任何阻止触摸的 CSS 属性或移除事件监听，否则"可取下"功能失效。

### 6.9 接近提示系统

**函数**: `updateHint(piece)` / `clearAllHints()`

**触发条件**: 拖动过程中，碎片中心与其目标中心距离 < `HINT_DISTANCE`（50px）

**提示效果**: 对应的 `.grid-cell` 添加 `.cell-hint` 类:
```css
.grid-cell.cell-hint {
    border-color: var(--accent);              /* 琥珀色边框 */
    background-color: rgba(232,168,56,0.06);  /* 淡琥珀色背景 */
}
```

**提示清除时机**:
- 拖动过程中：每次 `onPointerMove` 先 `clearAllHints`，再判断是否添加新提示
- 拖动结束时：`onPointerEnd` 中 `clearAllHints`

> ⚠️ **注意**: `HINT_DISTANCE`（50px）必须大于 `SNAP_THRESHOLD`（25px），否则提示区域完全被吸附区域覆盖，提示无意义。

### 6.10 完成检测与庆祝动画

**函数**: `checkCompletion()` / `showCompletion()`

**检测逻辑**:
```javascript
state.pieces.every(function(p) { return p.snapped; })
// 所有碎片的 snapped 标记均为 true 时触发
```

**庆祝动画重播机制**:
```javascript
function showCompletion() {
    var content = completionOverlay.querySelector('.completion-content');
    content.style.animation = 'none';     // 重置动画
    content.offsetHeight;                  // 强制回流，使重置生效
    content.style.animation = '';          // 恢复动画（重新触发 CSS animation）
    completionOverlay.classList.add('active');
}
```

> ⚠️ **修复记录**: 首次实现时未包含动画重置逻辑，导致重新游戏后再次完成时动画不播放。通过 `animation: none` + `offsetHeight` + `animation: ''` 三步强制重播。

### 6.11 参考图预览

**交互逻辑**:
```
点击 #ref-toggle:
  refVisible = !refVisible
  ├── true  → #ref-preview 添加 .visible 类（淡入 + 放大）
  └── false → #ref-preview 移除 .visible 类（淡出 + 缩小）

触摸屏幕其他区域:
  if (refVisible && 触摸目标不是 ref-preview 且不是 ref-toggle)
  → refVisible = false, 隐藏预览
```

**图片设置**: 在 `loadImage` 回调中，`refPreview.style.backgroundImage = 'url(' + imageUrl + ')'`

**重置时**: `refVisible = false; refPreview.classList.remove('visible');`

### 6.12 游戏重置

**函数**: `resetGame()`

**重置流程**:
```
1. 隐藏完成遮罩
2. 重置 state: completed=false, activePiece=null, zCounter=10
3. 隐藏参考图预览
4. 移除所有旧碎片 DOM (p.el.remove())
5. 清空 state.pieces 数组
6. 重新执行完整初始化链:
   calculateLayout() → createGrid() → createPieces()
   → calculateScatterPositions() → applyScatterPositions() → updateProgress()
```

> ⚠️ **内存管理**: 旧碎片通过 `el.remove()` 从 DOM 移除。由于碎片的事件监听器是通过闭包引用 `pieceData` 对象的，而 `pieceData` 被 `state.pieces` 引用，清空 `state.pieces = []` 后，旧碎片及其监听器均可被 GC 回收。

---

## 7. 数据流与生命周期

### 7.1 初始化流程

```
DOM Ready
  │
  └── init()
       ├── 绑定 document 级事件 (touchmove/move/touchend/up)
       ├── 绑定按钮事件 (restart/play-again)
       ├── 绑定参考图事件 (ref-toggle/touchstart)
       │
       └── loadImage(callback)
            │
            ├── [有 IMAGE_URL] → Image.onload/onerror → callback(url)
            └── [无 IMAGE_URL] → setTimeout 100ms → callback(defaultDataUrl)
                  │
                  └── callback(imageUrl)
                       ├── state.imageUrl = imageUrl
                       ├── refPreview.style.backgroundImage = url
                       ├── calculateLayout()
                       ├── createGrid()
                       ├── createPieces()
                       ├── calculateScatterPositions()
                       ├── applyScatterPositions()
                       ├── updateProgress()
                       └── setTimeout 300ms → loadingOverlay.classList.add('hidden')
```

### 7.2 游戏循环状态机

```
                    ┌─────────────────────────────┐
                    │         LOADING             │
                    │  (loading-overlay 可见)      │
                    └────────────┬────────────────┘
                                 │ 图片加载完成
                    ┌────────────▼────────────────┐
              ┌────►│         PLAYING             │◄────┐
              │     │  (碎片散落，等待拖动)        │     │
              │     └────┬───────────┬────────────┘     │
              │          │           │                   │
              │   拖动碎片  │     所有碎片吸附            │
              │          │           │                   │
              │   ┌──────▼──────┐   │                   │
              │   │  DRAGGING   │   │                   │
              │   │ (activePiece│   │                   │
              │   │  不为 null) │   │                   │
              │   └──────┬──────┘   │                   │
              │          │           │                   │
              │    松手   │           │                   │
              │          │           │                   │
              │   ┌──────▼──────┐   │                   │
              │   │  SNAP_CHECK │   │                   │
              │   │ (checkSnap) │   │                   │
              │   └──┬──────┬──┘   │                   │
              │      │      │       │                   │
              │  未吸附  已吸附     │                   │
              │      │      │       │                   │
              │      │  snapped=true │                   │
              │      │  updateProgress                  │
              │      │      │       │                   │
              │      │      ├── 不全完成 ──► PLAYING    │
              │      │      │                           │
              │      │      └── 全完成 ──┐              │
              │      │                   │              │
              │      │        ┌──────────▼─────────┐   │
              │      │        │     COMPLETED       │   │
              │      │        │ (completion-overlay │   │
              │      │        │      可见)          │   │
              │      │        └──────────┬─────────┘   │
              │      │                   │              │
              │      │            点击"再来一局"        │
              │      │                   │              │
              │      │        ┌──────────▼─────────┐   │
              │      │        │    resetGame()      │───┘
              │      │        │  清空 → 重新初始化   │
              │      │        └────────────────────┘
              │      │
              └──────┘ (碎片留在松手位置)
```

### 7.3 单次拖动的数据流

```
touchstart/mousedown on piece
  │
  ├── state.activePiece = piece
  ├── 计算 offsetX/offsetY (触摸点与碎片左上角偏移)
  ├── 若 piece.snapped → 清除 snapped 状态
  ├── 添加 .dragging 类
  ├── z-index 递增
  └── clearAllHints()
       │
       ▼ (连续触发)
touchmove/mousemove on document
  │
  ├── 计算 newX/newY = touchPos - areaOffset - offsetXY
  ├── clamp 到 [0, areaWidth-pieceSize] / [0, areaHeight-pieceSize]
  ├── 边界震动反馈
  ├── 更新 piece.currentX/currentY
  ├── 更新 el.style.left/top
  └── updateHint(piece)
       │
       ▼
touchend/mouseup on document
  │
  ├── state.activePiece = null (先清空，防止重入)
  ├── checkSnap(piece)
  │    ├── 吸附成功 → 移除 .dragging, 设 transition, 移动到目标,
  │    │             设 snapped=true, 添加 .snapped, snap-anim,
  │    │             updateProgress, checkCompletion → return true
  │    └── 未吸附 → return false
  ├── 若未吸附 → 移除 .dragging 类
  └── clearAllHints()
```

---

## 8. 事件绑定架构

### 8.1 事件绑定汇总

| 事件 | 目标 | 触发函数 | passive | 备注 |
|:---|:---|:---|:---|:---|
| `touchstart` | `.puzzle-piece` (×9) | `onPointerStart(e, piece)` | `false` | 需 preventDefault |
| `mousedown` | `.puzzle-piece` (×9) | `onPointerStart(e, piece)` | 默认 | PC 兼容 |
| `touchmove` | `document` | `onPointerMove(e)` | `false` | 需 preventDefault 阻止滚动 |
| `mousemove` | `document` | `onPointerMove(e)` | 默认 | PC 兼容 |
| `touchend` | `document` | `onPointerEnd(e)` | 默认 | — |
| `touchcancel` | `document` | `onPointerEnd(e)` | 默认 | — |
| `mouseup` | `document` | `onPointerEnd(e)` | 默认 | PC 兼容 |
| `click` | `#btn-restart` | `resetGame()` | — | — |
| `click` | `#btn-play-again` | `resetGame()` | — | — |
| `click` | `#ref-toggle` | 切换参考图 | — | — |
| `touchstart` | `document` | 关闭参考图 | `true` | 点击外部关闭 |

### 8.2 事件清理

`resetGame` 中通过 `p.el.remove()` 移除旧碎片 DOM。由于碎片的事件监听器是闭包引用，清空 `state.pieces = []` 后，旧碎片对象和监听器均可被 GC 回收。

document 级事件（touchmove/move/touchend/up）在 `init` 中绑定一次，永不移除，重置时无需处理。

---

## 9. 关键业务规则

### 9.1 吸附规则

| 规则 | 实现 | 不可修改原因 |
|:---|:---|:---|
| 仅判定碎片自己的目标位置 | `checkSnap` 只计算 `piece.targetX/Y` | 防止碎片吸附到错误位置 |
| 中心距离判定 | `dist(center1, center2) < THRESHOLD` | 比左上角距离更直觉 |
| 吸附后不锁定 | `onPointerStart` 中检查 `piece.snapped` 并重置 | 可取下重拼的核心 |
| 吸附动画同步 | `checkSnap` 内移除 `.dragging` + 统一 transition | 防止缩放与位移不同步 |

### 9.2 边界规则

| 规则 | 实现 |
|:---|:---|
| 碎片不超出 puzzle-area | `clamp(x, 0, areaWidth - pieceSize)` |
| 散落位置不超出 puzzle-area | `clamp(x, pad, areaWidth - ps - pad)` |
| 散落位置不遮挡九宫格 | zones 排除九宫格区域 |
| 散落位置防重叠 | 中心距离 < pieceSize × 0.45 判定 |

### 9.3 完成规则

| 规则 | 实现 |
|:---|:---|
| 所有碎片吸附才完成 | `pieces.every(p => p.snapped)` |
| 完成后不可拖动 | `onPointerStart` 检查 `state.completed` |
| 完成只触发一次 | `checkCompletion` 检查 `!state.completed` |
| 取下碎片后进度更新 | `onPointerStart` 中 `updateProgress()` |

---

## 10. 已知风险点与改造约束

### 10.1 🔴 高风险区域

| 风险点 | 描述 | 影响范围 | 规避方案 |
|:---|:---|:---|:---|
| `GRID_SIZE` 变更 | CSS `grid-template-columns/rows` 硬编码为 `repeat(3,1fr)` | 网格布局完全失效 | 需同步修改 CSS，或改为 JS 动态设置 |
| `background-position` 切割 | 依赖 `pieceSize × GRID_SIZE` 作为 backgroundSize | 图片显示错位 | 不可用 `gridWidth` 替代（含 GAP） |
| `touchmove` passive:false | 移除后页面会在拖动时滚动 | 拖动交互失效 | 不可改为 passive:true |
| 坐标系基于 puzzle-area | 所有坐标计算依赖 `puzzleArea.getBoundingClientRect()` | 坐标偏移 | 若改变 DOM 结构需重新校准 |

### 10.2 🟡 中等风险区域

| 风险点 | 描述 | 影响范围 | 规避方案 |
|:---|:---|:---|:---|
| 散落空间不足 | 小屏设备或大 GRID_SIZE 时，九宫格周围可能无足够空间 | 碎片重叠严重 | 降级为全屏散落（已实现 fallback） |
| `z-index` 无上限 | `state.zCounter` 持续递增，理论上可达 MAX_SAFE_INTEGER | 长时间游戏后层级异常 | 实际场景不可能触发，重置时归零 |
| 默认图无随机性 | `generateDefaultImage` 每次生成相同图案 | 视觉单调 | 可添加随机种子参数 |
| 完成检测时序 | `checkCompletion` 在 `checkSnap` 内调用，若 `updateProgress` 异步则可能漏检 | 当前同步调用无风险 | 保持同步调用链 |

### 10.3 🟢 低风险区域

| 风险点 | 描述 | 影响范围 | 规避方案 |
|:---|:---|:---|:---|
| CSS 变量命名 | `--bg-full` 仅在 `#grid` 上设置，子元素继承 | 若 grid 结构变化需重新传递 | 保持 grid 作为 CSS 变量宿主 |
| 动画重播 | `showCompletion` 需强制重置 animation | 重玩后动画不播放 | 已通过 offsetHeight 技巧修复 |

### 10.4 改造检查清单

在进行任何改造前，请逐项确认：

- [ ] 是否影响 `calculateLayout` 的尺寸计算链？
- [ ] 是否改变 `puzzle-area` 的定位上下文？
- [ ] 是否修改了 `touchmove` 的 passive 设置？
- [ ] 是否改变了 `checkSnap` 的返回值语义？
- [ ] 是否在吸附后添加了阻止触摸的属性？
- [ ] 是否修改了 `backgroundSize` 的计算公式？
- [ ] 是否改变了 `state.pieces` 的数据结构？
- [ ] 是否影响了 `resetGame` 的清理完整性？
- [ ] 新增 z-index 是否在合理层级范围内？
- [ ] 新增动画是否与现有 transition 冲突？

---

## 11. 性能特征与优化记录

### 11.1 性能指标

| 指标 | 目标值 | 当前实现 |
|:---|:---|:---|
| 首屏加载 | < 1s（WiFi）/ < 3s（弱网） | 单文件 + 默认图 Canvas 生成，无网络请求 |
| 拖动帧率 | 60fps | `will-change: transform,left,top` + `Math.round` 避免亚像素 |
| 内存占用 | < 10MB | 9 个 DOM 碎片 + 1 张图片，无 Canvas 常驻 |
| 资源大小 | < 500KB | 默认图 data URL ~50KB，无外部依赖 |

### 11.2 已实施的优化

| 优化项 | 技术 | 效果 |
|:---|:---|:---|
| 坐标取整 | `Math.round(newX)` | 避免亚像素渲染导致的模糊和抖动 |
| will-change 提示 | `will-change: transform,left,top` | 浏览器提前创建合成层 |
| 事件委托 | move/end 绑定在 document | 减少事件监听器数量，防止滑出碎片后丢失事件 |
| 被动监听 | 参考图关闭用 `{ passive: true }` | 不阻塞滚动 |
| 延迟隐藏加载遮罩 | `setTimeout 300ms` | 避免闪烁 |
| 入场动画错开 | `ENTRANCE_STAGGER_MS: 60` | 视觉节奏感，避免同时渲染 9 个动画 |

### 11.3 潜在优化方向

| 方向 | 描述 | 优先级 |
|:---|:---|:---|
| `requestAnimationFrame` | onPointerMove 中使用 rAF 节流 | 低（当前帧率已足够） |
| 图片懒加载 | 多图模式时按需加载 | 中（当前仅单图） |
| 触摸事件节流 | 高频 touchmove 中减少 DOM 查询 | 低（当前仅查 9 个 cell） |

---

## 12. 历史 Bug 修复记录

| 编号 | 问题描述 | 根因 | 修复方案 | 影响版本 |
|:---|:---|:---|:---|:---|
| BUG-001 | 网格单元格边框不可见 | `.grid-cell` 使用 `filter: opacity(0.08)` 导致边框也变透明 | 改用 `::before` 伪元素独立控制提示透明度 | v0.1 |
| BUG-002 | 网格 border 导致拼图块与格子 1px 偏移 | CSS Grid 的 `border` 参与布局计算，使单元格实际尺寸小于 pieceSize | 将 `border: 1px solid` 改为 `box-shadow: 0 0 0 1px`（不影响布局） | v0.2 |
| BUG-003 | 散落区域出现 NaN 坐标 | 小屏设备上 `maxX - minX` 可能为负值，`Math.random() × 负数` 产生 NaN | 每个区域添加 `if (maxX > min && maxY > min)` 验证 | v0.2 |
| BUG-004 | 吸附时碎片视觉跳跃 | `onPointerEnd` 中先移除 `.dragging`（缩放瞬间还原），再 `checkSnap`（位移动画开始），两者不同步 | `checkSnap` 返回布尔值，吸附时在内部统一移除 `.dragging` + 设置包含 `transform` 的 transition | v0.2 |
| BUG-005 | 重新游戏后完成动画不播放 | CSS animation 仅在首次添加类时触发，后续添加同类不会重播 | `showCompletion` 中通过 `animation:none` + `offsetHeight` + `animation:''` 强制重播 | v0.2 |
| BUG-006 | 外部图片无法加载（显示默认图） | `loadImage()` 中设置 `img.crossOrigin = 'anonymous'`，但外部图床未返回 `Access-Control-Allow-Origin` 响应头，浏览器直接拦截请求并触发 `onerror` 降级为默认图 | 移除 `img.crossOrigin = 'anonymous'`。因拼图使用 CSS `background-position` 切割方案（非 Canvas drawImage），无需读取像素数据，不要求 CORS | v0.3 |
| BUG-007 | 网格单元格背景提示残留 / 吸附后碎片绿色边框过重 | ① `.grid-cell::before` 伪元素以 `opacity:0.06` 显示原图对应区域作为提示，用户反馈干扰视觉；② `.snapped` 类的 `box-shadow: var(--shadow-snap)` 含 `0 0 0 2px var(--success)` 绿色边框，完成拼图后9块碎片均有明显绿框影响观感 | ① 将 `.grid-cell::before` 的 `opacity` 设为 `0` 隐藏提示；② 将 `.snapped` 的 box-shadow 改为无色边框版本，保留微弱阴影但不显示绿色轮廓线；同步调整 `#grid.complete-glow` 保持视觉一致性 | v0.3 |

---

> **文档维护说明**: 本文档应与代码同步更新。任何对 `index.html` 的功能改造或 Bug 修复，都需在对应章节补充记录，并在历史 Bug 修复记录中新增条目。
