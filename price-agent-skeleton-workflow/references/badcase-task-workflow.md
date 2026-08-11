# 验证任务 Badcase 分析

共享步骤见 [shared-steps.md](shared-steps.md)。

## 执行门禁

取得任务 ID 和页面当前 Prompt ID 后，输出阶段分析前必须实际完成：读取并按顺序执行 [badcase-processing-workflow.md](badcase-processing-workflow.md) `[B0]`～`[B8]`，以及其要求的共享步骤、规则和归因规范；查询本批最多 10 条验证结果和同任务 CDN 报告；从任务结果锁定基础提示词；按类目执行 `[S2]`；逐条核对商品快照与模型 `extracted`；完成去重、八类归因和本批统计。不得只报告“已加载流程”、预告下一步或要求用户再次回复才开始分析。

- 缺少任务 ID：只询问该标识。`promptVersionId<=0` 时不向用户索取 ID，只提示先在页面左侧
  选择本次验证任务使用的提示词后重试。
- 本批查询成功：当轮必须输出完整阶段分析；还有数据时才允许提示“继续分析”。
- 用户要求整合：当轮必须完成既有批次自检和归因合并；存在提示词规则缺陷时继续生成候选全文、
  执行 `[S3]` 并在校验成功后输出服务端原样 Diff，不存在时输出固定无修改整合结果。不得只输出
  整合计划，也不得在无提示词缺陷时调用 `[S3]`。
- 用户确认写入：只有已展示最终 Diff 时才执行 `[S5]`，不得为各批次分别写入。

## 跨轮续批锁

首批查询成功后，立即从“本轮请求 + 同次响应”建立不可变
`badcaseTaskLock={validationTaskId, promptVersionId, datasetId, ruleGroupId, operator}`：其中前两项取
本轮业务上下文且已通过响应反校验，`datasetId` 和 `ruleGroupId` 取同次响应，`operator` 取本轮业务
上下文。锁只用于当前会话中的该次整任务分析，不得被后续工具返回的新 ID、历史消息或模型推断改写。

用户说“继续分析”“下一批”或“整合修改建议”时，执行任何 MCP 前必须重新读取本轮业务上下文，
并先完成四项共同身份校验：

1. 本轮 `validationTaskId == badcaseTaskLock.validationTaskId`；
2. 本轮 `promptVersionId == badcaseTaskLock.promptVersionId`；
3. 本轮 `operator == badcaseTaskLock.operator`；
4. 上一批固定输出中的“续批锁”与内部锁逐字段一致。

共同身份校验通过后再按意图区分：

- **继续分析/下一批**：必须有上一批成功响应且 `data.page.has_more=true`；下一页严格等于上一批
  `data.page.page_no + 1`，禁止由用户文字指定或跳页。`has_more=false` 时固定回复
  `本任务 Badcase 已全部分析，无下一页；如需汇总结论，请回复「整合修改建议给我一个草稿」。`
- **整合修改建议**：不要求 `has_more=true`，不计算或查询下一页；只要共同身份校验通过且至少存在
  一批完整阶段分析，就允许整合。`has_more=true` 时也可整合，但必须如实保留未分析数量和回归风险；
  `has_more=false` 时正常整合全部已分析结果。

任一共同身份项不满足时立即停止，不查询下一页、不合并旧批次，并固定回复：
`续批上下文已变化：上一批 taskId=<旧值>、promptVersionId=<旧值>，本轮 taskId=<新值>、promptVersionId=<新值>。为避免混合不同验证任务或提示词，已停止继续分析；请重新发起整任务 Badcase 分析。`
不得自动用新值覆盖旧锁。用户明确重新发起整任务分析时，清空旧锁，从第 1 页建立新锁。

## 查询与复核

1. 仅当业务上下文缺少 `ruleGroupId` 时执行 `[S1]`；已有则跳过。
2. 只从本轮业务上下文取得 `validationTaskId` 和 `promptVersionId`，两者都必须大于 0；
   前者缺失时询问任务 ID，后者缺失时按上述页面选择提示停止。不得从用户文字、历史消息、
   CDN 或工具返回猜测/替换。CDN 不能代替任务 ID。
