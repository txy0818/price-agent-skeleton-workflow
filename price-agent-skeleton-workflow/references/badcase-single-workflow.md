# 单条 Badcase 分析

共享步骤见 [shared-steps.md](shared-steps.md)。

> **不得直接回复**：进入本流程后必须完整读取本文件及其要求的策略，按任务 ID 和 Case ID 查询
> 真实验证结果、核对快照和提示词后，才可按固定模板输出；提示词缺陷提案还必须完成真实 `[S3]`。
> 用户已给出结论、截图或 CDN 也不得替代上述步骤。

## 回复前执行门禁

取得任务 ID 和 Case ID 后，输出前必须实际完成：读取 [shared-steps.md](shared-steps.md) 与 [rule-loading-policy.md](rule-loading-policy.md)；查询指定验证结果和 CDN；从任务结果锁定基础提示词；执行 `[S2]`；逐项核对商品快照与模型 `extracted`；完成五类归因。不得只报告“已加载流程”或预告下一步。

- 缺少任务 ID 或 Case ID：只询问实际缺失的标识。
- 非提示词缺陷或证据不足：当轮输出完整分析结果，不生成或校验提示词。
- 确认属于提示词缺陷：当轮继续生成候选完整正文、实际执行 `[S3]`，并在 `valid=true` 后输出带服务端原样 Diff 的修改提案；不得等待用户再次回复才生成提案。

## 查询与复核

1. 仅当业务上下文缺少 `ruleGroupId` 时执行 `[S1]`；已有则跳过。
2. 必须取得验证任务 ID：用户本轮明确提供时优先，否则使用上下文 `validationTaskId`；
   两者都没有时要求补充。同时必须取得 `validation_case_id`，不能只凭文字或 CDN 正式归因。
3. 调用
   `tool_query_validation_result(validation_task_id=<任务ID>, label_filter=3, validation_case_id=<Badcase ID>, page_size=1, operator=当前业务上下文.operator)`。
   从返回的 `PromptVersionBaseInfo` 取得任务实际使用的 `prompt_version_id`、
   `version_name`、`version_status` 和完整 `prompt_content`，同时取得验证集、样本标题、
   `source_item_json`、`candidate_item_json`、人工/模型标签、`reason` 和 `raw_llm_response`；
   校验 Agent 及页面非零 `datasetId`。该提示词即本次基础版本，不再查询其他提示词。
   `prompt_content` 必须非空；为 `null`、空串或仅空白字符时立即停止，说明任务提示词尚未
   初始化，不能进行 Badcase 归因或修改。
   记录 `selectedPromptVersionId=PromptVersionBaseInfo.prompt_version_id`，要求大于 0，
   并锁定 `writeMode=EDIT`。
4. 按 `[S2]` 使用本条样本类目加载规则与必要映射。按 `human_label`、`model_label` 和
   `raw_llm_response`（为空时用 `analysis_process`）中的 `extracted`，依照
   [rule-loading-policy.md](rule-loading-policy.md) 的「Badcase 方向」定位嫌疑项。另调用
   `tool_query_cdn_report(validation_task_id=<任务ID>, operator=当前业务上下文.operator)`
   取得报告链接。
