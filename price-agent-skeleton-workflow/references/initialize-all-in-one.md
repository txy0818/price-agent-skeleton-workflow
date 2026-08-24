# 初始化提示词（全流程规范）

本文件只定义从初始化意图到展示初始化提案的完整执行规范；提案后的"展开完整提示词"、
"确认/同意/保存/创建草稿"由 `SKILL.md` 重新识别意图并路由到对应流程。初始化生成过程中禁止读取
其他 reference 或任何历史完整提示词作为生成范式。格式只能来自本文件第
[5.2 生成候选 promptContent（完整格式模板）] 节。

首条可见回复只能是初始化提案、已存在结果或明确错误。

---

## 路由说明

对任意提示词写操作，只要 `promptVersionId` 缺失、为 0 或仍为模板占位符就进入本流程；
用户具体使用新建、创建、初始化、修改、优化等哪种说法都不影响路由。本流程没有基础版本。

---

## 执行顺序

### 1. 确认版本路由

重新读取本轮业务上下文 `promptVersionId`；该值大于 0 时停止并按入口重新路由 EDIT；
否则锁定本轮初始化值为 0。

### 2. 补齐 ruleGroupId

- 已有 `ruleGroupId`：跳过。
- 缺失且 `agentId<=0`、缺失或为占位符：立即停止，返回业务上下文参数错误，不调用工具。
- 缺失且 `agentId>0`：调用 `query_agent_detail(agentId=上下文.agentId, operator=上下文.operator)`；
  仅 `baseResp.respCode=1` 时读取 `data.card.ruleGroupId`，仍为空则停止。

### 3. 查询提示词是否已存在

只调用一次：

`tool_query_prompt_skeleton(ruleGroupId=<ruleGroupId>, promptVersionId=0, versionName="", queryOnline=false, queryLatest=true, operator=<operator>)`

- 返回非空 `data.promptVersion`：告知 `data.promptVersion.versionName` 和
  `data.promptVersion.promptVersionId` 后停止；不加载规则、不生成或写入。
- 返回 `baseResp.respCode=21` 且去除首尾空白后的
  `baseResp.respDesc="未找到匹配的骨架版本"`：继续执行第 4 步。
- `respCode=21` 但 `respDesc` 不是上述精确文本：按异常响应停止。
- 调用失败、超时、响应不完整或其他非成功响应：立即停止，不重试。

### 4. 加载规则与映射

严格按本节顺序执行，不得跳过、合并 ID 或仅查询子集。

#### 4.1 类目规则查询

1. 调用 `tool_query_category_ids(ruleGroupId=上下文.ruleGroupId, agentId=上下文.agentId, operator=上下文.operator)`。
   要求 `baseResp.respCode=1` 且 `data.categoryIds` 非空；只使用该去重列表，
   禁止从历史、名称、提示词或模型记忆补类目 ID。响应中的 `ruleGroupId` 与当前上下文不一致时停止。

2. 遍历全部 `data.categoryIds`，逐个调用 `radar_query_price_rule(categoryId=<当前 categoryId>)`；
   不得漏查、合并 ID 或只查第一项。每次均要求 `result=1`、`error_msg` 为空、
   `data.baseRespInfo.respCode=1`、`data.labelCateRule.isExist=true`，且 `data.labelCateRule` 完整；
   任一 categoryId 不满足即停止，不得用其他类目的规则补齐。

3. 对每次 `radar_query_price_rule` 响应，读取 `data.labelCateRule` 并建立该 categoryId 的规则记录：
   作用域字段只取 `cateNameTree`；规则字段读取
   `ruleTableInfo.ruleTableInfo[]` 中的 `compareItem`、`infoSource`、`compareLogic`，
   以及同级的 `data.labelCateRule.specialRuleContent.ruleContent`。记录该 categoryId 下完整的 `compareItem` 列表，
   即"哪个 categoryId 有哪些比价项"；作用域和规则表非空才可继续。

#### 4.2 统一作用域过滤

