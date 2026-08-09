---
name: price-agent-skeleton-workflow
description: 处理 PriceStudio 同款判定提示词（骨架）的新建、查询、修改、优化、增删、保存、发布、验证任务和 Badcase；也处理对紧邻提示词提案的确认、同意、保存、创建草稿、拒绝或取消。出现上述业务请求或无具体方向的提示词写操作时使用；问候、能力介绍、碎片输入和范围外请求不使用。
---

# PriceStudio 提示词工作流

## 入口边界

- 本轮 `promptVersionId` 是唯一版本上下文；历史及工具返回的 Prompt ID 不得替换它。具体查询和
  确认取值统一按 [base-version-policy.md](references/base-version-policy.md) 与
  [shared-steps.md](references/shared-steps.md) 执行。
- 用户指定最新、线上、归档、草稿、ID、名称或版本时，执行
  [base-version-policy.md](references/base-version-policy.md) 的查询对象门禁，不自行查询或切换。
- 只使用真实上下文和 MCP 响应；禁止编造 ID、规则、映射、样本、正文、Diff 或工具结果。
- 写入前必须有合规提案和有效确认；失败或超时不重试。

## 识别操作

- 明确对象是“提示词/骨架”，或当前页面仅出现新建、修改、优化、完善等写操作词而未说对象 →
  提示词写操作。
- 仅说“完善规则”“修改规则”等泛化对象，未说明比价项、关系、阈值或具体改动 → 不进入 workflow、
  不调用 MCP；回复：“我还不确定你说的‘规则’具体指什么，请说明要调整的规则或具体内容。”
- “新增母子品牌：A→B”“删除材质映射：X→Y”等内容可唯一定位 → 精确提示词写操作。
- 写操作即使没有方向也是完整请求，禁止追问方向或索取 Badcase；只按 `promptVersionId` 选路，
  用户使用“初始化”或“修改”等措辞不能改变版本路由。

## 选择唯一 workflow

| 请求或状态 | 必须完整读取并执行 |
|---|---|
| 单条 Badcase | [badcase-single-workflow.md](references/badcase-single-workflow.md) |
| 验证任务或批量 Badcase | [badcase-task-workflow.md](references/badcase-task-workflow.md) |
| 用户文字描述的 Badcase | [badcase-description-workflow.md](references/badcase-description-workflow.md) |
| 仅查询当前提示词 | [base-version-policy.md](references/base-version-policy.md) |
| 紧邻上一条助手提案后的确认 | 恢复该提案 workflow，只执行 [shared-steps.md](references/shared-steps.md) `[S5]` |
| 提示词写操作且 `promptVersionId>0` | [edit-skeleton-workflow.md](references/edit-skeleton-workflow.md) |
| 提示词写操作且 `promptVersionId` 缺失、为 0 或占位符 | [initialize-skeleton-workflow.md](references/initialize-skeleton-workflow.md) |

## 不可跳过

选定 workflow 后完整读取该文件及其直接要求的 references，并严格按原顺序执行；只跳过 workflow
明确允许跳过的步骤。详细版本选择、规则与关系加载、骨架格式、S1～S5、Diff、确认、写入参数和
输出模板，分别以对应 reference 为唯一权威，本文件不补充或重解释。

凡产生提示词提案的 INITIALIZE、EDIT 或 Badcase 修复，首条可见回复前必须完成目标 workflow
要求的候选全文和 [shared-steps.md](references/shared-steps.md) `[S3]`。只有真实响应满足
`valid=true` 且 `diff_record_id>0` 才能按 `[S4]` 展示提案；否则只返回 workflow 规定的最终错误。
禁止用模型自检代替工具、先展示再校验、伪造 Diff，或把中间错误作为用户确认节点。

**凡 workflow 定义了提案，所有提案输出必须完全按照该 workflow 的提案模板和内容要求执行。**
禁止改标题、字段、顺序、围栏或确认话术，禁止省略、概括、重写、补充提案内容，也禁止改用模型
自行组织的格式；模板要求引用工具原文的部分必须逐字符复制。

确认只绑定可见的上一条助手提案；中间出现其他用户消息后永久失效，显式 `diffId` 也不能恢复。
可见上条含提案标题、非零 Diff ID 和确认话术时直接执行 `[S5]`，不得自行推测服务端消息序号；
是否满足紧邻关系只认写入工具返回。所有提案和成功回复严格使用 workflow/`[S4]`/`[S5]` 模板，
模板前后不加文字，不省略字段，不输出占位符，不改写服务端 Diff。
