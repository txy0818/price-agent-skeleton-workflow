# 修改提示词（全流程规范）

本文件包含 EDIT 流程的完整执行规范；`SKILL.md` 识别到 EDIT 后只读取本文件即可执行。
禁止读取其他 reference、历史完整提示词或示例文件作为生成范式。格式只能来自本文件第
[4.3 候选提示词格式模板（当前规则同步 + 格式重构硬分支）] 节。

> **命中即执行**：同一轮执行到校验工具和提案；用户说"新建"仍是基于当前版本派生草稿，`writeMode=EDIT`。

---

## 1. 分支判定

**先锁定分支，一次请求只能锁定一个，后续工具响应不得改变已锁定分支：**

1. **精确修改直达分支**：用户给出可唯一定位的增删改（材质组、母子品牌、比价项、规则句、章节等）。
   基础正文结构可用时跳过规则查询，只执行指定局部变更及必要机械一致性调整。
   若同时命中"格式重构硬分支"，则必须执行规则查询重构完整正文。

2. **当前规则同步分支**：用户要求"根据/按照/通过当前或最新的比价项规则、母子品牌关系、材质关系更新/同步提示词"或等价表达。
   触发判别：上述词必须作为**数据来源/依据**出现（如"根据母子品牌关系同步""按最新规则更新"），而非作为修改目标（如"新增一个母子品牌""删除这组品牌"属于精确修改直达分支）。
   本分支内部按用户 input 拆成三个可组合目标：
   - `priceRules=true`：用户明确提到"比价项规则""规则表""当前规则""最新规则"或等价表达；同步比价项、优先级、匹配逻辑、Step3 和输出结构。
   - `brand=true`：用户明确提到"母子品牌""品牌关系""品牌映射"或等价表达；只同步母子品牌映射。
   - `material=true`：用户明确提到"材质关系""材质映射""材质对照表"或等价表达；只同步材质引用和材质对照表。
   用户 input 同时命中多个目标时，锁定这些目标的组合；未命中的目标不得顺手修改、查询关系或在提案中声称已同步。
   命中 `brand` 或 `material` 时仍必须先完整执行类目规则查询，用真实 compareItem 判断门禁并获取对应 `cateNameTree` 作用域。

3. **格式重构硬分支**：查询到的 `promptContent` 明显不符合格式规范（仅为数字、短句、占位内容，或缺少固定章节、真实比价项、完整输出约束）。
   必须执行规则查询并重构为符合格式规范的完整提示词。不得询问用户优化方向，不得保留残缺结构。

4. **无方向普通修改**：仅说"修改/优化提示词"等，既无具体改动，也未显式指定"依据规则/母子品牌/材质"作为同步依据。
   基础正文结构合规时必须跳过规则查询，只依据正文现有内容修复格式、歧义、重复、矛盾、术语、Step 和 JSON 约束；
   不加载比价项、品牌或材质关系，不启用全量同步门禁，不借机补写或改写业务规则。

> **"优化提示词"单独出现时走分支 4；"根据规则/母子品牌/材质优化提示词"走分支 2，并只同步 input 命中的目标。**

> **"母子品牌"按完整业务词识别；不得将"母婴"类目误识别为母子品牌关系。**
> **残缺正文不是外部依赖错误**：`promptContent` 为 `1`、`111` 或其他残缺内容时，禁止声称"未找到配置"，
> 禁止要求用户补充骨架，必须执行格式重构硬分支。

---

## 2. 查询与判断

### 2.1 版本路由

重新读取本轮业务上下文 `promptVersionId`；不大于 0 时立即停止，回复路由错误（EDIT 流程要求 promptVersionId>0），不得在 EDIT 内部切换到 INIT 流程。

### 2.2 补充 ruleGroupId

- 已有 `ruleGroupId`：跳过。
- 缺失且 `agentId<=0` 或为占位符：立即停止，返回业务上下文参数错误。
- 缺失且 `agentId>0`：调用 `query_agent_detail(agentId=上下文.agentId, operator=上下文.operator)`；
  仅 `baseResp.respCode=1` 时读取 `data.card.ruleGroupId`，仍为空则停止。

### 2.3 基础版本查询

仅"当前提示词"允许执行精确查询。用户要求指定其他版本 ID、名称或切换版本时，只回复：
"当前操作仅使用本轮业务上下文中的提示词版本，不支持通过用户文字指定或切换版本。"

`tool_query_prompt_skeleton(ruleGroupId=上下文.ruleGroupId, promptVersionId=上下文.promptVersionId, versionName="", queryOnline=false, queryLatest=false, operator=上下文.operator)`

仅当 `baseResp.respCode=1` 且 `data.promptVersion` 非空时检查版本状态和正文；
正文的 `null`、空串或纯空白均视为空：

- `versionStatus` 为 1、2 或 3 且正文非空：允许作为只读基础派生新草稿；记录 `selectedPromptVersionId`，锁定 `writeMode=EDIT`。
- `versionStatus` 为 1、2 或 3 且正文为空：报告数据异常并停止，不得改走初始化。
- 其他 `versionStatus`：停止并报告未知状态。
- 响应版本 ID 不等于本轮 `promptVersionId`：报告查询数据错误并停止。

> `versionStatus` 枚举：1=草稿（可修改）、2=线上（可作为只读基础派生草稿）、3=归档（可修改）

### 2.4 审计 promptContent

按 4.3 节格式模板完整审计本轮查询所得 `promptContent`：
仅含数字、短句、占位内容，或缺任一固定主体章节（`# 角色`、`# 输入说明`、`# 执行步骤`、`# 判定规则`、`# 图片说明`、`# 输出格式`）、真实比价项和完整输出约束时，立即锁定"格式重构硬分支"；
不得等待用户补方向，不得把残缺原文直接提交校验。

---

