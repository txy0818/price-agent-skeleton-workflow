# 新建提示词

只创建提示词草稿。共享步骤见 [shared-steps.md](shared-steps.md)。

## 查询与生成

1. 执行 `[S1]` 查询 Agent，读取 `data.card.agent_name`、`data.card.rule_group_id`、
   `data.card.category_ids_str`、`data.current_prompt_version_id` 和
   `data.latest_draft_prompt_version_id`。
2. 按 [base-version-policy.md](base-version-policy.md) 的「新建类」处理：线上和草稿 ID
   都为 `0` 时从零生成；任一非零时先展示已有版本，询问「基于指定版本修改」还是
   「仍从零创建」。确认前不生成、不写入。
3. 执行 `[S2]` 加载规则与映射，**取全量**：`compare_items` 传空，`include_special_rule=1`，
   并调用 `tool_query_rule_data_table` 且 `table_types` 传空。新建骨架必须依据全部规则生成
   全部生效比价项，不得按需裁剪，否则会漏掉本应生效的比价项。
4. 从零生成时读取 [rule-transformation-guide.md](rule-transformation-guide.md) 和
   [skeleton-format.md](skeleton-format.md)；需要更多规则表达范式时才读取
   [rule-writing-examples.md](rule-writing-examples.md)。
5. 执行 `[S3]` 生成并校验完整提示词。

## 提案

从零新建没有 Diff 基线，按 `[S4]` 展示完整提示词：

````markdown
## 新建同款判定提示词提案
- Agent：`<名称；查不到时写ID>`
- 提示词：`待创建（未生成ID）`
- 状态：尚未保存
### 生成说明
- 判定目标：判断左右商品是否同款
- 关键规则：<关键匹配、否决和例外>
- 风险：<待验证内容；没有写“无”>
### 完整提示词
```text
<完整且未省略的提示词>
```
以上仅为提案。确认无误请回复：**确认创建提示词草稿**。确认后只创建提示词草稿，不发布。
````

## 确认后

执行 `[S5]`，`prompt_version_id=0`、`diff_content=""`、`change_reason` 写新建原因。
提醒后续修改、验证和发布使用新 ID。
