# 共享步骤

各 workflow 以 `[S1]`~`[S5]` 引用本文件的步骤，不再重复正文。

## S1 必要时补齐 ruleGroupId

- 当前业务上下文已有 `ruleGroupId`：跳过 S1，不调用 `query_agent_detail`。
- `ruleGroupId` 缺失：调用
  `query_agent_detail(agent_id=当前业务上下文.agentId, operator=当前业务上下文.operator)`，
  仅在 `base_resp.resp_code=1` 时读取 `data.card.rule_group_id`；仍为空则停止。

## S2 加载规则与映射

严格按 [rule-loading-policy.md](rule-loading-policy.md) 查询、校验和过滤。初始化先取规则组类目，
再逐类加载规则与映射；修改和 Badcase 按目标范围加载。相同工具、入参与作用域可复用本轮
结果；关键数据缺失时停止。

## S3 生成并校验完整提示词

生成完整提示词后，展示提案前调用
`tool_validate_prompt_skeleton(prompt_content=<生成的完整提示词>, operator=当前业务上下文.operator,
conversation_id=当前业务上下文.localConversationId,
base_prompt_version_id=selectedPromptVersionId,
rule_group_id=当前业务上下文.ruleGroupId, agent_id=当前业务上下文.agentId)`。

调用前要求 `selectedPromptVersionId>0`，且 `rule_group_id`、`agent_id` 至少一个大于 0；缺失时
停止并补查上下文，不得传 0 版本 ID。

Diff 由服务端依据基础版本计算并通过 `data.diff_content` 返回，**不自行书写 Diff**。

- `resp_code=1` 且 `data.valid=true` → 展示提案。
- `data.valid=false` → 按 `data.errors` 修正后重新校验，最多重试 2 次。
- 仍未通过或调用失败 → 返回校验错误，不展示提案，不调用任何写入或发布 MCP。

重试时只修正 `data.errors` 指出的部分，未报错章节原样沿用上一次提交的内容，不重新生成、
不重排、不顺手改写。协议要求提交全文，但全文中只有报错处允许变化。

校验通过的这份内容原样保留至写入，确认后不得重新生成、重排或调整。

### 记录修改建议 ID 与 Diff

`data.valid=true` 时服务端会把本次完整内容落成一条待决策记录，并返回 `data.diff_record_id`。
必须记住这个 ID，`[S5]` 依据它判定走哪条写入路径：

- `data.diff_record_id > 0` → 内容已在服务端落库，写入时只传 ID。
- `data.diff_record_id = 0` → 未落库（未取到会话 ID 或服务端落库失败），写入时走兜底路径。
  此时 `valid=true` 仍然有效，照常展示提案，不需重新校验。

同时记住 `data.diff_content`：这是服务端比对基础版本算出的差异正文，`[S4]` 展示 Diff 时
只能原样引用它，不得自行重写、改写或裁剪。

本接口在传入 `conversation_id` 时会写入数据，不得为了“再确认一下”而重复调用；
只在首次校验和按 `data.errors` 修正后的重试中调用。

## S4 展示提案

按场景决定展示内容：

- **修改类流程**（含 Badcase 修复）：只展示 Diff，不展开提示词全文。Diff 正文原样取
  `[S3]` 返回的 `data.diff_content`，套上 `diff` 语言标记的代码围栏输出；内容为空或提示
  无差异时如实说明，不自造 Diff。
- **初始化空草稿或空归档版本**：没有有效正文基线（`data.diff_content` 为空），展示完整提示词。

结构完整性由 S3 保证。
提案须标注「尚未保存」，并以确认话术结尾。

修改类流程中用户明确要求「展开完整提示词」时才完整输出且不得省略。

## S5 确认后写入草稿

写入方式必须在查询基础版本后已经锁定，确认阶段不得重新选择。调用前执行硬校验：

| `writeMode` | 唯一允许的 MCP | 必传版本字段 |
|---|---|---|
| `INITIALIZE` | `save_prompt_draft` | `base_prompt_version_id=selectedPromptVersionId>0` |
| `EDIT` | `tool_edit_prompt_skeleton` | `prompt_version_id=selectedPromptVersionId>0` |

字段缺失、为 0 或模式与 MCP 不匹配时，停止并重新执行精确查询；禁止调用写入接口。S3 返回的
`diff_record_id` 仅供 `EDIT` 使用，**不得让 `INITIALIZE` 改走 `tool_edit_prompt_skeleton`**。

**初始化写入**：调用一次 `save_prompt_draft(rule_group_id=当前业务上下文.ruleGroupId,
base_prompt_version_id=selectedPromptVersionId, prompt_content=<S3 已校验的完整内容>,
operator=当前业务上下文.operator)`。只允许原地填写该空版本；返回的 `prompt_version_id`
必须与 `selectedPromptVersionId` 一致、`version_no>0`、`version_name` 非空。即使 S3 返回
`diff_record_id>0`，也忽略该 ID，不得调用 `tool_edit_prompt_skeleton`。

**修改写入**：调用一次 `tool_edit_prompt_skeleton`。入参按 `[S3]` 返回的
`data.diff_record_id` 二选一，不要把两条路径的字段混传：

**路径一（`diff_record_id > 0`，优先）** —— 内容已在服务端，只传 ID：
`tool_edit_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=selectedPromptVersionId, operator=当前业务上下文.operator, source_type=2, prompt_diff_record_id=<S3 返回的 diff_record_id>)`。

不传 `prompt_content`：服务端一律以库中记录为准，传了也会被忽略。

**路径二（`diff_record_id = 0`，兜底）** —— 逐项传入内容：
`tool_edit_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=selectedPromptVersionId, prompt_content=<S3 已校验的完整内容>, operator=当前业务上下文.operator, source_type=2, conversation_id=当前业务上下文.localConversationId)`。

`prompt_content` 必须与 S3 校验时提交的内容完全一致。`conversation_id` 用于补齐这条草稿的
来源留档（diff 由服务端自行计算，无需传入），缺失不影响草稿创建。

两条路径都按 [SKILL.md](../SKILL.md) 的新草稿成功规则校验，返回名称、ID 与版本关系。
不自动运行验证或发布。

返回“该修改建议已处理或已失效”时，说明该建议已被其他路径写入，**不得重试、不得改走兜底
路径**，否则会重复创建草稿；改为向用户说明并请其刷新查看。

调用超时或返回不完整时，可调用 `query_agent_detail` 取得
`data.latest_draft_prompt_version_id` 做只读结果核实，再按该 ID
精确查询并比对基础版本、完整内容和创建信息；不能唯一确认就报告结果未知，不重试写入。