## 3. 规则与映射加载（条件性）

仅格式重构硬分支、当前规则同步分支才执行本节。
精确修改和结构合规的无方向普通修改**禁止**因"可能需要"而加载规则或关系。
当前规则同步分支只加载命中目标所需数据：`priceRules` 只需要类目规则；`brand` 需要类目规则和品牌关系；
`material` 需要类目规则和材质关系。多个目标同时命中时合并加载，已加载的数据复用一次，不重复查询。

### 3.1 类目规则查询

1. 调用 `tool_query_category_ids(ruleGroupId=上下文.ruleGroupId, agentId=上下文.agentId, operator=上下文.operator)`。
   要求 `baseResp.respCode=1` 且 `data.categoryIds` 非空；只使用该去重列表，禁止从历史或模型记忆补类目 ID。

2. 遍历全部 `data.categoryIds`，逐个调用 `radar_query_price_rule(categoryId=<当前 categoryId>)`；
   不得漏查、合并 ID 或只查第一项。每次均要求 `result=1`、`error_msg` 为空、
   `data.baseRespInfo.respCode=1`、`data.labelCateRule.isExist=true`，且 `data.labelCateRule` 完整；
   任一 categoryId 不满足即停止，不得用其他类目的规则补齐。

3. 对每次 `radar_query_price_rule` 响应，读取 `data.labelCateRule` 并建立该 categoryId 的规则记录：
   作用域字段只取 `cateNameTree`；规则字段读取
   `ruleTableInfo.ruleTableInfo[]` 中的 `compareItem`、`infoSource`、`compareLogic`，
   以及同级的 `data.labelCateRule.specialRuleContent.ruleContent`。记录该 categoryId 下完整的 `compareItem` 列表，
   即"哪个 categoryId 有哪些比价项"；作用域和规则表非空才可继续。

### 3.2 统一作用域过滤

品牌和材质分别收集各自触发类目的作用域：品牌只收集 `compareItem` 精确等于"品牌"的 categoryId
对应 `cateNameTree` 集合，记为 `brandScopeCateNameTreeSet`；材质只收集 `compareItem` 名称包含"材质"的
categoryId 对应 `cateNameTree` 集合，记为 `materialScopeCateNameTreeSet`。
后续只判断对应集合是否为空，并用对应作用域集合统一过滤候选品牌组或材质行；输出前按品牌组或材质行去重。
不得按每个类目分别输出一份，也不得扩大成"电商全站"。常识只能删除，不得新增、恢复、合并、改名或补别名。
当前规则同步分支中，只有 `brand=true` 才允许进入 3.3；只有 `material=true` 才允许进入 3.4。
未命中的映射目标即使作用域集合非空也不得查询关系、不得改写对应映射表。

### 3.3 主子品牌关系

格式重构硬分支中，`brandScopeCateNameTreeSet` 非空时必须执行本节；为空时不得执行、不得调用工具、
不得输出母子品牌映射表及任何品牌映射说明。当前规则同步分支中，仅当 `brand=true` 且
`brandScopeCateNameTreeSet` 非空时执行本节；`brand=true` 但集合为空时记录为"按真实比价项门禁不适用"，
不得调用工具、不得新增母子品牌映射表。
品牌映射只使用 `brandScopeCateNameTreeSet` 统一过滤母子品牌；不得使用其他 categoryId 的类目，
也不得把品牌范围扩大到全站或其他类目。

调用 `radar_query_brand_relation`；若 schema 明确提供 `keyword`，全量查询传 `keyword=""`；
不得自行添加 `cateName`、`categoryId` 或其他未定义参数。

成功条件：`result=1`、`error_msg` 为空、`data.baseResp.respCode=1`，且 `data.groups` 可解析。

读取：`groups[].mainBrand.id/brandName/isMainBrand/parentId`、`groups[].subBrands[].id/brandName/isMainBrand/parentId`、`groups[].subCount`。

每组必须满足：主品牌名称非空、`mainBrand.isMainBrand=true`、`mainBrand.parentId=0`；
至少一个子品牌，且每个子品牌名称非空、`isMainBrand=false`、`parentId=mainBrand.id`；
`subCount` 必须等于有效 `subBrands` 数量。不满足的整组删除。

过滤顺序（硬约束）：
1. 先遍历并校验 `data.groups` 全部组，不得预先截取前 N 组；
2. 再按 `brandScopeCateNameTreeSet` 对全部有效组做一次统一相关性过滤；
3. 删除跨行业、不确定和测试数据，并按 `mainBrand.id` 或 `mainBrand.brandName` 去重后，才对过滤结果应用 100 组上限；
4. 同一母品牌组不得因为命中多个 `cateNameTree` 重复输出；过滤后不足上限就只输出实际命中组，不得从已删除组回填或用常识补齐。

过滤后 0 组时，静默省略母子品牌映射表，不得输出空结果说明，也不得停止。
进入下一步前记录 `brandGroupCount = min(过滤后组数, 100)`。
禁止添加"及相关品牌""全站常见品牌"等扩大作用域的措辞。**100 组是上限，不是生成目标**。

用户明确指定某组的新增、删除、替换或成员调整属于精确修改，直接执行用户指令，不查询关系库复核。

### 3.4 同款材质关系

格式重构硬分支中，`materialScopeCateNameTreeSet` 非空时必须执行本节；为空时不得执行、不得调用工具、
不得输出材质表及任何材质说明。当前规则同步分支中，仅当 `material=true` 且
`materialScopeCateNameTreeSet` 非空时执行本节；`material=true` 但集合为空时记录为"按真实比价项门禁不适用"，
不得调用工具、不得新增材质表。
材质映射只使用 `materialScopeCateNameTreeSet` 统一过滤材质行；不得使用其他 categoryId 的类目，
也不得把材质范围扩大到全站或其他类目。

