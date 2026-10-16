---
name: commandcode-plan-get
description: 按套餐实时抓取 Command Code 模型清单（Name + Model ID），支持 go / goat / pro 三个套餐，按 Name 排序后落盘为当前工作目录 cache/models-<套餐>.md。Use when checking Command Code plan models, Go/GOAT/Pro model lists, or Command Code model updates.
---

# Command Code 套餐模型清单

按指定套餐实时抓取官网清单与 CLI 参考页，整理每个模型的 `Name` 与 `Model ID`，落盘 `cache/models-<套餐>.md`；全程不改动任何本地配置。

所有落盘产物都写进当前工作目录的 `cache/` 子目录（不存在则先创建），不在工作目录根下留文件。

## 参数

- 套餐标识：`go`、`goat`、`pro` 之一；缺省为 `goat`。
- 仅处理传入的这一个套餐，一次只生成一份清单。
- 套餐与页面、产物、数量标签的对应关系：

| 套餐 | 页面 | 产物 | 数量标签 |
| --- | --- | --- | --- |
| `go` | `https://commandcode.ai/docs/plans/go` | `cache/models-go.md` | `Go plan N` |
| `goat`（默认） | `https://commandcode.ai/docs/plans/goat` | `cache/models-goat.md` | `GOAT plan N` |
| `pro` | `https://commandcode.ai/docs/plans/pro` | `cache/models-pro.md` | `Pro plan N` |

## 步骤

### 1. 抓取模型 Name

`webfetch` 上表对应套餐的页面：

- 取 `Models included` 小节主表（列为 `Model | Context | Intelligence | Input | Output | Cache read | Cache write | Caps`）每行的模型链接文字为 `Name`（如 `[Gemini 3.8 Flash](/models/...)`）。
- 删除促销/定价角标（如 `Free`、`-50%`、`-98%`、`Off-peak`）；`New models`、`Premium models`、`Older models`、`Usage limits` 等后续表中的重复模型一律忽略，每模型只计一次。
- 校验：提取数量须等于该页数量标签 `N`；无该标签或数量不符则提示页面已改版，需更新规则。

### 2. 匹配 Model ID

`webfetch` `https://commandcode.ai/docs/reference/cli/models`：按 `Name` 精确匹配 `Model ID`，禁止凭名称、slug 或记忆猜测。该页与套餐无关，三个套餐共用。

### 3. 落盘 cache/models-<套餐>.md

把该套餐全部 `Name` 及对应 `Model ID` 写入当前工作目录 `cache/models-<套餐>.md`，Markdown 表格，按 `Name` 转小写后的 ASCII 码序升序排列（不按 `Model ID`、厂商或官网顺序）：

| Name | Model ID |
| --- | --- |
| `<官网原始 Name>` | `<Model ID>` |

- 未找到 ID 的保留该行，`Model ID` 填 `—`。
- 表格之后于文件最后一行单独写一行纯文本 `write by commandcode-plan-get`，作为产出技能标记；该行不参与表格与数据解析。
- 只写该套餐对应的这一个文件，不覆盖其它套餐产物。
- 回复只回执该文件路径，不在回复中输出清单内容。
