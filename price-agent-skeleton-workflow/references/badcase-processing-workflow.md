# Badcase 统一处理流程

本文件统一负责 Badcase 的输入查询、基础版本锁、规则/映射、CDN、证据、归因和处理动作。
入口 workflow 只负责锁定模式、维护该模式的必要状态并使用固定输出模板。锁定后必须连续执行
`[B0]`～`[B8]`，不得跳步、重复执行或在入口复制共同查询与处理逻辑。归因定义唯一来自
[badcase-attribution-policy.md](badcase-attribution-policy.md)，提示词校验与写入唯一来自
[shared-steps.md](shared-steps.md)。

## 入口模式锁

入口 workflow 必须在进入本文件前显式锁定且只锁定一个模式：

- `badcase-single-workflow.md` 锁定 `badcaseMode=SINGLE`；
- `badcase-task-workflow.md` 锁定 `badcaseMode=TASK`；
- `badcase-description-workflow.md` 锁定 `badcaseMode=DESCRIPTION`。

本文件只按已锁定模式执行对应查询分支，不得根据样本数量、字段是否存在或用户文字重新推断、选择
或切换模式。最终输出格式仍只由入口 workflow 定义。

## 统一内部分析记录

这不是接口对象、Java 类或需要向用户输出的 JSON。Agent 只需在内部为每条样本整理以下分析信息，
所有内容只能来自本轮可信输入或工具响应：

| 分析信息 | 内容要求 |
|---|---|
| 样本标识 | `SINGLE/TASK` 使用 `validation_case_id`；`DESCRIPTION` 记为“用户描述” |
| 类目 | 使用真实 `category_id`；无可信值时保持缺失 |
| 左右商品事实 | 分别记录标题、属性、商详和主图中的客观事实 |
| 人工标签与模型结论 | 记录人工标签、模型标签和模型理由 |
| 模型分析过程 | 记录各比价项左右 `value/source/match` |
| 规则与映射依据 | 记录本轮真实类目规则和必要映射 |
| 规则转换对照 | 逐项记录原始 `infoSource/compareLogic`、`expectedPriority/expectedMatch` 与提示词实际值 |
| 提示词对照 | 记录基础提示词名称、ID、相关原文和位置 |
| 唯一一级分类 | 从三类中只选择一个；标签不确定或证据不足时记录待核验结果，不强行分类 |
| 口径与根因证据 | 说明 SKU/SPU 口径事实及证据如何支持该分类和具体原因 |
| 处理动作 | 说明责任方和具体处理方式 |
| 回归方案 | 说明修复或补证后如何验证 |
| 是否修改提示词 | 第 2、3 类且具体原因可直接定位到提示词缺陷时写“是”；其余写“否” |

人标或模型任一侧为“不确定”时保留原值并进入 `[B5]` 的标签门禁。任何必要字段缺失都保留为空并
进入 `[B5]` 判断，不得用标题推类目、用模型抽取补商品事实、用当前规则
伪装验证时规则快照，或用另一版本提示词替换基础提示词。

## B0 校验入口模式锁

读取入口已锁定的 `badcaseMode`，只接受 `SINGLE`、`TASK`、`DESCRIPTION` 之一。`SINGLE/TASK` 还
必须与入口建立的 `badcaseRouteContext` 一致：上下文 `validationCaseId>0` 只能是 `SINGLE`，否则
上下文 `validationTaskId>0` 才能是 `TASK`。模式缺失、值不合法或不一致时立即停止并按入口重新
路由；禁止从用户文字解析 ID、自行补值或继续错误模式。一次分析中不得切换模式；整任务继续分析和
整合必须保持 `badcaseMode=TASK` 并通过原续批锁。

## B1 查询输入并锁定基础版本

仅当业务上下文缺少 `ruleGroupId` 时执行 `[S1]`；已有则跳过。随后只执行当前 `badcaseMode` 对应分支：

- `SINGLE`：只从本轮业务上下文取得 `validationTaskId`、`validationCaseId`、`promptVersionId` 和
  `operator`，三个 ID 都必须大于 0。只使用以下四个字段调用：

  `tool_query_validation_result(validation_task_id=上下文.validationTaskId, validation_case_id=上下文.validationCaseId, prompt_version_id=上下文.promptVersionId, operator=上下文.operator)`

  禁止附加 `label_filter` 或 `page`。只接受 `base_resp.resp_code=1 && result=1`，并要求响应任务 ID、
  Prompt ID、唯一一条结果的 Case ID 分别与请求一致，且 `is_correct=0`。