调用：`query_material_leaf(operator=上下文.operator)`

成功条件：外层 `result=1` 且 `data.baseResp.respCode=1`。

读取 `data.materialText[]`：每项已是服务端稳定渲染的材质组文本行，格式为
`<完整路径（用·分隔）>：<叶子材质1>、<叶子材质2>...`，例如 `贵金属·金·足金：999金、纯金、黄金`。

模型无需自行解析树结构。

过滤顺序（硬约束）：
1. 先遍历并校验 `materialText[]` 全部文本行，不得预先截取前 N 行；
2. 再按 `materialScopeCateNameTreeSet` 对全部有效行做一次统一相关性过滤；
3. 删除明显无关或不确定的行，并按文本行原文去重后，才对过滤结果应用 50 行上限；
4. 同一材质行不得因为命中多个 `cateNameTree` 重复输出；过滤后不足上限就只输出实际命中行，不得从已删除行回填或用常识补齐。

过滤后的文本行**原样**写入材质对照表附录，不得改写路径或成员名称。

过滤后 0 行时，静默省略材质引用和材质表，不得输出空结果说明，也不得停止。
进入下一步前记录 `materialGroupCount = min(过滤后行数, 50)`。

---

## 4. 生成并校验候选全文

候选全文是内部工具输入，不等于允许向用户展示。

各分支进入本节后的路径不同：

| 分支 | 4.1 | 4.2 | 4.3 格式模板 | 4.4 必检项 |
|---|---|---|---|---|
| 精确修改直达 | 跳过 | 跳过 | 跳过，在原文上局部修改 | 仅执行第 1、3、4、8 条；若局部修改涉及映射表，只校验用户指定局部变更、格式、数量和连续编号 |
| 当前规则同步 | `priceRules=true` 时必须；否则跳过 | `priceRules=true` 或格式重构时必须；否则跳过 | `priceRules=true` 时完整套用；仅 `brand/material=true` 时在原文上只替换对应映射区域 | 按命中目标执行相关项 |
| 格式重构硬分支 | 跳过 | 必须 | 必须完整套用 | 全部 |
| 无方向普通修改 | 跳过 | 跳过 | 跳过，在原文上修格式/歧义 | 仅执行第 1、3、4 条 |

### 4.1 EDIT 目标覆盖账本（当前规则同步分支专用）

当前规则同步分支命中 `priceRules=true` 时，必须在内部建立一张**比价项覆盖核对表**（禁止将其内容复制到 `promptContent`）。
对每个真实 categoryId 下的每个 `ruleTableInfo[]` 依次记录：
`categoryId`、`compareItem`、`cateNameTree`、原始 `infoSource`、稳定投影后的 `expectedPriority`、候选优先级、
该项完整特殊规则、真实 `compareLogic`、`expectedMatch`、候选全部匹配条件、基础正文对应内容、
候选正文对应内容、是否发生变更及原因。

只有每项都有候选落点、优先级与 `expectedPriority` 完全一致、候选匹配与 `expectedMatch` 逐项一致，
才算 `priceRules` 覆盖完成。

当前规则同步分支未命中 `priceRules` 时，不建立比价项覆盖核对表，不重建比价项章节、Step3 或输出结构；
只允许改写命中的 `brand` / `material` 映射区域及其必要的引用行、分隔线和连续编号。

### 4.2 比价项转换账本（当前规则同步 + 格式重构硬分支）

内部转换账本按"每个类目下的每个比价项"分别记录，用 `categoryId` 区分不同类目记录。同名 `compareItem` 出现在多个 categoryId 下时，
必须分别用各自类目记录中的 `cateNameTree`、`infoSource`、`data.labelCateRule.specialRuleContent.ruleContent` 和
`compareLogic` 推导一行账本；只有各 categoryId 推导出的 `expectedPriority` 和 `expectedMatch` 完全一致时，
生成提示词时才可合并为统一写法，否则必须在比价项章节按 `cateNameTree` 分条写。

为每个真实类目记录下的每个 `compareItem` 生成内部转换账本，每行记录：

| 字段 | 说明 |
|---|---|
| `categoryId` | 当前规则所属类目 ID |
| `cateNameTree` | 当前规则所属类目路径原文，例如 `美妆/彩妆` |
| `compareItem` | 真实名称 |
| `infoSource` 原文 | 当前 categoryId 下的原始字段 |
| `expectedPriority` | 按当前 categoryId 的来源过滤规则推导；生成提示词时必须与此完全一致 |
| 清洗后特殊规则 | 删除 SKU 后的当前 categoryId 下 `data.labelCateRule.specialRuleContent.ruleContent` 对应内容 |
| `compareLogic` 原文 | 当前 categoryId 下的原始字段 |
| 别名归一后标准 `compareLogic` | 见下方枚举 |
| `expectedMatch` | 清洗后特殊规则 + 兜底规则；生成提示词时必须与此完全一致 |

账本任一字段推导不通过时禁止继续生成。

#### 来源过滤（生成 expectedPriority）

1. 从左到右扫描完整 `infoSource`，将 `>`、`/`、`=` 都视为来源边界，拆成最小片段。
2. 任一片段只要包含大小写不敏感的连续字样 `sku`，该片段直接删除（即使同时含白名单词）。
3. 对剩余片段做白名单包含匹配：含 `商品属性` → `商品属性`；含 `商品标题` → `商品标题`；
   含 `主图` → `主图`；其他删除。
4. 保留原序（首次出现）去重，用 `>` 连接。过滤后无任何合法来源时，固定写
   `综合判断（商品标题/商品属性/主图同等参考）`。

#### compareLogic 枚举（单一事实源）

先去除输入首尾空白，再按以下受控别名精确归一；命中受控别名时追加对应标准匹配语义原文。
不得只输出逻辑名称，不得自行补充阈值、默认值或行业判断。

