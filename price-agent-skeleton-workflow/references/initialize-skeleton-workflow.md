# 初始化提示词
对任意提示词写操作，只要 `promptVersionId` 缺失、为 0 或仍为模板占位符就进入本流程；用户具体使用新建
、创建、初始化、重新生成、修改、优化、调整、增删或按照规则生成等哪一种说法都不影响路由。初始化没有基
础版本。共享步骤见 [shared-steps.md](shared-steps.md)。
## 必读
完整读取 [shared-steps.md](shared-steps.md)、[base-version-policy.md](base-version-policy.md)、
[skeleton-format.md](skeleton-format.md)、[rule-loading-policy.md](rule-loading-policy.md) 和
[rule-transformation-guide.md](rule-transformation-guide.md)、[enums.md](enums.md)。禁止读取 `full-skeleton-example.md` 或
任何历史完整提示词作为生成范式；格式只能来自 `skeleton-format.md`。随后按下列顺序执行，首条可见回复
只能是初始化提案、已存在结果或明确错误。
## 查询与生成
1. 重新读取本轮业务上下文 `promptVersionId`；不得使用进入 workflow 前缓存的值、上一轮值或任何
   工具返回的新版本 ID。该值大于 0 时停止并按入口重新路由 EDIT；否则锁定本轮初始化值为 0。
2. `ruleGroupId` 缺失时执行 `[S1]`；`ruleGroupId` 与 `agentId` 至少一个大于 0。
3. 生成前只调用一次：
   `tool_query_prompt_skeleton(ruleGroupId=<ruleGroupId>, promptVersionId=0, versionName="", queryOnline=false, queryLatest=true, operator=<operator>)`
   - 返回非空 `data.promptVersion`：告知其 `data.promptVersion.versionName` 和
     `data.promptVersion.promptVersionId` 后停止，不加载规则、不生成或写入。
   - 返回 `baseResp.respCode=21` 且去除首尾空白后的
     `baseResp.respDesc="未找到匹配的骨架版本"`（JSON 序列化展示为
     `baseResp.respCode=21`、`baseResp.respDesc="未找到匹配的骨架版本"`）：这是"明确不存在"的唯一判
定，表示当前规则组
     尚无可复用骨架；不得作为工具失败展示给用户，继续执行第 4 步加载规则并生成初始化候选。
   - `respCode=21` 但 `respDesc` 不是上述精确文本，或只有错误码/错误文案之一：不属于"明确
     不存在"，按异常响应停止。
   - 调用失败、超时、响应不完整或其他非成功响应：立即停止，不重试。
4. 按 `[S2]` 先用 `tool_query_category_ids` 取得全部 CategoryIds，再逐 ID 查询规则，并加载过滤后的
   品牌、材质关系，保留全部生效比价项。候选映射必须
   是本轮关系工具返回集合的子集；母子品牌 100 组、材质 50 组只是上限，不足时保持实际数量，
   禁止补齐、编造、跨行业拼接或复制示例。
5. 读取 [rule-transformation-guide.md](rule-transformation-guide.md)，为每个真实比价项生成
   `expectedPriority` 和 `expectedMatch` 内部账本，再严格按 `skeleton-format.md` 生成并反向解析候选
   全文逐项比对；来源顺序或匹配语义任一不一致时必须内部修正，不得改用其他结构、把示例当业务
   事实或进入 `[S3]`。
6. 用完整正文执行 `[S3]`，其中 `basePromptVersionId=0`；初始化的查重响应和 Diff 字段按 `[S3]` 处理。
## 提案
按 `[S4]` 展示。以下不是示例，而是初始化提案唯一允许的可见回复协议；必须以
`## 提示词初始化提案` 为第一行，逐项、按序套用，模板前后不得添加任何文字。所有尖括号字段
必须替换成本轮真实上下文、查询或 S3 返回值，不得原样输出占位符：
`````markdown
## 提示词初始化提案
- 基础提示词版本：`promptVersionId=0（初始化）`
- 修改建议 ID（diff_id）：`<原样引用 S3 返回的非零 data.diffRecordId>`
- 状态：尚未保存
### 合理性判断
- 初始化目标：为当前规则组创建首个 PriceStudio 比价 Agent 提示词
- 规则与映射依据：<本轮实际加载的类目规则、母子品牌和材质映射范围>
- 适用范围与风险：<关联类目范围、需要重点回归的规则或映射>
### 完整提示词
````text
<逐字符引用同次 S3 请求已提交并锁定的 promptContent；不得省略、拆分或重新生成>
````
以上仅为提案。确认无误请回复：**确认初始化提示词**。确认后服务端会再次检查该规则组是否已有提示词；已有则直接返回现有结果，仍不存在才创建首个提示词草稿。
`````
发送前按 `[S4]` 自检。
## 确认后
紧邻确认时执行 `[S5]`，使用 `sourceType=3`；其余取值、校验和回复完全按 `[S5]`。
