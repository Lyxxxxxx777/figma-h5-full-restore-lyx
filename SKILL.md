---
name: figma-h5-full-restore-lyx
description: End-to-end pipeline for restoring an H5 activity page 1:1 from a Figma link. Use when the user provides a Figma URL and asks to restore/build/implement a full H5 page (with or without a PRD). Orchestrates Figma asset export, page implementation, and local visual self-check by enforcing `figma-implement-design` and `prd-implementation-precheck` underneath, and depends on Vibma MCP and Chrome DevTools MCP. Do NOT use for single-component restoration (use `figma-implement-design`), pure PRD/bugfix work without Figma restoration (use `prd-implementation-precheck` or `verification`), or reverse Figma authoring (use `figma-use`).
disable-model-invocation: false
---

# Figma H5 全流程还原 (LYX)

## Overview

整合 Figma 切图 → H5 页面实现 → 视觉自测 三步流水线的端到端 skill。
强制启用 `figma-implement-design` 与 `prd-implementation-precheck` 两个底层 skill 全程生效，依赖 Vibma MCP 与 Chrome DevTools MCP 完成切图导出与本地视觉验证。

**Skill 名称**：`figma-h5-full-restore-lyx`
**作者标识**：lyx
**适用场景**：用户提供 Figma 链接，要求按设计稿 1:1 还原 H5 活动页

---

## 🧠 核心心智模型（最重要）

> **Figma 是源代码，不是参考图。**

任何视觉决策前，先在心里过 3 个问题：

1. **这个元素在 Figma 是什么节点 ID？**
   - → 调 `Vibma.frames.get` 看节点结构（depth=2），不要凭整图视觉判断"这是一张图还是几张图"
2. **它的 `absoluteBoundingBox` 是多少？**
   - → 这是 CSS 定位的真相，不要用 `flex justify-content: center` 凭感觉对齐
3. **它有没有兄弟节点是另一种状态？（visible=false）**
   - → tab 的选中/未选中、按钮的常态/按下态，必须一次切齐

**违反这 3 条 = 必然返工**。本 skill 实战统计：未走"测绘先行"流程的页面平均返工 6-7 轮，走完整流程的页面 3 轮内收敛。

---

---

## Skill Boundaries

- **使用本 skill**：用户给出 Figma 链接 + 想要还原 H5 页面（不论是否带需求文档）
- **不使用本 skill**：
  - 用户只想做单组件还原 → 改用 `figma-implement-design`
  - 用户只是改 PRD / bug fix，不涉及 Figma 还原 → 改用 `prd-implementation-precheck` 或 `verification`
  - 用户要在 Figma 内反向操作（创建/修改节点）→ 改用 `figma-use`

---

## Required Inputs（必须先收集，缺一不可）

在执行任何步骤之前，必须先收集并校验以下输入。**任何必传项缺失都立即中断流程，提示用户补齐后重启 skill**（售卖收费式硬中断）。

### 输入清单

| 序号 | 名称 | 类型 | 必填 | 说明 |
|---:|---|---|:-:|---|
| 1 | **Figma 链接** | string | ✅ 必填 | 形如 `https://figma.com/design/:fileKey/:fileName?node-id=1-2`。链接必须定位到具体 frame（设计稿屏幕） |
| 2 | **目标页面范围** | enum | 默认 = "第一个页面" | 取值：`"第一个页面"` \| `"所有页面"` \| `"指定 frame 列表"`。"页面"指 Figma 中的 frame，不是 Page |
| 3 | **需求文档路径** | string \| null | ⚠️ 可选但必问 | 路径或粘贴内容。**即使可选，也必须主动询问用户是否有**，无则按"仅按视觉还原"模式执行 |

### 收集流程

