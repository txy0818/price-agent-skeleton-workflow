# 单条 Badcase 分析

本流程只处理业务上下文指定的一条 Badcase。只调用一次 `tool_query_validation_result`，取得结果后由模型
结合返回内容自由分析；不读取其他 reference，不调用规则、品牌、材质、CDN、Prompt 校验或写入工具。

## 1. 上下文门禁

只从本轮业务上下文读取：`validationTaskId`、`validationCaseId`、`promptVersionId`、`operator`。前三个
ID 必须大于 0；禁止从用户文字提取、补写或替换 ID。缺失时固定回复：

`请通过页面的「一键分析 Badcase」按钮对单条 Badcase 发起分析，当前不支持手动填写 Badcase、Case ID 或验证任务 ID。`

## 2. 唯一工具调用

只调用一次：

`tool_query_validation_result(validationTaskId=上下文.validationTaskId, validationCaseId=上下文.validationCaseId, promptVersionId=上下文.promptVersionId, operator=上下文.operator)`

禁止传 `labelFilter`、`page`、`continuationToken`、`conversationId` 或其他字段；禁止重试。要求外层
`result=1`、`data.baseResp.respCode=1`，响应任务 ID、Prompt ID、唯一结果的 Case ID 与请求一致，且结果
满足 `isCorrect=0`。失败、空结果、多结果或身份不一致时，只返回真实错误并停止。

## 3. 自由分析

完整读取该次响应可用内容，包括任务 Prompt、人工/模型标签、`reason`、`analysisProcess`、左右商品标题
及商品 JSON。模型自行判断可能问题，可分析 Prompt、模型执行、标签、SKU/SPU、商品证据、输出格式或
证据不足；不强制枚举、判断树、逐项表或固定归因顺序。

约束只有以下四条：

1. 结论必须引用响应中的具体证据，区分“已确认”和“可能”；不得编造规则、映射、图片内容或工具结果。
2. 未调用真实规则/关系接口时，不得声称已确认当前规则或品牌、材质映射；只能指出 Prompt 自身明显缺失、
   内部冲突或疑似未同步。
3. 本流程只分析，不生成候选 Prompt、Diff，不调用 `tool_validate_prompt_skeleton` 或
   `tool_edit_prompt_skeleton`，也不请求用户确认写入。
4. 可见回复必须少于 5000 个中文字符；优先写最关键的 1～3 个问题，避免复述完整接口响应。

## 4. 唯一输出模板

严格使用以下模板，字段和顺序不变，模板前后不添加文字：

```markdown
## Badcase 分析
- 任务 / Badcase / 验证集：`<task_id> / <case_id> / <dataset_id；缺失写“未返回”>`
- 任务提示词：`<versionName；缺失写“未返回”>`（ID：`<promptVersionId>`）
- 标签：`human_label=<值>`（<映射>），`model_label=<值>`（<映射>）

### 发现的问题
1. **<问题名称>**：<接口证据及判断>
2. **<可选；无第二项则删除整行>**：<接口证据及判断>
3. **<可选；无第三项则删除整行>**：<接口证据及判断>

### 结论
- 主要问题：<一句话结论；无法确认时明确写证据不足>
- Prompt 是否可能需要调整：<是 / 否 / 待核验>
- 建议：<下一步处理或补证建议；不得声称已修改>
```

`human_label/model_label` 映射：`1=同款`、`2=非同款`、`3=无法判断`；缺失或其他值原样展示并写“未知”。
