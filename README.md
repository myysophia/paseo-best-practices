# paseo-best-practices

Paseo（`@getpaseo/cli`）的部署经验、运维踩坑与最佳实践整理。

Paseo 让你从手机驱动服务器上的 AI coding agent（claude / codex 等）——服务器跑一个 daemon，手机 app 通过它远程操作 agent，等于在手机上开了一个服务器端终端 + AI 助手。

## 文档

完整索引见 [docs/README.md](./docs/README.md)（含按场景导航）。按目录分组：

| 分组 | 内容 |
|---|---|
| [docs/getting-started/](./docs/getting-started/) | 入门与使用：[学习指南](./docs/getting-started/learning.md)、[使用手册](./docs/getting-started/usage.md) |
| [docs/ops/](./docs/ops/) | 运维手册：[Linux 运维+踩坑](./docs/ops/linux-ops.md)、[macOS 桌面版](./docs/ops/macos-desktop.md) |
| [docs/guides/](./docs/guides/) | 专项实践：[会话迁移](./docs/guides/session-migration.md)、[直连安全](./docs/guides/direct-connect.md)、[Tailscale](./docs/guides/tailscale.md)、[Skills 编排](./docs/guides/skills.md)、[Pi 自定义模型](./docs/guides/pi-custom-model.md) |
| [docs/internals/](./docs/internals/) | 原理深入：[架构](./docs/internals/architecture.md)、[跨 Provider 机制](./docs/internals/cross-provider.md) |
| [docs/adr/](./docs/adr/) | 决策记录（ADR）：不可逆决策的背景 / 选项 / 代价 |

## 推荐阅读顺序

按部署形态选入口：

- **Linux 服务器部署**：[ARCHITECTURE](./docs/internals/architecture.md) → [USAGE](./docs/getting-started/usage.md) → [OPS](./docs/ops/linux-ops.md) → [ADR](./docs/adr/adr.md)
- **macOS 桌面 App**：[MACOS_DESKTOP](./docs/ops/macos-desktop.md) → [USAGE](./docs/getting-started/usage.md) → [ARCHITECTURE](./docs/internals/architecture.md)
- **把历史 Codex / Claude 会话迁到 Paseo**：[SESSION_MIGRATION](./docs/guides/session-migration.md) → [USAGE](./docs/getting-started/usage.md)

## 贡献

这是一份持续积累的最佳实践集，欢迎通过 PR/Issue 补充你的踩坑和做法。
