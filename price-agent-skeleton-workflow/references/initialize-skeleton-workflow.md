# 初始化提示词

初始化目标是用户指定的空草稿或空归档版本；确认写入时若服务端发现它仍为空则原地填写，
若已非空则派生新草稿。线上版本禁止初始化。共享步骤见 [shared-steps.md](shared-steps.md)。

## 格式必读门槛

进入本 workflow 后，必须先完整读取 [skeleton-format.md](skeleton-format.md)，再生成骨架。
该文件是初始化的强制格式规范，不是可选参考。不得仅根据 workflow 摘要、
模型记忆、历史生成内容或 [full-skeleton-example.md](full-skeleton-example.md) 代替读取。
未能完整读取时必须停止，不得生成正文、调用校验工具或展示提案。

## 查询与生成

1. 要求用户本轮提供待初始化提示词 ID 或名称；未提供时先询问，不调用写入 MCP。
2. 仅当业务上下文缺少 `ruleGroupId` 时执行 `[S1]`；已有则跳过。再按
   [base-version-policy.md](base-version-policy.md) 使用用户给定的 ID 或名称调用
   `tool_query_prompt_skeleton` 精确查询。
3. 校验返回版本属于当前 `ruleGroupId`：
   - `version_status=2`（线上）→ 停止，提示用户选择草稿或归档版本；
   - `version_status=1/3` 且内容为空 → 作为待初始化版本继续；空指 `null`、空串或仅空白；
   - `version_status=1/3` 且内容非空 → 说明该版本已经初始化，直接提示用户，本次不初始化；
   - 其他状态或归属不一致 → 停止并说明真实状态。
   内容为空且状态允许时，立即记录
   `selectedPromptVersionId=data.prompt_version.prompt_version_id` 并锁定
   `writeMode=INITIALIZE`；ID 必须大于 0，否则停止。后续不得改写这两个值。
4. 按 `[S2]` 的初始化范围加载全部关联类目规则及过滤后的品牌、材质关系。保留所有生效
   比价项；母子品牌、材质分别最多 50 组，并在生成后再次删除跨行业和不确定项。禁止复制
   未过滤的全量映射；具体调用与过滤只以
   [rule-loading-policy.md](rule-loading-policy.md) 为准。
5. 确认已完整读取前置必读的 [skeleton-format.md](skeleton-format.md)，并读取
   [rule-transformation-guide.md](rule-transformation-guide.md) 后再生成；需要更多规则表达范式时才读取
   [rule-writing-examples.md](rule-writing-examples.md)。全文篇幅按
   [skeleton-format.md](skeleton-format.md) 的「生成原则」控制在约 1 万字；与「不得裁剪
   生效比价项」冲突时以保留规则为先，改为精简行文或把过长映射表转为引用。
6. 执行 `[S3]` 生成并校验完整提示词，`base_prompt_version_id=selectedPromptVersionId`。
   S3 返回的 `diff_record_id` 不改变 `writeMode=INITIALIZE`。

## 提案

待初始化版本没有有效正文基线，按 `[S4]` 展示完整提示词：

`````markdown
## 初始化同款判定提示词提案
- Agent：`<名称；查不到时写ID>`
- 提示词：`<待初始化版本名称>`（ID：`<待初始化版本ID>`）
- 状态：尚未保存
### 生成说明
- 判定目标：判断左右商品是否同款
- 类目范围：`<cateName>`（`<belongBusiness>`）
- 关键规则：<关键匹配、否决和例外>
- 风险：<待验证内容；没有写“无”>
### 完整提示词
````text
<完整且未省略的提示词>
````
以上仅为提案。确认无误请回复：**确认初始化提示词版本**。确认后写入草稿，不发布。
`````

完整提示词全文必须一次性放入上述同一个四反引号 `text` 围栏中，不得拆成多个围栏，
不得把部分全文放在围栏外。全文内部原有的三反引号 `json` 等代码围栏必须原样保留；
它们在四反引号外层围栏内只是提示词正文，不会提前结束外层内容块。

## 确认后

执行 `[S5]` 的 `INITIALIZE` 路径；只调用 `tool_edit_prompt_skeleton`，并传
`prompt_version_id=selectedPromptVersionId`。缺失或为 0 时停止，不得改用
`save_prompt_draft`。

服务端会在事务内重新查询该 ID 的 `prompt_content`：仍为空则原地填写，返回的
`new_prompt_version_id=selectedPromptVersionId`，对应 Diff 的 `base_prompt_version_id` 与
`new_prompt_version_id` 也都写该 ID；若此时内容已非空，则新增一个提示词草稿。成功后返回
工具实际返回的 `new_prompt_version_id`，并提醒后续修改、验证和发布都使用该 ID。
