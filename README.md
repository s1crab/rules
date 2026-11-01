# 代理规则整理方案 · 分层分流 + AI 优先

一套可直接导入的两份配置，按同一套设计原则产出一致的分组名与规则顺序。规则是 2026-10-06 的快照，放在本仓库里（含后续 Cursor / Grok Bot 补丁）。**本仓库当前为私有**：小火箭用 `raw.githubusercontent.com` 拉 RULE-SET / 配置 URL 时，未带 PAT 会失败。推荐把 `shadowrocket-AI.conf` 整文件复制进 App / iCloud；mihomo 读取同目录本地文件，不受影响。两边都不再请求上游规则仓库。不含广告拦截。

| 文件 | 客户端 |
|---|---|
| [`shadowrocket-AI.conf`](./shadowrocket-AI.conf) | Shadowrocket（iOS / macOS） |
| [`mihomo-AI.yaml`](./mihomo-AI.yaml) | mihomo / Clash.Meta（Clash Verge、OpenClash、FlClash…） |

---

## 一、什么叫「科学」：四条判据

1. **单一事实来源** —— 规则以本仓库里的文件为准。mihomo 用 `rules/mihomo/`（`type: file`，运行时不访问 GitHub）。Shadowrocket 不认本地路径：历史 RULE-SET 仍指向 `github.com/s1crab/rules` 的 raw；**私有仓库时 raw 拉不到**，Cursor / Grok Bot 规则已改为 **内联在 `.conf`**，单文件导入即可用。
2. **分层 + 优先级明确** —— 规则自上而下匹配，命中即停，**顺序就是语义**。
3. **匹配维度与意图匹配** —— 该用域名的用域名，只有静态服务和私网才用 IP。
4. **可观测、可回滚** —— 能查出某个请求命中了哪条规则；配置进 git，改坏能回退。

按「应用/用途」分组，而不是按「国家」分组：你要控制的是这条流量走哪个出口。

---

## 二、分层结构

两套配置的规则顺序完全一致：

| 层 | 内容 | 动作 | 为什么在这个位置 |
|---|---|---|---|
| L0 | 私网 CIDR、`.local` | DIRECT | 必须最先，否则内网设备会被代理 |
| L2a | **境内 AI** 域名（DeepSeek、豆包、Kimi、通义…） | DIRECT | 境内 AI 走代理会变慢、掉线甚至不可用 |
| L2b | **境外 AI** 规则集 | 🤖 AI 服务 | **必须早于 L3**，见下 |
| L2c | **Cursor / Grok Bot**（内联 + `cursor.list`） | 🤖 AI 服务 | **必须早于 L3**；勿进 L4 国内直连 |
| L3 | Google / YouTube / Telegram / GitHub / Microsoft / Apple | 代理或直连 | 用途明确、地区不敏感 |
| L4 | `ChinaMax` / `GEOSITE,cn` + `GEOIP,CN` | DIRECT | 国内直连，避免绕路 |
| L5 | `FINAL` / `MATCH` | 🚀 节点选择 | 兜底：新站点自动可用 |

### 为什么 AI 必须在 L3 之前

`gemini.google.com`、`copilot.microsoft.com`、`apple-relay.apple.com`、`*mask*.icloud.com`
**同时属于大厂域名空间**。如果 Google / Microsoft / Apple 规则排在前面，AI 规则永远不会命中，
你为 AI 单独挑的节点也就形同虚设。这是本方案里最容易出错、也最容易被忽略的一点。

---

## 三、AI 分组的专项设计（重点）

### 3.1 策略组：AI 只绑定一个节点

```
🤖 AI 服务 = select, AI节点
```

境外 AI 固定走这一台，出口 IP 不变。`select` 不会因为延迟自己换节点。`url-test` 会为了最低延迟来回切换，IP 一变，ChatGPT / Claude 就会掉登录。

`AI节点` 要换成 App 或订阅里显示的完整名字。Shadowrocket 里这一处写了两遍，mihomo 的 `filter: "^AI节点$"` 也要改成同一个名字。

### 3.2 AI 也要分「境内 / 境外」

这是最容易被忽略的一层：**境内 AI 必须直连**，境外 AI 必须走固定节点，两者不能混在一个分组里。

