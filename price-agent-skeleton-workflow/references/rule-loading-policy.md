# 规则查询与过滤

仅使用本轮 MCP 返回；不得补规则、改映射或复制全量候选。

## 查询顺序

1. 初始化时先调用 `tool_query_category_ids` 取得规则组类目；Badcase 优先使用样本类目。
2. 对每个类目调用 `radar_query_price_rule` 确定作用域。
3. 按流程调用品牌、材质关系工具，只写入与作用域相关的结果。

### 规则组类目

`tool_query_category_ids` 支持 `rule_group_id` 或 `agent_id`，至少传一个：

- 已有 `ruleGroupId`：传 `rule_group_id`，`agent_id=0`；
- 没有 `ruleGroupId`：传当前 `agentId`，`rule_group_id=0`；
- 两者都传时只用于校验归属一致，不要传可能冲突的值。

读取 `data.rule_group_id`、`data.agent_id`、`data.category_ids[]`。初始化时响应失败、归属冲突或
`category_ids` 为空即停止，不要求用户额外提供类目，也不得猜测。

### 类目规则

`radar_query_price_rule` 支持 `categoryId`、`categoryName`、`itemId`、`poolId`。有商品 ID 时
优先 `itemId`；已知精确类目时用 ID 或名称；不要传互相冲突的条件。

类目定位条件来自 `tool_query_category_ids.data.category_ids`，或 Badcase/验证任务返回的真实
商品与类目字段。空提示词不是类目来源。

读取 `data.labelCateRule`：

- 作用域：`belongBusiness`、`minCateId`、`cateName`、`cateNameTree`；
- 规则：`searchMethod`、`ruleTableInfo.ruleTableInfo[]` 中的 `compareItem`、`infoSource`、
  `compareLogic`。

要求 `result=1`、业务响应成功、`isExist=true`、作用域和规则表非空。

### 主子品牌

`radar_query_brand_relation(keyword:string)` 返回 `data.groups[]`（`mainBrand`、`subBrands[]`）。

- 初始化：传 `keyword=""`，再按当前 `cateName + belongBusiness` 做常识语义过滤。保留明显相关
  品牌，排除明显跨行业品牌；不确定项不写入。例如“服饰”应排除珠宝品牌“周大福”。
- 修改/Badcase：提取用户、基础提示词或商品中的真实品牌逐个查询，按品牌 ID 去重。

过滤只决定是否内嵌，不得改变返回的主子关系。

### 同款材质

`radar_query_material_relation(keyword:string)` 当前传 `keyword=""`。按以下顺序过滤：

1. 返回字段存在时，要求 `belongBusiness` 一致，且 `cateName` 一致或 `cateNameTree` 包含当前类目；
2. 类目 ID 存在时优先校验，ID 与名称冲突即停止；
3. 缺少作用域字段时，按材质名称和用途做常识过滤：明显相关才保留，不相关或不确定均省略。

## 流程范围

| 流程 | 类目规则 | 品牌 | 材质 |
|---|---|---|---|
| 初始化 | 先查规则组类目，再逐个查询 | 全量后按作用域过滤 | 全量后按作用域过滤 |
| 修改 | 查询改动涉及类目 | 涉及品牌才按关键词查 | 涉及材质才全量查后过滤 |
| Badcase | 按样本商品或类目查询 | 涉及品牌才按商品品牌查 | 涉及材质才全量查后过滤 |

跨类目时分别建立作用域，不得混用。相同工具、入参和作用域可复用本轮结果。

## 骨架体量

初始化目标约 1 万字：完整保留有效 `compareItem`、来源优先级、匹配逻辑和明确例外；品牌、
材质只保留当前作用域必要映射。过长时精简行文或改成运行时查询指令，不得硬截断规则。

## Badcase 方向

解析 `raw_llm_response`（空时用 `analysis_process`）中的 `extracted`：

| 人工/模型 | 方向 | 关注 |
|---|---|---|
| 同款/非同款 | 过严 | `match=false` |
| 非同款/同款 | 过宽 | `match=true` 但证据缺失 |
| 任意/无法判断 | 路径缺失 | 缺失项或无结论项 |

结合左右商品快照区分商品缺失、模型漏抽/抽错和规则松紧。没有历史规则快照时，须说明当前规则
可能已变化。比价项名称仅来自 `extracted` key、基础提示词章节或本轮 `compareItem`。

## 停止条件

顶层或业务响应失败、类目规则/作用域为空、关系响应无法解析、无法可靠过滤、或指定类目与返回
作用域冲突时停止。不得因本轮未加载某映射而删除基础提示词的其他章节。