品牌和材质分别收集各自触发类目的作用域：品牌只收集 `compareItem` 精确等于"品牌"的 categoryId
对应 `cateNameTree` 集合，记为 `brandScopeCateNameTreeSet`；材质只收集 `compareItem` 名称包含"材质"的
categoryId 对应 `cateNameTree` 集合，记为 `materialScopeCateNameTreeSet`。
后续只判断对应集合是否为空，并用对应作用域集合统一过滤候选品牌组或材质行；输出前按品牌组或材质行去重。不得按每个类目分别输出一份，
也不得扩大成"电商全站"。
常识只能删除，不得新增、恢复、合并、改名或补别名。

#### 4.3 主子品牌关系

`brandScopeCateNameTreeSet` 非空时必须执行本节；为空时不得执行、不得调用工具、
不得输出母子品牌映射表及任何品牌映射说明（双向硬门禁）。
品牌映射只使用 `brandScopeCateNameTreeSet` 统一过滤母子品牌；不得使用其他 categoryId 的类目，
也不得把品牌范围扩大到全站或其他类目。

调用 `radar_query_brand_relation`；若 schema 明确提供 `keyword`，全量查询传 `keyword=""`；
不得自行添加 `cateName`、`categoryId` 或其他未定义参数。

成功条件：`result=1`、`error_msg` 为空、`data.baseResp.respCode=1`，且 `data.groups` 可解析。

读取字段：
- `groups[].mainBrand.id/brandName/isMainBrand/parentId`
- `groups[].subBrands[].id/brandName/isMainBrand/parentId`
- `groups[].subCount`

每组必须满足：主品牌名称非空、`mainBrand.isMainBrand=true`、`mainBrand.parentId=0`；
至少一个子品牌，且每个子品牌名称非空、`isMainBrand=false`、`parentId=mainBrand.id`；
`subCount` 必须等于有效 `subBrands` 数量。不满足的整组删除。

过滤顺序（硬约束）：
1. 先遍历并校验 `data.groups` 全部组，不得预先截取前 N 组；
2. 再按 `brandScopeCateNameTreeSet` 对全部有效组做一次统一相关性过滤；
3. 删除跨行业、不确定和测试数据，并按 `mainBrand.id` 或 `mainBrand.brandName` 去重后，才对过滤结果应用 100 组上限；
4. 同一母品牌组不得因为命中多个 `cateNameTree` 重复输出；过滤后不足上限就只输出实际命中组，不得从已删除组回填或用常识补齐。

过滤后 0 组时，静默省略母子品牌映射表，不得输出空结果说明，也不得停止。
进入校验工具前记录 `brandGroupCount = min(过滤后组数, 100)`。
禁止添加"及相关品牌""全站常见品牌"等扩大作用域的措辞。**100 组是上限，不是生成目标**；
返回或过滤后只有 3 组就只写 3 组。

#### 4.4 同款材质关系

`materialScopeCateNameTreeSet` 非空时必须执行本节；为空时不得执行、不得调用工具、
不得输出材质表及任何材质说明（双向硬门禁）。
材质映射只使用 `materialScopeCateNameTreeSet` 统一过滤材质行；不得使用其他 categoryId 的类目，
也不得把材质范围扩大到全站或其他类目。

调用：`tool_query_material_leaf(operator=上下文.operator)`

成功条件：`baseResp.respCode=1`。

读取 `materialText[]`：每项已是服务端稳定渲染的材质组文本行，格式为
`<一级路径·...·父节点名>：<叶子材质1>、<叶子材质2>...`，
例如 `贵金属·金·足金：999金、纯金、黄金`。

模型无需自行解析树结构。

过滤顺序（硬约束）：
1. 先遍历并校验 `materialText[]` 全部文本行，不得预先截取前 N 行；
2. 再按 `materialScopeCateNameTreeSet` 对全部有效行做一次统一相关性过滤；
3. 删除明显无关或不确定的行，并按文本行原文去重后，才对过滤结果应用 50 行上限；
4. 同一材质行不得因为命中多个 `cateNameTree` 重复输出；过滤后不足上限就只输出实际命中行，不得从已删除行回填或用常识补齐。

过滤后的文本行**原样**写入材质对照表附录，不得改写路径或成员名称。

过滤后 0 行时，静默省略材质引用和材质表，不得输出空结果说明，也不得停止。
进入校验工具前记录 `materialGroupCount = min(过滤后行数, 50)`。

