# Badcase 统一处理流程

本文件统一负责 Badcase 的输入查询、基础版本锁、规则/映射、CDN、证据、归因和处理动作。
入口 workflow 只负责锁定模式、维护该模式的必要状态并使用固定输出模板。锁定后必须连续执行
`[B0]`～`[B8]`，不得跳步、重复执行或在入口复制共同查询与处理逻辑。归因定义唯一来自
[badcase-attribution-policy.md](badcase-attribution-policy.md)，提示词校验与写入唯一来自
[shared-steps.md](shared-steps.md)。

## 入口模式锁

入口 workflow 必须在进入本文件前显式锁定且只锁定一个模式：

- `badcase-single-workflow.md` 锁定 `badcaseMode=SINGLE`；
- `badcase-task-workflow.md` 锁定 `badcaseMode=TASK`。

本文件只按已锁定模式执行对应查询分支，不得根据样本数量、字段是否存在或用户文字重新推断、选择
或切换模式。最终输出格式仍只由入口 workflow 定义。

## 统一内部分析记录

这不是接口对象、Java 类或需要向用户输出的 JSON。Agent 只需在内部为每条样本整理以下分析信息，
所有内容只能来自本轮可信输入或工具响应：

| 分析信息 | 内容要求 |
|---|---|
| 样本标识 | 使用 `validation_case_id` |
| 类目 | 使用真实 `category_id`；无可信值时保持缺失 |
| 左右商品事实 | 分别记录标题、属性、商详和主图中的客观事实 |
| 人工标签与模型结论 | 记录人工标签、模型标签和模型理由 |
| 模型分析过程 | 记录各比价项左右 `value/source/match` |
| 规则与映射依据 | 记录本轮真实类目规则和必要映射 |
| 规则转换对照 | 逐项记录原始 `infoSource/compareLogic`、`expectedPriority/expectedMatch` 与提示词实际值 |
| 提示词对照 | 记录基础提示词名称、ID、相关原文和位置 |
| 唯一结论 | 从明确结论中只选择一个；标签不确定或证据不足时记录待核验结果，不强行归因 |
| 口径与根因证据 | 说明 SKU/SPU 口径事实及证据如何支持该分类和具体原因 |
| 处理动作 | 说明责任方和具体处理方式 |
| 回归方案 | 说明修复或补证后如何验证 |
| 是否修改提示词 | 模型判得过严或过宽且具体原因可直接定位到提示词缺陷时写“是”；其余写“否” |

人标或模型任一侧为“不确定”时保留原值并进入 `[B5]` 的标签门禁。任何必要字段缺失都保留为空并
进入 `[B5]` 判断，不得用标题推类目、用模型抽取补商品事实、用当前规则
伪装验证时规则快照，或用另一版本提示词替换基础提示词。

## B0 校验入口模式锁

读取入口已锁定的 `badcaseMode`，只接受 `SINGLE`、`TASK` 之一，并
必须与入口建立的 `badcaseRouteContext` 一致：上下文 `validationCaseId>0` 只能是 `SINGLE`，否则
上下文 `validationTaskId>0` 才能是 `TASK`。模式缺失、值不合法或不一致时立即停止并按入口重新
路由；禁止从用户文字解析 ID、自行补值或继续错误模式。一次分析中不得切换模式；整任务继续分析和
整合必须保持 `badcaseMode=TASK` 并确认各批服务端续批身份一致。

## B1 查询输入并锁定基础版本

仅当业务上下文缺少 `ruleGroupId` 时执行 `[S1]`；已有则跳过。随后只执行当前 `badcaseMode` 对应分支：

- `SINGLE`：只从本轮业务上下文取得 `validationTaskId`、`validationCaseId`、`promptVersionId` 和
  `operator`，三个 ID 都必须大于 0。只使用以下四个字段调用：

  `tool_query_validation_result(validation_task_id=上下文.validationTaskId, validation_case_id=上下文.validationCaseId, prompt_version_id=上下文.promptVersionId, operator=上下文.operator)`

  禁止附加 `label_filter` 或 `page`。只接受 `base_resp.resp_code=1 && result=1`，并要求响应任务 ID、
  Prompt ID、唯一一条结果的 Case ID 分别与请求一致，且 `is_correct=0`。
