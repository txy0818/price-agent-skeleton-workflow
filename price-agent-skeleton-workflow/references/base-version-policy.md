# 基础版本选择策略

普通修改只能使用当前业务上下文提供的 `promptVersionId` 精确查询非空基础版本并派生草稿。
初始化仅在 `promptVersionId` 缺失或为 0 时进入，不存在基础版本。

## 取得与路由

这里只读取当前业务上下文中的 `promptVersionId`，不要求用户重复提供。

命中即停止，不继续向下推断：

1. `当前业务上下文.promptVersionId>0` → 以该 ID 精确查询，并校验规则组归属。
2. `promptVersionId` 缺失或为 0 → 统一进入 `initialize-skeleton`；不再根据用户说修改、优化、
   新增内容、Badcase 或其他骨架操作而停止或改走其他 workflow。

修改或 Badcase 精确查询成功后，查询目标始终是左侧选定项对应的业务上下文
`promptVersionId`；用户消息中的 ID 或名称不得覆盖：

- `version_status=2`（线上）且 `prompt_content` 非空 → 允许查看，也允许作为只读基础进入
  `edit-skeleton` 并派生新草稿；禁止覆盖线上版本，不得要求用户另选或提供草稿 ID；
- `version_status=2`（线上）且 `prompt_content` 为空 → 停止并报告正文为空的数据异常；
- `version_status=1/3` 且 `prompt_content` 为空 → 停止并报告版本正文为空的数据异常；不得进入初始化；
- `version_status=1/3` 且 `prompt_content` 非空 → 骨架已初始化：
  - 普通的新建、初始化、优化、重新生成或修改骨架诉求，统一进入 `edit-skeleton`，不得因用户使用
    “初始化/新建”等措辞而停止；
  - 只说“优化/修改/调整/整理/重新生成/初始化/新建提示词（或骨架）”等普通骨架操作且未给
    具体方向时，使用 `edit-skeleton` 的默认整理目标继续，不得追问提示词 ID、名称或具体方向；
  - 用户明确要求分析或处理 Badcase 时，进入对应的 Badcase workflow，不进入 `edit-skeleton`；
  - Badcase workflow 分析后若确认属于 Prompt 缺陷，再按照该 workflow 的后续步骤提出骨架
    修改建议。
- Badcase 的 `prompt_content` 为空 → 停止并要求先初始化，不得生成分析结论或修改提案。

`prompt_content` 的空判断以 `null`、空串或仅空白字符为准。不得仅凭 `version_no`、
`version_status` 或版本名称推断是否已经初始化。

禁止从历史消息、`当前提示词`、`刚才那个`等表述推断或切换版本。左侧已选中的线上版本可以
作为查询目标和派生新草稿的基础版本。

不向用户索取提示词 ID 或名称。上下文缺失 `promptVersionId` 时，只有初始化流程可继续。

## 查询

`tool_query_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=当前业务上下文.promptVersionId, version_name="", query_online=false, operator=当前业务上下文.operator)`

仅当 `resp_code=1` 且 `data.prompt_version` 非空、`prompt_content` 非空时视为可修改基础版本；
空正文按本文规则停止，不进入初始化。
