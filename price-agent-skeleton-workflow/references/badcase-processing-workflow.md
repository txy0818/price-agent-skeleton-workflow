# Badcase Prompt 缺失检查流程

本流程只检查验证任务基础 Prompt 的五类静态缺失：Step3、比价项规则、母子品牌映射、材质映射、输出
格式。入口 workflow 只负责锁定 `SINGLE/TASK`
模式和固定输出；共同查询、对照与处理统一执行 `[B0]`～`[B7]`。

完整读取 [badcase-attribution-policy.md](badcase-attribution-policy.md)、
[rule-loading-policy.md](rule-loading-policy.md)、[rule-transformation-guide.md](rule-transformation-guide.md)、
[enums.md](enums.md) 和 [shared-steps.md](shared-steps.md)。

## B0 入口门禁

- `SINGLE`：业务上下文必须满足 `validationTaskId>0 && validationCaseId>0 && promptVersionId>0`。
- `TASK`：业务上下文必须满足 `validationTaskId>0 && validationCaseId<=0 && promptVersionId>0`。
- 禁止从用户文字提取或覆盖任何 ID；不满足时按入口固定提示停止。

## B1 查询验证结果并锁定 Prompt

调用合同、成功门禁、字段路径、TASK 首批/续批 Token 规则完全沿用入口 workflow 已定义的
`tool_query_validation_result` 约束。只接受 `isCorrect=0` 的结果。

锁定响应中的：`validationTaskId`、`datasetId`、每条 `validationCaseId/categoryId`，以及
`promptVersion.{promptVersionId,ruleGroupId,versionName,promptContent}`。`promptContent` 必须非空，响应
任务、Prompt、规则组必须与业务上下文一致。禁止查询或替换成其他 Prompt 版本。

解析并只读使用 `humanLabel`、`modelLabel`、`reason` 和 `analysisProcess` 生成 Badcase 检查入口；它们
只能说明“这条 Badcase 应优先检查哪里”，不能证明 Prompt 缺失。商品 JSON 和图片不读取。

## B2 从 Badcase 建立检查入口

先解析当前结果的结构化 Badcase 字段：

- `humanLabel/modelLabel`：只确定错误方向，不判断人工或模型谁错；
- `analysisProcess`：解析为 `Map<比价项,{left,right,match,reason}>`，只读取比价项键和 `match`；
- 顶层 `reason`：只辅助定位模型声称依赖的比价项，不作为 Prompt 缺失证据。

按下表建立 `badcaseSignals`，并在最终缺失对照中记录“由哪个 Badcase 字段触发”：

| Badcase 信号 | 优先引出的检查 | 原因 |
|---|---|---|
| `humanLabel=1 && modelLabel=2` | `analysisProcess` 中全部 `match=false` 比价项 → `COMPARE_ITEM_MISSING`；涉及品牌/材质时再检查对应映射 | 这些是模型实际用于否决同款的项目 |
| `humanLabel=2 && modelLabel=1` | 当前类目全部有效比价项 → `COMPARE_ITEM_MISSING` | 模型可能因 Prompt 漏项而完全没有生成对应分析键，不能只看已有 `analysisProcess` |
| 按 Prompt 当前通用 Step3 聚合 `analysisProcess.match` 得出的方向与 `modelLabel` 不一致 | `STEP3_MISSING` | 顶层结论与逐项结果出现综合判定异常信号 |
| 真实规则存在“货号匹配逻辑”，且 Badcase 涉及货号否决或放行 | `STEP3_MISSING` | 需要核验 Prompt 是否遗漏货号不构成否决的 Step3 例外 |
| 品牌项出现不一致、交集、中英文或母子品牌语义，或品牌项未出现在模型分析中 | `BRAND_MAPPING_MISSING` | 需要核验合法母子品牌映射是否显式写入 Prompt |
| 材质项出现同义材质/同组材质语义，或材质项未出现在模型分析中 | `MATERIAL_MAPPING_MISSING` | 需要核验合法材质组是否显式写入 Prompt |
| `analysisProcess` 解析失败、结构缺键、生效比价项未输出、顶层结果字段异常 | `OUTPUT_FORMAT_MISSING` | 形成输出协议可能缺失的信号，但仍须以 Prompt 对格式规范的静态对照确认 |

