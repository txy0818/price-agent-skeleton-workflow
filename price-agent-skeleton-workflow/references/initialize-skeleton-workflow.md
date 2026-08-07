# 初始化提示词

初始化只在当前业务上下文的 `promptVersionId` 缺失或为 0 时进入。`createAgent` 不再预建 Prompt，
因此初始化没有基础版本；不得要求用户提供提示词 ID 或名称。共享步骤见
[shared-steps.md](shared-steps.md)。

## 格式必读门槛

进入本 workflow 后，必须先完整读取 [skeleton-format.md](skeleton-format.md)，再生成骨架。
该文件是初始化的强制格式规范，不是可选参考。未能完整读取时停止，不得生成正文、调用校验
工具或展示提案。

## 查询与生成

1. 校验 `当前业务上下文.promptVersionId`：大于 0 表示已有提示词，直接说明并停止初始化；
   缺失、为 0 或仍为模板占位符时继续。
2. 仅当业务上下文缺少 `ruleGroupId` 时执行 `[S1]`；已有则跳过。`ruleGroupId` 与 `agentId`
   至少一个必须大于 0。
3. **生成前先查重**：只调用一次
   `tool_query_prompt_skeleton(rule_group_id=<ruleGroupId>, prompt_version_id=0, version_name="",
   query_online=false, query_latest=true, operator=<operator>)`：
   - 查询成功且 `data.prompt_version` 非空：立即把其中的 `version_name` 和 `prompt_version_id`
     告知用户并停止；不得加载规则、生成提示词、展示提案或调用写入 MCP；
   - 明确返回“骨架版本不存在”：说明当前规则组没有 Prompt，继续初始化；
   - MCP 调用失败：按只读 MCP 规则重试一次，仍失败则停止，不得继续生成。
4. 按 `[S2]` 的初始化范围加载全部关联类目规则及过滤后的品牌、材质关系。保留所有生效
   比价项；母子品牌、材质分别最多 50 组，并删除跨行业和不确定项。
5. 确认已完整读取 [skeleton-format.md](skeleton-format.md)，并读取
   [rule-transformation-guide.md](rule-transformation-guide.md) 后生成完整提示词。
6. 使用生成的完整正文执行 `[S3]` 的正式初始化校验：
   `tool_validate_prompt_skeleton(prompt_content=<完整提示词>, operator=<operator>,
   conversation_id=<localConversationId>, base_prompt_version_id=0,
   rule_group_id=<ruleGroupId>, agent_id=<agentId>)`。
7. 若返回 `data.prompt_exists=true`，说明预检后已有其他流程创建 Prompt；立即把
   `data.existing_prompt_name`、
   `data.existing_prompt_version_id` 告知用户，
   不展示初始化提案、不记录确认话术、不调用写入 MCP。只有 `prompt_exists=false` 且
   `valid=true` 时才进入提案。

初始化提案对应的 Diff 记录中，`base_prompt_version_id=0`，`new_prompt_version_id` 为空；不得用
Agent ID、规则组 ID 或其他字段代填。

## 提案

按 `[S4]` 展示完整提示词，标注「尚未保存」和 `prompt_version_id=0（初始化）`，结尾使用：

```text
以上仅为提案。确认无误请回复：确认初始化提示词。确认后服务端会再次检查该规则组是否已有
提示词；已有则直接返回现有结果，仍不存在才创建首个提示词草稿。
```

完整提示词全文必须一次性放入同一个四反引号 `text` 围栏中，不得拆分或省略。

## 确认后

执行 `[S5]` 的 `INITIALIZE` 路径，只调用一次
`tool_edit_prompt_skeleton(rule_group_id=<ruleGroupId>, prompt_version_id=0, operator=<operator>,
source_type=3, prompt_diff_record_id=<S3返回ID>)`。若走兜底路径则保持 `prompt_version_id=0`，并传入
与 S3 完全一致的 `prompt_content` 和 `conversation_id`。

服务端必须再次按 `rule_group_id` 检查：已有 Prompt 时直接返回该 Prompt 的真实结果，不创建；
没有时才创建首个草稿。成功后把工具实际返回的 `new_prompt_version_id`、`new_prompt_name`、
`version_no` 告知用户，后续上下文应使用该新 ID。
