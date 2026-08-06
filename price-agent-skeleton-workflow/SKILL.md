---
name: price-agent-skeleton-workflow
description: 为 PriceStudio 初始化、修改或保存同款判定提示词（骨架），分析验证任务或用户描述的 Badcase，或按业务规则调整提示词。用户提到“提示词”“骨架”“Badcase”时使用；任何写入必须先展示提案并取得明确确认。
---

# PriceStudio 提示词工作流

## 硬约束

- `agentId` 非零、`operator` 非空；MCP schema 有 `operator` 时原样传入。缺失即停止。
- 进入初始化流程后，在生成任何骨架正文前必须完整读取
  [skeleton-format.md](references/skeleton-format.md)；不得凭模型记忆、历史回答或示例推测标准格式。
  未读取成功时停止，不得进入生成、校验或提案。
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
- `writeMode` 一旦锁定，校验结果、`diffRecordId` 和用户确认都不得改变它；两种模式确认后
  都只调用 `tool_edit_prompt_skeleton`，并传锁定的 `selectedPromptVersionId`。
- 提案前执行 [shared-steps.md](references/shared-steps.md) 的 S3；确认后写入内容必须与校验内容
  完全一致。修改类只展示服务端 `data.diff_content`，用户要求时才展开全文。
- 线上版本禁止初始化和修改。草稿、归档版本按内容路由：空内容可初始化，非空内容可修改。
- `tool_edit_prompt_skeleton` 会在事务内查询基础版本正文：仍为空则原地填写，返回的
  `new_prompt_version_id=基础ID`，Diff 的基础/新版本 ID 也都为该 ID；已非空则派生新草稿，
  返回的 `new_prompt_version_id` 必须与基础 ID 不同。
- 两种结果都必须满足：返回成功、`new_prompt_version_id>0`、`version_no>0`、
  `new_prompt_name` 非空；后续修改、验证和发布一律使用返回的 `new_prompt_version_id`。
- 不自动验证或发布。

写入前必须再次检查：`writeMode` 与 MCP 匹配，且 `selectedPromptVersionId>0`。任一不满足就停止并
重新精确查询，禁止把版本 ID 留空、传 0、改用 `save_prompt_draft` 或从其他字段猜 ID。

## 路由

每次只读取目标 workflow，并同时读取 [shared-steps.md](references/shared-steps.md) 和
[base-version-policy.md](references/base-version-policy.md)：

- 初始化：必须同时完整读取
  [initialize-skeleton-workflow.md](references/initialize-skeleton-workflow.md) 和
  [skeleton-format.md](references/skeleton-format.md)；后者是生成前置必读，不是按需参考。
- 修改：[edit-skeleton-workflow.md](references/edit-skeleton-workflow.md)
- 单条 Badcase：[badcase-single-workflow.md](references/badcase-single-workflow.md)
- 整次任务 Badcase：[badcase-task-workflow.md](references/badcase-task-workflow.md)
- 用户描述 Badcase：[badcase-description-workflow.md](references/badcase-description-workflow.md)
- 纯查询：不加载 workflow，只调用对应只读 MCP。

目标语义不清但已给提示词 ID/名称时，可精确只读查询后按内容是否为空路由；其他情况先澄清。
S2 读取 [rule-loading-policy.md](references/rule-loading-policy.md)。生成全文时按 workflow 读取
[rule-transformation-guide.md](references/rule-transformation-guide.md)；初始化必须已按上述前置门槛完整读取
[skeleton-format.md](references/skeleton-format.md)。枚举存疑时读取 [enums.md](references/enums.md)；
[full-skeleton-example.md](references/full-skeleton-example.md) 仅在用户明确要求完整示例时读取。
