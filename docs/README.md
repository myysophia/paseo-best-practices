# Paseo 文档目录

> Paseo 部署的完整文档集。覆盖两种部署形态：**Linux 服务器**（`/opt/paseo`）与 **macOS 桌面 App**（`/Applications/Paseo.app`）。
>
> 文档按读者意图分五组：`getting-started`（入门使用）/ `ops`(运维) / `guides`（专项最佳实践）/ `internals`（原理深入）/ `adr`（决策记录）。

---

## 文档清单

### getting-started —— 入门与使用

| 文档 | 定位 | 给谁看 |
|---|---|---|
| [**learning.md**](./getting-started/learning.md) | **学习指南** —— 外部资源清单、学习路径、5 条原则、动手练习 | 想系统学习的人 |
| [**usage.md**](./getting-started/usage.md) | **使用手册** —— web UI / 手机配对 / 常用 CLI 命令 / 排查 | 使用者（手机/web UI 端） |

### guides —— 专项最佳实践

| 文档 | 定位 | 给谁看 |
|---|---|---|
| [**session-migration.md**](./guides/session-migration.md) | **会话迁移最佳实践** —— 筛选并安全导入 Codex / Claude 历史会话 | 需要在手机端续聊的人 |
| [**direct-connect.md**](./guides/direct-connect.md) | **直连最佳实践** —— 抓包验证明文风险、三层防御、加密方案对比、决策清单 | 加 host 选直连前必读 |
| [**tailscale.md**](./guides/tailscale.md) | **Tailscale 科普指南** —— 两层架构、NAT 穿透与 DERP/直连决策机制(§四,含 netcheck/status 实战)、subnet router/exit node、最佳实践、收费、paseo 实战案例 | 想搞懂 Tailscale 的人,尤其"为什么我走 DERP 不走直连"的新手 |
| [**skills.md**](./guides/skills.md) | **Orchestration skills 安装与最佳实践** —— 安装状态、验证、handoff/loop/committee/advisor 使用规范 | 使用 Paseo 编排能力的人 |
| [**pi-custom-model.md**](./guides/pi-custom-model.md) | **Pi 自定义模型配置** —— 中转站接入多厂商模型、models.json 字段详解、SOP、5 个坑 | 用 Pi provider + 中转/自定义模型的人 |

### ops —— 运维手册

| 文档 | 定位 | 给谁看 |
|---|---|---|
| [**linux-ops.md**](./ops/linux-ops.md) | **Linux 运维手册 + 踩坑记录** —— systemd 托管、端口/SG/密码、8 个踩坑、自检脚本 | 运维（改服务器配置前必读） |
| [**macos-desktop.md**](./ops/macos-desktop.md) | **macOS 桌面版运维手册** —— App/daemon 关系、进程结构、配置调优、端口冲突清理、开机自启 | Mac 用户、桌面版排查 |

### internals —— 原理深入

| 文档 | 定位 | 给谁看 |
|---|---|---|
| [**architecture.md**](./internals/architecture.md) | **原理与架构** —— 三层进程模型、连接拓扑、环境继承 | 想搞懂"为什么"的人 |
| [**cross-provider.md**](./internals/cross-provider.md) | **跨 Provider 实现** —— fork/handoff 的文本交接机制、源码数据流、能力边界 | 想搞懂跨 provider 上下文传递的人 |

### adr —— 决策记录

| 文档 | 定位 | 给谁看 |
|---|---|---|
| [**adr.md**](./adr/adr.md) | **决策记录(ADR)** —— 不可逆决策的背景/选项/代价 | 改架构决策前先读对应 ADR |

---

## 按场景找文档

**入门使用**

- **「我想系统学习 Paseo，看哪些资料？」** → [LEARNING](./getting-started/learning.md)
- **「我想用手机/web 控制 agent」** → [USAGE](./getting-started/usage.md)
- **「我想把 Codex / Claude 历史会话迁到 Paseo」** → [SESSION_MIGRATION](./guides/session-migration.md)