3. 调用
   `tool_query_validation_result(validation_task_id=上下文.validationTaskId, label_filter=3, page={page_no:<当前批次页码>,page_size:10}, operator=上下文.operator, validation_case_id=0, prompt_version_id=上下文.promptVersionId)`。
   首批 `page_no=1`，继续分析时递增。只接受 `base_resp.resp_code=1 && result=1`；要求响应
   `data.validation_task_id` 和 `data.prompt_version.prompt_version_id` 分别与请求完全一致，且每条
   `data.results[].is_correct=0`。任一不符立即报告上下文/响应不一致并停止。
   只按以下精确字段读取，禁止用字段含义猜路径：
   - 任务/验证集/摘要：`data.validation_task_id`、`data.dataset_id`、
     `data.validation_summary_json`；
   - 基础提示词：`data.prompt_version.{prompt_version_id,rule_group_id,version_no,version_name,
     version_status,prompt_content}`；
   - 每条结果：`data.results[].{validation_result_id,validation_case_id,category_id,source_item_id,
     candidate_item_id,source_title,candidate_title,human_label,model_label,is_correct,reason,
     analysis_process,radar_task_id,radar_task_url,source_item_json,candidate_item_json}`；
   - 分页：`data.page.page_no/page_size/total/has_more`。
   `data.dataset_id` 和每条 `data.results[].category_id` 必须大于 0；上下文 `ruleGroupId>0` 时要求
   `data.prompt_version.rule_group_id` 与其一致。响应没有 `agent_id`，禁止声称已由本接口校验 Agent。
   继续分析时还要求 `data.dataset_id==badcaseTaskLock.datasetId` 且
   `data.prompt_version.rule_group_id==badcaseTaskLock.ruleGroupId`；任一变化按续批上下文不一致停止。
   该提示词即本次基础版本，不再查询其他提示词。
   `data.prompt_version.prompt_content` 必须非空；为 `null`、空串或仅空白字符时立即停止，说明任务提示词尚未
   初始化，不能进行 Badcase 归因、分批分析或整合修改。
   记录 `selectedPromptVersionId=data.prompt_version.prompt_version_id`，要求大于 0，
   并锁定 `writeMode=EDIT`。
   一次最多分析 10 条，不得自行调大 `page.page_size`：每条会带回两组商品快照和逐项
   `analysis_process`；单批 20 条会挤占上下文并导致前批阶段结论被压缩丢失。还有更多时按
   `page.total/has_more` 报告剩余数。
   用户要求继续时查询下一页；
   必须保持同一任务 ID 和 `PromptVersionBaseInfo.prompt_version_id`，并按
   `validation_case_id` 去重。
4. 仅提取本页 `data.results[].category_id`，去重后逐个按 `[S2]` 调用
   `radar_query_price_rule(categoryId=<当前去重后的category_id>)`，再按各类目规则实际包含的
   比价项加载必要映射；禁止从标题、商品 JSON、提示词或模型输出猜类目，禁止跨类目混用。
   逐条按
   `human_label`、`model_label` 和 `analysis_process` 中的逐项抽取结果定位嫌疑项。依照
   [rule-loading-policy.md](rule-loading-policy.md) 的「Badcase 方向」定位嫌疑项。随后调用
   `tool_query_cdn_report(validation_task_id=上下文.validationTaskId, prompt_version_id=上下文.promptVersionId, operator=上下文.operator)`；
   只接受 `base_resp.resp_code=1 && result=1`，并只从 `data.report_cdn_url`、
   `data.report_summary_json` 读取报告信息。链接为空视为无 CDN，继续分析并使用无链接输出分支；
   报告摘要不能替代逐条验证结果。
