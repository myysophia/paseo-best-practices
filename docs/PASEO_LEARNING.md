# Paseo 学习指南

> 面向**想系统学习 Paseo 的人**。把分散在官方文档、HN、Reddit、社区的内容整理成一条清晰的路径，并提炼可复用的原则。
> 部署运维看 [`PASEO_OPS.md`](./PASEO_OPS.md)（Linux）或 [`PASEO_MACOS_DESKTOP.md`](./PASEO_MACOS_DESKTOP.md)（Mac 桌面），使用手册看 [`PASEO_USAGE.md`](./PASEO_USAGE.md)。

---

## 一、学习路径

按这个顺序学，每一步对应"建心智模型 → 掌握工具 → 学真实工作流 → 进阶"。

| 阶段 | 目标 | 预计时间 | 主要资源 |
|---|---|---|---|
| 1️⃣ 建心智模型 | 理解 daemon/client 架构 + 设计哲学 | 1 小时 | 官方 docs + 作者 HN 自述 |
| 2️⃣ 掌握 CLI 原语 | 会用 `run/loop/schedule/chat` 做编排 | 2 小时 | explainx 命令速查 + 实操 |
| 3️⃣ 学真实工作流 | 看别人怎么把 paseo 嵌入日常 | 1 小时 | HN 讨论 + Reddit |
| 4️⃣ 进阶玩法 | 多 agent 编排 / 移动端配合 / 对比选型 | 持续 | 作者推特 + 对比页 |

---

## 二、核心资源清单

### A. 官方文档（必读，质量最高）

