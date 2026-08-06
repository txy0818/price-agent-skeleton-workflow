# 基础版本选择策略

任何修改都只能派生新的提示词草稿，不得覆盖基础版本。只有草稿和归档版本可用于初始化或
修改；线上版本禁止初始化、修改和 Badcase 修改。

## 取得与路由

命中即停止，不继续向下推断：

1. 本轮输入显式提供提示词 ID 或名称 → 精确查询该版本，并校验 ID、名称及规则组归属。
2. 本轮未提供，但当前流程有任务上下文自带的提示词（如
   `tool_query_validation_result` 返回的 `PromptVersionBaseInfo`）→ 直接使用，不再查询其他版本。
3. 都没有 → 按下方处理，不得自行选择。

用户直接要求初始化、修改或描述 Badcase 时，本轮必须提供提示词 ID 或名称。精确查询成功后：

- `version_status=2`（线上）→ 停止并提示用户改选草稿或归档版本；
- `version_status=1/3` 且 `prompt_content` 为空 → 进入 `initialize-skeleton`；
- `version_status=1/3` 且 `prompt_content` 非空 → 已初始化；普通修改进入 `edit-skeleton`，
  Badcase 进入对应 workflow。用户只要求初始化时，直接说明已初始化，不自动修改；
- Badcase 的 `prompt_content` 为空 → 停止并要求先初始化，不得生成分析结论或修改提案。

`prompt_content` 的空判断以 `null`、空串或仅空白字符为准。不得仅凭 `version_no`、
`version_status` 或版本名称推断是否已经初始化。

禁止从历史消息、`当前提示词`、`刚才那个`等表述推断，禁止选择线上版本。

## 都没有时

**修改类**（`edit-skeleton`、`badcase-description`）先询问：

```text
本轮未指定提示词名称或 ID。请提供需要作为基础版本的草稿或归档提示词 ID 或名称。
```

用户给 ID 或名称后精确查询；未明确选择则停止，不生成、不写入。

**初始化类**（`initialize-skeleton`）要求用户提供待初始化提示词 ID 或名称。精确查询结果必须
为草稿或归档且 `prompt_content` 为空。非空时说明该版本已经初始化；线上时停止。

## 查询

`tool_query_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=<本轮ID；无则0>, version_name=<本轮名称；无则空>, query_online=false, operator=当前业务上下文.operator)`

仅当 `resp_code=1` 且 `data.prompt_version` 非空时视为查询成功；`prompt_content` 允许为空，
为空时按本文规则进入初始化流程。需要修改、Badcase 分析或发布时再要求内容非空。
ID 与名称同时提供时按 ID 查询，并校验返回 `version_name` 一致；不一致时说明冲突并要求修正。