- `TASK`：只从本轮业务上下文取得 `validationTaskId`、`promptVersionId` 和 `operator`，前两个 ID
  必须大于 0。首批页码为 1，继续分析页码为上一批 `page_no+1`；每页固定 10 条。只使用以下字段调用：

  `tool_query_validation_result(validation_task_id=上下文.validationTaskId, label_filter=3, prompt_version_id=上下文.promptVersionId, operator=上下文.operator, page={page_no:<当前批次页码>,page_size:10})`

  禁止传 `validation_case_id`。只接受 `base_resp.resp_code=1 && result=1`，并要求响应任务 ID、Prompt ID
  与请求一致，所有结果均为 `is_correct=0`；继续分析还必须通过入口维护的 `badcaseTaskLock`。
- `DESCRIPTION`：只使用本轮业务上下文的 `promptVersionId`，要求大于 0；按
  [base-version-policy.md](base-version-policy.md) 精确查询当前提示词，并要求响应 Prompt ID 与上下文
  一致。左右商品事实、人工标签、模型结论、理由和模型过程只取用户本轮输入，缺失项保持为空；
  禁止从用户自由文本接受 Prompt ID 替换页面上下文。

`SINGLE/TASK` 只按以下精确路径读取，禁止根据字段含义猜测路径：

- 任务：`data.validation_task_id`、`data.dataset_id`、`data.validation_summary_json`；
- 基础提示词：`data.prompt_version.{prompt_version_id,rule_group_id,version_no,version_name,
  version_status,prompt_content}`；
- 样本：`data.results[].{validation_result_id,validation_case_id,category_id,source_item_id,
  candidate_item_id,source_title,candidate_title,human_label,model_label,is_correct,reason,
  analysis_process,radar_task_id,radar_task_url,source_item_json,candidate_item_json}`；
- `TASK` 分页：`data.page.{page_no,page_size,total,has_more}`。

`dataset_id` 和每条 `category_id` 必须大于 0；上下文 `ruleGroupId>0` 时，响应
`data.prompt_version.rule_group_id` 必须与其一致。继续分析还要求响应 `dataset_id`、`rule_group_id`
与 `badcaseTaskLock` 一致。响应没有 `agent_id`，不得声称本接口校验了 Agent。三种模式均要求
`prompt_content` 非空；响应通过后统一锁定 `selectedPromptVersionId=上下文.promptVersionId`、
`writeMode=EDIT`，该正文就是分析基础版本，不再查询其他提示词，也不得用响应 Prompt ID 覆盖上下文
值。任何必需 ID、正文、规则组或请求/响应一致性校验失败时立即停止。

## B2 规范化样本

逐条整理内部分析记录中的商品事实、标签、理由和模型分析过程。`SINGLE/TASK` 必须
校验 `is_correct=0`；`DESCRIPTION` 必须保留用户没有提供的字段为空。先按 Case 去重再分析，同一
Case 不得因重复分页被统计两次。

## B3 查询规则、必要映射与报告

- `SINGLE`：只使用 `data.results[0].category_id` 按 `[S2]` 查询该类目规则与规则实际需要的映射；
  随后调用
  `tool_query_cdn_report(validation_task_id=上下文.validationTaskId, prompt_version_id=上下文.promptVersionId, operator=上下文.operator)`，
  只读取 `data.report_cdn_url` 和 `data.report_summary_json`。
- `TASK`：提取本页 `data.results[].category_id`，去重后逐类目按 `[S2]` 查询规则与必要映射，禁止
  跨类目混用；随后按同一任务和 Prompt ID 调用上述 `tool_query_cdn_report`。
- `DESCRIPTION`：仅从本轮可信业务上下文或用户提供的可验证结构化字段取得 `categoryId`；存在时
  按 `[S2]` 查询争议范围内的规则与必要映射，不存在时保持规则/映射为空并进入 `[B5]`；不查询 CDN。

