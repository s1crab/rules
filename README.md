# 代理规则整理方案 · 分层分流 + AI 优先

一套可直接导入的两份配置，按同一套设计原则产出一致的分组名与规则顺序。规则是 2026-10-06 的快照，放在本仓库里。Shadowrocket 通过本仓库的 raw 地址引用；mihomo 读取同目录文件。两边都不再请求上游仓库。不含广告拦截。

| 文件 | 客户端 |
|---|---|
| [`shadowrocket-AI.conf`](./shadowrocket-AI.conf) | Shadowrocket（iOS / macOS） |
| [`mihomo-AI.yaml`](./mihomo-AI.yaml) | mihomo / Clash.Meta（Clash Verge、OpenClash、FlClash…） |

---

## 一、什么叫「科学」：四条判据

1. **单一事实来源** —— 规则以本仓库里的文件为准。mihomo 用 `rules/mihomo/`。Shadowrocket 不认本地路径，所以 `RULE-SET` 指向 `github.com/s1crab/rules` 的 raw 地址，而不是上游仓库。
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
Apple Intelligence。

**容易被漏掉、但漏了会「功能异常」的第三方依赖**：

```
api.statsig.com            # 灰度实验，漏了会缺失部分能力
api-iam.intercom.io        # 客服/反馈
*.ingest.sentry.io         # 错误上报
livekit.cloud              # 语音/实时对话
```

很多人规则只抄主域名，结果「感觉模型变笨了 / 语音不能用」——原因通常在这里。

### 3.4 DNS 与 QUIC

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

**Shadowrocket**：在 App 里添加配置 URL。仓库需要是公开的，并且 `main` 上已经有这些文件，否则小火箭拉不到。

```
https://raw.githubusercontent.com/s1crab/rules/main/shadowrocket-AI.conf
```

**mihomo**：把本目录当作配置目录，或把 `mihomo-AI.yaml` 和 `rules/mihomo/` 一起放进 Clash Verge / mihomo 的配置目录，两边的相对路径要保持不变。

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
3. **查漏**：依次打开 ChatGPT、Claude、Gemini、Copilot，在连接列表里看有没有
   **掉进兜底规则**的域名 —— 有的话就是 AI 规则集的漏网之鱼，补进本地规则。
4. **查顺序**：确认 AI 规则的行号小于 Google / Microsoft / Apple 规则。

---

## 六、常见坑

| 坑 | 后果 | 正确做法 |
|---|---|---|
| AI 规则排在大厂规则之后 | AI 规则永不命中 | AI 必须前置 |
| 境内 AI 走代理 | 变慢、掉线、不可用 | 境内 AI 显式 `DIRECT` |
| AI 组用 `url-test` / `load-balance` / 多节点 `fallback` | IP 横跳 → 掉登录、风控 | AI 组只放一个节点，用 `select` |
| 用 `GEOIP` / `IP-CIDR` 判 AI | AI 都在 Cloudflare/Fastly/Azure 上，同段混装，必然误伤 | 只用域名匹配 |
| `DOMAIN-KEYWORD` 写 `ai`、`gpt`、`claude` | 大面积误伤无关站点 | 只写品牌唯一串，如 `openaicom` |
| `RULE-SET` 改回上游仓库地址 | 又去信任别人的规则 | 只引用 `s1crab/rules` 里的文件 |
| 关掉域名嗅探 | 域名规则全部失效 | 保持开启 |
| 节点本身是脏 IP / 共享机房 IP | 规则再对也会被风控 | 原生 IP、独享节点优先 |

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