- `TASK`：首批只从本轮业务上下文取得 `validationTaskId`、`promptVersionId`、`operator` 和
  `localConversationId`，三个 ID 必须大于 0。首批页码为 1、每页固定 10 条，只使用以下字段调用：

  `tool_query_validation_result(validation_task_id=上下文.validationTaskId, label_filter=3, prompt_version_id=上下文.promptVersionId, operator=上下文.operator, conversation_id=上下文.localConversationId, page={page_no:1,page_size:10})`

  禁止传 `validation_case_id` 和 `continuation_token`。只接受 `base_resp.resp_code=1 && result=1`，并要求
  响应任务 ID、Prompt ID 与请求一致，所有结果均为 `isCorrect=0`。`data.data.page.hasMore=true`
  时还必须存在非空 `data.data.continuationToken`；`hasMore=false` 时该字段必须为空。

  继续分析时，当前用户消息必须紧邻上一条成功的整任务阶段分析助手回复；中间出现任何其他用户
  消息后，上一批 Token 永久失去续批资格。通过紧邻门禁后，只能从该上一批成功工具响应取得完整
  `data.data.continuationToken`；该 Token 已由上一批固定模板原样展示，但续批时仍只能从紧邻上一批
  成功工具响应取得，不得从用户粘贴内容或更早历史回复恢复。不得解析、改写或根据历史页码生成。
  仅使用以下三个字段调用：

  `tool_query_validation_result(conversation_id=上下文.localConversationId, operator=上下文.operator, continuation_token=<上一批服务端原样值>)`

  续批禁止传 `validation_task_id`、`prompt_version_id`、`label_filter`、`validation_case_id` 或 `page`。
  非紧邻，或 Token 缺失、空、过期、上下文不匹配、工具失败时立即停止，不得重试、猜页码、恢复
  历史 Token 或改用历史 ID；要求用户通过页面“一键分析 Badcase”重新发起。

`SINGLE/TASK` 按工具真实响应的 camelCase 路径读取。外层成功门禁为
`result=1 && data.baseResp.respCode=1`，业务数据根节点为 `data.data`：

- 任务：`data.data.validationTaskId`、`data.data.datasetId`、`data.data.validationSummaryJson`；
- 基础提示词：`data.data.promptVersion.{promptVersionId,ruleGroupId,versionNo,versionName,
  versionStatus,promptContent}`；
- 样本：`data.data.results[].{validationResultId,validationCaseId,categoryId,sourceItemId,
  candidateItemId,sourceTitle,candidateTitle,humanLabel,modelLabel,isCorrect,reason,
  analysisProcess,radarTaskId,radarTaskUrl,sourceItemJson,candidateItemJson}`；
- `TASK` 分页：`data.data.page.{pageNo,pageSize,total,hasMore}`、`data.data.continuationToken`。

`analysisProcess`、`sourceItemJson`、`candidateItemJson` 都是 JSON 字符串，必须先解析后读取；解析失败
保留原始字符串并进入 `[B5]`，禁止把字符串当成已解析对象访问。下文为便于描述，将解析结果分别记为
`parsedAnalysis`、`parsedSourceItem`、`parsedCandidateItem`。

`datasetId` 和每条 `categoryId` 必须大于 0；上下文 `ruleGroupId>0` 时，响应
`data.data.promptVersion.ruleGroupId` 必须与其一致。续批的任务、Prompt、验证集、规则组、操作人、会话和页码
由服务端 Token 校验；响应仍必须与首批阶段记录的 `validationTaskId`、`promptVersionId`、`datasetId`
和 `ruleGroupId` 一致。响应没有 `agentId`，不得声称本接口校验了 Agent。两种模式均要求
`promptContent` 非空；响应通过后统一锁定 `selectedPromptVersionId=上下文.promptVersionId`、
`writeMode=EDIT`，该正文就是分析基础版本，不再查询其他提示词，也不得用响应 Prompt ID 覆盖上下文
值。任何必需 ID、正文、规则组或请求/响应一致性校验失败时立即停止。

