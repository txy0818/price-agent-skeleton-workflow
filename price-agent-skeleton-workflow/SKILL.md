---
name: price-agent-skeleton-workflow
description: 处理 PriceStudio 同款判定提示词（骨架）的新建、查询、修改、优化、增删、保存、发布、验证任务和 Badcase，包括“一键分析 Badcase”、单条、整任务、用户描述 Badcase，以及整任务的“继续分析”“下一批”“整合修改建议给我一个草稿”；也处理对紧邻提示词提案的确认、同意、保存、创建草稿、拒绝或取消。出现上述业务请求或无具体方向的提示词写操作时使用；问候、能力介绍、碎片输入和范围外请求不使用。
---

# PriceStudio 提示词工作流

## 入口边界

- 进入 Skill 时立即重新读取本轮业务上下文的 `promptVersionId` 并覆盖上一轮值；该值是唯一版本
  上下文，历史及工具返回的 Prompt ID 不得替换它。具体查询和
  确认取值统一按 [base-version-policy.md](references/base-version-policy.md) 与
  [shared-steps.md](references/shared-steps.md) 执行。
- 用户指定最新、线上、归档、草稿、ID、名称或版本时，执行
  [base-version-policy.md](references/base-version-policy.md) 的查询对象门禁，不自行查询或切换。
- 只使用真实上下文和 MCP 响应；禁止编造 ID、规则、映射、样本、正文、Diff 或工具结果。
- 写入前必须有合规提案和有效确认；失败或超时不重试。

## 识别操作

- “新建提示词”“创建提示词”“初始化提示词”“优化提示词”“改善提示词”“修改提示词”
  “完善提示词”“重写提示词”及对应“骨架”说法，即使
  没有说明方向或具体内容，也属于完整的提示词写操作；禁止追问，直接按本轮 `promptVersionId`
  进入下表 EDIT 或 INITIALIZE。
- 当前页面仅出现“新建、修改、优化、改善、完善”等写操作词而未说对象，也按提示词写操作处理。
- 仅说“优化规则”“改善规则”“完善规则”“修改规则”等泛化对象，且未说明比价项、母子品牌、
  材质、阈值或具体改动 → 不进入 workflow、不调用 MCP；只回复：“我还不确定你说的‘规则’具体
  指什么，请说明要调整的规则或具体内容。”
- “修改规则：<具体内容>”、明确某个比价项，或给出可执行的规则增删改内容 → 提示词写操作；
  不因用户使用“规则”一词而追问。
- “新增母子品牌：A→B”“删除材质映射：X→Y”等内容可唯一定位 → 精确提示词写操作。
- “展开完整提示词”“查看完整提示词”“展示完整提示词” → 展开上一条有效提案锁定的候选全文，
  不是范围外请求，也不是重新查询页面当前提示词。
- “确认”“同意”“保存”“创建草稿”等表达 → 提案决策操作；无论是否紧邻都不是范围外请求，
  直接进入 [shared-steps.md](references/shared-steps.md) `[S5]`，入口不提前判断或拒绝。
- “继续分析”“继续分析 Badcase”“下一批”及等价表达 → 恢复当前会话已建立续批锁的验证任务
  Badcase workflow，只执行其“继续分析/下一批”分支；不得重新路由单条、用户描述或普通 EDIT。
  当前会话没有有效续批锁时，只回复：“当前会话没有可继续的整任务 Badcase 分析，请重新发起
  「一键分析 Badcase」。”禁止从历史 ID 猜测或新建续批锁。
- “整合修改建议”“整合修改建议给我一个草稿”及等价表达 → 恢复当前会话已建立续批锁的验证任务
  Badcase workflow，只执行其“整合修改建议”分支；不得重新查询下一页、改走普通 EDIT 或另造提案。
  当前会话没有有效续批锁或完整阶段分析时，只回复：“当前会话没有可整合的整任务 Badcase 阶段
  结果，请先发起「一键分析 Badcase」。”
- 用户提供具体左右商品事实、人工标签和模型结论，且明确没有真实验证任务/Case，或者同时满足
  “已提供上述具体 Badcase 信息”和本轮 `validationTaskId<=0 && validationCaseId<=0` → 用户描述
  Badcase workflow；不得因为缺少 Task ID 而询问验证任务 ID。缺失类目或决定性证据由该 workflow
  的证据不足分支处理。
- 用户明确说“整任务”“批量 Badcase”“整个验证任务” → 验证任务 Badcase workflow；该明确范围
  优先于本轮可能残留的 `validationCaseId`。用户明确说“单条”“这个 Case”或指定当前 Case → 单条
  Badcase workflow。
- “一键分析 Badcase”“分析 Badcase”“查看 Badcase”及等价表达且没有上述明确范围 → Badcase 分析
  操作；按本轮业务上下文路由：`validationCaseId>0` 进入单条流程，否则 `validationTaskId>0` 进入
  验证任务流程；两者都缺失且没有具体商品描述时只询问验证任务 ID。禁止因用户未在文字中重复 ID
  而改走普通 EDIT。
- 写操作即使没有方向也是完整请求，禁止追问方向或索取 Badcase；只按 `promptVersionId` 选路，
  用户使用“初始化”或“修改”等措辞不能改变版本路由。

## 选择唯一 workflow

