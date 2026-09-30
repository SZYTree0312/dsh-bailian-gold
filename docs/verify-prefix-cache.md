# 实测：百炼上的前缀缓存到底通不通

日期：2026-09-30 · 端点：本机 desktop profile 配的百炼专属 MaaS 实例
（`llm-xaxsovymhpyl50jm.cn-beijing.maas.aliyuncs.com`）· 模型：`qwen3.8-flash`

本仓库此前所有结论都来自源码静态分析，README 里挂着「尚未实机验证」。这份文档把
其中最要紧的一条 —— **前缀缓存到底能不能命中** —— 用真实请求测掉了。

## 1. 隐式缓存：通过

OpenAI 兼容壳 `/compatible-mode/v1/chat/completions`，同一段 10391 token 的前缀连发三次：

| 次序 | prompt_tokens | cached_tokens |
|---|---|---|
| 1 | 10391 | **0** |
| 2 | 10391 | **10240** |
| 3 | 10391 | **10240** |

**结论：命中 10240 / 10391 = 98.5%。** 第一次建立缓存（按输入价 100% 计费），
第二、三次命中（按输入价 20% 计费）。前缀缓存是通的，preset 的立论成立。

未命中的 151 token 是每轮新增的那条 user 消息。

## 2. 显式缓存：同样通过

Anthropic 端点 `/apps/anthropic/v1/messages`，`system` 块打 `cache_control: {type: ephemeral}`：

| 次序 | input_tokens | cache_creation | cache_read |
|---|---|---|---|
| 1 | 14 | **10377** | 0 |
| 2 | 14 | 0 | **10377** |
| 3 | 14 | 0 | **10377** |

**结论：显式断点可用，且覆盖 10377 / 10391 = 99.9% —— 比隐式多盖住 137 token。**
第一次按输入价 125% 计费创建，之后按 10% 计费命中。

## 3. 两种缓存的计费与取舍

