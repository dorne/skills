---
name: commandcode-goat-models
description: 实时读取 Command Code GOAT Plan 模型清单，匹配 API Model ID，并按指定供应商优先级从 models.dev 获取模型元数据，生成匹配报告 result.md 与结果文件 models.json。Use when checking GOAT models, Command Code model updates, or generating models.dev metadata for Command Code model IDs.
---

# Command Code GOAT Plan 模型核对

从 Command Code 官网获取 GOAT Plan 模型及其 API Model ID，再从 `models.dev` 匹配模型元数据。最终在当前目录生成 `result.md` 和 `models.json`。除这两个结果文件外，不修改或生成其他工作区文件；临时下载数据存放于操作系统临时目录，并在结束时清理。

## 数据源

- GOAT Plan 模型清单：<https://commandcode.ai/docs/plans/goat>
- Command Code API Model ID：<https://commandcode.ai/docs/reference/cli/models>
- 模型元数据：<https://models.dev/api.json>

## 流程

### 1. 获取并校验 GOAT Plan 模型

读取 GOAT Plan 页面 `Models included` 小节的模型链接文字作为原始 `Name`。

- 移除模型名旁的促销或定价角标，例如 `Free`、`-50%`、`-98%`、`Off-peak`；不把角标纳入 `Name`。
- 只统计 `Models included` 中的模型。其他分区（如 `Usage limits`、`Premium models`、`New models`、`Older models`）不用于补充清单或重复计数。
- 以 `GOAT plan N` 标签中的 `N` 校验提取数量。页面没有该标签或数量不一致时，停止处理并说明页面结构已变化；不要继续生成结果文件。

### 2. 将模型名称映射到 API Model ID

读取 API Model ID 页面，按原始 `Name` 精确查找对应的 `Model ID`。不得根据模型名、slug 或记忆推断 ID。

- 找到时记录官网给出的 `Model ID`。
- 找不到时保留原始 `Name`，并将该项标记为「ID 未找到」。
- 按 `Name` 的小写字符串进行 ASCII 升序排序；后续清单、报告和 JSON 均沿用此顺序，不再另行排序。

### 3. 获取 models.dev 数据

在操作系统临时目录中新建唯一目录 `commandcode-goat-models-<时间戳>`，下载并保存 `https://models.dev/api.json`。若 `webfetch` 内容被截断，改用 shell 下载（例如 `Invoke-WebRequest` 或 `curl`）。此临时目录仅用于存放下载数据及必要的临时文件。

### 4. 按优先级匹配模型元数据

供应商优先级如下，命中后立即停止，不再查询后续供应商：

```text
opencode-go → openrouter → openai → anthropic
```

对每个已找到 `Model ID` 的模型，按以下规则确定查找键：

1. 若原始 `Model ID` 命中下方映射表，以映射值作为查找键；否则以原始 `Model ID` 作为查找键。
2. 按供应商优先级逐一查找。在每个供应商内，先比较完整键是否精确相等；未命中时，再按下述规范化规则比较。
3. 命中后记录供应商 ID、供应商模型表中的实际键及模型属性。

映射表的键是 Command Code 的原始 `Model ID`，值是 models.dev 查找键：

```json
{
  "deepseek/deepseek-v4-flash-fast": "deepseek/deepseek-v4-flash",
  "deepseek/deepseek-v4.1-flash-fast": "deepseek/deepseek-v4.1-flash",
  "moonshotai/Kimi-K2.7-Code-Highspeed": "moonshotai/kimi-k2.7-code",
  "Qwen/Qwen3.8-Max": "qwen/qwen3.8-max-0902",
  "zai-org/GLM-5.2-Fast": "z-ai/glm-5.2",
  "gpt-5.6-sol": "openai/gpt-5.6-sol",
  "gpt-6-luna": "openai/gpt-6-luna",
  "gpt-5.6-luna": "openai/gpt-5.6-luna",
  "claude-sonnet-5-5": "anthropic/claude-sonnet-5.5"
}
```

规范化仅用于精确键查找失败后的回退比较：

- 将键转为小写。
- 仅对第一个 `/` 前的供应商标识应用别名：`minimaxai` → `minimax`、`xai` → `x-ai`、`mistral` → `mistralai`、`zai-org` → `z-ai`。
- 移除键末尾的一个限定符：`:free`、`-free`、`-paid` 或 `-0`。
- 若多个供应商键规范化后相同，优先选择不带 `:free` 的键；如仍有多个候选，按原始键 ASCII 升序选择，确保结果确定。

匹配来源和方式分别记录：

- 来源是命中的供应商 ID。
- 若原始 `Model ID` 使用了映射表，方式记为 `映射`，不论映射后的查找是精确还是规范化命中。
- 未使用映射表时，完整键相等记为 `精确`；仅规范化后相等记为 `规范化`。
- 所有供应商均未命中时，记为「元数据未找到」。

生成 JSON 属性时，复制命中模型的全部元数据，但删除顶层 `id` 和 `canonical_model_id`，并将顶层 `name` 覆盖为 Command Code 页面提取的原始 `Name`。不得沿用 models.dev 的 `name`，以免映射或规范化命中时丢失官网名称后缀或格式。

### 5. 生成结果文件

在当前目录写入且仅写入以下两个结果文件。两者的模型顺序均沿用步骤 2 的排序。

#### `result.md`

文件内容为覆盖全部 GOAT 模型的 Markdown 表格，列顺序固定如下：

| Model ID | Name | 状态 | 供应商 | 匹配键 | 方式 |
| --- | --- | --- | --- | --- | --- |

字段规则：

- `Model ID`：API ID 查找成功时写原始 ID；查找失败时写 `未匹配`。
- `Name`：始终写官网提取的原始名称。
- `状态`：只有元数据匹配成功时写 `✅`；ID 或元数据任一未找到时写 `❌`。
- `供应商`、`匹配键`、`方式`：元数据匹配成功时按步骤 4 填写；未成功时均写 `—`。

表格前后不添加其他说明文字。

#### `models.json`

写入合法 JSON，顶层结构为 `{"models": {...}}`。仅包含元数据匹配成功的模型；每个键为原始 Command Code `Model ID`，每个值为步骤 4 处理后的模型属性。键的顺序沿用步骤 2 的排序。未找到 ID 或未匹配到元数据的模型不写入 JSON。

```json
{
  "models": {
    "<原始 Model ID>": {
      "name": "<Command Code 原始 Name>",
      "...": "其他元数据（已删除 id 与 canonical_model_id）"
    }
  }
}
```

### 6. 清理临时数据

确认 `result.md` 与 `models.json` 已成功写入当前目录后，删除本次创建的 `commandcode-goat-models-<时间戳>` 临时目录及其内容。不得删除或覆盖其他目录中的文件。
