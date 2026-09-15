---
tags: [ag-agent, choujiang, frontend, performance, mcp]
created: 2026-08-30
project: AG-Agent/choujiang
---

# 抽奖页面开发记录（choujiang 项目）
![image.jpg](images/image.jpg)
## 任务

按设计图高保真开发单文件 HTML 抽奖页面（`choujiang/index.html`，零外部依赖）：玻璃球内带匿名编号的彩球滚动，开始/停止按钮控制，停止后随机弹出中奖编号球。真实名单已移除，当前仅使用脱敏演示数据。

## 最终交付

- **文件**：`choujiang/index.html`（单文件，约 900 行）
- **运行**：`python -m http.server 8765`（choujiang 目录），访问 `http://localhost:8765`，强刷 Ctrl+F5
- **当前名单**：**31 个匿名演示编号**
- **功能**：物理滚动（重力+漩涡+碰撞）→ 停止随机中奖 → 飞球弧线动画 → 中奖球+结果面板+彩带爆发+合成音效

## 名单管理

### 数据源

- 原始数据源及人员明细已从项目文档移除
- 页面使用匿名编号，避免暴露真实身份
- 静态页面不连接数据库，演示名单变更需人工同步

### 变更历史

| 阶段 | 名单 | 说明 |
|---|---|---|
| 初始 | 匿名演示名单 | 未保留真实人员信息 |
| 当前 | 31 个匿名编号 | 用于页面功能演示 |

### 页面自适应逻辑

- 小球半径按人数分档：≤10→44px、≤16→38px、≤24→32px、≤36→27px、≤48→23px、>48→20px
- 12 色配色盘循环分配避免相邻同色；姓名字号随字数/球径自适应（下限 10px）

## 性能优化（核心经验）

### 问题：INP 2488ms（点击开始抽奖后卡死约 2.5 秒）

### 根因

**合成层内 box-shadow 动画迫使整层重光栅化**：点击开始 → 机身 shake（transform 动画）将机身提升为 GPU 合成层；同时球体 `box-shadow` 过渡在跑 → 合成层内容变化导致含 5 层大模糊阴影（90/130px 光晕）的整层每帧重新光栅化，弱核显上光栅化排队 2.5s，paint 无法产出。

### 修复清单

| 问题 | 修复 |
|---|---|
| 球体 box-shadow 过渡 | 独立 `.sphere-glow` 层，阴影静态烘焙、只过渡 opacity（纯 GPU 合成） |
| canvas 嵌在机身层内 | `#ballsCanvas` 加 `will-change: transform` 独立成层，隔离重绘范围 |
| 1180² 锥形渐变旋转层 | 改静态一次性光栅化（repeating-conic-gradient 不参与动画） |
| 按钮禁用态 filter 过渡 | 改 opacity（合成器友好） |
| `transition: all` | 显式只过渡 opacity + transform |
| 60 次 shadowBlur 文字烘焙 | 双层偏移文字（暗字+白字），快 5~10 倍 |
| resize 风暴 | rAF 合并 + 150ms 防抖重烘焙 |
| 全屏纸屑/球体画布 | dpr 上限 1.5 / 1.75 |

### 架构级优化（前一轮）

- **60 个 DOM 小球 → 单 Canvas sprite 烘焙**：球外观（渐变/暗环/高光/姓名）一次性画进离屏位图，运行时 60 次 drawImage，DOM 合成层从 60+ 降到 1
- **碰撞 O(n²) → sweep-and-prune**：按 x 排序剪枝（1770 对 → ~400 对）
- **静止休眠**：连续 45 帧低速即 sleep，跳过物理与重绘
- 隐藏元素动画 `animation-play-state: paused`，展示时才运行

### 验证结果

- 加载期/滚动期 longtask：0 个
- 点击开始 → 下次绘制 33ms；点击停止 → 36ms（良好阈值 200ms）

## 中奖提示音（Web Audio 合成，零音频文件）

- **飞行时**：上行琶音 C5→E5→G5→C6（三角波，期待感）
- **揭晓时**：C 大调高位和弦绽放 C6+E6+G6+C7（约 1s 衰减）
- **关键点**：AudioContext 必须在用户手势内 `resume()`（点击停止时预热），规避自动播放策略；失败静默降级

## 视觉还原关键常量（1600×900 舞台坐标系）

```
CX=590, CY=390, SPHERE_R=220    // 机器球心/玻璃球半径（居中偏左）
WX=1000, WY=328, WINNER_SIZE=210 // 中奖球中心/尺寸（紧贴玻璃球右侧）
```

- 整体 `transform: scale(var(--s))` 等比适配任意窗口；屏幕坐标经 getBoundingClientRect 换算
- 设计稿要素：银色铬金属支撑臂、四层金属底座+蓝色 LED 环、糖果质感按钮（蓝▶/红■）、金色立体标题+月桂枝、旋转已改静态的放射光束、星芒（clip-path 四角星）+金色粒子流

## 踩坑与经验

1. **Chrome 中 box-shadow 无法 GPU 合成**：「父层 transform 动画 + 子层 box-shadow 过渡」组合 = 每帧重光栅化整层大模糊阴影，弱核显 INP 可达 2.5s
2. favicon 404 → 内联 SVG data URI
3. 用户会手动编辑 NAMES 数组，**改文件前必须先 Read 确认当前名单**，避免覆盖手工修改
4. `will-change` 是双刃剑：滥用导致层爆炸，仅用于确实需要独立成层/逐帧更新的元素

## 验证方法（Browser agent 模式）

- URL 带版本参数绕过缓存（`?v=N`）
- 四态截图：初始/滚动/结果/持续 + closeup 特写
- 量化指标替代主观判断：FPS（rAF 帧计数）、longtask（PerformanceObserver）、点击延迟（点击→双 rAF 差值）、DOM 数量、边界溢出检测
- 每轮验证检查控制台零报错

## 相关链接

- 项目笔记：[[taste-skill安装记录-choujiang]]
- 名单数据源：已脱敏的本地演示名单
