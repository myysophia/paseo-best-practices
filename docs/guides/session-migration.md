# Paseo 会话迁移最佳实践

> 本文把本机 Codex CLI/desktop 与 Claude Code 历史会话导入 Paseo 的流程固化下来。目标是让少量有价值的上下文可以在 Paseo 桌面端或手机端继续使用，而不是复制代码、复制会话文件或创建新的 worktree。

## 一、先记住三个边界

1. **迁移是登记上下文，不是复制数据。** 原始 provider session 始终保留；导入不会复制项目文件，也不应自动创建新的 worktree。
2. **默认只盘点，不批量导入。** 只有用户明确给出 session ID，或从候选清单中逐条选定后，才执行导入。
3. **daemon 是共享控制面。** daemon 不可达时不能自行 `start`、`restart`；同一个共享 dirty `cwd` 同时只允许一个会写文件的 agent。

适用 provider：`codex`、`claude`。本地 session 的 `source=codex` 或 `source=claude` 只表示来源，不等于 Paseo 当前存在可用的 provider ID。

## 二、迁移前的决策表

| 现场情况 | 处理方式 |
|---|---|
| 用户只说“看看有哪些会话” | 只读盘点，输出候选，不导入 |
| 用户明确给出 session ID | 先做 daemon/provider/重复导入预检，再逐条导入 |
| 候选较多 | 按项目、更新时间、未完成工作和是否已导入筛选，建议保留约 10–30 条 |
| daemon 返回 `DAEMON_NOT_RUNNING` 或 transport/WebSocket 错误 | 报告状态和下一步授权需求，停止迁移；不自行重启 |
| 目标 `cwd` 有其他正在写入的 agent | 暂停该条迁移，先隔离 worktree 或等现有 agent 完成 |
| session 已经导入 | 不重复导入，使用现有 Paseo agent 继续 |

## 三、标准工作流

### 1. 确认范围

先明确以下信息：

- 目标项目的**原始绝对路径**（`cwd`）；默认不跨项目盘点。
- provider：`codex`、`claude` 或 `all`。
- 项目短名和标签，例如 `project=sapient`、`source=codex`。
- 是只盘点、核验某个 ID，还是导入用户已经选定的会话。

未明确选择时，不应把“最近修改过的所有会话”直接导入。

### 2. 只读预检 Paseo

先确认 CLI 参数和 daemon 状态：

```bash
paseo import --help
paseo daemon status --json
```

只有 daemon 连通后，才继续读取 provider 和已登记 agent：

```bash
paseo provider ls --json
paseo ls --all --global --json
```

provider ID 必须以实际的 `paseo provider ls --json` 输出为准。不要把本地 session 的来源名直接当作 `--provider` 参数。

若出现以下任一情况，应停止在预检阶段：

- `DAEMON_NOT_RUNNING`；
- `Transport closed (code 1006)` 或其他 WebSocket/transport 错误；
- 无法可靠读取全局 Paseo agent 列表。

此时可以只读查看 `~/.paseo/daemon.log` 辅助诊断，但不得自行执行 `paseo start`、`paseo daemon start` 或任何 restart。是否恢复 daemon，需要用户明确授权，并应另按部署方式执行对应的运维流程。

### 3. 生成本地候选清单

Skill 内置的盘点脚本只读取会话元数据、路径和文件修改时间，不输出对话内容：

```bash
python3 scripts/inventory_sessions.py \
  --provider all \
  --cwd /path/to/project \
  --since-days 30 \
  --limit 50
```

需要机器可读结果时：

```bash
python3 scripts/inventory_sessions.py \
  --provider codex \
  --cwd /path/to/project \
  --since-days 14 \
  --json
```

脚本的读取范围和行为是固定的：

- Codex：`~/.codex/sessions/**/*.jsonl`；
- Claude Code：`~/.claude/projects/**/*.jsonl`；
- 每个 JSONL 最多读取开头 128 行；
- 通过可验证的 session 元数据提取 ID 与 `cwd`，无法确认时计入 `unresolved_count`，不臆测映射；
- 同一 provider/session ID 只保留最新文件记录；
- `--cwd` 按解析后的绝对路径完全匹配；
- 默认不展示未解析文件，排障时才加 `--include-unresolved`。

