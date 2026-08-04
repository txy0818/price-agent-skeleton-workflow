# 共享步骤

各 workflow 以 `[S1]`~`[S5]` 引用本文件的步骤，不再重复正文。

## S1 查询 Agent

调用
`query_agent_detail(agent_id=当前业务上下文.agentId, operator=当前业务上下文.operator)`。
仅当 `base_resp.resp_code=1` 时读取 `data`。

## S2 加载规则与映射

按 [rule-loading-policy.md](rule-loading-policy.md) 取得本轮需要的 `price_rule_json`、
`special_rule_json` 和 `data_table_json`。比价规则只按类目收窄，类目内一律取全量；
映射表按需取。

- 调用 `tool_query_price_rule` 时 `include_special_rule` 一律传 `1`，`category_ids`
  **有就传、没有传空**：能确定当前 workflow 关联的类目 ID 时传入，否则传空表示取规则组全部
  类目的全量规则；返回该类目下全部比价项规则，不按比价项名称过滤。
- `tool_query_rule_data_table` 仅在改动涉及品牌或材质判定时调用，`table_types` 只传相关类型。
- 同一会话中作用域完全一致的规则与映射直接复用上文，不重复调用。

返回内容按该策略的「结果校验」处理；关键数据缺失时停止，不带着不完整规则继续生成。

## S3 生成并校验完整提示词

生成完整提示词后，展示提案前调用
`tool_validate_prompt_skeleton(prompt_content=<生成的完整提示词>, operator=当前业务上下文.operator, conversation_id=当前业务上下文.localConversationId, base_prompt_version_id=<基础提示词ID；从零新建传0>)`。

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
- **从零新建**：没有 Diff 基线（`data.diff_content` 为空），展示完整提示词。

结构完整性由 S3 保证。
提案须标注「尚未保存」，并以确认话术结尾。

修改类流程中用户明确要求「展开完整提示词」时才完整输出且不得省略。

## S5 确认后写入草稿

用户明确确认后调用**一次** `tool_edit_prompt_skeleton`。入参按 `[S3]` 返回的
`data.diff_record_id` 二选一，不要把两条路径的字段混传：

**路径一（`diff_record_id > 0`，优先）** —— 内容已在服务端，只传 ID：
`tool_edit_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=<基础提示词ID；从零新建传0>, operator=当前业务上下文.operator, source_type=2, prompt_diff_record_id=<S3 返回的 diff_record_id>)`。

不传 `prompt_content`：服务端一律以库中记录为准，传了也会被忽略。

**路径二（`diff_record_id = 0`，兜底）** —— 逐项传入内容：
`tool_edit_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=<基础提示词ID；从零新建传0>, prompt_content=<S3 已校验的完整内容>, operator=当前业务上下文.operator, source_type=2, conversation_id=当前业务上下文.localConversationId)`。

`prompt_content` 必须与 S3 校验时提交的内容完全一致。`conversation_id` 用于补齐这条草稿的
来源留档（diff 由服务端自行计算，无需传入），缺失不影响草稿创建。

两条路径都按 [SKILL.md](../SKILL.md) 的新草稿成功规则校验，返回名称、ID 与版本关系。
不自动运行验证或发布。

返回“该修改建议已处理或已失效”时，说明该建议已被其他路径写入，**不得重试、不得改走兜底
路径**，否则会重复创建草稿；改为向用户说明并请其刷新查看。

调用超时或返回不完整时，先调用 S1 取得 `data.latest_draft_prompt_version_id`，再按该 ID
精确查询并比对基础版本、完整内容和创建信息；不能唯一确认就报告结果未知，不重试写入。
