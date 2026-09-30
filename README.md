# 百炼成金模式（bailian-gold）

挂在 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)（dsh）上的
agent preset，为**阿里云百炼（Model Studio）**这条路线做前缀缓存优化。
不是独立运行时、不是 fork。

上游基线：`0.2.0-rc.2`（必须 pin，上游明示会有破坏性变更）。

名字取「百炼成钢」的谐音：百炼是平台，成金是结果——反复锤炼，省的是金子。

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

百炼把缓存命中的输入定价 0.1 元/百万，未命中 0.8 元/百万——**8 倍差价**。
所以这个 preset 的一等指标不是 token 数，是 **prefix cache hit rate**。

由此推出三条与其他 harness 相反的做法：

1. **工具目录全程固定**，不按阶段裁剪或扩展。上游 plan-mode 提示词里已经写着
   "The tool catalog stays the same across modes for request-cache stability"——
   这里把它从 plan mode 推广到整个会话。
2. **历史只追加**，压缩只做定点区间替换，系统提示永不进压缩区。
3. **动态状态一律推到尾部**，绝不注入系统提示。

## 与梁神模式的关系

形态相同（都是一个 `@deepseek-ai/dsh-agent-preset` 声明，经 bundle patch 装入 profile），
机制相反。梁神模式靠**切换工具目录**做首轮锚定——在 DeepSeek 官方 API 上划算，
在百炼上一个会话要摧毁两次前缀缓存。这里改用「工具目录常驻 + 首轮输出预算
（`bootstrapMaxTokens`，只改采样参数、对缓存零影响）+ 尾部锚定指令」达到同一目的。

## 目录

```
presets/bailian-gold.patch.yml            preset 定义，主产物
examples/aliyun-route.patch.yml           阿里云 route（OpenAI 兼容壳）
examples/aliyun-anthropic-route.patch.yml 阿里云 route（Anthropic 端点，更接近原生）
docs/spike-01-*.md                        前缀可否锁住的源码级验证
.ref/                                     上游参考源码（gitignore，只读）
```

## 接入

完整可用需要两部分，分工不同：

| 部分 | 层级 | 管什么 |
|---|---|---|
| `presets/bailian-gold.patch.yml` | Agent（preset） | 工具面、提示词、压缩策略 |
| `examples/aliyun-route.patch.yml` | Host（profile patch） | 模型路由与容量 |

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

**不要设 model 的 `maxTokens`**：它会同时成为每请求输出默认值，压低 `messageBudget`，
让压缩更早触发，与本预设目标相反。

### 协议层：两条路，以及一条补不了的

百炼同时提供 OpenAI 兼容壳（`/compatible-mode/v1`）和 Anthropic 兼容端点
（`/apps/anthropic`，`POST …/v1/messages`）。后者是更接近原生的那条：

| | OpenAI 壳 | Anthropic 端点 |
|---|---|---|
| 思考输出 | `reasoning_content` 字段（非标参数要被拦成 header 才能透传） | 原生 thinking 内容块 |
| 工具调用 | OpenAI function call | 原生 `tool_use` / `tool_result` 块 |
| 缓存 | 仅隐式 | 隐式 + **`cache_control: {type: ephemeral}` 显式断点** |
| `systemPromptUpdate: in-history` | ✗ | ✗ |
| `/v1/models` 发现 | ✓ | ✗（须手写 models）|

配置见 `examples/aliyun-anthropic-route.patch.yml`。

**`in-history` 补不了，两边都堵**：官方 DeepSeek 靠它保证"系统提示变更时追加而非
重写开头"，是保前缀缓存的关键。但百炼 Anthropic 文档明确"**`system` 是顶层参数，
`messages` 数组不接受 system 角色**"，端点语义就不支持；pi-ai 侧的
`supportsMidConvoSystemMessages` / `supportsMidConvoToolAdditions` 两个能力位
在 compat 表里是 `withhold`（只能由 pi-ai 内置 catalog 声明），手申报路由拿不到。

**照搬 `llm-deepseek` 的默认 catalog 声明 `in-history`，打这个端点会让系统提示直接失效。**

替代方案是把前缀稳定性交给 harness 侧——本 preset 的三条设计本就是在保证系统提示不变，
不需要端点配合。这条原生机制是**用设计绕过去的，不是补出来的**。

## 安装

前提：profile 里已有 `llm-pi-ai` 的 `aliyun` route（见 `examples/`，OpenAI 壳与
Anthropic 端点两种方案按需选一）。

```bash
dsh plugin --profile web add github:SZYTree0312/dsh-bailian-gold
```

装完后在会话预设选择器里选 **百炼成金模式**。卸载：

```bash
dsh plugin --profile web remove dsh-bailian-gold
```

> **尚未实机验证。** 本 preset 的结论全部来自对上游 `0.2.0-rc.2` 源码的静态分析
> 与端点探测，还没有跑过一次真实会话。前缀能否锁住、压缩触发点是否如预期，
> 都待实测确认后再依赖。

## 状态

- [x] 源码级验证：前缀可锁（见 `docs/spike-01-context-compaction.md`）
- [x] preset 第一版：纯配置，基于 upstream `standard` 改造
- [x] provider 校准为 `aliyun`，route 配套配置见 `examples/`
- [x] 定位修正：从「Qwen 专用」改为「百炼全平台」——差异化在缓存，不在厂商
- [ ] 实机验证：装进 profile 跑一次，确认 preset 加载与压缩触发点
- [ ] `bootstrapMaxTokens` 落点（已定位到 `llm-pi-ai`，具体参数待确认）
- [ ] 缓存命中率可观测：把 `cached_tokens` 暴露到 telemetry
