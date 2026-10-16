---
name: commandcode-plan-set
description: 先下载 models.dev 的 api.json 到 cache/，再调用 commandcode-plan-sync 取套餐清单匹配结果，把状态为 ❌ 的套餐模型按 canonical 优先级 openrouter → vercel → anthropic 过滤 api.json 后匹配，命中的写入该 canonical 对应的 opencode 服务商 models。Use when applying Command Code plan models into the local opencode provider config, wiring up missing plan models, or syncing plan model definitions into opencode.json.
---

# Command Code 套餐模型写入 opencode 配置

把套餐里本地配置还没有（`❌`）的模型，按 `canonical` 过滤后的 models.dev 元数据匹配，写成标准条目补进全局 opencode 配置；全程只新增，不改写已有条目。

所有落盘产物都写进当前工作目录的 `cache/` 子目录（不存在则先创建），不在工作目录根下留文件。

## 参数

- 套餐标识：`go`、`goat`、`pro` 之一；缺省为 `goat`。
- 只处理传入的这一个套餐。

## 步骤

### 1. 先下载 api.json

下载 `https://models.dev/api.json`，落盘到当前工作目录 `cache/api.json`（存在即覆盖）。这是本技能的唯一上游元数据来源，不要改用记忆或其它来源；第 4 步会按 canonical 把它拆成 `cache/api.<canonical>.json`，后续候选一律从这些拆分文件读。

因为 JSON 没有注释语法，在该文件 JSON 顶层加一个字段 `"_writeBy": "commandcode-plan-set"` 作为产出技能标记；它不参与读取，技能取候选只读第 4 步拆出的 `cache/api.<canonical>.json` 的 `models`。

### 2. 取套餐清单与匹配状态

- 套餐为 `goat`（默认）：调用 `commandcode-plan-sync`，取它写出的 `cache/sync-plan-report.md`。
- 套餐为 `go`、`pro`：调用 `commandcode-plan-get`（传入该套餐）得到 `cache/models-<套餐>.md`，再按 `commandcode-plan-sync` 的匹配规则（`modelID` 与 `name` 双全等）与本地配置核对，得到同格式的 `cache/result-<套餐>.md`，其最后一行同样写 `write by commandcode-plan-set`。

两种方式得到的表都含四列：`Name`、`Model ID`、状态（`✅`/`❌`）、服务商、配置模型键。

### 3. 读本地 opencode 配置里的 canonical

读取全局配置 `~/.config/opencode/opencode.json`（Windows `%USERPROFILE%\.config\opencode\opencode.json`），取键名以 `commandcode` 开头的服务商，记录每个的 `canonical`（如 `commandcode`→`openrouter`、`commandcode-anthropic`→`anthropic`、`commandcode-extensions`→`vercel`）。

不要拿 api.json 里无关服务商来匹配；只用这些 `canonical` 值对应的 `cache/api.<canonical>.json` 候选集合。

### 4. 按 canonical 拆分 api.json 并取候选

先把配置里出现的每个 `canonical` 值，从 `api.json[<canonical>]` 原样落盘为 `cache/api.<canonical>.json`（存在即覆盖，顶层加与 `api.json` 相同的 `"_writeBy": "commandcode-plan-set"` 标记）。该文件只含这一个 canonical 自己的 provider 元数据（`id`、`npm`、`api`、`doc` 等）与 `models`，不含其它 canonical 的任何内容。它是正式的候选来源，和 `api.json` 一样参与流程计算，后面第 5 步的匹配只读它、不再回读 `api.json`。

候选集合取 `cache/api.<canonical>.json` 的 `models`，候选里每条包含键名（`models` 下的 key）、`id`、`name`。当前配置里是 `openrouter`、`vercel`、`anthropic` 三个；若以后新增了别的 canonical，把它排在优先级表末位（`anthropic` 之后）。

### 5. 对 ❌ 条目匹配

只看状态为 `❌` 的行，逐条按下述规则在 `cache/api.<canonical>.json` 的候选里找：

- canonical 优先级固定为 `openrouter` → `vercel` → `anthropic`：从高到低依次在对应候选集合里找，第一个命中的 canonical 就是目标，命中后不再看后面的 canonical。只要 openrouter 里有这条模型，就必须写入 openrouter 对应的服务商，不能因为 vercel 的名字碰巧更接近而落到 vercel。
- 例外：`Name` 以 `Claude` 开头 → 只在 `anthropic` 候选里找（Claude 走 `@ai-sdk/anthropic` 的 `commandcode-anthropic`，不能被 openrouter / vercel 里的 `anthropic/*` 镜像抢走）。
- 单个 canonical 内的匹配顺序：
  1. 先比 `id`：候选 `id` 与 `Model ID` 忽略大小写后全等。
  2. 该 canonical 内没有 id 命中，再比 `name`：候选 `name` 与 `Name` **归一化后**全等。归一化指转小写，并删掉空格、`-`、`_`、`.`、`:`、`+`——models.dev 与官网在大小写和标点上经常不一致（如官网 `MiniMax M2.5` 对应 openrouter 的 `MiniMax-M2.5`、vercel 的 `MiniMax M2.5`），严格全等会把本应属 openrouter 的模型误判到 vercel。
