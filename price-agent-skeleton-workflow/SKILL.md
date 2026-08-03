---
name: price-agent-skeleton-workflow
description: 为 PriceStudio 新建、修改、保存或发布同款判定提示词（骨架），分析验证任务或用户直接描述的 Badcase，或按用户提出的规则要求修改提示词。用户提到“提示词”或“骨架”均适用；相关写入必须先展示提案并取得明确确认。
---

# PriceStudio 提示词工作流

## 边界

### 上下文前提

- `agentId` 非零、`operator` 非空，缺失则停止。调用任意 MCP 时，只要其 schema 含
  `operator`，必须原样传入该值；不得省略、猜测、改写或用默认操作人，重试与只读核实同样如此。
- `ruleGroupId` 一律传当前业务上下文的值，不校验非零、不与查询返回的规则组比对、
  不被查询结果覆盖。上下文缺失时，唯一来源是本轮 `query_agent_detail` 的
  `data.card.rule_group_id`，取得后本轮固定使用；仍为空属于关键数据缺失，停止并说明。
- 不猜测业务 ID、规则、样本或 CDN。可选的 `datasetId` 与 `validationTaskId` 在当前流程
  必需时，要求用户本轮补充，不从历史消息或 CDN 链接推断。

### 失败处理

- 只读 MCP 遇超时、限流或临时网络错误重试一次；仍失败则说明失败工具与 `resp_desc` 后停止。
- 业务错误（参数缺失、权限不足、数据不存在、归属冲突、关键数据为空）不重试，
  直接说明需要用户补充或修正的内容。
- 写入 MCP 失败、超时或返回不完整时不重试写入，改用只读查询核实。仅当查到唯一且内容一致的
  目标记录才能说明成功，否则告知“本次未能确认操作成功，请先查询最新状态后再操作”并停止。

### 提示词写入

- 展示提案前必须按 `shared-steps.md` 的 `[S3]` 完成格式校验；未通过不展示提案，
  也不调用任何写入或发布 MCP。校验通过的那份内容原样保留至写入，确认后不得重新生成、
  重排、补写或裁剪。
- 修改类流程默认只展示 Diff，不展开提示词全文；仅当用户明确要求「展开完整提示词」时
  完整输出且不得省略。
- 用户确认修改后只能新建草稿版本，不得覆盖基础版本；保存草稿与发布线上是两个独立动作，
  分别取得用户确认。
- 调用 `tool_edit_prompt_skeleton` 后，仅当返回成功、`new_prompt_version_id>0`、
  `version_no>0`、`new_prompt_name` 非空，且基于已有版本时新 ID 不同于基础 ID，
  才能说明创建成功，并明确版本关系：
  - 基于已有版本：`基于提示词 <base_prompt_name>（ID：<base_prompt_version_id>），新建提示词草稿 <new_prompt_name>（ID：<new_prompt_version_id>）。`
  - 从零新建：`从零新建提示词草稿 <new_prompt_name>（ID：<new_prompt_version_id>）。`

## 路由

根据用户目标读取最少必要的 workflow：

- 新建提示词：`references/new-skeleton-workflow.md`
- 修改或优化提示词（含用户直接提出的规则新增、删除或调整）：`references/edit-skeleton-workflow.md`
- 验证任务中的单条 Badcase：`references/badcase-single-workflow.md`
- 整次验证任务 Badcase：`references/badcase-task-workflow.md`
- 用户直接描述 Badcase：`references/badcase-description-workflow.md`
- 发布提示词：`references/publish-workflow.md`
- 纯查询（查看 Agent、提示词、规则、验证任务或结果，不产生任何修改）：不加载 workflow，
  直接调用对应只读 MCP，并按本文件的失败与重试规则处理。

用户目标无法归入以上任一类，或同时可能落入多类且无法判断时，先向用户澄清目标，
不得猜测加载 workflow，也不得先行调用写入 MCP。

单一目标只读取对应的一个文件；用户明确提出多个目标时，读取对应的多个文件并按依赖顺序执行，不得加载无关 workflow。单条与整次 Badcase 同时分析时复用任务查询结果，并按 case ID 去重。用户同时要求分析、修改和发布时，依次完成“分析 → 新提示词草稿 → 验证 → 发布确认”，不得跳步。

各 workflow 引用的 `[S1]`~`[S5]` 定义在 `references/shared-steps.md`；加载任何 workflow 时同时读取。
基础提示词版本的选择统一按 `references/base-version-policy.md`。
`[S2]` 展开的规则与映射按需加载规则在 `references/rule-loading-policy.md`。
对 `version_status`、`task_status`、`label_filter`、`source_type`、`include_special_rule`
等取值含义存疑时，读取 `references/enums.md`。

生成或重组完整提示词时，按当前 workflow 的指引读取格式与转换指南。`full-skeleton-example.md` 是补充参考，不代表当前业务事实，默认不加载；仅在用户明确要求查看完整示例时读取。
