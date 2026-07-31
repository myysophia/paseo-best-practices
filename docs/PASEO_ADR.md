# ADR: Paseo 部署决策记录

> Architecture Decision Records。每条记录一个不可逆(或难逆)的工程决策:为什么这么决定、否了什么、代价是什么。改这些决定前先读对应 ADR。
>
> ADR 索引
> - [ADR-0001 用 systemd 托管 paseo,而非手动/脚本启动](#adr-0001-用-systemd-托管-paseo)
> - [ADR-0002 用 EnvironmentFile 注入 IS_SANDBOX,而非写 .bashrc](#adr-0002-用-environmentfile-注入-is_sandbox)
> - [ADR-0003 开放公网直连,而非只走 relay](#adr-0003-开放公网直连而非只走-relay)
> - [ADR-0004 relay 保留启用,与直连并存](#adr-0004-relay-保留启用与直连并存)

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

- **状态:** Accepted
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

## 决策变更流程

改这些决定时:
1. 在本文件追加新 ADR(如 ADR-0005),不要改旧的,旧的标 `Superseded by ADR-000X`。
2. 同步更新 `PASEO_OPS.md` 的命令和 `PASEO_ARCHITECTURE.md` 的拓扑。
3. 涉及 systemd 单元的改动,用验证命令确认生效(见 OPS 末尾自检脚本)。
