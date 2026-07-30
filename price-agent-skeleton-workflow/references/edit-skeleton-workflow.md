# 修改提示词

## 查询与提案

1. 调用
   `queryAgentDetail(agent_id=当前业务上下文.agentId, operator=当前业务上下文.operator)`
   查询 Agent 详情；仅当 `base_resp.resp_code=1` 时读取 `data`。所有 MCP 结果按
   `SKILL.md` 的失败与重试规则处理。
2. 调用
   `ToolQueryPromptSkeleton(rule_group_id=<真实规则组ID>, prompt_version_id=<可选提示词ID>, version_name=<可选提示词名称>, operator=当前业务上下文.operator, query_online=<是否查询线上版本>)`。
   根据本轮输入填写参数，不得从历史或“刚才那个”推断；未提供的可选参数不传。
   ID 和名称都有时按 ID 查询，并校验返回名称与用户输入一致。两者均未给时，先提示
   “本轮未指定提示词名称或 ID，是否使用当前线上骨架作为本次修改的基础版本？”；
   用户明确同意后才传 `query_online=true`，不同意则要求提供提示词 ID 或名称。
   每次查询仅当 `base_resp.resp_code=1` 且 `data.prompt_version` 非空时视为成功。
3. 归档、草稿或线上版本均可作为基础版本；任何修改都只能派生新的提示词草稿，不得覆盖基础版本。
4. 调用
   `ToolQueryPriceRule(rule_group_id=<规则组ID>, include_special_rule=1, operator=当前业务上下文.operator)`，
   只取 `data.price_rule_json` 和 `data.special_rule_json`；调用
   `ToolQueryRuleDataTable(rule_group_id=<规则组ID>, operator=当前业务上下文.operator)`，
   只取 `data.data_table_json`。
   结构或 JSON 变化时读取
   [skeleton-format.md](skeleton-format.md)，需要规则范式时才读
   [rule-writing-examples.md](rule-writing-examples.md)。

````markdown
## 提示词修改提案
- 基础提示词：`<名称>`（ID：`<ID>`，状态：`<线上/草稿>`）
- 状态：尚未保存
### 修改原因
<依据、目标和不修改部分>
### Diff
```diff
- <旧规则>
+ <新规则>
```
### 修改后完整提示词
```text
<完整且未省略的提示词>
```
确认后创建新提示词草稿，不覆盖基础版本，也不自动发布。
````

## 确认后

调用一次
`ToolEditPromptSkeleton(rule_group_id=<规则组ID>, prompt_version_id=<基础提示词ID>, prompt_content=<已确认的完整内容>, change_reason=<修改原因>, diff_content=<已展示Diff>, source_type=2, operator=当前业务上下文.operator)`。
仅当 `resp_code=1`、新 ID/版本号大于 0、名称非空且新 ID 不同于基础 ID 时说明成功。写出新提示词名称、ID、版本号及版本关系；本次变更的后续修改、验证和发布使用新 ID，不自动验证或发布。调用超时或返回不完整时，先调用
`queryAgentDetail(agent_id=当前业务上下文.agentId, operator=当前业务上下文.operator)`
取得 `data.latest_draft_prompt_version_id`，再按该 ID 精确查询并比对基础版本、完整内容和创建信息；不能唯一确认就报告结果未知，不再次编辑。
