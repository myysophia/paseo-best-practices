# Paseo 直连(Direct Connect)最佳实践

> 「添加主机」时选 Direct connect 该怎么配、为什么、风险是什么。
> 配套阅读:[`PASEO_OPS.md`](./PASEO_OPS.md)(端口/密码/防火墙操作)、[`PASEO_USAGE.md`](./PASEO_USAGE.md)(CLI 用法)。

---

## 一、为什么直连需要额外注意

Paseo daemon 既能通过 **relay**(中继)走,也能通过 **direct connect**(直连)走:

| 模式 | 加密 | 开公网端口 | 链路 |
|---|---|---|---|
| **Relay** | ✅ 端到端加密(Curve25519 + XSalsa20-Poly1305),relay 无法解密 | ❌ daemon 只出站 | 手机 → relay(海外)→ 服务器,多一跳 |
| **Direct** | ❌ **默认明文 HTTP/WS** | ✅ 需开入站端口 | 手机 → 服务器,少一跳 |

直连省一跳、延迟低,但 daemon 监听端口暴露在网络里,**默认流量是明文**。密码只做"认证"(防未授权访问),**不加密流量**(防窃听)。

---

## 二、抓包验证:明文到底暴露了什么

> 实测证据,不是推测。在本机 lo 上抓一个带标记的 curl 请求。

### 测试请求

```bash
curl -H "X-Probe-Secret: SUPER_SECRET_VALUE_12345" \
     -H "X-Probe-Prompt: 帮我删掉数据库里的 users 表" \
     --data '{"probe":"SENSITIVE_PAYLOAD_99999","password":"hunter2"}' \
     "http://127.0.0.1:8767/api/health?probe=URL_QUERY_SECRET"
```

### 抓到的包(tcpdump ASCII dump)

```
POST /api/health?probe=URL_QUERY_SECRET HTTP/1.1        ← URL query 明文
Host: 127.0.0.1:8767
X-Probe-Secret: SUPER_SECRET_VALUE_12345                 ← header 明文
X-Probe-Prompt: ... users ...                            ← header 内容明文
{"probe":"SENSITIVE_PAYLOAD_99999","password":"hunter2"} ← body 明文

HTTP/1.1 404 Not Found                                   ← 响应也明文
```

**所有标记字符串都 1:1 出现在 pcap 里**——URL、header、JSON body、响应,无一例外。

### 真实场景里这意味着什么

Paseo daemon 的 API 流量包含这些字段(出现频率,来自代码扫描):

| 字段 | 出现次数 | 含义 |
|---|---|---|
| `message` | 615 | 你发给 agent 的每一条消息 |
| `command` | 491 | agent 跑的 shell 命令 / 工具调用 |
| `cwd` | 185 | agent 的工作目录(暴露服务器目录结构) |
| `prompt` | 114 | 任务提示词 |
| `password` | 26 | 各类密码字段 |
| `authorization` | 20 | 认证头(含 `Bearer <password>`) |

**结论:在 `0.0.0.0` + 密码模式下,任何能在网络路径上抓包的人都能读到——**

- 你给 agent 发的每一句任务(包括敏感指令、代码片段、密钥)
- agent 跑的每一条 shell 命令(`rm`、`git push`、`docker`...)
- 工作目录路径(暴露服务器文件结构)
- `Authorization: Bearer <密码>` 头本身(密码虽是 bcrypt 哈希存盘,但传输时是明文 bearer token)

密码只挡住"连不进来的人",挡不住"在线缆上偷听的人"。

---

## 三、三层防御(官方文档推荐)

直连的安全是三层叠加,**任何一层单独都不够**:

### 第一层:密码认证(必备,但只是开始)

```bash
paseo daemon set-password          # 写 bcrypt 哈希到 config.json
# 或:export PASEO_PASSWORD='<强密码>'  (Docker / systemd env)
systemctl restart paseo
```

- 行为:未带 `Authorization: Bearer <密码>` 的请求 → `401`;`/api/health` 例外(给负载均衡探活用);web UI 静态页可匿名加载,但 API/WS 仍需密码。
- **不加密流量**,只防未授权访问。
- CLI 加 host 时要带同一个密码:
  ```bash
  PASEO_PASSWORD='<密码>' paseo --host <IP>:<port> ls
  ```

### 第二层:hostnames 允许列表(防 DNS 重绑定)

config.json:
```json
{
  "daemon": {
    "hostnames": ["your-hostname", ".your-domain.com"]
  }
}
```

| 值 | 行为 |
|---|---|
| `[]`(默认) | 允许 localhost、`*.localhost`、所有 IP |
| `["hostname",".domain.com"]` | 允许指定主机/域名 + 默认值 |
| `true` | 允许任意(**不推荐**) |

Docker 用 `PASEO_HOSTNAMES`。**绑 `0.0.0.0` 时必须设这个**,防止伪造 Host 头绕过浏览器同源策略。

### 第三层:网络层(防火墙 / 绑定地址)

按安全度从高到低选绑定方式:

| 绑定 | 场景 | 风险 |
|---|---|---|
| **Unix socket** | 纯 CLI,不联网 | 最低 |
| `127.0.0.1:port` | 本机 / 反代后端 | 低 |
| **Tailscale IP**(`100.x.y.z`) | 手机+服务器同 tailnet | **低(推荐)** |
| `0.0.0.0:port` | 公网直连 | **高,必须密码+防火墙+hostnames** |

防火墙源 CIDR 优先级:手机出口 IP `/32`(最窄)> 运营商网段 > `0.0.0.0/0`(最宽,风险最高)。

---

## 四、加密流量的三种方案(选一个)

直连要加密,得在 daemon 前面加一层。官方推荐程度从高到低:

### 方案 A:Tailscale(官方最推荐)

**唯一能让"直连 + 加密 + 不开公网端口"三者兼得的方案。**

1. 服务器和手机都装 Tailscale,加入同一 tailnet。
2. daemon 改绑 Tailscale IP:
   ```json
   { "daemon": { "listen": "100.x.y.z:8767" } }
   ```
3. 把 Tailscale hostname 加到 `daemon.hostnames` 和 `cors.allowedOrigins`。
4. 手机 app 加 host 时填 `100.x.y.z:8767`。
5. **关掉**云防火墙的公网 8767 入站——daemon 只在 tailnet 内可达。

流量走 WireGuard 加密,daemon 不暴露公网,GFW 抖动也改善(走 Tailscale 的 DERP 中继兜底)。

### 方案 B:反向代理 + HTTPS(Caddy/Nginx)

适合需要用域名 / 浏览器访问 web UI 的场景。

```caddyfile
paseo.your-domain.com {
  reverse_proxy 127.0.0.1:8767
}
```

- daemon 绑 `127.0.0.1:8767`(只本机),反代负责 TLS。
- `cors.allowedOrigins` 加白 `https://paseo.your-domain.com`。
- `daemon.hostnames` 加 `paseo.your-domain.com`。
- 云防火墙只开 443,不开 8767。

### 方案 C:纯 relay(放弃直连)

最省事:不开公网端口,daemon 只出站连 relay,手机也连 relay。代价是多一跳海外 relay,国内链路可能抖。如果直连的主要目的是降延迟,这个方案等于放弃直连;如果直连的主要目的是"避免依赖海外 relay",选 A 或 B。

### 方案 D:双入口并存(`0.0.0.0` + SG 分层 + tailnet)

适合「跨网客户端要加密 + 办公网段设备要免 Tailscale」的混合场景。daemon 监听 `0.0.0.0:8767`(同时接到公网和 tailnet 流量),靠**云防火墙分层放行**控制来源:

| 来源 | 路径 | 加密 | SG 规则 |
|---|---|---|---|
| 手机 / 跨网客户端 | tailnet `<TS_IP>:8767` | ✅ WireGuard | 不经云防火墙(走虚拟网卡) |
| 办公网段设备 | 公网 `<SERVER_IP>:8767` | ❌ 明文 | office-sg `/30` 全端口 |

**关键约束:**
- bastion SG 必须**撤销 `0.0.0.0/0`**,只留办公网段 `/30` 规则——否则 daemon 全网暴露。
- 手机配 host 时**必须填 tailnet IP**(`100.x.y.z`),填了公网 IP 就退化成明文。
- 办公网段内若担心嗅探,设备也可改装 Tailscale 走 tailnet。

详见 [ADR-0006](./PASEO_ADR.md)。这是当前部署采用的模式。

---

## 五、信任模型

- **配对 QR 码 / 链接 = 信任锚**,含 daemon 公钥,**当密码一样保管**。
- 泄露后:`paseo daemon pair` 重新生成 + `systemctl restart paseo`,轮换 session ID 和 relay pairing。
- 加 host 时在 web UI / 手机 app / CLI **三处用同一个 `PASEO_PASSWORD`**。

---

## 六、决策清单

加 host 选直连前,过一遍这个清单:

- [ ] 已 `paseo daemon set-password`,密码是长随机串
- [ ] daemon 已绑密码且重启加载(`journalctl -u paseo | grep authentication`)
- [ ] `daemon.hostnames` 已设(不是默认 `[]` 或 `true`)
- [ ] 防火墙源 CIDR 已收窄到最小范围
- [ ] **理解密码不加密流量**,确认网络路径可信,或已上 TLS / Tailscale
- [ ] `cors.allowedOrigins` 只含需要的来源(浏览器场景)
- [ ] 配对 QR 没贴公开地方

**任何一项没打勾就不要用 `0.0.0.0` 直连公网。** 默认走 relay 更安全。

---

## 七、参考

- [Security – Paseo Docs](https://paseo.sh/docs/security)
- [Configuration – Paseo Docs](https://paseo.sh/docs/configuration)
- [SECURITY.md – getpaseo/paseo (GitHub)](https://github.com/getpaseo/paseo/blob/main/SECURITY.md)
- [Docker setup – getpaseo/paseo (GitHub)](https://github.com/getpaseo/paseo/blob/main/public-docs/docker.md)