| 请求或状态 | 必须完整读取并执行 |
|---|---|
| “继续分析”“下一批”，且当前会话存在有效整任务续批锁 | 恢复 [badcase-task-workflow.md](references/badcase-task-workflow.md)，只执行继续分析分支 |
| “整合修改建议”“整合修改建议给我一个草稿”，且当前会话存在有效续批锁和阶段分析 | 恢复 [badcase-task-workflow.md](references/badcase-task-workflow.md)，只执行整合修改建议分支 |
| 明确整任务/批量/整个验证任务 | [badcase-task-workflow.md](references/badcase-task-workflow.md)，优先于残留 `validationCaseId` |
| 单条 Badcase，或“一键分析 Badcase”且本轮 `validationCaseId>0` | [badcase-single-workflow.md](references/badcase-single-workflow.md) |
| 验证任务/批量 Badcase，或“一键分析 Badcase”且 `validationCaseId<=0 && validationTaskId>0` | [badcase-task-workflow.md](references/badcase-task-workflow.md) |
| 具体左右商品事实、人工标签和模型结论，且无真实 Task/Case | [badcase-description-workflow.md](references/badcase-description-workflow.md) |
| 仅查询当前提示词 | [base-version-policy.md](references/base-version-policy.md) |
| 紧邻上一条有效提案后要求展开/查看/展示完整提示词 | 恢复该提案 workflow，只执行 [shared-steps.md](references/shared-steps.md) `[S4]` 的展开全文分支 |
| 确认/同意/保存/创建草稿 | 直接执行 [shared-steps.md](references/shared-steps.md) `[S5]`；所有门禁和错误均由 `[S5]` 处理 |
| 任意提示词写操作且本轮 `promptVersionId>0` | [edit-skeleton-workflow.md](references/edit-skeleton-workflow.md) |
| 任意提示词写操作且本轮 `promptVersionId` 缺失、为 0 或占位符 | [initialize-skeleton-workflow.md](references/initialize-skeleton-workflow.md) |

`promptVersionId>0` 命中 EDIT 后禁止再查找 INITIALIZE workflow；当前正文残缺不是切换流程或要求
用户补正文的理由，必须由 EDIT 按其格式重构分支处理。

## 不可跳过

选定 workflow 后，禁止用自由回答、模型自检、占位数据或“流程骨架”代替实际执行。凡 workflow
要求读取文件或调用 MCP，首条可见回复前必须存在本轮对应的真实读取和调用记录；缺任一记录只能
返回具体文件/工具错误，禁止继续输出提案、确认话术、完整提示词或范围外固定回复。

选定 workflow 后完整读取该文件及其直接要求的 references，并严格按原顺序执行；只跳过 workflow
明确允许跳过的步骤。详细版本选择、规则与关系加载、骨架格式、S1～S5、Diff、确认、写入参数和
输出模板，分别以对应 reference 为唯一权威，本文件不补充或重解释。

凡产生提示词提案的 INITIALIZE、EDIT 或 Badcase 修复，首条可见回复前必须完成目标 workflow
要求的候选全文和 [shared-steps.md](references/shared-steps.md) `[S3]`。只有真实响应满足
`valid=true` 且 `diff_record_id>0` 才能按 `[S4]` 展示提案；否则只返回 workflow 规定的最终错误。
禁止用模型自检代替工具、先展示再校验、伪造 Diff，或把中间错误作为用户确认节点。

只有本轮真实 `tool_validate_prompt_skeleton` 调用返回 `base_resp.resp_code=1 && data.valid=true &&
data.diff_record_id>0`，才存在有效提案。无本轮 S3 调用记录、`diff_record_id=0`、空 Diff、占位 Diff 或
未锁定候选全文时，禁止声称校验通过、套用提案模板、请求确认或响应“展开完整提示词”；按 `[S3]`
返回明确错误。

**凡 workflow 定义了提案，所有提案输出必须完全按照该 workflow 的提案模板和内容要求执行。**
禁止改标题、字段、顺序、围栏或确认话术，禁止省略、概括、重写、补充提案内容，也禁止改用模型
自行组织的格式；模板要求引用工具原文的部分必须逐字符复制。

Badcase workflow 的分析模板与提案模板具有同等强制力：必须以目标 workflow 指定的 `##` 标题
为第一行并逐项保留字段、顺序和表格。禁止改成 `【...】`、自由分析、任务概览、下一步选项或
“如果需要”；缺少任一必需 MCP 调用记录时禁止输出分析模板。
三个 Badcase workflow 发送任何正常可见结果前，都必须逐项通过各自的“输出硬校验”；整任务
最终存在提示词缺陷时使用其专用整合提案模板，并同时通过 `edit-skeleton-workflow.md` 中适用于 Diff、
Diff ID 和确认话术的硬校验。
三个流程还必须完整读取并执行 [badcase-attribution-policy.md](references/badcase-attribution-policy.md)，
只按其中八类归因、冲突消解顺序和处理动作输出；只有“提示词规则缺陷”允许进入 S3 和 Diff。
Badcase 的共同执行顺序唯一以
[badcase-processing-workflow.md](references/badcase-processing-workflow.md) `[B0]`～`[B8]` 为准；三个
入口文件只补充输入查询、分页/续批和固定输出差异，不得另造一套分析处理流程。

确认类表达统一直接进入 `[S5]`，入口不得提前判断、拒绝或回复固定错误；紧邻关系、Diff 和写入条件
只按 `[S5]` 处理，不得自行推测服务端消息序号。所有提案和成功回复严格使用 workflow/`[S4]`/`[S5]` 模板，
模板前后不加文字，不省略字段，不输出占位符，不改写服务端 Diff。
