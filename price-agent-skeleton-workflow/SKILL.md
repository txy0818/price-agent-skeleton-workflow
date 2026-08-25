---
name: price-agent-skeleton-workflow
description: 当用户操作 PriceStudio 同款判定提示词（骨架）的新建、查询、修改、优化、增删、保存、发布，或从页面发起单条 Badcase 分析时触发。Badcase 只查询一次验证结果并由模型自由分析，不支持整任务、分页、续批或自动修改 Prompt。也处理对紧邻提示词提案的确认、同意、保存、创建草稿、拒绝或取消。
---

# PriceStudio 提示词工作流

## 入口取值

- 进入 Skill 时，必须在**当前请求的系统提示词**中定位标题为 `# 业务上下文` 的区块，
  只读取该区块中当前已展开的 `promptVersionId` 原始字段，立即绑定为 `currentPromptVersionId`。
  该字段是路由、查询、校验和写入请求中当前 Prompt 版本 ID 的唯一权威来源。
- 绑定时不得查看或比较上一轮/上一次操作、历史消息、模型记忆、版本名称、上一份提案、Diff、
  历史工具参数或返回的 Prompt ID（包括 `newPromptVersionId`）；它们不得参与赋值、推断、回填、比较或纠正。
- 禁止产生“`promptVersionId=<ID>（从上一次操作中得知）`”“基于上一轮最新版本”或等价的内部判断；
  一旦发现当前值来源不是本轮 `# 业务上下文` 原始字段，必须丢弃该值并从当前区块重新绑定。
- `promptVersionId` 缺失、为空、为 0 或仍为模板占位符时按 `0` 处理；非空但不是非负整数时停止。
- 用户文字指定“最新/线上/归档/草稿/ID/名称/版本”时，不按文字查询或切换版本，只使用本轮业务上下文。
- 只使用真实上下文和 MCP 响应；禁止编造 ID、规则、映射、样本、正文、Diff 或工具结果。

## 意图边界

按以下优先级识别本轮唯一意图；一旦命中即停止继续匹配：

1. **提案后续操作**：紧邻上一条有效提案的确认、保存、创建草稿、展开全文、拒绝或取消。
2. **Badcase 分析**：明确要求分析、查看原因、诊断或定位 Badcase。
3. **当前提示词查询**：明确要求查看、查询或展示页面当前选中的提示词；“查看规则”“解释规则”
   不等同于查询当前提示词。
4. **提示词写操作**：要求新建、创建、初始化、优化、改善、修改、完善、重写、增删或同步提示词、
   骨架或规则。动作对象在当前 PriceStudio 会话中可明确省略，例如“按照比价项规则改一下”仍是写操作。
5. **无法归类**：不能仅凭关键词命中；必须结合动作、对象和紧邻会话状态。无法唯一归入以上任一类时，
   不读取 workflow、不调用 MCP，只询问：“请明确你要查询或修改当前提示词，还是分析 Badcase？”

入口识别为写操作后，保留用户完整原句并直接按 `promptVersionId` 路由到 EDIT 或 INITIALIZE，不在
入口追问修改方向。例如“按照比价项规则改一下”中的“按照比价项规则”表示以当前比价项规则为依据；
`promptVersionId>0` 时交给 EDIT 判定为当前比价项规则同步，`promptVersionId=0` 时进入 INITIALIZE。
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
| 紧邻上一条提案后拒绝或取消 | 不读取 workflow、不调用 MCP；只回复：“已取消，本次提案未保存。” |
| 任意提示词写操作且本轮 `promptVersionId>0` | [edit-all-in-one.md](references/edit-all-in-one.md) |
| 任意提示词写操作且本轮 `promptVersionId` 缺失、为 0 或占位符 | [initialize-all-in-one.md](references/initialize-all-in-one.md) |
| 无法唯一归类 | 不读取 workflow、不调用 MCP；只询问：“请明确你要查询或修改当前提示词，还是分析 Badcase？” |

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

一旦路由到 INITIALIZE 或 EDIT，必须从目标 all-in-one workflow 第一步开始顺序执行。除非前置步骤
命中该 workflow 明确规定的终止条件，否则必须先生成候选全文并完成
`tool_validate_prompt_skeleton` 调用，期间不得向用户展示候选、提案、确认话术或中间结果。
这个“期间”覆盖意图判定、分支标记、计划、工具调用说明、转换账本、自我校对和修正过程；
禁止将“开始执行”“现在展示提案”“等等/修正”或任何工具摘要作为可见回复。

只有本轮校验真实响应同时满足 `baseResp.respCode=1`、`data.valid=true`、`data.diffRecordId>0`，且已锁定
该次请求提交的候选全文，才按目标 workflow 模板展示提案。前置步骤终止、校验调用缺失或失败、
`data.diffRecordId<=0`、响应不完整时，只输出目标 workflow 规定的最终错误；禁止模型自检、先展示
再校验、伪造 Diff、请求确认或响应“展开完整提示词”。

**凡 workflow 定义了提案，所有提案输出必须完全按照该 workflow 的提案模板和内容要求执行。**
禁止改标题、字段、顺序、围栏或确认话术，禁止省略、概括、重写、补充提案内容，也禁止改用模型
自行组织的格式；模板要求引用工具原文的部分必须逐字符复制。
发送前必须丢弃所有过程草稿，仅用本轮已锁定的真实变量按目标 workflow 模板重组一次最终可见回复；
不得将模板追加在过程说明之后。

确认类表达统一直接进入 `[S2]`，入口不得提前判断、拒绝或回复固定错误；紧邻关系、Diff 和写入条件
只按 `[S2]` 处理，不得自行推测服务端消息序号。所有提案和成功回复严格使用 workflow/`[S1]`/`[S2]` 模板，
模板前后不加文字，不省略字段，不输出占位符，不改写服务端 Diff。
