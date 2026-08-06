# 修改提示词

用户要求修改、优化提示词，或直接要求新增、删除、调整某项判定规则，都走本流程。
共享步骤见 [shared-steps.md](shared-steps.md)。

## 查询与判断

1. 仅当业务上下文缺少 `ruleGroupId` 时执行 `[S1]`；已有则跳过。
2. 按 [base-version-policy.md](base-version-policy.md) 确定并查询基础提示词版本。
   线上版本立即停止；草稿或归档版本的 `prompt_content` 必须非空，为空时改走
   `initialize-skeleton` 初始化流程。
   非空时记录 `selectedPromptVersionId=data.prompt_version.prompt_version_id`，要求大于 0，
   并锁定 `writeMode=EDIT`；后续不得切换为 `save_prompt_draft`。
3. **不调用验证任务或验证结果查询。** 用户只要求改规则时，模型不得自行发起
   `tool_query_validation_result` 等验证类查询；需要依据验证数据分析时属于 Badcase 流程，
   由用户明确提出后按对应 workflow 处理。
4. 按 `[S2]` 加载改动范围内的规则与必要映射。用户提到的比价项名称对不上取回的类目规则
   或基础提示词 `## 比价项` 章节时，
   先向用户澄清，不擅自改写成相近名称。
5. 对照取回的规则、映射与基础提示词判断本次修改是否合理。要求与业务规则冲突、缺少关键
   条件或可能扩大误判时，说明冲突和需要确认的问题，停止且不生成草稿。
6. 合理时生成最小必要修改，保留无关规则；未取回规则的比价项视为本轮不涉及，其对应章节
   从基础提示词原样保留，既不改动也不删除。结构或 JSON 变化时读取
   [skeleton-format.md](skeleton-format.md)，需要规则范式时才读取
   [rule-writing-examples.md](rule-writing-examples.md)。改后全文篇幅仍按
   [skeleton-format.md](skeleton-format.md) 的「生成原则」控制在 1 万字以内；基础提示词
   本已超出时不借机大幅删减，只保证本次改动不再显著增长。
7. 执行 `[S3]` 生成并校验修改后的完整提示词。
8. Diff 由 `[S3]` 的 `data.diff_content` 给出，不自行计算或书写。

## 提案

按 `[S4]` 展示：

````markdown
## 提示词修改提案
- Agent：`<名称；查不到时写ID>`
- 基础提示词：`<名称>`（ID：`<ID>`，状态：`<草稿/归档>`）
- 状态：尚未保存
### 合理性判断
- 修改目标：<用户的原始要求或优化方向>
- 规则与映射依据：<支持本次修改的规则依据>
- 影响与风险：<适用范围、冲突和需要回归的项>
- 不修改部分：<明确保留未改动的规则>
### Diff
```diff
<原样引用 S3 返回的 data.diff_content>
```
完整提示词已生成并通过格式校验，未在此展开。需查看请回复「展开完整提示词」。
以上仅为修改提案。确认后创建新提示词草稿，不覆盖基础版本，也不自动发布。
确认无误请回复：**确认创建提示词草稿**。
````

诉求本身已经很明确时，「合理性判断」各项可以写得简短，但不得省略「影响与风险」和
「不修改部分」——这两项用于确认改动范围没有超出预期。

## 确认后

执行 `[S5]` 的 `EDIT` 路径，`prompt_version_id=selectedPromptVersionId`。缺失或为 0 时停止，
不得改用 `save_prompt_draft`。