**受控别名**：
- `通用匹配逻辑`：左右两侧该比价项一致，或者信息不冲突，或者存在交集，即可匹配。
- `优先完美匹配逻辑`：左右两侧该比价项一致或信息不冲突即可匹配。
- `影响价格匹配逻辑`：左右两侧该比价项一致或信息不冲突即可匹配。
- `严格匹配逻辑`：左侧有该比价项时，右侧必须有且一致才可匹配；左侧有、右侧无时不可匹配；
  左侧无、右侧有时可匹配。
- `货号匹配逻辑`：
  - 货号不一致/缺失 → **禁止**仅凭货号判 different，必须继续看其他项
  - 可忽略符号，去符号后一致即匹配
  - **除货号外其他项均 match=true → 必须判 same**（货号不一致不构成否决）

按类目记录分别判断是否使用货号 Step3：收集 `compareLogic=货号匹配逻辑` 的 categoryId 对应
`cateNameTree` 集合，记为 `partNoStep3CateNameTreeSet`；其余 categoryId 对应 `cateNameTree` 集合记为
`normalStep3CateNameTreeSet`。`partNoStep3CateNameTreeSet` 非空时，必须为该集合生成以下完整综合判定；
`normalStep3CateNameTreeSet` 非空时，仍保留普通 Step3。两个集合都非空时，按适用类目分成两个 Step3 小节，
小节标题中用逗号拼接对应 `cateNameTree` 原文；若只有一个集合非空，则只输出一个 Step3，标题固定为
`**Step3：综合判定**`，不得写"适用于"。不得把货号 Step3 扩大到没有货号匹配逻辑的类目。
生成时将下方普通 Step3 和/或货号 Step3 直接填充到 4.3 完整格式模板的 Step3 指定位置。

```markdown
**Step3：综合判定<仅当 partNoStep3CateNameTreeSet 也非空时追加：（适用于：<normalStep3CateNameTreeSet 中的 cateNameTree 原文，多个用逗号分隔>）>**
—基于所有生效比价项的`match`字段得出`result`：
- 所有生效比价项`match=true`时，必须判定为`same`。
- 任一生效比价项`match=false`时，必须判定为`different`。
- 严禁跳过Step1、Step2直接给出`result`；必须先完成抽取和逐项匹配，再得出最终结论。

**Step3：综合判定<仅当 normalStep3CateNameTreeSet 也非空时追加：（适用于：<partNoStep3CateNameTreeSet 中的 cateNameTree 原文，多个用逗号分隔>）>**
—基于所有生效比价项的`match`字段得出`result`：
- 先排除货号项，仅聚合除货号外的其他生效比价项。
- 除货号外所有生效比价项`match=true`时，必须判定为`same`。
- 除货号外任一生效比价项`match=false`时，必须判定为`different`。
- 货号不一致、缺失或`match=false`绝对不构成否决，不得作为判定`different`的理由。
- 严禁跳过Step1、Step2直接给出`result`；必须先完成抽取和逐项匹配，再得出最终结论。
```

其他非空且未命中标准枚举或受控别名的 `compareLogic`：只保留真实原文，不解释、不近似映射。
空值或去除首尾空白后为 `""`、`"-"`、`"—"` 时，不生成兜底语义。


#### 匹配内容规则（生成 expectedMatch）

`data.labelCateRule.specialRuleContent.ruleContent` 可能存在多种原始格式：以 `$序号. 比价项名称：` 为段标题开始、`$` 为段标题结束，后接换行和段内容；若已去掉 `$` 符号，则以 `序号. 比价项名称：` 行为段起始、下一个 `序号+1.` 行为段结束；也可能是 `比价项名称` 独占一行，下一行开始写该比价项解释，直到下一个比价项名称行、下一个序号标题或文本结束；也可能是一整段自然语言说明，没有明显段标题。

每项"匹配"只能按以下顺序组成：
1. 解析当前 categoryId 下的 `data.labelCateRule.specialRuleContent.ruleContent`：按上述分段规则过滤全文，找到归属于当前 `compareItem` 的内容块；
   若原文是一整段自然语言说明且无法按标题分段，则自行筛选其中与当前 `compareItem` 直接相关的句子或片段；
   取标题行冒号后的内容、独占比价项名称行之后的解释，以及后续正文作为该比价项的专属规则原文；
   只把关于当前比价项的内容放入该比价项"匹配"中，无对应内容块时此步骤结果为空。
   删除原文中所有 SKU 来源、SKU 图片、SKU 标题、SKU 包装等内容；混合句删除后语义不完整时整句删除，
   禁止把 SKU 改写为主图、商品标题或商品属性。
2. 处理该项真实 `compareLogic`：先取当前 categoryId 下当前 `compareItem` 对应的 `compareLogic` 原文；
   若命中上方受控别名，则追加该别名对应的标准匹配语义原文，例如"左右两侧该比价项一致，或者信息不冲突，即可匹配。"；
   若 `compareLogic` 不是上方任何受控别名，但接口返回了非空原文，则直接追加这段原文；
   若接口未返回 `compareLogic` 或内容为空，则不追加匹配逻辑。

无法唯一归属某个比价项但影响多项的特殊规则写入"总原则"，不得新建比价项。
禁止根据比价项名称、旧提示词或模型常识补写规则。

#### 三类特殊比价项

**品牌**：仅当 `brandScopeCateNameTreeSet` 非空时，按 3.3 查询母子品牌关系；只用该集合统一筛选母子品牌，
过滤并去重后放入母子品牌映射表；集合为空时禁止查询和输出。
附录不能代替品牌的特殊规则或兜底。不能因为名称是"品牌"就自行添加母子品牌互认规则。

