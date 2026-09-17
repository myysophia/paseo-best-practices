# 浏览器/GUI Agent 方案对比（BrowserSkill · Computer Use · Midscene）

> 三种主流"让 AI 操作界面"方案的原理解析与选型指南。
>
> 适用场景：需要为 Agent/测试工程挑选浏览器或 GUI 自动化方案时；想搞懂"为什么有的方案能复用登录态、有的不能"时。

---

## 一、核心结论（TL;DR）

1. **三者解决的是不同问题，不是同一赛道的竞品**：
   - **BrowserSkill**（Tencent）：让日常 coding agent（Claude Code / Cursor / Codex…）**复用你已登录的浏览器**干活，且不打断你本人使用 —— "个人助理"路线。
   - **Computer Use**（Anthropic）：给模型一双"眼睛"一双手，操作**整台电脑**的任意应用 —— "通用机器人"路线。
   - **Midscene**（web-infra-dev，字节 Web Infra 团队维护）：**E2E 测试**为主的 GUI Agent，视觉定位 + 视觉断言 + HTML 报告，一套 Agent API 覆盖 Web / Android / iOS / HarmonyOS / 桌面应用 —— "自动化测试"路线。
2. **感知方式是分水岭**：Midscene 与 Computer Use 走视觉路线（截图 → 定位 → 动作），必须有视觉模型；BrowserSkill 走浏览器扩展 API 路线（结构化指令 + DOM），纯文本模型也能用。
3. **登录态复用**：BrowserSkill 是设计核心；Midscene 默认白板（Playwright 新建 context），但 Chrome 插件模式可直接在你当前浏览器里跑；Computer Use 取决于部署（官方 Docker 白板 / 真机模式复用但接管键鼠）。
4. **选型一句话**：让 Agent 用你已登录的网站干日常活 → **BrowserSkill**；写 E2E 测试或跨端（移动端）自动化 → **Midscene**；必须操作浏览器之外的原生应用 → **Computer Use**（沙箱部署）。

---

## 二、三方总览表

| 维度 | BrowserSkill | Computer Use | Midscene |
|---|---|---|---|
| 一句话定位 | 让 Agent 用你的真实浏览器，不打断你 | 让模型操作整台电脑 | GUI Agent for E2E Testing |
| 控制层级 | 浏览器内部（扩展 API） | 操作系统层（截屏 + 模拟键鼠） | 多层级：Web 走 Playwright/Puppeteer/CDP，Android 走 ADB，iOS 走 WDA，桌面走 native 控件 |
| 感知方式 | DOM + 截图，结构化指令（导航/点击元素/scroll-to 原语） | 纯视觉：像素坐标 → 键鼠事件 | 纯视觉为主（截图定位，DOM 仅 opt-in 用于数据提取） |
| 模型要求 | 任意模型，**纯文本模型可用** | 必须视觉模型 + API 原生支持 computer 工具 | 必须多模态（Qwen3.x / Doubao-Seed-2.1 / GLM-4.6V / Gemini-Flash / UI-TARS，支持开源自托管） |
| 平台覆盖 | 仅 Chromium 系浏览器 | 整个桌面（任意原生应用） | Web + Android + iOS + HarmonyOS + 桌面应用，还可接自定义接口 |
| Agent 集成 | 任意能调 shell 的 agent（`bsk` CLI + skill） | 需模型 API 支持 computer 工具 | TypeScript SDK + Chrome 插件（Playground）+ AI agent skills（社区另有 Python/Java 移植） |
| 登录态 | **核心卖点**：复用真实浏览器 profile | Docker 白板；真机模式复用但接管键鼠 | 默认白板；Chrome 插件模式可复用当前浏览器 |
| 与用户并行 | ✅ 独立 Agent Window + 标签借用需显式确认 | ❌ 控制哪块屏幕就独占哪块屏幕 | 测试场景无此概念（CI 无人值守跑） |
| 人工介入 | 内置 human-in-loop（captcha/登录弹窗求助） | 无内置机制 | Playground 供人工调试指令 |
| 可观测性 | 任务型，无报告概念 | 截图序列回放 | HTML 报告：截图/元素定位/AI 决策过程/断言结果，供人和 AI 共同排查 |
| 典型成本 | 低（结构化命令，一步到位） | 高（每步全屏截图 → 推理循环） | 低（官方数据：AppControlBench 60 任务总成本 $0.59，Doubao Seed 2.1 Turbo） |

---

## 三、Midscene vs BrowserSkill：五个关键差异

### 1. 目标场景：测试思维 vs 助理思维

Midscene 的 API 处处是测试工程的形状：

```typescript
const agent = new PlaywrightAgent(page);
await agent.aiAct('Search for headphones, then filter the results to under $100');
await agent.aiWaitFor('The filtered search results are displayed');
await agent.aiAssert('Every product in the search results has a price below $100');
```

- `aiAct`（自主流程）/ `aiWaitFor`（等待状态）/ `aiAssert`（视觉断言）/ `aiQuery`（结构化数据提取）
- **Midscene Test**（`@midscene/test`）：YAML 声明测试意图 + TypeScript Node 编排 API 调用/数据准备/清理，还能源码生成能力清单供人和 AI 共同维护用例
- 每次运行产出交互式 HTML 报告