### 5. 生成并校验候选全文

候选全文是内部工具输入，不等于允许向用户展示。生成前必须先完成以下步骤。

#### 5.1 比价项转换（账本）

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

##### 来源过滤（生成 expectedPriority）

1. 从左到右扫描完整 `infoSource`，将 `>`、`/`、`=` 都视为来源边界，拆成最小片段。
2. 任一片段只要包含大小写不敏感的连续字样 `sku`，该片段直接删除（即使同时含白名单词）。
3. 对剩余片段做白名单包含匹配：含 `商品属性` → `商品属性`；含 `商品标题` → `商品标题`；
   含 `主图` → `主图`；其他删除。
4. 保留原序（首次出现）去重，用 `>` 连接。过滤后无任何合法来源时，固定写
   `综合判断（商品标题/商品属性/主图同等参考）`。

品牌来源链强制解析示例（不是候选内容，仅说明算法）：
`sku图中商品本身品牌信息=比价系统品牌/等级（站内）/商品属性栏（站外）>主图文字说明>商品标题>商品详情`
→ `sku图中商品本身品牌信息(删除)`、`比价系统品牌(删除)`、`等级（站内）(删除)`、
  `商品属性栏（站外）→商品属性(保留)`、`主图文字说明→主图(保留)`、
  `商品标题(保留)`、`商品详情(删除)`
→ `expectedPriority = 商品属性>主图>商品标题`

##### compareLogic 枚举（单一事实源）

先去除输入首尾空白，再按以下受控别名精确归一；命中受控别名时追加对应标准匹配语义原文。
不得只输出逻辑名称，不得自行补充阈值、默认值或行业判断。

**受控别名**：
- `通用匹配逻辑`：左右两侧该比价项一致，或者信息不冲突，或者存在交集，即可匹配。
- `优先完美匹配逻辑`：左右两侧该比价项一致或信息不冲突即可匹配。
- `影响价格匹配逻辑`：左右两侧该比价项一致或信息不冲突即可匹配。
- `严格匹配逻辑`：左侧有该比价项时，右侧必须有且一致才可匹配；左侧有、右侧无时不可匹配；
  左侧无、右侧有时可匹配。
- `货号匹配逻辑`：
  - 货号不一致/缺失→**禁止**仅凭货号判different，必须继续看其他项
  - 可忽略符号，去符号后一致即匹配
  - **除货号外其他项均match=true→必须判same**（货号不一致不构成否决）

按类目记录分别判断是否使用货号 Step3：收集 `compareLogic=货号匹配逻辑` 的 categoryId 对应
`cateNameTree` 集合，记为 `partNoStep3CateNameTreeSet`；其余 categoryId 对应 `cateNameTree` 集合记为
`normalStep3CateNameTreeSet`。`partNoStep3CateNameTreeSet` 非空时，必须为该集合生成以下完整综合判定；
`normalStep3CateNameTreeSet` 非空时，仍保留普通 Step3。两个集合都非空时，按适用类目分成两个 Step3 小节，
小节标题中用逗号拼接对应 `cateNameTree` 原文；若只有一个集合非空，则只输出一个 Step3，标题固定为
`**Step3：综合判定**`，不得写"适用于"。不得把货号 Step3 扩大到没有货号匹配逻辑的类目。
生成时将下方普通 Step3 和/或货号 Step3 直接填充到 5.2 完整格式模板的 Step3 指定位置。

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


##### 匹配内容规则（生成 expectedMatch）

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

##### 三类特殊比价项

**品牌**：仅当 `brandScopeCateNameTreeSet` 非空时，按 4.3 查询母子品牌关系；只用该集合统一筛选母子品牌，
过滤并去重后放入母子品牌映射表；集合为空时禁止查询和输出。
附录不能代替品牌的特殊规则或兜底。不能因为名称是"品牌"就自行添加母子品牌互认或直接判同规则。

**材质**：当且仅当 `materialScopeCateNameTreeSet` 非空时，材质章节在"匹配"之后额外增加独立条目
`- **材质对照表**：见下方附录。`；只用该集合统一筛选材质行，过滤并去重后放入材质对照表；
这行不属于"匹配"，不得缩进到"匹配"子项中，不得计入 `expectedMatch`。

