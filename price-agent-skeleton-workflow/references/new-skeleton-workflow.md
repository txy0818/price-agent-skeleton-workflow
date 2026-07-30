# 新建提示词

只创建提示词草稿。

## 查询与生成

1. 调用
   `queryAgentDetail(agent_id=当前业务上下文.agentId, operator=当前业务上下文.operator)`
   查询 Agent 详情。仅当 `base_resp.resp_code=1` 时读取 `data.card.agent_name`、
   `data.card.rule_group_id`、`data.card.category_ids_str`、`data.current_prompt_version_id`
   和 `data.latest_draft_prompt_version_id`。所有 MCP 结果按 `SKILL.md` 的失败与重试规则处理。
2. 线上和草稿 ID 都为 `0` 时从零生成；任一非零时先查询并展示已有版本，询问“基于指定版本修改”还是“仍从零创建”。确认前不生成、不写入。
3. 调用
   `ToolQueryPriceRule(rule_group_id=<规则组ID>, include_special_rule=1, operator=当前业务上下文.operator)`，
   只取 `data.price_rule_json` 和 `data.special_rule_json`；调用
   `ToolQueryRuleDataTable(rule_group_id=<规则组ID>, operator=当前业务上下文.operator)`，
   只取 `data.data_table_json`。
4. 从零生成时读取 [rule-transformation-guide.md](rule-transformation-guide.md) 和
   [skeleton-format.md](skeleton-format.md)；需要更多规则表达范式时才读取
   [rule-writing-examples.md](rule-writing-examples.md)。完整参考示例
   [full-skeleton-example.md](full-skeleton-example.md) 默认不加载，仅在用户明确要求查看完整示例时读取。
5. 展示提案前调用
   `ToolValidatePromptSkeleton(prompt_content=<生成的完整提示词>, operator=当前业务上下文.operator)`。
   仅当 `base_resp.resp_code=1` 且 `data.valid=true` 时展示提案。`data.valid=false`
   时根据 `data.errors` 修正完整提示词后重新校验，最多重试 2 次；仍未通过或 MCP
   调用失败时，返回校验错误，不展示提案，也不调用任何写入或发布接口。

## 提案

````markdown
## 新建同款判定提示词提案
- Agent：`<名称；查不到时写ID>`
- 提示词：`待创建（未生成ID）`
- 状态：尚未保存
### 生成说明
- 判定目标：判断左右商品是否同款
- 关键规则：<关键匹配、否决和例外>
- 风险：<待验证内容；没有写“无”>
### 完整提示词
```text
<完整且未省略的提示词>
```
以上仅为提案。确认无误请回复：**确认创建提示词草稿**。确认后只创建提示词草稿，不发布。
````

## 确认后

调用一次
`ToolEditPromptSkeleton(rule_group_id=<规则组ID>, prompt_version_id=0, prompt_content=<已确认的完整内容>, change_reason=<新建原因>, diff_content="", source_type=2, operator=当前业务上下文.operator)`。

仅当 `resp_code=1`、新 ID/版本号大于 0 且名称非空时说明成功。写出新提示词名称、ID、版本号，提醒本次创建的后续修改、验证和发布使用新 ID；不自动验证或发布。调用超时或返回不完整时，先调用
`queryAgentDetail(agent_id=当前业务上下文.agentId, operator=当前业务上下文.operator)`
取得 `data.latest_draft_prompt_version_id`，再按该 ID 精确查询并比对规则组、完整内容和创建信息；不能唯一确认就报告结果未知，不再次创建。
