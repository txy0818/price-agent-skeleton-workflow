# 单条 Badcase 分析

共享步骤见 [shared-steps.md](shared-steps.md)。

## 执行门禁

取得任务 ID、Case ID 和页面当前 Prompt ID 后，输出前必须实际完成：读取并按顺序执行 [badcase-processing-workflow.md](badcase-processing-workflow.md) `[B0]`～`[B8]`，以及其要求的共享步骤、规则和归因规范；查询指定验证结果和 CDN；从任务结果锁定基础提示词；执行 `[S2]`；逐项核对商品快照与模型 `extracted`；完成八类归因。不得只报告“已加载流程”或预告下一步。

- 缺少任务 ID 或 Case ID：只询问实际缺失的标识。`promptVersionId<=0` 时不向用户索取 ID，
  只提示先在页面左侧选择本次验证任务使用的提示词后重试。
- 非提示词缺陷或证据不足：当轮输出完整分析结果，不生成或校验提示词。
- 确认属于提示词缺陷：当轮继续生成候选完整正文、实际执行 `[S3]`，并在 `valid=true` 后输出带服务端原样 Diff 的修改提案；不得等待用户再次回复才生成提案。

## 查询与复核

1. 仅当业务上下文缺少 `ruleGroupId` 时执行 `[S1]`；已有则跳过。
2. 只从本轮业务上下文取得 `validationTaskId`、`validationCaseId` 和 `promptVersionId`；
   三者都必须大于 0。`validationTaskId` 或 `validationCaseId` 缺失时只询问缺失项；
   `promptVersionId<=0` 时按上述页面选择提示停止。不能从用户文字、历史消息、CDN 或工具返回
   猜测/替换这些 ID，也不能只凭文字或 CDN 正式归因。
3. 调用
   `tool_query_validation_result(validation_task_id=上下文.validationTaskId, label_filter=3, page={page_no:1,page_size:1}, operator=上下文.operator, validation_case_id=上下文.validationCaseId, prompt_version_id=上下文.promptVersionId)`。
   只接受 `base_resp.resp_code=1 && result=1`；“验证样本结果不存在”或“指定验证样本不是Badcase”
   直接如实返回并停止。要求响应 `data.validation_task_id`、
   `data.prompt_version.prompt_version_id` 分别与请求完全一致，`data.results` 恰好一条，且
   `data.results[0].validation_case_id` 等于请求值、`data.results[0].is_correct=0`；
   任一不符立即报告上下文/响应不一致并停止。
   只按以下精确字段读取，禁止用字段含义猜路径：
   - 任务/验证集/摘要：`data.validation_task_id`、`data.dataset_id`、
     `data.validation_summary_json`；
   - 基础提示词：`data.prompt_version.{prompt_version_id,rule_group_id,version_no,version_name,
     version_status,prompt_content}`；
   - 单条结果：`data.results[0].{validation_result_id,validation_case_id,category_id,source_item_id,
     candidate_item_id,source_title,candidate_title,human_label,model_label,is_correct,reason,
     analysis_process,radar_task_id,radar_task_url,source_item_json,candidate_item_json}`，以及 Kconf
     开启时才可能返回的 `data.results[0].raw_llm_response`；
   - 分页：`data.page.page_no/page_size/total/has_more`。
   `data.dataset_id`、`data.results[0].category_id` 必须大于 0；上下文 `ruleGroupId>0` 时要求
   `data.prompt_version.rule_group_id` 与其一致。响应没有 `agent_id`，禁止声称已由本接口校验 Agent。
   该提示词即本次基础版本，不再查询其他提示词。
   `data.prompt_version.prompt_content` 必须非空；为 `null`、空串或仅空白字符时立即停止，说明任务提示词尚未
   初始化，不能进行 Badcase 归因或修改。
   记录 `selectedPromptVersionId=data.prompt_version.prompt_version_id`，要求大于 0，
   并锁定 `writeMode=EDIT`。
4. 将 `data.results[0].category_id` 作为本条唯一可信类目 ID，按 `[S2]` 调用
   `radar_query_price_rule(categoryId=data.results[0].category_id)` 并加载该规则实际需要的映射；
   禁止从标题、商品 JSON、提示词或模型输出猜类目。按 `human_label`、`model_label` 和
   `analysis_process` 中的逐项抽取结果定位嫌疑项；`raw_llm_response` 非空时仅补充核对其
   顶层 `result/reason/confidence/key_diff_point` 及解析前原文。依照
   [rule-loading-policy.md](rule-loading-policy.md) 的「Badcase 方向」定位嫌疑项。另调用
   `tool_query_cdn_report(validation_task_id=上下文.validationTaskId, prompt_version_id=上下文.promptVersionId, operator=上下文.operator)`。
   `base_resp.resp_code=1 && result=1` 时只读取 `data.report_cdn_url` 和
   `data.report_summary_json`；“验证报告不存在”视为无 CDN，继续分析并使用无链接输出分支；
   其他失败如实报告并停止。CDN 摘要不能替代逐条验证结果。
