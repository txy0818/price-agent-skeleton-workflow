# 验证任务 Badcase 分析

共享步骤见 [shared-steps.md](shared-steps.md)。

## 执行门禁

取得任务 ID 和页面当前 Prompt ID 后，输出阶段分析前必须实际完成：读取 [shared-steps.md](shared-steps.md) 与 [rule-loading-policy.md](rule-loading-policy.md)；查询本批最多 10 条验证结果和 CDN；从任务结果锁定基础提示词；按类目执行 `[S2]`；逐条核对商品快照与模型 `extracted`；完成去重、五类归因和本批统计。不得只报告“已加载流程”、预告下一步或要求用户再次回复才开始分析。

- 缺少任务 ID：只询问该标识。`promptVersionId<=0` 时不向用户索取 ID，只提示先在页面左侧
  选择本次验证任务使用的提示词后重试。
- 本批查询成功：当轮必须输出完整阶段分析；还有数据时才允许提示“继续分析”。
- 用户要求整合：当轮必须完成既有批次自检、合并、候选全文生成和 `[S3]` 校验，并在 `valid=true` 后输出最终服务端原样 Diff；不得只输出整合计划。
- 用户确认写入：只有已展示最终 Diff 时才执行 `[S5]`，不得为各批次分别写入。

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
   该提示词即本次基础版本，不再查询其他提示词。
   `data.prompt_version.prompt_content` 必须非空；为 `null`、空串或仅空白字符时立即停止，说明任务提示词尚未
   初始化，不能进行 Badcase 归因、分批分析或整合修改。
   记录 `selectedPromptVersionId=data.prompt_version.prompt_version_id`，要求大于 0，
   并锁定 `writeMode=EDIT`。
   一次最多分析 10 条，不得自行调大 `page.page_size`：每条会带回两组商品快照、逐项
   `analysis_process`，且 Kconf 开启时还会带回整段 `raw_llm_response`；单批 20 条会挤占上下文
   并导致前批阶段结论被压缩丢失。还有更多时按 `page.total/has_more` 报告剩余数。
   用户要求继续时查询下一页；
   必须保持同一任务 ID 和 `PromptVersionBaseInfo.prompt_version_id`，并按
   `validation_case_id` 去重。
4. 仅提取本页 `data.results[].category_id`，去重后逐个按 `[S2]` 调用
   `radar_query_price_rule(categoryId=<当前去重后的category_id>)`，再按各类目规则实际包含的
   比价项加载必要映射；禁止从标题、商品 JSON、提示词或模型输出猜类目，禁止跨类目混用。
   逐条按
   `human_label`、`model_label` 和 `analysis_process` 中的逐项抽取结果定位嫌疑项；
   `raw_llm_response` 非空时仅补充核对顶层 `result/reason/confidence/key_diff_point` 和原文。
   依照 [rule-loading-policy.md](rule-loading-policy.md) 的「Badcase 方向」定位
   嫌疑项。另调用
   `tool_query_cdn_report(validation_task_id=上下文.validationTaskId, prompt_version_id=上下文.promptVersionId, operator=上下文.operator)`。
   `base_resp.resp_code=1 && result=1` 时只读取 `data.report_cdn_url` 和
   `data.report_summary_json`；“验证报告不存在”视为无 CDN，继续分析并使用无链接输出分支；
   其他失败如实报告并停止。CDN 摘要不能替代逐条验证结果。
5. 逐条先用 `source_item_json` 与 `candidate_item_json` 核对商品事实，再用
   `analysis_process` 各比价项的
   `left`/`right` 的 `value`、`source` 看模型实际抽到了什么，两边对比得出结论；
   并结合 `source_title`、`candidate_title`、`reason`、规则、映射和任务提示词复核。
   两个 `*_item_json` 是 `{"detail":商详,"imageUrl":主图链接}` 结构，内容来自验证集导入时的
   快照，不是实时商品数据；导入 Excel 未填商详与图片时该字段为空。据此区分：
   - 快照里有该属性而 `value` 为空或为 `缺失` → **模型没抽到**，属于抽取或骨架问题。
   - 快照里确实没有该属性 → 归为映射或数据问题，不得因此改骨架。
   - 快照与模型抽取一致但客观事实不支持人工标签 → 疑似人工标签错误。
   - `*_item_json` 为空时只能基于标题与 `extracted` 判断；需要商详或主图才能定论时
     归为证据不足，不得拿模型自己的抽取结果去否定人工标签。
   - `analysis_process` 为空时，即使 `raw_llm_response` 存在也只可用于排查解析异常；该条归为
     数据问题，不据原始自由文本直接修改提示词。
   `is_correct=0` 不证明人工标签正确或提示词错误。
   接口不返回可用的规则快照，无法确认本次验证跑的是哪一版规则；因此发现骨架写法与当前
   本轮 `radar_query_price_rule` 返回的类目规则不一致时，必须同时考虑“规则在本次验证后已变更”这一可能，
   在建议中说明并提示用当前基础提示词重跑验证确认，不得直接断定为骨架缺陷。