## B2 规范化样本

逐条整理内部分析记录中的商品事实、标签、理由和模型分析过程，必须
校验 `is_correct=0`。先按 Case 去重再分析，同一
Case 不得因重复分页被统计两次。

标签方向只能根据结构化字段原始数值确定，不得从 `reason`、`analysis_process` 或自然语言反推：

| 字段 | 原始值 | 含义 |
|---|---:|---|
| `humanLabel` | `1` | 人工标签：同款 |
| `humanLabel` | `2` | 人工标签：非同款 |
| `humanLabel` | `3` | 人工标签：无法判断 |
| `modelLabel` | `1` | 模型结论：同款 |
| `modelLabel` | `2` | 模型结论：非同款 |
| `modelLabel` | `3` | 模型结论：无法判断 |

每条内部分析记录必须同时保留并展示 `humanLabel=<原始值>`、`modelLabel=<原始值>` 及其中文映射。
`humanLabel` 不为 `1/2/3`、`modelLabel` 不为 `1/2/3` 或字段缺失时，记录非法/缺失原值并进入 `[B5]`，
禁止自行补值。`humanLabel=3` 或 `modelLabel=3` 时，固定进入“待核验（标签不确定）”，不得继续
建立逐项证据或判断 Prompt、模型、人工标签及 SKU/SPU 归因。

## B3 查询规则与报告

- `SINGLE`：只使用 `data.data.results[0].categoryId` 按 `[S2]` 查询该类目规则；
  随后调用
  `tool_query_cdn_report(validation_task_id=上下文.validationTaskId, prompt_version_id=上下文.promptVersionId, operator=上下文.operator)`，
  只读取 `data.report_cdn_url` 和 `data.report_summary_json`。
- `TASK`：提取本页 `data.data.results[].categoryId`，去重后逐类目按 `[S2]` 查询规则，禁止
  跨类目混用；随后按同一任务和 Prompt ID 调用上述 `tool_query_cdn_report`。

两种模式都禁止从标题、商品 JSON、图片、提示词或模型输出猜类目。规则查询不得重复；CDN
链接为空不阻断分析，报告摘要不能替代逐条验证结果。规则查询成功后必须完整读取
[rule-transformation-guide.md](rule-transformation-guide.md) 和 [enums.md](enums.md)，供 `[B4]`
计算 `[B4]` 标签冲突表规定范围内每个比价项的标准期望值；不得把规则原文未经转换直接与提示词比较。

## B4 建立逐项证据表

先根据明确的人工标签与模型标签确定核验范围，不得预先猜测“争议比价项”。本表只决定查哪些项，
不直接决定整条样本的最终归因：

| 标签冲突 | 必须核验的比价项 | 核验目标 |
|---|---|---|
| `humanLabel=2`、`modelLabel=1`（人标非同、模型标同） | 当前类目规则中的所有有效比价项；逐项寻找模型遗漏的不匹配项 | 查找足以使 SPU 判非同的决定性差异，并确认模型是否漏抽、漏判或错误放行 |
| `humanLabel=1`、`modelLabel=2`（人标同、模型标非同） | 解析后的 `parsedAnalysis` 中所有 `match=false` 的比价项；不得只选其中一个 | 逐项确认模型用于否决同款的事实、来源、匹配和聚合是否成立 |

### B4.1 逐项结果枚举

每个待核验比价项完成规则、Prompt、商品事实、来源优先级、图片、模型抽取和匹配校验后，必须填写
唯一 `itemFinding`；不得直接在逐项证据表中填写“模型错误 / Prompt 缺陷 / 人工标签疑似错误”等整条归因：