5. 逐条先用 `source_item_json` 与 `candidate_item_json` 核对商品事实，再用
   `analysis_process` 各比价项的
   `left`/`right` 的 `value`、`source` 看模型实际抽到了什么，两边对比得出结论；
   并结合 `source_title`、`candidate_title`、`reason`、规则、映射和任务提示词复核。
   两个 `*_item_json` 是 `{"detail":商详,"imageUrl":主图链接}` 结构，内容来自验证集导入时的
   快照，不是实时商品数据；导入 Excel 未填商详与图片时该字段为空。据此区分：
   - 快照里有该属性而 `value` 为空或为 `缺失` → **模型没抽到**；提示词抽取要求缺失/错误时归为
     提示词规则缺陷，提示词要求明确时归为模型抽取或推理异常。
   - 快照里确实没有该属性，且能确认是映射或导入字段错误 → 归为映射或样本数据问题；无法确认
     缺失原因且该属性决定结论时归为证据不足。两种情况都不得因此改骨架。
   - 快照与模型抽取一致但客观事实不支持人工标签 → 疑似人工标签错误。
   - `*_item_json` 为空时只能基于标题与 `extracted` 判断；需要商详或主图才能定论时
     归为证据不足，不得拿模型自己的抽取结果去否定人工标签。
   - `analysis_process` 为空不能单独归为任何确定问题：已由其他结构化证据确认样本字段或快照异常时
     归为映射或样本数据问题；已确认解析、标签转换、落库或接口出参异常时归为解析或系统链路异常；
     无法确认缺失发生层级时归为证据不足。三种情况都不得修改提示词。
   `is_correct=0` 不证明人工标签正确或提示词错误。
   接口不返回可用的规则快照，无法确认本次验证跑的是哪一版规则；因此发现骨架写法与当前
   本轮 `radar_query_price_rule` 返回的类目规则不一致时，必须同时考虑“规则在本次验证后已变更”这一可能，
   在建议中说明并提示用当前基础提示词重跑验证确认，不得直接断定为骨架缺陷。
6. 将本页去重后的每条样本分别整理成统一内部分析记录，按统一流程 `[B4]`～`[B6]`
   逐条建立证据、执行充分性门禁并选择唯一一级归因，再统计八类数量；不得由总体准确率或摘要反推
   单条根因。
   仅提示词缺陷可修改，禁止针对个别商品写过拟合规则。只有一条样本命中且缺少直接因果证据时
   归为证据不足；即使只有一条，商品事实、可信规则、任务提示词和错误结果之间的直接因果链完整时，
   仍可归为提示词规则缺陷，但修改必须写成可泛化规则，并在回归方案中覆盖同类和反向样本。

## 分批分析

每批结果只是当前会话中的阶段分析，不写入提示词表或 Diff 表，也不调用
`tool_edit_prompt_skeleton`。后续批次必须合并当前会话中同一任务、同一基础提示词的既有
阶段结论，不得把每批建议分别创建为草稿。非提示词缺陷不输出空 Diff；CDN 必须是最后一行。

````markdown
## 验证任务 Badcase 阶段分析
- Agent：`<名称；查不到时写ID>`
- 任务 / 验证集：`<task_id> / <dataset_id>`
- 基础提示词：`<名称>`（ID：`<PromptVersionBaseInfo.prompt_version_id>`）
- 续批锁：`task=<validation_task_id>; prompt=<prompt_version_id>; dataset=<dataset_id>; ruleGroup=<rule_group_id>`
- 批次：第 `<page>` 批
- Badcase：共 `<total>` 条，本批分析 `<analyzed>` 条，累计已分析 `<累计去重数>` 条
### 分类统计
| 类型 | 数量 | 处理 |
|---|---:|---|
| 提示词规则缺陷 | <数> | 修改并回归 |
| 模型未遵循规则 | <数> | 观察执行 |
| 模型抽取或推理异常 | <数> | 优化模型能力 |
| 业务规则问题 | <数> | 修订业务规则 |
| 疑似人工标签错误 | <数> | 复核标签 |
| 映射或样本数据问题 | <数> | 修复映射或数据 |
| 解析或系统链路异常 | <数> | 排查解析链路 |
| 证据不足 | <数> | 补充证据 |
### 明细
| Badcase ID | 人工/模型标签 | 类型 | 关键证据（快照事实 vs extracted） | 建议 |
|---|---|---|---|---|
| `<case_id>` | <标签> | <类型> | <比价项：快照事实 / 左右 value / match> | <建议> |
### 本批候选修改
- 缺陷与样本：<仅确认的提示词缺陷及 case ID>
- 候选建议：<待最终整合的规则修改；无缺陷写“本批无提示词修改建议”>
- 不写入提示词：<标签、数据、映射或执行问题>
- 与前批关系：<新增/重复/补充/冲突；首批写“首批”>
- 分页状态：<has_more=true 时写“本次已分析 <analyzed> 条，剩余 <remaining> 条；可回复「继续分析」”；否则写“本任务 Badcase 已全部分析”>
- 整合状态：<累计存在提示词规则缺陷时写“可回复「整合修改建议给我一个草稿」”；否则写“当前累计无可整合的提示词修改建议”>
验证集报告：[查看 CDN 报告](<真实链接>)
````