- 命中唯一一条：记住该候选的键名作为配置键，并记住这条命中的 canonical 值。
- 没命中，或同一 canonical 内命中多条无法判定：跳过并计入「未匹配」，不要凭名称或记忆猜。

### 6. 组织并写入

对每条命中，按下列格式准备数据（`Model ID`、`Name` 用官网原文）：

```json
"<命中候选的键名>": {
  "modelID": "<Model ID>",
  "name": "<Name>"
}
```

写入第 5 步命中的那个 canonical 值所对应的 opencode 服务商（`canonical` 相等的那个 `commandcode*`）的 `models` 下：

- 排序规则：条目在 `models` 内按 `name` 转小写后的 ASCII 码序升序排列。目标服务商现有的 `models` 本来就是这个顺序，所以每条新增要插到它自己的位置——排在第一个「小写 `name` 不小于它」的既有条目之前——不能整块堆到开头或末尾。同一服务商本次有多条新增时，先按同一规则把它们排序，再依次插入。
- 比较方式：小写 `name` 的逐字符码元比较（JS 里 `a < b` 那种），不要用 `localeCompare`，也不要按 `Model ID` 或配置键排序——后两者的顺序和配置里现有的不一样，插进去会把既有顺序看起来打乱。
- 只新增键；该服务商 `models` 下已有同名键则跳过并计入「已存在」，不覆盖、不改写。
- 不改动状态为 `✅` 的条目，也不改动 `commandcode*` 之外的服务商。
- 保持 JSON 缩进 2 空格，其余内容不变。写回前先自查一遍每个被改动的服务商其 `models` 仍按上述顺序排列，不满足就修正后再写。

写回前先把 `opencode.json` 备份为 `opencode.json.bak-<yyyyMMdd-HHmmss>`，再整体写回。

### 7. 落盘报告 cache/set-plan-report.md

在当前工作目录 `cache/` 下写 `cache/set-plan-report.md`（存在即覆盖），把这次执行的判断过程完整留下来，便于事后核对每一条为什么写、为什么不写：

````markdown
# Command Code <套餐> 套餐配置写入报告

- 执行时间：<yyyy-MM-dd HH:mm:ss>
- 套餐：<go|goat|pro>
- 数据来源：`cache/sync-plan-report.md`（<条目总数> 条）、`cache/api.json`（https://models.dev/api.json，下载于 <yyyy-MM-dd HH:mm:ss>）、匹配实际读取的 `cache/api.<canonical>.json`
- 目标配置：`<opencode.json 绝对路径>`，备份 `<opencode.json.bak-yyyyMMdd-HHmmss>`

## 1. canonical 映射与候选规模

匹配优先级：`openrouter` → `vercel` → `anthropic`（`Claude` 开头的条目只用 `anthropic`）。

| opencode 服务商 | canonical | cache/api.<canonical>.json 候选模型数 |
| --- | --- | --- |
| `<服务商 id>` | `<canonical>` | `<n>` |

## 2. 新增写入

| Name | Model ID | canonical | 目标服务商 | 配置键 | 匹配方式 |
| --- | --- | --- | --- | --- | --- |
| `<Name>` | `<Model ID>` | `<canonical>` | `<服务商 id>` | `<键名>` | `id`/`name` |

无新增时该表写一行「（无）」。

## 3. 已存在，跳过

| Name | Model ID | 目标服务商 | 配置键 |
| --- | --- | --- | --- |

无跳过时该表写一行「（无）」。

## 4. 未匹配

| Name | Model ID | 搜索过的 canonical |
| --- | --- | --- |

无未匹配时该表写一行「（无）」。

## 5. 未改动

- 状态 `✅` 的条目 <n> 条：保持原样，不重写、不改元数据。
- `commandcode*` 之外的服务商：未读取、未改动。

## 汇总

新增 <x> 条 / 已存在跳过 <y> 条 / 未匹配 <z> 条（套餐共 <t> 条，已配置 <n> 条）
````

表里每一行都要有明确的匹配依据：`匹配方式` 写实际命中的是 `id`（忽略大小写全等）还是 `name`（归一化全等）；`未匹配` 那栏按 `openrouter` → `vercel` → `anthropic` 的顺序写清这条搜过哪些 canonical，避免事后看不出是「api.json 里确实没有」还是「规则没搜到」。第 2、3、4 张表都按 `Name` 转小写后的 ASCII 码序升序排列，方便和配置里写成同一顺序对照。汇总行之后于文件最后一行单独写一行纯文本 `write by commandcode-plan-set`，作为产出技能标记；该行不参与表格与数据解析。

### 8. 回执

回复只给一行回执：`cache/set-plan-report.md` 的路径，以及新增 / 已存在跳过 / 未匹配 三个条数；不在回复里输出报告或配置内容。