| `itemFinding` | 判定条件 | 后续动作 |
|---|---|---|
| `ITEM_OK` | Prompt 正确实现规则，模型取值、来源和 `match` 均与客观证据及规则一致 | 保留为已核验正常项 |
| `PROMPT_DEFECT_CAUSAL` | Prompt 与规则不一致，且“不调用接口的规则因果复核”确认该差异会改变最终错误结论 | 记录直接 Prompt 根因，交 `[B6]` 归因为 Prompt 缺陷 |
| `PROMPT_DEFECT_NON_CAUSAL` | Prompt 与规则不一致，但该差异未改变本项有效结论、仍有其他合法决定项，或不能证明会改变最终结论 | 记录伴随 Prompt 缺陷，不得仅凭本项修改 Prompt |
| `MODEL_SOURCE_ERROR` | Prompt 来源优先级正确，但模型跳过有明确值的更高优先级来源、使用不允许来源，或 `source` 缺失/无法归一且实际路径可确认错误 | 记录模型来源执行错误 |
| `MODEL_EXTRACTION_ERROR` | Prompt 已允许正确来源，但模型漏抽或错抽客观值 | 记录模型抽取错误 |
| `MODEL_MATCH_ERROR` | 左右值及来源正确，但模型 `match` 未按规则、特殊条件或合法映射计算 | 记录模型逐项匹配错误 |
| `EVIDENCE_INSUFFICIENT` | 图片、商品事实、规则、Prompt、映射或模型过程缺失、模糊或冲突，无法完成本项判断 | 列明缺失证据，交 `[B5]` 判断是否阻断整条归因 |

`itemFinding` 只描述单个比价项。人工标签是否错误、是否属于 SKU/SPU 维度差异、模型最终聚合是否错误，
都不是逐项结果，必须在全部必查项完成后由 `[B6]` 选择唯一 `caseAttribution`。

### B4.2 执行顺序

对每条样本严格按下表执行。除 Prompt 和规则外，字段路径均相对于当前
`data.data.results[当前样本]`；不得跳步、提前归因或根据字段含义猜测其他路径：

| 顺序 | 读取字段或工具结果 | 必须执行 | 产出 |
|---:|---|---|---|
| 1 | `humanLabel`、`modelLabel` | 原样记录数值并按 `[B2]` 枚举映射；任一为 `3` 或非法/缺失时停止逐项核验 | 标签方向或 `LABEL_UNCERTAIN` |
| 2 | 解析 `analysisProcess` 得到 `parsedAnalysis`；实际结构为 `Map<比价项名,{left,right,match,reason}>` | `humanLabel=2 && modelLabel=1` 时以真实类目规则的全部有效 `compareItem` 为范围；`humanLabel=1 && modelLabel=2` 时筛选 `parsedAnalysis` 中全部 `match=false` 的键 | 完整待核验比价项清单 |
| 3 | 本轮真实类目规则中的 `compareItem/infoSource/compareLogic/特殊规则`；必要时读取品牌或材质映射工具响应 | 按待核验项查询必要映射，并按规则转换指南计算标准值 | 每项 `expectedPriority/expectedMatch` 及合法映射 |
| 4 | `data.data.promptVersion.promptContent` | 定位每个待核验项的优先级、匹配条件及原文位置，与标准值精确对比 | Prompt 实际优先级/匹配条件及差异 |
| 5 | 左侧 `sourceTitle` 和解析后的 `parsedSourceItem.{detail,imageUrl,brand,shopName,categoryName}`；右侧 `candidateTitle` 和 `parsedCandidateItem.{detail,imageUrl,brand,shopName,categoryName}` | 按规则和 Prompt 允许来源读取标题、属性/商详、品牌及必要图片；图片必须按图片补充校验执行 | 左右客观值、来源、原始证据及歧义 |
| 6 | 当前比价项 `parsedAnalysis[比价项名].{left.value,left.source,right.value,right.source,match,reason}` | 按待核验比价项名称精确取同名键；将模型值、来源和匹配结果与步骤 3～5 并列比较 | 来源执行、抽取和逐项匹配核验结果 |
| 7 | 步骤 3 的正确规则与步骤 5 的客观证据 | 重新计算该项标准左右值及核验后 `match`，不得沿用模型值自证 | 唯一 `itemFinding`、直接根因或伴随异常 |
| 8 | 全部必查项的 `itemFinding`、核验后 `match`、顶层 `modelLabel` 和 `reason` | 检查逐项结果能否支持顶层模型结论；再按下方 B4.3 分支判断并交 `[B5]`、`[B6]` | 唯一 `caseAttribution` 或待核验结果 |

