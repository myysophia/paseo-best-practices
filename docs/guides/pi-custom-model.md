# Pi 自定义模型配置最佳实践（Paseo 集成）

> 在 Paseo 中使用 Pi provider + 中转站自定义模型（非官方 API endpoint）的完整配置流程、验证方法与踩坑记录。
>
> 适用场景：通过 Anthropic 协议兼容中转（如 claude-code-proxy 类服务）接入 Claude / GPT / GLM / DeepSeek 等多厂商模型，统一在 Paseo 手机端 / 桌面端驱动。

---

## 一、核心结论

1. **Pi 的自定义模型完全可以在 Paseo 下使用。** Paseo 原生适配 Pi（`pi --mode rpc` 子进程），Pi 读自己的 `~/.pi/agent/models.json`，Paseo 不干预模型定义，模型列表会原样透传到 Paseo 的模型选择器。
2. **配置入口只有一个：`~/.pi/agent/models.json`**（daemon 所在机器上）。改完立即生效，无需重启 daemon（Pi 进程按会话启动）。
3. **中转站域名会失效**——本文最大的坑。全球 DNS 都查不到的域名，本地怎么配都白搭，先做连通性探测再写配置。

---

## 二、配置文件结构与字段说明

`~/.pi/agent/models.json`：

```json
{
  "providers": {
    "anthropic": {
      "baseUrl": "https://cpa.jxcq.work",
      "api": "anthropic-messages",
      "apiKey": "!jq -er '.env.ANTHROPIC_AUTH_TOKEN' /Users/ninesun/.claude/settings.json",
      "models": [
        {
          "id": "z-ai/glm-5.3-flash",
          "name": "GLM 5.3 Flash",
          "reasoning": true,
          "input": ["text", "image"],
          "contextWindow": 1000000,
          "maxTokens": 128000
        }
      ]
    }
  }
}
```

| 字段 | 说明 | 坑 |
|---|---|---|
| `baseUrl` | 中转站地址，Pi 向其 `/v1/messages` 发 Anthropic messages 协议请求 | 域名失效是静默的，旧配置不会报错提醒 |
| `api` | 协议类型，中转走 `anthropic-messages` | — |
| `apiKey` | 支持 `!command` 形式动态提取（shell 命令前缀 `!`），避免 key 明文落盘 | 命令必须可被 daemon 环境执行（PATH 内） |
| `id` | **中转站按此字符串路由**，必须与中转站侧注册名完全一致 | 带斜杠的 id（如 `z-ai/glm-5.3-flash`）合法 |
| `contextWindow` / `maxTokens` | 影响上下文压缩触发时机与请求上限 | 虚报 1M 窗口而后端不支持时，长会话可能报错 |
| `reasoning` / `input` | 能力声明，控制 thinking 档位与多模态 | — |

**同一 provider 可挂任意厂商模型**：GPT/GLM/DeepSeek 都以 `anthropic-messages` 协议借道中转，对 Pi 而言都是"anthropic provider 下的模型 id"。

### 关联文件

- `~/.pi/agent/settings.json`：`defaultProvider` + `defaultModel` 决定 Pi 默认模型。**改完 models.json 记得核对 defaultModel 指向的 id 仍存在**，否则新会话起不来或回落异常。
- `~/.pi/agent/models-store.json`：Pi 缓存的官方模型目录（自动从 api.anthropic.com 拉取），**不要手动编辑**，与自定义 models.json 是两回事。
- 备份惯例：改前 `cp models.json models.json.bak.<日期>-<标签>`。

---

## 三、标准操作流程（SOP）

### Step 1：连通性探测（写配置之前）

用 curl 直接打中转站，确认域名解析 + key 有效 + 目标模型路由可达：

```bash
curl -s --max-time 20 -o /tmp/probe.json -w "%{http_code}\n" \
  -X POST "https://<中转域名>/v1/messages" \
  -H "content-type: application/json" \
  -H "x-api-key: $(jq -er '.env.ANTHROPIC_AUTH_TOKEN' ~/.claude/settings.json)" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model":"<模型id>","max_tokens":16,"messages":[{"role":"user","content":"say ok"}]}'
```

- HTTP 200 = 通；逐个模型探测一遍（中转站按 id 路由，A 通不代表 B 通）。
- **HTTP 000 + `Could not resolve host`** → 域名问题，进入"坑 #1"排查。

### Step 2：写入 models.json

按上节结构写入，改前备份。

### Step 3：核对 settings.json

`defaultModel` 指向实际存在的 id。

### Step 4：Pi 本体冒烟（绕开 Paseo 先验证 Pi 层）

```bash
cd /tmp && echo 'say ok' | pi --print --model "<模型id>" --no-session
```

输出 `ok` 即 Pi → 中转 → 模型 全链路通。

### Step 5：Paseo 端到端验证

```bash
paseo run --provider pi --model "z-ai/glm-5.3-flash" \
  --title "pi-custom-model-verify" --new-workspace local -d \
  "只回复两个字：收到。不要使用任何工具。"

# 稍等片刻查看输出
paseo logs <AGENT_ID>
```

