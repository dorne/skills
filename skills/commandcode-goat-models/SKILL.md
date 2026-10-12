---
name: commandcode-goat-models
description: 实时抓取 Command Code GOAT Plan 模型清单并匹配 API Model ID，再从 models.dev 按供应商优先级（opencode-go/openrouter/openai/anthropic）、映射表与规范化规则匹配模型元数据，输出匹配情况表与结果 JSON。Use when checking GOAT models, Command Code model updates, or generating models.dev metadata for Command Code model IDs.
---

# Command Code GOAT Plan 模型核对

实时抓取官网 GOAT 模型清单，匹配 `Model ID`，再从 `models.dev` 按供应商优先级、映射表与规范化规则为每个模型匹配元数据，最后输出「匹配情况」表与结果 JSON。全过程不改动任何本地配置。

## 步骤

### 1. 抓取与提取

`webfetch` `https://commandcode.ai/docs/plans/goat`：

- 取 `Models included` 小节链接文字为模型 `Name`（如 `[Gemini 3.8 Flash](/models/...)`）。
- 删除促销/定价角标（如 `Free`、`-50%`、`-98%`、`Off-peak`）；忽略 `Usage limits`、`Premium models`、`New models`、`Older models` 等表中的重复模型，每模型只计一次。
- 校验：提取数量须等于标签 `GOAT plan N` 的 `N`；无该标签则提示页面已改版，需更新规则。

### 2. 匹配 Model ID

`webfetch` `https://commandcode.ai/docs/reference/cli/models`：

- 按 `Name` 精确匹配 `Model ID`，禁止凭名称、slug 或记忆猜测；匹配不到标记 `未匹配`（步骤 6 表格中该行 Model ID 填 `未匹配`、状态 `❌`）。
- 排序：`Name` 转小写后按 ASCII 码序升序（不按 Model ID、厂商或官网顺序）。本技能所有输出都按此排序。

### 3. 整理模型清单

将步骤 1、2 的结果整理为两列 `| Name | Model ID |` 的工作清单，供步骤 5、6 引用；该清单是内部产物，不在回复中单独输出（回复只含步骤 6 的两部分）。

### 4. 准备临时目录并下载 models.dev 数据

- 在操作系统临时文件夹（如 `$env:TEMP` / `$TMPDIR`）新建 `commandcode-goat-models-<时间戳>` 文件夹；本技能所有写入操作均只写入该文件夹。
- 获取 `https://models.dev/api.json`（约 5 MB）并保存到该文件夹；若 `webfetch` 结果过大被截断，改用 shell 下载（如 `Invoke-WebRequest` / `curl`）。

### 5. 匹配模型

供应商优先级：`["opencode-go", "openrouter", "openai", "anthropic"]`（顺序即优先级，可按需增删）。

对步骤 3 的每个 `Model ID`，先确定查找键（优先级从高到低）：

1. **映射表优先**：命中下方映射表（结构 `{ "Model ID": "目标键" }`，直接在本文件内增删即可）时，以「目标键」为查找键；随后仍按供应商优先级与精确/规范化规则匹配它。

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

2. **规则匹配**：未命中映射表时，用当前 `Model ID` 本身作为查找键。

「查找键」是用于在供应商模型表中检索的键；步骤 6 表格的「匹配键」填供应商侧实际命中的键。

拿到查找键后，按供应商优先级依次匹配；同一供应商先精确键匹配，未中再按键规范化匹配：

- 规范化规则：转小写；`/` 前厂商标识按别名表归一（`minimaxai`→`minimax`、`xai`→`x-ai`、`mistral`→`mistralai`、`zai-org`→`z-ai`）；去掉结尾限定符 `:free`、`-free`、`-paid`、`-0`。规范化键冲突时优先取不带 `:free` 的键。
- 命中即采用该模型属性（记录时删除 `id` 和 `canonical_model_id`），并记录来源供应商，不再尝试后续供应商。
- **`name` 永远以原始 Model Name 为准**：无论精确、映射还是规范化命中，结果里的 `name` 一律覆盖为步骤 2 匹配到的 Command Code 原始 `Name`，不保留 models.dev 的名字。因为映射/规范化会指向另一条 models.dev 记录，其 `name` 可能丢掉 `Fast`、`HighSpeed`、`0902` 等后缀，或与官网名称大小写/空格不一致（例如 `deepseek/deepseek-v4.1-flash-fast` 会错误地拿到 `DeepSeek V4.1 Flash`，此时必须用官网原始名 `DeepSeek V4.1 Flash Fast` 覆盖）。
- 所有供应商均未命中：跳过，但仍记录该 `Model ID`，供步骤 6 汇总。

### 6. 输出结果

除步骤 4 的临时文件外不再生成产物文件，直接在回复中输出，分两部分；**两部分均按 `Name` 转小写 ASCII 升序排列**（同步骤 2）：

1. **匹配情况**：覆盖步骤 3 全部 `Model ID` 的 Markdown 表格：

| Model ID | Name | 状态 | 供应商 | 匹配键 | 方式 |
| --- | --- | --- | --- | --- | --- |
| `<Model ID>` | 该 `Model ID` 对应的原始 `Name`（取自官网，始终与结果 JSON 的 `name` 一致） | `✅` / `❌` | 命中的供应商 id，如 `openrouter`、`openai`；未匹配 `—` | 供应商侧实际命中的键；未匹配 `—` | `映射` / `精确` / `规范化`；未匹配 `—` |

- 状态：`✅` 匹配、`❌` 未匹配。
- 供应商：命中的供应商 id，说明该模型取自哪个提供商。
- 方式：`映射`（经映射表命中）、`精确`（原键精确命中）、`规范化`（归一后命中）。判定优先级：命中映射表即记 `映射`；未命中映射表时，精确键命中记 `精确`，归一命中记 `规范化`。

2. **结果 JSON**：仅包含匹配到的模型，`models` 的键按同一顺序排列；每条属性一律删除 `id` 与 `canonical_model_id`，且 `name` 一律为步骤 2 的原始 `Name`（见步骤 5），格式如下：

```json
{
  "models": {
    "Model ID": {
      "...": "匹配到的属性（已删除 id、canonical_model_id）"
    }
  }
}
```

### 7. 清理临时文件

输出完成后，彻底删除步骤 4 新建的 `commandcode-goat-models-<时间戳>` 文件夹及其全部内容。