官方计费（[上下文缓存](https://help.aliyun.com/document_detail/2862577.html)）：

| | 隐式缓存 | 显式缓存 |
|---|---|---|
| 开启 | 自动，无法关闭 | 需 `cache_control: {type: ephemeral}` |
| 建立缓存计费 | 输入价 × **100%** | 输入价 × **125%** |
| 命中计费 | 输入价 × **20%** | 输入价 × **10%** |
| 最小 token | 1024 | 1024 |
| 有效期 | 不确定（系统定期清理） | 5 分钟，命中后重置 |
| 互斥 | 单个请求只能用其中一种 | |

### 转折点算得出来：第 4 轮

设一段 T token 的稳定前缀连续重复 N 轮（缓存保持温热），单位是「输入价 × T」：

```
隐式 = 1.00            + 0.20 × (N−1)
显式 = 1.25            + 0.10 × (N−1)
差   = −0.25 + 0.10 × (N−1)    > 0  当且仅当 N ≥ 4
```

| 轮数 N | 隐式相对成本 | 显式相对成本 | 显式省 |
|---|---|---|---|
| 3 | 1.40 | 1.45 | −3.6%（隐式略优）|
| 5 | 1.80 | 1.65 | 8.3% |
| 10 | 2.80 | 2.15 | 23.2% |
| 20 | 4.80 | 3.15 | 34.4% |
| 50 | 10.80 | 6.15 | **43.1%** |
| 100 | 20.80 | 11.15 | **46.4%** |

渐近上限是 **50%**。

按 0.8 元/百万输入价、6 万 token 前缀、50 轮估：

| | 建立 | 每轮命中 | 50 轮合计 |
|---|---|---|---|
| 隐式 | 0.048 元 | 0.0096 元 | **0.518 元** |
| 显式 | 0.060 元 | 0.0048 元 | **0.295 元** |

**差 0.223 元，降 43%。**

### 判断

**本 preset 的设计恰好落在显式缓存的最优区。** 它的全部功力都花在「让前缀字节稳定、
每轮原样重发」上 —— 而稳定前缀重复轮数越多，显式缓存越划算。编码会话动辄几十上百轮，
第 4 轮的转折点轻松越过。

所以：**推荐走 Anthropic 端点 + `cache_control`**，见
[`examples/aliyun-anthropic-route.patch.yml`](../examples/aliyun-anthropic-route.patch.yml)。
这条建议以前只作为「更接近原生」的备选写着，其实它的第一价值是**省钱**。

**代价**：显式缓存有效期固定 5 分钟且命中后重置；隐式是系统不定期清理、没有明确上限。
会话中间长时间挂着不动（超过 5 分钟）而前缀又没被清理的话，显式要按 125% 重建一次，
隐式只按 100% 重建。**交互密集的会话选显式，搁置型会话两者差别不大。**

## 4. 附带发现：`qwen3.8-flash` 默认开思考

同一个端点，不带任何思考参数直接发：

```
message 键: ['role', 'reasoning_content', 'content']
reasoning_content 长度 = 108 字符
usage: completion_tokens=80, completion_tokens_details={reasoning_tokens: 55, text_tokens: 80}
```

**80 个输出 token 里 55 个是思考 —— 69%。** 显式传 `enable_thinking: false` 后：

```
message 键: ['role', 'content']
usage: completion_tokens=1
```

思考内容全没了。

### 为什么这笔钱是「看不见的」

`llm-pi-ai` 对模型的推理能力采用「未声明即不具备」的判定
（`catalog.ts` → `resolveModelReasoning`：`reasoningEfforts` 缺省时
`reasoning: base?.reasoning ?? false`，注释原话是 *a hand-declared model has none and does not reason*）。

本 preset 的 route **没有声明 `reasoningEfforts`**，于是 pi-ai 认为这个模型不思考、
**不发任何思考参数**；但端点的默认行为是开思考，照常产出并计费。

> 官方原文（[Qwen3.7-Max 定价说明](https://developer.aliyun.com/article/1765221)）：
> 「开启 `enable_thinking` 深度思考，模型输出的思维链内容**全部计入输出 Token 统计**，会产生计费消耗。」

输出是 2.7 元/百万，**是缓存命中价的 27 倍**。这是一条每轮都在跑、既不受 harness 控制、
也不在 harness 视野里的开销。

### 怎么控制

route 的模型条目上声明推理等级（写法见 pi-ai README「带推理与协议兼容运行」）：

```yaml
models:
  - id: qwen3.8-flash
    name: qwen3.8-flash
    contextWindow: 991808
    reasoningEfforts:      # 键 = 选择器等级，值 = 分派时在协议里发送的拼写
      off:                 # 只有 off 允许留空
      high: high
    compat:
      thinkingFormat: qwen # 取值必须在 pi-ai 的 THINKING_FORMAT_GATE 里
```

**⚠️ 未验证。** 上表是从 pi-ai 的配置文档与 `catalog.ts` 的校验逻辑推出来的，
本仓库**没有实机跑过**。`thinkingFormat: qwen` 到 wire 参数的映射在 pi-ai 库内部，
本地源码看不到。写错可能让每次请求都失败，上生产前先拿一条小请求试。

**也别急着关思考。** 关掉省的是输出 token，但编码 agent 少了推理容易多绕几轮 ——
而每一轮都要付新内容的全价。**这是取舍，不是纯赚。** 建议先按上面的写法把它变成
「可见、可调」，观察到真实用量再决定档位。

## 5. 验证边界

**已实测**：本机专属 MaaS 实例上 `qwen3.8-flash` 的隐式缓存、显式缓存、默认思考行为。
用的人造前缀 10391 token，连发 3 次。三次实验共消耗约 **0.027 元**。

**未验证**：

- 真实会话里的命中率（人造前缀是理想情况；真实会话有工具目录、历史增长、压缩）
- `thinkingFormat: qwen` 的 wire 映射
- 通用端点（`dashscope.aliyuncs.com`）与专属实例行为是否一致
- 各模型差异：`deepseek-v4.1-flash` 等第三方模型的缓存与思考行为未测
