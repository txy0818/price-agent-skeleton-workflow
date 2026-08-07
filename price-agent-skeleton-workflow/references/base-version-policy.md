# 基础版本选择策略

只使用当前业务上下文的 `promptVersionId`，不得要求用户重复提供或从用户消息、历史记录、查询结果推断或切换版本。

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