| 链接 | 价值 |
|---|---|
| [docs.paseo.sh](https://paseo.sh/docs) | 起点，含三种安装方式 |
| [Configuration](https://paseo.sh/docs/configuration) | 配置字段全集（你已经在用） |
| [Providers](https://paseo.sh/docs/providers) | **核心概念**：Paseo 不自带 agent，它启动并监督你已装的 CLI（Claude Code / Codex / OpenCode），保留你的订阅、MCP、skills、config |
| [Supported Agents](https://paseo.sh/agents) | 全量 agent 列表 |
| [Alternatives: Conductor](https://paseo.sh/docs/alternatives/conductor) | 通过对比理解 Paseo 定位 |
| [Security 文档](https://github.com/getpaseo/paseo/blob/main/public-docs/security.md) | daemon 部署形态的安全考量 |

### B. 作者亲述（理解"为什么"）

| 链接 | 关键内容 |
|---|---|
| [Show HN: 作者帖](https://news.ycombinator.com/item?id=47575827) ⭐ | 演化史（散步语音 → 文字 → diff review → 多 agent → 跨机器）+ 四条设计原则 + 名字由来（Paseo = 西班牙语"散步"） |
| [Show HN: Beautiful 讨论](https://news.ycombinator.com/item?id=48377250) ⭐ | 真用户工作流 + 维护者对订阅额度的说明（程序化调用只用一小部分）+ 预告 5h/7d 用量统计 |
| [Reddit: I built an open-source app for Claude Code](https://www.reddit.com/r/ClaudeAI/comments/1s4x9mb/) | 作者亲自介绍 |
| [Discord](https://discord.gg/jz8T2uahpH) | 实时社区 |

### C. CLI 命令速查（实用）

| 链接 | 用途 |
|---|---|
| [explainx.ai/paseo](https://explainx.ai/skills/getpaseo/paseo/paseo) ⭐ | 完整命令清单，四个模块（run / loop / schedule / chat）的参数全在这里 |

### D. 进阶 & 周边

| 链接 | 用途 |
|---|---|
| [GitHub: getpaseo/paseo](https://github.com/getpaseo/paseo) | 源码 + issues |
| [App Store: Paseo](https://apps.apple.com/us/app/paseo-remote-coding-agents/id6758887924) | iOS app |
| [作者推特 @moboudra](https://x.com/moboudra) | 跟进新特性 |
| [YouTube: Paseo overview](https://www.youtube.com/watch?v=-roCTMUQoV4) | 视频概览 |
| [Invisible Daemon 架构思考](https://cocoindex.io/blogs/building-an-invisible-daemon/) | 独立博客，讲本地 daemon 设计模式 |

---

## 三、从所有资料中提炼出的 5 条原则

### 原则 1：Paseo 的本质是"agent 进程的监护者"

不是 inference 替代品，而是**让你的 Claude Code / Codex 订阅、MCP、skills 在远程/移动场景下保持工作**。

**含义**：你不用迁移任何东西。装上 Paseo，它直接用你已有的 `~/.claude/settings.json`、MCP server、订阅认证。本地怎么用，Paseo 里就怎么用。

### 原则 2：CLI 是被低估的原语

作者原话："**CLI 正在演化成更高级编排、循环、agent 团队的基础原语**。"

真正的价值不在 GUI，而在四个命令模块：

```bash
# run —— 一次性 agent
paseo run "重构 utils.ts" --worktree refactor-utils
paseo run "后台跑测试" --detach

# loop —— 自动 worker-verifier 循环（跑到验证通过为止）
paseo loop run "写一个 X" --verify "确保所有测试通过"

# schedule —— 定时主动 agent
paseo schedule create "整理昨天日志" --cron "0 9 * * *"

# chat —— 多 agent 异步协调（持久化聊天室）
paseo chat post <room> "@agentA 帮我看下这个 PR"
```

### 原则 3：订阅额度是硬约束

Claude Code 订阅在程序化使用下**只能用到一小部分额度**。作者在 HN 亲口确认（[来源](https://news.ycombinator.com/item?id=48377250)）：

> Claude Code (via the subscription) will continue working under Paseo but it will consume a different pool of credits... you will be able to use only a fraction of your usage in Paseo.

**对策**：
- 重度使用走 API 计费（配置 `ANTHROPIC_API_KEY`）
- 轻度使用控制节奏，长任务用 `--detach` 放后台
- 等官方的 5h/7d 用量统计功能上线（作者预告中）

### 原则 4：移动端的真正价值是"离桌不离心"

不是让你在手机上写代码。作者澄清过（[来源](https://news.ycombinator.com/item?id=48377250)）：

> It is more about being able to step away from the desk without losing access to the work.

适合的场景：
- 长 agent 运行时**走开**做别的（20 分钟任务，散步回来审阅）
- 通勤/带娃时 brainstorm、triage PR、触发任务
- 跨地点工作：家 → 咖啡馆 → 公园，agent 一直在跑

不适合：在手机上做精细编码、长会话 review。

### 原则 5：Prompt 编写通用技巧（来自 explainx）

**✅ 该做**：
- 清晰具体的 prompt（不要"帮我弄一下"，要"在 `src/auth.ts` 里加一个 `validateToken` 函数，返回 boolean"）
- 提供相关上下文和约束（语言、文件路径、依赖库版本）
- 审阅并迭代输出
- 记录成功的 prompt 模式

**❌ 别做**：
- 盲目接受输出（一定 review）
- 把敏感信息（密钥、token）塞进 prompt
- 指望 agent 替代人判断（架构决策、安全审查）
- 超出 skill 能力范围的任务硬塞

---

## 四、动手练习（10 分钟试一遍）

最能体会 Paseo 价值的三条命令：

```bash
# 1. 一次性 agent（--worktree 隔离，不污染主分支）
paseo run "给当前项目写一个 README" --worktree test-readme

# 2. 自动循环（worker + verifier，循环到满足）
paseo loop run "重构 utils.ts" \
  --verify "运行 npm test，所有用例必须通过" \
  --max-iterations 5

# 3. 定时任务（cron）
paseo schedule create "每天早上 9 点整理昨天日志" --cron "0 9 * * *"
```

跑完看输出：
- `paseo ls` 看所有 agent 状态
- `paseo logs <id>` 看某个 agent 的活动时间线
- `paseo attach <id>` 实时跟随输出流

---

## 五、跟踪 Paseo 演进

Paseo 还在快速迭代，几个值得关注的方向：

| 方向 | 当前状态 | 关注点 |
|---|---|---|
| CLI 原语 | 已有 run/loop/schedule/chat | 作者明确说是"更大的方向" |
| 用量统计 5h/7d | 开发中 | 最被请求的功能，影响 API 成本控制 |
| ACP 支持 | 已支持 Gemini CLI，等 Antigravity | ACP 是标准协议，扩展性来源 |
| MDX / 图表渲染 | 未支持，作者想做 Mermaid | 远程审阅体验提升 |
| 团队/企业层 | 未商业化 | 核心开源、便利层收费 |

**订阅作者推特 [@moboudra](https://x.com/moboudra)** 是最直接的新特性跟进渠道。

---

## 六、与其他文档的关系

| 如果你在找…… | 去看 |
|---|---|
| 怎么部署到 Linux 服务器 | [`PASEO_OPS.md`](./PASEO_OPS.md) |
| macOS 桌面版怎么用 | [`PASEO_MACOS_DESKTOP.md`](./PASEO_MACOS_DESKTOP.md) |
| 手机/web 怎么连、怎么用 | [`PASEO_USAGE.md`](./PASEO_USAGE.md) |
| daemon/client 三层架构原理 | [`PASEO_ARCHITECTURE.md`](./PASEO_ARCHITECTURE.md) |
| 不可逆决策的背景 | [`PASEO_ADR.md`](./PASEO_ADR.md) |
| **系统学习路径 + 外部资源** | **本文** |