`analysisProcess` 只表示模型实际处理过程，只允许用于步骤 2 的否决项筛选和步骤 6 的模型执行核验；
不得使用其中的 `value/source/reason` 代替步骤 5 的左右商品客观证据。`reason` 只作辅助说明，不能覆盖
结构化的比价项键、`left/right.value/source` 和 `match`。

### B4.3 逐项判断树

完成 B4.2 后，严格执行下表。每一行都必须写出“比较对象、比较结果、命中分支”，不得只给最终枚举：

| 判断顺序 | 比较对象 | 命中条件 | 结果或下一步 |
|---:|---|---|---|
| 1 | 真实规则的 `expectedPriority/expectedMatch` vs `promptContent` 实际实现 | 不一致 | 先记录 Prompt 差异，再执行步骤 2 的规则因果复核 |
| 2 | 用正确规则、已确认客观值计算本项标准 `match`，并按任务 Prompt 明确的 Step3 规则聚合全部已核验项 | 标准最终结论与当前错误 `modelLabel` 不同，且与证据支持的正确方向一致 | `PROMPT_DEFECT_CAUSAL` → 整条 `PROMPT_DEFECT`；否则记 `PROMPT_DEFECT_NON_CAUSAL` 并继续 |
| 3 | 左右客观事实 vs `parsedAnalysis[比价项名].left/right.value/source` | Prompt 正确，但模型来源选择、漏抽或错抽导致值不一致 | `MODEL_SOURCE_ERROR` 或 `MODEL_EXTRACTION_ERROR` |
| 4 | 按正确规则计算的标准 `match` vs `parsedAnalysis[比价项名].match` | 模型值/来源正确，但 `match` 不一致 | `MODEL_MATCH_ERROR` |
| 5 | 全部核验后 `match` 的 Step3 聚合结果 vs 顶层 `modelLabel` | 逐项均正确但顶层结论不一致 | 模型聚合错误 → 整条 `MODEL_ERROR` |
| 6 | 全部必查项、SPU 规则和人工 SKU 标签 | 模型 SPU 结论正确，差异仅来自 SKU/SPU 判定口径 | `SKU_SPU_SCOPE_DIFFERENCE` |
| 7 | 全部必查项、SPU 规则和人工标签 | 模型结论有充分证据，且不能用 SKU/SPU 口径解释人标冲突 | `HUMAN_LABEL_SUSPECTED_ERROR`，交标注侧复核 |
| 8 | 以上任一步所需的决定性规则、Prompt、商品、图片、映射或模型过程 | 缺失、模糊、解析失败或冲突未解决 | `EVIDENCE_INSUFFICIENT` |

模型错误分支也必须检查是否改变最终结论：能改变才归因为 `MODEL_ERROR`，否则只记伴随异常。

### B4.4 操作细则

#### 来源优先级

对每个待核验比价项，无论优先级是否包含图片，都必须结合 `expectedPriority`、提示词实际优先级和
`parsedAnalysis[比价项名]` 中左右侧实际记录的 `source`，逐侧执行来源优先级校验：

1. 将左右侧 `source` 分别归一为提示词优先级中的规范来源，明确模型实际使用的来源层级；
2. 按提示词实际优先级从高到低检查位于实际 `source` 之前的所有来源，并逐项记录“有明确有效值/
   无有效值/来源缺失、模糊或不可读”；
3. 更高优先级来源存在明确有效值，模型却使用低优先级来源时，记录“来源优先级执行异常”，并列出
   被跳过的高优先级值和模型实际采用的低优先级值；
4. 所有更高优先级来源均无有效值时，模型降级使用当前来源合法；
5. 任一更高优先级来源缺失、模糊或无法确认时，不得判定模型来源选择正确或错误，保留证据缺口并
   进入 `[B5]`；
