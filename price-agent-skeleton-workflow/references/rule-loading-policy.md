# 规则加载策略

新建或修改提示词、分析 Badcase、生成规则修改建议前，必须按本策略取得当前有效的
`price_rule_json`、`special_rule_json` 和 `data_table_json`。优先使用 Hash 判断历史
规则是否仍有效；`snapshot_time_ms` 只用于审计，不能代替 Hash。

## 查询作用域

历史规则只有在以下条件完全一致时才能复用：

- `ruleGroupId`
- `categoryIds`
- `includeSpecialRule`
- `tableTypes`

比较前对 `categoryIds` 去重并升序排列，对 `tableTypes` 去重并按字典序排列。空集合表示
由服务端按规则组确定范围，与显式集合属于不同作用域。不同作用域的 Hash 和 JSON
不得混用。

## 历史数据

在当前可见会话中查找同一作用域最近一次成功取得的数据：

- `priceRuleHash` 与完整 `priceRuleJson`
- `specialRuleHash` 与完整 `specialRuleJson`
- `dataTableHash` 与完整 `dataTableJson`

Hash 和对应完整 JSON 同时存在才可复用。历史工具结果被截断、只有摘要或只有 Hash 时，
对应 `known_*_hash` 必须传空字符串，不得猜测或拼接规则。

## 查询比价规则

调用
`tool_query_price_rule(rule_group_id=<规则组ID>, category_ids=<当前类目ID列表>, include_special_rule=1, operator=当前业务上下文.operator, known_price_rule_hash=<历史priceRuleHash；没有则空>, known_special_rule_hash=<历史specialRuleHash；没有则空>)`。
仅当 `base_resp.resp_code=1` 时处理响应。

分别处理比价规则和特殊规则：

- 对应 `*_not_modified=true` 时，返回 Hash 必须非空、等于请求中的历史 Hash，且当前仍可
  读取同一作用域的历史完整 JSON；满足后复用历史 JSON。
- 对应 `*_not_modified=false` 时，返回 JSON 和新 Hash 必须均非空；使用新 JSON 和新 Hash。
- `*_not_modified=true` 但历史完整 JSON 已不可见时，重新调用同一工具，只把该部分
  `known_*_hash` 置空；另一部分仍满足复用条件时可继续携带其历史 Hash。
- `include_special_rule=0` 时不要求也不复用特殊规则相关 JSON、Hash 和状态。

## 查询映射表

调用
`tool_query_rule_data_table(rule_group_id=<规则组ID>, category_ids=<当前类目ID列表>, table_types=<需要的映射表类型；全部时传空>, operator=当前业务上下文.operator, known_data_table_hash=<历史dataTableHash；没有则空>)`。
仅当 `base_resp.resp_code=1` 时处理响应。

- `data.not_modified=true` 时，`data.data_table_hash` 必须非空、等于请求中的历史 Hash，
  且当前仍可读取同一作用域的历史完整 `dataTableJson`；满足后复用历史 JSON。
- `data.not_modified=false` 时，`data.data_table_json` 和 `data.data_table_hash` 必须
  均非空；使用新 JSON 和新 Hash。
- `data.not_modified=true` 但历史完整 JSON 已不可见时，使用相同作用域重新调用，
  并传 `known_data_table_hash=""` 取得完整 JSON。

## 异常和使用

以下情况属于关键数据不完整，按 `SKILL.md` 的失败规则停止：

- `not_modified=true`，但请求未携带对应 Hash、历史完整 JSON 不存在，或返回 Hash
  与请求 Hash 不一致；
- `not_modified=false`，但对应 JSON 或新 Hash 为空；
- JSON 无法解析，或返回内容与当前查询作用域不一致。

取得最终数据后：

- `price_rule_json` 用于比价项、信息来源优先级和匹配逻辑；
- `special_rule_json` 用于特殊规则、适用范围和例外；
- `data_table_json` 用于母子品牌、材质分组等映射；
- 三部分分别判断是否变化，允许只刷新发生变化的部分；
- 不得组合新 Hash 与旧 JSON，不得补充 MCP 未返回的业务规则或映射。
