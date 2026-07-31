# 用户直接提出规则修改要求

本流程用于用户直接要求新增、删除或调整某项判定规则，不分析验证结果。

## 查询与判断

1. 本轮输入必须明确提供基础提示词 ID 或提示词名称；两者都没有时立即停止并回复：
   `请提供需要修改的提示词 ID 或提示词名称。`
   不得默认选择线上提示词，也不得从历史消息、“当前提示词”或“刚才那个”等表述推断。
   ID 和名称同时提供时，必须精确查询并确认指向同一版本；不一致时提示用户修正。
2. 调用
   `tool_query_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=<本轮提示词ID；未提供则0>, version_name=<本轮提示词名称；未提供则空>, operator=当前业务上下文.operator)`。
   仅当 `base_resp.resp_code=1` 时读取 `data.prompt_version`；取得非空 `prompt_content`。
3. 归档、草稿或线上版本均可作为基础版本；任何修改都只能派生新的提示词草稿，不得覆盖基础版本。
4. 不调用验证任务或验证结果查询。按
   [rule-loading-policy.md](rule-loading-policy.md) 加载当前查询作用域下有效的
   `price_rule_json`、`special_rule_json` 和 `data_table_json`。必须先进行 Hash
   校验；命中且历史完整 JSON 仍可见时复用，否则按策略取得完整 JSON。对照这些规则、
   映射与目标提示词判断用户要求是否合理。
5. 要求与业务规则冲突、缺少关键条件或可能扩大误判时，说明冲突和需要确认的问题，停止且不生成草稿。
6. 要求合理时生成最小必要修改，保留无关规则；结构或 JSON 变化时读取 [skeleton-format.md](skeleton-format.md)，需要规则范式时才读取 [rule-writing-examples.md](rule-writing-examples.md)。

## 提案

````markdown
## 规则修改提案
- Agent：`<名称；查不到时写ID>`
- 基础提示词：`<名称>`（ID：`<ID>`，状态：`<线上/草稿>`）
- 状态：尚未保存
### 合理性判断
- 用户要求：<原始修改目标>
- 规则与映射依据：<依据>
- 影响与风险：<适用范围、冲突和回归项>
### Diff
```diff
- <旧规则>
+ <新规则>
```
### 修改后完整提示词
```text
<完整且未省略的提示词>
```
以上仅为修改提案。确认无误请回复：**确认创建提示词草稿**。
````

## 确认后

调用一次
`tool_edit_prompt_skeleton(rule_group_id=<规则组ID>, prompt_version_id=<基础提示词ID>, prompt_content=<已确认的完整内容>, change_reason=<规则修改原因>, diff_content=<已展示Diff>, source_type=2, operator=当前业务上下文.operator)`。
仅当返回成功、新 ID/版本号大于 0、名称非空且新 ID 不同于基础 ID 时说明新提示词草稿创建成功。写出新提示词名称、ID、版本号及与基础版本的关系；后续修改、验证和发布使用新 ID，不自动运行验证或发布。