三种模式都禁止从标题、商品 JSON、图片、提示词或模型输出猜类目。规则/映射查询不得重复；CDN
链接为空不阻断分析，报告摘要不能替代逐条验证结果。规则查询成功后必须完整读取
[rule-transformation-guide.md](rule-transformation-guide.md) 和 [enums.md](enums.md)，供 `[B4]`
计算每个争议比价项的标准期望值；不得把规则原文未经转换直接与提示词比较。

必要映射按争议项和真实 `compareItem` 双重门控：争议项为“材质”且本轮规则确实包含“材质”时，
必须按 `[S2]` 调用 `radar_query_material_relation` 并保留本轮原始响应、过滤后的合法材质组及节点证据；
争议项为“品牌”且本轮规则确实包含“品牌”时，必须调用 `radar_query_brand_relation` 并保留本轮原始
响应及过滤后的合法品牌组。
真实 `compareItem` 不包含对应项时禁止为了分析而查询或引用该映射。工具失败、响应缺失或无法确认
关系时保留为证据缺口，不得用模型常识补映射。

## B4 建立逐项证据表

对每个争议比价项建立以下四层对照，字段不得合并或省略：

1. **客观事实层**：左/右快照中商品标题、商品属性、主图分别提供的事实及精确位置；人工/模型标签。
   “买一发三”等数量事实只有在快照标题、属性或主图中明确出现时才能记录，禁止根据模型
   `value`、促销常识或最终标签反推。
2. **规则期望层**：原始 `compareItem`、原始 `infoSource`、原始 `compareLogic`、该项全部特殊规则，
   以及严格按 [rule-transformation-guide.md](rule-transformation-guide.md) 和 [enums.md](enums.md)
   计算出的 `expectedPriority`、`expectedMatch`。来源过滤必须扫描原始 `infoSource` 的全部片段，
   包括冒号后或第二段来源链；同一规范来源重复时去重但不得截断后续合法来源。
3. **提示词实现层**：基础提示词中该比价项的实际优先级、全部匹配条件及原文位置；分别与
   `expectedPriority`、`expectedMatch` 做精确对比，明确记录缺失来源、来源重排、额外来源、规则遗漏、
   规则增写或完全一致。禁止只写笼统的“规则与映射依据”。
4. **模型执行层**：`analysis_process` 中左/右 `value/source`、`match`、理由和最终模型标签；与客观
   事实层并列比较，确认模型是否读取了规则允许的真实来源。禁止用模型自己的抽取结果证明商品事实。

品牌和材质还必须增加“映射传递链”核验：

- **材质**：先确认左右客观材质；再在本轮材质关系响应中逐项定位节点，只有同一真实 `parentId`
  下的直接 `isLeaf=true` 叶子才属于同组；然后检查该合法组是否完整写入基础提示词材质附录；最后
  检查模型 `analysis_process` 是否抽取了两侧材质并实际应用该组。依次区分“关系库没有该组”
  “关系库有但提示词省略”“提示词已有但模型省略某侧材质”“模型已抽取但未应用映射”。禁止因为
  两个名称都出现在关系响应中就认定同组。
- **品牌**：先确认左右客观品牌；再检查本轮母子品牌响应是否存在该真实关系且通过当前类目过滤；
  然后检查该组是否完整写入基础提示词母子品牌表；最后检查模型是否抽取了两侧品牌并应用该关系。
  依次区分“关系库没有该组”“关系库有但提示词省略”“提示词已有但模型省略某侧品牌”“模型已
  抽取但未应用映射”。同集团但不在本轮真实关系中的品牌不得凭常识互认。

除品牌/材质映射外，每条争议项还必须按顺序排查以下异常来源并只记录有证据支持的项：

1. **商品事实或图片识别问题**：快照缺字段、图片模糊、OCR 错字、主体定位错误；
2. **来源转换问题**：合法来源被遗漏、重排或补入，冒号后第二段来源链被截断；
3. **提示词规则问题**：特殊规则遗漏或错误、`compareLogic` 未展开/展开错误、模型常识被擅自加入；
4. **模型抽取问题**：提示词已允许该来源，但模型漏抽、错抽、左右值或 `source` 填错；
5. **逐项匹配问题**：抽取值正确但 `match` 未按特殊规则、兜底或映射计算；
6. **Step3 聚合问题**：逐项 `match` 正确但最终 `result` 错误，尤其检查货号例外；
7. **规则或版本不一致**：分析使用的当前规则、关系或 Prompt 与验证任务实际快照不一致；
8. **标签及 SKU/SPU 口径问题**：人工标签不确定、模型标签不确定，或属于合理粒度差异。

