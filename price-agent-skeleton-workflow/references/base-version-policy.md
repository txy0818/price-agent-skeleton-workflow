# 基础版本选择策略

只使用本轮最新业务变量中的 `promptVersionId`。本轮值具有最高优先级，覆盖历史消息、旧提案、
上一轮工具返回和模型记忆中的任何 Prompt ID；每轮进入本策略时必须重新读取，禁止沿用上一轮值。
不得要求用户重复提供或从用户消息、历史记录、版本名称及查询结果推断或切换版本。

本策略只约束 Prompt 版本 ID，不覆盖提案的 `diff_record_id`。用户确认上一轮提案时，
`diff_record_id` 仍取该提案同次 S3 返回值，并按 `[S5]` 使用。

提案生成时记录该轮归一化后的 `proposalPromptVersionId`：缺失、0 或模板占位符统一为 0，
大于 0 时记录精确 ID。确认提案时，本轮最新 `promptVersionId` 必须与其完全一致；否则旧提案
失效，禁止继续原 `EDIT/INITIALIZE` 或使用旧提案内容。普通新请求只按本轮值路由。

## 路由与校验

- `promptVersionId>0`：精确查询该 ID 并校验规则组归属。普通提示词操作进入修改流程；Badcase 请求进入对应 Badcase 流程。
- `promptVersionId` 缺失、为 0 或仍为模板占位符：进入初始化流程，不存在基础版本。

精确查询：

`tool_query_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=当前业务上下文.promptVersionId, version_name="", query_online=false, operator=当前业务上下文.operator)`

仅当 `resp_code=1` 且 `data.prompt_version` 非空时检查版本状态和正文；正文的 `null`、空串或纯空白均视为空：

- `version_status` 为 1、2 或 3 且正文非空：允许查看或作为只读基础派生新草稿；禁止覆盖原版本。
- `version_status` 为 1、2 或 3 且正文为空：报告数据异常并停止，不得改走初始化；Badcase 不得继续归因或生成提案。
- 其他 `version_status`：停止并报告未知状态。
- 普通修改没有具体方向：使用修改流程的默认整理目标，不得追问 ID、名称或方向。
- Badcase：仅确认属于提示词缺陷后才提出修改。

不得依据 `version_no`、`version_status` 或版本名称推断正文是否已初始化。
