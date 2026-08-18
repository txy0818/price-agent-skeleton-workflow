---
name: price-agent-skeleton-workflow
description: 处理 PriceStudio 同款判定提示词（骨架）的新建、查询、修改、优化、增删、保存和发布，也处理页面发起的单条 Badcase 分析。Badcase 只查询一次验证结果并由模型自由分析，不支持整任务、分页、续批或自动修改 Prompt。也处理对紧邻提示词提案的确认、同意、保存、创建草稿、拒绝或取消。
---

# PriceStudio 提示词工作流

## MCP 运行时命名

- Tool 名称固定使用下划线，例如 `tool_validate_prompt_skeleton`；这是 MCP 的方法标识。
- Tool 的入参与 `message.content.data` 中的业务响应字段固定使用万擎 Schema 的小驼峰，例如
  `promptContent`、`baseResp`、`diffRecordId`。不得使用 proto/Java 源码中的下划线字段名。
- 网关外层异常字段 `error_msg` 是唯一例外；其余字段一律按小驼峰读取。

## 入口边界

- 进入 Skill 时只读取本轮业务上下文中已展开的 `promptVersionId`，绑定为
  `currentPromptVersionId` 并丢弃历史同名值。历史消息、模型记忆、版本名称、提案及工具返回的任何
  Prompt ID 不得参与赋值、比较或纠正。字段缺失、为空或为模板占位符时规范化为 0 并走
  INITIALIZE；非空但不是非负整数时停止，禁止从历史补值。具体查询和
  确认取值统一按 [base-version-policy.md](references/base-version-policy.md) 与
  [shared-steps.md](references/shared-steps.md) 执行。
- 用户指定最新、线上、归档、草稿、ID、名称或版本时，执行
  [base-version-policy.md](references/base-version-policy.md) 的查询对象门禁，不自行查询或切换。
- 只使用真实上下文和 MCP 响应；禁止编造 ID、规则、映射、样本、正文、Diff 或工具结果。
- 写入前必须有合规提案和有效确认；工具调用失败、超时或响应不完整时不重试。只有
  `tool_validate_prompt_skeleton` 明确返回 `valid=false` 时，才允许按 `[S3]` 根据 `data.errors`
  修正候选后有限重试；这不是传输失败重试。

## 识别操作

- “新建提示词”“创建提示词”“初始化提示词”“优化提示词”“改善提示词”“修改提示词”
  “完善提示词”“重写提示词”及对应“骨架”说法，即使
  没有说明方向或具体内容，也属于完整的提示词写操作；禁止追问，直接按本轮 `promptVersionId`
  进入下表 EDIT 或 INITIALIZE。
- 仅说“优化规则”“改善规则”“完善规则”“修改规则”等泛化对象，且未说明比价项、母子品牌、
  材质、阈值或具体改动 → 不进入 workflow、不调用 MCP；只回复：“我还不确定你说的‘规则’具体
  指什么，请说明要调整的规则或具体内容。”
- “修改规则：<具体内容>”、明确某个比价项，或给出可执行的规则增删改内容 → 提示词写操作；
  不因用户使用“规则”一词而追问。
- “新增母子品牌：A→B”“删除材质映射：X→Y”等内容可唯一定位 → 精确提示词写操作。
- “展开完整提示词”“查看完整提示词”“展示完整提示词” → 展开上一条有效提案锁定的候选全文，
  不是范围外请求，也不是重新查询页面当前提示词。
- “确认”“同意”“保存”“创建草稿”等表达 → 提案决策操作；无论是否紧邻都不是范围外请求，
  直接进入 [shared-steps.md](references/shared-steps.md) `[S5]`，入口不提前判断或拒绝。
  命中后不得执行其他 workflow、查询或自由分析；下一项业务动作必须是 `[S5]` 规定的写入调用。
  没有本轮写入工具真实响应时，禁止声称提案无效、已失效、版本不匹配或要求重新发起。
  写入调用前读取当前确认回合业务上下文中的 `promptVersionId`：显式非负整数直接使用；缺失、空值
  或模板占位符统一补为 `0`。不得从上一条提案的基础版本、历史消息或工具结果补值。只要存在有效
  非零 `diff_id`，就不得因 `promptVersionId` 缺失而停止，必须以 `promptVersionId=0` 调用写入工具；
  Diff 与版本关联交给服务端校验。仅当当前上下文显式提供负数或非数字值时停止并提示参数非法。
- 用户输入任何 Badcase 相关内容时，直接校验本轮业务上下文的 `validationTaskId` 和
  `validationCaseId`；只有两者都大于 0 才允许继续。任一缺失、为 0
  或为占位符时，不调用任何 Badcase workflow 或 MCP，只回复：“请通过页面的「一键分析 Badcase」
  按钮发起分析，当前不支持手动填写 Badcase、Case ID 或验证任务 ID。”用户消息或 input 中即使写有
  Badcase ID、Case ID、验证任务 ID 或 `#数字`，也不得将其作为上下文、不得向用户索取 ID。
  门禁通过后固定进入单条流程。禁止从用户文字或 input 中的
  `#数字`、Badcase ID、Case ID、任务 ID 等描述提取、补写或覆盖业务上下文，也不得
  因用户未在文字中重复 ID 而改走普通 EDIT。
- Badcase 只执行 [badcase-single-workflow.md](references/badcase-single-workflow.md)：只调用一次
  `tool_query_validation_result`，随后由模型基于响应自由分析并按固定模板输出；不调用其他业务 MCP，
  不生成 Prompt、Diff 或写入建议，输出少于 5000 个中文字符。
- 写操作即使没有方向也是完整请求，禁止追问方向或索取 Badcase；只按 `promptVersionId` 选路，
  用户使用“初始化”或“修改”等措辞不能改变版本路由。

