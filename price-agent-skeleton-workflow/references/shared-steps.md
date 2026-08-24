# 共享步骤

本文件只保留被 `SKILL.md` 独立路由使用的 `[S1]` 展开全文和 `[S2]` 确认写入。

## S1 展开完整提示词

用户紧邻有效提案回复“展开完整提示词”“查看完整提示词”或等价表达时，只执行本分支：上一条必须
来自真实校验工具调用，且可见提案中的 `diff_id` 与该次响应的非零 `data.diffRecordId` 一致、
`data.valid=true`，该次请求参数中的 `promptContent` 非空。

- `diff_id`：只从紧邻的上一条可见提案读取，用于定位并核对同次校验调用；展开结果不输出
  `diff_id` 或 `diffContent`。
- `promptContent`：只从生成该提案的同次 `tool_validate_prompt_skeleton` 请求参数读取；不得从
  `diffContent`、当前提示词、历史提案或模型记忆反推、拼接或重新生成。
- 找不到匹配的校验调用、无法取得其请求 `promptContent`，或任一门禁不满足时，只返回：
  “上一条不是可展开的有效提示词提案，请重新发起提示词修改。”

满足条件时，不调用 MCP、不重新查询、不重新生成、不解释，只逐字符输出锁定的同一个
`promptContent`。使用四反引号 `text` 围栏，确保正文内部三反引号保持原样：

`````markdown
````text
<逐字符复制同次校验工具调用已锁定的 promptContent>
````
`````

展开只用于查看，不构成确认。由于确认只接受紧邻提案的用户消息，展开后原提案不能再直接确认；
用户随后要求写入时须重新生成提案。

## S2 确认后写入草稿

### 确认门禁

本节是确认意图命中后的唯一执行路径。进入后禁止输出任何自然语言或执行其他工具，必须先按下方
参数调用 `tool_edit_prompt_skeleton`。

### 参数与调用

- `promptDiffRecordId`：紧邻提案同次校验工具调用的非零 ID。
- `promptVersionId`：进入 `[S2]` 时，沿用 `SKILL.md` 入口从当前确认回合业务上下文绑定的
  `currentPromptVersionId`，并记录为 `confirmationPromptVersionId`。入口值显式为非负整数时
  直接使用；字段未注入、为空或仍为模板占位符时，统一令 `confirmationPromptVersionId=0`。
  不得从上一条提案的“基础提示词 ID”、历史消息、上一轮工具参数、Diff 记录、版本名称、
  `newPromptVersionId` 或模型记忆补值。只要存在有效非零 `promptDiffRecordId`，就不得因
  `promptVersionId` 缺失而阻止确认写入，必须以 `promptVersionId=0` 调用；Diff 与版本关联由服务端
  校验。仅当上下文显式提供负数或非数字值时停止，只回复：“当前确认回合提供的 promptVersionId
  非法，未执行提示词写入。”
- `sourceType`：INITIALIZE=3，EDIT=2。原提案模式只决定此字段。
- 不传 `promptContent`；服务端以 Diff 记录正文为准。

`confirmationPromptVersionId>0` 时，先调用一次：

`tool_query_prompt_skeleton(ruleGroupId=上下文.ruleGroupId, promptVersionId=confirmationPromptVersionId, versionName="", queryOnline=false, queryLatest=false, operator=上下文.operator)`

只从该次响应记录基础版本的 `versionName/id/versionNo`；查询失败或响应版本 ID 与
`confirmationPromptVersionId` 不一致时不写入。不得按“最新/线上/归档/草稿/ID/名称/版本”等用户
文字另行查找或切换版本，不得从上一条提案、历史消息、上一轮工具参数、Diff 记录、版本名称、
`newPromptVersionId` 或模型记忆补值。`confirmationPromptVersionId=0` 时不调用查询工具，基础提示词
记为“无（初始化）”。

前置查询通过或无需查询后，只调用一次写入工具：

`tool_edit_prompt_skeleton(ruleGroupId=上下文.ruleGroupId, promptVersionId=confirmationPromptVersionId, operator=上下文.operator, sourceType=<3或2>, promptDiffRecordId=<紧邻提案Diff ID>)`

调用返回前禁止产生用户可见文字。调用后只能依据本次真实 `baseResp`、`data`、`result`、`error_msg` 输出：成功
时使用下方成功模板；业务拒绝时逐字转述真实原因；调用失败或响应不完整时报告真实工具错误。禁止
用模型生成的理由替代工具响应。

服务端校验 Diff 基础版本关联关系。不匹配、已处理或失效时据实报告，不切换版本、不重新校验、
不传正文、不重试。超时或响应不完整时，可用 `data.card.latestDraftPromptVersionId`
只读核实；不能唯一确认就报告结果未知，禁止重试写入。

### 成功回复

字段只取写入前基础版本查询和本次响应，模板外不加内容：

```markdown
## 提示词草稿创建成功
- 修改建议 ID（diff_id）：`<本次 promptDiffRecordId>`
- 基础提示词：`<versionName>`（ID：`<promptVersionId>`，版本号：`<versionNo>`）
- 新提示词：`<data.newPromptName>`（ID：`<data.newPromptVersionId>`，版本号：`<data.versionNo>`）
- 状态：草稿已创建
```

初始化的基础提示词写 `无（promptVersionId=0，初始化）`；`data.promptExists=true` 时新提示词
字段仍取响应，状态改为“已存在，未新建”。不得总结变更内容；返回的新 ID 仅展示，不更新上下文。
