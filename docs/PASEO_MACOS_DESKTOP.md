# Paseo macOS 桌面版运维手册

> 本文专门覆盖 **macOS 桌面 App（`/Applications/Paseo.app`）** 场景。如果你在 Linux 服务器上跑 `paseo daemon`，请看 [PASEO_OPS.md](./PASEO_OPS.md)。

桌面版和服务器版是**两种完全不同的部署形态**，很多在 Linux 上是常识的东西，在桌面版上不成立。先把差异讲清楚，再给配置与排查指南。

---

## 一、桌面版 vs 服务器版

| 维度 | macOS 桌面版 | Linux 服务器版 |
|---|---|---|
| 安装形态 | `/Applications/Paseo.app`（Electron） | `npm i -g @getpaseo/cli` 或二进制 |
| daemon 进程 | App 的子进程（`Paseo Helper`），由 App 主进程托管 | `paseo daemon` 独立进程 |
| 托管/自启 | App 自己拉起，无 launchd plist | systemd 单元，`enabled` 开机自启 |
| 版本号 | 跟 App 走（`Info.plist: CFBundleShortVersionString`），无独立 daemon 版本 | `paseo --version` |
| 配置文件 | `~/.paseo/config.json`（同服务器版） | 同左 |
| Home 目录 | 默认 `~/.paseo`，可用 `PASEO_HOME` 覆盖 | 同左 |
| 重启方式 | 退出 App + 重新打开 / `open -a Paseo` | `systemctl restart paseo` |
| 密码 | 一般不需要（loopback） | 强制必设（见 [OPS 坑 #4](./PASEO_OPS.md)） |
| 远程连接 | 通过 `relay` 中转（`relay.enabled: true`） | 直连 + relay 兜底 |

**关键事实**：桌面版没有独立 daemon，它就是 App 内部的一个 worker。App 退了，daemon 就没了（除非有僵尸 worker，见 [坑 #2](#坑-2--app-退出后-worker-变僵尸占端口)）。

---

## 二、当前部署快照（作者本机）

| 项 | 值 |
|---|---|
| App 版本 | `0.2.5`（`CFBundleShortVersionString` = `CFBundleVersion`） |
| 配置 schema 版本 | `config.json: "version": 1`（**不是 daemon 版本**） |
| 监听 | `127.0.0.1:6767`（loopback only，不开公网） |
| 配置文件 | `~/.paseo/config.json` |
| 中继 | 启用（`relay.enabled: true`），用于手机远程连本机 |
| 密码 | 未设（仅 loopback，风险可接受） |
| 开机自启 | **未配置 launchd**，依赖 macOS 登录项（系统设置 → 通用 → 登录项与扩展） |
| worktrees 体积 | 约 **2.2 GB**（位于 `~/.paseo/worktrees/`） |

---

## 三、进程结构（排查时先认清这些）

```
/Applications/Paseo.app/Contents/MacOS/Paseo                  ← Electron 主进程
  ├─ Paseo Helper (GPU)                                       ← GPU 进程
  ├─ Paseo Helper (Network)                                   ← 网络进程
  ├─ Paseo Helper (Renderer)                                  ← 渲染进程
  ├─ Paseo Helper .../daemon-worker.js            ← **daemon 本体**，监听 6767
  └─ Paseo Helper .../terminal-worker-process.js              ← 终端 worker
```

### 常用排查命令

```bash
# 看 daemon 是否在监听
lsof -nP -iTCP:6767 -sTCP:LISTEN

# 看所有 Paseo 子进程
ps aux | grep -i "Paseo Helper" | grep -v grep

# 看 daemon 日志（trace 级别，默认 10MB × 2 份轮转）
tail -f ~/.paseo/daemon.log

# 看 App 版本
defaults read /Applications/Paseo.app/Contents/Info.plist CFBundleShortVersionString
```

---

## 四、配置调优建议

基于官方文档 https://paseo.sh/docs/configuration 与本机实际情况，以下 4 项对 macOS 桌面版用户真正有用，按优先级排：

### 🔴 强烈建议

#### 1. `log.file.rotate` —— 日志体积控制
默认文件日志是 `trace` 级别，长期跑会写满磁盘。显式声明轮转，并把 level 降到 `info`。

```json
"log": {
  "file": {
    "level": "info",
    "path": "daemon.log",
    "rotate": { "maxSize": "10m", "maxFiles": 2 }
  }
}
```

> 注：默认就是 `trace + 10m × 2`，本条主要是把 level 调成 `info` 大幅减体积。

#### 2. `worktrees.root` —— 挪出 `~/.paseo` 省磁盘
`~/.paseo/worktrees/` 会随着使用不断增长（本机已达 2.2 GB）。如果有外置盘 / 更大的分区，建议挪出去。

```json
"worktrees": { "root": "/Volumes/External/paseo-worktrees" }
```

相对路径基于 `PASEO_HOME` 解析。**注意**：只影响**新建**的 worktree，已有的不会自动迁移。

### 🟡 看场景

#### 3. `daemon.auth.password` —— 密码保护
仅 loopback 使用**不需要**。但只要满足以下任一条件就**强烈建议**设密码：
- 改监听 `0.0.0.0` 或局域网 IP
- 通过 `relay` 远程连接（手机 / 平板）

设置方法（一行命令，写入 bcrypt hash 到 `config.json`）：

```bash
paseo daemon set-password
```

或环境变量：`PASEO_PASSWORD=my-secret paseo daemon start`

#### 4. `$schema` —— 编辑器自动补全
零成本体验项，VS Code / Cursor / Zed 打开 `config.json` 时有字段提示和校验。

```json
{
  "$schema": "https://paseo.sh/schemas/paseo.config.v1.json",
  ...
}
```

### ❌ 桌面版通常用不上的

- `daemon.hostnames` —— loopback 监听不需要 host 白名单
- `features.webUi` —— 用桌面 App 就不需要 daemon serve 浏览器 UI
- `daemon.listen: 0.0.0.0` —— 没远程直连需求就不用
- `PASEO_VOICE_*` / `PASEO_DICTATION_*` —— 默认 local provider，没配 OpenAI key 时用不上

---

## 五、踩坑记录

### 坑 #1 —— 端口冲突 `EADDRINUSE 127.0.0.1:6767`

**现象**：daemon.log 反复出现：
```
Error: listen EADDRINUSE: address already in use 127.0.0.1:6767
DaemonRunner: Worker crashed (code 1). Restarting worker...
```
App 可用，但日志疯狂滚动。

**根因**：同一个 App 下有两个 worker 在抢端口 —— 一个老 worker 一直占着 6767，新 worker 抢不到就崩，`DaemonRunner` 再重启它，循环。

**清理步骤**（已验证）：
```bash
# 1. 优雅退出 App
osascript -e 'tell application "Paseo" to quit'
sleep 2

# 2. 确认子进程都退出（正常应为空）
ps aux | grep -i "Paseo Helper" | grep -v grep

# 3. 如果还有残留（僵尸 worker），杀掉
kill <pid1> <pid2>

# 4. 确认端口已释放（应为空）
lsof -nP -iTCP:6767 -sTCP:LISTEN

# 5. 重启 App
open -a Paseo
sleep 8

# 6. 验证新 daemon 起来了
lsof -nP -iTCP:6767 -sTCP:LISTEN   # 应该能看到新 PID
tail -5 ~/.paseo/daemon.log        # 应该看到 "Bootstrap complete"
```

### 坑 #2 —— App 退出后 worker 变僵尸占端口

**现象**：`Paseo.app` 主进程已退出，但 `Paseo Helper` 子进程还在跑，占着 6767。下次开 App 就触发 [坑 #1](#坑-1--端口冲突-eaddrinuse-1270016767)。

**根因**：App 正常退出时应该带走所有 Helper，但某些异常情况（崩溃 / 强制杀主进程）会留下孤儿。

**预防**：每次关 App 后，顺手 `ps aux | grep "Paseo Helper" | grep -v grep` 扫一眼。

### 坑 #3 —— `config.json: "version": 1` 不是 daemon 版本

**误读**：以为是 daemon 版本号，到处找 0.2.5 在哪写。

**正解**：那个 `version` 是**配置文件 schema 版本**，用于未来 config 格式升级时做迁移。daemon 没有独立版本号，跟 App 走，查 `Info.plist`。

---

### 坑 #4 —— 运行中切 `auto` 报 `auto mode unavailable for this model`

**现象**：在已运行的 Claude agent 上切 auto 模式，UI 报 `Cannot set permission mode to auto: auto mode unavailable for this model`，daemon.log 里能搜到完整 stack（搜 `set_agent_mode_request error`），错误源在 `@anthropic-ai/claude-agent-sdk` 的 `set_agent_mode` handler。

**根因**：SDK 对 `auto` 权限模式有**模型白名单**，只放当前一代（`claude-sonnet-4-6` / `claude-opus-4-6`）。旧 agent 状态文件（`~/.paseo/agents/<workspace>/<agentId>.json`）固化的 `claude-sonnet-4`（老 ID）不在白名单里。**这不是 daemon bug，重启 daemon 无用，还会杀掉所有运行中的 agent。**

**确认 agent 当前 model**：
```bash
python3 -c "
import json
d=json.load(open('$HOME/.paseo/agents/<workspace>/<agentId>.json'))
print('provider=', d.get('provider'))
print('config.model=', d.get('config',{}).get('model'))
print('config.modeId=', d.get('config',{}).get('modeId'))
"
```

**修复路径**（两层配合，详见 [ADR-0007](./PASEO_ADR.md)）：
1. **中转层**：把你 Claude API 代理的 `claude-sonnet-4` alias 到 `claude-sonnet-4-6`。
2. **Paseo 端**：用 `update_agent({ agentId, settings: { model: "claude-sonnet-4-6" } })` 改 agent model，再切 `auto`。新建 agent 用 `provider: "claude/sonnet"`，让 Paseo 自动解析到当前一代，不要硬写 model 字符串。

**注意**：`auto` 模式不是 "allow all"。它用分类器判断每个 permission prompt，对 `/tmp` 写入这类边界操作仍会弹 request（实测）。要零打断用 `bypassPermissions`（风险自担）。

---

## 六、开机自启配置（可选）

桌面版默认**不会**开机自启。两种做法：

### 方法 A：macOS 登录项（推荐）
系统设置 → 通用 → 登录项与扩展 → 把 `Paseo.app` 拖进去。这是 Apple 官方方式，零配置。

### 方法 B：launchd plist（高级）
如果你想要更可控的自启（比如崩溃自动重启），可以写一个 `~/Library/LaunchAgents/sh.paseo.daemon.plist`。但要注意：

- **不能直接启动 daemon worker**，它是 App 的子进程，脱离 App 起不来
- 只能启动整个 `Paseo.app`：`<string>open -a Paseo</string>`
- 这样做会和 App 自己的进程托管**重复**，不推荐

绝大多数人选方法 A 就够了。

---

## 七、与服务器版文档的交叉引用

| 如果你在找…… | 去看 |
|---|---|
| 三层进程模型 / relay 拓扑 | [ARCHITECTURE](./PASEO_ARCHITECTURE.md) |
| 手机连不上（链路 vs 模型）排查 | [OPS 坑 #6](./PASEO_OPS.md) |
| 改端口 / 开公网 / 设密码的安全顺序 | [OPS 坑 #4](./PASEO_OPS.md) |
| `IS_SANDBOX=1` 的环境继承问题 | [OPS 坑 #1](./PASEO_OPS.md)（桌面版几乎不会遇到，因为不用 root） |
| `auto` 模式模型白名单 / alias 方案 | [坑 #4](#坑-4--运行中切-auto-报-auto-mode-unavailable-for-this-model) + [ADR-0007](./PASEO_ADR.md) |
| 语音 / 听写的 provider 配置 | https://paseo.sh/docs/configuration → Voice 段 |