| 类别 | 处理 | 服务 |
|---|---|---|
| 境内 AI | `DIRECT`（写死在 L2a） | DeepSeek、豆包 / 火山方舟、Kimi / Moonshot、通义 / 百炼、智谱 / ChatGLM、文心 / 千帆、混元、MiniMax、阶跃、零一万物、百川、讯飞星火、硅基流动、魔搭、商汤 |

境内 AI 走代理不仅慢，还常因为 IP 归属地变化触发登录校验或直接不可用；
反过来，境外 AI 走直连则完全用不了。

### 3.3 境外 AI 规则覆盖范围

Shadowrocket 的境外 AI 在 `rules/shadowrocket/ai.list`（含 Apple Intelligence），由配置里的 `RULE-SET` 指向本仓库。mihomo 读取 `rules/mihomo/` 里的 `category-ai-not-cn.list`、`openai.list`、`anthropic.list`、`google-gemini.list`，再加 `AINA.txt` 兜底。

覆盖的服务：ChatGPT / OpenAI、Claude / Anthropic、Gemini / AI Studio / DeepMind /
NotebookLM、Microsoft Copilot、GitHub Copilot、Grok / x.ai、OpenRouter、Perplexity、
Apple Intelligence、**Cursor / Grok Bot（L2c）**。

**容易被漏掉、但漏了会「功能异常」的第三方依赖**：

```
api.statsig.com            # 灰度实验，漏了会缺失部分能力
api-iam.intercom.io        # 客服/反馈
*.ingest.sentry.io         # 错误上报
livekit.cloud              # 语音/实时对话
```

很多人规则只抄主域名，结果「感觉模型变笨了 / 语音不能用」——原因通常在这里。

### 3.4 Cursor / Grok Bot（L2c）

官方域名与 TLS / 流式要求见：