无链接时最后一行写：`验证集报告：当前验证任务未返回 CDN 报告链接`。

### 阶段分析输出硬校验

发送前逐项检查；任一项不满足就在内部重组回复，不向用户展示错误版本：

1. 第一行必须严格等于 `## 验证任务 Badcase 阶段分析`，模板前后不得添加文字。
2. 必须保留模板全部字段、三级标题、表头和顺序；禁止输出任务概览、耗时、Token、自由发挥的
   下一步说明或模板中不存在的段落。
3. 输出前必须存在本轮真实 `tool_query_validation_result`、`tool_query_cdn_report`，以及本页去重后
   每个 `category_id` 对应的规则/必要映射查询；缺任一必需记录只返回该步骤真实错误。
4. “续批锁”必须逐字段来自 `badcaseTaskLock`，并与本轮请求及响应一致；继续分析时必须存在上一批
   锁校验通过的记录，禁止只因用户说了“继续分析”就沿用历史 ID。
5. 八类数量之和必须等于本批去重后的明细数；明细按 `validation_case_id` 去重；`total`、
   `has_more` 和剩余数量只能由 `data.page` 计算，不得根据摘要或历史回复猜测。
6. 阶段分析不得出现 Diff、`diff_id`、S3 成功话术、确认创建话术或写入操作；候选修改只能作为
   待整合结论。
7. `分页状态` 和 `整合状态` 必须按模板给定条件各选择一个固定句式，不得保留条件占位符。
8. CDN 行必须是最后一行；有链接只用真实 `tool_query_cdn_report.data.report_cdn_url`，无链接使用
   固定无链接文本。

## 整合修改建议

用户要求“整合修改建议给我一个草稿”时：

1. 只合并当前会话中任务 ID、验证集 ID 和基础提示词 ID 均一致的阶段分析；按
   `validation_case_id` 去重。整合前自检各批阶段结论是否完整可见；任一批次不可见时，
   必须用同一任务 ID、同一 `PromptVersionBaseInfo.prompt_version_id` 和原 `page.page_no` 重新查询该页，
   不得跳过也不得凭记忆补写。
   执行整合前同样必须通过“跨轮续批锁”的本轮上下文校验；不一致时禁止整合。
2. 合并指向同一规则的重复建议；结合商品快照证据、`extracted` 抽取结果、比价规则和映射
   处理冲突。除“提示词规则缺陷”外的其他七类不得写入提示词。
3. 若合并后没有提示词规则缺陷，按统一流程 `[B7]` 不执行 `[S3]`，只能使用下方“无修改整合结果
   模板”。若存在提示词规则缺陷，将同一续批锁下的缺陷记录合并后交给 `[B7]`，只生成一份候选完整
   提示词并执行一次有效 `[S3]`；最终 Diff 和失败处理完全按 `[B7]`，不得自行书写或另造校验流程。
4. 存在提示词规则缺陷时，完整读取 [edit-skeleton-workflow.md](edit-skeleton-workflow.md) 的“输出硬
   校验”，复用其 Diff 原样复制、非零 Diff ID 和确认话术门禁，但必须使用下方“提示词缺陷整合提案
   模板”展示，保留 Task、Dataset、批次、Badcase、分析进度和同任务最近一次 CDN 查询结果。
   模板外不添加文字，此时不得
   调用写入 MCP。

