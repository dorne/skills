---
name: commandcode-goat-models
description: 实时抓取 Command Code GOAT Plan 模型清单并匹配 API Model ID，再从 models.dev 按供应商优先级（opencode-go/openrouter/openai/anthropic）、映射表与规范化规则匹配模型元数据，把匹配情况表落盘为 result.md、结果 JSON 落盘为 models.json。Use when checking GOAT models, Command Code model updates, or generating models.dev metadata for Command Code model IDs.
---

# Command Code GOAT Plan 模型核对

实时抓取官网 GOAT 清单并匹配 `Model ID`，再从 `models.dev` 按优先级、映射表与规范化规则匹配元数据，落盘 `result.md` 与 `models.json`；全程不改动任何本地配置。

## 通用约定

- **排序**：所有清单与输出按 `Name` 转小写后的 ASCII 码序升序（不按 `Model ID`、厂商或官网顺序）。
- **`name`**：结果里的 `name` 一律用官网原始 `Name` 覆盖（映射/规范化可能命中丢掉后缀的 models.dev 记录）。
- **属性裁剪**：每条匹配记录删除 `id` 与 `canonical_model_id`。
- **未匹配**：`Name` 找不到 `Model ID`，或找不到任何供应商记录，均记 `未匹配`、表格状态 `❌`。
- **写盘**：中间文件只写步骤 4 的临时目录；产物 `result.md`、`models.json` 写当前工作目录。

## 步骤

### 1. 抓取与提取

`webfetch` `https://commandcode.ai/docs/plans/goat`：

- 取 `Models included` 小节链接文字为模型 `Name`（如 `[Gemini 3.8 Flash](/models/...)`）。
- 删除促销/定价角标（如 `Free`、`-50%`、`-98%`、`Off-peak`）；`Usage limits`、`Premium models`、`New models`、`Older models` 等表中的重复模型一律忽略，每模型只计一次。
- 校验：提取数量须等于标签 `GOAT plan N` 的 `N`；无该标签则提示页面已改版，需更新规则。

### 2. 匹配 Model ID

`webfetch` `https://commandcode.ai/docs/reference/cli/models`：按 `Name` 精确匹配 `Model ID`，禁止凭名称、slug 或记忆猜测。

### 3. 整理模型清单

把步骤 1、2 的结果整理为 `| Name | Model ID |` 工作清单供步骤 5、6 引用；不落盘、不在回复中输出。

### 4. 准备临时目录并下载 models.dev 数据

在系统临时目录（`$env:TEMP` / `$TMPDIR`）新建 `commandcode-goat-models-<时间戳>` 文件夹，下载 `https://models.dev/api.json`（约 5 MB）保存其中；`webfetch` 结果过大被截断时改用 shell 下载（`Invoke-WebRequest` / `curl`）。

### 5. 匹配模型

供应商优先级 `["openrouter", "opencode-go", "openai", "anthropic"]`（顺序即优先级，可按需增删）。对步骤 3 的每个 `Model ID` 确定「查找键」（用于在供应商模型表中检索的键）：命中下方映射表（结构 `{ "Model ID": "目标键" }`，可直接增删）则取「目标键」，否则取该 `Model ID` 本身。

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
  "claude-sonnet-5-5": "anthropic/claude-sonnet-5.5",
  "claude-haiku-5-5": "anthropic/claude-haiku-5.5"
}
```

拿到查找键后按优先级逐供应商匹配，每个供应商先精确键、未中再规范化键；命中即采用该记录属性、记录供应商与「匹配键」（供应商侧实际命中的键），并停止后续匹配：

- 规范化：转小写；`/` 前厂商按别名归一（`minimaxai`→`minimax`、`xai`→`x-ai`、`mistral`→`mistralai`、`zai-org`→`z-ai`）；去掉结尾 `:free`、`-free`、`-paid`、`-0`；冲突时优先保留不带 `:free` 的键。
- 命中后的 `name` 与属性裁剪按「通用约定」处理；全部供应商未命中则跳过该模型，但仍计入步骤 6 汇总。

### 6. 落盘结果

把步骤 5 的结果写入当前工作目录，回复只回执这两个文件路径：

1. **匹配情况** → `result.md`，覆盖步骤 3 全部 `Model ID` 的 Markdown 表格：

| Model ID | Name | 状态 | 供应商 | 匹配键 | 方式 |
| --- | --- | --- | --- | --- | --- |
| `<Model ID>` | 原始官网 `Name` | `✅`/`❌` | 供应商 id（如 `openrouter`、`openai`）或 `—` | 供应商侧命中的键或 `—` | `映射`/`精确`/`规范化` 或 `—` |

- 供应商：该模型取自哪个提供商；方式：命中映射表记 `映射`，否则精确键命中记 `精确`、归一命中记 `规范化`，未匹配记 `—`。

2. **结果 JSON** → `models.json`，仅含匹配到的模型，`models` 的键按同一顺序排列：

```json
{
  "models": {
    "Model ID": {
      "...": "匹配到的属性（已删除 id、canonical_model_id，name 为原始 Name）"
    }
  }
}
```

### 7. 清理临时文件

落盘完成后，彻底删除步骤 4 新建的 `commandcode-goat-models-<时间戳>` 文件夹及其全部内容。
