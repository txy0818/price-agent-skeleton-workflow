# 发布提示词

保存、验证、发布相互独立；Agent 不自动运行验证。所有 MCP 结果按 `SKILL.md` 的失败与重试
规则处理。“保存”“同意修改”“可以”“用这个”**不等于**确认发布。

## 选择版本

1. 用户本轮必须给待发布提示词 ID 或名称；未给则要求补充，不从历史选择。
2. 调用
   `tool_query_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=<待发布ID；无则0>, version_name=<待发布名称；无则空>, query_online=false, operator=当前业务上下文.operator)`
   精确查询。仅当 `resp_code=1` 且 `data.prompt_version` 非空时读取；确认 ID、名称与本轮
   输入一致，`version_status=1` 且 `prompt_content` 非空。
3. 调用同一工具，改传 `prompt_version_id=0, version_name="", query_online=true`
   查询当前线上提示词。仅当 `resp_code=1` 时判断：
   - `data.prompt_version` 非空且 ID>`0` → 确认 `version_status=2` 且 `prompt_content`
     非空，进入「已有线上版本」门禁；
   - `data.prompt_version` 为空或 ID=`0` → 确认无线上提示词，进入「首次发布」门禁；
   - 查询失败、权限不足或结构不完整 → **不得当作“没有线上版本”**，按失败规则停止。
4. 记录两侧 `PromptVersionBaseInfo` 的 ID、名称、版本号、状态、完整内容、最近准确率和 CDN
   链接；待发布完整内容用于计算「线上→待发布」Diff。不得用待发布版本的 `latest_accuracy`
   代替 `compare_prompt_validation` 的比较结果。

## 验证门禁

### 已有线上版本

`compare_prompt_validation(rule_group_id=当前业务上下文.ruleGroupId, online_prompt_version_id=<线上ID>, draft_prompt_version_id=<待发布ID>, operator=当前业务上下文.operator)`

仅当线上/草稿任务均成功、提示词 ID 分别准确、使用同一验证集，且**同时**满足
`accuracy_delta>0` 与待发布版本 `accuracy>=0.8` 时继续。仅有提升但绝对准确率不足 `0.8`
时不得发布。

缺任务时提醒用户用同一验证集手动运行；数据集不同、验证的不是当前草稿、准确率未提升或
`accuracy<0.8` 时停止。新提示词 ID 不得复用旧版本验证结果。

### 首次发布

不调用 `compare_prompt_validation`，也不构造不存在的线上版本 ID。必须取得验证任务 ID：
用户本轮明确提供时优先，否则用上下文 `validationTaskId`；都没有时要求补充，不从历史推断。

`query_validation_task(validation_task_id=<验证任务ID>, operator=当前业务上下文.operator)`

仅当 `resp_code=1`、`data.task` 非空，且同时满足以下条件时继续：
`validation_task_id` 等于请求 ID、`prompt_version_id` 等于待发布 ID、`task_status=3`、
`accuracy>=0.8`。否则提醒用户对当前草稿手动运行验证并达到 `0.8` 以上，然后停止。
当前草稿是独立的新版本 ID，不得使用其他草稿或基础版本的验证结果。

## 展示与确认

门禁通过后展示 Agent、待发布提示词名称/ID/版本号，以及：

- 已有线上版本：展示「线上→待发布」的 Diff，**不展开待发布全文**；同时给出两个任务 ID、
  共同验证集、两侧准确率和提升值。发布决策看的是相对线上的变更，因此以 Diff 为主。
- 首次发布：无线上基线可做 Diff，完整展示待发布内容；标注“当前无线上版本”，给出验证任务
  ID、验证集 ID、准确率和 `report_cdn_url`。

用户明确要求「展开完整提示词」时才完整输出待发布全文且不得省略。然后询问：

“确认将提示词 `<version_name>`（ID：`<prompt_version_id>`）的以上内容发布到线上吗？”

本轮不发布。用户明确确认后：

1. 按「选择版本」的两个查询重新取数，确认待发布提示词的 ID、状态和完整内容未变化，
   并重新判断当前是否存在线上提示词。
2. 重新执行对应门禁：
   - 仍有同一线上提示词 → 用重新查询得到的两个 ID 再执行 `compare_prompt_validation`，
     确认门禁仍通过；
   - 仍无线上提示词 → 用同一验证任务 ID 再次调用 `query_validation_task`，确认仍满足
     首次发布全部条件且 `accuracy>=0.8`；
   - 原无线上但确认前出现了线上提示词，或线上 ID 已变化 → **停止发布**，按最新线上版本
     重新执行完整对比和确认流程。
3. 调用一次
   `tool_publish_prompt_skeleton(prompt_version_id=<待发布ID>, operator=当前业务上下文.operator)`。
4. 仅当 `resp_code=1` 且 `current_prompt_version_id=<待发布ID>` 时说明成功。超时或结果
   不完整时，用 `query_online=true` 查询当前线上提示词；仅当其 `prompt_version_id` 等于
   待发布 ID 时确认已发布，否则报告结果未知且不再次发布。
