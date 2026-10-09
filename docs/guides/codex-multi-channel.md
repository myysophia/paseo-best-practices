# Codex 多通道认证与路由最佳实践（CLI / ChatGPT 客户端 / Paseo）

> 在 Paseo 中让 Codex 同时使用 **ChatGPT Pro 订阅**与**第三方中转站/API**两条通道的完整原理、配置流程与踩坑记录。
>
> 适用场景：有 ChatGPT 订阅但也被官方高峰限流卡过脖子，同时手握中转站按量通道，想让两条通道并存、按任务自由选择计费方式。

---

## 一、核心结论

1. **单个 Codex CLI 安装做不到多通道并存**：登录态（`~/.codex/auth.json`）一次只有一份，OAuth 与 API key 互相覆盖；全局路由（`config.toml` 的 `model_provider`）一次只有一条。
2. **Paseo 可以做到**：custom provider 的 `env` 注入机制让"路由 + 凭证"从全局唯一配置降级为**按 agent 会话注入**，不动 `auth.json`，全局订阅登录与 per-session 中转凭证互不干扰。
3. **ChatGPT 桌面客户端不能切换**：它是订阅产品的入口，认证跟随 App 内登录的账号，不暴露路由/端点配置——但它**共享 `~/.codex/config.toml`**，改这份文件会影响 App 内 Codex 功能的行为。
4. **"model not supported" 报错先查 CLI 版本再怀疑模型名**：旧版 CLI 不认识新发布的模型，升级后模型列表会变（实测 0.150.1 → 0.162.0，gpt-6 系列从"不认识"变成官方 `[list]` 模型）。

---

## 二、三者的配置关系（先理清拓扑）

```
~/.codex/  ←—— Codex 的"家"（CODEX_HOME），CLI 与 ChatGPT App 的 Codex 功能共享
├── config.toml         # 全局路由（model_provider）+ 默认模型 + MCP + [desktop] GUI 设置
├── auth.json           # CLI 的唯一登录态：ChatGPT OAuth tokens 或 API key（二选一）
└── models_cache.json   # 官方模型缓存（按 CLI 版本刷新，模型列表的权威来源）

~/.paseo/config.json    ←—— Paseo 的 provider 定义
├── 内置 codex provider  # 直接复用 ~/.codex/auth.json 的登录态（订阅通道）
└── custom provider（extends: "codex"）
    └── env: OPENAI_API_KEY (+ OPENAI_BASE_URL)
        → Paseo 为每个 agent 会话翻译成 per-session 的 model_providers 注入
        → 凭证从环境变量走，不读写 auth.json → 与全局登录态并存
```

| 维度 | Codex CLI | ChatGPT 桌面客户端 | Paseo |
|---|---|---|---|
| 二进制 | PATH 上的 `codex`（如 brew 装） | App 内嵌 CodexCLI（`/Applications/ChatGPT.app/.../codex-cli/`） | spawn PATH 上的 `codex`（与 CLI 同一个） |
| 认证 | `auth.json`（单一，OAuth ↔ API key 互斥） | App 内账号登录 | 订阅通道复用 `auth.json`；custom 通道用 env key |
| 路由配置 | `config.toml` 全局唯一 | 共享 `config.toml`，但不可自定义端点 | **per-provider / per-session 注入** |
| 模型列表 | `models_cache.json`（随版本变） | 跟随 App 版本 | `paseo provider models codex`（自动发现 + additionalModels 合并） |
| 多通道 | 不能并存（可 `-c` 临时切换） | 不能 | **能** |

**两个容易混淆的点：**

- **版本三元组**：brew CLI、ChatGPT App 内嵌 CodexCLI、Paseo 调用的 CLI 可能是三个不同版本。模型列表、行为随版本变——排错时先 `codex --version` 确认在跟谁说话。
- **config.toml 是共享的**：`[desktop]` 段（主题、通知）、MCP servers、skills 启停都在同一份文件里，ChatGPT App 与 CLI 都读写它。改路由注释时注意别误伤 App 依赖的段落。

---

## 三、为什么 Paseo 能自由切换（机制拆解）

本质：**Paseo 是进程编排层**。每个 agent 会话由 Paseo 独立 spawn 一个 codex 进程，它能完全控制该进程的环境变量与启动配置。

对每个 `extends: "codex"` 的 custom provider，Paseo 做两件事：

1. **透传环境变量**：把 `env` 里的 `OPENAI_API_KEY` / `OPENAI_BASE_URL` 传给 codex 进程；
2. **翻译成 Codex 原生配置**（注入 per-thread config）：

