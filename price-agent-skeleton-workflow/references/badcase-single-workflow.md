# 单条 Badcase 分析

## 查询与复核

1. 调用
   `query_agent_detail(agent_id=当前业务上下文.agentId, operator=当前业务上下文.operator)`
   查询 Agent 详情；仅当 `base_resp.resp_code=1` 时读取 `data`。所有 MCP 结果按
   `SKILL.md` 的失败与重试规则处理。
2. 必须取得验证任务 ID：用户本轮明确提供时优先，否则使用上下文 `validationTaskId`；两者都没有时要求补充。同时必须取得 `validation_case_id`，不能只凭文字或 CDN 正式归因。
3. 调用
   `tool_query_validation_result(validation_task_id=<任务ID>, label_filter=3, validation_case_id=<Badcase ID>, page_size=1, operator=当前业务上下文.operator)`，
   从返回的 `PromptVersionBaseInfo` 直接取得任务实际使用的 `prompt_version_id`、
   `version_name`、`version_status` 和完整 `prompt_content`，同时取得验证集、商品快照、
   人工/模型标签和原因；校验 Agent 及页面非零 `datasetId`。该提示词既用于解释
   当次结果，也作为确认修改后派生新提示词草稿的基础版本，不再重复查询其他提示词。
4. 按 [rule-loading-policy.md](rule-loading-policy.md) 加载当前查询作用域下有效的
   `price_rule_json`、`special_rule_json` 和 `data_table_json`。必须先进行 Hash
   校验；命中且历史完整 JSON 仍可见时复用，否则按策略取得完整 JSON。调用
   `tool_query_cdn_report(validation_task_id=<任务ID>, operator=当前业务上下文.operator)` 取得报告链接。
5. 结合标题、属性、主图、标签、模型原因、规则快照、映射和任务提示词复核。`is_correct=0` 不证明人工标签正确或提示词错误。
6. 归为提示词规则缺陷、模型未遵循规则、疑似人工标签错误、映射或数据问题、证据不足。仅提示词缺陷可修改，禁止为单个商品写过拟合规则。

## 输出

非提示词缺陷不输出 Diff；CDN 必须是最后一行。

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
<有缺陷：请确认是否按以上建议创建新提示词草稿。>
<无缺陷：该问题不属于提示词规则缺陷，不建议修改。>
验证集报告：[查看 CDN 报告](<真实链接>)
````

无链接时最后一行写：`验证集报告：当前验证任务未返回 CDN 报告链接`。

## 确认后

只有提示词缺陷且用户明确确认，才调用一次
`tool_edit_prompt_skeleton(rule_group_id=<规则组ID>, prompt_version_id=<PromptVersionBaseInfo.prompt_version_id>, prompt_content=<完整新提示词>, change_reason=<Badcase修复原因>, diff_content=<已展示Diff>, source_type=2, operator=当前业务上下文.operator)`。
调用成功后按 `SKILL.md` 的格式返回基础提示词与新草稿的名称和 ID；本轮不再输出 CDN，
也不自动运行验证或发布。写入结果不明确时先只读核实，无法唯一确认则报告结果未知且
不重试写入。