### 提示词缺陷整合提案模板

````markdown
## 验证任务 Badcase 修改提案
- Agent：`<名称；查不到时写ID>`
- 任务 / 验证集：`<task_id> / <dataset_id>`
- 基础提示词：`<名称>`（ID：`<PromptVersionBaseInfo.prompt_version_id>`）
- 修改建议 ID（diff_id）：`<原样引用同次 S3 返回的非零 data.diff_record_id>`
- 状态：尚未保存
- 已整合批次：<批次列表>
- 缺陷 Badcase：<按 validation_case_id 去重后的 ID 列表>
- 已分析 / 未分析：`<累计去重数> / <remaining>`
### 合理性判断
- 修改目标：<合并后的提示词缺陷和修改目标>
- 规则与映射依据：<本轮真实规则、必要映射与样本证据>
- 影响与风险：<覆盖范围、未分析数量、冲突和回归风险>
- 不修改部分：<明确排除的其他七类问题及保留规则>
### Diff
```diff
<逐字符复制同次 tool_validate_prompt_skeleton 返回的 data.diff_content>
```
完整提示词已生成并通过格式校验，未在此展开。需查看请回复「展开完整提示词」。
以上仅为修改提案。确认后创建新提示词草稿，不覆盖基础版本。
确认无误请回复：**确认创建提示词草稿**。
验证集报告：[查看 CDN 报告](<真实链接>)
````

无链接时将模板最后一行替换为：`验证集报告：当前验证任务未返回 CDN 报告链接`。

### 无修改整合结果模板

````markdown
## 验证任务 Badcase 整合结果
- Agent：`<名称；查不到时写ID>`
- 任务 / 验证集：`<task_id> / <dataset_id>`
- 基础提示词：`<名称>`（ID：`<PromptVersionBaseInfo.prompt_version_id>`）
- 已整合批次：<批次列表>
- 已分析 / 未分析：`<累计去重数> / <remaining>`
### 整合结论
- 提示词规则缺陷：0
- 是否修改提示词：否
- 依据：<各批证据与去重后的结论>
- 建议与回归：不建议修改提示词；<标签、执行、映射、数据或补证处理建议>
当前累计问题不属于提示词规则缺陷，不建议修改提示词。
验证集报告：[查看 CDN 报告](<真实链接>)
````

无链接时最后一行写：`验证集报告：当前验证任务未返回 CDN 报告链接`。

### 整合输出硬校验

1. 无提示词规则缺陷时第一行必须严格等于 `## 验证任务 Badcase 整合结果`，完整保留上述模板
   字段和顺序，不得输出 Diff、S3 成功话术或确认创建话术；CDN 必须是最后一行。
2. 存在提示词规则缺陷时第一行必须严格等于 `## 验证任务 Badcase 修改提案`，完整保留专用模板
   的任务上下文、合理性判断、Diff 和确认话术；输出前必须存在本轮真实
   `tool_validate_prompt_skeleton` 成功调用记录，且必须通过 `edit-skeleton-workflow.md` 中适用于 Diff、
   Diff ID 和确认话术的全部硬校验。不得使用无修改模板或普通 EDIT 提案模板。
3. 两个分支都必须按 `validation_case_id` 去重，且任务 ID、验证集 ID、基础提示词 ID 完全一致；
   覆盖批次、Badcase ID、未分析数量和回归风险不得省略或猜测。
4. 两个分支的 CDN 行都必须是最后一行；有链接只用同任务最近一次真实
   `tool_query_cdn_report.data.report_cdn_url`，无链接使用固定无链接文本，不得在 CDN 后添加确认或解释。

## 确认创建后

只有最终整合结果存在提示词缺陷且用户紧邻确认，才执行 `[S5]` 的 EDIT 路径。
仅调用一次并创建一个新草稿，不为各批次分别创建草稿。本轮不再输出 CDN。