5. 先用 `source_item_json` 与 `candidate_item_json` 核对商品事实，再用
   `analysis_process` 各比价项的
   `left`/`right` 的 `value`、`source` 看模型实际抽到了什么，两边对比得出结论；
   并结合 `source_title`、`candidate_title`、`reason`、规则、映射和任务提示词复核。
   两个 `*_item_json` 是 `{"detail":商详,"imageUrl":主图链接}` 结构，内容来自验证集导入时的
   快照，不是实时商品数据；导入 Excel 未填商详与图片时该字段为空。据此区分：
   - 快照里有该属性而 `value` 为空或为 `缺失` → **模型没抽到**；提示词抽取要求缺失/错误时归为
     提示词规则缺陷，提示词要求明确时归为模型抽取或推理异常。
   - 快照里确实没有该属性，且能确认是映射或导入字段错误 → 归为映射或样本数据问题；无法确认
     缺失原因且该属性决定结论时归为证据不足。两种情况都不得因此改骨架。
   - 快照与模型抽取一致但客观事实不支持人工标签 → 疑似人工标签错误。
   - `*_item_json` 为空时只能基于标题与 `extracted` 判断；需要商详或主图才能定论时
     归为证据不足，不得拿模型自己的抽取结果去否定人工标签。
   - `analysis_process` 为空时不能单独归为任何确定问题：`raw_llm_response` 非空且能确认原始结果
     完整、只在结构化解析或落库后丢失时，归为解析或系统链路异常；已由其他结构化证据确认样本字段
     或快照异常时归为映射或样本数据问题；否则归为证据不足。不据原始自由文本修改提示词。
   `is_correct=0` 不证明人工标签正确或提示词错误。
   接口不返回可用的规则快照，无法确认本次验证跑的是哪一版规则；因此发现骨架写法与当前
   本轮 `radar_query_price_rule` 返回的类目规则不一致时，必须同时考虑“规则在本次验证后已变更”这一可能，
   在建议中说明并提示用当前基础提示词重跑验证确认，不得直接断定为骨架缺陷。
6. 将本条查询和复核结果整理成统一内部分析记录，按统一流程 `[B4]`～`[B6]` 建立证据、
   执行充分性门禁并选择唯一一级归因。
   若基础提示词已明确要求某条件下 `match=false`，而 `analysis_process` 对同一事实给出
   `match=true`，必须归为“模型未遵循规则”，不得改写成“提示词过宽/过严”。例如提示词明确
   “版本或代际不同视为不匹配”，模型却将“第二代”解释为营销表述并给 `match=true`，属于模型
   未遵循规则，不是提示词规则缺陷。
   仅提示词缺陷可修改，禁止为单个商品写过拟合规则。
7. 严格执行统一流程 `[B7]`：只有一级归因为“提示词规则缺陷”才生成候选完整提示词并实际执行
   `[S3]`；其他七类不生成候选正文、不调用校验工具。随后按 `[B8]` 选择下方唯一固定模板。

## 输出

按 `[S4]` 展示。以下两个模板不是示例，而是本流程仅允许的可见回复协议；必须先按归因结果选择
唯一分支，再逐项、按序套用。回复必须以 `## Badcase 分析` 为第一行，模板前后不得添加任何
文字；CDN 必须是最后一行。禁止把两个模板合并、输出条件占位符或自行组织格式。

### 非提示词缺陷模板

归为提示词规则缺陷以外的其他七类时，只能使用本模板；
不执行 `[S3]`，不输出 Diff，不询问修改方向或提供下一步选项。