**货号**：仅当某个类目记录下该项真实 `compareLogic=货号匹配逻辑` 时，按上方枚举展开，并将该类目的
`cateNameTree` 纳入 `partNoStep3CateNameTreeSet`；这些类目使用货号 Step3，其余类目仍使用普通 Step3。
不能因为 `compareItem` 名称是"货号"就添加货号例外，也不得把货号 Step3 扩大到没有货号匹配逻辑的类目。

##### 品牌映射格式与编号

每组格式固定为 `序号. 母品牌→子品牌1|子品牌2`，每个主品牌组独占一行。
过滤或删除任一组后，必须重新编号：删除第 K 个品牌组时，原第 K+1 个品牌组自动改成第 K 个，
原第 K+2 个品牌组自动改成第 K+1 个，直到最后一组。最终品牌组编号必须是 `1、2、3...N`
连续递增，不得保留断号、重复号、跳号或旧编号。
过滤结果为 0 组时静默省略品牌映射表。

##### 比价项章节格式与编号

每个比价项章节标题格式固定为 `### 序号. 比价项名称`，每个真实 `compareItem` 独占一个章节。
按 `category` 裁剪、过滤、删除或合并任一比价项后，必须重新编号：删除第 K 个章节时，原第 K+1 个章节
自动改成第 K 个，原第 K+2 个章节自动改成第 K+1 个，直到最后一项。最终章节编号必须是 `1、2、3...N`
连续递增，不得保留断号、重复号、跳号或旧编号。

#### 5.2 生成候选 promptContent（完整格式模板）

本节规定候选提示词的完整结构。所有生成约束（"使用边界""生成原则"等）均为内部规范，
**禁止将这些规范句子复制、改写或解释后写入 `promptContent`**。只有下方模板中
从 `# 角色` 至 `# 输出格式`（含 8 条字段规则）的内容才是正文结构。

以下模板是生成候选提示词的直接依据。所有尖括号内容必须替换为本轮真实数据；条件不适用时删除
对应行，不得留下占位符、`TODO` 或空标题。
**禁止将本节的规范说明句子复制、改写或解释后写入 `promptContent`**；只有模板围栏内从 `# 角色` 至 `# 输出格式`（含 8 条字段规则）的内容才是正文结构。

**输入说明表生成规则**（仅用于生成，不得写入 `promptContent`）：遍历所有有效 `compareItem`，以比价项名称为行；同名 `compareItem` 出现在多个 categoryId 下时合并到同一行，各 `cateNameTree` 原文用逗号拼接；只出现在部分 categoryId 下的只填那几个 categoryId 的 `cateNameTree`，不得把无关类目填进来；同名 `compareItem` 不出现重复行；`cateNameTree` 已代表该路径下的全部子类目。

**比价项章节生成规则**（仅用于生成，不得写入 `promptContent`）：生成 `## 比价项` 前，必须先按
`compareItem` 对 5.1 内部转换账本分组；每个 `compareItem` 只生成一个章节，不得按 `categoryId`
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

