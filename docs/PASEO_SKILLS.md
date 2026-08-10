# Paseo Skills 安装与最佳实践

> 本文记录本仓库当前的 Paseo orchestration skills 安装状态、验证结果和安全使用规范。

## 当前状态

2026-08-10 已执行官方安装命令：

```bash
npx skills add getpaseo/paseo
```

由于命令在 `/opt/paseo` 工作区执行，安装器将 skill 放在项目级 `.agents/skills/`，并生成 `skills-lock.json`。当前安装了 7 个 skill：

| Skill | 用途 |
|---|---|
| `paseo` | Paseo CLI、workspace、agent、schedule、heartbeat 参考手册 |
| `paseo-handoff` | 将当前任务交接给另一个 agent |
| `paseo-loop` | 带 worker/verifier 的有界迭代循环 |
| `paseo-committee` | 两个不同 provider 的分析委员会 |
| `paseo-advisor` | 单 agent 的只读第二意见 |
| `release-beta` | Paseo beta 发布流程 |
| `release-stable` | Paseo stable 发布流程 |

如果要让其他项目也能使用这些 skill，应在对应项目中执行安装命令，或按安装器支持的 agent 范围配置共享目录；不要手动复制并绕过 lock 文件。

## 验证记录

已完成以下只读验证：

- 7 个目录均存在 `SKILL.md`，frontmatter 中的 skill 名称与目录一致。
- `skills-lock.json` 的 7 个条目均来自 `getpaseo/paseo`，并包含 hash。
- Paseo CLI 在 PATH 中，版本为 `0.2.4`。
- `paseo workspace --help` 暴露 workspace create/list/archive。
- `paseo loop --help` 暴露 loop run/list/inspect/logs/stop。

尚未执行会创建 agent、workspace 或 loop 的测试，因为本机运行时当前不可用：`paseo status` 报告 Connected Daemon unreachable，并检测到 stale PID。恢复 daemon 前应先人工确认旧进程是否真的退出；不要直接重启，以免终止正在运行的 agent。

## 推荐使用顺序

1. 先读取 `/paseo` 参考 skill，确认 CLI/tool surface、workspace 隔离和 daemon 语义。
2. 需要选择 provider 或创建 agent 前，实际读取 `~/.paseo/orchestration-preferences.json`。不要在 skill 或 prompt 中硬编码 provider；文件缺失时使用合理默认值，并明确告知用户。
3. 明确任务边界、相关文件、验收标准和权限范围，再选择 handoff、advisor、committee 或 loop。
4. 只在 daemon 健康、provider 可用且测试命令明确时执行编排操作。
5. 对每次编排保留 agent/workspace/loop ID，完成后检查日志和验证结果，并归档不再需要的资源。

## 各 skill 的最佳实践

### `/paseo-handoff`

- 适用于任务转交，不等于只转发一句短 prompt。
- briefing 至少包含：任务、背景、相关文件、当前状态、已尝试方案、决策、验收标准和约束。
- 明确要求 worktree 隔离时才创建 worktree；涉及代码修改的并行任务优先使用隔离 workspace。
- 默认不要等待 agent 完成，记录 agent ID 和 workspace ID 供后续跟踪。

### `/paseo-loop`

- 只用于确实需要 worker/verifier 反复生命周期的任务；简单定时检查优先用 heartbeat。
- 设置 `--max-iterations` 或 `--max-time`，不要启动无界循环。
- 可客观判断的结果使用 `--verify-check`；需要判断代码完整性的结果使用 `--verify`；复杂任务可以同时使用二者。
- worker 和 verifier 尽量使用不同 provider；仅在轮询外部状态时设置 `--sleep`。
- 需要审计每轮结果时使用 `--archive`，结束后停止并清理 loop。

### `/paseo-committee`

- 适合根因分析、困难规划和卡住后的换角度审查，不适合简单实现。
- 两名成员应使用不同 provider；prompt 要求深入追问根因，并明确“只分析、不编辑”。
- 先让委员会收敛计划，再由主 agent 实现；实现后把 diff 发回委员会复核。
- 委员会成员不得创建、修改或删除文件。

### `/paseo-advisor`

- 适合第二意见、风险审查和方案判断，不适合委托实际开发。
- 提供自包含上下文和相关文件路径，并明确要求给出带理由的建议。
- advisor prompt 必须保持只读；最终取舍仍由主 agent 或用户决定。

## 安全与运维规则

- 安装器提示所有 skill 会以完整 agent 权限运行；安装后应审阅 `SKILL.md`，尤其是 release 和 loop 类 skill。
- 本次安装器评估：`paseo` 的 Gen 结果为 Safe、Socket 为 0 alerts、Snyk 为 Medium Risk；`paseo-loop` 的 Gen/Snyk 为 Medium Risk；`paseo-committee` 有 1 个 Socket alert。这里的评估结果不是授权替代品，首次使用前应人工审阅源码和实际权限。
- 不要把 token、密码或完整环境变量放进 briefing、日志、提交或工作区文件。
- 修改代码前优先使用 worktree 隔离；委员会和 advisor 保持只读。
- daemon 故障排查顺序：查看 `~/.paseo/daemon.log`，执行 `paseo daemon status`，必要时检查 `curl -s http://127.0.0.1:6767/api/health`。重启 daemon 会影响所有 agent，必须先获得明确批准。
- 任何 loop 都要有停止条件；完成后用 `paseo loop stop <id>`，不再需要的 agent/workspace 应归档。

## 更新与回滚检查

更新前先查看工作区改动和 `skills-lock.json`：

```bash
git status --short
npx skills add getpaseo/paseo
git diff -- skills-lock.json
```

更新后重新检查所有 `SKILL.md`、CLI help 和安全评估。若新版本行为不符合预期，保留 lock 文件和 diff，暂停使用受影响 skill，并在明确版本策略后再回滚或重新安装；不要直接删除未知目录。
