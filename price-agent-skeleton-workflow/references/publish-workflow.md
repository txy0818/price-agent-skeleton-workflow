# 发布提示词

保存、验证、发布相互独立；Agent 不自动运行验证。所有 MCP 结果按 `SKILL.md`
的失败与重试规则处理。

## 选择版本

1. 用户本轮必须给待发布提示词 ID 或名称；未给则要求补充，不从历史选择。
2. 调用
   `ToolQueryPromptSkeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=<待发布提示词ID；未提供则0>, version_name=<待发布提示词名称；未提供则空>, query_online=false, operator=当前业务上下文.operator)`，
   精确查询本次待发布提示词。仅当 `base_resp.resp_code=1` 且
   `data.prompt_version` 非空时读取第一份 `PromptVersionBaseInfo`；确认 ID、名称与本轮输入
   一致，`version_status=1` 且 `prompt_content` 非空。
3. 调用
   `ToolQueryPromptSkeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=0, version_name="", query_online=true, operator=当前业务上下文.operator)`，
   查询当前线上提示词。仅当 `base_resp.resp_code=1` 且 `data.prompt_version` 非空时读取
   第二份 `PromptVersionBaseInfo`；确认 `version_status=2` 且 `prompt_content` 非空。
4. 记录两份 `PromptVersionBaseInfo` 的 ID、名称、版本号、状态、完整内容、最近准确率和
   CDN 链接；不得用待发布版本的 `latest_accuracy` 直接代替发布门禁比较结果。

## 验证门禁

调用
`ComparePromptValidation(rule_group_id=当前业务上下文.ruleGroupId, online_prompt_version_id=<线上PromptVersionBaseInfo.prompt_version_id>, draft_prompt_version_id=<待发布PromptVersionBaseInfo.prompt_version_id>, operator=当前业务上下文.operator)`。
仅当线上/草稿任务均成功、提示词 ID 分别准确、使用同一验证集且
`accuracy_delta>0` 时继续。

缺任务时提醒用户用同一验证集手动运行；数据集不同、验证的不是当前草稿或准确率未提升时停止。新提示词 ID 不得复用旧版本验证结果。

## 展示与确认

门禁通过后完整展示 Agent、待发布提示词名称/ID/版本号、两个任务 ID、共同验证集、两侧准确率、提升值和待发布完整内容，询问：

“确认将提示词 `<待发布PromptVersionBaseInfo.version_name>`（ID：
`<待发布PromptVersionBaseInfo.prompt_version_id>`）的以上完整内容发布到线上吗？”

本轮不发布。用户明确确认后：

1. 按“选择版本”的两个 `ToolQueryPromptSkeleton` 调用重新查询待发布提示词和当前线上
   提示词，确认两个 ID、状态和完整内容均未变化。
2. 使用重新查询得到的两个提示词 ID 再执行 `ComparePromptValidation`，确认门禁仍通过。
3. 调用一次
   `ToolPublishPromptSkeleton(prompt_version_id=<待发布PromptVersionBaseInfo.prompt_version_id>, operator=当前业务上下文.operator)`。
4. 仅当 `resp_code=1` 且
   `current_prompt_version_id=<待发布PromptVersionBaseInfo.prompt_version_id>` 时说明成功。
   调用超时或结果不完整时，调用
   `ToolQueryPromptSkeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=0, version_name="", query_online=true, operator=当前业务上下文.operator)`
   查询当前线上提示词；仅当
   `data.prompt_version.prompt_version_id=<待发布PromptVersionBaseInfo.prompt_version_id>`
   时确认已发布，否则报告结果未知且不再次发布。

“保存”“同意修改”“可以”“用这个”不等于确认发布。