```
START
  │
  ├─ 用户消息中是否包含 Figma 链接？
  │    ├─ 否 → ❌ 中断，提示："请提供 Figma 链接（必传）"
  │    └─ 是 → 提取 fileKey 与 nodeId，继续
  │
  ├─ 用户是否说明了实现范围？
  │    ├─ 否 → 提问："默认实现链接定位的第一个页面（frame），是否要实现所有页面？" 给三选项
  │    └─ 是 → 记录范围，继续
  │
  ├─ 用户是否提到需求文档？
  │    ├─ 否 → 主动询问："有需求文档/PRD 吗？无则按设计稿视觉为主实现，请确认"
  │    └─ 是 → 记录路径
  │
  └─ ✅ 全部收齐 → 进入"环境检查"
```

### 收集示例（用户初次输入不全时）

> ❌ **中断回复模板**（缺 Figma 链接）：
> 「检测到本次任务缺少必传参数，已中断流程。请补齐以下信息后重新发起：
> 1. ✅ Figma 链接（必传）：请提供具体定位到 frame 的 URL，如 `https://figma.com/design/xxx?node-id=1-2`
> 2. ⚙️ 目标范围（可选，默认实现第一个 frame）：第一个页面 / 所有页面 / 指定列表
> 3. 📄 需求文档（可选，影响交互逻辑还原精度）：路径 或 'no' 表示纯视觉还原」

---

## Required Environment（必须先检查，缺则提示安装）

在收齐输入后、执行步骤前，必须检查依赖环境。**只检查"是否可调用"，不尝试代装**。

### 依赖清单

| 类型 | 名称 | 用途 | 检查方式 |
|---|---|---|---|
| Skill | `figma-implement-design` | Figma → 代码标准工作流 | `use_skill({"skill_name": "figma-implement-design"})` 是否成功 |
| Skill | `prd-implementation-precheck` | PRD 实施预检流程 | `use_skill({"skill_name": "prd-implementation-precheck"})` 是否成功 |
| MCP | **Vibma** | Figma 设计稿读取 + 切图导出 | `search_tool({"query": "Vibma frames"})` 是否能返回工具 |
| MCP | **Chrome DevTools MCP** | 本地视觉验证 | `search_tool({"query": "Chrome DevTools take_screenshot"})` 是否能返回工具 |

### 检查流程

```
逐项调用 search_tool / use_skill 探测
  │
  ├─ 全部通过 → 进入步骤 1
  │
  └─ 有缺失 → ❌ 中断，输出"缺失依赖清单 + 安装文档链接"，请用户手动安装后重启 skill
```

### 缺失提示模板

> ⚠️ **依赖未就绪，已中断流程。请安装以下组件后重启：**
>
> | 缺失项 | 类型 | 安装文档 |
> |---|---|---|
> | `figma-implement-design` | Skill | 联系管理员或参考 CodeMaker Skills 文档 |
> | Vibma | MCP | https://vibma.dev/docs/install （示例） |
> | Chrome DevTools MCP | MCP | https://github.com/ChromeDevTools/mcp |
>
> AI 不会代装环境（MCP 与 Skill 装起来有环境依赖），请你在本地完成后重新发起任务。

---

## Project Environment Detection（项目约定嗅探）

在执行步骤前，自动嗅探目标项目的现有约定，找不到再问用户。

### 嗅探项

| 项目 | 嗅探方式 | 默认值（嗅探不到时使用） |
|---|---|---|
| 技术栈 | 读 `package.json` 的 `dependencies` 看 vue/react | 询问用户 |
| 设计稿基准宽 | 读现有页面 CSS 中的 `calc(... / N * 100vw)` | 750（H5 默认） |
| 资源根目录 | grep `figma-assets` / `src/assets/figma` | `figma-assets/` |
| Vite 别名 | 读 `vite.config.js` 的 `resolve.alias` | 提示用户加 `@figma` |
| 路由文件 | glob `src/router/**/*.{js,ts}` | 询问用户 |

如果嗅探不到关键项（如技术栈），主动问一句即可，不要中断。

---

## Workflow（核心三步流水线，严格执行）

### 第 0 步：一次性激活底层 Skill

```jsonc
use_skill({
  "skill_name": [
    "figma-implement-design",
    "prd-implementation-precheck"
  ]
})
```