6. `source` 缺失、无法归一或不属于提示词允许来源时，记录“模型来源信息异常”；
   无法确认实际取值路径时进入 `[B5]`；
7. 左右两侧必须分别完成上述校验，不得只检查其中一侧。

例如某项提示词实际优先级为 `主图>商品标题`，模型 `source=商品标题`：必须先检查主图。
若主图已清晰提供有效值，则确认模型跳过主图并错用标题，属于“来源优先级执行异常”；若主图无有效
信息，则使用标题属于合法降级；若主图缺失或无法看清，则进入证据不足，不得二选一猜测。

#### 图片

任一待核验比价项满足以下任一条件时，除完成上述通用来源优先级校验外，还必须读取对应图片：

- `expectedPriority` 或提示词实际优先级包含 SKU 图、主图、主图文字、外包装文字、详情图或其他图片来源；
- 标题、属性或商详不能确认该项事实，但图片可能提供决定性证据；
- 模型 `source` 为图片来源；
- 图片与标题、属性、商详或模型抽取值存在冲突。

图片核验必须分别记录左右侧的：图片类型及位置、可见原始文字/OCR、可辨识的商品主体或外观特征、
对应比价项的观察值、是否清晰可信及歧义。不得只写“已看图”或“主图显示”，也不得用“常规单品”
“通常为 1 件”等常识代替图片证据。

#### 不调用接口的规则因果复核

本步骤不是重新运行模型或调用验证接口，只做确定性规则推导：

1. 使用步骤 B4.2 已确认的左右客观值和正确 `expectedMatch`，计算该项标准 `match`；
2. 对其他必查项也只使用已核验的标准 `match`；
3. 严格读取任务 `promptContent` 中明确的 Step3 聚合规则。本例接口为“全部 `match=true` → same；
   任一 `match=false` → different”，不得自行发明其他聚合逻辑；
4. 用标准 `match` 集合推导标准最终结论，并与当前 `modelLabel` 比较；
5. Prompt 差异使标准最终结论从当前错误方向改变为证据支持的正确方向时，记
   `PROMPT_DEFECT_CAUSAL`；结果不变时记 `PROMPT_DEFECT_NON_CAUSAL`；任一客观值、标准 `match`、
   其他决定项或聚合规则无法确认时记 `EVIDENCE_INSUFFICIENT`。

不得假定“Prompt 修正后模型一定会正确抽取”，也不得把本步骤描述成回归验证。直接根因是能够通过
上述确定性推导改变最终错误结论的问题；不能改变最终结论的问题只记录为伴随异常。

图片能够清晰支持该项事实且不与更高优先级来源冲突时，将观察值作为客观事实参与匹配判定；
图片缺失、模糊、裁切不全、主体无法定位或与其他来源冲突时，必须保留冲突并进入 `[B5]`，不得猜测。

#### 数量

**数量项的图片核验**：除读取“3 件装”“拍一发三”等主图文字外，还必须统计图中清晰、独立可辨识的
商品实物个数并记录歧义。只有按优先级证明同侧标题、主图文字和主图实物计数均无有效数量信息时，
才允许使用规则定义的默认数量；不得以“常规单品”或常识补成 `1`。

#### 品牌和材质映射

品牌和材质必须增加映射传递链核验：

- **材质**：先确认左右客观材质；再在本轮材质关系响应中逐项定位节点，只有同一真实 `parentId`
  下的直接 `isLeaf=true` 叶子才属于同组；然后检查该合法组是否完整写入基础提示词材质附录；最后
  检查模型 `parsedAnalysis` 是否抽取了两侧材质并实际应用该组。依次区分“关系库没有该组”
  “关系库有但提示词省略”“提示词已有但模型省略某侧材质”“模型已抽取但未应用映射”。禁止因为
  两个名称都出现在关系响应中就认定同组。
