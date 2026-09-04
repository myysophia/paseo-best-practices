# Paseo 跨 Provider 实现

> 基于 [getpaseo/paseo](https://github.com/getpaseo/paseo) 源码分析。结论：**跨 provider 不迁移会话，上下文以纯文本简报交接**。

## 核心设计

各 provider 会话格式互不兼容（Claude 的 JSONL ≠ Codex 的 rollout），Paseo 选择**不转换格式**，用文本作为通用货币：

| 机制 | 载体 | 适用场景 |
|---|---|---|
| Fork | `chat_history` 文本附件，自动生成 | UI 上手动分叉会话到新工作区 |
| `/paseo-handoff` | handoff 简报 prompt | agent 间显式交接任务 |
| Paseo subagent | 新会话 + 通知回传 | 编排（`Claude Code => Codex`） |

三者同一哲学：**显式文本交接优于隐式迁移**。

## Fork 数据流（源码）

```
前端 use-fork-agent.ts          服务端 session.ts:7337        activity-curator.ts
点击 fork ──────────────────► agent.fork_context.request ──► fetchTimeline(limit:0 全量)
boundary 决定截断点                                              │ 仅保留 user/assistant/tool_call
                                                               ▼
新工作区草稿 ◄──────────────── chat_history 文本附件 <──── <chat-history-summary> 包装
(provider/model 预填可改)        prompt-attachments.ts: 首条 prompt 的第一个内容块
```

### 关键实现点

| 文件 | 作用 |
|---|---|
| `packages/app/src/hooks/use-fork-agent.ts` | boundary 语义：传 messageId 截到该轮；不传则含流式中回复的全量投影（支持中途 fork）。源 provider/model 仅作草稿默认值，提交前可换任意 provider |
| `packages/server/src/server/agent/activity-curator.ts` | `selectForkContextRows` 按 epoch+seq 校验并截断（epoch 不匹配报 "position no longer available"）；策展参数 `maxItems:0, includeKinds: ["user_message","assistant_message","tool_call"], includeExternalToolInput:false` —— reasoning/todo/error 丢弃，工具入参不带走 |
| `packages/app/src/screens/new-workspace-fork-context.ts` | `remapDraftCwdToWorkspace` 把历史里的路径重映射到新工作区目录；chat_history 排除在工作区自动命名外 |
| `packages/server/src/server/agent/prompt-attachments.ts` | 新 agent 首条 prompt：`[...chatHistory, 用户消息, ...其他附件]`，历史永远排最前 |

## 能力边界

- `update_agent` 只能改 model/thinking/mode，**不能换 provider** —— 换 provider 必然是新会话
- 已运行会话保持启动时的认证，凭证过期需开新会话
- fork 交接的是"剧本摘要"而非"录像带"：思考过程、工具完整输入丢失
- heartbeat 回到同一会话，不跨 provider

## 参考

- 官方文档：[Skills](https://paseo.sh/docs/skills.md) · [Orchestration](https://paseo.sh/docs/orchestration.md) · [Providers](https://paseo.sh/docs/providers.md)
- 源码版本：2026-09 main 分支
