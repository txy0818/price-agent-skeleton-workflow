# 初始化提示词

> **不得直接回复**：进入本流程后必须完整读取本文件及其要求的共享步骤、规则加载策略、转换指南
> 和 `skeleton-format.md`，完成查重、规则加载、候选全文及真实 `[S3]` 校验后，才可按固定初始化
> 提案模板回复。用户已给出完整方向也不得跳过上述步骤。

对任意提示词写操作，只要 `promptVersionId` 缺失、为 0 或仍为模板占位符就进入本流程；用户具体使用新建、创建、初始化、重新生成、修改、优化、调整、增删或按照规则生成等哪一种说法都不影响路由。初始化没有基础版本。共享步骤见 [shared-steps.md](shared-steps.md)。

## 必读门槛

生成正文前完整读取 [skeleton-format.md](skeleton-format.md)；读取失败即停止，不得生成、校验或展示提案。[full-skeleton-example.md](full-skeleton-example.md) 仅在格式细节仍不明确时查看相关片段，不得复制其中的业务数据。

## 回复前执行门禁

本节是必须在第一条用户可见答复前实际完成的顺序，不是计划、建议或说明。除工具返回明确错误或查到已有提示词外，完成全部门禁前禁止输出任何用户可见答复：

1. 完整读取 [shared-steps.md](shared-steps.md)、[base-version-policy.md](base-version-policy.md)、[skeleton-format.md](skeleton-format.md)、[rule-loading-policy.md](rule-loading-policy.md) 和 [rule-transformation-guide.md](rule-transformation-guide.md)。
2. 确认业务上下文 `promptVersionId` 确实缺失、为 0 或为占位符；只要大于 0 就立即退出本流程并改走 EDIT，禁止继续初始化。
3. 实际查询规则组是否已有提示词；明确不存在时立即执行 `[S2]`，加载全部关联类目规则及必要映射。
4. 在内存中生成并自检完整候选正文，随后实际执行 `[S3]`；`valid=false` 时在本轮内部按 errors 自动修正并用完允许的重试次数，期间禁止向用户展示错误或询问是否同意补全。
5. 仅在 `prompt_exists=false`、`valid=true` 且 `diff_record_id>0` 后，当轮输出完整初始化提案；用户确认只用于后续写入。

初始化请求本身已经授权执行查询、候选正文生成、自动修正和校验，不需要用户再次确认开始或继续。禁止输出路由说明、步骤清单、进度、校验中间错误或异步承诺，包括“已进入/加载初始化流程”“接下来会查询”“正在生成/校验”“生成完成后再给你”“等你回复继续生成”“同意后补全/修复”。首条正常答复必须直接是 `valid=true` 的完整初始化提案；自动重试后仍失败则只返回最终错误。只有合规提案展示后的实际写入需要确认。

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

按 `[S4]` 展示。以下不是示例，而是初始化提案唯一允许的可见回复协议；必须以
`## 提示词初始化提案` 为第一行，逐项、按序套用，模板前后不得添加任何文字。所有尖括号字段
必须替换成本轮真实上下文、查询或 S3 返回值，不得原样输出占位符：

`````markdown
## 提示词初始化提案
- 基础提示词版本：`prompt_version_id=0（初始化）`
- 修改建议 ID（diff_id）：`<原样引用 S3 返回的非零 data.diff_record_id>`
- 状态：尚未保存
### 合理性判断
- 初始化目标：为当前规则组创建首个 PriceStudio 比价 Agent 提示词
- 规则与映射依据：<本轮实际加载的类目规则、母子品牌和材质映射范围>
- 适用范围与风险：<关联类目范围、需要重点回归的规则或映射>
### 完整提示词
````text
<同次 S3 校验通过的完整 prompt_content；不得省略、拆分或重新生成>
````
以上仅为提案。确认无误请回复：**确认初始化提示词**。确认后服务端会再次检查该规则组是否已有提示词；已有则直接返回现有结果，仍不存在才创建首个提示词草稿。
`````

发送前按 `[S4]` 输出协议自检。全文必须放入同一个四反引号 `text` 围栏，全文内部的三反引号
代码围栏原样保留。只有取得非零 `diff_record_id` 才能展示本提案；ID 缺失或为 0 时返回明确错误，
不得确认或写入。

## 确认后

执行 `[S5]`：`prompt_version_id` 重新读取本轮最新业务变量 `promptVersionId`，`source_type=3`，不得
因提案生成时是初始化而强制改回 0。服务端校验本轮版本与 Diff 基础版本的关联关系；匹配且防重
命中时返回现有 Prompt，不创建，创建成功后告知实际的 `new_prompt_version_id`、
`new_prompt_name` 和 `version_no`，后续使用新 ID。
