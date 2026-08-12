# 单条 Badcase Prompt 缺失检查

进入本流程立即锁定 `badcaseMode=SINGLE`。完整读取并执行
[badcase-processing-workflow.md](badcase-processing-workflow.md)、
[badcase-attribution-policy.md](badcase-attribution-policy.md)、
[rule-loading-policy.md](rule-loading-policy.md) 和 [shared-steps.md](shared-steps.md)。

同一轮完成验证结果、真实规则、必要映射、Prompt 缺失对照和 CDN 查询。只分析 Prompt 缺失；禁止分析
模型错误、人工标签、SKU/SPU 口径、商品图片或其他原因。

- `PROMPT_MISSING`：生成补齐缺失后的候选完整 Prompt，执行 `[S3]`，使用“存在缺失”模板。
- `NO_PROMPT_MISSING` 或 `PROMPT_MISSING_UNCONFIRMED`：不执行 `[S3]`，使用“未修改”模板。

## 存在缺失

````markdown
## Badcase Prompt 缺失检查
- Agent：`<名称；查不到时写ID>`
- 任务 / Badcase / 验证集：`<task_id> / <case_id> / <dataset_id>`
- 任务提示词：`<名称>`（ID：`<ID>`）
- 检查结果：`PROMPT_MISSING`
- 修改建议 ID（diff_id）：`<同次 S3 非零 data.diff_record_id>`
- 状态：尚未保存
### 缺失对照
| 缺失类型 | Badcase 触发信号 | 类目 / 对象 | 真实规则、关系或格式依据 | Prompt 当前实现 | 确认缺失 | 建议补入 |
|---|---|---|---|---|---|---|
| <五类枚举之一> | <标签方向、比价项键/match 或全量兜底> | <逐项列出> | <原始依据及转换结果> | <原文和位置或“未实现”> | <缺失内容> | <通用补齐内容> |
### 结论
- 是否修改提示词：是
- 说明：基础 Prompt 未完整实现上述真实规则或合法映射；本流程不判断该缺失是否导致模型误判。
### Diff
```diff
<逐字符复制同次 tool_validate_prompt_skeleton 返回的 data.diff_content>
```
完整提示词已生成并通过格式校验，未在此展开。需查看请回复「展开完整提示词」。
以上仅为修改提案。确认后创建新提示词草稿，不覆盖基础版本。
确认无误请回复：**确认创建提示词草稿**。
验证集报告：[查看 CDN 报告](<真实链接>)
````

## 未修改

````markdown
## Badcase Prompt 缺失检查
- Agent：`<名称；查不到时写ID>`
- 任务 / Badcase / 验证集：`<task_id> / <case_id> / <dataset_id>`
- 任务提示词：`<名称>`（ID：`<ID>`）
- 检查结果：<`NO_PROMPT_MISSING` / `PROMPT_MISSING_UNCONFIRMED`>
### 检查证据
- 核验范围：Step3、类目全部有效比价项、必要品牌映射、必要材质映射、输出格式
- 对照结果：<未发现缺失；或列明无法取得的规则/Prompt/映射证据>
### 结论
- 是否修改提示词：否
- 说明：<未发现 Prompt 缺失；或证据不足，补齐后重跑>。本流程不继续分析其他 Badcase 原因。
验证集报告：[查看 CDN 报告](<真实链接>)
````

CDN 无链接时替换最后一行。模板前后不得添加文字。缺失模板必须有本轮成功 S3、非零 diff_id 和非空
Diff；未修改模板禁止出现 Diff、确认话术或非 Prompt 缺失归因。