**材质**：当且仅当 `materialScopeCateNameTreeSet` 非空时，材质章节在"匹配"之后额外增加独立条目
`- **材质对照表**：见下方附录。`；只用该集合统一筛选材质行，过滤并去重后放入材质对照表；
这行不属于"匹配"，不得缩进到"匹配"子项中，不得计入 `expectedMatch`。

**货号**：仅当某个类目记录下该项真实 `compareLogic=货号匹配逻辑` 时，按上方枚举展开，并将该类目的
`cateNameTree` 纳入 `partNoStep3CateNameTreeSet`；这些类目使用货号 Step3，其余类目仍使用普通 Step3。
不能因为 `compareItem` 名称是"货号"就添加货号例外，也不得把货号 Step3 扩大到没有货号匹配逻辑的类目。

#### 品牌映射格式与编号

每组格式固定为 `序号. 母品牌→子品牌1|子品牌2`，每个主品牌组独占一行。
过滤或删除任一组后，必须重新编号：删除第 K 个品牌组时，原第 K+1 个品牌组自动改成第 K 个，
原第 K+2 个品牌组自动改成第 K+1 个，直到最后一组。最终品牌组编号必须是 `1、2、3...N`
连续递增，不得保留断号、重复号、跳号或旧编号。
过滤结果为 0 组时静默省略品牌映射表。

#### 比价项章节格式与编号

每个比价项章节标题格式固定为 `### 序号. 比价项名称`，每个真实 `compareItem` 独占一个章节。
按 `category` 裁剪、过滤、删除或合并任一比价项后，必须重新编号：删除第 K 个章节时，原第 K+1 个章节
自动改成第 K 个，原第 K+2 个章节自动改成第 K+1 个，直到最后一项。最终章节编号必须是 `1、2、3...N`
连续递增，不得保留断号、重复号、跳号或旧编号。

### 4.3 候选提示词格式模板（当前规则同步 + 格式重构硬分支）

以下模板是生成候选提示词的直接依据。所有尖括号内容必须替换为本轮真实数据；条件不适用时删除
对应行，不得留下占位符、`TODO` 或空标题。
**禁止将本节的规范说明句子复制、改写或解释后写入 `promptContent`**；只有模板围栏内从 `# 角色` 至
`# 输出格式`（含 8 条字段规则）的内容才是正文结构。

**输入说明表生成规则**（仅用于生成，不得写入 `promptContent`）：遍历所有有效 `compareItem`，以比价项名称为行；同名 `compareItem` 出现在多个 categoryId 下时合并到同一行，各 `cateNameTree` 原文用逗号拼接；只出现在部分 categoryId 下的只填那几个 categoryId 的 `cateNameTree`，不得把无关类目填进来；同名 `compareItem` 不出现重复行；`cateNameTree` 已代表该路径下的全部子类目。

**比价项章节生成规则**（仅用于生成，不得写入 `promptContent`）：生成 `## 比价项` 前，必须先按
`compareItem` 对 4.2 内部转换账本分组；每个 `compareItem` 只生成一个章节，不得按 `categoryId`
重复生成章节。组内所有 categoryId 的 `expectedPriority` 和 `expectedMatch` 均逐字符一致时，输出统一的
`优先级` 和 `匹配`；只要任一 categoryId 的 `expectedPriority` 或 `expectedMatch` 不一致，必须分别按
`cateNameTree` 输出 `优先级（<cateNameTree>）` 和 `匹配（<cateNameTree>）`。下方模板中的"情况一/情况二"
只是内部选择逻辑，最终 `promptContent` 中不得出现"情况一"、"情况二"或未替换的示例类目。

````markdown
# 角色

你是一个电商同款商品判定专家，负责判断左侧商品与右侧竞品是否为「同款」。严格按照下方【执行步骤】和【判定规则】逐项核查，输出结构化JSON。

---

# 输入说明

左侧商品信息中包含`category`字段，格式为多级类目路径（用`>`分隔），例如
`"服装>上衣>T恤"`。**比价项的适用范围由`category`决定**：

|比价项|适用类目（`category`命中任一前缀即生效）|
|--------|-------------------------------------------|
|<真实比价项1>|<该比价项所属 categoryId 的 cateNameTree 原文；多个类目用逗号分隔>|
|<真实比价项2>|<该比价项所属 categoryId 的 cateNameTree 原文；多个类目用逗号分隔>|

**不适用的比价项不参与判定**：不抽取、不输出到`extracted`、不计入`match`综合判定。

---

# 执行步骤

**Step0：读取category**
—根据上表确定本次需要判定的比价项子集（默认全部<本轮生效比价项数量>项，剔除不适用项）

**Step1：逐项抽取**
—仅对生效比价项，从左右商品的商品属性、商品标题或主图中提取`value`和`source`，填入`extracted`的`left`/`right`字段

**Step2：逐项匹配**
—对每个生效比价项，按下方匹配规则判定`match`（`true`/`false`）和`reason`

<按 4.2 "货号匹配逻辑"下方的 Step3 生成规则，在此填入普通 Step3、货号 Step3 或两者；若两者都存在才在标题中标明适用 cateNameTree。>

<若本轮真实特殊规则明确规定综合判定例外，在此追加；没有则删除本行。>

---

# 判定规则

## 总原则

1. <来自当前规则的生效比价项综合关系>
2. 高优先级来源有明确信息时，不得用低优先级来源覆盖；多来源信息相同时`source`用`+`拼接。
3. `result`和总`reason`只能基于生效比价项的`match`字段，不得引入比价项以外的信息。
4. `value`必须来自商品属性、商品标题或主图中的明确信息，不得从编码、数字前缀或模糊线索推断。
5. <来自当前规则的特殊例外；没有则删除本行>

## 比价项（最多<本轮生效比价项数量>项，按`category`裁剪）

### 1. <比价项名称>

