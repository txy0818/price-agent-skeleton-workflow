# 基础版本选择策略

任何修改都只能派生新的提示词草稿，不得覆盖基础版本。草稿、线上和归档版本均可作为基础
版本；归档只限制**直接发布**，不限制作为派生基础。

## 取得顺序

命中即停止，不继续向下推断：

1. 本轮输入显式提供提示词 ID 或名称 → 精确查询该版本。
2. 本轮未提供，但当前流程有任务上下文自带的提示词（如
   `tool_query_validation_result` 返回的 `PromptVersionBaseInfo`）→ 直接使用，不再查询其他版本。
3. 都没有 → 按下方处理，不得自行选择。

禁止从历史消息、`当前提示词`、`刚才那个`等表述推断，禁止把线上版本当默认值。

## 都没有时

**修改类**（`edit-skeleton`、`badcase-description`）先询问：

```text
本轮未指定提示词名称或 ID。请提供需要作为基础版本的提示词 ID 或名称；
若希望以当前线上提示词为基础，请回复「使用线上提示词」。
```

用户给 ID 或名称 → 精确查询；明确回复「使用线上提示词」→ 才可传 `query_online=true`；
未明确选择 → 停止，不生成、不写入。

**新建类**（`new-skeleton`）：`current_prompt_version_id` 与
`latest_draft_prompt_version_id` 都为 `0` → 从零生成；任一非零 → 展示已有版本，
询问「基于指定版本修改」还是「仍从零创建」，确认前不生成、不写入。

## 查询

`tool_query_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=<本轮ID；无则0>, version_name=<本轮名称；无则空>, query_online=<仅用户明确同意时true>, operator=当前业务上下文.operator)`

仅当 `resp_code=1` 且 `data.prompt_version`、`prompt_content` 均非空时视为成功。
ID 与名称同时提供时按 ID 查询，并校验返回 `version_name` 一致；不一致时说明冲突并要求修正。