**专项实践**

- **「Tailscale 是什么 / 怎么工作 / 怎么用」** → [TAILSCALE](./guides/tailscale.md)
- **「为什么我 Tailscale 走 DERP 不走直连 / 怎么判断当前路径」** → [TAILSCALE §四](./guides/tailscale.md)
- **「添加主机选 Direct connect 怎么配 / 明文风险」** → [DIRECT_CONNECT](./guides/direct-connect.md)
- **「怎么安装/安全使用 Paseo skills」** → [SKILLS](./guides/skills.md)
- **「Pi 想接中转站模型 / 模型全连不上」** → [PI_CUSTOM_MODEL](./guides/pi-custom-model.md)（域名失效先看坑 #1）
- **「跨 provider 怎么交接上下文 / fork 原理」** → [CROSS_PROVIDER](./internals/cross-provider.md)

**故障排查**

- **「我用的是 Mac 桌面 App，不是服务器」** → [MACOS_DESKTOP](./ops/macos-desktop.md)
- **「macOS 上 daemon 报 EADDRINUSE / 端口被占」** → [MACOS_DESKTOP 坑 #1](./ops/macos-desktop.md)
- **「运行中切 auto 模式报 `auto mode unavailable for this model`」** → [MACOS_DESKTOP 坑 #4](./ops/macos-desktop.md) + [ADR-0007](./adr/adr.md)
- **「手机连不上了」** → [USAGE §六](./getting-started/usage.md) 快速排查，深度排查 [OPS 坑 #7 / #8](./ops/linux-ops.md)
- **「我要改端口 / 开公网 / 改密码」** → [OPS §二](./ops/linux-ops.md) + [坑 #4](./ops/linux-ops.md)
- **「手机报 root/sudo 拦截」** → [坑 #1 / #2 / #3](./ops/linux-ops.md)

**原理与决策**

- **「paseo 到底怎么把手机请求送到 claude 的」** → [ARCHITECTURE](./internals/architecture.md)
- **「当初为什么用 systemd / 为什么开公网」** → [ADR](./adr/adr.md)

---

## 推荐阅读顺序

1. [ARCHITECTURE](./internals/architecture.md) —— 先建心智模型
2. [USAGE](./getting-started/usage.md) —— 上手用
3. [OPS](./ops/linux-ops.md) —— 接手运维（重点看踩坑）
4. [ADR](./adr/adr.md) —— 需要改决策时翻

---

## 当前部署快照

| 项 | 值 |
|---|---|
| 服务器 | 你的服务器（公网 IP `<SERVER_IP>`） |
| **接入方式** | **双入口并存**（见下方） |
| 监听 | `0.0.0.0:8767`（同时覆盖公网 + tailnet + lo） |
| 入口 1：tailnet | `<TS_IP>:8767`（WireGuard 加密，手机/跨网客户端用，**推荐**） |
| 入口 2：公网 | `<SERVER_IP>:8767`（明文，仅办公网段 SG 放行，办公网内设备用） |
| 云防火墙 | bastion SG 撤销了 `0.0.0.0/0`；office-sg 保留办公网段 `/30` 全端口规则 |
| systemd | `paseo.service` (enabled, 开机自启) |
| 密码 | 已设 (`config.json: auth.password`) |
| 中继 | 启用 (`relay.paseo.sh:443`)，作兜底 |
| Tailscale DNS 接管 | **两端都关**（`--accept-dns=false`，防断网） |
| 版本 | paseo 0.7.2 |

> **两条入口怎么选**：手机/跨网客户端优先走 tailnet（加密）；办公网段内的设备走公网 IP（明文，但已在可信网段）。详见 [ADR-0005](./adr/adr.md) 和 [DIRECT_CONNECT](./guides/direct-connect.md)。
>
> 演进历史:`0.0.0.0:8767` 纯公网明文 → 切 tailnet `100.x.y.z:8767` → 改回 `0.0.0.0:8767` 双入口并存（公网收窄到办公网段 + tailnet 加密）。
