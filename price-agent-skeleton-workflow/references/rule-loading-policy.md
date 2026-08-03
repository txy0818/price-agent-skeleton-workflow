# 规则加载策略

取得当前有效的 `price_rule_json`、`special_rule_json` 和 `data_table_json`。
按当前流程的实际需要决定加载范围：新建骨架取全量，修改类流程只取相关部分。

## 为什么按需

完整规则与映射合计约 10~20k token，其中映射表通常占大头。新建骨架必须依据全部规则生成
全部比价项，无法裁剪；而修改类流程往往只涉及一到两个比价项，把全量规则读进上下文会挤占
可用长度，也让关键规则被无关内容淹没。

## 加载范围

### 比价规则

`tool_query_price_rule(rule_group_id=当前业务上下文.ruleGroupId, category_ids=<当前类目ID列表>, include_special_rule=<1或0>, compare_items=<按需比价项名称列表；全部时传空>, operator=当前业务上下文.operator)`

`compare_items` 传空表示返回全部比价项，与 `category_ids` 传空的语义一致。

| 流程 | `compare_items` | `include_special_rule` |
|---|---|---|
| 新建提示词 | 空（全量） | `1` |
| 修改提示词、用户直接提规则要求 | 目标比价项；无法定位时传空 | 涉及特殊规则时 `1`，否则 `0` |
| Badcase 分析 | 从模型输出定位的嫌疑比价项；定位不到时传空 | `1` |

### 映射表

`tool_query_rule_data_table(rule_group_id=当前业务上下文.ruleGroupId, category_ids=<当前类目ID列表>, table_types=<需要的类型；全部时传空>, operator=当前业务上下文.operator)`

当前已知取值为 `brand_mapping`（母子品牌）和 `material_mapping`（材质分组）。
**遇到未列出的映射类型时不得自行编造 `table_types` 取值**，改为传空取全部。

| 流程 | 是否调用 | `table_types` |
|---|---|---|
| 新建提示词 | 必须 | 空（全量） |
| 修改类流程，改动涉及品牌或材质判定 | 必须 | 仅相关类型 |
| 修改类流程，改动不涉及映射 | **不调用** | — |

判断依据是本轮改动的比价项：涉及品牌相关判定取 `brand_mapping`，涉及材质相关判定取
`material_mapping`。仅调整价格阈值、标题匹配方式、数量或尺码等与映射无关的规则时不调用
本工具。无法判断改动是否涉及映射时，按涉及处理并传空取全部。

## 定位目标比价项

比价项名称是业务中文名（如品牌、货号、材质、厚薄、外观、数量、尺码、克重），且随规则组
变化。**必须从本轮真实数据中提取，不得凭记忆或猜测拼写。** 允许的来源只有以下三类：

1. **Badcase 类流程**：解析 `ToolValidationCaseResult.raw_llm_response`（为空时退回
   `analysis_process`）中的 `extracted` 对象，其 key 即该次判定实际生效的比价项清单。
   按 [提取嫌疑比价项](#提取嫌疑比价项) 选出目标项。
2. **修改类流程**：读取基础提示词 `prompt_content` 的 `## 比价项` 章节，其中的比价项名称
   即可选集合；再结合用户诉求选出目标项。
3. **用户明确指名**：用户本轮直接点明要改哪个比价项时优先采用，但仍需确认该名称存在于
   上述任一来源中；对不上时向用户澄清，不擅自改写成相近名称。

以上来源都不可用，或提取到的名称无法与可选集合对应时，**一律传空 `compare_items` 取全量**，
不得传入未经确认的名称。

### 提取嫌疑比价项

先按 `human_label`、`model_label` 判定错误方向，再据此决定在 `extracted` 中找哪一类项。
方向不同则排查路径不同，不得不分方向地一律套用同一条筛选条件。

| `human_label` | `model_label` | 错误方向 | 在 `extracted` 中选取 |
|---|---|---|---|
| 1 同款 | 2 非同款 | 漏放：判定过严 | `match=false` 的项 |
| 2 非同款 | 1 同款 | 误放：判定过宽 | `match=true` 但 `left.value` 或 `right.value` 为空或为 `缺失` 的项 |
| 任意 | 3 无法判断 | 决策路径缺失 | `extracted` 中缺失的比价项 key，或未能给出结论的项 |

三类的成因各不相同：`match=false` 是判为不一致的直接原因；两侧抽不到证据时模型常默认判为
一致，这是漏判同款的典型成因；`model_label=3` 通常不是规则松紧问题，而是骨架缺少证据不足时
的兜底分支，须据此定位相关比价项，不得直接按过严或过宽处理。

`key_diff_point` 非空时，其指向的比价项也纳入。同一样本命中多个方向时合并去重。

按上表选不出任何项（如全部 `match=true` 且证据齐全）时无法定位，传空 `compare_items`
取全量：此时误判原因通常是某条规则整体过宽，需要对比全部规则才能发现。

`category_id` 必须与 `compare_items` 同时对齐：调用时 `category_ids` 传本条样本的
`category_id`，规则按类目生效，类目不一致会取回无关规则。

## 结果校验

仅当 `resp_code=1` 时处理返回内容。以下属于关键数据不完整，按 `SKILL.md` 的失败规则停止：

- `price_rule_json` 为空，或无法解析为 JSON；
- `include_special_rule=1` 但 `special_rule_json` 为空；
- 调用了 `tool_query_rule_data_table` 但 `data_table_json` 为空或无法解析。

传入了 `compare_items` 却返回空规则集合时，不得据此认定"该比价项没有规则"，
而应改为传空 `compare_items` 重新取全量后确认。名称拼写错误与规则确实不存在无法从
空结果中区分，静默接受会导致生成出缺失规则的提示词。

## 使用

`price_rule_json` 用于比价项、来源优先级和匹配逻辑；`special_rule_json` 用于特殊规则、
适用范围和例外；`data_table_json` 用于母子品牌、材质分组等映射。

**不得补充 MCP 未返回的规则或映射。** 按需加载时未取回的比价项规则视为本轮不涉及，
既不写入提示词的改动范围，也不据此删除提示词中已有的对应章节；修改类流程只改动目标
比价项，其余章节从基础提示词原样保留。

同一会话中已经取得且作用域一致的规则与映射，直接复用上文内容，不重复调用。
`ruleGroupId`、`categoryIds`、`compare_items` 或 `table_types` 任一项与上次不同时，
属于不同作用域，必须重新调用。
