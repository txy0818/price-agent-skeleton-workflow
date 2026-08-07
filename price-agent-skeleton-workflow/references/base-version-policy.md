# 基础版本选择策略

普通修改只能使用当前业务上下文提供的 `promptVersionId` 精确查询非空基础版本并派生草稿。
初始化仅在 `promptVersionId` 缺失或为 0 时进入，不存在基础版本。

## 取得与路由

命中即停止，不继续向下推断：

1. `当前业务上下文.promptVersionId>0` → 以该 ID 精确查询，并校验规则组归属。
2. `promptVersionId` 缺失或为 0 → 只允许初始化；修改、Badcase 或验证意图直接说明当前
   Agent 尚无提示词并停止。

修改或 Badcase 精确查询成功后：

- `version_status=2`（线上）→ 停止并提示用户改选草稿或归档版本；
- `version_status=1/3` 且 `prompt_content` 为空 → 停止并报告版本正文为空的数据异常；不得进入初始化；
- `version_status=1/3` 且 `prompt_content` 非空 → 已初始化；普通修改进入 `edit-skeleton`，
  Badcase 进入对应 workflow。用户只要求初始化时，直接说明已初始化，不自动修改；
- Badcase 的 `prompt_content` 为空 → 停止并要求先初始化，不得生成分析结论或修改提案。

`prompt_content` 的空判断以 `null`、空串或仅空白字符为准。不得仅凭 `version_no`、
`version_status` 或版本名称推断是否已经初始化。

禁止从历史消息、`当前提示词`、`刚才那个`等表述推断，禁止选择线上版本。

不向用户索取提示词 ID 或名称。上下文缺失 `promptVersionId` 时，只有初始化流程可继续。

## 查询

`tool_query_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=当前业务上下文.promptVersionId, version_name="", query_online=false, operator=当前业务上下文.operator)`

仅当 `resp_code=1` 且 `data.prompt_version` 非空、`prompt_content` 非空时视为可修改基础版本；
空正文按本文规则停止，不进入初始化。
