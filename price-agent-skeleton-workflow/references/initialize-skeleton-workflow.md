# 初始化提示词

对任意提示词写操作，只要 `promptVersionId` 缺失、为 0 或仍为模板占位符就进入本流程；用户具体使用新建、创建、初始化、重新生成、修改、优化、调整、增删或按照规则生成等哪一种说法都不影响路由。初始化没有基础版本。共享步骤见 [shared-steps.md](shared-steps.md)。

## 必读门槛

生成正文前完整读取 [skeleton-format.md](skeleton-format.md)；读取失败即停止，不得生成、校验或展示提案。[full-skeleton-example.md](full-skeleton-example.md) 仅在格式细节仍不明确时查看相关片段，不得复制其中的业务数据。

## 回复前执行门禁

本节是必须实际完成的顺序，不是计划、建议或说明。除工具返回明确错误或查到已有提示词外，完成全部门禁前禁止输出任何用户可见答复：

1. 完整读取 [shared-steps.md](shared-steps.md)、[base-version-policy.md](base-version-policy.md)、[skeleton-format.md](skeleton-format.md)、[rule-loading-policy.md](rule-loading-policy.md) 和 [rule-transformation-guide.md](rule-transformation-guide.md)。
2. 确认业务上下文 `promptVersionId` 确实缺失、为 0 或为占位符；只要大于 0 就立即退出本流程并改走 EDIT，禁止继续初始化。
3. 实际查询规则组是否已有提示词；明确不存在时立即执行 `[S2]`，加载全部关联类目规则及必要映射。
4. 在内存中生成并自检完整候选正文，随后实际执行 `[S3]`；`valid=false` 时按 errors 修正并在允许次数内重试。
5. 仅在 `prompt_exists=false` 且 `valid=true` 后，当轮输出完整初始化提案；用户确认只用于后续写入。

禁止把“已加载初始化流程 / 下一步将查询 / 等你回复继续生成”作为正常结果。

## 查询与生成

1. `promptVersionId>0` 时停止初始化；否则继续。
2. `ruleGroupId` 缺失时执行 `[S1]`；`ruleGroupId` 与 `agentId` 至少一个大于 0。
3. 生成前只调用一次：
   `tool_query_prompt_skeleton(rule_group_id=<ruleGroupId>, prompt_version_id=0, version_name="", query_online=false, query_latest=true, operator=<operator>)`
   - 返回非空 `data.prompt_version`：告知其 `version_name` 和 `prompt_version_id` 后停止，不加载规则、不生成或写入。
   - 明确不存在：继续。
   - 调用失败：按只读规则重试一次，仍失败则停止。
4. 按 `[S2]` 加载全部关联类目规则及过滤后的品牌、材质关系，保留全部生效比价项；母子品牌最多 100 组、材质最多 50 组，优先删除低相关、不常见、跨行业及不确定项。
5. 读取 [rule-transformation-guide.md](rule-transformation-guide.md)，严格按 `skeleton-format.md` 生成并自检完整正文，不得改用其他结构或把示例当业务事实。
6. 用完整正文执行 `[S3]`，其中 `base_prompt_version_id=0`。若 `data.prompt_exists=true`，返回 `existing_prompt_name` 和 `existing_prompt_version_id` 后停止；仅 `prompt_exists=false` 且 `valid=true` 时展示提案。初始化 Diff 的 `new_prompt_version_id` 留空。

## 提案

按 `[S4]` 展示完整正文，标注「尚未保存」和 `prompt_version_id=0（初始化）`。全文放入同一个四反引号 `text` 围栏，不得拆分或省略。结尾固定为：

```text
以上仅为提案。确认无误请回复：确认初始化提示词。确认后服务端会再次检查该规则组是否已有提示词；已有则直接返回现有结果，仍不存在才创建首个提示词草稿。
```

## 确认后

执行 `[S5]` 的 `INITIALIZE` 路径：`prompt_version_id=0`、`source_type=3`。服务端防重命中时返回现有 Prompt，不创建；创建成功后告知实际的 `new_prompt_version_id`、`new_prompt_name` 和 `version_no`，后续使用新 ID。
