# 验证任务 Badcase 分析

## 查询与复核

1. 调用
   `query_agent_detail(agent_id=当前业务上下文.agentId, operator=当前业务上下文.operator)`
   查询 Agent 详情；仅当 `base_resp.resp_code=1` 时读取 `data`。所有 MCP 结果按
   `SKILL.md` 的失败与重试规则处理。
2. 必须取得验证任务 ID：用户本轮明确提供时优先，否则使用上下文 `validationTaskId`；两者都没有时要求补充。CDN 不能代替任务 ID。
3. 调用
   `tool_query_validation_result(validation_task_id=<任务ID>, label_filter=3, page_size=20, operator=当前业务上下文.operator)`。
   从返回的 `PromptVersionBaseInfo` 直接取得任务实际使用的 `prompt_version_id`、
   `version_name`、`version_status` 和完整 `prompt_content`，同时取得验证集、商品快照、
   标签和原因；校验 Agent。该提示词既用于解释本次验证结果，也作为确认修改后
   派生新提示词草稿的基础版本，不再重复查询其他提示词。一次最多分析 20 条；还有更多时
   报告剩余数。用户要求继续时查询下一页；必须保持同一任务 ID 和
   `PromptVersionBaseInfo.prompt_version_id`，并按 `validation_case_id` 去重。
4. 按 [rule-loading-policy.md](rule-loading-policy.md) 加载当前查询作用域下有效的
   `price_rule_json`、`special_rule_json` 和 `data_table_json`。必须先进行 Hash
   校验；命中且历史完整 JSON 仍可见时复用，否则按策略取得完整 JSON。调用
   `tool_query_cdn_report(validation_task_id=<任务ID>, operator=当前业务上下文.operator)` 取得报告链接。
5. 逐条结合标题、属性、主图、标签、模型原因、规则快照、映射和任务提示词复核。`is_correct=0` 不证明人工标签正确或提示词错误。
6. 归为提示词规则缺陷、模型未遵循规则、疑似人工标签错误、映射或数据问题、证据不足。仅提示词缺陷可修改，禁止针对个别商品写过拟合规则。

## 分批分析

每批结果作为当前会话中的阶段分析，不写入提示词表或 Diff 表，也不调用
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
| Badcase ID | 人工/模型标签 | 类型 | 关键证据 | 建议 |
|---|---|---|---|---|
| `<case_id>` | <标签> | <类型> | <证据> | <建议> |
### 本批候选修改
- 缺陷与样本：<仅确认的提示词缺陷及 case ID>
- 候选建议：<待最终整合的规则修改；无缺陷写“本批无提示词修改建议”>
- 不写入提示词：<标签、数据、映射或执行问题>
- 与前批关系：<新增/重复/补充/冲突；首批写“首批”>
<超过20：本次已分析20条，剩余 <remaining> 条；可回复“继续分析”。>
<需要整合时：可回复“整合修改建议给我一个草稿”。>
验证集报告：[查看 CDN 报告](<真实链接>)
````

无链接时最后一行写：`验证集报告：当前验证任务未返回 CDN 报告链接`。

## 整合修改建议

用户要求“整合修改建议给我一个草稿”时：

1. 只合并当前会话中任务 ID、验证集 ID 和基础提示词 ID 均一致的阶段分析；按
   `validation_case_id` 去重。会话中阶段结果缺失时，不凭记忆补写；重新查询缺失批次或
   明确说明只能基于当前可见批次整合。
2. 合并指向同一规则的重复建议；结合商品证据、比价规则和映射处理冲突。疑似人工标签错误、
   模型执行、映射或数据问题以及证据不足不得写入提示词。
3. 基于 `PromptVersionBaseInfo.prompt_content` 生成一份合并后的完整提示词和一份最终
   Diff。说明覆盖的批次、Badcase ID、尚未分析的数量和回归风险。
4. 展示最终 Diff 与完整提示词，声明“尚未保存”，询问用户是否确认创建一个新提示词草稿；
   此时不得调用写入 MCP。

## 确认创建后

只有最终整合结果存在提示词缺陷，且用户在看到最终 Diff 和完整提示词后明确确认，才调用一次
`tool_edit_prompt_skeleton(rule_group_id=<规则组ID>, prompt_version_id=<PromptVersionBaseInfo.prompt_version_id>, prompt_content=<完整新提示词>, change_reason=<Badcase修复原因>, diff_content=<已展示Diff>, source_type=2, operator=当前业务上下文.operator)`。
`diff_content` 使用最终展示且经用户确认的 Diff 正文，不含 Markdown 围栏。仅调用一次并创建
一个新草稿，不为各批次分别创建草稿。仅当返回成功、新 ID/版本号大于 0、名称非空且新 ID
不同于基础 ID 时说明成功；在 CDN 前写出新版本关系，不自动发布。写入结果不明确时先只读
核实，无法唯一确认则报告结果未知且不重试写入。