```toml
model_provider = "codex-relay"          # 本会话强制走该 provider

[model_providers.codex-relay]
base_url = "https://cpa.jxcq.work/v1"   # 取自 OPENAI_BASE_URL（自动补 /v1）
wire_api = "responses"
env_key = "OPENAI_API_KEY"              # 凭证从环境变量读
requires_openai_auth = false            # 关键：跳过 auth.json 登录流程
```

`requires_openai_auth = false` 是并存机制的钥匙：**Codex 只在走官方认证时才读 `auth.json`**。custom provider 的凭证走 `env_key` 指定的环境变量，`auth.json` 里的 ChatGPT OAuth 完全不被触碰——于是：

- 内置 `codex` provider → 走 `auth.json` → Pro 订阅
- `codex-relay` provider → 走 env key → 中转站按量

同一台机器、同一个 CLI、同时可用，按启动 agent 时的 provider 选择分流。

### 对比：CLI 原生的切换能力

| 方式 | 能力 | 限制 |
|---|---|---|
| `codex login` / `codex login --with-api-key` | 切换登录方式 | **互相覆盖**，不能并存 |
| 改 `config.toml` 的 `model_provider` | 切换端点 | 全局生效，影响所有会话；且切中转后 `auth.json` 需要是中转的 key（会顶掉订阅登录） |
| `codex exec -c model_provider=xxx -c 'model_providers.xxx={...}'` | 单次临时切换 | 每条命令都要带一长串参数 |
| `CODEX_HOME=~/.codex-relay codex ...` | 多套配置完全隔离 | 每套 home 要单独 login，UI/工具链不感知 |

CLI 能"切换"，不能"并存"；Paseo 把切换粒度细化到 per-session，才实现了并存。

---

## 四、双通道接入 SOP

### Step 0：确认现状

```bash
codex --version                                    # CLI 版本（决定认识的模型）
codex login status                                 # 当前登录方式
jq '{key: (.OPENAI_API_KEY != null), tokens: (.tokens != null), mode: .auth_mode}' ~/.codex/auth.json
grep -n "model_provider" ~/.codex/config.toml      # 有没有历史遗留路由
```

**登录前必备**：备份 `auth.json`（login 会覆盖）：

```bash
cp ~/.codex/auth.json ~/.codex/auth.json.bak-apikey-$(date +%Y%m%d)
```

### Step 1：登录订阅通道

```bash
codex login        # 浏览器 OAuth，登录 ChatGPT 账号
```

### Step 2：排查 config.toml 路由劫持（本篇坑 #1）

`model_provider = "custom"` 指向中转站时，所有请求（包括 OAuth）都被发到中转站。注释掉：

```toml
# model_provider = "custom"    # 切回中转请用 Paseo 的 codex-relay，不要恢复这行
```

**不要用恢复 `model_provider` 的方式切中转**：此时 `auth.json` 是 ChatGPT token，中转会拿它当 key 用，照样 401。

### Step 3：建中转通道 profile

`~/.paseo/config.json` 的 `agents.providers` 加：

```json
"codex-relay": {
  "extends": "codex",
  "label": "Codex (中转站)",
  "env": {
    "OPENAI_API_KEY": "<中转站key>",
    "OPENAI_BASE_URL": "https://<中转域名>"
  },
  "models": [{ "id": "<中转模型id>", "label": "<显示名>", "isDefault": true }]
}
```

注意：**custom 端点的模型必须显式声明**，Paseo 不会为 Codex 自动发现它们；端点必须支持 OpenAI **Responses API**（不只是 chat completions）。

### Step 4：重载并逐通道验证

```bash
paseo reload
paseo provider ls --json | jq '.[] | {id, available}'          # 两通道均 available
paseo provider models codex                                     # 订阅通道模型列表
codex exec --sandbox read-only -m <官方模型> "Reply with exactly: OK"   # 订阅通道冒烟
```

中转通道可用 `-c` 内联参数模拟 Paseo 的注入来冒烟（不经过 Paseo）：

```bash
OPENAI_API_KEY=<key> codex exec --sandbox read-only -m <中转模型> \
  -c 'model_providers.relay={name="relay",wire_api="responses",requires_openai_auth=false,base_url="https://<中转域名>/v1",env_key="OPENAI_API_KEY"}' \
  -c model_provider=relay "Reply with exactly: OK"
```

### Step 5：升级 CLI 后刷新模型认知

```bash
codex --version                                  # 确认升级到位
jq -r '.models[] | "\(.slug) [\(.visibility)]"' ~/.codex/models_cache.json   # 官方模型权威列表
paseo reload && paseo provider models codex      # Paseo 自动重新发现
```

内置 provider 的 `additionalModels` 只用来补**官方 `[hide]` 状态**的模型（如 `gpt-5.5`，自动发现拿不到）；`[list]` 状态的模型 Paseo 会自动发现，不要重复声明。

