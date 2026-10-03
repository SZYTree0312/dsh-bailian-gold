# 百炼成金模式（bailian-gold）

挂在 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（dsh）上的
agent preset，为**阿里云百炼（Model Studio）**这条路线做前缀缓存优化。
不是独立运行时、不是 fork。

上游基线：`0.2.0-rc.2`（必须 pin，上游明示会有破坏性变更）。

名字化用成语「百炼成钢」：百炼是平台，成金是结果——反复锤炼，省的是金子。

## 适用范围：一个 preset，全部模型

这份 preset 的差异化**与模型厂商无关**，只跟端点的隐式前缀缓存有关。所以百炼上的
Qwen / DeepSeek / Kimi / GLM / MiniMax 全部共用同一个 preset。

每个模型的差异落在两处，都不需要第二个 preset、更不需要 git branch：

- **窗口大小、是否支持思考** → route 的 `models` 条目（Host 层）
- **压缩比例** → `modelPolicies`（按 provider + model 精确覆盖）

### 为什么百炼上的 DeepSeek 比官方 DeepSeek 更需要它

dsh 的 `standard` preset 是为官方 DeepSeek API 设计的，那个端点有
`systemPromptUpdate: in-history` 兜底——系统提示变更时**追加在缓存历史之后，而不是重写开头**。

百炼两条路都拿不到这个能力（原因见下文「协议层」）。于是 `standard` 在百炼上会放任
系统提示重写、前缀缓存整体失效。**本 preset 不依赖它**——前缀稳定性由 harness 侧自己保证。
这就是「阿里云下的 DeepSeek 和官方的用起来不一样」的根源。

## 一句话原理

百炼对缓存命中的输入按输入单价的折扣计费 —— **隐式命中 20%，显式命中 10%**。
所以这个 preset 的一等指标不是 token 数，是 **prefix cache hit rate**。

由此推出三条与其他 harness 相反的做法：

1. **工具目录全程固定**，不按阶段裁剪或扩展。上游 plan-mode 提示词里已经写着
   "The tool catalog stays the same across modes for request-cache stability"——
   这里把它从 plan mode 推广到整个会话。
2. **历史只追加**，压缩只做定点区间替换，系统提示永不进压缩区。
3. **动态状态一律推到尾部**，绝不注入系统提示。

### 已实测（2026-09-30）

在同一实例上发三次同一段 10391 token 的前缀：

| | 第 1 次 | 第 2 次 | 第 3 次 |
|---|---|---|---|
| OpenAI 壳（隐式）| cached 0 | **cached 10240** | **cached 10240** |
| Anthropic 端点（显式）| create 10377 | **read 10377** | **read 10377** |

**两种缓存都通，隐式覆盖 98.5%、显式覆盖 99.9%。** 完整数据、计费对照与推算见
[`docs/verify-prefix-cache.md`](docs/verify-prefix-cache.md)。

## 成本核算：三条杠杆，按大小排

| token 类别 | 单价 | 谁在管 |
|---|---|---|
| 命中缓存的输入 | 输入价 × 20%（隐式）/ 10%（显式）| 本 preset 的全部设计 |
| 未命中的输入（每轮新增）| 输入价 × 100% | 压缩期减压阀 + 提示词纪律 |
| 输出（含思考）| 输出价，**不参与缓存** | 提示词纪律（部分）|

**输入是绝对大头。** 一段稳定前缀每轮原样重发，会话越长，缓存折扣的杠杆越大。

### 杠杆一：把隐式缓存换成显式缓存

设一段 T token 的稳定前缀连续重复 N 轮：

```
隐式 = 1.00 + 0.20 × (N−1)
显式 = 1.25 + 0.10 × (N−1)      ← 创建贵 25%，命中便宜一半
转折点：N ≥ 4
```

| 轮数 | 隐式 | 显式 | 显式省 |
|---|---|---|---|
| 3 | 1.40 | 1.45 | −3.6% |
| 10 | 2.80 | 2.15 | 23.2% |
| 50 | 10.80 | 6.15 | **43.1%** |
| 100 | 20.80 | 11.15 | **46.4%** |

