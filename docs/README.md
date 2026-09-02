# Paseo 文档目录

> Paseo 部署的完整文档集。覆盖两种部署形态：**Linux 服务器**（`/opt/paseo`）与 **macOS 桌面 App**（`/Applications/Paseo.app`）。

---

## 文档清单

| 文档 | 定位 | 给谁看 |
|---|---|---|
| [**PASEO_LEARNING.md**](./PASEO_LEARNING.md) | **学习指南** —— 外部资源清单、学习路径、5 条原则、动手练习 | 想系统学习的人 |
| [**TAILSCALE_GUIDE.md**](./TAILSCALE_GUIDE.md) | **Tailscale 科普指南** —— 两层架构、NAT 穿透与 DERP/直连决策机制(§四,含 netcheck/status 实战)、subnet router/exit node、最佳实践、收费、paseo 实战案例 | 想搞懂 Tailscale 的人,尤其"为什么我走 DERP 不走直连"的新手 |
| [**PASEO_USAGE.md**](./PASEO_USAGE.md) | **使用手册** —— web UI / 手机配对 / 常用 CLI 命令 / 排查 | 使用者（手机/web UI 端） |
| [**PASEO_SESSION_MIGRATION.md**](./PASEO_SESSION_MIGRATION.md) | **会话迁移最佳实践** —— 筛选并安全导入 Codex / Claude 历史会话 | 需要在手机端续聊的人 |
| [**PASEO_DIRECT_CONNECT.md**](./PASEO_DIRECT_CONNECT.md) | **直连最佳实践** —— 抓包验证明文风险、三层防御、加密方案对比、决策清单 | 加 host 选直连前必读 |
| [**PASEO_OPS.md**](./PASEO_OPS.md) | **Linux 运维手册 + 踩坑记录** —— systemd 托管、端口/SG/密码、8 个踩坑、自检脚本 | 运维（改服务器配置前必读） |
| [**PASEO_MACOS_DESKTOP.md**](./PASEO_MACOS_DESKTOP.md) | **macOS 桌面版运维手册** —— App/daemon 关系、进程结构、配置调优、端口冲突清理、开机自启 | Mac 用户、桌面版排查 |
| [**PASEO_ARCHITECTURE.md**](./PASEO_ARCHITECTURE.md) | **原理与架构** —— 三层进程模型、连接拓扑、环境继承 | 想搞懂"为什么"的人 |
| [**PASEO_PI_CUSTOM_MODEL.md**](./PASEO_PI_CUSTOM_MODEL.md) | **Pi 自定义模型配置** —— 中转站接入多厂商模型、models.json 字段详解、SOP、5 个坑 | 用 Pi provider + 中转/自定义模型的人 |
| [**PASEO_ADR.md**](./PASEO_ADR.md) | **决策记录(ADR)** —— 不可逆决策的背景/选项/代价 | 改架构决策前先读对应 ADR |
| [**PASEO_SKILLS.md**](./PASEO_SKILLS.md) | **Orchestration skills 安装与最佳实践** —— 安装状态、验证、handoff/loop/committee/advisor 使用规范 | 使用 Paseo 编排能力的人 |

---

## 按场景找文档

- **「我想系统学习 Paseo，看哪些资料？」** → [LEARNING](./PASEO_LEARNING.md)
- **「我想用手机/web 控制 agent」** → [USAGE](./PASEO_USAGE.md)
- **「我想把 Codex / Claude 历史会话迁到 Paseo」** → [SESSION_MIGRATION](./PASEO_SESSION_MIGRATION.md)
- **「Tailscale 是什么 / 怎么工作 / 怎么用」** → [TAILSCALE_GUIDE](./TAILSCALE_GUIDE.md)
- **「为什么我 Tailscale 走 DERP 不走直连 / 怎么判断当前路径」** → [TAILSCALE_GUIDE §四](./TAILSCALE_GUIDE.md)
- **「添加主机选 Direct connect 怎么配 / 明文风险」** → [DIRECT_CONNECT](./PASEO_DIRECT_CONNECT.md)
- **「我用的是 Mac 桌面 App，不是服务器」** → [MACOS_DESKTOP](./PASEO_MACOS_DESKTOP.md)
- **「macOS 上 daemon 报 EADDRINUSE / 端口被占」** → [MACOS_DESKTOP 坑 #1](./PASEO_MACOS_DESKTOP.md)
- **「运行中切 auto 模式报 `auto mode unavailable for this model`」** → [MACOS_DESKTOP 坑 #4](./PASEO_MACOS_DESKTOP.md) + [ADR-0007](./PASEO_ADR.md)
- **「手机连不上了」** → [USAGE §六](./PASEO_USAGE.md) 快速排查，深度排查 [OPS 坑 #7](./PASEO_OPS.md)
- **「我要改端口 / 开公网 / 改密码」** → [OPS §二](./PASEO_OPS.md) + [坑 #4](./PASEO_OPS.md)
- **「手机报 root/sudo 拦截」** → [坑 #1 / #2 / #3](./PASEO_OPS.md)
- **「paseo 到底怎么把手机请求送到 claude 的」** → [ARCHITECTURE](./PASEO_ARCHITECTURE.md)
- **「当初为什么用 systemd / 为什么开公网」** → [ADR](./PASEO_ADR.md)
- **「Pi 想接中转站模型 / 模型全连不上」** → [PI_CUSTOM_MODEL](./PASEO_PI_CUSTOM_MODEL.md)（域名失效先看坑 #1）
- **「怎么安装/安全使用 Paseo skills」** → [SKILLS](./PASEO_SKILLS.md)

---

## 推荐阅读顺序

1. [ARCHITECTURE](./PASEO_ARCHITECTURE.md) —— 先建心智模型
2. [USAGE](./PASEO_USAGE.md) —— 上手用
3. [OPS](./PASEO_OPS.md) —— 接手运维（重点看踩坑）
4. [ADR](./PASEO_ADR.md) —— 需要改决策时翻

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
| 版本 | paseo 0.2.4 |

> **两条入口怎么选**：手机/跨网客户端优先走 tailnet（加密）；办公网段内的设备走公网 IP（明文，但已在可信网段）。详见 [ADR-0005](./PASEO_ADR.md) 和 [DIRECT_CONNECT](./PASEO_DIRECT_CONNECT.md)。
>
> 演进历史:`0.0.0.0:8767` 纯公网明文 → 切 tailnet `100.x.y.z:8767` → 改回 `0.0.0.0:8767` 双入口并存（公网收窄到办公网段 + tailnet 加密）。