<按 5.1 "货号匹配逻辑"下方的 Step3 生成规则，在此填入普通 Step3、货号 Step3 或两者；若两者都存在才在标题中标明适用 cateNameTree。>

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
- **匹配（<cateNameTree-A>）**：<该类目的 expectedMatch>
- **优先级（<cateNameTree-B>）**：<该类目的 expectedPriority>
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
      "left": {
        "value": "",
        "source": ""
      },
      "right": {
        "value": "",
        "source": ""
      },
      "match": true,
      "reason": ""
    },
    "<生效比价项2>": {
      "left": {
        "value": "",
        "source": ""
      },
      "right": {
        "value": "",
        "source": ""
      },
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

上面的 `字段规则：` 标题及其下方 8 条规则是候选提示词的固定必选正文，不是格式说明或可选注释。
生成时必须逐条保留，不得概括、合并、改写或省略。JSON 围栏结束不代表 `# 输出格式` 章节结束，
只有 8 条字段规则全部输出后该章节才完整。具体类目不涉及材质时，也必须保留多部位材质的字段格式规则。

##### 章节结构要求

完整提示词必须包含以下章节并严格保持顺序：
1. `# 角色`
2. `# 输入说明`
3. `# 执行步骤`
4. `# 判定规则` → `## 总原则` → `## 比价项` → `## 映射表`（条件性，见映射门禁）
5. `# 图片说明`
6. `# 输出格式`

一级章节之间使用独立的 `---` 分隔。除条件性 `## 映射表` 外，不得改名、合并、调序或省略固定章节。

##### 映射表写法

**母子品牌**：
- 标题格式：`### 母子品牌映射表`，禁止写动态组数。
- 格式：每行 `序号. 母品牌→子品牌1|子品牌2`，相同母品牌不拆多组。
- 仅当材质表也需输出时，最后一条母子品牌之后空一行并单独输出 `---`，再空一行输出材质表标题。

**材质**：
- 标题固定：`### 材质对照表（同组内可互匹配，跨组不可）`
- 每组：`**组名（完整路径）**：成员1、成员2`，直接使用 `materialText[]` 过滤后的文本行原样输出。
- 过滤后为 0 行时省略材质引用行和整个材质表，不输出空结果说明。
- 最多 50 组；实际不足时不补齐。

##### 完成检查

生成候选全文后逐项检查，任一项不满足先修复再调用校验工具：

**结构**：
- 是否包含全部固定章节并保持正确顺序？一级章节之间是否使用 `---`？
- 是否完整包含 Step0 至 Step3，且没有跳过抽取或匹配直接给结果？
- 是否删除全部尖括号占位符、`TODO`、空标题和无内容条目？

**业务规则**：
- 是否只使用本轮 MCP 返回的比价项、类目、优先级、阈值、例外和映射？
- 是否已将来源归一为仅含`商品属性`、`商品标题`、`主图`，删除全部 SKU？投影全空时是否使用固定综合判断兜底？
- 每个比价项是否先写专属规则、再写对应兜底逻辑，且两者均未遗漏？

**映射与长度**：
- 母子品牌是否不超过 100 组、材质是否不超过 50 行（来自 `materialText[]` 过滤结果）？
- 母子品牌标题是否不写组数？序号是否连续 `1..N`，格式是否统一为 `母品牌→子品牌1|子品牌2`？
- 材质组是否直接来自本轮 `materialText[]` 过滤行，路径和成员均未被改写？
- 全文是否大体在 1 万字以内，且没有为压缩长度删除规则？

**JSON**：
- JSON 示例是否可解析，且包含 `result`、`reason`、`confidence`、`extracted`？
- `extracted` 是否只包含生效比价项，没有不适用项或空对象？
- `confidence` 是否为范围内的具体数字字面量，而不是字符串或区间？
- `key_diff_point` 是否始终存在，并与 `result` 一致？
- JSON 围栏后是否仍完整包含 `字段规则：` 及其下方 8 条固定规则？

#### 5.3 调用校验前的必检项

1. 从 5.2 完整格式模板中完整复制 `# 输出格式` 固定块至第 8 条 `key_diff_point` 规则，不因长度或类目裁剪。
2. 生成内部转换账本（见 5.1），命中受控别名时候选必须是标准枚举的完整展开，不得残留别名原文。
3. JSON 围栏后保留 `字段规则：` 和完整 8 条规则。
4. 对母子品牌映射表和比价项序列分别从 1 开始整体重写连续编号；重写后再次扫描，若仍断号则继续修复。
5. 映射门禁：
   - `brandScopeCateNameTreeSet` 非空时：必须有本轮品牌关系查询与过滤记录；过滤后非空输出映射表，为空静默省略；
     品牌映射必须通过 4.3 的完整过滤、数量、格式和连续编号门禁。
   - `materialScopeCateNameTreeSet` 非空时：对应比价项章节必须存在固定行 `- **材质对照表**：见下方附录。`，
     且下方必须存在材质对照表附录，二者缺一不得调用校验；`materialGroupCount>0` 时输出真实行，
     为 0 时静默省略引用和材质表。
   - `brandScopeCateNameTreeSet` 和 `materialScopeCateNameTreeSet` 均为空时，省略整个 `## 映射表` 章节。
   - 候选全文出现"无符合""无可输出""未命中""暂无"或任何映射门控说明，均视为空表泄漏，
     必须删除后重新生成，禁止调用校验。
   - 母子品牌标题不得包含"共 N 组"等动态组数。
6. 全文约 1 万字；接近或超过时只删除重复解释、重复示例、同义表达；
   不得裁掉真实比价项、规则边界、例外、固定章节或 JSON 字段，不得硬截断。

#### 5.4 硬门禁

1. **来源投影门禁**：每个 `compareItem` 的 `expectedPriority` 必须通过来源过滤规则推导，不得省略、
   重排或补入原文不存在的来源；`商品属性栏（站外）` 必须保留并归一为 `商品属性`；
   `主图文字说明` 必须保留并归一为 `主图`；品牌来源链必须按强制解析用例处理，
   最终 `expectedPriority` 必须逐字符等于 `商品属性>主图>商品标题`。
2. **关系响应门禁**：品牌、材质正文只能来自本轮真实关系接口响应及过滤结果；
   材质直接使用本轮 `materialText[]` 过滤后的文本行，不得修改或补充。
3. **候选泄漏门禁**：候选全文不得出现任何生成器门控、条件输出或内部算法说明，
   也禁止用"无符合""未命中""暂无"等空结果正文占位。

#### 5.5 调用校验

`tool_validate_prompt_skeleton(promptContent=<候选全文>, operator=上下文.operator, conversationId=上下文.localConversationId, basePromptVersionId=0, ruleGroupId=上下文.ruleGroupId, agentId=上下文.agentId)`

`ruleGroupId`、`agentId` 至少一个大于 0。

初始化先处理 `data.promptExists`：为 true 时返回 `data.existingPromptVersionId` /
`data.existingPromptName` 后停止；为 false 才继续。

- `baseResp.respCode=1 && data.valid=true && data.diffRecordId>0`：成功，锁定本次 `promptContent`，
  保留 `diffRecordId`、`diffContent`。后续展示引用这个已提交变量，禁止重新生成。
  初始化时 `data.diffContent` 为空是允许的，不得据此报错或把 `diffRecordId` 推断为 0。
- `baseResp.respCode=1 && data.valid=true && data.diffRecordId<=0`：立即异常结束，只回复
  "提示词校验未生成有效修改建议（diffRecordId=0），无法形成提案。"禁止重试。
- `data.valid=false`：只按 `data.errors` 修正报错处，其他正文不变，最多重试 2 次。
- 其他情况或重试后仍失败：只返回最终错误，不展示提案、不写入。

Diff 只取服务端 `data.diffContent`，禁止自行书写。只能首次及按 errors 重试时调用。

### 6. 展示提案

严格使用下方提案模板：标题为第一行，字段和顺序不变，替换全部占位符；模板外不加寒暄、摘要或结论。

按下方格式展示。以下是初始化提案唯一允许的可见回复协议；必须以 `## 提示词初始化提案` 为第一行，
逐项、按序套用，模板前后不得添加任何文字。所有尖括号字段必须替换成本轮真实值：

`````markdown
## 提示词初始化提案
- 基础提示词版本：`promptVersionId=0（初始化）`
- 修改建议 ID（diff_id）：`<原样引用 校验工具返回的非零 data.diffRecordId>`
- 状态：尚未保存
### 合理性判断
- 初始化目标：为当前规则组创建首个 PriceStudio 比价 Agent 提示词
- 规则与映射依据：<本轮实际加载的类目规则、母子品牌和材质映射范围>
- 适用范围与风险：<关联类目范围、需要重点回归的规则或映射>
### 完整提示词
````text
<逐字符引用同次校验工具调用已提交并锁定的 promptContent；不得省略、拆分或重新生成>
````
以上仅为提案。确认无误请回复：**确认初始化提示词**。确认后服务端会再次检查该规则组是否已有提示词；已有则直接返回现有结果，仍不存在才创建首个提示词草稿。
`````

发送前逐项自检：占位符是否全部替换、`promptContent` 是否与校验工具请求变量是同一字符串对象、
`diffRecordId` 是否非零。

初始化：逐字符展示同次校验工具调用已提交并锁定的 `promptContent`，放入四反引号 `text` 围栏；
正文内三反引号原样保留，不得省略、拆分、重生成或写"其余组"。