---

## 五、踩坑记录

### 坑 #1：config.toml 历史路由劫持订阅请求（本篇核心坑）

**现象**：`codex login` 明明成功（`auth_mode: chatgpt`），启动 agent 却报 `401 Invalid API key`，错误 URL 是中转站域名。

**根因**：`config.toml` 里遗留 `model_provider = "custom"` → 所有请求强制路由到中转站 → 拿 OAuth token 当中转 key → 401。

**判据**：**看错误里的 URL 定通道**——`chatgpt.com/backend-api` = 官方订阅；中转域名 = 路由被 config.toml 或 env 劫持。

**解法**：注释 `model_provider`；中转需求全部走 Paseo 的 custom provider profile。

### 坑 #2："model not supported" 的真因可能是 CLI 版本

**现象**：Pro 订阅下选 `gpt-6.1-sol` 报 `The model is not supported when using Codex with a ChatGPT account`。

**误判**：以为是"中转站专属模型名"（中转站确实有同名模型，容易误导）。

**真因**：CLI 0.150.1 不认识新模型；升级 0.162.0 后同名的 `gpt-6.1-sol` 就是官方 `[list]` 模型。**排错顺序**：先 `codex --version` + 查 `models_cache.json`，再怀疑模型名本身。

### 坑 #3："at capacity" 是订阅模式常态

ChatGPT 订阅在高峰期会限流（`Selected model is at capacity`），API key 模式不会。**换模型或重试即可，不要当配置错误排查**。中转站也可能转发上游同样的错误。

### 坑 #4：直连官方的网络抖动

WebSocket 偶发 `Connection reset by peer`，Codex 自动降级 HTTPS transport 后偶发超时。表现为重连 5 次失败 + 超时报错，但重试通常能过。与代理规则有关（参考 [pi-custom-model.md 坑 #2](./pi-custom-model.md)），不阻塞使用。

### 坑 #5：备份文件里藏着历史真相

`~/.codex/` 下有多个 `auth.json.bak*` / `config.toml.bak*`。本次排错中，4KB 的旧 `auth.json.bak`（OAuth 长度特征）暴露了"曾用订阅登录"的历史，71 字节的现役 key（官方 key 通常 160+ 字符）暴露了"实为中转 key"的事实。**切换认证方式前先备份，排错时看备份文件大小与字段结构能快速定位历史路径**。

---

## 六、验证清单（Checklist）

- [ ] `codex login status` 显示 ChatGPT 登录（订阅通道）
- [ ] `config.toml` 无生效的 `model_provider` 劫持（注释或删除）
- [ ] `codex exec -m <官方模型>` 冒烟通过（订阅通道）
- [ ] Paseo 中转 provider `available=true`，`models` 显式声明了中转模型
- [ ] 中转通道冒烟通过（Paseo 启动 agent 或 `-c` 内联模拟）
- [ ] `paseo provider models codex` 列表与官方 `models_cache.json` 的 `[list]` 模型一致
- [ ] `auth.json` 已按日期备份，中转 key 另有安全存放处

---

## 七、与 Paseo 其他文档的关系

- Pi provider + 同类中转站的接入（models.json 结构、中转域名失效排查）见 [pi-custom-model.md](./pi-custom-model.md)
- Paseo spawn 子进程的环境继承机制见 [architecture.md](../internals/architecture.md)
- 跨 provider 机制（内置 provider 与 custom profile 的关系）见 [cross-provider.md](../internals/cross-provider.md)
- 官方文档：[Custom providers](https://paseo.sh/docs/custom-providers.md)、[Codex](https://paseo.sh/docs/codex.md)

---

## 附：当前配置快照（2026-10-09）

| 项 | 值 |
|---|---|
| Codex CLI | 0.162.0（brew，PATH 上的同一二进制供 Paseo 调用） |
| 订阅通道 | 内置 `codex` provider，ChatGPT Pro OAuth |
| 中转通道 | `codex-relay` provider → `https://cpa.jxcq.work`（Responses API） |
| 官方模型 | gpt-6.1-sol / gpt-6-astra / gpt-6-sol / gpt-6-luna / gpt-5.6-sol / terra / luna（自动发现）+ gpt-5.5（additionalModels 补，官方 hide） |
| 中转独有 | gpt-5.3-codex-spark |
| 默认模型 | `config.toml` → gpt-6.1-sol |
| 备份 | `~/.codex/auth.json.bak-apikey-20261009`（中转 key） |
| 验证状态 | 订阅通道 gpt-5.6-terra / gpt-6.1-sol 通过；中转通道 gpt-6-sol 通过 |
