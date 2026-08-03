# 单条 Badcase 分析

共享步骤见 [shared-steps.md](shared-steps.md)。

## 查询与复核

1. 执行 `[S1]` 查询 Agent。
2. 必须取得验证任务 ID：用户本轮明确提供时优先，否则使用上下文 `validationTaskId`；
   两者都没有时要求补充。同时必须取得 `validation_case_id`，不能只凭文字或 CDN 正式归因。
3. 调用
   `tool_query_validation_result(validation_task_id=<任务ID>, label_filter=3, validation_case_id=<Badcase ID>, page_size=1, operator=当前业务上下文.operator)`。
   从返回的 `PromptVersionBaseInfo` 取得任务实际使用的 `prompt_version_id`、
   `version_name`、`version_status` 和完整 `prompt_content`，同时取得验证集、商品快照、
   人工/模型标签和原因；校验 Agent 及页面非零 `datasetId`。该提示词即本次基础版本，
   不再查询其他提示词。
4. 执行 `[S2]` 加载规则与映射，**按需取**：先按 `human_label` 与 `model_label` 判定错误方向，
   再解析本条 `ToolValidationCaseResult.raw_llm_response`（为空时退回 `analysis_process`）中的
   `extracted`，按 [rule-loading-policy.md](rule-loading-policy.md) 的「提取嫌疑比价项」
   依该方向选出目标项作为 `compare_items`，`category_ids` 传本条 `category_id`；
   选不出时传空取全量。嫌疑项涉及品牌或材质判定时才调用
   `tool_query_rule_data_table`，且只传相关 `table_types`。另调用
   `tool_query_cdn_report(validation_task_id=<任务ID>, operator=当前业务上下文.operator)`
   取得报告链接。
5. 用 `source_item_json` 与 `candidate_item_json` 核对客观事实，再结合标题、主图、标签、
   模型原因、规则、映射和任务提示词复核。`is_correct=0` 不证明人工标签正确或提示词错误；
   客观事实不支持人工标签时归为疑似人工标签错误，字段本身缺失时归为数据问题。
   同时将 `rule_context_snapshot_json` 与本轮取回的规则比对，若规则已变更而本次验证跑的是
   旧快照，建议重跑验证而非修改提示词。
6. 归为提示词规则缺陷、模型未遵循规则、疑似人工标签错误、映射或数据问题、证据不足。
   仅当同时成立时才能归为提示词规则缺陷：已从 `extracted` 定位到具体嫌疑比价项；
   商品快照核对后客观事实支持人工标签；能在 `PromptVersionBaseInfo.prompt_content` 中
   指到具体某一条规则并说明它为何导致该结果；且该规则与当前 `price_rule_json` 不一致。
   骨架已忠实翻译规则、只是规则本身如此时属于业务规则问题，不得改提示词。
   仅提示词缺陷可修改，禁止为单个商品写过拟合规则。
7. 仅在确认属于提示词缺陷时执行 `[S3]` 生成并校验完整提示词；否则不生成、不校验。

## 输出

按 `[S4]` 展示。非提示词缺陷不输出 Diff；CDN 必须是最后一行。

````markdown
## Badcase 分析
- Agent：`<名称；查不到时写ID>`
- 任务 / Badcase / 验证集：`<task_id> / <case_id> / <dataset_id>`
- 任务提示词：`<名称>`（ID：`<ID>`）
### 样本与证据
- 左 / 右商品：<标题>
- 人工标签 / 模型结论：<标签>
| 比价项 | 左侧证据 | 右侧证据 | 复核 |
|---|---|---|---|
| <比价项> | <属性/标题/主图> | <属性/标题/主图> | <结论> |
### 结论与建议
- 类型：<五类之一>
- 是否修改提示词：<是/否>
- 依据：<关键证据>
- 建议与回归：<方案；无缺陷写“不建议修改提示词”>
```diff
- <原规则>
+ <建议规则>
```
<有缺陷：完整提示词已生成并通过格式校验，未在此展开。需查看请回复「展开完整提示词」。
请确认是否按以上建议创建新提示词草稿。>
<无缺陷：该问题不属于提示词规则缺陷，不建议修改。>
验证集报告：[查看 CDN 报告](<真实链接>)
````

无链接时最后一行写：`验证集报告：当前验证任务未返回 CDN 报告链接`。

## 确认后

只有提示词缺陷且用户明确确认，才执行 `[S5]`，`prompt_version_id` 传
`PromptVersionBaseInfo.prompt_version_id`，`change_reason` 写 Badcase 修复原因。
本轮不再输出 CDN。
