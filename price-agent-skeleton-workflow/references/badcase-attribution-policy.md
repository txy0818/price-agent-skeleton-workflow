# Badcase Prompt 缺失判定规范

Badcase 分析只回答一个问题：验证任务实际使用的 Prompt 是否存在下方五类静态缺失或未同步。这里的
“缺失”包括“正确实现缺失并被错误实现替代”，因此规则内容错误也命中对应类型。禁止分析或输出
模型错误、人工标签错误、SKU/SPU 口径、图片识别、来源执行、抽取、逐项匹配、最终聚合及其他原因。

## 唯一检查清单

| `missingType` | 检查内容 | 证据来源 |
|---|---|---|
| `STEP3_MISSING` | `Step3` 缺失或未完整写入当前真实 `compareLogic` 对综合判定的例外；例如真实为“货号匹配逻辑”时，不能只保留“任一 match=false 即 different” | 任务 `promptContent` + 当前真实类目规则 + `enums.md` |
| `COMPARE_ITEM_MISSING` | 生效比价项章节缺失，或该项的来源优先级、匹配逻辑、特殊规则未过滤、转换错误或未完整转换；包括规则修改后 Prompt 未同步 | 任务 `promptContent` + 当前真实类目规则 + 转换指南 |
| `BRAND_MAPPING_MISSING` | 当前规则包含品牌且本轮过滤后存在合法母子品牌组，但 Prompt 未显式写入或组成员不完整 | 任务 `promptContent` + 本轮品牌关系响应 |
| `MATERIAL_MAPPING_MISSING` | 当前规则包含材质，但 Prompt 缺少材质引用、附录、固定空结果或本轮合法材质组 | 任务 `promptContent` + 本轮材质关系响应 |
| `OUTPUT_FORMAT_MISSING` | 固定章节、JSON 示例、字段或字段规则缺失/不符合规范，或 `extracted` 未按全部生效比价项生成 | 任务 `promptContent` + `skeleton-format.md` |

只允许检查这五类。五类内部的应有内容被写错、截断或被错误内容替代时，视为正确实现缺失；五类之外
的问题不分析。

## 结果枚举

| `promptMissingFinding` | 充分条件 | 是否修改 Prompt |
|---|---|---|
| `PROMPT_MISSING` | 可信真实规则或本轮合法映射明确要求某内容，基础 Prompt 对应内容缺失或只实现了一部分 | 是 |
| `NO_PROMPT_MISSING` | 已完成全部必查项对照，基础 Prompt 完整实现真实规则及必要合法映射 | 否 |
| `PROMPT_MISSING_UNCONFIRMED` | 规则、Prompt、类目或必要映射缺失/失败，无法完成完整对照 | 否，补齐证据后重跑 |

每条样本只能选择一个结果。`PROMPT_MISSING` 不要求证明缺失导致本条模型误判；本流程检查的是 Prompt
同步完整性，不做 Badcase 因果归因。不得使用 `PROMPT_DEFECT`、`MODEL_ERROR`、
`HUMAN_LABEL_SUSPECTED_ERROR`、`SKU_SPU_SCOPE_DIFFERENCE` 等旧归因枚举。

## “缺失”的边界

以下属于 Prompt 缺失：

- `expectedPriority` 中存在合法来源，但 Prompt 对应比价项遗漏该来源；
- Prompt 只保留部分来源链，特别是冒号后第二段来源链被截断；
- `expectedMatch`、特殊规则或 `compareLogic` 转换结果未写入或仅写入部分条件；
- 同一规则在 Prompt 不同章节实现不完整，导致必要条件实际缺失；
- 本轮关系接口返回且通过类目过滤的必要品牌/材质组未完整写入对应附录。
- 货号等真实 `compareLogic` 要求的 Step3 例外未同步；
- `skeleton-format.md` 规定的固定章节、完整 JSON 示例或固定字段规则未完整写入。

以下不在本流程分析范围：五类之外的额外规则、文本冲突、措辞歧义、模型未执行已有规则，以及任何商品
事实或标签判断。五类内部的来源顺序、规则内容或格式与真实规范不一致，仍按“正确实现缺失/未同步”记录。

## 输出要求

结论只展示缺失对照：比价项或映射、真实规则/关系依据、Prompt 当前原文或位置、缺失内容、建议补入内容。
还必须简要记录触发该检查的 Badcase 结构化信号，例如“`modelLabel=2` 且数量 `match=false`”；不得展示
商品事实、SKU/SPU 口径、模型 `analysisProcess` 的左右 value/source 或非 Prompt 问题推测。
