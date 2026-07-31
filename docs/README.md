# Paseo 文档目录

> Paseo 部署的完整文档集。覆盖两种部署形态：**Linux 服务器**（`/opt/paseo`）与 **macOS 桌面 App**（`/Applications/Paseo.app`）。

---

## 文档清单

| 文档 | 定位 | 给谁看 |
|---|---|---|
| [**PASEO_LEARNING.md**](./PASEO_LEARNING.md) | **学习指南** —— 外部资源清单、学习路径、5 条原则、动手练习 | 想系统学习的人 |
| [**PASEO_USAGE.md**](./PASEO_USAGE.md) | **使用手册** —— web UI / 手机配对 / 常用 CLI 命令 / 排查 | 使用者（手机/web UI 端） |
| [**PASEO_OPS.md**](./PASEO_OPS.md) | **Linux 运维手册 + 踩坑记录** —— systemd 托管、端口/SG/密码、8 个踩坑、自检脚本 | 运维（改服务器配置前必读） |
| [**PASEO_MACOS_DESKTOP.md**](./PASEO_MACOS_DESKTOP.md) | **macOS 桌面版运维手册** —— App/daemon 关系、进程结构、配置调优、端口冲突清理、开机自启 | Mac 用户、桌面版排查 |
| [**PASEO_ARCHITECTURE.md**](./PASEO_ARCHITECTURE.md) | **原理与架构** —— 三层进程模型、连接拓扑、环境继承 | 想搞懂"为什么"的人 |
| [**PASEO_ADR.md**](./PASEO_ADR.md) | **决策记录(ADR)** —— 5 条不可逆决策的背景/选项/代价 | 改架构决策前先读对应 ADR |

---

## 按场景找文档

- **「我想系统学习 Paseo，看哪些资料？」** → [LEARNING](./PASEO_LEARNING.md)
- **「我想用手机/web 控制 agent」** → [USAGE](./PASEO_USAGE.md)
- **「我用的是 Mac 桌面 App，不是服务器」** → [MACOS_DESKTOP](./PASEO_MACOS_DESKTOP.md)
- **「macOS 上 daemon 报 EADDRINUSE / 端口被占」** → [MACOS_DESKTOP 坑 #1](./PASEO_MACOS_DESKTOP.md)
- **「手机连不上了」** → [USAGE §六](./PASEO_USAGE.md) 快速排查，深度排查 [OPS 坑 #7](./PASEO_OPS.md)
- **「我要改端口 / 开公网 / 改密码」** → [OPS §二](./PASEO_OPS.md) + [坑 #4](./PASEO_OPS.md)
- **「手机报 root/sudo 拦截」** → [坑 #1 / #2 / #3](./PASEO_OPS.md)
- **「paseo 到底怎么把手机请求送到 claude 的」** → [ARCHITECTURE](./PASEO_ARCHITECTURE.md)
- **「当初为什么用 systemd / 为什么开公网」** → [ADR](./PASEO_ADR.md)

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
| 监听端口 | `<PORT>`（如 `8767`，0.0.0.0） |
| 云防火墙 | 入站放行该端口 TCP，仅限需要的来源 CIDR |
| systemd | `paseo.service` (enabled, 开机自启) |
| 密码 | 已设 (`config.json: auth.password`) |
| 中继 | 启用 (`relay.paseo.sh:443`)，与直连并存 |
| 版本 | paseo 0.2.4 |
