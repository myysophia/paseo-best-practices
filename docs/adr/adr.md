# ADR: Paseo 部署决策记录

> Architecture Decision Records。每条记录一个不可逆(或难逆)的工程决策:为什么这么决定、否了什么、代价是什么。改这些决定前先读对应 ADR。
>
> ADR 索引
> - [ADR-0001 用 systemd 托管 paseo,而非手动/脚本启动](#adr-0001-用-systemd-托管-paseo)
> - [ADR-0002 用 EnvironmentFile 注入 IS_SANDBOX,而非写 .bashrc](#adr-0002-用-environmentfile-注入-is_sandbox)
> - [ADR-0003 开放公网直连,而非只走 relay](#adr-0003-开放公网直连而非只走-relay)
> - [ADR-0004 relay 保留启用,与直连并存](#adr-0004-relay-保留启用与直连并存)
> - [ADR-0005 从公网明文直连切到 Tailscale 加密直连](#adr-0005-从公网明文直连切到-tailscale-加密直连)
> - [ADR-0006 双入口并存：公网（办公网段）+ Tailscale tailnet](#adr-0006-双入口并存公网办公网段--tailscale-tailnet)
> - [ADR-0007 用模型 alias 让旧 model ID 通过 auto 模式校验](#adr-0007-用模型-alias-让旧-model-id-通过-auto-模式校验)

---

## ADR-0001 用 systemd 托管 paseo

- **状态:** Accepted
- **日期:** 2026-07-30

**背景**
paseo 需要长期运行,且必须随服务器重启自动起来。手动 `paseo daemon start` 每次重启服务器都得手动恢复,且没人监护(崩了不会自起)。

**选项**
1. systemd 服务(前台 `--foreground`,Type=simple)
2. 手动启动 / nohup / screen
3. 自己写 supervisor 脚本

**决定**
选 1。`/etc/systemd/system/paseo.service`,`ExecStart` 用 `paseo daemon start --foreground`,enabled 开机自启。

**理由**
- 开机自启、崩了自拉起(`Restart=on-failure`)、统一日志(`journalctl -u paseo`)。
- `--foreground` 让 systemd 直接持有进程,符合 Type=simple 语义。
- paseo 自己的 Supervisor 子层仍保留,做 daemon 级监护;systemd 做最外层。

**后果**
- ✅ 重启自动恢复,运维命令统一为 `systemctl ...`。
- ⚠️ **强约束:不准手动 `paseo daemon start/restart`。** 手动启动会绕过 systemd 的 EnvironmentFile(丢 IS_SANDBOX),且和 systemd 实例抢端口。此约束已踩坑多次(见 OPS 坑 #3)。
- ⚠️ `Restart=on-failure` 不救干净退出(SIGTERM 的 status=0),需手动 start,或改 `Restart=always`(见 OPS 坑 #5)。

---

## ADR-0002 用 EnvironmentFile 注入 IS_SANDBOX

- **状态:** Accepted
- **日期:** 2026-07-30

**背景**
claude 以 root 跑,源码硬性拒绝 `root + --dangerously-skip-permissions`,除非 `IS_SANDBOX=1`。手机上的 claude 由 paseo daemon 派生,**不读 `.bashrc`**。需要可靠地把 `IS_SANDBOX=1` 塞进 daemon 的环境。

**选项**
1. systemd `EnvironmentFile=~/.paseo/paseo.env`(600 权限)
2. 写 `export IS_SANDBOX=1` 到 `~/.bashrc`
3. 写到 `/etc/environment`
4. 写进 paseo 的 agent 配置

**决定**
选 1,辅以 `/etc/environment` 兜底。

**理由**
- 选项 2 无效——paseo 派生终端不读 `.bashrc`(反复踩坑)。
- 选项 3(`/etc/environment`)对登录 shell 生效,但依赖「paseo 从登录 shell 启动」;如果将来从别的方式起就不稳。
- 选项 1 直接绑定 systemd 启动链,**只要 paseo 经 systemd 起,变量必然在**——和「不准手动起」约束(ADR-0001)互锁,确定性最高。
- 选项 4 不可行:paseo agent 配置无 env 字段。
- 600 权限避免 token 明文落到 644 的 unit 文件里。

**后果**
- ✅ IS_SANDBOX 注入确定性强,可由 `/proc/<daemon>/environ` 验证。
- ⚠️ 强依赖 ADR-0001 的约束:一旦手动启动,变量就丢。
- ⚠️ 加减环境变量(如换 token)要改 `paseo.env` + `systemctl restart paseo`,不是改 bashrc。

---

## ADR-0003 开放公网直连,而非只走 relay

- **状态:** ~~Accepted~~ → **Superseded by ADR-0005**（2026-08-03，因明文窃听风险切到 Tailscale）
- **日期:** 2026-07-30

**背景**
手机若与服务器跨网(如手机在国内、服务器在海外),relay 模式链路是 `手机 → 海外 relay → 服务器`,两段海外 + GFW,频繁断重连(daemon.log 里上万条 disconnect/reconnect)。

**选项**
1. 只走 relay(默认,零配置)
2. 开公网端口,手机 Direct connect 直连服务器
3. 自建 relay

**决定**
选 2,relay 保留(见 ADR-0004)。

**理由**
- 直连去掉中间那一跳,链路从两段海外变一段,抖动改善。
- 手机 app 原生有 "Direct connect" tab,支持直连。
- relay 保留作兜底,直连连不上自动回落,无副作用。

**后果**
- ✅ 少一跳,延迟和稳定性改善。
- ⚠️ **暴露面扩大**:daemon 能以 root 起终端,公网可达 = 高价值目标。强制要求设密码(`auth.password`),且密码必须强。
- ⚠️ **手机若仍过 GFW + 长途海底光缆**,直连只是「好一点」,不是「稳」(地理决定)。
- ⚠️ 改 daemon 端口必须同步更新云防火墙/安全组入站规则(见 OPS 坑 #4)。
- ⚠️ 模型代理(`$ANTHROPIC_BASE_URL`)的稳定性独立于连接方式(「卡住」可能是代理,见 OPS 坑 #6)。

---

## ADR-0004 relay 保留启用,与直连并存

- **状态:** Accepted
- **日期:** 2026-07-30

**背景**
开了直连后,是否关掉 relay(`--no-relay` / config.json `relay.enabled=false`)?

**选项**
1. 关 relay,纯直连
2. relay + 直连并存

**决定**
选 2。`config.json: relay.enabled = true` 保持不变。

**理由**
- relay 作为 fallback:直连连不上(GFW 抖、IP 变、防火墙规则变)时仍可访问,可用性更高。
- 并存无冲突,daemon 同时维护监听端口和出站 wss。
- 关掉 relay 省的那点资源/暴露面微不足道,不值得牺牲 fallback。

**后果**
- ✅ 双通道,可用性最高。
- ⚠️ daemon.log 会持续有 relay 心跳/断重连日志(正常噪音,非故障)。

---

## 关于「公网入站源 CIDR」的补充决策

直连模式需在云防火墙/安全组放行入站端口。源 CIDR 怎么选,取决于你的场景,这里只给原则,不给具体决策(因为和具体云厂商、网络环境强相关):

| 选项 | 适用 | 代价 |
|---|---|---|
| 收窄到手机出口 IP `/32` | 手机 IP 固定(少见) | IP 一变就连不上,需高频维护 |
| 限定运营商/办公网段 | 能确定稳定网段 | 网段宽窄不一,收窄效果有限 |
| 全开 `0.0.0.0/0` | 手机 IP 频繁变化、无法收窄 | **风险最高**:全网可探,完全依赖 daemon 密码兜底 |

无论选哪种:**密码必须强、定期轮换**;paseo daemon 能以 root 起终端,任何认证绕过/RCE 漏洞都等于服务器 root。若长期不用,优先收窄或关闭入站。

---

## ADR-0005 从公网明文直连切到 Tailscale 加密直连

- **状态:** ~~Accepted~~ → **Superseded by ADR-0006**（双入口并存）
- **日期:** 2026-08-03
- **取代:** ADR-0003 的公网直连部分（ADR-0003 标记为 Superseded by ADR-0005）

**背景**
ADR-0003 选了「开公网端口 + 密码」的直连。实测抓包验证（见 [DIRECT_CONNECT](../guides/direct-connect.md) §二）：`0.0.0.0` + 密码模式下，所有 API 流量（任务内容、shell 命令、工作目录、Authorization 头）在公网线缆上**全明文**。密码只防未授权访问，不防窃听。办公网段/同云用户都可被动嗅探。

**选项**
1. 保持 `0.0.0.0` + 密码（明文，有窃听风险）
2. 在 daemon 前加 Caddy/Nginx 反代做 HTTPS（需域名 + 证书运维）
3. **Tailscale：daemon 绑 tailnet IP，WireGuard 加密，不开公网端口**
4. 纯 relay（放弃直连的低延迟）

**决定**
选 3。daemon 监听从 `0.0.0.0:8767` 改为 `<TS_IP>:8767`，两端 Tailscale 客户端都 `--accept-dns=false`（规避 MagicDNS 接管导致的断网问题，见下），撤销云防火墙公网 8767 入站规则。

**理由**
- 唯一能让「直连 + 加密 + 不开公网端口」三者兼得。
- WireGuard 加密所有流量，消除嗅探风险。
- daemon 不再暴露公网，攻击面归零（公网扫描扫不到）。
- Tailscale DERP 兜底中继也能改善国内到海外的连通性。
- 客户端断网问题（MagicDNS 接管系统 DNS）有确定修复：`tailscale set --accept-dns=false`。

**代价 / 注意**
- ⚠️ 依赖 Tailscale 基础设施。tailnet 挂了 = 直连断（relay 仍作兜底）。
- ⚠️ 每台接入设备都要装 Tailscale + 加入同一 tailnet。新设备接入成本略增。
- ⚠️ **Tailscale MagicDNS 会改系统 DNS 导致断网**（GitHub issues #14924/#10225/#16985）。**强制约束：所有节点 `tailscale set --accept-dns=false`**，或 admin console 关 MagicDNS / 不勾 Override local DNS。
- ⚠️ `--accept-dns=false` 后失去 tailnet 自定义 DNS 短名，但用 `100.x.y.z` IP 直连不影响功能。

**回滚**
改回 `0.0.0.0:8767` + 重开云防火墙 8767 入站即可（旧配置已备份在 `~/.paseo/config.json.bak.*`）。

---

## ADR-0006 双入口并存：公网（办公网段）+ Tailscale tailnet

- **状态:** Accepted
- **日期:** 2026-08-03
- **取代:** ADR-0005（ADR-0005 标记为 Superseded by ADR-0006）

**背景**
ADR-0005 把 daemon 绑死在 tailnet IP，公网完全关闭。但实际需求是：
- 手机 / 跨网客户端要走 tailnet（加密、绕 GFW）。
- 办公网段内的设备（如办公电脑）希望直接走公网 IP，不必每台都装 Tailscale。

绑死 tailnet 后，办公网段设备无法访问。

**选项**
1. 只 tailnet（ADR-0005 的方案，办公网段要装 Tailscale 才能用）
2. 只公网（ADR-0003 的方案，明文 + 跨网不安全）
3. **双入口：`0.0.0.0:8767` 监听 + SG 分层放行 + 客户端按场景选入口**
4. 反代 + HTTPS（需域名/证书运维）

**决定**
选 3。daemon 监听改回 `0.0.0.0:8767`（同时覆盖公网、tailnet、lo）；云防火墙只放行需要的来源：
- bastion SG：撤销 `0.0.0.0/0`（全网开放）规则。
- office-sg：保留办公网段 `/30` 全端口规则（办公网段设备走公网 IP）。
- tailnet 流量不经云防火墙（走 Tailscale 虚拟网卡，daemon 在 `0.0.0.0` 也能接到）。

客户端入口选择：
- **手机 / 跨网客户端** → tailnet `<TS_IP>:8767`（WireGuard 加密，**推荐**）
- **办公网段设备** → 公网 `<SERVER_IP>:8767`（明文，但已在可信办公网段内，可接受）

**理由**
- 兼顾「跨网加密」和「办公网段免 Tailscale」两个需求。
- `0.0.0.0` 监听同时接到 tailnet 和公网流量，无需多实例。
- SG 收窄后公网入口只在办公网段可达，攻击面比 ADR-0003 的全网开放小。
- 手机走 tailnet 仍享受 WireGuard 加密 + DERP 兜底。

**代价 / 注意**
- ⚠️ 公网入口仍是明文 HTTP/WS。办公网段内可接受；若担心同网段嗅探，办公网段设备也可改装 Tailscale 走 tailnet。
- ⚠️ 监听 `0.0.0.0` 意味着如果 SG 配错（误开 `0.0.0.0/0`），daemon 立刻全网暴露。**强约束：SG 规则改动后必须复核**（`aws ec2 describe-security-groups` 确认 8767 入站来源 CIDR）。
- ⚠️ 客户端要清楚选哪个入口——配 host 时填错 IP（手机填了公网 IP）会走明文，失去加密。
- ⚠️ daemon 能以 root 起终端，密码必须强、定期轮换。

**回滚**
- 想完全关公网：改回 `listen: "<TS_IP>:8767"` + `systemctl restart paseo`（SG 规则可留着不影响）。
- 想完全开公网（不推荐）：bastion SG 重新加 `0.0.0.0/0`。

---

## ADR-0007 用模型 alias 让旧 model ID 通过 auto 模式校验

- **状态:** Accepted
- **日期:** 2026-08-06

**背景**
Paseo 0.2.5 起的 Claude agent 支持 `auto` 权限模式（用模型分类器自动放行 permission prompt）。但 SDK 层（`@anthropic-ai/claude-agent-sdk`，路径在 App.asar 内）对 `auto` 模式有**模型白名单**：只有当前一代模型（如 `claude-sonnet-4-6`、`claude-opus-4-6`）能开。

旧会话的 agent 状态文件（`~/.paseo/agents/<workspace>/<agentId>.json`）里固化了 `claude-sonnet-4`（老一代 ID）。运行中切 `auto` 会硬报错：

```
Cannot set permission mode to auto: auto mode unavailable for this model
```

错误从 SDK 的 `set_agent_mode` handler 抛出，Paseo 只把它透传。daemon.log 里搜 `set_agent_mode_request error` 能看到完整 stack。**这不是 daemon bug，重启 daemon 无用，还会杀掉所有运行中的 agent。**

**选项**
1. 等官方放白名单（被动，没法用）
2. 删掉老 agent 重开（丢失上下文）
3. 在中转层（自建 Claude API 代理）把 `claude-sonnet-4` alias 到 `claude-sonnet-4-6`，并让 Paseo 端把 agent model 改成 `claude-sonnet-4-6`
4. 全程不用 `auto`，回退 `acceptEdits` / `default`（功能降级）

**决定**
选 3。两层配合：
- **中转代理**：把旧 model ID 转发到当前一代模型（具体 alias 在你自己的中转层配置，不在 Paseo 这边动）。
- **Paseo 端**：用 `update_agent` 把 agent 的 `settings.model` 改成 `claude-sonnet-4-6`，然后再切 `modeId: "auto"`。新建 agent 时直接用 `provider: "claude/sonnet"`（Paseo 会自动解析到当前一代），不要硬写 model 字符串。

**理由**
- 保留老 agent 的会话上下文（不用重建）。
- 别人的 Claude API 客户端（非 Paseo）也能从 alias 受益，集中在中转层做一次即可。
- SDK 白名单是硬约束，绕不过；alias 是把 SDK 看到的 model ID 改成白名单内 ID，根治。

**代价 / 注意**
- ⚠️ 中转层的 alias 是**全局**的，其他用到 `claude-sonnet-4` 字符串的客户端都会被改写。需要确认下游没有依赖旧 ID 做路由判断的逻辑。
- ⚠️ `auto` 模式不是 "allow all"——它用分类器判断每个 permission prompt。对 `/tmp` 写入、跨工作区操作这类边界动作仍会弹 permission request（实测验证）。如果想要真正零打断，用 `bypassPermissions`（风险自担）。
- ⚠️ SDK 白名单随版本演进。未来再发新一代（如 `claude-sonnet-5-x`）时，`claude-sonnet-4-6` 也会变旧，需要再调 alias。

**验证**
- 运行中切：`update_agent({ agentId, settings: { model: "claude-sonnet-4-6" } })` → `set_agent_mode({ agentId, modeId: "auto" })`，daemon.log 无 `set_agent_mode_request error`。
- 新建：`create_agent({ provider: "claude/sonnet", settings: { modeId: "auto" }, ... })`，返回 `currentModeId: "auto"` 且无 permission 错误。

**回滚**
- 把 agent 的 `settings.model` 改回旧 ID + 切回 `acceptEdits` / `default`。
- 中转层 alias 撤掉。

---

## 决策变更流程

改这些决定时:
1. 在本文件追加新 ADR(如 ADR-0008),不要改旧的,旧的标 `Superseded by ADR-000X`。
2. 同步更新 `linux-ops.md` 的命令和 `architecture.md` 的拓扑。
3. 涉及 systemd 单元的改动,用验证命令确认生效(见 OPS 末尾自检脚本)。