6. 归为提示词规则缺陷、模型未遵循规则、疑似人工标签错误、映射或数据问题、证据不足。
   仅当同时成立时才能归为提示词规则缺陷：已从 `extracted` 定位到具体嫌疑比价项；
   商品快照核对后客观事实支持人工标签；能在 `PromptVersionBaseInfo.prompt_content` 中
   指到具体某一条规则并说明它为何导致该结果；且该规则与当前类目规则不一致。
   快照缺失而无法核对客观事实时归为证据不足，不得直接归为人工标签错误。
   骨架已忠实翻译规则、只是规则本身如此时属于业务规则问题，不得改提示词。
   仅提示词缺陷可修改，禁止针对个别商品写过拟合规则。只有单条命中的规则问题按证据不足
   处理，待后续批次出现同类样本再确认。

## 分批分析

每批结果只是当前会话中的阶段分析，不写入提示词表或 Diff 表，也不调用
`tool_edit_prompt_skeleton`。后续批次必须合并当前会话中同一任务、同一基础提示词的既有
阶段结论，不得把每批建议分别创建为草稿。非提示词缺陷不输出空 Diff；CDN 必须是最后一行。

````markdown
## 验证任务 Badcase 阶段分析
- Agent：`<名称；查不到时写ID>`
- 任务 / 验证集：`<task_id> / <dataset_id>`
- 基础提示词：`<名称>`（ID：`<PromptVersionBaseInfo.prompt_version_id>`）
- 批次：第 `<page>` 批
- Badcase：共 `<total>` 条，本批分析 `<analyzed>` 条，累计已分析 `<累计去重数>` 条
### 分类统计
| 类型 | 数量 | 处理 |
|---|---:|---|
| 提示词规则缺陷 | <数> | 修改并回归 |
| 模型未遵循规则 | <数> | 观察执行 |
| 疑似人工标签错误 | <数> | 复核标签 |
| 映射或数据问题 | <数> | 修复数据 |
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
<超过10：本次已分析10条，剩余 <remaining> 条；可回复“继续分析”。>
<需要整合时：可回复“整合修改建议给我一个草稿”。>
验证集报告：[查看 CDN 报告](<真实链接>)
````

无链接时最后一行写：`验证集报告：当前验证任务未返回 CDN 报告链接`。

## 整合修改建议

用户要求“整合修改建议给我一个草稿”时：

1. 只合并当前会话中任务 ID、验证集 ID 和基础提示词 ID 均一致的阶段分析；按
   `validation_case_id` 去重。整合前自检各批阶段结论是否完整可见；任一批次不可见时，
   必须用同一任务 ID、同一 `PromptVersionBaseInfo.prompt_version_id` 和原 `page.page_no` 重新查询该页，
   不得跳过也不得凭记忆补写。
2. 合并指向同一规则的重复建议；结合商品快照证据、`extracted` 抽取结果、比价规则和映射
   处理冲突。疑似人工标签错误、模型执行、映射或数据问题以及证据不足不得写入提示词。
3. 基于 `PromptVersionBaseInfo.prompt_content` 执行 `[S3]` 生成并校验一份合并后的完整
   提示词，最终 Diff 取 `[S3]` 返回的 `data.diff_content`，不自行书写。说明覆盖的批次、
   Badcase ID、尚未分析的数量和回归风险。
4. 完整读取 [edit-skeleton-workflow.md](edit-skeleton-workflow.md) 的“提案”和“输出硬校验”，按其
   固定模板及 `[S4]` 展示最终提案；「合理性判断」写明覆盖批次、Badcase ID、未分析数量和回归
   风险。模板外不添加文字，此时不得调用写入 MCP。

## 确认创建后

只有最终整合结果存在提示词缺陷且用户紧邻确认，才执行 `[S5]` 的 EDIT 路径。
仅调用一次并创建一个新草稿，不为各批次分别创建草稿。本轮不再输出 CDN。
