# 共享步骤

各 workflow 只引用 `[S1]`～`[S5]`，本文件是这些步骤的唯一执行定义。

## S1 补齐 ruleGroupId

- 已有 `ruleGroupId`：跳过，不调用 `query_agent_detail`。
- 缺失且 `agentId<=0`、缺失或为占位符：立即停止并返回业务上下文参数错误，不调用工具。
- 缺失且 `agentId>0`：调用 `query_agent_detail(agent_id=上下文.agentId, operator=上下文.operator)`；仅
  `base_resp.resp_code=1` 时读取 `data.card.rule_group_id`，仍为空则停止。

## S2 加载规则与映射

严格执行 [rule-loading-policy.md](rule-loading-policy.md)。初始化固定先调用
`tool_query_category_ids`，再用返回的每个 `categoryId` 调 `radar_query_price_rule`；修改
按目标范围加载类目规则。汇总全部有效 `ruleTableInfo[].compareItem` 后执行映射硬门禁：精确包含
“品牌”时必须调用品牌关系 MCP 并按规则响应的 `cateName` 过滤，不包含“品牌”时禁止调用；精确
包含“材质”时必须加载材质关系 Skill 并调用其 MCP，不包含“材质”时禁止调用。进入 S3 前必须有
类目 ID、逐类规则，以及规则实际要求的品牌/材质关系调用记录；未要求的映射不得调用。关键数据
缺失即停止；映射只能筛选工具返回项。用户精确增删改按目标 workflow 的直达例外执行。

## S3 生成并校验完整提示词

候选全文是内部工具输入，不等于允许向用户展示。依据目标 workflow、完整
[skeleton-format.md](skeleton-format.md) 和可信业务数据生成；格式规范、转换指南、workflow 步骤和
完成检查都是生成器内部约束，禁止复制、改写、概括或解释后写入 `prompt_content`，正文只能包含
“完整格式模板”规定的实际提示词内容。基础正文仅在已完整合规且本轮无需修改时才可原样使用。
`111`、残缺正文或缺固定章节时必须先重构，不得先提交失败再要求外部补充。

调用校验前完成以下全部检查：

1. 从格式规范完整复制 `# 输出格式` 固定块至第 8 条 `key_diff_point` 规则，不因长度或类目裁剪。
2. INITIALIZE、格式重构及 EDIT 当前规则全量同步分支必须完整执行
   [rule-transformation-guide.md](rule-transformation-guide.md)，并读取 [enums.md](enums.md)。生成内部逐项
   转换账本，每行记录真实 `compareItem`、原始 `infoSource`、`expectedPriority`、候选优先级、清洗后的
   特殊规则、原始 `compareLogic`、别名归一后的标准 `compareLogic`、`expectedMatch` 和候选匹配条件。
   命中受控别名时，候选必须是标准枚举的完整展开，不得残留别名原文。账本或指南任一硬校验不通过时禁止
   S3；本文件不另行复述比价项转换算法。
3. JSON 围栏后保留 `字段规则：` 和完整 8 条规则：`category`、具体 `confidence`、缺失值、三种
   `source`、布尔 `match`、多来源拼接、多部位材质、`key_diff_point`。
4. 生成候选全文时主动修复连续编号，不得只检查或原样保留断号：对母子品牌映射表和比价项序列
   分别提取全部非空条目，删除旧数字前缀，保持当前条目顺序，从 1 开始整体重写为 `1..N`。
   删除原第 K 项时，原 K+1 自动变 K、原 K+2 自动变 K+1，直至末项。重写后再次扫描；若仍出现
   `1,3`、`2,4`、重复号、首项非 1 或末项不等于条目数，必须继续修复，修复完成后才能调用下方
   `tool_validate_prompt_skeleton`。只有无法可靠解析条目边界时才停止并报告格式错误。
