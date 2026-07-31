---
name: price-agent-skeleton-workflow
description: 为 PriceStudio 新建、修改、保存或发布同款判定提示词（骨架），分析验证任务或用户直接描述的 Badcase，或按用户提出的规则要求修改提示词。用户提到“提示词”或“骨架”均适用；相关写入必须先展示提案并取得明确确认。
---

# PriceStudio 提示词工作流

## 边界

- 当前上下文必须提供非零 `agentId`。
- `ruleGroupId` 完全信任当前业务上下文传入值；直接用于 MCP 入参，不校验非零、不与
  Agent、提示词或验证任务返回的规则组比对，也不使用查询结果覆盖。
- 当前上下文必须提供非空 `operator`。调用任意 MCP 时，若该工具的实际 schema
  包含 `operator` 入参，必须原样传入当前业务上下文的 `operator`；不得省略、猜测、
  改写或使用默认操作人。重试和只读核实调用仍使用同一个 `operator`。
- 文档中的 MCP 调用统一写成
  `tool_name(field=value, ..., operator=当前业务上下文.operator)`；列出流程依赖的业务
  入参，不用“传某字段”等自然语言代替调用参数。
- 只读 MCP 遇到超时、限流或临时网络错误时重试一次；仍失败则说明失败工具和 `resp_desc`，然后停止。
- 参数缺失、权限不足、数据不存在、归属冲突或关键数据为空属于业务错误，不重试；明确说明需要用户补充或修正的内容。
- 写入 MCP 失败、超时或返回不完整时不得自动重试写入；改用只读查询核实结果。只有查到唯一且内容一致的目标记录才能说明成功，否则告知用户“本次未能确认操作成功，请先查询最新状态后再操作”，然后停止。
- 不猜测业务 ID、规则、样本或 CDN。
- 草稿、线上和归档提示词均可作为修改的基础版本；修改只能派生新的提示词草稿，不得覆盖
  任一基础版本。归档状态只限制直接发布，不限制作为派生基础。
- 用户确认修改提示词后，必须创建新的草稿版本，不得覆盖基础版本；保存草稿和发布线上是两个独立动作，分别取得用户确认。

## 路由

根据用户目标读取最少必要的 workflow：

- 新建提示词：[references/new-skeleton-workflow.md](references/new-skeleton-workflow.md)
- 修改或优化提示词：[references/edit-skeleton-workflow.md](references/edit-skeleton-workflow.md)
- 验证任务中的单条 Badcase：[references/badcase-single-workflow.md](references/badcase-single-workflow.md)
- 整次验证任务 Badcase：[references/badcase-task-workflow.md](references/badcase-task-workflow.md)
- 用户直接描述 Badcase：[references/badcase-description-workflow.md](references/badcase-description-workflow.md)
- 用户直接提出规则修改要求：[references/direct-rule-change-workflow.md](references/direct-rule-change-workflow.md)
- 发布提示词：[references/publish-workflow.md](references/publish-workflow.md)

单一目标只读取对应的一个文件；用户明确提出多个目标时，读取对应的多个文件并按依赖顺序执行，不得加载无关 workflow。单条与整次 Badcase 同时分析时复用任务查询结果，并按 case ID 去重。用户同时要求分析、修改和发布时，依次完成“分析 → 新提示词草稿 → 验证 → 发布确认”，不得跳步。

当前 workflow 需要比价规则或映射表时，同时读取
[references/rule-loading-policy.md](references/rule-loading-policy.md)，按三个 Hash 分别判断复用或刷新。

生成或重组完整提示词时，按当前 workflow 的指引读取格式与转换指南。`full-skeleton-example.md` 是补充参考，不代表当前业务事实，默认不加载；仅在用户明确要求查看完整示例时读取。