- **品牌**：先确认左右客观品牌；再检查本轮母子品牌响应是否存在该真实关系且通过当前类目过滤；
  然后检查该组是否完整写入基础提示词母子品牌表；最后检查模型是否抽取了两侧品牌并应用该关系。
  依次区分“关系库没有该组”“关系库有但提示词省略”“提示词已有但模型省略某侧品牌”“模型已
  抽取但未应用映射”。同集团但不在本轮真实关系中的品牌不得凭常识互认。

任何提示词修改都必须忠实实现 `expectedPriority/expectedMatch` 或通用证据约束，禁止把单条样本的
品牌、商品名、“买1送1”“两支”等具体事实硬编码进通用 Prompt。

## B5 证据充分性门禁

先检查标签：人标或模型任一侧为“不确定”时，输出“待核验（标签不确定）”，记录不确定侧并停止
分类和提示词修改。标签明确后，再判断能否确认商品事实、模型实际处理、可信规则、
`expectedPriority/expectedMatch` 与提示词的精确差异及错误因果链。缺少决定性图片、属性、类目、规则
或结构化模型过程时，输出“待核验（证据不足：暂时无法归因）”并停止分类；不得为了形成修改建议
把未知项写成事实。
验证时规则快照不可用且判断依赖规则版本变化时，同样停止并建议重跑。“证据不足”不是一级分类。

## B6 选择唯一一级分类

完整执行 [badcase-attribution-policy.md](badcase-attribution-policy.md) 的 SKU/SPU 口径门禁和明确结论条件，
为每条证据充分且标签明确的样本填写“唯一结论、口径与根因证据、具体原因、处理动作、回归方案、
是否修改提示词”。标签不确定或证据不足时记录对应待核验结果并停止，不得强行归因。整任务必须
先逐条执行门禁和分类，再统计各明确结论、标签不确定和证据不足数量。

只使用 [badcase-attribution-policy.md](badcase-attribution-policy.md) 的“一级分类、归因与处理唯一映射”表：

1. 用 `humanLabel/modelLabel` 和 SPU 核验结论命中唯一一级分类行；
2. 再用直接根因选择该分类下唯一 `caseAttribution`；
3. 直接复制同一行的“是否修改 Prompt”和“处理方式”，禁止在 B7 重新判断或覆盖。

## B7 执行处理动作

- `caseAttribution` 不为 `PROMPT_DEFECT`：固定“是否修改提示词=否”，不生成候选正文，不调用
  `[S3]`，不输出 Diff、校验成功或确认创建话术。
- `caseAttribution=PROMPT_DEFECT`：固定“是否修改提示词=是”。单条基于该条证据生成一份非空
  候选完整提示词；整任务仅在用户要求整合时，合并同一服务端续批身份下各批次已确认的缺陷，并
  生成一份候选完整提示词。服务端续批身份以首批响应及后续原样回传的 `continuationToken` 为准，
  不得使用模型维护的本地锁判断批次归属。
  随后严格执行 [shared-steps.md](shared-steps.md) `[S3]`，实际调用：

  `tool_validate_prompt_skeleton(prompt_content=<候选完整提示词>, operator=上下文.operator, conversation_id=上下文.localConversationId, base_prompt_version_id=selectedPromptVersionId, rule_group_id=上下文.ruleGroupId, agent_id=上下文.agentId)`

  只有同次响应满足
  `base_resp.resp_code=1 && data.valid=true && data.diff_record_id>0 && data.diff_content非空`，才形成
  可见提案；Diff 必须逐字符复制该响应的 `data.diff_content`。失败时按 `[S3]` 返回真实错误，禁止
  输出提案或确认话术。

## B8 输出与回归

使用入口文件唯一固定模板，不得由统一内核另造通用回复。输出中的一级分类、具体原因、是否修改
提示词、处理建议和 Diff 必须与内部分析记录一致。整任务各明确结论数量、标签不确定数量与证据不足数量
之和必须等于本批
去重明细数；只有“是否修改提示词=是”的模型判得过严或过宽样本进入候选修改合并。

确认创建草稿仍只按 [shared-steps.md](shared-steps.md) `[S5]` 执行，统一内核不得直接写入。
