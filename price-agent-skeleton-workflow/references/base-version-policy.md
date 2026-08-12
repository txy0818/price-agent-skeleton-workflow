# 基础版本选择策略

## 查询对象门禁

仅“当前提示词/当前选中的提示词”允许执行下述精确查询。用户要求最新、线上最新、线上、归档、
草稿最新/草稿，或指定/切换提示词 ID、名称、版本时（即使同时说“当前”），不得调用任何提示词
查询 MCP，仅回复：“当前操作仅使用本轮业务上下文中的提示词版本，不支持通过用户文字指定或切换版本。”

进入本策略时只读取本轮业务上下文中已展开的 `promptVersionId`，绑定为
`currentPromptVersionId`，同时丢弃上一轮已解析值。`new_prompt_version_id`、
`existing_prompt_version_id` 和其他历史 Prompt ID 仅供展示，不能参与赋值、比较或纠正。每次 MCP
调用都直接从 `currentPromptVersionId` 新建参数，不复用上一轮参数。字段缺失、为空或为模板占位符
时规范化为 0 并进入 INITIALIZE；非空但不是非负整数时停止。

## 路由与校验

- `promptVersionId>0`：精确查询该 ID 并校验规则组归属。普通提示词操作进入修改流程；单条 Badcase 按其独立 workflow 查询验证结果。
- `promptVersionId` 缺失、为 0 或仍为模板占位符：进入初始化流程，不存在基础版本。

精确查询：

`tool_query_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=currentPromptVersionId, version_name="", query_online=false, query_latest=false, operator=当前业务上下文.operator)`

仅当 `resp_code=1` 且 `data.prompt_version` 非空时检查版本状态和正文；正文的 `null`、空串或纯空白均视为空：

- `version_status` 为 1、2 或 3 且正文非空：允许查看或作为只读基础派生新草稿；禁止覆盖原版本。
- `version_status` 为 1、2 或 3 且正文为空：报告数据异常并停止，不得改走初始化。
- 其他 `version_status`：停止并报告未知状态。
- 普通修改没有具体方向：使用修改流程的默认整理目标，不得追问 ID、名称或方向。

不得依据 `version_no`、`version_status` 或版本名称推断正文是否已初始化。
不得根据查询响应或历史提案中的新版本 ID 声称页面当前选中了其他版本，也不得要求用户切换到该 ID。