情况一：该比价项在所有涉及的 categoryId 下 `expectedPriority` 和 `expectedMatch` 完全一致：
- **优先级**：<统一的 expectedPriority>
- **匹配**：<统一的 expectedMatch>

情况二：该比价项在不同 categoryId 下 `expectedPriority` 或 `expectedMatch` 不同，按类目分条写：
- **优先级（<cateNameTree-A>）**：<该类目的 expectedPriority>
- **优先级（<cateNameTree-B>）**：<该类目的 expectedPriority>
- **匹配（<cateNameTree-A>）**：<该类目的 expectedMatch>
- **匹配（<cateNameTree-B>）**：<该类目的 expectedMatch>

按本轮查询结果为每个生效比价项生成一个章节，并保持连续编号。

映射章节必须先在内部得到 `brandGroupCount` 和 `materialGroupCount`，然后一次性拼接：
- `brandGroupCount=0 && materialGroupCount=0`：不输出任何映射相关字符；
- `brandGroupCount>0`：输出 `## 映射表` 及非空母子品牌表；
- `materialGroupCount>0`：输出 `## 映射表` 及非空材质表；
- 两者均大于 0 时，在两个非空子表之间输出 `---`。

---

# 图片说明

第1张=左侧商品主图，第2张=右侧商品主图。图片辅助判断<本轮允许从主图判断的生效比价项>。信息来源仅限三种：商品属性、商品标题、主图。高优先级来源已有明确信息时以其为准。

---

# 输出格式

严格只输出JSON，禁止额外文字、注释或```符号。

**`extracted`中只包含生效的比价项**：不适用的比价项按`category`裁剪后不出现在`extracted`对象中，不抽取、不输出空对象、不参与综合判定。

```json
{
  "result": "same或different",
  "reason": "一句话关键判定依据",
  "confidence": 0.0,
  "extracted": {
    "<生效比价项1>": {
      "left": { "value": "", "source": "" },
      "right": { "value": "", "source": "" },
      "match": true,
      "reason": ""
    },
    "<生效比价项2>": {
      "left": { "value": "", "source": "" },
      "right": { "value": "", "source": "" },
      "match": true,
      "reason": ""
    },
    "key_diff_point": ""
  }
}
```

字段规则：

- `extracted`仅包含按`category`生效的比价项；不适用项整体省略，不输出空对象。
- `confidence`取值范围为`0.0`～`1.0`（含边界）的浮点数，越大表示判定把握越高；必须输出具体数字字面量（如`0.85`），严禁输出`0.0~1.0`、`0.0-1.0`等区间记法。
- `value`抽取不到时填`"缺失"`，`source`填`""`。
- `source`仅允许三种标准来源：`商品属性`、`商品标题`、`主图`；其他来源填`""`。
- `match=true`表示匹配或不影响判断，`match=false`表示不匹配；`match=false`必须能由对应比价项规则解释。
- 多来源信息相同时，`source`使用`+`拼接，如`"商品属性+商品标题"`。
- 材质`value`如果分多部位，必须按部位输出，格式为`帮面:XX;鞋底:YY;鞋垫:ZZ`，缺失的部位省略；服装等其他类目使用本轮规则规定的真实部位名。
- `key_diff_point`始终保留；`result="same"`时必须为空字符串，`result="different"`时填写导致否决的关键差异点。
````

`字段规则：` 标题及其下方 8 条规则是候选提示词的固定必选正文，生成时必须逐条保留，不得概括、合并、改写或省略。
JSON 围栏结束不代表 `# 输出格式` 章节结束，只有 8 条字段规则全部输出后该章节才完整。

**章节结构要求**：完整提示词必须包含以下章节并严格保持顺序：
`# 角色` → `# 输入说明` → `# 执行步骤` → `# 判定规则`（含`## 总原则`、`## 比价项`、`## 映射表`条件性） → `# 图片说明` → `# 输出格式`。
一级章节之间使用独立的 `---` 分隔；除条件性 `## 映射表` 外，不得改名、合并、调序或省略固定章节。

**映射表写法**：
- 母子品牌：标题 `### 母子品牌映射表（<真实适用类目>）`，禁止写动态组数；每行 `序号. 母品牌→子品牌1|子品牌2`；仅当材质表也需输出时，最后一条后加 `---` 分隔。
- 材质：标题固定 `### 材质对照表（同组内可互匹配，跨组不可）`；每组直接使用 `data.materialText[]` 过滤后的文本行原样输出；最多 50 组，实际不足时不补齐。

### 4.4 调用校验前的必检项

1. 从 4.3 格式模板中完整复制 `# 输出格式` 固定块至第 8 条 `key_diff_point` 规则，不因长度或类目裁剪。
2. 执行过 4.2 时，必须生成内部转换账本；命中受控别名时候选必须是标准枚举的完整展开，
   不得残留别名原文。仅同步 `brand/material` 且 `priceRules=false` 时跳过本项。
3. JSON 围栏后保留 `字段规则：` 和完整 8 条规则。
4. 对母子品牌映射表和比价项序列分别从 1 开始整体重写连续编号；重写后再次扫描，若仍断号则继续修复。
5. 映射门禁：
   - 格式重构硬分支，或当前规则同步分支命中 `brand=true` 时：`brandScopeCateNameTreeSet` 非空则必须有本轮品牌关系查询与过滤记录；
     过滤后非空输出映射表，为空静默省略；品牌映射必须通过 3.3 的完整过滤、数量、格式和连续编号门禁。
     `brand=false` 时不得查询品牌关系，不得改写母子品牌映射表。
   - 格式重构硬分支，或当前规则同步分支命中 `material=true` 时：`materialScopeCateNameTreeSet` 非空则对应比价项章节必须存在固定行
     `- **材质对照表**：见下方附录。`，且下方必须存在材质对照表附录，二者缺一不得调用校验；
     `materialGroupCount>0` 时输出真实行，为 0 时静默省略引用和材质表。`material=false` 时不得查询材质关系，
     不得改写材质引用行或材质表。
   - 需要重建映射章节时，若参与重建的 `brandScopeCateNameTreeSet` 和 `materialScopeCateNameTreeSet` 均为空，
     省略整个 `## 映射表` 章节。未命中的映射目标保留基础正文原有内容，不参与本条判断。
   - 候选全文出现"无符合""无可输出""未命中""暂无"或任何映射门控说明，均视为空表泄漏，
     必须删除后重新生成，禁止调用校验。
   - 母子品牌标题不得包含"共 N 组"等动态组数。
