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

必须先依据目标 workflow、完整的 `skeleton-format.md` 以及已加载的可信规则生成候选完整提示词，
再把这份候选内容传给校验接口。禁止直接把基础版本原文当作候选内容提交校验，除非该原文已经
完整合规且本轮确实不需要任何修改。基础正文仅为 `111`、残缺片段或缺少固定章节时，必须先
重构为完整骨架；不得先校验原文，再以原文校验失败为由要求用户、系统侧或产品同学补骨架。

调用校验接口前必须执行以下生成门禁，任一项不满足都先修复候选全文：

1. 先从`skeleton-format.md`完整复制`# 输出格式`固定块，直到第8条`key_diff_point`规则结束；
   再生成前面的动态业务章节。固定块不得因全文长度、类目或比价项不同而裁剪。
2. 每个普通比价项默认只有`优先级`和`匹配`；仅部分类目生效时增加`适用类目`。不得把
   `抽取规则`、`归一化规则`、`例外与边界`、`具体规则名`机械套给每一项。仅当可信规则存在
   无法并入前两项的独立条件时才允许追加；仅当
   `radar_query_price_rule.data.labelCateRule`明确返回规则名称时才原样使用，没有名称的条件并入
   `匹配`。不得自行命名或使用模型自拟的通用栏目扩写。
3. JSON代码围栏后必须存在`字段规则：`及完整8条规则，并逐项检查：`category`裁剪、具体数字
   `confidence`、缺失值、三种`source`、布尔`match`、多来源`+`拼接、多部位材质格式、
   `key_diff_point`常驻。只出现标题或其中部分规则仍视为未完成，禁止调用
   `tool_validate_prompt_skeleton`。
4. 对本轮新增、删除或调序影响的业务编号序列执行整体重编号：比价项三级标题和母子品牌条目
   必须从 1 连续递增、无缺号和重复号；母子品牌标题组数必须等于最终实际条目数。编号未归一化
   时禁止调用 `tool_validate_prompt_skeleton`。

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

- `resp_code=1`、`data.valid=true` 且 `data.diff_record_id>0` → 展示提案。
- `data.valid=false` → 在本轮内部按 `data.errors` 自动修正后重新校验，最多重试 2 次。重试期间不得展示中间错误，不得询问用户是否同意补全、修复或继续。
- `data.valid=true` 但 `data.diff_record_id<=0`，或自动重试后仍未通过、调用失败 → 直接返回最终错误，不展示提案、不调用写入 MCP，也不得请求用户授权后再生成或修复。

`data.valid=false` 时返回的 `data.diff_record_id=0` 或空 `data.diff_content` 只是“尚未形成有效
提案”的伴随结果，不是停止自动修正候选正文的理由，也不得向用户表述为“服务端未提供修改方案”。
服务端只负责校验候选正文并计算 Diff；候选完整正文必须由本流程在校验前生成。

校验是提案的前置工具操作，不是待用户确认的计划。修改类请求中，本轮必须先完成调用并取得
`data.valid=true`、`data.diff_content` 和 `data.diff_record_id`，然后才能输出提案及确认话术。
初始化类请求同理：只有 `data.valid=true` 的完整候选正文才是可确认的初始化提案；结构缺失、
`data.valid=false`、错误摘要或修复计划都不是提案，禁止附加“同意后补全”“确认后生成”等话术。

重试时只修正 `data.errors` 指出的部分，未报错章节原样沿用上一次提交的内容，不重新生成、
不重排、不顺手改写。协议要求提交全文，但全文中只有报错处允许变化。

校验通过的这份内容原样保留至写入，确认后不得重新生成、重排或调整。

### 记录修改建议 ID 与 Diff

`data.valid=true` 时必须同时取得 `data.diff_record_id>0`，表示校验正文、基础版本和规则组已作为
待决策提案绑定并落库。缺失或为 0 时不构成可确认提案，禁止进入 `[S4]`、`[S5]`。

同时记住 `data.diff_content`：这是服务端比对基础版本算出的差异正文，`[S4]` 展示 Diff 时
只能原样引用它，不得自行重写、改写或裁剪。

本接口在传入 `conversation_id` 时会写入数据，不得为了“再确认一下”而重复调用；
只在首次校验和按 `data.errors` 修正后的重试中调用。

## S4 展示提案

### 所有提案的输出协议

目标 workflow 的提案模板不是示例，而是唯一允许的可见回复格式。必须以模板首个标题作为回复
第一行，逐项保持标题、字段和顺序；替换全部占位符并解析条件分支，禁止原样输出尖括号说明。
模板前后不得添加寒暄、执行摘要、结论或其他正文。发送前不满足时只在内部重组，不展示错误版本。