````markdown
## Badcase 分析
- Agent：`<名称；查不到时写ID>`
- 任务 / Badcase / 验证集：`<task_id> / <case_id> / <dataset_id>`
- 任务提示词：`<名称>`（ID：`<ID>`）
### 样本与证据
- 左 / 右商品：<source_title> / <candidate_title>
- 人工标签 / 模型结论：<标签>
- 证据来源：<analysis_process；raw_llm_response 非空时写“analysis_process + raw_llm_response”>；商品快照：<有/无>
| 比价项 | 左侧快照 | 左侧抽取（value / source） | 右侧快照 | 右侧抽取（value / source） | match | 复核 |
|---|---|---|---|---|---|---|
| <比价项> | <快照事实；无快照写“—”> | <value / source> | <快照事实；无快照写“—”> | <value / source> | <true/false> | <结论> |
### 结论与建议
- 类型：<模型未遵循规则 / 模型抽取或推理异常 / 业务规则问题 / 疑似人工标签错误 / 映射或样本数据问题 / 解析或系统链路异常 / 证据不足之一>
- 是否修改提示词：否
- 依据：<关键证据>
- 建议与回归：不建议修改提示词；<该类型对应的复核或处理建议>
该问题不属于提示词规则缺陷，不建议修改。
验证集报告：[查看 CDN 报告](<真实链接>)
````

### 提示词缺陷模板

仅归为“提示词规则缺陷”且已完成真实 `[S3]` 时使用：

````markdown
## Badcase 分析
- Agent：`<名称；查不到时写ID>`
- 任务 / Badcase / 验证集：`<task_id> / <case_id> / <dataset_id>`
- 任务提示词：`<名称>`（ID：`<ID>`）
- 修改建议 ID（diff_id）：`<原样引用 S3 返回的非零 data.diff_record_id>`
- 状态：尚未保存
### 样本与证据
- 左 / 右商品：<source_title> / <candidate_title>
- 人工标签 / 模型结论：<标签>
- 证据来源：<analysis_process；raw_llm_response 非空时写“analysis_process + raw_llm_response”>；商品快照：<有/无>
| 比价项 | 左侧快照 | 左侧抽取（value / source） | 右侧快照 | 右侧抽取（value / source） | match | 复核 |
|---|---|---|---|---|---|---|
| <比价项> | <快照事实；无快照写“—”> | <value / source> | <快照事实；无快照写“—”> | <value / source> | <true/false> | <结论> |
### 结论与建议
- 类型：提示词规则缺陷
- 是否修改提示词：是
- 依据：<关键证据>
- 建议与回归：<修改方案、适用范围和回归项>
### Diff
```diff
<逐字符复制同次 tool_validate_prompt_skeleton 返回的 data.diff_content>
```
完整提示词已生成并通过格式校验，未在此展开。需查看请回复「展开完整提示词」。
以上仅为修改提案。确认后创建新提示词草稿，不覆盖基础版本。
确认无误请回复：**确认创建提示词草稿**。
验证集报告：[查看 CDN 报告](<真实链接>)
````

无链接时将所选模板最后一行替换为：`验证集报告：当前验证任务未返回 CDN 报告链接`。

### 输出硬校验

发送前逐项检查；任一项不满足就在内部重组回复，不向用户展示错误版本：

1. 第一行必须严格等于 `## Badcase 分析`；禁止使用 `【Badcase 单条分析结果】`、其他标题、
   标题前文字或模板后补充。
2. 必须保留所选模板的全部字段、三级标题、表头和顺序；禁止增加“当前验证任务整体情况”、
   耗时、Token、“如果需要下一步操作”或修改方向选项。
3. 输出前必须存在本轮真实 `tool_query_validation_result`、目标类目的规则/必要映射查询以及
   `tool_query_cdn_report` 调用记录；缺任一记录只返回该步骤真实错误，禁止输出分析结论。
4. 类型只能是八类之一，并与“是否修改提示词”一致：只有“提示词规则缺陷”写“是”并使用缺陷
   模板；其他七类一律写“否”并使用非缺陷模板。
5. 归因前必须逐字对照 `data.prompt_version.prompt_content` 的目标规则与
   `data.results[0].analysis_process`。提示词已明确规则而模型输出相反时，只能归为“模型未遵循
   规则”，禁止声称提示词宽松、过严或缺少该规则。
6. 非缺陷模板不得出现 Diff、S3 成功话术、确认话术或要求用户选择修改方向；缺陷模板必须展示
   同次 `tool_validate_prompt_skeleton` 的非零 `data.diff_record_id`，并原样输出同次
   `data.diff_content`，否则禁止输出缺陷模板。
7. CDN 行必须是最后一行；有链接只用真实 `data.report_cdn_url`，无链接使用固定无链接文本。

## 确认后

只有提示词缺陷且用户紧邻确认，才执行 `[S5]` 的 EDIT 路径。
本轮不再输出 CDN。
