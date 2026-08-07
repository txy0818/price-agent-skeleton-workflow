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

- 提示词版本只取当前业务上下文的 `promptVersionId`，不再要求用户提供 ID 或名称，也不得从
  用户文字、历史消息或查询结果猜测。`promptVersionId` 缺失或为 0 时只允许进入 `INITIALIZE`；
  其他意图立即说明尚未初始化并停止。`promptVersionId>0` 时，只要用户要求“初始化骨架”、
  “新建骨架”、“重新生成骨架”或“修改骨架”，一律进入
  [edit-skeleton-workflow.md](references/edit-skeleton-workflow.md)，使用该 ID 精确查询并锁定
  `writeMode=EDIT`；不得因“初始化/新建”字样停止或改走 `INITIALIZE`。确认后仍基于该版本
  创建一个新的骨架草稿，不覆盖原版本。
- 上一条只适用于普通骨架操作。用户明确要求分析或处理 Badcase 时，必须进入对应 Badcase
  workflow；只有分析确认属于 Prompt 缺陷后，才按该 workflow 的后续步骤提出修改建议，不能
  直接改走普通 `edit-skeleton`。
- `EDIT` 生成完整正文时，[skeleton-format.md](references/skeleton-format.md) 的结构与格式约束
  优先级高于基础 `prompt_content` 的既有写法。基础正文格式不合理时必须一并修复到符合规范；
  对不冲突的业务规则、映射和用户本轮明确修改予以保留，不得借格式修复擅自改变业务含义。
- 用户明确给出可直接落到提示词中的精确改动（例如“母子品牌新增：汾酒：竹叶青酒”）时，
  该文本就是本次修改目标：只查询基础提示词、做最小修改、校验并展示服务端 Diff。不得调用
  关系或规则 MCP 再证明用户输入，也不得因查询不到而拒绝、改写或反复权衡。
- `writeMode` 一旦锁定，校验结果、`diffRecordId` 和用户确认都不得改变它；两种模式确认后
  都只调用 `tool_edit_prompt_skeleton`：初始化传 `prompt_version_id=0`，修改传锁定的
  `selectedPromptVersionId`。
- 提案前执行 [shared-steps.md](references/shared-steps.md) 的 S3；确认后写入内容必须与校验内容
  完全一致。修改类只展示服务端 `data.diff_content`，用户要求时才展开全文。
- 线上版本禁止修改。修改只接受正文非空的草稿或归档版本；查到空正文时停止并报告数据异常，
  不切换到初始化，因为初始化只允许 `promptVersionId` 缺失或为 0。
- 初始化校验与确认写入都按 `ruleGroupId` 再次检查是否已有 Prompt；已有时直接把服务端返回的
  Prompt 结果告知用户，不生成提案、不写入。确实不存在时才创建首个草稿；初始化 Diff 的
  `base_prompt_version_id=0`，`new_prompt_version_id` 不填。
- 修改时 `tool_edit_prompt_skeleton` 仍按锁定的基础版本派生新草稿，返回的
  `new_prompt_version_id` 必须与基础 ID 不同。
- 两种结果都必须满足：返回成功、`new_prompt_version_id>0`、`version_no>0`、
  `new_prompt_name` 非空；后续修改和验证使用返回的 `new_prompt_version_id`。

写入前必须再次检查：`INITIALIZE` 只传 `prompt_version_id=0`，`EDIT` 只传锁定且大于 0 的
`selectedPromptVersionId`。禁止改用 `save_prompt_draft` 或从其他字段猜 ID。

## 路由

每次只读取目标 workflow，并同时读取 [shared-steps.md](references/shared-steps.md) 和
[base-version-policy.md](references/base-version-policy.md)：

- 初始化：必须同时完整读取
  [initialize-skeleton-workflow.md](references/initialize-skeleton-workflow.md) 和
  [skeleton-format.md](references/skeleton-format.md)；后者是生成前置必读，不是按需参考。
- 修改：必须同时完整读取 [edit-skeleton-workflow.md](references/edit-skeleton-workflow.md) 和
  [skeleton-format.md](references/skeleton-format.md)；格式规范在修改中同样是强制约束。
- 单条 Badcase：[badcase-single-workflow.md](references/badcase-single-workflow.md)
- 整次任务 Badcase：[badcase-task-workflow.md](references/badcase-task-workflow.md)
- 用户描述 Badcase：[badcase-description-workflow.md](references/badcase-description-workflow.md)
- 纯查询：不加载 workflow，只调用对应只读 MCP。

目标语义不清时先澄清；不得让用户输入的提示词 ID/名称覆盖业务上下文 `promptVersionId`。
S2 读取 [rule-loading-policy.md](references/rule-loading-policy.md)。生成全文时按 workflow 读取
[rule-transformation-guide.md](references/rule-transformation-guide.md)；初始化必须已按上述前置门槛完整读取
[skeleton-format.md](references/skeleton-format.md)。枚举存疑时读取 [enums.md](references/enums.md)；
[full-skeleton-example.md](references/full-skeleton-example.md) 仅在用户明确要求完整示例时读取。