- 含 Diff 的提案：`### Diff` 或 `### 修改建议` 后必须使用 `diff` 围栏，并逐字符复制同次 S3
  的 `data.diff_content`，保留行首 `+`、`-`、空格、标题和顺序；禁止改用 `text` 围栏，或摘要、
  解释、改写、补删行、重新编号。空格开头是上下文行，不代表新增、删除或“仅保留”。
- 初始化提案：完整正文必须逐字符使用同次 S3 校验通过的候选 `prompt_content`，放入模板规定的
  单个四反引号 `text` 围栏；不得省略、拆分、重生成或使用基础原文替代。

按场景决定展示内容：

- **修改类流程**（含 Badcase 修复）：只展示 Diff，不展开提示词全文。Diff 正文原样取
  `[S3]` 返回的 `data.diff_content`，套上 `diff` 语言标记的代码围栏输出；内容为空或提示
  无差异时如实说明，不自造 Diff。同时展示同一次 S3 返回的
  非零 `data.diff_record_id`（用户可见名称统一写“修改建议 ID（diff_id）”）。禁止在 Diff 前后附加完整提示词或以
  「第 1/N 段」方式分段输出全文。
  提案中必须同时写明：`当前提示词由页面左侧选定的提示词决定（以当前业务上下文 promptVersionId 为准）`。
- **初始化首个 Prompt**：没有有效正文基线，展示完整提示词。
  全文必须作为一块内容放在同一个四反引号 `text` 围栏中，禁止拆块或把部分内容输出到围栏外；
  全文内部的三反引号代码围栏原样保留。同时展示同次 S3 返回的非零 `data.diff_record_id`，
  用户可见名称统一写“修改建议 ID（diff_id）”。

结构完整性由 S3 保证。
提案须标注「尚未保存」，并以确认话术结尾。

修改类流程中用户明确要求「展开完整提示词」时才完整输出且不得省略。

## S5 确认后写入草稿

确认阶段重新读取本轮最新业务变量 `promptVersionId`，不得沿用生成提案时的上一轮版本 ID。
`prompt_diff_record_id` 仍使用紧邻上一轮提案同次 `[S3]` 返回的非零 ID。

写入只允许调用 `tool_edit_prompt_skeleton`。`prompt_version_id` 原样取本轮最新
`promptVersionId`：数值为 0 时传 0，大于 0 时传该值；缺失或仍为模板占位符时停止。原提案的
`writeMode` 只决定 `source_type`，不得用它把本轮版本改回提案生成时的版本。

**初始化、修改和 Badcase 修复写入**：都只调用一次 `tool_edit_prompt_skeleton`，组合使用上一轮提案
保留的 Diff ID 与本轮最新版本 ID：
`tool_edit_prompt_skeleton(rule_group_id=当前业务上下文.ruleGroupId, prompt_version_id=当前业务上下文.promptVersionId, operator=当前业务上下文.operator, source_type=<INITIALIZE 传3；EDIT传2>, prompt_diff_record_id=<紧邻上一轮提案同次 S3 返回的 diff_record_id>)`。

不得在模型侧用上一轮版本替换本轮 `promptVersionId`，也不得因怀疑不匹配而重新校验或生成 Diff。
服务端会用 Diff 记录保存的 `base_prompt_version_id` 校验本轮 `prompt_version_id`；返回
“`prompt_version_id与修改建议的基础版本不一致`”时，据实告知当前页面版本已不再匹配该提案，
不切换版本、不改传正文、不重试。

不传 `prompt_content`：服务端一律以库中记录为准，传了也会被忽略。

初始化确认时服务端再次按规则组查重：已有就直接返回现有 Prompt，没有才创建首个草稿；初始化
Diff 的基础版本 ID 为 0、新版本 ID 为空。修改仍基于锁定版本派生新草稿。写入后把工具实际返回的 ID、
名称和版本关系告知用户；后续修改和验证使用该 ID。

初始化写入响应 `data.prompt_exists=true` 表示确认时已经存在 Prompt：明确告知用户本次未新建，
直接展示返回的 `new_prompt_version_id`、`new_prompt_name`、`version_no`。

返回“该修改建议已处理或已失效”时，说明该建议已被其他路径写入，**不得重试或改传正文**，
否则会重复创建草稿；改为向用户说明并请其刷新查看。

调用超时或返回不完整时，可调用 `query_agent_detail` 取得
`data.latest_draft_prompt_version_id` 做只读结果核实，再按该 ID
精确查询并比对基础版本、完整内容和创建信息；不能唯一确认就报告结果未知，不重试写入。