> 💡 这两个底层 skill 在整个流水线全程生效，不需要在每步重新激活。

---

### 第 1 步：切图（Vibma 全量切图）

**目标**：把 Figma 链接中涉及的所有图都切出来，包括子组件层中的小元素图，全部存到项目的资源目录。

---

#### 🔬 1.A 测绘先行（Pre-Cut Survey）—— **切图前必做**

> **核心原则：Figma 是源代码，任何视觉决策先看节点结构 / 坐标 / 文案三件套，不要凭整图视觉判断。**

每个待切的复合模块（按钮、卡片、列表项、底栏等）在导出前，按以下三步"测绘"：

**1.A.1 读节点结构**

```jsonc
Vibma.frames.get({ id: nodeId, depth: 2 })
```

判断：
- 这是单一 RECTANGLE / IMAGE 节点 → 直接切整图
- 这是 GROUP 含多个 RECTANGLE/GROUP 子节点 → 必须递归看每个子节点的角色
- 含 `visible: false` 的兄弟节点 → 这通常是"未选中态/隐藏态"，必须切齐（见 1.D）

**1.A.2 读真实坐标**

```jsonc
Vibma.frames.get({ id: nodeId, fields: ["absoluteBoundingBox"], verbose: true })
```

记录每个子元素相对父级的 `(x, y, w, h)`。**这是 CSS 绝对定位的真相**，不要用 `flex justify-content: center` 等"看起来合理"的方式凭感觉对齐。

**1.A.3 读真实文案**

```jsonc
Vibma.frames.get({ id: textNodeId, fields: ["characters"] })
```

任何 TEXT 节点的文案都必须从这里读。**严禁自编 mock 文案**（自编必然偏离设计稿，必然返工）。

---

#### 🔪 1.B 切图（Cut）

**操作流程**：

1. 解析 Figma 链接：提取 `fileKey` 与 `nodeId`
2. 读节点结构：`Vibma.frames.get({ id: nodeId, depth: -1 })`
3. 完成 1.A 测绘，识别可切的子节点（按以下优先级递归）：
   - 整张 frame（用作设计稿参考图，存 `_full.png`）
   - 一级模块（如「PART 01 任务区」「PART 02 抽奖区」）
   - 卡片底板（**纯净底板，不含占位文字**）
   - 按钮、标题装饰、图标
   - 单个奖品 / 列表项 / 头像 / 立绘等
4. 逐个 `Vibma.frames.export({ id, format: "PNG", scale: 2, outputPath: "<绝对路径>" })`
5. 写入 `figma-assets/manifest.json`：每张切图记录 `{ nodeId, desc, usedIn }`

**文件分类规范**：

```
figma-assets/
├── manifest.json
├── pages/
│   └── <页面编号-页面名>/
│       ├── _full.png          # 整张页面参考图
│       └── <模块切图>.png      # 页面专属切图
└── components/
    ├── buttons/                # 按钮（不带文字的纯底图）
    ├── cards/                  # 卡片底板（纯净，不含占位文字）
    ├── icons/                  # 图标
    └── misc/                   # 标题装饰、tab、背景图等
```

---

#### 🎯 1.C 资源粒度规范（关键，避免返工最多的环节）

**按钮 / 卡板 / 标题装饰类组件，按以下规则拆切**：

| Figma 节点构成 | 切图策略 | 实现方式 |
|---|---|---|
| 单一 RECTANGLE 含 IMAGE fill | 整图导出 | 一个 `<img>` 即可 |
| 底板 RECTANGLE + 文字 GROUP（**两者并列为兄弟节点**）| 各切一张 | DOM 双层叠加：底板 absolute 平铺 + 文字图按真实坐标 absolute 浮于上层 |
| 整图含文字 + 国际化需求 | 切**纯净底板** + 文字用 DOM 渲染 | `<img>` 底板 + `<span>` 文字 |
| 整图自带文字 + 无国际化 | 整图导出 | `<img>` 单层，**严禁再叠 `<span>` 文字层** |

