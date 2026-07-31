# Paseo 使用手册

> 面向**使用者**（手机端 / web UI 端用户）。怎么连、怎么发任务、怎么管 agent。
> 运维和踩坑看 [`PASEO_OPS.md`](./PASEO_OPS.md)，原理看 [`PASEO_ARCHITECTURE.md`](./PASEO_ARCHITECTURE.md)。

---

## 一、访问入口

这台机器上的 paseo daemon 对外有两个入口，用哪个都行：

| 入口 | 地址 | 适用场景 |
|---|---|---|
| **公网直连** | `http://<SERVER_IP>:<PORT>` | 跨网时抖动小（省一跳 relay）；浏览器 web UI |
| **中继(relay)** | 手机 app 扫 `paseo daemon pair` 的配对码 | 无需记 IP，走 `relay.paseo.sh:443` 兜底 |

**web UI（浏览器）**：直接打开 `http://<SERVER_IP>:<PORT>/`，用 daemon 密码登录（见下）。

**手机 app**：在 app 里加一个 daemon，扫码或手填直连地址。

---

## 二、登录 / 密码

- daemon 设了密码（存在 `config.json` 的 `auth.password`，bcrypt 哈希）。
- web UI 首次打开会要求输入密码。
- CLI 操作本机 daemon 时也要带密码：
  ```bash
  export PASEO_PASSWORD='<你的密码>'   # 写进当前 shell，之后 paseo 命令不再问
  paseo ls                              # 否则会报 requires a password
  ```
- 想改密码：`paseo daemon set-password`（会重写 `config.json` 的 `auth.password`），改完 `systemctl restart paseo`。

---

## 三、配对新设备

```bash
paseo daemon pair            # 打印 QR 码 + 配对链接
paseo daemon pair --json     # 机器可读，含 relay URL
```

手机 app 用扫码或点链接完成配对。配对信息里含 relay endpoint（`relay.paseo.sh:443`），所以**手机不需要和服务器在同一网络**。

---

## 四、常用 CLI 命令

先 `export PASEO_PASSWORD=...`，否则下面所有命令都会 401。

### 列 / 创建 / 看 agent

```bash
paseo ls                       # 列出 agent（默认排除归档的）
paseo run "帮我看下 nginx 日志"  # 创建并启动一个 agent
paseo inspect <id>             # 看某个 agent 的详细信息
paseo logs <id>                # 看 agent 的活动时间线
paseo attach <id>              # 实时跟随 agent 的输出流（类似 tail -f）
```

### 控制 agent

```bash
paseo send <id> "再看一下内存"   # 给已存在的 agent 追加消息/任务
paseo stop <id>                 # 中断正在跑的 agent（空闲的 no-op）
paseo wait <id>                 # 阻塞等 agent 变 idle
paseo archive <id>              # 软删除（归档）
paseo delete <id>               # 硬删除（先中断再删）
```

### 杂项

```bash
paseo status                    # 本地 daemon 状态（端口、PID、relay）
paseo clone <github-repo>       # 克隆 repo 并注册为 paseo workspace
paseo import <id>               # 把已有 provider session 导入为 paseo agent
```

---

## 五、CLI 输出格式

默认是 table。需要脚本处理时切格式：

```bash
paseo ls --json                 # JSON
paseo ls -o yaml                # YAML
paseo ls -q                     # 只输出 ID（脚本友好）
paseo ls --no-headers           # 去掉表头
```

---

## 六、连不上的快速排查

1. **先看 daemon 活没活**（在你登录服务器后）：
   ```bash
   systemctl is-active paseo
   ss -ltn | grep 8767
   ```
2. **看密码对不对**：CLI 报 `requires a password` / web UI 一直 401 = 密码错或 `PASEO_PASSWORD` 没设。
3. **看云防火墙/安全组**：确认对应端口的入站规则已放行（具体命令依云厂商而定）。
4. **更深入的链路/模型排查**：见 [`PASEO_OPS.md` 坑 #7](./PASEO_OPS.md)。

国内手机移动数据下抖动是地理决定的（GFW + 太平洋），不是配置问题——见 [OPS 坑 #8](./PASEO_OPS.md)。

---

## 七、不要做的事

- **别手动 `paseo daemon start`**：这台机器已交给 systemd，手动起会丢 `IS_SANDBOX=1` 环境变量，手机端 claude 会报 root 拦截。统一走 `systemctl {start,stop,restart} paseo`。详见 [OPS 坑 #3](./PASEO_OPS.md)。
- **别把 8767 暴露成无密码**：daemon 派生的 claude 是 root 身份，谁拿到这个端口就等于拿到 root shell。密码必须强。
- **别改端口忘了同步防火墙**：云防火墙/安全组按端口匹配，daemon 换端口必须同步改入站规则（见 [OPS 坑 #4](./PASEO_OPS.md)）。