候选筛选应优先考虑：是否仍有未完成工作、最后一次有效活动时间、项目归属、是否已导入，以及是否会与现有 agent 争用同一工作目录。不要仅按文件修改时间批量迁移；应排除审批包装、已中止副本和历史回放。

### 4. 检查重复导入与并发写入

对每个候选 source ID，在 `paseo ls --all --global --json` 的**实际返回结构**中核对是否已经存在。不要只依赖 agent 名称或本地文件名判断。

同时检查目标 `cwd` 是否已有运行中的 Paseo agent，尤其是可能修改文件的 agent。共享 dirty 工作目录下，迁移本身不会提供隔离；需要隔离修改时，必须另行取得授权后创建 Paseo worktree workspace。

### 5. 逐条导入

每条已确认、未重复且无并发风险的会话单独执行：

```bash
paseo import <source-session-id> \
  --provider <provider-id> \
  --cwd /path/to/project \
  --label project=<project-slug> \
  --label source=<codex-or-claude> \
  --label status=resume \
  --json
```

约束：

- `<provider-id>` 使用预检阶段实测的值；
- `<source-session-id>`、`cwd` 和标签必须来自已确认的本地数据或用户输入；
- 不把 shell 变量、对话内容或未验证字符串拼接进命令；
- 一次只处理一条，失败后先保留错误、source ID、provider ID 与 daemon 状态，修复前置条件后最多在用户确认下重试一次。

### 6. 立即验证结果

导入返回 Paseo agent ID 后，马上执行：

```bash
paseo inspect <agent-id> --json
paseo ls --all --global --json
```

至少核对以下字段：

- source/session 是否对应原始会话；
- provider 是否为预检得到的 provider ID；
- `cwd` 是否为原始项目路径；
- `project`、`source`、`status=resume` 标签是否齐全；
- agent 状态是否符合预期；
- 全局列表是否只新增了本次预期的一条记录。

验证通过后，用户可以继续对话：

```bash
paseo attach <agent-id>
paseo send <agent-id> "<下一条任务>"
```

## 四、迁移后的管理

- 原始 Codex/Claude session 永不因迁移而删除。
- 暂时不用的 Paseo agent 使用软归档：

  ```bash
  paseo archive <agent-id>
  ```

- 归档前确认没有未保存的工作或待处理审批；归档只影响 Paseo agent 展示，不等于删除原始 provider session。
- 同一共享 `cwd` 中的导入 agent 不应长期并发写入；要同时开展独立任务，优先使用独立 worktree。
- 定期重新盘点时，先检查已有导入记录，再处理新候选，避免重复导入。

## 五、隐私与审计要求

会话清单可能包含本地绝对路径、session ID 和项目名称。盘点输出只在用户授权范围内使用，不发送到外部服务，也不要把对话内容写入报告或日志。

迁移记录至少应保留：目标 `cwd`、provider、source ID、Paseo agent ID、导入时间、验证结果和失败时的 CLI 错误。凭据、密码、token 和会话正文不应记录。

## 六、最小验收标准

一次迁移只有满足以下条件才算完成：

1. daemon 在迁移前已被只读核验为可达；
2. provider ID 来自实时 provider 列表；
3. source ID 未重复导入，且目标 `cwd` 无共享写入冲突；
4. 导入是逐条、用户明确选择的；
5. `inspect` 与全局列表已核对 source/session、provider、`cwd`、标签和新增数量；
6. 原始 session 仍保留，用户已获得 `attach` 或 `send` 的续聊入口。

相关文档：

- [usage.md](../getting-started/usage.md)：常用 CLI 与用户侧操作；
- [architecture.md](../internals/architecture.md)：daemon、agent 和 provider 的架构关系；
- [linux-ops.md](../ops/linux-ops.md)：daemon、systemd、端口和连接故障排查。