渐近上限 **50%**。本 preset 的设计恰好落在显式缓存的最优区 —— 它的全部功力都花在
「让前缀字节稳定、每轮原样重发」上，而稳定前缀重复轮数越多，显式越划算。

→ 走 [`examples/aliyun-anthropic-route.patch.yml`](examples/aliyun-anthropic-route.patch.yml)。
这条路线以前只作为「更接近原生」的备选写着，**它的第一价值其实是省钱**。

> 代价：显式缓存有效期固定 5 分钟（命中后重置）；隐式由系统不定期清理、没有明确上限。
> 交互密集的会话选显式；搁置型会话两者差别不大。

### 杠杆二：别让压缩触发

压缩是唯一会主动把前缀打断的操作，而它的触发点由 route 的 `contextWindow` 决定 ——
**这一项没填对，上面所有设计都白搭**，见下文。

### 杠杆三：思考是笔隐形输出开销

`qwen3.8-flash` 在百炼上**默认开思考**，而 `llm-pi-ai` 对未声明 `reasoningEfforts`
的模型判定为「不思考」（源码注释原话：*a hand-declared model has none and does not
reason*），于是**既不发控制参数，也不在 harness 视野里**。实测一次普通提问，
80 个输出 token 里 55 个是思考（69%）。官方明确「思维链内容全部计入输出 Token 统计」。

输出价是缓存命中价的几十倍，这是一条每轮都在跑的开销。**怎么控制见
[`docs/verify-prefix-cache.md`](docs/verify-prefix-cache.md) 第 4 节**（配置写法**未验证**，
且关掉思考是否划算取决于它换来的轮数 —— 是取舍，不是纯赚）。

## 与梁神模式的关系

形态相同（都是一个 `@deepseek-ai/dsh-agent-preset` 声明，经 bundle patch 装入 profile），
机制相反。梁神模式靠**切换工具目录**做首轮锚定——在 DeepSeek 官方 API 上划算，
在百炼上一个会话要摧毁两次前缀缓存。这里改用「工具目录常驻 + 首轮输出预算
（`bootstrapMaxTokens`，只改采样参数、对缓存零影响）+ 尾部锚定指令」达到同一目的。

## 目录

```
presets/bailian-gold.patch.yml            preset 定义，主产物（0.1.7+ / 0.2.x）
compat/0.1.5/                             dsh 0.1.5-rc.x 的兼容版（旧 preset 机制）
examples/aliyun-anthropic-route.patch.yml 阿里云 route（Anthropic 端点）★ 本预设配套，用它
examples/aliyun-route.patch.yml           阿里云 route（OpenAI 兼容壳，备选）
docs/spike-01-*.md                        前缀可否锁住的源码级验证
docs/verify-prefix-cache.md               实测：缓存命中、显式 vs 隐式计费、思考开销
docs/compare-whale-elite.md               与鲸英模式的逐项对比
.ref/                                     上游参考源码（gitignore，只读）
```

## 版本兼容

覆盖 dsh 的多个正式发行版（rc 线，alpha 不算）。分界点在 **0.1.7-rc.1**：

| dsh 版本 | preset 机制 | 用哪个 |
|---|---|---|
| **0.1.7-rc.1 及以后**（含 0.2.0-rc.x）| `@deepseek-ai/dsh-agent-preset` 插件行，经 profile patch 装入 | `presets/bailian-gold.patch.yml` |
| **0.1.5-rc.1 ~ rc.3** | `@deepseek-ai/dsh-agent-presets`（**复数**）扫描 `<dshHome>/.agent-presets/` | `compat/0.1.5/` |

从 0.1.7-rc.1 起，上游把 preset 拆成 `agent-preset-registry` + `agent-preset` 两个包，
preset 也从「一个独立文件」变成「一行插件声明」。

