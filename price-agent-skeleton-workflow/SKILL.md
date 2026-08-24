---
name: price-agent-skeleton-workflow
description: 当用户操作 PriceStudio 同款判定提示词（骨架）的新建、查询、修改、优化、增删、保存、发布，或从页面发起单条 Badcase 分析时触发。Badcase 只查询一次验证结果并由模型自由分析，不支持整任务、分页、续批或自动修改 Prompt。也处理对紧邻提示词提案的确认、同意、保存、创建草稿、拒绝或取消。
---

# PriceStudio 提示词工作流

## 入口取值

- 进入 Skill 时只读取本轮业务上下文中已展开的 `promptVersionId`，绑定为 `currentPromptVersionId`。
  历史消息、模型记忆、版本名称、提案和工具返回的 Prompt ID 不得参与赋值、比较或纠正。
- `promptVersionId` 缺失、为空、为 0 或仍为模板占位符时按 `0` 处理；非空但不是非负整数时停止。
- 用户文字指定“最新/线上/归档/草稿/ID/名称/版本”时，不按文字查询或切换版本，只使用本轮业务上下文。
- 只使用真实上下文和 MCP 响应；禁止编造 ID、规则、映射、样本、正文、Diff 或工具结果。

## 意图边界

- “新建/创建/初始化/优化/改善/修改/完善/重写提示词、骨架或规则”都属于提示词写操作；禁止在入口
  追问方向，直接按 `promptVersionId` 路由到 EDIT 或 INITIALIZE。
- EDIT 内部四类分支（精确修改直达、当前规则同步、格式重构硬分支、无方向普通修改）只由
  [edit-all-in-one.md](references/edit-all-in-one.md) 根据用户完整表达判定和执行；入口不得把“规则”
  预判为泛化对象，也不得自行展开工具调用或候选生成。
- Badcase 只接受本轮业务上下文 `validationTaskId>0 && validationCaseId>0`。任一缺失、为 0 或占位符时，
  直接使用路由表中的固定提示，不读取 workflow。禁止从用户文字、input 或 `#数字` 提取、补写或覆盖
  Badcase 上下文。

## 路由表

| 请求或状态 | 必须完整读取并执行 |
|---|---|
| Badcase 意图且上下文 `validationTaskId` 或 `validationCaseId` 缺失、为 0 或占位符 | 不读取 workflow，不调用 MCP；只回复：“请通过页面的「一键分析 Badcase」按钮发起分析，当前不支持手动填写 Badcase、Case ID 或验证任务 ID。” |
| Badcase 意图且上下文 `validationTaskId>0 && validationCaseId>0` | [badcase-single-workflow.md](references/badcase-single-workflow.md) |
| 仅查询当前提示词 | 不读取 workflow；完整执行本文件下方「当前提示词查询」小节 |
| 紧邻上一条有效提案后要求展开/查看/展示完整提示词 | 只读取并执行 [shared-steps.md](references/shared-steps.md) `[S1]` 的展开全文分支；不恢复或重跑原提案 workflow |
| 确认/同意/保存/创建草稿 | 直接执行 [shared-steps.md](references/shared-steps.md) `[S2]`；所有门禁和错误均由 `[S2]` 处理 |
| 任意提示词写操作且本轮 `promptVersionId>0` | [edit-all-in-one.md](references/edit-all-in-one.md) |
| 任意提示词写操作且本轮 `promptVersionId` 缺失、为 0 或占位符 | [initialize-all-in-one.md](references/initialize-all-in-one.md) |

## 执行纪律

- 选定 workflow 后完整读取路由表中的目标文件，并严格按目标文件顺序执行；只跳过 workflow 明确允许
  跳过的步骤。Badcase、INITIALIZE、EDIT 均以路由表中的单个目标文件为执行权威。
- 选定 workflow 后，禁止用自由回答、模型自检、占位数据或“流程骨架”代替实际执行。凡 workflow 要求
  读取文件或调用 MCP，首条可见回复前必须存在本轮对应的真实读取和调用记录。
- 必须从目标 workflow 的第一步按顺序执行，不得因入口出现工具名、用户要求优化或已有正文而直接跳到
  候选生成、格式校验或提案展示；候选格式、校验时机、失败处理和重试次数只以目标 workflow 为准。
- `promptVersionId>0` 命中 EDIT 后禁止再查找 INITIALIZE workflow；当前正文残缺不是切换流程或要求用户补正文的理由，
  必须由 EDIT 的格式重构硬分支处理。
- INITIALIZE 以及 EDIT 中需要加载真实规则/关系的分支，必须以本轮原始规则响应、关系响应、目标 workflow
  的比价项转换账本或关系过滤记录完成业务门禁；`tool_validate_prompt_skeleton` 只做格式校验，不能替代业务一致性检查。

## 当前提示词查询

仅查询当前提示词时，沿用入口绑定的 `currentPromptVersionId`：`currentPromptVersionId>0` 时调用
`tool_query_prompt_skeleton(ruleGroupId=上下文.ruleGroupId, promptVersionId=currentPromptVersionId, versionName="", queryOnline=false, queryLatest=false, operator=上下文.operator)`；
缺失、为 0 或占位符时报告不存在当前提示词；负数或非数字时报参数非法。查询成功后只展示当前提示词
版本信息和正文，不生成提案、不调用校验、不写入。用户要求最新、线上、归档、草稿、ID、名称或版本时，
只回复：“当前操作仅使用本轮业务上下文中的提示词版本，不支持通过用户文字指定或切换版本。”

## 提案、展开、确认

凡产生提示词提案的 INITIALIZE 或 EDIT，首条可见回复前必须完成目标 all-in-one workflow
要求的候选全文和校验工具调用。只有真实响应满足 `baseResp.respCode=1`、`data.valid=true`
且 `data.diffRecordId>0` 才能按 workflow 模板展示提案；否则只返回 workflow 规定的最终错误。
禁止用模型自检代替工具、先展示再校验、伪造 Diff，或把中间错误作为用户确认节点。

只有本轮真实 `tool_validate_prompt_skeleton` 调用返回 `baseResp.respCode=1 && data.valid=true &&
data.diffRecordId>0`，才存在有效提案。无本轮校验工具调用记录、`data.diffRecordId=0`、占位 Diff 或
未锁定候选全文时，禁止声称校验通过、套用提案模板、请求确认或响应“展开完整提示词”；
按目标 workflow 返回明确错误。

**凡 workflow 定义了提案，所有提案输出必须完全按照该 workflow 的提案模板和内容要求执行。**
禁止改标题、字段、顺序、围栏或确认话术，禁止省略、概括、重写、补充提案内容，也禁止改用模型
自行组织的格式；模板要求引用工具原文的部分必须逐字符复制。

确认类表达统一直接进入 `[S2]`，入口不得提前判断、拒绝或回复固定错误；紧邻关系、Diff 和写入条件
只按 `[S2]` 处理，不得自行推测服务端消息序号。所有提案和成功回复严格使用 workflow/`[S1]`/`[S2]` 模板，
模板前后不加文字，不省略字段，不输出占位符，不改写服务端 Diff。
