# 共享步骤

各 workflow 只引用 `[S1]`～`[S5]`，本文件是这些步骤的唯一执行定义。

## S1 补齐 ruleGroupId

- 已有 `ruleGroupId`：跳过，不调用 `query_agent_detail`。
- 缺失：调用 `query_agent_detail(agent_id=上下文.agentId, operator=上下文.operator)`；仅
  `base_resp.resp_code=1` 时读取 `data.card.rule_group_id`，仍为空则停止。

## S2 加载规则与映射

严格执行 [rule-loading-policy.md](rule-loading-policy.md)。初始化固定先调用
`tool_query_category_ids`，再用返回的每个 `categoryId` 调 `radar_query_price_rule`；随后加载
母子品牌、材质关系 Skill 并调用各自 MCP，主要按规则响应的 `cateName` 过滤。进入 S3 前必须有
类目 ID、逐类规则、品牌关系和材质关系的真实调用记录。修改和 Badcase 按目标范围加载。关键数据
缺失即停止；映射只能筛选工具返回项。用户精确增删改按目标 workflow 的直达例外执行。

## S3 生成并校验完整提示词

候选全文是内部工具输入，不等于允许向用户展示。依据目标 workflow、完整
[skeleton-format.md](skeleton-format.md) 和可信业务数据生成；基础正文仅在已完整合规且本轮无需
修改时才可原样使用。`111`、残缺正文或缺固定章节时必须先重构，不得先提交失败再要求外部补充。

调用校验前完成六项检查：

1. 从格式规范完整复制 `# 输出格式` 固定块至第 8 条 `key_diff_point` 规则，不因长度或类目裁剪。
2. 候选正文已完整遵守 [rule-transformation-guide.md](rule-transformation-guide.md)；适用类目表、
   Step0、比价项标题和 `extracted` key 与本轮 `ruleTableInfo[].compareItem` 数量、名称、顺序完全一致，
   且每行使用真实 `cateNameTree` 路径。任一不一致不得调用 S3。
3. JSON 围栏后保留 `字段规则：` 和完整 8 条规则：`category`、具体 `confidence`、缺失值、三种
   `source`、布尔 `match`、多来源拼接、多部位材质、`key_diff_point`。
4. 生成候选全文时主动修复连续编号，不得只检查或原样保留断号：对母子品牌映射表和比价项序列
   分别提取全部非空条目，删除旧数字前缀，保持当前条目顺序，从 1 开始整体重写为 `1..N`。
   删除原第 K 项时，原 K+1 自动变 K、原 K+2 自动变 K+1，直至末项。重写后再次扫描；若仍出现
   `1,3`、`2,4`、重复号、首项非 1 或末项不等于条目数，必须继续修复，修复完成后才能调用下方
   `tool_validate_prompt_skeleton`。只有无法可靠解析条目边界时才停止并报告格式错误。
5. 映射门禁已通过：`compareItem` 无“品牌”时不存在母子品牌映射表，无“材质”时不存在材质对照表；
   母子品牌标题不得包含“共 N 组”等动态组数。任一不符不得调用 S3。
6. 最终 `prompt_content` 整体约 1 万字（包含规则、映射和输出格式）。接近或超过时只删除重复解释、
   重复示例、同义表达，并按作用域过滤非必要映射；不得裁掉真实比价项、规则边界、例外、固定章节
   或 JSON 字段，也不得硬截断正文。
调用：

`tool_validate_prompt_skeleton(prompt_content=<候选全文>, operator=上下文.operator, conversation_id=上下文.localConversationId, base_prompt_version_id=<INITIALIZE传0；EDIT传selectedPromptVersionId>, rule_group_id=上下文.ruleGroupId, agent_id=上下文.agentId)`

`rule_group_id`、`agent_id` 至少一个大于 0；EDIT 的 `selectedPromptVersionId>0`。初始化先处理
`data.prompt_exists`：为 true 时返回 `existing_prompt_version_id/name` 后停止；为 false 才继续。

- `resp_code=1 && data.valid=true && data.diff_record_id>0`：成功，锁定本次请求传入的完整
  `prompt_content`，并保留同次响应的 `diff_record_id`、`diff_content`。后续展示继续引用这个
  已提交变量，禁止重新生成。
- `resp_code=1 && data.valid=true && data.diff_record_id<=0`：立即异常结束，只回复
  “提示词校验未生成有效修改建议（diff_record_id=0），无法形成提案。”禁止重试、提案、确认、
  写入或展开全文。
- `valid=false`：只按 `data.errors` 修正报错处，其他正文不变，最多重试 2 次。
- 其他情况或重试后仍失败：只返回最终错误，不展示提案、不写入、不请求用户授权后再修复。