0.1.5 的安装方式不同（旧机制不走 plugin 命令，是把目录复制到
`~/.dsh/.agent-presets/<id>/`），差异清单与操作步骤见
[`compat/README.md`](compat/README.md)。

## 接入

**本 preset 自带两部分，装完即完整**，不必再手动叠加 `examples/` 里的 route：

| 段 | 层级 | 管什么 |
|---|---|---|
| `insert` · `preset-bailian-gold` | Agent（preset） | 工具面、提示词、压缩策略 |
| `override` · `llm-pi-ai` | Host（profile patch） | 模型路由：固定走 Anthropic 端点 |

第二段是 **id-targeted override**：base 的 core bundle 里已经有 `llm-pi-ai`
（默认带的是 OpenAI 兼容壳），这里按 id 把它覆盖成
`api: anthropic-messages` + `/apps/anthropic`，并把 `contextWindow: 991808` 写好。

> cordis 规则：`insert` 只用于新增，顶层 `- id:` + `config:` 用于按 id 覆盖，
> 且**覆盖会替换整个 config**。这也是为什么那段里把 route 拥有的每个键都重述了一遍。

想改回或微调（换专属实例域名、增删模型、改 compat）：在你自己的 profile
`cordis.patch.yml` 或 `--patch` overlay 里再覆盖一次即可 ——
那些层在本 preset **之后**应用，盖得回来。

### route 名必须对齐

provider 标识取 profile 里 llm-pi-ai 配置的 route 名。本机是 **`aliyun`**
（不是 `dashscope`——写错不会报错，`modelPolicies` 会静默失效，整套策略照常退回上游默认）。
preset 里的 `modelPolicies.provider` 与 `summarizationProvider` 都按此写。

### 为什么 route 的 contextWindow 是关键数

compaction 的实际触发点是：

```
min(contextWindow × thresholdRatio,  messageBudget − headroomTokens)
messageBudget = contextWindow − 每请求输出预留
```

窗口不够大时，**右边那项才是真正生效的**，`thresholdRatio` 形同虚设。以本机默认值
（`defaultContextWindow` 262144、`defaultMaxTokens` 32768、headroom 65536）代入：

```
messageBudget  = 262144 − 32768 = 229376
pressureBudget = 229376 − 65536 = 163840
触发点 = min(262144 × 0.92 = 241172,  163840) = 163840
```

`thresholdRatio: 0.92` 期望的 241172 永远够不着。把 `contextWindow` 开到 Qwen3.8-Flash
官方标准模式的最大输入 **991808**，触发点升到约 **893504**，常规会话根本压不到——
前缀缓存也就不会因为一次 `replace` 而整体失效。

> ⚠️ **不填 `contextWindow` 就等于放弃这个 preset 的主要收益。** 它回落到
> `defaultContextWindow` 262144，触发点只有 **163840** —— 一段正常的编码会话够得着，
> 压缩（连带前缀缓存失效）会真的发生。`examples/` 里已经写好，**照抄到 profile 的
> `cordis.patch.yml` 才算数**。

**不要设 model 的 `maxTokens`**：它会同时成为每请求输出默认值，压低 `messageBudget`，
让压缩更早触发，与本预设目标相反。

### 协议层：两条路，本预设只走 Anthropic 端点

百炼同时提供 OpenAI 兼容壳（`/compatible-mode/v1`）和 Anthropic 兼容端点
（`/apps/anthropic`，`POST …/v1/messages`）。**本预设只走后者**：

| | OpenAI 壳 | Anthropic 端点 |
|---|---|---|
| 思考输出 | `reasoning_content` 字段（非标参数要被拦成 header 才能透传） | 原生 thinking 内容块 |
| 工具调用 | OpenAI function call | 原生 `tool_use` / `tool_result` 块 |
| 缓存 | 仅隐式（命中 2 折 ¥0.16/M） | 隐式 + **`cache_control` 显式断点（命中 1 折 ¥0.08/M）** |
| 尾部追加 system | — | **端点支持**（实测见下） |
| `/v1/models` 发现 | ✓ | ✗（须手写 models） |

