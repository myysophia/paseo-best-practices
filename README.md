# paseo-best-practices

Paseo（`@getpaseo/cli`）的部署经验、运维踩坑与最佳实践整理。

Paseo 让你从手机驱动服务器上的 AI coding agent（claude / codex 等）——服务器跑一个 daemon，手机 app 通过它远程操作 agent，等于在手机上开了一个服务器端终端 + AI 助手。

## 文档

| 文档 | 内容 |
|---|---|
| [docs/README.md](./docs/README.md) | 文档总索引 + 按场景导航 |
| [docs/PASEO_ARCHITECTURE.md](./docs/PASEO_ARCHITECTURE.md) | 原理与架构：三层进程模型、连接拓扑、配置与凭据流向 |
| [docs/PASEO_USAGE.md](./docs/PASEO_USAGE.md) | 使用手册：web UI / 手机配对 / 常用 CLI 命令 / 排查 |
| [docs/PASEO_OPS.md](./docs/PASEO_OPS.md) | 运维手册 + 踩坑记录：systemd 托管、端口/密码/公网、自检脚本 |
| [docs/PASEO_ADR.md](./docs/PASEO_ADR.md) | 决策记录（ADR）：不可逆决策的背景 / 选项 / 代价 |

## 推荐阅读顺序

1. [ARCHITECTURE](./docs/PASEO_ARCHITECTURE.md) —— 建心智模型
2. [USAGE](./docs/PASEO_USAGE.md) —— 上手用
3. [OPS](./docs/PASEO_OPS.md) —— 接手运维（重点看踩坑）
4. [ADR](./docs/PASEO_ADR.md) —— 需要改决策时翻

## 贡献

这是一份持续积累的最佳实践集，欢迎通过 PR/Issue 补充你的踩坑和做法。