这里的“未出现在模型分析中”只有在真实类目规则确认该比价项生效后才成立，不得根据名称猜测。

## B2.1 扩展为完整核验范围

按本批结果中的 `categoryId` 去重。对每个真实类目检查该类目规则中的**全部有效比价项**，不得按标签方向、
`match=false` 项或模型分析过程缩小最终确认范围。`badcaseSignals` 只决定检查优先顺序；五类静态检查仍
全部执行，原因是 Prompt 缺失可能恰好导致模型没有生成对应分析项。

`tool_query_validation_result` 只负责提供任务实际 `promptContent`、样本真实 `categoryId` 和任务身份：

- 它可以直接支持 Prompt 正文中 Step3、比价项章节、映射章节和输出格式“是否写了”的检查；
- 它不能单独证明比价项正确规则或映射全集；比价项必须查询当前真实类目规则，品牌/材质必须查询关系接口；
- `analysisProcess` 只能产生检查信号，不能直接判定这五类缺失。

## B3 查询真实规则和必要映射

逐类目按 `[S2]` 查询真实规则，并按规则转换指南把每个有效 `compareItem` 转成：

- `expectedPriority`：稳定过滤、归一、去重后的完整合法来源链；
- `expectedMatch`：匹配逻辑、特殊规则和 `compareLogic` 的完整实现要求。

只有真实规则确实包含“材质”时查询 `radar_query_material_relation`；只有真实规则确实包含“品牌”时查询
`radar_query_brand_relation`。仅保留本轮接口响应且通过类目过滤的合法映射；材质组仍须满足同一真实
`parentId` 下直接 `isLeaf=true` 叶子规则。接口失败或必要证据不完整时，对受影响类目记录
`PROMPT_MISSING_UNCONFIRMED`，不得用常识或历史关系补齐。

同时按任务和 Prompt ID 调用 `tool_query_cdn_report`；链接仅用于最终展示，不参与判断。

## B4 按固定五类建立 Prompt 缺失对照表

逐类目、逐有效比价项执行，品牌/材质映射另列附录对照：

| 对照字段 | 必须记录的内容 |
|---|---|
| 缺失类型 | 五个 `missingType` 之一 |
| 类目与对象 | `categoryId`、比价项/映射/Step3/输出格式 |
| Badcase 触发证据 | `humanLabel/modelLabel`、`analysisProcess` 键与 `match`、顶层 `reason` 中实际引出本项检查的字段；全量兜底写“类目全部有效项检查” |
| 真实依据 | 原始 `infoSource/compareLogic/特殊规则` 或本轮关系响应中的合法组 |
| 应有实现 | 转换后的完整 `expectedPriority/expectedMatch` 或完整映射组 |
| Prompt 实现 | `promptContent` 对应章节的原文和位置；找不到写“未实现” |
| 缺失内容 | 只列“应有实现”中 Prompt 未覆盖的内容；无缺失写“无” |
| 逐项结果 | `PROMPT_MISSING / NO_PROMPT_MISSING / PROMPT_MISSING_UNCONFIRMED` |

严格按以下顺序检查，五类全部执行，不得发现一类后提前停止：

1. **`STEP3_MISSING`**：先检查 Prompt 是否完整包含 Step0～Step3 及通用综合判定；再逐项扫描真实
   `compareLogic`。只有真实存在“货号匹配逻辑”时，才要求 Step3 写入“货号不一致不构成否决；除货号
   外其他生效项均匹配则判 same”的例外。缺少即记缺失；不得因比价项名称叫“货号”自行要求例外。
