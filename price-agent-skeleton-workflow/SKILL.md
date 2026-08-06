---
name: price-agent-skeleton-workflow
description: 为 PriceStudio 初始化、修改或保存同款判定提示词（骨架），分析验证任务或用户描述的 Badcase，或按业务规则调整提示词。用户提到“提示词”“骨架”“Badcase”时使用；任何写入必须先展示提案并取得明确确认。
---

# PriceStudio 提示词工作流

## 硬约束

- `agentId` 非零、`operator` 非空；MCP schema 有 `operator` 时原样传入。缺失即停止。
- `ruleGroupId` 使用当前业务上下文；缺失时仅可从本轮 `query_agent_detail.data.card.rule_group_id`
  取得，之后固定，不被其他返回覆盖。
- 不猜 ID、规则或样本；流程必需的可选 ID 缺失时要求用户本轮补充。
- 只读 MCP 的临时错误重试一次；业务错误不重试。写入失败或超时绝不重试写入，只读核实；
  无法唯一确认内容一致时报告结果未知。

## 提示词选择与写入

- 初始化、修改和 Badcase 修改必须取得用户本轮指定的提示词 ID 或名称，并通过
  `tool_query_prompt_skeleton` 精确查询；不得用最新草稿、线上版本或历史选择替代。
- 查询成功后立即保存不可变的 `selectedPromptVersionId=data.prompt_version.prompt_version_id`；
  必须大于 0。按内容锁定 `writeMode`：空 → `INITIALIZE`，非空 → `EDIT`。Badcase 遇空内容停止。
- `writeMode` 一旦锁定，校验结果、`diffRecordId` 和用户确认都不得改变它：
  `INITIALIZE` 只能调用 `save_prompt_draft`；`EDIT` 只能调用 `tool_edit_prompt_skeleton`。
- 提案前执行 [shared-steps.md](references/shared-steps.md) 的 S3；确认后写入内容必须与校验内容
  完全一致。修改类只展示服务端 `data.diff_content`，用户要求时才展开全文。
- 线上版本禁止初始化和修改。草稿、归档版本按内容路由：空内容可初始化，非空内容可修改。
- 初始化确认后用 `save_prompt_draft` 原地填写指定空版本；成功必须满足：返回成功、
  `prompt_version_id=基础ID`、`version_no>0`、`version_name` 非空。
- 修改或 Badcase 确认后用 `tool_edit_prompt_skeleton` 派生草稿；成功必须满足：返回成功、
  `new_prompt_version_id>0` 且不同于基础 ID、`version_no>0`、`new_prompt_name` 非空。
- 不自动验证或发布。

写入前必须再次检查：`writeMode` 与 MCP 匹配，且 `selectedPromptVersionId>0`。任一不满足就停止并
重新精确查询，禁止把版本 ID 留空、传 0、改用另一个写入 MCP 或从其他字段猜 ID。

## 路由

每次只读取目标 workflow，并同时读取 [shared-steps.md](references/shared-steps.md) 和
[base-version-policy.md](references/base-version-policy.md)：

- 初始化：[initialize-skeleton-workflow.md](references/initialize-skeleton-workflow.md)
- 修改：[edit-skeleton-workflow.md](references/edit-skeleton-workflow.md)
- 单条 Badcase：[badcase-single-workflow.md](references/badcase-single-workflow.md)
- 整次任务 Badcase：[badcase-task-workflow.md](references/badcase-task-workflow.md)
- 用户描述 Badcase：[badcase-description-workflow.md](references/badcase-description-workflow.md)
- 纯查询：不加载 workflow，只调用对应只读 MCP。

目标语义不清但已给提示词 ID/名称时，可精确只读查询后按内容是否为空路由；其他情况先澄清。
S2 读取 [rule-loading-policy.md](references/rule-loading-policy.md)。生成全文时按 workflow 读取
[rule-transformation-guide.md](references/rule-transformation-guide.md) 与
[skeleton-format.md](references/skeleton-format.md)。枚举存疑时读取 [enums.md](references/enums.md)；
[full-skeleton-example.md](references/full-skeleton-example.md) 仅在用户明确要求完整示例时读取。