## 选择唯一 workflow

| 请求或状态 | 必须完整读取并执行 |
|---|---|
| Badcase 意图且上下文 `validationTaskId` 或 `validationCaseId` 缺失、为 0 或占位符 | 不调用 MCP；只输出入口门禁的固定按钮提示 |
| Badcase 意图且上下文 `validationTaskId>0 && validationCaseId>0` | [badcase-single-workflow.md](references/badcase-single-workflow.md) |
| 仅查询当前提示词 | [base-version-policy.md](references/base-version-policy.md) |
| 紧邻上一条有效提案后要求展开/查看/展示完整提示词 | 恢复该提案 workflow，只执行 [shared-steps.md](references/shared-steps.md) `[S4]` 的展开全文分支 |
| 确认/同意/保存/创建草稿 | 直接执行 [shared-steps.md](references/shared-steps.md) `[S5]`；所有门禁和错误均由 `[S5]` 处理 |
| 任意提示词写操作且本轮 `promptVersionId>0` | [edit-skeleton-workflow.md](references/edit-skeleton-workflow.md) |
| 任意提示词写操作且本轮 `promptVersionId` 缺失、为 0 或占位符 | [initialize-skeleton-workflow.md](references/initialize-skeleton-workflow.md) |

`promptVersionId>0` 命中 EDIT 后禁止再查找 INITIALIZE workflow；当前正文残缺不是切换流程或要求
用户补正文的理由，必须由 EDIT 按其格式重构分支处理。

选定 workflow 后，每次工具调用都必须从本轮 `currentPromptVersionId` 重新构造版本参数，禁止复用
上一轮请求参数。工具返回的新版本 ID 只允许展示或校验本次响应，不得用于声称页面选中版本已变化，
不得要求用户切换到该 ID 后重试。

## 不可跳过

选定 workflow 后，禁止用自由回答、模型自检、占位数据或“流程骨架”代替实际执行。凡 workflow
要求读取文件或调用 MCP，首条可见回复前必须存在本轮对应的真实读取和调用记录；缺任一记录只能
返回具体文件/工具错误，禁止继续输出提案、确认话术、完整提示词或范围外固定回复。

选定 workflow 后完整读取该文件及其直接要求的 references，并严格按原顺序执行；只跳过 workflow
明确允许跳过的步骤。详细版本选择、规则与关系加载、骨架格式、S1～S5、Diff、确认、写入参数和
输出模板，分别以对应 reference 为唯一权威，本文件不补充或重解释。

INITIALIZE 和 EDIT 全量同步在生成候选前必须完成以下三项硬门禁；任一不满足即停止，不得调用 S3：

1. **来源投影门禁**：逐个 `compareItem` 从本轮规则接口原始 `infoSource` 按 `>`、`/`、`=` 拆成最小
   片段，再删除含 SKU 的片段并归一合法来源。`商品属性栏（站外）`必须保留并归一为`商品属性`。
   对品牌来源链
   `sku图中商品本身品牌信息=比价系统品牌/等级（站内）/商品属性栏（站外）>商品标题>商品详情`，
   候选优先级必须为`商品属性>商品标题`；输出`商品标题`即为失败。
2. **关系响应门禁**：品牌、材质正文只能来自本轮真实关系接口响应及过滤结果。材质必须使用
   `tool_query_material_leaf` 返回的 `materialText[]` 文本行，经类目过滤后原样输出；有效行
   不超过 50 时必须全部输出，超过 50 时才截为 50；禁止只选首行、代表行，禁止改写路径或成员，
   禁止复制历史 Prompt、示例或模型记忆中的材质内容。
3. **候选泄漏门禁**：候选全文不得出现任何生成器门控、条件输出或内部算法说明，
   也禁止用“无符合”“未命中”“暂无”等空结果正文占位。某映射过滤结果为空时整个对应子表不存在；
   品牌和材质均为空时整个`## 映射表`不存在。

上述门禁必须以本轮原始规则响应、关系响应和内部覆盖账本为证据；仅在 reference 中读到算法或示例
不算完成。`tool_validate_prompt_skeleton` 只做格式校验，不能替代这些业务一致性检查。

凡产生提示词提案的 INITIALIZE 或 EDIT，首条可见回复前必须完成目标 workflow
要求的候选全文和 [shared-steps.md](references/shared-steps.md) `[S3]`。只有真实响应满足
`baseResp.respCode=1`、`data.valid=true` 且 `data.diffRecordId>0` 才能按 `[S4]` 展示提案；否则只返回 workflow 规定的最终错误。
禁止用模型自检代替工具、先展示再校验、伪造 Diff，或把中间错误作为用户确认节点。

只有本轮真实 `tool_validate_prompt_skeleton` 调用返回 `baseResp.respCode=1 && data.valid=true &&
data.diffRecordId>0`，才存在有效提案。无本轮 S3 调用记录、`data.diffRecordId=0`、占位 Diff 或
未锁定候选全文时，禁止声称校验通过、套用提案模板、请求确认或响应“展开完整提示词”；按 `[S3]`
返回明确错误。

**凡 workflow 定义了提案，所有提案输出必须完全按照该 workflow 的提案模板和内容要求执行。**
禁止改标题、字段、顺序、围栏或确认话术，禁止省略、概括、重写、补充提案内容，也禁止改用模型
自行组织的格式；模板要求引用工具原文的部分必须逐字符复制。

确认类表达统一直接进入 `[S5]`，入口不得提前判断、拒绝或回复固定错误；紧邻关系、Diff 和写入条件
只按 `[S5]` 处理，不得自行推测服务端消息序号。所有提案和成功回复严格使用 workflow/`[S4]`/`[S5]` 模板，
模板前后不加文字，不省略字段，不输出占位符，不改写服务端 Diff。