**🚨 反例**（实战返工原因）：

❌ 把 GROUP 节点（只含图标+文字）当作"完整按钮"导出，结果按钮没有底板  
❌ 切了带文字的 PNG，又因为"想方便维护/i18n"叠了 `<span>` 文字 → 文字双层重影  
❌ 用 `flex justify-content: center` 强行居中底板上的图标层，但 Figma 真实是偏移定位

**✅ 正例**：

```jsonc
// 1. 测绘
Vibma.frames.get({ id: "1:1393", fields: ["absoluteBoundingBox"] })
// → 底板：x=-1798, y=-695, w=206, h=62

Vibma.frames.get({ id: "1:1394", fields: ["absoluteBoundingBox"] })
// → 图标+文字：x=-1752, y=-685, w=114, h=36
// → 相对偏移：left=46, top=10

// 2. 切图（两张分开）
Vibma.frames.export({ id: "1:1393", outputPath: "buttons/btn-task-bg.png" })
Vibma.frames.export({ id: "1:1394", outputPath: "buttons/btn-task-share-icon.png" })

// 3. 实现（按真实坐标叠加）
<button class="btn-task">
  <img class="btn-task-bg" src="btn-task-bg.png" />
  <img class="btn-task-icon" src="btn-task-share-icon.png" />
</button>

.btn-task-bg { position: absolute; inset: 0; width: 100%; }
.btn-task-icon { position: absolute; left: vw(46); top: vw(13); height: vw(36); }
```

---

#### 🔀 1.D 多状态资源一次切齐

**场景**：tab/按钮的"选中/未选中态"在 Figma 中是兄弟节点，**只有一个 visible=true**，另一个 `visible: false`。

**操作流程**：

```jsonc
// 1. 列出所有状态节点
Vibma.frames.get({ id: 父GROUP, depth: 2 })
// → 找出所有 visible: false 的兄弟节点

// 2. 临时打开可见性
Vibma.frames.update({ items: [
  { id: "1:914", visible: true },
  { id: "1:927", visible: true }
]})

// 3. 导出所有状态
Vibma.frames.export({ id: "1:914", outputPath: "...-off.png" })
Vibma.frames.export({ id: "1:918", outputPath: "...-on.png" })
Vibma.frames.export({ id: "1:923", outputPath: "...-off.png" })
Vibma.frames.export({ id: "1:927", outputPath: "...-on.png" })

// 4. 立刻还原可见性（不污染 Figma 文件）
Vibma.frames.update({ items: [
  { id: "1:914", visible: false },
  { id: "1:927", visible: false }
]})
```

**⚠️ 关键**：导出后必须**立刻还原**原始可见性，避免污染设计稿。

---

#### 1.E 切图关键规则总结

- ❌ 严禁导出"卡片整图"作为底图使用（会带占位文字导致 DOM 重影）
- ❌ 严禁切了带文字 PNG 后再叠 `<span>` 文字（必然重影）
- ❌ 严禁导出"GROUP 节点（只含图标+文字）"当完整按钮使用（没底板）
- ❌ 严禁用 `flex/center` 强行居中需要绝对定位的图层
- ✅ 必须挑出"底板"子节点单独导出
- ✅ 必须读 `absoluteBoundingBox` 算相对坐标
- ✅ 必须读 TEXT 节点 `characters` 字段获取真实文案
- ✅ 多状态资源必须一次切齐（临时改 visible，导出后还原）
- ✅ 同一占位 IMAGE 节点（导出后文件大小完全一致）只保留一张

**完成判据**：所有切图都已落盘，manifest.json 已更新。

---

### 第 2 步：实现（按 PRD + 设计稿写代码）

**目标**：基于切图与需求文档，使用项目现有约定写出可运行的页面代码。

**子步骤 2.1：需求预检（仅当用户提供需求文档时执行）**

按 `prd-implementation-precheck` skill 的 Precheck Report 模板输出：
- Summary（1-2 句意图）
- ✅ Covered edge cases
- ⚠️ Missing edge cases
- 🔴 Blockers
- 🟡 Warnings
- Questions for User