6. 全文约 1 万字；接近或超过时只删除重复解释、重复示例、同义表达；
   不得裁掉真实比价项、规则边界、例外、固定章节或 JSON 字段，不得硬截断。
7. **当前规则同步分支专用硬校验**：
   - `priceRules=true` 时，比价项覆盖核对表中真实类目记录、比价项数量、名称、顺序与各 categoryId 下 `ruleTableInfo[]` 完全一致；
   - `priceRules=false` 时，不得生成比价项覆盖核对表，不得在提案中声称已同步或核对比价项规则；
   - 候选全文通过本次命中目标相关的全部硬校验；
   - 用户请求的每个命中目标均记录为"已修改""已核对且一致"或"按真实比价项门禁不适用"，不得静默忽略；
   - 用户未命中的目标必须记录为"未请求，保持原文"，不得顺手修改或在提案中列为已同步；
   - 若服务端 Diff 只包含映射变化，只有在 `priceRules=false` 且用户只请求品牌/材质同步时允许；若 `priceRules=true`，
     则必须已逐项证明所有比价项规则完全一致，否则禁止形成提案。
   - 任一项不满足都不得调用校验，不得形成只修改映射表的不完整提案。
8. 精确修改直达分支若涉及映射表，只执行用户指定局部变更及必要机械一致性校验：
   不调用关系 MCP，不要求 `brandScopeCateNameTreeSet` 或 `materialScopeCateNameTreeSet`；
   但仍必须保持映射格式、数量上限、非空标题、连续编号、材质引用行与材质表互相一致。任一条件不满足禁止调用校验。

### 4.5 硬门禁

1. **来源投影门禁**：每个 `compareItem` 的 `expectedPriority` 必须通过来源过滤规则推导，不得省略、
   重排或补入原文不存在的来源；`商品属性栏（站外）` 必须归一为 `商品属性`；
   过滤后来源顺序和合法来源集合完全由算法决定，不得凭印象写固定值。
2. **关系响应门禁**：品牌、材质正文只能来自本轮真实关系接口响应及过滤结果；
   材质直接使用本轮 `data.materialText[]` 过滤后的文本行，不得修改或补充。
3. **候选泄漏门禁**：候选全文不得出现任何生成器门控、条件输出或内部算法说明，
   也禁止用"无符合""未命中""暂无"等空结果正文占位。

### 4.6 调用校验

`tool_validate_prompt_skeleton(promptContent=<候选全文>, operator=上下文.operator, conversationId=上下文.localConversationId, basePromptVersionId=selectedPromptVersionId, ruleGroupId=上下文.ruleGroupId, agentId=上下文.agentId)`

`ruleGroupId`、`agentId` 至少一个大于 0；`basePromptVersionId` 使用 2.3 节记录的 `selectedPromptVersionId`。

- `baseResp.respCode=1 && data.valid=true && data.diffRecordId>0`：成功，锁定本次 `promptContent`，保留 `diffRecordId`、`diffContent`。
  `data.diffContent` 在 EDIT 中不允许为空。
- `baseResp.respCode=1 && data.valid=true && data.diffRecordId<=0`：立即异常结束，回复
  "提示词校验未生成有效修改建议（diffRecordId=0），无法形成提案。"禁止重试。
- `data.valid=false`：只按 `data.errors` 修正报错处，其他正文不变，最多重试 2 次。
- 其他情况或重试后仍失败：只返回最终错误，不展示提案、不写入。

Diff 只取服务端 `data.diffContent`，禁止自行书写。

---

## 5. 展示提案

以下是修改提案唯一允许的可见回复协议；必须以 `## 提示词修改提案` 为第一行，
逐项、按序套用，模板前后不得添加任何文字。所有尖括号字段必须替换成本轮真实值：

````markdown
## 提示词修改提案
- Agent：`<名称；查不到时写ID>`
- 基础提示词：`<名称>`（ID：`<ID>`）
- 当前提示词来源：页面左侧选定的提示词（以当前业务上下文 `promptVersionId` 为准）
- 修改建议 ID（diff_id）：`<原样引用 tool_validate_prompt_skeleton 返回的非零 data.diffRecordId>`
- 状态：尚未保存
### 合理性判断
- 修改目标：<用户的原始要求或优化方向>
- 规则与映射依据：<支持本次修改的规则依据>
- 影响与风险：<适用范围、冲突和需要回归的项>
- 不修改部分：<明确保留未改动的规则>
### Diff
```diff
<原样引用 tool_validate_prompt_skeleton 返回的 data.diffContent>
```
完整提示词已生成并通过格式校验，未在此展开。需查看请回复「展开完整提示词」。
以上仅为修改提案。确认后创建新提示词草稿，不覆盖基础版本。
确认无误请回复：**确认创建提示词草稿**。
````

**输出硬校验**（发送前逐项检查，任一项不满足内部重组）：
1. 保留标题、字段、顺序和四条"合理性判断"；精确修改的依据写"用户明确指定"，不得声称来自未调用的 MCP。
2. `### Diff` 后必须紧跟语言标记为 `diff` 的代码围栏，禁止使用 `text`、`markdown` 或无语言围栏。
3. 围栏内容逐字符复制 `data.diffContent`；不得摘要、解释、改写、补行、删行、重新编号。
4. 只有非零 `diffRecordId` 才能形成提案；除用户要求展开全文外禁止在模板外追加完整提示词。
5. 精确修改提案只陈述用户明确指定的局部目标；不得声称执行了未触发的全量规则、适用类目、品牌或材质核对。