- [Configure TLS-inspecting proxies](https://cursor.com/docs/grok-bot/proxies)
- [Enterprise network configuration](https://cursor.com/docs/enterprise/network-configuration)

| 模式 | Shadowrocket | mihomo |
|---|---|---|
| 主路径 | **内联** `DOMAIN-SUFFIX` / `DOMAIN` 于 `shadowrocket-AI.conf`（私有仓也能单文件导入） | `RULE-SET,cursor` → `rules/mihomo/cursor.list` |
| 维护副本 | `rules/shadowrocket/cursor.list`（公开镜像或带 PAT 时可改 RULE-SET） | 同上 |

**覆盖清单（优先宽匹配）**：

| 规则 | 覆盖 |
|---|---|
| `DOMAIN-SUFFIX,cursor.sh` | `*.cursor.sh`（chat / 登录 / 审批 / API） |
| `DOMAIN-SUFFIX,cursorvm.com` | `*.cursorvm.com` **与** 嵌套 `*.*.cursorvm.com`（后缀匹配足够） |
| `DOMAIN-SUFFIX,cursor.com` | `cursor.com`、`*.cursor.com`、`downloads.cursor.com` |
| `DOMAIN-SUFFIX,cursor-cdn.com` | 静态资源 |
| `DOMAIN-SUFFIX,cursorapi.com` | 扩展市场等 API |
| `DOMAIN,accounts.x.ai` / `DOMAIN,accounts.spacex.ai` + `DOMAIN-SUFFIX,spacex.ai` | 登录相关（`x.ai` 已在 `ai.list`） |
| `DOMAIN,connectors-gateway.grok.com` | Connector 网关（`grok.com` 后缀已在 `ai.list`，显式列出便于核对） |
| `DOMAIN,anysphere-binaries.s3.us-east-1.amazonaws.com` | 客户端更新包 |

**推荐用法（手机 / Shadowrocket）**：

1. 打开 **TUN / 增强模式**，让非代理感知的 App（含 Grok Bot 桌面若走系统扩展）的流量进规则。
2. AI 组固定单节点（本方案已设），避免出口 IP 横跳。
3. **规则无法在手机上强制「绕过公司网关的 TLS 解密 / 响应缓冲」**。若公司 SWG（Zscaler 等）仍在做 SSL inspection 或缓冲 SSE/HTTP2，聊天或 `cursorvm` 建连仍会挂；那是网关侧配置问题，见官方文档的 bypass / streaming 要求。

**已知缺口（规则层做不到的）**：

- 公司网关 TLS 检查 / 下载扫描 / 响应缓冲 —— 需 IT 对上述域名 Do Not Inspect、关 buffering。
- 私有仓 `raw.githubusercontent.com` RULE-SET —— 无 PAT 时拉不到；本仓 L2c 因此内联。
- 嵌套 `*.*.cursorvm.com` —— 用 `DOMAIN-SUFFIX,cursorvm.com` 已覆盖；勿只写单层通配就以为够（那是网关 ACL 的坑，不是 Shadowrocket 后缀规则的坑）。

### 3.5 DNS 与 QUIC

- AI 域名走 **DoH 且经代理解析**（Shadowrocket 的 `#proxy` 后缀），避免污染。
- mihomo 用 `nameserver-policy` 做分流 DNS：`geosite:cn` 走国内 DoH，`geolocation-!cn` 走国外 DoH。
- 域名嗅探（`sniffer` / Shadowrocket 内置）**必须开启**，否则 HTTPS 只能看到 IP，域名规则全部失效。
- QUIC：AI 应用大量使用 HTTP/3。
  - `block-quic = all-proxy`（已设为默认）= QUIC 也走代理，体验最好且不泄漏真实 IP；
  - 节点 UDP 质量差时改成 `all`，拦掉 QUIC 强制回退 TCP，最稳。
- iOS 上的现实约束：Shadowrocket 是 Network Extension，**进程名分流基本不可用**，
  `USER-AGENT` 对 HTTPS 也要 MITM 才有效 —— 所以 iOS 只能靠域名，域名清单质量决定一切。

---

## 四、导入

### 4.1 Shadowrocket（iOS）—— 私有仓库现实路径

本仓库是 **private**。小火箭用配置 URL / RULE-SET 访问

```
https://raw.githubusercontent.com/s1crab/rules/main/...
```

时，**未附带 GitHub PAT 会 404**，历史 `ai.list` / `china.list` 等 RULE-SET 同样受影响。Cursor / Grok Bot（L2c）已全部 **内联进** `shadowrocket-AI.conf`，因此至少 L2c 不依赖 raw。

**推荐（无公开镜像时）**：

1. 在能访问本仓的电脑上打开 [`shadowrocket-AI.conf`](./shadowrocket-AI.conf)，整文件复制。
2. iPhone：Shadowrocket → 配置 → **从剪贴板导入** / iCloud Drive 打开该文件；或用隔空投送 / 备忘录同步后再导入。
3. 首页确认已有名为 `AI节点` 的节点（或把配置里的名字改成你的节点全名）。
4. 启用配置，打开 **TUN / 增强模式**，连一次 ChatGPT / Cursor 相关域名，在「连接」页确认命中 `🤖 AI 服务`。

**可选（你接受把 token 写进 URL 时）**：

```
https://<GITHUB_PAT>@raw.githubusercontent.com/s1crab/rules/main/shadowrocket-AI.conf
```

PAT 需 `repo` 读权限；任何人拿到完整 URL 等于拿到读仓权限，只适合个人设备、勿分享。公开 fork / 镜像后可改回无 token 的 raw。

**若仓库变公开**：可直接用

```
https://raw.githubusercontent.com/s1crab/rules/main/shadowrocket-AI.conf
```

并按需把 L2c 改回 `RULE-SET,.../cursor.list`（列表文件已备好）。

### 4.2 mihomo

把本目录当作配置目录，或把 `mihomo-AI.yaml` 和 `rules/mihomo/` 一起放进 Clash Verge / mihomo 的配置目录，两边的相对路径要保持不变。`cursor` 规则集为本地 `type: file`，不依赖 GitHub。

**必改项（订阅）**：

```yaml
proxy-providers:
  sub:
    url: "https://example.com/your-subscription-url"   # ← 换成你的订阅链接
```

没有订阅、只有手工节点时：注释掉 `proxy-providers`，取消 `proxies` 的注释填入节点，
并删除各组的 `use: [sub]`。

**必改项（AI 节点名）**：把 Shadowrocket 和 mihomo 里的 `AI节点` 换成你那一台的完整名字。

---

## 五、验证（没验证过的规则等于没写）

1. **看命中**：Shadowrocket「连接」页 / mihomo 的 `external-controller`（如
   [metacubexd](https://github.com/MetaCubeX/metacubexd)）里访问 `chatgpt.com`，
   确认命中的是 AI 规则集、策略是 `🤖 AI 服务`。
2. **看出口**：访问 `ipinfo.io` / `ip.sb`，确认出口 IP 的 ASN 与地区就是 AI 组里那个节点。
3. **查漏**：依次打开 ChatGPT、Claude、Gemini、Copilot、**Cursor / Grok Bot**，在连接列表里看有没有
   **掉进兜底规则**的域名 —— 有的话就是 AI / Cursor 规则的漏网之鱼，补进本地规则。
4. **查顺序**：确认 L2a / L2b / **L2c** 的行号都小于 Google / Microsoft / Apple 规则。
5. **Cursor 专项**：访问 `api2.cursor.sh`、任意 `*.cursorvm.com` 应走 `🤖 AI 服务`；登录页 `accounts.x.ai` / `accounts.spacex.ai` 同理。若 App 仍卡在 computer setup，先排除公司网关 TLS 检查（见 §3.4）。

---

## 六、常见坑

| 坑 | 后果 | 正确做法 |
|---|---|---|
| AI 规则排在大厂规则之后 | AI 规则永不命中 | AI 必须前置 |
| 境内 AI 走代理 | 变慢、掉线、不可用 | 境内 AI 显式 `DIRECT` |
| AI 组用 `url-test` / `load-balance` / 多节点 `fallback` | IP 横跳 → 掉登录、风控 | AI 组只放一个节点，用 `select` |
| 用 `GEOIP` / `IP-CIDR` 判 AI | AI 都在 Cloudflare/Fastly/Azure 上，同段混装，必然误伤 | 只用域名匹配 |
| `DOMAIN-KEYWORD` 写 `ai`、`gpt`、`claude` | 大面积误伤无关站点 | 只写品牌唯一串，如 `openaicom` |
| `RULE-SET` 改回上游仓库地址 | 又去信任别人的规则 | 只引用本仓文件；私有仓优先内联 / 本地 file |
| 私有仓仍用无 token 的 raw RULE-SET | 小火箭拉规则失败 / 配置残缺 | 复制整份 `.conf`；或 PAT；或公开镜像 |
| Cursor 放进 L4 国内直连 | chat / cursorvm 走直连失败 | 保持 L2c → 🤖 AI 服务 |
| 关掉域名嗅探 | 域名规则全部失效 | 保持开启 |
| 节点本身是脏 IP / 共享机房 IP | 规则再对也会被风控 | 原生 IP、独享节点优先 |
| 只放行域名、公司网关仍 TLS 解密 / 缓冲 | Grok Bot computer setup 卡住、SSE 假死 | 网关侧对 Cursor 域 bypass inspection + 关 buffering（规则层无法代劳） |

---

## 七、两派总体架构的取舍

| | 黑名单派 `FINAL,DIRECT` | 白名单派 `FINAL,PROXY`（本方案） |
|---|---|---|
| 新站点可用性 | 需手工加规则 | 自动可用 |
| 流量 / 延迟 | 省 | 略费 |
| 适合 | 有维护习惯、国内使用为主 | 外网与 AI 使用为主 |

本方案选白名单派 + 高质量国内直连规则集 + 前置 AI 精准集：
AI 流量不会被误判，新出的 AI 站点也自动可用，只有少数需要指定地区的服务才手工加规则。

---

## 八、参考来源

下面这些仓库只是 2026-10-06 这次快照的来源，配置里不再请求它们。

- [iab0x00/ProxyRules](https://github.com/iab0x00/ProxyRules) — AI / AINA 文本规则
- [blackmatrix7/ios_rule_script](https://github.com/blackmatrix7/ios_rule_script) — Shadowrocket 分类规则（含 Apple、ChinaMax 的域名表）
- [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) — mihomo 用的域名与中国 IP 列表
- [LOWERTOP/Shadowrocket](https://deepwiki.com/LOWERTOP/Shadowrocket/6.1-configuration-file-format) — Shadowrocket 配置格式说明
- [Johnshall/Shadowrocket-ADBlock-Rules-Forever](https://github.com/Johnshall/Shadowrocket-ADBlock-Rules-Forever) — 分组与 `policy-regex-filter` 写法参考
- [Cursor · Grok Bot proxies](https://cursor.com/docs/grok-bot/proxies) — Cursor / Grok Bot 域名与 TLS 检查说明
- [Cursor · Enterprise network configuration](https://cursor.com/docs/enterprise/network-configuration) — 企业网络 / 流式连通性