配置见 `examples/aliyun-anthropic-route.patch.yml`。

**`in-history` 的真实情况（2026-10-04 实测更正）**

这里原先写「端点语义不支持、两边都堵」—— 那是照抄官方文档，没实测。拿真实 key
打过去之后，两个实验都推翻了它：

- **谁优先**：顶层 `system` 要求含 AAA、`messages` 内 `system` 要求含 BBB
  → 回复同时含 AAA 和 BBB。**合并生效，不是覆盖** —— `messages` 数组接受 system 角色。
- **保不保前缀**：固定前缀带 `cache_control`，连发 3 轮、每轮在尾部追加一条新 `system`：
  `create=1802` → `read=1802` → `read=1802`。追加的 system 语义生效，
  **而前缀缓存一次都没断**。

也就是说：动态状态可以追加在历史尾部更新系统提示，不必重写开头那条消息 ——
这正是 `in-history` 想要的效果，**端点层完全给得起**。

真正堵的是中间层：pi-ai 的 `supportsMidConvoSystemMessages`（及
`supportsMidConvoToolAdditions`）在 compat 表里是 `withhold`，只由内置 catalog
声明，手申报路由拿不到，dsh 因此不会那样组织请求。

**所以策略不变，但理由变了**：前缀稳定性依旧交给 harness 侧三条设计 ——
不是端点给不起，而是中间层能力位没放开。若哪天该位放开，或改用自建 fork，
尾部追加 system 这条路可以直接用起来，`includeRuntimeContext: false`
那个全有全无的妥协也就不再必要。

## 安装

前提：profile 里已有 `llm-pi-ai` 的 `aliyun` route。**用 Anthropic 端点那份**
（`examples/aliyun-anthropic-route.patch.yml`，实测公共端点即可，不必专属实例）。
OpenAI 壳那份（`examples/aliyun-route.patch.yml`）仅作为备选保留。

```bash
dsh plugin --profile web add github:SZYTree0312/dsh-bailian-gold
```

装完后在会话预设选择器里选 **百炼成金模式**。卸载：

```bash
dsh plugin --profile web remove dsh-bailian-gold
```

> **部分实机验证。** 前缀缓存是否可命中已用真实请求测通（见
> [`docs/verify-prefix-cache.md`](docs/verify-prefix-cache.md)）；其余结论来自对上游
> `0.2.0-rc.2` 源码的静态分析。**本 preset 本身还没有跑过一次完整真实会话** ——
> 工具目录是否真的全程不变、压缩触发点是否如预期、真实会话的命中率，都待实测。

## 状态

- [x] 源码级验证：前缀可锁（见 `docs/spike-01-context-compaction.md`）
- [x] preset 第一版：纯配置，基于 upstream `standard` 改造
- [x] provider 校准为 `aliyun`，route 配套配置见 `examples/`
- [x] 定位修正：从「Qwen 专用」改为「百炼全平台」——差异化在缓存，不在厂商
- [x] 对比鲸英模式并采纳 `includeRuntimeContext: false` 与 prompt 纪律；
      工具结果裁剪一项经 2026-09-30 复核**降级为「压缩期减压阀」**，
      且其取值等于上游默认（见 `docs/compare-whale-elite.md`）
- [x] **实测：前缀缓存可命中**（隐式 98.5% / 显式 99.9%），
      并算出隐式→显式的成本转折点在第 4 轮（见 `docs/verify-prefix-cache.md`）
- [x] route 已内置进 preset（`llm-pi-ai` override 段，含 `contextWindow: 991808`），装完即生效
- [ ] 实机验证：装进 profile 跑一次完整会话，确认 preset 加载与压缩触发点
- [ ] 思考开销的控制（`reasoningEfforts` + `compat.thinkingFormat`）—— 写法未验证
- [ ] `bootstrapMaxTokens` 落点（已定位到 `llm-pi-ai`，具体参数待确认）
- [ ] 缓存命中率可观测：把 `cached_tokens` 暴露到 telemetry
