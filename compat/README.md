# 版本兼容

本 preset 覆盖 dsh 的**多个正式发行版（rc 线，alpha 不算）**。

## 分界点：0.1.7-rc.1

| dsh 版本 | preset 机制 | 用哪个 |
|---|---|---|
| **0.1.7-rc.1 及以后**（含 0.2.0-rc.x） | `@deepseek-ai/dsh-agent-preset` 插件行，经 profile patch 装入 | `presets/bailian-gold.patch.yml` |
| **0.1.5-rc.1 ~ rc.3** | `@deepseek-ai/dsh-agent-presets`（**复数**）扫描 `<dshHome>/.agent-presets/` | `compat/0.1.5/` |

从 0.1.7-rc.1 起，上游把 preset 拆成 `agent-preset-registry` + `agent-preset` 两个包，
preset 也从"一个独立文件"变成"一行插件声明"。0.1.5 之前是另一套。

## 0.1.7+ 怎么装

```bash
dsh plugin --profile web add github:SZYTree0312/dsh-bailian-gold
```

## 0.1.5 怎么装

旧机制**不走 plugin 命令**，是把 preset 目录复制过去：

```bash
# Windows (PowerShell)
Copy-Item -Recurse compat\0.1.5 "$env:USERPROFILE\.dsh\.agent-presets\bailian-gold"

# macOS / Linux
cp -r compat/0.1.5 ~/.dsh/.agent-presets/bailian-gold
```

重启 dsh，在预设选择器里选「百炼成金模式」。

## 0.1.5 版的已知差异

| 项 | 0.1.7+ | 0.1.5 |
|---|---|---|
| workflow 插件 | `@deepseek-ai/dsh-workflow-ptc` | `@deepseek-ai/dsh-workflow-worker-thread` |
| `dsh-plugin-manager` | 存在（本 preset 里本就 `disabled: true`） | 不存在，已整行移除 |
| `compaction-basic.headroomTokens` | 支持 | 不支持（本 preset 未使用该字段，无影响） |

其余插件名与配置字段两边一致——已逐项核对 `dsh-v0.1.5-rc.3` 的源码：
`persona.includeRuntimeContext`、`compaction-basic` 的 `thresholdRatio` / `retainRatio` /
`modelPolicies` / `summarizationProvider`、`tool-result-pruner` 的
`thresholdChars` / `headChars` / `tailChars` 在 0.1.5 全部存在。

## 为什么不用 git branch

两份文件**格式不同但内容同源**，而且互不冲突——一份是 profile patch，一份是预设目录里的文件，
可以共存于同一分支。用目录区分的好处是：核心策略改一次就同步一次。

开 branch 意味着每次改动都要维护两份并手动保持同步，是纯负债。
真到了两份文件的**内容**开始分叉那天（而不只是格式不同），再考虑分支。
