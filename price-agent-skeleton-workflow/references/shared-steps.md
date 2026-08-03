# 共享步骤

各 workflow 以 `[S1]`~`[S5]` 引用本文件的步骤，不再重复正文。

## S1 查询 Agent

调用
`query_agent_detail(agent_id=当前业务上下文.agentId, operator=当前业务上下文.operator)`。
仅当 `base_resp.resp_code=1` 时读取 `data`。

## S2 加载规则与映射

按 [rule-loading-policy.md](rule-loading-policy.md) 取得本轮需要的 `price_rule_json`、
`special_rule_json` 和 `data_table_json`。加载范围由当前 workflow 指定：新建类取全量，
修改与 Badcase 类只取相关比价项与映射类型。

- 调用 `tool_query_price_rule` 时按该策略传 `compare_items`；无法可靠定位目标比价项时传空
  取全量，不得传入未经确认的名称。
- `tool_query_rule_data_table` 仅在改动涉及品牌或材质判定时调用，`table_types` 只传相关类型。
- 同一会话中作用域完全一致的规则与映射直接复用上文，不重复调用。

返回内容按该策略的「结果校验」处理；关键数据缺失时停止，不带着不完整规则继续生成。

## S3 生成并校验完整提示词

生成完整提示词后，展示提案前调用
`tool_validate_prompt_skeleton(prompt_content=<生成的完整提示词>, operator=当前业务上下文.operator)`。

- `resp_code=1` 且 `data.valid=true` → 展示提案。
- `data.valid=false` → 按 `data.errors` 修正后重新校验，最多重试 2 次。
- 仍未通过或调用失败 → 返回校验错误，不展示提案，不调用任何写入或发布 MCP。

重试时只修正 `data.errors` 指出的部分，未报错章节原样沿用上一次提交的内容，不重新生成、
不重排、不顺手改写。协议要求提交全文，但全文中只有报错处允许变化。

校验通过的这份内容原样保留至写入，确认后不得重新生成、重排或调整。

## S4 展示提案

按场景决定展示内容：

- **修改类流程**（含 Badcase 修复）：只展示 Diff，不展开提示词全文。
- **从零新建**：没有 Diff 基线，展示完整提示词。

两者都不输出字符数、章节数等无法精确统计的自报数据；结构完整性由 S3 保证。
提案须标注「尚未保存」，并以确认话术结尾。

修改类流程中用户明确要求「展开完整提示词」时才完整输出且不得省略。

## S5 确认后写入草稿

用户明确确认后调用**一次**
`tool_edit_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=<基础提示词ID；从零新建传0>, prompt_content=<S3 已校验的完整内容>, change_reason=<变更原因>, diff_content=<已展示Diff；从零新建传"">, source_type=2, operator=当前业务上下文.operator)`。

`prompt_content` 必须与 S3 校验时提交的内容完全一致。按 `SKILL.md` 的新草稿成功规则校验，
返回名称、ID 与版本关系。不自动运行验证或发布。

调用超时或返回不完整时，先调用 S1 取得 `data.latest_draft_prompt_version_id`，再按该 ID
精确查询并比对基础版本、完整内容和创建信息；不能唯一确认就报告结果未知，不重试写入。