5. 先用 `source_item_json` 与 `candidate_item_json` 核对商品事实，再用
   `raw_llm_response`（为空时退回 `analysis_process`）中 `extracted` 各比价项的
   `left`/`right` 的 `value`、`source` 看模型实际抽到了什么，两边对比得出结论；
   并结合 `source_title`、`candidate_title`、`reason`、规则、映射和任务提示词复核。
   两个 `*_item_json` 是 `{"detail":商详,"imageUrl":主图链接}` 结构，内容来自验证集导入时的
   快照，不是实时商品数据；导入 Excel 未填商详与图片时该字段为空。据此区分：
   - 快照里有该属性而 `value` 为空或为 `缺失` → **模型没抽到**，属于抽取或骨架问题。
   - 快照里确实没有该属性 → 归为映射或数据问题，不得因此改骨架。
   - 快照与模型抽取一致但客观事实不支持人工标签 → 疑似人工标签错误。
   - `*_item_json` 为空时只能基于标题与 `extracted` 判断；需要商详或主图才能定论时
     归为证据不足，不得拿模型自己的抽取结果去否定人工标签。
   - `raw_llm_response` 与 `analysis_process` 同时为空时归为数据问题，本条不做归因。
   `is_correct=0` 不证明人工标签正确或提示词错误。
   接口不返回可用的规则快照，无法确认本次验证跑的是哪一版规则；因此发现骨架写法与当前
   本轮 `radar_query_price_rule` 返回的类目规则不一致时，必须同时考虑“规则在本次验证后已变更”这一可能，
   在建议中说明并提示用当前基础提示词重跑验证确认，不得直接断定为骨架缺陷。
6. 归为提示词规则缺陷、模型未遵循规则、疑似人工标签错误、映射或数据问题、证据不足。
   仅当同时成立时才能归为提示词规则缺陷：已从 `extracted` 定位到具体嫌疑比价项；
   商品快照核对后客观事实支持人工标签；能在 `PromptVersionBaseInfo.prompt_content` 中
   指到具体某一条规则并说明它为何导致该结果；且该规则与当前类目规则不一致。
   快照缺失而无法核对客观事实时归为证据不足，不得直接归为人工标签错误。
   骨架已忠实翻译规则、只是规则本身如此时属于业务规则问题，不得改提示词。
   仅提示词缺陷可修改，禁止为单个商品写过拟合规则。
7. 仅在确认属于提示词缺陷时执行 `[S3]` 生成并校验完整提示词；否则不生成、不校验。

## 输出

按 `[S4]` 展示。以下不是示例，而是本流程唯一允许的可见回复协议；必须以
`## Badcase 分析` 为第一行，逐项、按序套用，模板前后不得添加任何文字。非提示词缺陷不输出
Diff；CDN 必须是最后一行。

````markdown
## Badcase 分析
- Agent：`<名称；查不到时写ID>`
- 任务 / Badcase / 验证集：`<task_id> / <case_id> / <dataset_id>`
- 任务提示词：`<名称>`（ID：`<ID>`）
### 样本与证据
- 左 / 右商品：<source_title> / <candidate_title>
- 人工标签 / 模型结论：<标签>
- 证据来源：<raw_llm_response / analysis_process（退回）>；商品快照：<有/无>
| 比价项 | 左侧快照 | 左侧抽取（value / source） | 右侧快照 | 右侧抽取（value / source） | match | 复核 |
|---|---|---|---|---|---|---|
| <比价项> | <快照事实；无快照写“—”> | <value / source> | <快照事实；无快照写“—”> | <value / source> | <true/false> | <结论> |
### 结论与建议
- 类型：<五类之一>
- 是否修改提示词：<是/否>
- 依据：<关键证据>
- 建议与回归：<方案；无缺陷写“不建议修改提示词”>
```diff
<原样引用 S3 返回的 data.diff_content>
```
<有缺陷：完整提示词已生成并通过格式校验，未在此展开。需查看请回复「展开完整提示词」。
请确认是否按以上建议创建新提示词草稿。>
<无缺陷：该问题不属于提示词规则缺陷，不建议修改。>
验证集报告：[查看 CDN 报告](<真实链接>)
````

发送前按 `[S4]` 输出协议自检并解析 `<有缺陷>`/`<无缺陷>` 分支，不得输出分支说明或其他占位符。
无链接时最后一行写：`验证集报告：当前验证任务未返回 CDN 报告链接`。

## 确认后

只有提示词缺陷且用户明确确认，才执行 `[S5]` 的 `EDIT` 路径，
`prompt_version_id=本轮最新业务变量 promptVersionId`，不得沿用提案生成时记录的
`selectedPromptVersionId`。
本轮不再输出 CDN。