**等待用户确认**："Proceed as-is, or update the PRD?"

**子步骤 2.2：代码实现**

根据"目标页面范围"决定执行节奏：

| 范围 | 执行方式 |
|---|---|
| `第一个页面` | 实现单个 frame 后进入第 3 步自测 |
| `所有页面` | **一口气全部跑完，不在每个页面后中断确认**，全部完成后再统一进入第 3 步 |
| `指定 frame 列表` | 按列表顺序实现，每个完成后短暂自测，全部完成后整体复检 |

**实现规则**（继承 `figma-implement-design`）：

1. 先 `read_file` 同项目已完成页面，沿用其样式约定（命名、单位、Teleport 模式等）
2. 先 `grep_search` 已有路由 / store / 组件，能复用就复用
3. 所有视觉元素用 `<img :src="切图" />` 渲染
4. ❌ 严禁用 CSS 渐变 / 纯色模拟按钮、卡板、标题装饰
5. ✅ CSS 仅做布局、间距、简单背景色
6. **坐标驱动定位**：复合元素（按钮、tab、徽章等）的内部子图层定位，必须按第 1.A 步测绘的 `absoluteBoundingBox` 做 `position: absolute + left/top` 精确还原，**严禁用 `flex justify-content: center` 凭感觉对齐**
7. **文案严格抄设计稿**：所有 TEXT 节点的 `characters` 都来自 Figma，**严禁自编文案**（如"完成后获得 1 次抽奖机会"等想当然写法）
8. **禁文字双层**：
   - 切图自带文字 → 直接 `<img>`，**严禁再叠 `<span>` 任何文字层**
   - 需要 i18n / 动态文案 → 切**纯净底板**（不含文字版），用 DOM 渲染文字
9. 缺切图时回到第 1 步用 Vibma 补切，**绝不用 CSS 模拟应付**

**子步骤 2.3：编译校验**

```bash
run_terminal_cmd("npm run dev")
```

- 检查启动是否报错
- 检查 console 是否有 import 错误、未定义变量等
- 有错则修复后再进第 3 步

**完成判据**：dev server 启动成功，目标页面无 console error。

---

### 第 3 步：自测（视觉对照 + 自动迭代）

**目标**：通过 Chrome DevTools MCP 截图与 Figma 设计稿做像素级对照，自动迭代直到趋近设计稿。

**操作流程**：

1. 设视口：`Chrome DevTools.resize_page({ width: 375, height: 812 })`
2. 跳转：`Chrome DevTools.navigate_page({ type: "url", url: "http://localhost:<port>/<route>" })`
3. **Console 双检**：`Chrome DevTools.list_console_messages({ types: ["error", "warn"] })` 必须 0 条
4. 截图：`Chrome DevTools.take_screenshot({ fullPage: true, filePath: "<项目根>/.preview/<页面名>-vN.png" })`
5. **分段对照**：长页面按 4 段切（顶部 / 中上 / 中下 / 底部），每段做"实现 vs Figma"并排画布对照（推荐用 `@napi-rs/canvas` 写个 `compare-screens.mjs` 脚本）
6. 输出"差异报告"：按下表分类列差异

**差异分类 → 修复策略对照表**（实战常见 7 类）：

| 差异类型 | 表现 | 修复策略 |
|---|---|---|
| 切图错用 | 按钮没底板 / 文字浮空 / 图层缺失 | 回第 1.A 步重新测绘节点结构，补切兄弟节点 |
| 文字双层重影 | 同一文字渲染两遍模糊 | 删 `<span>` 或换"纯底板"切图（见 1.C） |
| 占位文字混入底图 | 卡片底图自带"选手001"等 mock 字 | 切纯净底板节点，不要切包含文字的祖父 GROUP |
| 子图层位置偏移 | 按钮上图标偏左 / 飘出胶囊 | 回第 1.A 步读 `absoluteBoundingBox`，按真实坐标 absolute 定位 |
| 文案错 / 数据错 | 任务文案、奖品名、按钮字与 Figma 不一致 | 回第 1.A 步读 TEXT 节点 `characters`，更新 store 数据 |
| 多状态资源缺失 | tab 没有未选中态 / 切换无视觉变化 | 回第 1.D 步临时改 visible 切齐所有状态 |
| 尺寸 / 颜色 / 间距错 | 卡片宽度、字号、纯背景色与设计稿差异 | 直接 `edit` 改 CSS（vw 单位） |
| CSS 模拟应付 | 用渐变模拟按钮 / 卡板 / 标题 | ❌ **严禁通过**，必须用切图替换 |