Diff 只取服务端 `data.diff_content`，禁止自行书写。校验调用会落 Diff 记录，只能首次及按 errors
重试时调用，不得为“再确认”重复调用。校验通过的候选原样保留至写入，不重新生成或调整。

## S4 展示提案

严格使用目标 workflow 的唯一模板：标题为第一行，字段和顺序不变，替换全部占位符并解析条件
分支；模板外不加寒暄、摘要或结论。

- 修改/Badcase 修复：只展示同次 S3 的非零 `diff_record_id` 和逐字符复制的 `diff_content`，使用
  `diff` 围栏并保留 `+`、`-`、空格、标题和顺序；不得摘要、解释、补删或重编号。提案写明当前
  提示词由页面左侧选择的 `promptVersionId` 决定。用户明确要求展开全文时，逐字符输出同次 S3
  请求已经提交并锁定的 `prompt_content`，不得重新生成。
- 初始化：逐字符展示同次 S3 请求已经提交并锁定的 `prompt_content`，放入一个四反引号 `text`
  围栏；正文内三反引号原样保留，不得省略、拆分、重生成或写“其余组”。展示变量与 S3 请求变量
  必须是同一字符串对象；展示完成前不得覆盖、拼接或改写。

提案统一展示“修改建议 ID（diff_id）”、标注“尚未保存”并以模板确认话术结尾。

### 展开完整提示词

用户紧邻有效提案回复“展开完整提示词”“查看完整提示词”或等价表达时，只执行本分支：上一条必须
来自真实同次 S3，且 `valid=true`、`diff_record_id>0`、锁定的 `prompt_content` 非空；否则只返回
“上一条不是可展开的有效提示词提案，请重新发起提示词修改。”

满足条件时，不调用 MCP、不重新查询、不重新生成、不解释，只逐字符输出锁定的同一个
`prompt_content`。使用四反引号 `text` 围栏，确保正文内部三反引号保持原样：

`````markdown
````text
<逐字符复制同次 S3 已锁定的 prompt_content>
````
`````

展开只用于查看，不构成确认。由于确认只接受紧邻提案的用户消息，展开后原提案不能再直接确认；
用户随后要求写入时须重新生成提案。

## S5 确认后写入草稿

### 确认门禁

只允许确认可见的上一条助手提案；中间出现其他用户消息后永久失效，即使提供 `diffId` 也只能
重新生成。若上条含目标提案标题、非零 Diff ID 和确认话术，必须调用写入工具；模型不得推测
数据库序号。

### 参数与调用

- `prompt_diff_record_id`：紧邻提案同次 S3 的非零 ID。
- `prompt_version_id`：重新读取本轮上下文 `promptVersionId`，禁止使用提案旧版本或上轮返回的
  `new_prompt_version_id`；缺失/占位符停止。
- `source_type`：INITIALIZE=3，EDIT/Badcase=2。原提案模式只决定此字段。
- 不传 `prompt_content`；服务端以 Diff 记录正文为准。

`promptVersionId>0` 时先按 [base-version-policy.md](base-version-policy.md) 精确查询本轮基础版本并
记录 `version_name/id/version_no`；失败则不写入。为 0 时基础提示词记为“无（初始化）”。

只调用一次：

`tool_edit_prompt_skeleton(rule_group_id=上下文.ruleGroupId, prompt_version_id=上下文.promptVersionId, operator=上下文.operator, source_type=<3或2>, prompt_diff_record_id=<紧邻提案Diff ID>)`

服务端校验 Diff 基础版本关联关系。不匹配、已处理或失效时据实报告，不切换版本、不重新校验、
不传正文、不重试。超时或响应不完整时，可用 `query_agent_detail.latest_draft_prompt_version_id`
只读核实；不能唯一确认就报告结果未知，禁止重试写入。

### 成功回复

字段只取写入前基础版本查询和本次响应，模板外不加内容：

```markdown
## 提示词草稿创建成功
- 修改建议 ID（diff_id）：`<本次 prompt_diff_record_id>`
- 基础提示词：`<version_name>`（ID：`<prompt_version_id>`，版本号：`<version_no>`）
- 新提示词：`<data.new_prompt_name>`（ID：`<data.new_prompt_version_id>`，版本号：`<data.version_no>`）
- 状态：草稿已创建
```

初始化的基础提示词写 `无（prompt_version_id=0，初始化）`；`data.prompt_exists=true` 时新提示词
字段仍取响应，状态改为“已存在，未新建”。不得总结变更内容；返回的新 ID 仅展示，不更新上下文。
