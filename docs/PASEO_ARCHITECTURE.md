# Paseo 原理与架构

> 本文档描述 paseo 的工作原理、组件分层、连接拓扑,以及它在这台机器上的具体部署形态。运维操作见 `PASEO_OPS.md`,决策记录见 `PASEO_ADR.md`。

---

## 一、Paseo 是什么

Paseo(`@getpaseo/cli`,当前 0.2.4)是一个**让你从手机驱动服务器上 AI coding agent** 的工具。服务器上跑一个 daemon,手机 app 通过它远程操作服务器上的 claude(或其他 agent),效果就像在手机上开了一个服务器端的终端 + AI 助手。

核心价值:**NAT 穿透**(手机不用和服务器在同一网络)+ **移动端 UI**(手机上发任务、看 agent 输出、审批操作)。

---

## 二、组件分层(三层进程模型)

paseo 在服务器上是**三层进程**结构:

```
paseo daemon start --foreground   ← systemd 托管(Type=simple)
   │
   ├─ Paseo Supervisor            ← 监护层,负责拉起/重启 daemon
   │     │
   │     └─ Paseo Daemon          ← 核心服务层
   │           │  (监听 0.0.0.0:8767,维护到 relay 的出站 wss)
   │           │
   │           ├─ terminal-worker-process   ← 每个手机会话的终端宿主
   │           │     │
   │           │     └─ claude --output-format stream-json ...   ← 实际的 AI agent 进程
   │           │
   │           └─ (其他 agent provider:codex / copilot / cursor / opencode ...)
```

**各层职责:**

| 层 | 进程示例 | 职责 |
|---|---|---|
| **CLI/前台** | `node .../paseo daemon start --foreground` | systemd 直接管理的进程(MainPID),`--foreground` 保证不 daemonize,让 systemd 能跟踪 |
| **Supervisor** | `Paseo Supervisor` | 监护 daemon,daemon 崩了由它重启;它的环境变量在它启动那一刻就固定 |
| **Daemon** | `Paseo Daemon` | 核心服务:监听本地端口、维护 relay 长连接、管理 agent 会话、对外提供 WebSocket API |
| **terminal-worker** | `terminal-worker-process.js` | 每个手机会话派生一个,承载 PTY 和 agent 进程 |
| **agent** | `claude --output-format stream-json ...` | 真正干活的 AI agent,以 stream-json 协议和 daemon 通信 |

> **关键:** 手机上看到的 claude 进程,环境变量是从 **daemon 继承**来的(经 supervisor → daemon → worker → claude 这条链)。`terminal.js` 里 `createExternalProcessEnv(process.env, input.env)` 就是把 daemon 的 `process.env` 作为终端环境的基础。

---

## 三、连接拓扑(两种模式)

paseo 手机端有两条路连到 daemon:

### 模式 A:Relay(中继,默认)
```
手机 app ──> relay.paseo.sh(Fly.io,美国,anycast) <── 出站 wss:443 ── daemon
```
- daemon **主动外连** `wss://relay.paseo.sh:443`(出站,不需要开入站端口/端口转发)。
- 手机也连到同一个 relay,relay 桥接两端。
- **优点:** 零配置,NAT 后也能用。
- **缺点:** 多一跳(海外 relay),国内手机链路长、易抖。
- 流量 E2EE。

### 模式 B:Direct connect(直连)
```
手机 app ──────────> daemon 的监听端口(本机 0.0.0.0:8767)
```
- 手机 app 配对界面选 "Direct connect" tab,填 `主机IP:端口` + 密码。
- 绕开 relay,少一跳。
- **前提:** 端口要对公网开放(云防火墙/安全组入站放行该端口),且**必须设密码**(`paseo daemon set-password`)。
- 手机在 Direct connect tab 填 `<服务器公网IP>:<端口>` + 密码。

**当前部署两种都开**(relay 作兜底,直连为主)。手机 Direct connect 连不上时自动回落到 relay。

---

## 四、Agent Provider 机制

paseo 不绑定 claude,它是一个**多 provider 的 agent 宿主**。`server/agent/providers/` 下支持:

- **claude**(主要用这个)—— 通过 `@anthropic-ai/claude-agent-sdk` 驱动 `claude` CLI
- codex / copilot / cursor / opencode / kiro / trae / acp / omp / pi 等

每个 agent 会话由 daemon 派生一个子进程(如 `claude --output-format stream-json --model <X> --resume=<id> ...`),用 stream-json 协议双向通信。手机端的「换模型」= 改这个子进程的 `--model` 参数(新会话才生效;resume 旧会话沿用旧参数)。

---

## 五、配置与凭据流向

```
/etc/systemd/system/paseo.service
   └─ EnvironmentFile=/root/.paseo/paseo.env   ← IS_SANDBOX + ANTHROPIC_* 注入 daemon
          │
          ▼ (daemon 继承 → worker 继承 → claude 继承)
/root/.paseo/config.json   ← listen 端口、relay 开关、auth.password(bcrypt)
/root/.paseo/paseo.env     ← 注入到 daemon 的环境变量
~/.claude/settings.json    ← claude 自己读的 env 块(凭据 + 模型映射),claude 进程启动时再覆盖一层
```

**两条凭据来源,别搞混:**
1. `paseo.env`(systemd 注入)→ 决定 daemon 派生的 claude 进程的**初始环境**。
2. `~/.claude/settings.json` 的 `env` 块 → claude 启动时自己读取并覆盖。
两者都指向同一个模型代理(`ANTHROPIC_BASE_URL`)和 token。

---

## 六、关键端口与外部依赖

| 项 | 值 | 说明 |
|---|---|---|
| daemon 监听 | `0.0.0.0:<PORT>`(自选,如 8767) | 直连 + 本地 CLI 用 |
| relay 地址 | `wss://relay.paseo.sh:443` | 海外 anycast,出站 |
| 手机直连 | `<服务器公网IP>:<PORT>` | 服务器公网 IP |
| 模型代理 | `$ANTHROPIC_BASE_URL` | claude 的 ANTHROPIC_BASE_URL,所有模型请求走这里 |

---

## 七、数据流:手机发一条消息到 claude 回响应

```
1. 手机 app ──(wss)──> relay 或 直连 8767 ──> daemon
2. daemon 把消息转成 stream-json,通过 stdin 喂给 claude 进程
3. claude 调模型(经 `$ANTHROPIC_BASE_URL` 代理)──> 模型返回
4. claude 的 stdout(stream-json 事件流)──> daemon 捕获 ──> 转发回手机
5. 手机 app 渲染 agent 的流式输出 / 工具调用 / 审批请求
```

「卡住」通常卡在第 3 步(模型代理不响应)或第 1 步(链路抖)。排查见 `PASEO_OPS.md` 坑 #7。

---

## 八、与 systemd 的关系

paseo 由 `paseo.service` 托管:
- `ExecStart = node ... paseo daemon start --foreground` —— 前台跑,systemd 直接管。
- `EnvironmentFile=/root/.paseo/paseo.env` —— 注入 IS_SANDBOX 等关键变量。
- `Restart=on-failure` —— 失败退出自动重启(但干净退出 status=0 不重启,见 OPS 坑 #6)。
- `User=root` —— 整条链以 root 跑(这也是为什么 IS_SANDBOX 是命根子)。

**核心约束:必须经 systemd 启动,否则丢了 IS_SANDBOX → root 拦截。** 手动 `paseo daemon start` 会绕过 paseo.env,且和 systemd 抢实例。
