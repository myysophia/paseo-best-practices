# Paseo 运维手册 & 踩坑记录

> 本文档记录 paseo daemon 在 Linux 服务器上的部署方式、运维命令，以及调试过程中踩过的所有坑。**改 paseo 之前先读这个。**

---

## 一、核心架构(先理解,再操作)

```
手机 app ──(配对)──> paseo daemon(本机:<PORT>)── 派生 ──> claude 进程
                              │
                              └── 同时维护一条出站 wss ──> relay.paseo.sh 作兜底
```

**关键事实:手机上跑的 claude 进程,环境变量是从 paseo daemon 继承的,不读 `.bashrc`。** 这是大多数坑的根源。

---

## 二、日常运维:只用 systemd,别手动

paseo 已托管给 systemd。**永远用这两组命令,不要手动 `paseo daemon start/restart`:**

```bash
systemctl status paseo        # 看状态
systemctl restart paseo       # 重启
systemctl stop paseo          # 停
systemctl start paseo         # 起
journalctl -u paseo -f        # 实时日志
tail -f ~/.paseo/daemon.log   # 更细的文件日志
```

涉及文件:
- `/etc/systemd/system/paseo.service` —— systemd 单元(前台 `--foreground` 跑)
- `~/.paseo/paseo.env` —— **环境变量注入文件(600)**,内容:
  - `IS_SANDBOX=1` ← 命根子,详见坑 #1
  - `ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` / 各默认模型
- `~/.paseo/config.json` —— paseo 配置:`listen` 端口、`relay.enabled`、`auth.password` 等
- `~/.claude/settings.json` —— claude 自己读的 env 块(凭据 + 模型映射)

改完配置:`systemctl restart paseo`。

---

## 三、踩坑记录(按时间/严重度)

### 坑 #1 —— `--dangerously-skip-permissions cannot be used with root/sudo`
**现象:** 手机连上 paseo,claude 一启动就报这个错,无法用。

**根因链(绕了大弯才定位):**
1. 服务器上跑 claude 是 **root** 身份。
2. claude 源码里有一条安全检查:`getuid()===0 && IS_SANDBOX!=="1" && CLAUDE_CODE_BUBBLEWRAP!=="1"` → root + 跳权限 → 拒绝。
3. 手机上的 claude 是 **paseo daemon 派生的**,环境继承自 daemon 进程,**不读 `.bashrc`**。
4. 所以往 `~/.bashrc` 写 `export IS_SANDBOX=1`、`source` 多少次都**完全无效**——paseo 拉终端时不走 bash。