#### 展开完整提示词

用户紧邻有效提案回复"展开完整提示词"、"查看完整提示词"或等价表达时，只执行本分支。上一条必须来自本轮真实 `tool_validate_prompt_skeleton` 响应，且
`data.valid=true`、`data.diffRecordId>0`、锁定的 `promptContent` 非空；否则只返回
"上一条不是可展开的有效提示词提案，请重新发起提示词修改。"

满足条件时，不调用 MCP、不重新查询、不重新生成、不解释，只逐字符输出锁定的 `promptContent`。使用四反引号 `text` 围栏，确保正文内部三反引号保持原样：

`````markdown
````text
<逐字符复制本轮 tool_validate_prompt_skeleton 已锁定的 promptContent>
````
`````

展开只用于查看，不构成确认。展开后原提案不能再直接确认；随后要求写入时须重新生成提案。

---

## 6. 确认后写入草稿

#### 确认门禁

本节是确认意图命中后的唯一执行路径。进入后禁止输出任何自然语言或执行其他工具，必须先按下方参数调用 `tool_edit_prompt_skeleton`。

只允许确认可见的上一条助手提案。若上一条 assistant 可见消息包含以下三项，模型侧门禁通过，
必须使用该非零 ID 调用写入工具：
1. `## 提示词修改提案`
2. 非零修改建议 ID（`diff_id`）
3. 明确确认话术

模型不得推测数据库消息序号、相邻状态、Diff 状态或基础版本关联，也不得在调用前输出
"上一条不是有效提案""提案已失效""请重新发起"。这些事实全部由服务端写入工具校验。
只有上一条 assistant 可见消息客观缺少上述任一项时，才允许不调用工具并说明缺少哪一项；
不得笼统声称提案无效。没有本轮 `tool_edit_prompt_skeleton` 真实响应时，禁止生成任何服务端拒绝、
确认失败、版本不匹配或提案失效结论。

#### 写入参数

- `promptDiffRecordId`：紧邻提案本轮 `tool_validate_prompt_skeleton` 返回的非零 `data.diffRecordId`。
- `promptVersionId`：进入确认写入时，从**当前确认回合注入的业务上下文块**读取 `promptVersionId`
  原始展开值，并记录为 `confirmationPromptVersionId`。原始值显式为非负整数时直接使用；
  字段未注入、为空或仍为模板占位符时，统一令 `confirmationPromptVersionId=0`。不得从上一条提案的
  "基础提示词 ID"、历史消息、上一轮工具参数、Diff 记录、版本名称、`newPromptVersionId` 或模型记忆补值。
  只要存在有效非零 `promptDiffRecordId`，就不得因 `promptVersionId` 缺失而阻止确认写入，必须以
  `promptVersionId=0` 调用；Diff 与版本关联由服务端校验。仅当上下文显式提供负数或非数字值时停止，
  只回复："当前确认回合提供的 promptVersionId 非法，未执行提示词写入。"
- `sourceType=2`（模型优化）
- 不传 `promptContent`；服务端以 Diff 记录正文为准。

`promptVersionId>0` 时先按 2.3 节精确查询本轮基础版本并记录 `versionName/id/versionNo`；失败则不写入。
为 0 时基础提示词记为"无（初始化）"，但 EDIT 提案确认通常应由服务端拒绝其基础版本不匹配。

`tool_edit_prompt_skeleton(ruleGroupId=上下文.ruleGroupId, promptVersionId=confirmationPromptVersionId, operator=上下文.operator, sourceType=2, promptDiffRecordId=<紧邻提案Diff ID>)`

调用前逐字段生成 `writeParameterLedger`：
- `promptDiffRecordId` 来源必须是上一条可见提案；
- `promptVersionId` 来源必须是当前确认回合业务上下文的显式非负整数，或由缺失、空值、模板占位符按规则规范化得到的 `0`；
- `ruleGroupId`、`operator` 来源必须是当前确认回合业务上下文；
- `sourceType` 固定由原提案的 EDIT 模式决定为 `2`。

任一字段来源不符合时禁止调用；但当前确认回合缺失 `promptVersionId` 属于合法规范化场景，不得阻止调用。
禁止因为提案中存在基础版本 ID 就把该 ID 当作当前确认回合的 `promptVersionId`；版本关联以服务端校验结果为准。

调用返回前禁止产生用户可见文字。调用后只能依据本次真实 `baseResp`、`data`、`result`、`error_msg` 输出：
成功时使用下方成功模板；业务拒绝时逐字转述真实原因；调用失败或响应不完整时报告真实工具错误。
禁止用模型生成的理由替代工具响应。

服务端校验 Diff 基础版本关联关系。不匹配、已处理或失效时据实报告，不切换版本、不重新校验、不传正文、不重试。
超时或响应不完整时，可用 `data.card.latestDraftPromptVersionId` 只读核实；不能唯一确认就报告结果未知，禁止重试写入。

#### 成功回复模板

```markdown
## 提示词草稿创建成功
- 修改建议 ID（diff_id）：`<本次 promptDiffRecordId>`
- 基础提示词：`<名称>`（ID：`<selectedPromptVersionId>`）
- 新提示词：`<data.newPromptName>`（ID：`<data.newPromptVersionId>`，版本号：`<data.versionNo>`）
- 状态：草稿已创建
```

`data.promptExists=true` 时新提示词字段仍取响应，状态改为"已存在，未新建"。不得总结变更内容；
返回的新 ID 仅展示，不更新上下文。