模型选择器中的模型名来自 `name` 字段；`--model` 参数用 `id`。

### Step 6：清理验证用 agent

验证完的临时 agent / workspace 在 UI 或 CLI 里删掉，避免列表堆积。

---

## 四、踩坑记录

### 坑 #1：中转站域名全球 DNS 失效（本篇核心坑）

**现象**：所有模型 curl 探测 HTTP 000，`Could not resolve host`。

**排查路径**（按顺序）：

1. `nslookup <域名>` —— 本机 DNS 查不到 ≠ 域名没了，可能是内网 DNS 污染
2. 走系统代理查公共 DoH：
   ```bash
   curl -s -x http://127.0.0.1:7897 "https://cloudflare-dns.com/dns-query?name=<域名>&type=A" -H "accept: application/dns-json"
   ```
3. **判据**：DoH 返回 `"Status":3`（NXDOMAIN）且 Authority 段是权威 NS 的 SOA → 该记录在权威 DNS 上就不存在，不是本地问题。

**根因**：中转站迁移/更换域名后，旧域名 DNS 记录被删。`models.json` 里的旧 `baseUrl` 不会自动提醒。

**解法**：从中转站提供方拿到新域名，全局替换 `baseUrl`。

**经验**：中转域名不稳定时，探测脚本（Step 1）应作为日常健康检查；域名更换属于"静默故障"，Pi 只会在会话中报请求错误，不会告诉你"域名失效了"。

### 坑 #2：本机代理对 TLS 的干扰

macOS 系统代理（127.0.0.1:7897）CONNECT 隧道能建立，但 `SSL_ERROR_SYSCALL`（TLS 握手被重置）——代理规则可能对该域名走 DIRECT 而本地网络又不通，或代理节点问题。**curl 默认不读系统代理**（只认 env），行为与 GUI 应用不一致，排查时注意区分测试路径。

### 坑 #3：defaultModel 悬空

`settings.json` 的 `defaultModel: "claude-sonnet-4-5"` 在 models.json 里已无此 id（模型列表更新后遗留）。每次改 models.json 后核对默认模型指向，或干脆在 Paseo UI 里手动选模型。

### 坑 #4：模型条目名实不符的遗留脏数据

旧配置里存在 `id: "claude-fable-5-dd-5.4-korg"`、`name: "Grok 4.5（中转）"` 的调试遗留条目——中转站按 id 路由，name 只是显示名，实际打到什么模型只有中转站知道。**配置时要逐个用 Step 1 的探测确认真实路由**（返回体里的 `model` 字段是后端真实模型名，如探测 `claude-sonnet-4-6` 返回 `"model":"glm-5.3"` 说明中转做了映射）。

### 坑 #5：`timeout` 命令在 macOS zsh 缺失

`timeout 60 pi ...` 在 macOS 上 `command not found`（GNU coreutils 命令）。用 pi 自带参数（`--no-session`）+ Bash 工具的 timeout 参数替代，或 `brew install coreutils` 后用 `gtimeout`。

---

## 五、验证清单（Checklist）

配置完成后逐项打勾：

- [ ] 所有目标模型 curl 探测 HTTP 200（返回体 `model` 字段符合预期路由）
- [ ] `models.json` 已备份，`baseUrl` 为当前有效域名
- [ ] `settings.json` 的 `defaultModel` 指向 models.json 中存在的 id
- [ ] `pi --print --model <id>` 冒烟通过（每个模型至少一次）
- [ ] `paseo run --provider pi --model <id>` 端到端返回正常
- [ ] Paseo UI 模型选择器可见自定义模型名
- [ ] 验证用 agent/workspace 已清理

---

## 六、与 Paseo 其他文档的关系

- Pi 在 Paseo 下的进程模型、extension 生态（pi-mcp-adapter / pi-web-access 等）见 [architecture.md](../internals/architecture.md)
- daemon 环境继承（apiKey 的 `!command` 为何要关注 PATH）见 [architecture.md](../internals/architecture.md) 环境继承章节
- auto 模式模型白名单（模型 id 与权限模式的校验关系）见 [adr.md ADR-0007](../adr/adr.md)

---

## 附：当前配置快照（2026-08-31）

| 项 | 值 |
|---|---|
| 中转站 | `https://cpa.jxcq.work`（旧 `cpa.imdev.work` 已失效，见坑 #1） |
| 协议 | anthropic-messages |
| key 来源 | `~/.claude/settings.json` 的 `ANTHROPIC_AUTH_TOKEN`（jq 动态提取） |
| 模型数 | 8（3 Claude + 3 GPT-5.6 + GLM-5.3-flash + DeepSeek-V4-pro） |
| 默认模型 | `claude-sonnet-4-6`（1M 窗口） |
| 备份 | `~/.pi/agent/models.json.bak.20260831-newrelay` |
| 验证状态 | curl 6/6 通过；pi print 抽测 4/4 通过；paseo run 端到端 2/2 通过 |