5. INITIALIZE、格式重构及 EDIT 当前规则全量同步分支必须通过映射门禁：`compareItem` 有“品牌”时
   必须有本轮品牌关系查询与过滤记录；过滤后非空必须输出映射表，
   为空必须静默省略该表；无“品牌”时必须不存在该表；
   品牌映射必须通过 [rule-transformation-guide.md](rule-transformation-guide.md)“品牌”小节的完整
   过滤、数量、格式和连续编号门禁；任一不符不得调用 S3。有“材质”时，对应比价项章节必须存在固定行
   `- **材质对照表**：见下方附录。`，且下方必须存在材质对照表附录，二者缺一不得调用 S3；过滤后
   有有效组则输出真实组，为 0 组则静默省略引用和材质表，不得补齐或阻断；品牌和材质均为空时省略
   整个 `## 映射表` 章节；无“材质”
   时必须不存在材质查询记录、材质表及任何材质说明。禁止用“未涉及映射”“若后续新增”“本轮不内嵌”
   等文字代替门禁结果。母子品牌标题不得包含“共 N 组”等动态组数。任一不符不得调用 S3。EDIT
   精确修改直达分支不执行本项全量映射门禁，只校验用户指定局部修改、必要机械一致性和全文结构；
   不得因未查询关系 MCP 或真实比价项集合不含品牌/材质而撤销用户明确指定的关系修改。
   拼接候选前必须先分别计算过滤后的 `brandGroupCount`、`materialGroupCount`，禁止先创建映射标题再
   填内容。某项计数为 0 时，该子表从标题到正文必须整体不存在；两项均为 0 时，`## 映射表` 及相邻
   分隔线必须整体不存在。最终全文出现“无符合”“无可输出”“未命中”“暂无”或任何映射门控说明，
   均视为空表泄漏，必须删除后重新生成，禁止调用 S3。
   材质不得只选代表组：必须遍历所有非叶子父节点建立覆盖账本；有效组不超过 50 时必须全部输出，
   超过 50 时才允许按规则截为 50。候选组数与覆盖账本有效组数（封顶 50）不一致时禁止 S3。
   非精确分支输出材质表前还必须建立内部材质分组账本：每组记录真实父节点 ID/名称、从一级根节点
   到该父节点的完整祖先路径、每个成员 ID/名称/`isLeaf`/`parentId`；至少一个
   `isLeaf=true && parentId=该组父节点ID` 的直接末端叶子即可成组，单成员组也必须保留。候选组名
   必须逐字符等于完整祖先路径按
   `·`连接的结果，成员必须与该父节点的直接叶子集合逐项一致。不同父节点合并、父节点充当成员、
   单叶子组、凭名称相似归组、路径漏级/增级/调序或使用非`·`分隔符时禁止 S3。
   必须将候选表每个成员逐项反查本轮响应并记录命中的节点 ID；存在任一 `isLeaf=false`、父节点不同、
   无唯一命中或把多级后代压平到一级类别下时禁止 S3。仅“名称存在于响应”不算通过。
6. 最终 `prompt_content` 整体约 1 万字（包含规则、映射和输出格式）。接近或超过时只删除重复解释、
   重复示例、同义表达，并按作用域过滤非必要映射；不得裁掉真实比价项、规则边界、例外、固定章节
   或 JSON 字段，也不得硬截断正文。
7. 仅当 EDIT 命中“当前规则全量同步分支”时，必须已经完成其 `editCoverageLedger`、适用类目表逐行校验和
   品牌/材质双向门禁。不得把“规则已查询”“基础正文格式合规”或“准备最小修改”等同于目标已覆盖；
   任一真实比价项未逐项核对，或用户请求的规则、品牌、材质任一维度被静默忽略时，不得调用下方
   校验工具。
调用：

`tool_validate_prompt_skeleton(prompt_content=<候选全文>, operator=上下文.operator, conversation_id=上下文.localConversationId, base_prompt_version_id=<INITIALIZE传0；EDIT传selectedPromptVersionId>, rule_group_id=上下文.ruleGroupId, agent_id=上下文.agentId)`

`rule_group_id`、`agent_id` 至少一个大于 0；EDIT 的 `selectedPromptVersionId>0`。初始化先处理
`data.prompt_exists`：为 true 时返回 `existing_prompt_version_id/name` 后停止；为 false 才继续。

- `base_resp.resp_code=1 && data.valid=true && data.diff_record_id>0`：成功，锁定本次请求传入的完整
  `prompt_content`，并保留同次响应的 `diff_record_id`、`diff_content`。后续展示继续引用这个
  已提交变量，禁止重新生成。
- `base_resp.resp_code=1 && data.valid=true && data.diff_record_id<=0`：立即异常结束，只回复
  “提示词校验未生成有效修改建议（diff_record_id=0），无法形成提案。”禁止重试、提案、确认、
  写入或展开全文。
- `valid=false`：只按 `data.errors` 修正报错处，其他正文不变，最多重试 2 次。
- 其他情况或重试后仍失败：只返回最终错误，不展示提案、不写入、不请求用户授权后再修复。