**正确解法:** 把 `IS_SANDBOX=1` 注入到 **paseo daemon 的启动环境**:
- 临时:`IS_SANDBOX=1 paseo daemon start`(但手动启会埋别的坑,见 #3)
- 持久(推荐):写进 `~/.paseo/paseo.env`,由 systemd 的 `EnvironmentFile` 注入。

**验证方法:** 查实际 daemon 进程的环境,不要看 `.bashrc`:
```bash
PORT=$(node -e "console.log(require('~/.paseo/config.json').daemon.listen.split(':').pop())" 2>/dev/null || echo 8767)
DPID=$(ss -ltnp | grep :$PORT | grep -oE "pid=[0-9]+" | cut -d= -f2)
tr '\0' '\n' < /proc/$DPID/environ | grep IS_SANDBOX   # 必须有 IS_SANDBOX=1
```

---

### 坑 #2 —— `paseo restart` 不一定能带上新环境变量
**现象:** 设了 `IS_SANDBOX=1` 再 `paseo restart`,手机还是报坑 #1 的错。

**根因:** paseo 的进程结构是 `Paseo Supervisor → Paseo Daemon`。`paseo restart` 让 **Supervisor** 重新拉起 daemon,**新 daemon 继承的是 Supervisor 的旧环境**(没有 IS_SANDBOX)。Supervisor 是常驻的,它的环境在它启动那一刻就定了。

**正确解法:** 把 Supervisor + Daemon **一起停掉**,重新启动,且启动时带上变量:
```bash
paseo daemon stop --force        # 会把 supervisor + daemon 都停掉
IS_SANDBOX=1 paseo daemon start  # 或交给 systemd(推荐)
```

---

### 坑 #3 —— 手动启动的 daemon 会丢 IS_SANDBOX(最常见复发原因)
**现象:** 端口改完、明明配好了,手机又报坑 #1。

**根因:** 改端口时 daemon 被**手动 `paseo daemon start`** 重启过,手动启动不经过 systemd → 不读 `paseo.env` → 没有 `IS_SANDBOX=1`。同时这个手动 daemon 占着端口,systemd 那个起不来(`Another Paseo daemon is already running`)。

**正确解法:**
1. 杀掉手动 daemon:`kill -TERM <supervisor_pid>`(先 `ps -eo pid,ppid,cmd | grep Paseo` 找)。
2. `systemctl reset-failed paseo && systemctl start paseo`。
3. **以后永远只走 systemd,别手动起。**

**自检:** 只要手机又报 root 拦截,99% 是又被手动启动过。第一时间查 daemon 进程的 IS_SANDBOX 计数(见坑 #1 验证方法)。

---

### 坑 #4 —— 改端口 / 开公网的安全顺序
**改端口:** 改 `config.json` 的 `daemon.listen` → `systemctl restart paseo` → 确认 `ss -ltn | grep <port>`。

**开公网(直连手机)安全顺序,别反:**
1. **先设密码**:`paseo daemon set-password`(写进 `config.json` 的 `auth.password`,bcrypt)。
2. **重启**让它加载密码:`systemctl restart paseo`。
3. **最后才开云防火墙**。顺序反了 = 把一个未认证的 root 入口敞在公网。
4. 改了 daemon 端口,**必须同步更新云防火墙/安全组的入站规则**——防火墙规则按端口匹配,旧端口规则对新端口无效。

---

### 坑 #5 —— systemd `Restart=on-failure` 不救「干净退出」
**现象:** paseo 被 SIGTERM 干净停掉(`status=0/SUCCESS`)后没自动起来,手机连不上。

**根因:** `Restart=on-failure` 只在**失败退出**(非 0)时重启。SIGTERM 是干净退出(status=0),不触发重启。

**应对:** 手动 `systemctl start paseo`。如需更皮实,可把单元里的 `Restart=` 改成 `always`(但要小心崩溃循环)。

---

### 坑 #6 —— 「卡住」先分清是链路还是模型
**现象:** 手机上 claude 命令执行一半卡住。

**排查(从快到慢):**
1. 看 daemon 是否还活着:`systemctl is-active paseo`、`ss -ltn | grep <port>`。
2. 查 claude 进程卡在哪:
   ```bash
   CLPID=$(ps -eo pid,cmd | grep "claude --output-format" | grep -v grep | awk '{print $1}' | head -1)
   ss -tnp | grep "pid=$CLPID"          # 看它连到哪
   cat /proc/$CLPID/wchan                # ep_poll = 在等网络
   ```
   如果有一条 `ESTAB → <模型代理地址>` 且 wchan 是 `ep_poll` = **卡在等模型响应**,和 paseo 链路无关。
3. 直接测模型代理:
   ```bash
   curl -v --max-time 15 $ANTHROPIC_BASE_URL/v1/messages \
     -H "x-api-key: $ANTHROPIC_AUTH_TOKEN" -H "anthropic-version: 2023-06-01" \
     -H "content-type: application/json" \
     -d '{"model":"<模型名>","max_tokens":16,"messages":[{"role":"user","content":"hi"}]}'
   ```
   超时/404/错误 = 代理或模型的问题。

---

### 坑 #7 —— 国内手机连不稳的客观现实
relay 链路是 `国内手机 → (GFW+太平洋) → relay(海外) → 服务器`。改直连后变成 `手机 → (GFW+太平洋) → 服务器`,省掉中间那一跳,但 **GFW 那段还在**,国内移动数据下仍会抖,这是地理决定的,不是配置问题。直连地址:在手机 Direct connect tab 填 `<服务器公网IP>:<端口>`。

---

## 四、配置模板(套到你自己的环境)

| 项 | 值 |
|---|---|
| 服务器 | 你自己的服务器(公网 IP `<SERVER_IP>`) |
| 监听端口 | 自选(如 `8767`,避开浏览器受限端口如 6666/6667) |
| 云防火墙 | 入站放行该端口的 TCP,**仅限需要的来源 CIDR** |
| systemd 服务 | `paseo.service`(enabled,开机自启) |
| 密码 | 强制必设(`config.json: auth.password`,bcrypt) |
| 中继 | 启用,作兜底 |
| IS_SANDBOX | 由 `~/.paseo/paseo.env` 经 systemd `EnvironmentFile` 注入 |

## 五、一键自检脚本

怀疑手机连不上时,跑这个(把 `PORT` 换成你的端口):
```bash
PORT=8767
echo "服务: $(systemctl is-active paseo)"
ss -ltnp 2>/dev/null | grep :$PORT
DPID=$(ss -ltnp 2>/dev/null | grep :$PORT | grep -oE "pid=[0-9]+" | cut -d= -f2)
[ -n "$DPID" ] && echo "daemon pid=$DPID, IS_SANDBOX=$(tr '\0' '\n' < /proc/$DPID/environ | grep -c '^IS_SANDBOX=1')"
[ -n "$DPID" ] && [ "$(tr '\0' '\n' < /proc/$DPID/environ | grep -c '^IS_SANDBOX=1')" != "1" ] && echo "⚠️ IS_SANDBOX 丢了 → 多半被手动启动了"
```