BrowserSkill 则是任务思维：`tab borrow`（借用你的标签页，需确认）→ 干活 → 归还，`request-help` 处理 captcha/登录。

### 2. 感知路线：纯视觉 vs 扩展 API

Midscene 明确选择**视觉优先**（README 原文："screenshot-based UI actions avoid sending large DOM trees to the model"）：

- 优点：icon-only 按钮、`<canvas>`、跨域 iframe、原生 App 控件统统能定位，不写 selector、不加语义标注
- 代价：必须配多模态模型；每步有截图推理开销（官方用便宜视觉模型压成本）

BrowserSkill 走**扩展 API** 路线：

- 优点：结构化指令精确、便宜，纯文本模型可用
- 代价：只覆盖 Chromium；图形验证码等场景对纯文本模型不可解（README 明确承认，仅视觉模型可尝试）

### 3. 平台广度：一套 API 打通五端

Midscene 的同一组 Agent API 跨 Web/Android/iOS/HarmonyOS/桌面，并提供"截图 + 动作能力"自定义接口接入任意设备（社区已有机械臂 + 视觉 + 语音的车机测试案例）。BrowserSkill 只守浏览器一层，Computer Use 只守桌面一层。

### 4. 登录态：谁复用、怎么复用

| 方案 | 默认状态 | 复用路径 |
|---|---|---|
| Midscene | Playwright/Puppeteer 新建 context = 白板 | ① Chrome 插件 Playground 直接在**你当前浏览器**里跑；② Playwright persistent context / 连接已有浏览器实例 |
| BrowserSkill | **默认即复用**你的真实 profile | 借用你已打开的标签页需显式确认 + 归还 |
| Computer Use | Docker + 虚拟显示器 + 全新 Firefox = 白板 | macOS 真机模式（AppleScript 控制真实屏幕）可复用，但任务期间键鼠被接管 |

注意：Midscene 的"复用"和 BrowserSkill 的"复用"**性质不同**——前者是测试便利（省掉造登录 fixture），后者是产品核心（真实账号的日常操作）。

### 5. 生态绑定

- Midscene：TypeScript SDK 为主，模型自行配置（支持开源自托管），测试框架集成 Playwright/Puppeteer
- BrowserSkill：不绑定语言/框架/模型，只要 agent 能跑 shell 就能用
- Computer Use：绑定支持 computer 工具的模型 API

---

## 四、Computer Use 简评（为何排最后）

- **能力最广、代价最大**：整台机器的控制权意味着每步全屏截图的高成本循环、动态 UI 上的坐标失误，以及极高的 prompt injection 风险（被控屏幕上的恶意内容可直接指挥 agent）。
- **官方推荐的隔离部署天然无登录态**：Docker 参考实现里的浏览器是白板；要用真机登录态就得接受"任务期间人机无法并行"。
- 业界趋势正在向"扩展/浏览器内"路线收敛（Anthropic 自家的 Claude for Chrome 即是类似思路），Computer Use 更适合作为兜底：只有必须操作浏览器之外的原生应用时才启用。

---

## 五、选型决策清单

1. 要不要操作**浏览器之外**的东西（原生 App / 终端 / 文件对话框）？
   → 是 → **Computer Use**（务必沙箱/容器部署）
2. 是写 **E2E 测试**，或要做**移动端（Android/iOS/鸿蒙）**自动化？
   → 是 → **Midscene**（视觉断言 + HTML 报告 + 跨端 API）
3. 是让日常 coding agent 用**你已登录的网站**干活，且要和你**并行使用**浏览器？
   → 是 → **BrowserSkill**（Agent Window + 标签借用 + human-in-loop）
4. 都要？可以**组合**：CI 里 Midscene 跑回归测试，本地 Claude Code + BrowserSkill 干日常活，Computer Use 留给原生应用的边角场景。

---

## 六、与 Paseo 的关系

Paseo 自带 `browser_*` 工具（App 内置的 resident webview 标签页），相当于**第四条路线：零配置的隔离浏览器**——没有你的登录态，但 agent 可以自己开标签页查文档、验证 UI、截图，完全不用碰你的主浏览器。选型时可按此排序：

| 需求 | 选择 |
|---|---|
| Agent 自己查资料 / 自验证（无登录要求） | **Paseo 内置 browser_\***（零配置，隔离安全） |
| 需要你的登录态、与你并行使用浏览器 | **BrowserSkill** |
| 测试工程化 / 移动端自动化 | **Midscene** |
| 浏览器之外的原生应用 | **Computer Use**（沙箱） |

---

## 参考

- [Tencent/BrowserSkill README.zh-CN.md](https://github.com/Tencent/BrowserSkill/blob/main/README.zh-CN.md)（2026-09-17 抓取）
- [web-infra-dev/midscene README](https://github.com/web-infra-dev/midscene)（2026-09-17 抓取，含基准数据：AndroidWorld 93.1% / MobileWorld 78.6% / AppControlBench 96.7%，成本 $0.59/60 任务）
- Anthropic Computer Use 官方文档与 quickstart
