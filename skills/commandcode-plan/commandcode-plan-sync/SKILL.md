---
name: commandcode-plan-sync
description: 先以默认值调用 commandcode-plan-get 技能得到 GOAT 套餐模型清单，再把清单中每个 Name 与 Model ID 同本地全局 opencode 配置里 commandcode 开头的服务商模型做全等匹配，最后把匹配情况表落盘为当前工作目录 cache/sync-plan-report.md。Use when syncing Command Code plan models against the local opencode provider config, or checking which plan models are wired up locally.
---

# Command Code 清单与本地 opencode 配置核对

先取套餐清单，再与本地 opencode 配置里 `commandcode` 开头的服务商模型逐条全等匹配，落盘 `cache/sync-plan-report.md`；全程只读配置，不改动任何本地配置。

所有落盘产物都写进当前工作目录的 `cache/` 子目录（不存在则先创建），不在工作目录根下留文件。

## 步骤

### 1. 取套餐清单

调用 `commandcode-plan-get` 技能，不传套餐（使用默认值 `goat`），拿到当前工作目录的 `cache/models-goat.md`。

读取该文件，得到全部条目的 `Name` 与 `Model ID`，并保持其既有顺序（即 `Name` 升序）。

### 2. 读本地 opencode 配置

读取全局配置 `~/.config/opencode/opencode.json`（Windows 为 `%USERPROFILE%\.config\opencode\opencode.json`），若该路径不存在则用 `%USERPROFILE%` 下的同路径推断并提示用户确认。

取 `providers` 中键名以 `commandcode` 开头的服务商（如 `commandcode`、`commandcode-anthropic`、`commandcode-extensions`），收集其 `models` 下每个模型的：

- 所属服务商 id（`providers` 下的键）
- 模型键（`models` 下的键）
- `modelID`
- `name`

### 3. 全等匹配

对步骤 1 的每个 `Model ID` 与 `Name`，在上一步候选里找同时满足以下两条的记录：

- 候选的 `modelID` 与 `Model ID` 全等
- 候选的 `name` 与 `Name` 全等

「全等」指逐字符完全相等、区分大小写，不做大小写归一、别名映射、模糊或前缀匹配；两条必须同时成立，任一不成立即不算命中。

- 命中：记录其服务商 id 与模型键；命中多个时用 `、` 连接后全部列出。
- 未命中：状态记 `❌`，服务商与模型键记 `—`。

### 4. 落盘 cache/sync-plan-report.md

把匹配结果写入当前工作目录 `cache/sync-plan-report.md`，覆盖步骤 1 的全部条目，顺序不变：

| Name | Model ID | 状态 | 服务商 | 配置模型键 |
| --- | --- | --- | --- | --- |
| `<官网原始 Name>` | `<Model ID>` | `✅`/`❌` | `<服务商 id>` 或 `—` | `<models 下的键>` 或 `—` |

表格下方补一行汇总：`匹配 x/y`（y 为清单条目总数，x 为命中数）。

- 汇总行之后于文件最后一行单独写一行纯文本 `write by commandcode-plan-sync`，作为产出技能标记；该行不参与表格与数据解析。
- 只列出清单中的模型，不额外列出配置里有、清单里没有的模型。
- 回复只回执该文件路径，不在回复中输出清单内容。