**自动迭代规则**：

- 改完再走一次 1-5，截图命名递增 `-v2.png` `-v3.png`
- 每次返工必须更新 `manifest.json`（标注废弃资源 + 新增资源）
- **直到达成 100% 视觉对齐再停**（不限制迭代次数，但每轮简短汇报"修了什么 / 还差什么"）
- 单页面理想收敛：3 轮内（一次切齐资源 + 一次对照微调 + 一次 console + 视觉双检）

**已知忽略项**：

- `Teleport to body` 的 fixed 底部 Tab 在 fullPage 截图中位置漂移属于 Chrome headless 渲染 bug，真机正常，不当作差异（如果设计稿底栏本身就是页面流而非吸底，建议改用 `position: relative`）
- emoji 在 Chrome headless 截图中可能渲染为方块乱码，真机正常，不当作差异

**完成判据**：

- 截图与设计稿主要视觉元素 100% 对齐
- `manifest.json` 已同步本轮新增切图
- 清理临时预览文件（如 `preview-*.png`、过期 `-vN.png`）

---

## Output Expectations

每个步骤结束后输出简短汇报，格式如下：

```markdown
### 第 N 步：<步骤名> ✅

- 执行了什么（一句话）
- 产物（文件 / 切图 / 截图）
- 发现的问题（如有）
- 下一步动作
```

完整流程结束后，输出**总结报告**：

```markdown
## 任务完成 🎉

### 输入参数
- Figma 链接: ...
- 范围: ...
- 需求文档: ...

### 产出
- 切图: N 张（含 K 个子节点）
- 页面: M 个 frame 已实现
- 代码: <文件列表>

### 还需用户确认
- ⏳ <如有未决问题列在这里>
```

---

## 常见坑位（继承自 figma-implement-design 与本次实战）

### 切图相关

| 坑位 | 表现 | 根因 | 解决 |
|---|---|---|---|
| 切图整图含占位文字 | DOM 文字与图片文字重影 | 切了祖父 GROUP，包含了 mock 文字子节点 | 用 Vibma 挑"底板"子节点单独导出 |
| 把 GROUP（仅图标+文字）当完整按钮切 | 按钮没底板，图标飘空 | 没区分"底板 RECTANGLE + 图标 GROUP"两个并列兄弟节点 | 1.A 测绘读 depth=2 结构，底板和图标分开切 |
| 多状态资源缺失 | tab 切换无视觉变化 | 只切了 visible=true 的当前态，忽略 visible=false 的兄弟节点 | 1.D 临时改 visible 切齐，导出后立即还原 |
| 同一占位 IMAGE 重复切 | 多张资源文件大小完全相同 | Figma 多节点共用同一占位图 | 保留一张，manifest 标注复用关系 |
| Vibma 节点 visible=false 直接 export | 导出 ~149 字节空 PNG | Vibma 不会渲染隐藏节点 | 先 `frames.update({ visible: true })` 再 export |

### 实现相关

