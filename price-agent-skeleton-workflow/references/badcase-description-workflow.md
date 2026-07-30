# 用户描述的 Badcase

本流程用于用户直接用文字、图片或商品信息描述 Badcase，但没有提供真实验证任务与 Case ID 的场景。

## 提取与查询

1. 本轮输入必须明确提供基础提示词 ID 或提示词名称；两者都没有时立即停止并回复：
   `请提供需要分析和修改的提示词 ID 或提示词名称。`
   不得默认选择线上提示词，也不得从历史消息、“当前提示词”或“刚才那个”等表述推断。
   ID 和名称同时提供时，必须精确查询并确认指向同一版本；不一致时提示用户修正。
2. 从本轮输入提取左右商品标题、属性、主图、人工标签、模型结论和模型理由；不得把缺失信息补写成事实。
3. 调用
   `ToolQueryPromptSkeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=<本轮提示词ID；未提供则0>, version_name=<本轮提示词名称；未提供则空>, operator=当前业务上下文.operator)`。
   仅当 `base_resp.resp_code=1` 时读取 `prompt_version`。
4. 调用
   `ToolQueryPriceRule(rule_group_id=<规则组ID>, include_special_rule=1, operator=当前业务上下文.operator)`，
   只取 `data.price_rule_json` 和 `data.special_rule_json`；调用
   `ToolQueryRuleDataTable(rule_group_id=<规则组ID>, operator=当前业务上下文.operator)`，
   只取 `data.data_table_json`。
5. 判断证据是否足以复核争议比价项：涉及外观必须有左右主图；涉及标题、属性或映射必须有对应原始信息；必须知道人工标签与模型结论。信息不足时列出缺失项并停止，不生成 Diff、完整提示词或草稿。
6. 信息充分时结合规则、映射和基础提示词复核。Badcase 可能是提示词缺陷、模型未遵循规则、疑似人工标签错误、映射或数据问题、证据不足，不得默认归因于提示词。

## 输出

````markdown
## Badcase 初步分析
- Agent：`<名称；查不到时写ID>`
- 基础提示词：`<名称>`（ID：`<ID>`）
- 分析范围：用户直接描述，未关联真实验证任务
### 样本与证据
- 左侧商品：<标题、属性和主图信息>
- 右侧商品：<标题、属性和主图信息>
- 人工标签 / 模型结论：<标签>
- 模型理由：<理由>
### 结论
- 类型：<五类之一>
- 依据：<规则、映射和商品证据>
- 是否建议修改提示词：<是/否>
### 修改建议
<仅在确认属于提示词缺陷时输出建议>
```diff
- <原规则>
+ <建议规则>
```
### 修改后完整提示词
```text
<完整且未省略的提示词>
```
<有缺陷：以上为修改提案；确认无误请回复“确认创建提示词草稿”。>
<无缺陷：当前问题不属于提示词规则缺陷，不建议修改提示词。>
````

本场景没有真实验证任务，不输出或编造 CDN 链接。无提示词缺陷时省略 Diff 和完整提示词。

## 确认后

只有证据充分、确认属于提示词缺陷且用户明确确认，才调用一次
`ToolEditPromptSkeleton(rule_group_id=<规则组ID>, prompt_version_id=<基础提示词ID>, prompt_content=<已确认的完整内容>, change_reason=<Badcase修复原因>, diff_content=<已展示Diff>, source_type=2, operator=当前业务上下文.operator)`。
仅当返回成功、新 ID/版本号大于 0、名称非空且新 ID 不同于基础 ID 时说明新提示词草稿创建成功；不自动运行验证或发布。
