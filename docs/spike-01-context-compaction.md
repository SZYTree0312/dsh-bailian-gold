# Spike 01 — 前缀能不能锁住

日期：2026-09-29 · 上游：`deepseek-ai/deepseek-harness` @ `0.2.0-rc.2`

## 问题

整套 Qwen 缓存策略的前提是：会话的 prompt 前缀在整个生命周期里保持字节稳定。
若 `context` 或 `compaction` 会整体重写 messages 数组，前提不成立，策略落空。

## 结论

**前缀可锁。而且上游已经按缓存友好在设计了，比预期保守得多。**

原判断中"压缩可能重写前缀"的担心不成立——系统提示被明确排除在压缩之外，
压缩是局部区间替换，且上游在实现里主动为复用 provider KV cache 构造真前缀。

## 证据

**1. 系统提示永不进压缩区** — `packages/compaction/compaction-basic/src/region.ts:131`

```js
const firstIdx = systemHead(session, surfaceNodes[0]) === undefined ? 0 : 1
```

`systemHead()` 只认 surface node 0。压缩范围从 `firstIdx` 起算，node 0 被跳过。
`selectCompactableRange()` 的 doc comment 写明："A `system/message` at surface node 0 is never inside the range."

**2. 压缩是局部区间替换，不是重写数组** — 同文件 `:506`

```js
session.append('user/message', checkpointMessage, {
  surfaceOp: { op: 'replace', startSeq: start, endSeq: end },
  sourceEventSeqs: [startEvent.seq, summaryEvent.seq, ...shadowedSeqs],
})
```

surface 是 append-only + 定点 replace。区间外的内容原封不动。

**3. 上游已经在为 KV cache 设计** — 同文件 `:532`（`buildSummarizationInput` doc comment）

> Reconstruct the last routed request's **cacheable prefix** for the shadowed region...
> The summarizer appends only the compaction instruction after this, so the call is a
> genuine prefix of the conversation and **reuses the provider's KV cache**.

摘要请求本身按 system + tools + region messages 拼成对话的真前缀，
只在尾部追加压缩指令。

**4. 指令注入是增量追加，不改 head** — `packages/context/agent-instructions/src/render.ts`

`renderInstructionChanges()` 渲染 `set` / `replace` / `remove` 变更，并配
`REPLACEMENT_AGENT_INSTRUCTIONS_INTRO`（"This complete baseline replaces all earlier
baselines"）。基线落在 system head 里保持不动，后续变更作为新事件追加到尾部。

**5. 上游自己就在守"工具目录不变"** — `packages/bundle/web-app/presets/standard.patch.yml:56`

plan-mode 提示词原文：

> The tool catalog stays the same across modes **for request-cache stability**.
> ...those tools remain listed to keep the tool catalog unchanged.

**6. 压缩策略可按模型精确覆盖** — `packages/compaction/compaction-basic/src/config.ts`

`modelPolicies` 按 `provider` + `model` 精确匹配，可覆盖 `thresholdRatio` /
`retainRatio` / `headroomTokens` / `summarizationProvider` / `summarizationModel` 等。
Qwen 的差异化大部分是纯配置，不必改代码。

## 被修正的判断

| 原判断 | 实况 |
|---|---|
| compaction 可能整体重写 messages，前缀锁不住 | 明确保留 system head，replace 只动选中区间 |
| 需要自建 context 插件锁前缀 | `modelPolicies` 已能覆盖多数差异化，第一版可纯配置 |
| 尾部保留策略需自己实现 | 上游默认 `retainRatio: 0.16` 已是尾部 verbatim 保留 |

## 仍待验证

- **工具目录的实际稳定性**：`session.requestHeader()` 返回的 `tools` 是否随
  runtime 状态变化（需读 `packages/session`，本轮未拉）。
- **`bootstrapMaxTokens` 的落点**：梁神模式用它限制首轮输出（`max_tokens=1024`），
  只改采样参数、对缓存零影响，是想继承的机制。dsh 侧有没有现成的请求参数钩子，
  需读 `packages/llm` 确认。
- ~~**provider/model 标识**：`modelPolicies` 里的 `provider` 用 `dashscope` 是推测值，
  需按实际路由配置校准。~~ **已解决（2026-09-29）**：本机桌面端
  `~/.dsh/profiles/web/profiles/desktop/cordis.patch.yml` 里 `llm-pi-ai` 已声明
  provider id 为 **`aliyun`**（阿里云百炼 OpenAI 兼容端点），model `qwen3.8-flash`。
  preset 已回填为 `aliyun`。

## 副产物

- `.ref/standard.patch.yml` — 上游 standard preset 全文（146 行），本 preset 的基线。