2. **`COMPARE_ITEM_MISSING`**：逐个真实有效 `compareItem` 检查适用范围、独立章节和输出
   `extracted` key；再将完整原始 `infoSource` 过滤转换为 `expectedPriority`，将 `compareLogic` 与特殊规则
   转为 `expectedMatch`，逐条检查 Prompt。来源链截断、合法来源未过滤出来、来源顺序不一致、匹配条件
   漏写或与真实规则不一致，均记为正确实现缺失/未同步。
3. **`BRAND_MAPPING_MISSING`**：仅当前规则精确包含“品牌”时查询品牌关系。过滤后非空，Prompt 必须
   显式包含母子品牌表且每个保留组完整；为空则不要求编造映射。未调用或接口失败不得判“无缺失”。
4. **`MATERIAL_MAPPING_MISSING`**：仅当前规则精确包含“材质”时查询材质关系。Prompt 必须同时包含
   材质项中的附录引用和材质附录；过滤后有合法组须显式写组，0 组须写格式规范规定的固定空结果。
5. **`OUTPUT_FORMAT_MISSING`**：完整读取 `skeleton-format.md`，检查固定章节顺序、只输出 JSON 的约束、
   可解析 JSON 示例、`result/reason/confidence/extracted/key_diff_point` 及固定字段规则；任一必需块缺失即记。

某类所需真实依据或 Prompt 无法取得时，该类记 `PROMPT_MISSING_UNCONFIRMED`；有明确应有内容且 Prompt
未覆盖时记 `PROMPT_MISSING`；完成该类全部检查且均覆盖时记 `NO_PROMPT_MISSING`。

不得写成“Badcase 的 `match/source/reason` 证明 Prompt 缺失”。完整证据链必须是：

`Badcase 字段产生检查信号 → categoryId 定位真实规则 → 规则/关系/格式规范得到应有实现 → 任务 promptContent 显示缺失或未同步`。

例如数量真实来源转换后为 `商品标题>主图`，Prompt 只写 `商品标题`：记录“缺失来源：主图”，结果为
`PROMPT_MISSING`。无需读图、判断本条商品主图是否有数量，也无需证明模型因此误判。

品牌示例中，真实优先级若转换为 `商品属性>商品标题`，Prompt 未保留 `商品属性`，归入
`COMPARE_ITEM_MISSING`；品牌匹配条件与真实 `expectedMatch` 不一致，也归入同类并列出“真实应有内容 /
Prompt 当前错误内容”。母子品牌表未显式写入另归 `BRAND_MAPPING_MISSING`，不得与品牌比价项缺失合并。

## B5 汇总单条结果

- 任一必查项为 `PROMPT_MISSING` → 整条 `promptMissingFinding=PROMPT_MISSING`；合并全部缺失项。
- 无缺失，但任一必查项为 `PROMPT_MISSING_UNCONFIRMED` → 整条同枚举。
- 全部必查项均为 `NO_PROMPT_MISSING` → 整条同枚举。

禁止继续检查模型执行、商品事实、标签正确性、SKU/SPU 口径或误判因果。`NO_PROMPT_MISSING` 只表示
未发现 Prompt 内容缺失，不表示模型或人工标签正确。

## B6 选择处理动作

- `PROMPT_MISSING`：是否修改提示词=是。单条生成补齐全部已确认缺失后的候选完整 Prompt 并执行
  `[S3]`；TASK 阶段只记录候选建议，用户要求整合时合并同一续批身份下的已确认缺失，再生成一份候选
  完整 Prompt 并执行一次 `[S3]`。
- `NO_PROMPT_MISSING`：是否修改提示词=否，不生成候选，不分析其他原因。
- `PROMPT_MISSING_UNCONFIRMED`：是否修改提示词=否，列明缺失的规则、Prompt 或映射证据，补齐后重跑。

候选只能忠实补入真实规则或合法映射中缺失的通用内容，禁止写入单条商品事实、品牌常识或模型修复策略。

## B7 输出

使用 SINGLE/TASK 入口定义的固定模板。输出内容只能来自 B4～B6：Prompt 缺失状态、规则对照、缺失项、
是否修改和补齐建议。不得恢复旧归因字段或输出任何非 Prompt 缺失分析。
