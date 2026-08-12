# 验证任务 Badcase Prompt 缺失检查

进入本流程立即锁定 `badcaseMode=TASK`。完整读取并执行
[badcase-processing-workflow.md](badcase-processing-workflow.md)、
[badcase-attribution-policy.md](badcase-attribution-policy.md)、
[rule-loading-policy.md](rule-loading-policy.md) 和 [shared-steps.md](shared-steps.md)。

只检查 Prompt 缺失。分批、续批 Token、紧邻续批和查询参数沿用统一流程；每批按 Case 去重，并按类目
对照全部有效规则及必要映射。阶段分析不调用 `[S3]`。

## 分批分析

````markdown
## 验证任务 Badcase Prompt 缺失阶段检查
- Agent：`<名称；查不到时写ID>`
- 任务 / 验证集：`<task_id> / <dataset_id>`
- 基础提示词：`<名称>`（ID：`<prompt_version_id>`）
- 续批 Token：<hasMore=true 时原样值；否则“无”>
- 批次：第 `<page>` 批
- Badcase：共 `<total>` 条，本批检查 `<analyzed>` 条，累计已检查 `<累计去重数>` 条
### Prompt 缺失统计
| 检查结果 | 数量 | 处理 |
|---|---:|---|
| `PROMPT_MISSING` | <数> | 纳入候选补齐 |
| `NO_PROMPT_MISSING` | <数> | 不修改；不继续分析其他原因 |
| `PROMPT_MISSING_UNCONFIRMED` | <数> | 补齐规则、Prompt 或映射证据后重跑 |
### 明细
| Badcase ID | 类目 | Badcase 触发信号 | 检查结果 | 缺失类型 | 缺失对照或证据缺口 |
|---|---|---|---|---|---|
| `<case_id>` | `<category_id>` | <标签方向、比价项键/match 或全量兜底> | <枚举> | <五类之一或“无”> | <真实依据、Prompt 实现、缺失内容；无缺失时写“五类全部覆盖”> |
### 本批候选补齐
- 确认缺失与样本：<按规则缺失去重；无则写“无”>
- 候选建议：<通用补齐内容；无则写“本批无 Prompt 补齐建议”>
- 与前批关系：<新增/重复/补充/冲突；首批写“首批”>
- 分页状态：<固定条件句式>
- 下一步：<hasMore=true 时“请直接回复「继续分析」查看下一批”；否则提示整合>
- 整合状态：<累计有 PROMPT_MISSING 时可整合；否则无可整合建议>
验证集报告：[查看 CDN 报告](<真实链接>)
````

三种结果数量之和必须等于本批去重明细数。只有 `hasMore=true` 且 Token 非空时展示“继续分析”。CDN
必须是最后一行。不得出现模型、人工标签、SKU/SPU、商品事实、图片、来源执行、match 或其他归因。

## 整合修改建议

只合并当前会话中任务、验证集、基础 Prompt 和服务端续批身份一致的阶段检查，按 Case 和缺失规则去重：

- 至少一条 `PROMPT_MISSING`：补齐全部已确认缺失，生成一份候选完整 Prompt，执行一次 `[S3]`，输出
  任务/验证集/基础 Prompt/已检查进度、缺失清单、同次 diff_id 与原样 Diff，并请求确认创建草稿。
- 没有 `PROMPT_MISSING`：不执行 `[S3]`，输出 `## 验证任务 Badcase Prompt 缺失整合结果`，写明
  `确认 Prompt 缺失：0`、三态统计、是否修改提示词=否，以及未分析数量或证据缺口。

整合和确认写入继续遵守 [shared-steps.md](shared-steps.md) 的 `[S3]`～`[S5]`。候选只补真实规则或合法
映射的缺失内容；禁止夹带其他优化。