Diff 只取服务端 `data.diff_content`，禁止自行书写。校验调用会落 Diff 记录，只能首次及按 errors
重试时调用，不得为“再确认”重复调用。校验通过的候选原样保留至写入，不重新生成或调整。

## S4 展示提案

严格使用目标 workflow 的唯一模板：标题为第一行，字段和顺序不变，替换全部占位符并解析条件
分支；模板外不加寒暄、摘要或结论。

- 修改：只展示同次 S3 的非零 `diff_record_id` 和逐字符复制的 `diff_content`，使用
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

本节是确认意图命中后的唯一执行路径。进入后禁止输出任何自然语言或执行其他工具，必须先按下方
参数调用 `tool_edit_prompt_skeleton`。

只允许确认可见的上一条助手提案。若上一条 assistant 可见消息包含以下三项：

1. `## 提示词修改提案` 或 `## 提示词初始化提案`；
2. 非零修改建议 ID（`diff_id`）；
3. 明确确认话术；

则模型侧门禁已经通过，必须使用该非零 ID 调用写入工具。模型不得推测数据库消息序号、相邻状态、
Diff 状态或基础版本关联，也不得在调用前输出“上一条不是有效提案”“提案已失效”“请重新发起”。
这些事实全部由服务端写入工具校验。

只有上一条 assistant 可见消息客观缺少上述任一项时，才允许不调用工具并准确说明缺少哪一项；不得
笼统声称提案无效。没有本轮 `tool_edit_prompt_skeleton` 真实响应时，禁止生成任何服务端拒绝、确认
失败、版本不匹配或提案失效结论。

### 参数与调用

- `prompt_diff_record_id`：紧邻提案同次 S3 的非零 ID。
- `prompt_version_id`：进入 `[S5]` 时，从**当前确认回合注入的业务上下文块**读取
  `promptVersionId` 原始展开值，并记录为 `confirmationPromptVersionId`。原始值显式为非负整数时
  直接使用；字段未注入、为空或仍为模板占位符时，统一令 `confirmationPromptVersionId=0`。
  不得从上一条提案的“基础提示词 ID”、历史消息、上一轮工具参数、Diff 记录、版本名称、
  `new_prompt_version_id` 或模型记忆补值。只要存在有效非零 `prompt_diff_record_id`，就不得因
  `promptVersionId` 缺失而阻止确认写入，必须以 `prompt_version_id=0` 调用；Diff 与版本关联由服务端
  校验。仅当上下文显式提供负数或非数字值时停止，只回复：“当前确认回合提供的 promptVersionId
  非法，未执行提示词写入。”
- `source_type`：INITIALIZE=3，EDIT=2。原提案模式只决定此字段。
- 不传 `prompt_content`；服务端以 Diff 记录正文为准。

`promptVersionId>0` 时先按 [base-version-policy.md](base-version-policy.md) 精确查询本轮基础版本并
记录 `version_name/id/version_no`；失败则不写入。为 0 时基础提示词记为“无（初始化）”。

只调用一次：

`tool_edit_prompt_skeleton(rule_group_id=上下文.ruleGroupId, prompt_version_id=confirmationPromptVersionId, operator=上下文.operator, source_type=<3或2>, prompt_diff_record_id=<紧邻提案Diff ID>)`

调用前逐字段生成 `writeParameterLedger`：

- `prompt_diff_record_id` 来源必须是上一条可见提案；
- `prompt_version_id` 来源必须是当前确认回合业务上下文的显式非负整数，或由缺失、空值、模板占位符
  按规则规范化得到的 `0`；
- `rule_group_id`、`operator` 来源必须是当前确认回合业务上下文；
- `source_type` 只由原提案的 INITIALIZE/EDIT 模式决定。

任一字段来源不符合时禁止调用；但当前确认回合缺失 `promptVersionId` 属于合法规范化场景，不得阻止
调用。禁止因为提案中存在基础版本 ID 就把该 ID 当作当前确认回合的 `promptVersionId`；版本关联以
服务端校验结果为准。

调用返回前禁止产生用户可见文字。调用后只能依据本次真实 `base_resp/data/result/error_msg` 输出：成功
时使用下方成功模板；业务拒绝时逐字转述真实原因；调用失败或响应不完整时报告真实工具错误。禁止
用模型生成的理由替代工具响应。

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