同一条可能同时存在多个现象，但 `[B6]` 仍须依据最直接的错误因果链选择唯一一级分类；无法确定
哪一环导致最终错误时进入证据不足，不得把所有可能原因都写成已确认根因。

例如原始数量来源为
`比价维度>sku图文字说明>商品标题>商品内容物数量：sku标题>商品外包装文字说明>sku图文字说明>商品标题>主图文字说明>详情图文字说明`
时，扫描全部片段并稳定过滤后必须得到`expectedPriority=商品标题>主图`：两个“商品标题”归一后
去重，`主图文字说明`归一为`主图`，其余来源删除。若提示词只写`商品标题`，证据表必须记录
“缺失主图来源”，不得判为规则同步完成。本例只说明转换算法，不预设任何样本的数量事实或根因。

最后单独记录因果结论：提示词差异是否实际导致本条模型漏抽取、错匹配或错判。存在提示词差异但
模型已从其他合法来源正确取得同一事实时，只能记录“提示词存在缺陷但尚不能证明是本条误判原因”；
不得仅凭文本不一致直接归因为提示词问题。

## B5 证据充分性门禁

先检查标签：人标或模型任一侧为“不确定”时，输出“待核验（标签不确定）”，记录不确定侧并停止
分类和提示词修改。标签明确后，再判断能否确认商品事实、模型实际处理、可信规则、
`expectedPriority/expectedMatch` 与提示词的精确差异及错误因果链。缺少决定性图片、属性、类目、规则
或结构化模型过程时，输出“待核验（证据不足）”并停止分类；不得为了形成修改建议把未知项写成事实。
验证时规则快照不可用且判断依赖规则版本变化时，同样停止并建议重跑。“证据不足”不是一级分类。

## B6 选择唯一一级分类

完整执行 [badcase-attribution-policy.md](badcase-attribution-policy.md) 的 SKU/SPU 口径门禁和三类条件，
为每条证据充分且标签明确的样本填写“唯一一级分类、口径与根因证据、具体原因、处理动作、回归方案、
是否修改提示词”。标签不确定或证据不足时记录对应待核验结果并停止，不得强行三选一。整任务必须
先逐条执行门禁和分类，再统计三类、标签不确定和证据不足数量。

## B7 执行处理动作

- “合理的 SKU/SPU 粒度差异”、两种待核验结果，或第 2、3 类但具体原因不在提示词：将“是否修改提示词”
  记为“否”，不生成候选正文，不调用 `[S3]`，不输出 Diff、校验成功或确认创建话术。
- 第 2、3 类且具体原因可直接定位到提示词缺陷：将“是否修改提示词”记为“是”。单条/描述基于该条证据生成一份非空
  候选完整提示词；整任务仅在用户要求整合时合并同一锁下已确认缺陷并生成一份候选完整提示词。
  随后严格执行 [shared-steps.md](shared-steps.md) `[S3]`，实际调用：

  `tool_validate_prompt_skeleton(prompt_content=<候选完整提示词>, operator=上下文.operator, conversation_id=上下文.localConversationId, base_prompt_version_id=selectedPromptVersionId, rule_group_id=上下文.ruleGroupId, agent_id=上下文.agentId)`

  只有同次响应满足
  `base_resp.resp_code=1 && data.valid=true && data.diff_record_id>0 && data.diff_content非空`，才形成
  可见提案；Diff 必须逐字符复制该响应的 `data.diff_content`。失败时按 `[S3]` 返回真实错误，禁止
  输出提案或确认话术。

## B8 输出与回归

使用入口文件唯一固定模板，不得由统一内核另造通用回复。输出中的一级分类、具体原因、是否修改
提示词、处理建议和 Diff 必须与内部分析记录一致。整任务三类数量、标签不确定数量与证据不足数量
之和必须等于本批
去重明细数；只有“是否修改提示词=是”的第 2、3 类样本进入候选修改合并。

确认创建草稿仍只按 [shared-steps.md](shared-steps.md) `[S5]` 执行，统一内核不得直接写入。
