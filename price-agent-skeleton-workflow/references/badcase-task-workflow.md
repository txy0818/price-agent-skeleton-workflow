# 验证任务 Badcase 分析

共享步骤见 [shared-steps.md](shared-steps.md)。

## 查询与复核

1. 仅当业务上下文缺少 `ruleGroupId` 时执行 `[S1]`；已有则跳过。
2. 必须取得验证任务 ID：用户本轮明确提供时优先，否则使用上下文 `validationTaskId`；
   两者都没有时要求补充。CDN 不能代替任务 ID。
3. 调用
   `tool_query_validation_result(validation_task_id=<任务ID>, label_filter=3, page_size=10, operator=当前业务上下文.operator)`。
   从返回的 `PromptVersionBaseInfo` 取得任务实际使用的 `prompt_version_id`、
   `version_name`、`version_status` 和完整 `prompt_content`，同时取得验证集、样本标题、
   `source_item_json`、`candidate_item_json`、标签、`reason` 和 `raw_llm_response`；校验 Agent。
   该提示词即本次基础版本，不再查询其他提示词。
   `prompt_content` 必须非空；为 `null`、空串或仅空白字符时立即停止，说明任务提示词尚未
   初始化，不能进行 Badcase 归因、分批分析或整合修改。
   记录 `selectedPromptVersionId=PromptVersionBaseInfo.prompt_version_id`，要求大于 0，
   并锁定 `writeMode=EDIT`。
   一次最多分析 10 条，不得自行调大 `page_size`：每条会带回两组商品快照和整段
   `raw_llm_response`，单批 20 条会挤占上下文并导致前批阶段结论被压缩丢失。还有更多时报告剩余数。
   用户要求继续时查询下一页；
   必须保持同一任务 ID 和 `PromptVersionBaseInfo.prompt_version_id`，并按
   `validation_case_id` 去重。
4. 按 `[S2]` 对本批样本类目去重后分别加载规则与必要映射，禁止跨类目混用。逐条按
   `human_label`、`model_label` 和 `raw_llm_response`（为空时用 `analysis_process`）中的
   `extracted`，依照 [rule-loading-policy.md](rule-loading-policy.md) 的「Badcase 方向」定位
   嫌疑项。另调用
   `tool_query_cdn_report(validation_task_id=<任务ID>, operator=当前业务上下文.operator)`
   取得报告链接。
5. 逐条先用 `source_item_json` 与 `candidate_item_json` 核对商品事实，再用
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
   - `raw_llm_response` 与 `analysis_process` 同时为空时归为数据问题，该条不做归因。
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
   仅提示词缺陷可修改，禁止针对个别商品写过拟合规则。只有单条命中的规则问题按证据不足
   处理，待后续批次出现同类样本再确认。

## 分批分析

每批结果只是当前会话中的阶段分析，不写入提示词表或 Diff 表，也不调用
`tool_edit_prompt_skeleton`。后续批次必须合并当前会话中同一任务、同一基础提示词的既有
阶段结论，不得把每批建议分别创建为草稿。非提示词缺陷不输出空 Diff；CDN 必须是最后一行。

````markdown
## 验证任务 Badcase 阶段分析
- Agent：`<名称；查不到时写ID>`
- 任务 / 验证集：`<task_id> / <dataset_id>`
- 基础提示词：`<名称>`（ID：`<PromptVersionBaseInfo.prompt_version_id>`）
- 批次：第 `<page>` 批
- Badcase：共 `<total>` 条，本批分析 `<analyzed>` 条，累计已分析 `<累计去重数>` 条
### 分类统计
| 类型 | 数量 | 处理 |
|---|---:|---|
| 提示词规则缺陷 | <数> | 修改并回归 |
| 模型未遵循规则 | <数> | 观察执行 |
| 疑似人工标签错误 | <数> | 复核标签 |
| 映射或数据问题 | <数> | 修复数据 |
| 证据不足 | <数> | 补充证据 |
### 明细
| Badcase ID | 人工/模型标签 | 类型 | 关键证据（快照事实 vs extracted） | 建议 |
|---|---|---|---|---|
| `<case_id>` | <标签> | <类型> | <比价项：快照事实 / 左右 value / match> | <建议> |
### 本批候选修改
- 缺陷与样本：<仅确认的提示词缺陷及 case ID>
- 候选建议：<待最终整合的规则修改；无缺陷写“本批无提示词修改建议”>
- 不写入提示词：<标签、数据、映射或执行问题>
- 与前批关系：<新增/重复/补充/冲突；首批写“首批”>
<超过10：本次已分析10条，剩余 <remaining> 条；可回复“继续分析”。>
<需要整合时：可回复“整合修改建议给我一个草稿”。>
验证集报告：[查看 CDN 报告](<真实链接>)
````

无链接时最后一行写：`验证集报告：当前验证任务未返回 CDN 报告链接`。

## 整合修改建议

用户要求“整合修改建议给我一个草稿”时：

1. 只合并当前会话中任务 ID、验证集 ID 和基础提示词 ID 均一致的阶段分析；按
   `validation_case_id` 去重。整合前自检各批阶段结论是否完整可见；任一批次不可见时，
   必须用同一任务 ID 与同一 `PromptVersionBaseInfo.prompt_version_id` 重新查询该页，
   不得跳过也不得凭记忆补写。
2. 合并指向同一规则的重复建议；结合商品快照证据、`extracted` 抽取结果、比价规则和映射
   处理冲突。疑似人工标签错误、模型执行、映射或数据问题以及证据不足不得写入提示词。
3. 基于 `PromptVersionBaseInfo.prompt_content` 执行 `[S3]` 生成并校验一份合并后的完整
   提示词，最终 Diff 取 `[S3]` 返回的 `data.diff_content`，不自行书写。说明覆盖的批次、
   Badcase ID、尚未分析的数量和回归风险。
4. 按 `[S4]` 展示最终 Diff，声明“尚未保存”，询问用户是否确认创建一个新提示词草稿；
   此时不得调用写入 MCP。

## 确认创建后

只有最终整合结果存在提示词缺陷，且用户在看到最终 Diff 后明确确认，才执行 `[S5]` 的
`EDIT` 路径，`prompt_version_id=selectedPromptVersionId`。
仅调用一次并创建一个新草稿，不为各批次分别创建草稿。本轮不再输出 CDN。