| 坑位 | 表现 | 根因 | 解决 |
|---|---|---|---|
| 切图自带文字又叠 `<span>` | 同一文字渲染两遍模糊重影 | "想方便维护"心理叠了 DOM 文字层 | 切图自带文字 → 严禁 `<span>`；要 i18n → 切纯净底板 |
| 子图层位置用 flex/center | 按钮上图标偏移 / 飘出胶囊 | 用"看起来合理"方式凭感觉对齐 | 1.A 读 `absoluteBoundingBox`，按真实坐标 absolute 定位 |
| 自编 mock 文案 | 任务标题、奖品名、按钮字与设计稿不一致 | 没读 TEXT 节点 `characters`，凭印象写 | 任何文案都用 `Vibma.frames.get({ fields: ["characters"] })` 读真实值 |
| 用 CSS 渐变模拟按钮/卡板 | 视觉粗糙、与设计稿色差大 | 缺切图时图省事用 CSS 应付 | ❌ 严禁，必须回第 1 步补切 |
| Vue `require()` 不生效 | 报错 "require is not defined" | Vite 不支持 require | 改用 `import xxx from '@figma/...'` 静态导入 |

### 截图自测相关

| 坑位 | 表现 | 根因 | 解决 |
|---|---|---|---|
| fullPage 截图中 fixed 元素漂移 | 吸底 Tab 出现在页面中部 | Chrome headless 渲染特性 | 真机正常可忽略；若设计稿本就非吸底，改 `position: relative` |
| emoji 显示成方块乱码 | 截图里 👍 ❤️ 变 □ | Chrome headless 缺 emoji font | 真机正常可忽略；若必须可视化，换 SVG 图标 |
| 整页太长一张图看不清差异 | 1500x7400 图缩到屏内细节糊 | 没分段对照 | 用 canvas 脚本切 4 段（顶/中上/中下/底）做并排对照 |

### 工具相关

| 坑位 | 表现 | 解决 |
|---|---|---|
| MCP 工具未激活 | `use_mcp_tool` 直接失败 | 先 `search_tool` 激活 |
| `edit` "old_string 不唯一" | 替换目标多处出现 | 扩大上下文或 `replace_all: true` |
| Windows 命令分页器卡住 | `git log` 等卡住 | 追加 ` \| cat` |

---

## Examples

### 示例 1：完整调用

> 用户：「用这个 figma 链接 https://figma.com/design/abc/xyz?node-id=1-464 还原 H5 页面，需求文档在 `docs/prd-event-page.md`，要求实现所有页面」

**AI 行为**：
1. ✅ 参数齐全（链接 + 范围"所有页面" + 需求文档），跳过中断
2. 检查 Skill / MCP → 全部就绪
3. 嗅探项目环境 → Vue3 + 750 基准宽 + `@figma` 别名已配
4. 激活 `figma-implement-design` + `prd-implementation-precheck`
5. 第 1 步：Vibma 全量切图，写 manifest
6. 第 2.1 步：读需求文档输出 Precheck Report，等用户确认
7. 第 2.2-2.3 步：依次实现所有 frame，启 dev server 校验
8. 第 3 步：每个页面截图对照 → 自动迭代 → 全部 100% 对齐
9. 输出总结报告

### 示例 2：参数不全（中断）

> 用户：「帮我做个 H5 页面」

**AI 行为**：直接输出"中断回复模板"，列出缺失的必传参数（Figma 链接），不进行任何后续操作。

### 示例 3：环境缺失（中断）

> 用户：「用 https://figma.com/design/.../?node-id=1-2 还原首页」（参数齐全）

**AI 行为**：
1. ✅ 参数齐全
2. ❌ 检查 MCP 时 Vibma 不可用 → 输出"依赖未就绪"模板，列出 Vibma 安装文档链接，中断

---

## 与底层 Skill 的关系

```
figma-h5-full-restore-lyx (本 skill)
  │
  ├─ 第 0 步一次性激活
  │   ├─ figma-implement-design  ─┐
  │   └─ prd-implementation-precheck ─┤  ← 全程生效，约束第 1-3 步
  │                                  │
  ├─ 第 1 步：Vibma 切图 ─────────────┤
  ├─ 第 2 步：写代码 + dev 校验 ──────┤
  └─ 第 3 步：Chrome DevTools 自测 ───┘
```

本 skill 的价值在于**串联整个端到端流程并强制参数 / 环境校验**，避免遗漏前置条件导致中途返工。
