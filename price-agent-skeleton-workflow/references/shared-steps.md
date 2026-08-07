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

这里的「生成完整提示词」是工具调用和后续写入所需的内部步骤，不代表允许在提案中输出全文。
修改类流程必须继续执行 S4 的展示边界；S3 产生的全文不得直接进入用户可见回复。

生成完整提示词后，展示提案前调用
`tool_validate_prompt_skeleton(prompt_content=<生成的完整提示词>, operator=当前业务上下文.operator,
conversation_id=当前业务上下文.localConversationId,
base_prompt_version_id=<INITIALIZE传0；EDIT传selectedPromptVersionId>,
rule_group_id=当前业务上下文.ruleGroupId, agent_id=当前业务上下文.agentId)`。

`rule_group_id`、`agent_id` 至少一个大于 0。`INITIALIZE` 必须传 0；`EDIT` 必须要求
`selectedPromptVersionId>0`。

初始化时先看 `data.prompt_exists`：为 true 时直接返回 `data.existing_prompt_version_id` 和
`data.existing_prompt_name`，不得继续生成提案或调用写入；为 false 时才按下述 `valid`、Diff
规则继续。修改时保持原有校验行为。

Diff 由服务端依据基础版本计算并通过 `data.diff_content` 返回，**不自行书写 Diff**。

- `resp_code=1` 且 `data.valid=true` → 展示提案。
- `data.valid=false` → 按 `data.errors` 修正后重新校验，最多重试 2 次。
- 仍未通过或调用失败 → 返回校验错误，不展示提案，不调用写入 MCP。

校验是提案的前置工具操作，不是待用户确认的计划。修改类请求中，本轮必须先完成调用并取得
`data.valid=true`、`data.diff_content` 和 `data.diff_record_id`，然后才能输出提案及确认话术。

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
  无差异时如实说明，不自造 Diff。同时展示同一次 S3 返回的
  `data.diff_record_id`（用户可见名称可写“修改建议 ID”）。禁止在 Diff 前后附加完整提示词或以
  「第 1/N 段」方式分段输出全文。
- **初始化首个 Prompt**：没有有效正文基线，展示完整提示词。
  全文必须作为一块内容放在同一个四反引号 `text` 围栏中，禁止拆块或把部分内容输出到围栏外；
  全文内部的三反引号代码围栏原样保留。

结构完整性由 S3 保证。
提案须标注「尚未保存」，并以确认话术结尾。

修改类流程中用户明确要求「展开完整提示词」时才完整输出且不得省略。

## S5 确认后写入草稿

写入方式必须在查询基础版本后已经锁定，确认阶段不得重新选择。调用前执行硬校验：

| `writeMode` | 唯一允许的 MCP | 必传版本字段 |
|---|---|---|
| `INITIALIZE` | `tool_edit_prompt_skeleton` | `prompt_version_id=0` |
| `EDIT` | `tool_edit_prompt_skeleton` | `prompt_version_id=selectedPromptVersionId>0` |

字段缺失、为 0 或调用了其他写入 MCP 时，停止并重新执行精确查询；禁止调用写入接口。

**初始化和修改写入**：都只调用一次 `tool_edit_prompt_skeleton`。入参按 `[S3]` 返回的
`data.diff_record_id` 二选一，不要把两条路径的字段混传：

**路径一（`diff_record_id > 0`，优先）** —— 内容已在服务端，只传 ID：
`tool_edit_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=<INITIALIZE传0；EDIT传selectedPromptVersionId>, operator=当前业务上下文.operator, source_type=<INITIALIZE 传3；EDIT传2>, prompt_diff_record_id=<S3 返回的 diff_record_id>)`。

不传 `prompt_content`：服务端一律以库中记录为准，传了也会被忽略。

**路径二（`diff_record_id = 0`，兜底）** —— 逐项传入内容：
`tool_edit_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=<INITIALIZE传0；EDIT传selectedPromptVersionId>, prompt_content=<S3 已校验的完整内容>, operator=当前业务上下文.operator, source_type=<INITIALIZE 传3；EDIT传2>, conversation_id=当前业务上下文.localConversationId)`。

`prompt_content` 必须与 S3 校验时提交的内容完全一致。`conversation_id` 用于补齐这条草稿的
来源留档（diff 由服务端自行计算，无需传入），缺失不影响草稿创建。

初始化确认时服务端再次按规则组查重：已有就直接返回现有 Prompt，没有才创建首个草稿；初始化
Diff 的基础版本 ID 为 0、新版本 ID 为空。修改仍基于锁定版本派生新草稿。两条路径都把工具实际返回的 ID、
名称和版本关系告知用户；后续修改和验证使用该 ID。

初始化写入响应 `data.prompt_exists=true` 表示确认时已经存在 Prompt：明确告知用户本次未新建，
直接展示返回的 `new_prompt_version_id`、`new_prompt_name`、`version_no`。

返回“该修改建议已处理或已失效”时，说明该建议已被其他路径写入，**不得重试、不得改走兜底
路径**，否则会重复创建草稿；改为向用户说明并请其刷新查看。

调用超时或返回不完整时，可调用 `query_agent_detail` 取得
`data.latest_draft_prompt_version_id` 做只读结果核实，再按该 ID
精确查询并比对基础版本、完整内容和创建信息；不能唯一确认就报告结果未知，不重试写入。
