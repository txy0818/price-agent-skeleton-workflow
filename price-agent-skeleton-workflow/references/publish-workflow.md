# 发布提示词

保存、验证、发布相互独立；Agent 不自动运行验证。所有 MCP 结果按 `SKILL.md`
的失败与重试规则处理。

## 选择版本

1. 用户本轮必须给待发布提示词 ID 或名称；未给则要求补充，不从历史选择。
2. 调用
   `tool_query_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=<待发布提示词ID；未提供则0>, version_name=<待发布提示词名称；未提供则空>, query_online=false, operator=当前业务上下文.operator)`，精确查询本次待发布提示词。仅当 `base_resp.resp_code=1` 且`data.prompt_version` 非空时读取；确认 ID、名称与本轮输入一致，`version_status=1` 且 `prompt_content` 非空。
3. 调用
   `tool_query_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=0, version_name="", query_online=true, operator=当前业务上下文.operator)`，
   查询当前线上提示词。仅当 `base_resp.resp_code=1` 时判断结果：
   - `data.prompt_version` 非空且 ID 大于 `0`：确认`version_status=2` 且 `prompt_content` 非空，进入“已有线上版本”门禁；
   - `data.prompt_version` 为空或 ID 等于 `0`：确认当前没有线上提示词，进入“首次发布”门禁；
   - 查询失败、权限不足或返回结构不完整：不得当作“没有线上版本”，按失败规则停止。
4. 记录已取得的 `PromptVersionBaseInfo` 的 ID、名称、版本号、状态、完整内容、最近准确率
   和 CDN 链接。已有线上版本时，不得用待发布版本的 `latest_accuracy` 直接代替
   `compare_prompt_validation` 的比较结果。

## 验证门禁

### 已有线上版本

调用
`compare_prompt_validation(rule_group_id=当前业务上下文.ruleGroupId, online_prompt_version_id=<线上PromptVersionBaseInfo.prompt_version_id>, draft_prompt_version_id=<待发布PromptVersionBaseInfo.prompt_version_id>, operator=当前业务上下文.operator)`。
仅当线上/草稿任务均成功、提示词 ID 分别准确、使用同一验证集且
`accuracy_delta>0` 时继续。

缺任务时提醒用户用同一验证集手动运行；数据集不同、验证的不是当前草稿或准确率未提升时停止。新提示词 ID 不得复用旧版本验证结果。

### 首次发布

没有线上提示词时不调用 `compare_prompt_validation`，也不构造不存在的线上版本 ID。
必须取得验证任务 ID：用户本轮明确提供时优先，否则使用当前上下文
`validationTaskId`；两者都没有时要求补充，不从历史推断。调用
`query_validation_task(validation_task_id=<验证任务ID>)`。

仅当 `base_resp.resp_code=1`、`data.task` 非空，并同时满足以下条件时继续：

- `data.task.validation_task_id` 等于请求的验证任务 ID；
- `data.task.prompt_version_id` 等于待发布提示词 ID；
- `data.task.task_status=3`；
- `data.task.accuracy>=0.8`。

任务不存在、未成功、验证的不是当前草稿或准确率低于 `0.8` 时，提醒用户对当前草稿
手动运行验证并达到 `0.8` 及以上，然后停止。当前草稿是独立的新版本 ID，不得使用其他
草稿或基础版本的验证结果。

## 展示与确认

门禁通过后完整展示 Agent、待发布提示词名称/ID/版本号和待发布完整内容：

- 已有线上版本：展示两个任务 ID、共同验证集、两侧准确率和提升值；
- 首次发布：明确标注“当前无线上版本”，展示验证任务 ID、验证集 ID、准确率和
  `report_cdn_url`。

然后询问：

“确认将提示词 `<待发布PromptVersionBaseInfo.version_name>`（ID：
`<待发布PromptVersionBaseInfo.prompt_version_id>`）的以上完整内容发布到线上吗？”

本轮不发布。用户明确确认后：

1. 按“选择版本”的两个 `tool_query_prompt_skeleton` 调用重新查询待发布提示词和当前线上
   提示词，确认待发布提示词的 ID、状态和完整内容未变化，并重新判断当前是否存在
   线上提示词。
2. 重新执行对应门禁：
   - 仍有同一线上提示词：使用重新查询得到的两个提示词 ID 再执行
     `compare_prompt_validation`，确认门禁仍通过；
   - 仍无线上提示词：使用同一验证任务 ID 再次调用 `query_validation_task`，确认任务
     仍满足首次发布的全部条件且 `accuracy>=0.8`；
   - 原来无线上提示词但确认发布前出现了线上提示词，或线上提示词 ID 已变化：停止发布，
     按最新线上版本重新执行完整对比和确认流程。
3. 调用一次
   `tool_publish_prompt_skeleton(prompt_version_id=<待发布PromptVersionBaseInfo.prompt_version_id>, operator=当前业务上下文.operator)`。
4. 仅当 `resp_code=1` 且
   `current_prompt_version_id=<待发布PromptVersionBaseInfo.prompt_version_id>` 时说明成功。
   调用超时或结果不完整时，调用
   `tool_query_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=0, version_name="", query_online=true, operator=当前业务上下文.operator)`
   查询当前线上提示词；仅当
   `data.prompt_version.prompt_version_id=<待发布PromptVersionBaseInfo.prompt_version_id>`
   时确认已发布，否则报告结果未知且不再次发布。

“保存”“同意修改”“可以”“用这个”不等于确认发布。
