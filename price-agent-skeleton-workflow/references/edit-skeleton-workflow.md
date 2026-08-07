# 修改提示词

对任意提示词写操作，只要 `promptVersionId>0` 就进入本流程并基于当前版本派生新草稿；用户具体使用新建、创建、初始化、重新生成、修改、优化、调整、增删或按照规则生成等哪一种说法都不影响路由。共享步骤见 [shared-steps.md](shared-steps.md)。

> **命中即执行**：进入本流程后立即执行“查询与判断”，并在同一轮持续执行到 `[S3]` 校验和“提案”；不得向用户解释为何不能初始化、不得只预告下一步，也不得等待用户再次确认才开始生成提案。用户对“新建”的表述表示基于当前版本派生新草稿，不改变 `writeMode=EDIT`。

> **无验证数据直达**：用户没有提供或要求 Badcase、验证任务、验证结果、分析验证集时，直接按本流程处理。这些数据不是进入 EDIT 的前置条件，不调用相关 MCP，也不因缺少它们而停止或澄清。

> **无方向普通修改**：立即查询当前 `promptVersionId`，不得询问 ID、名称或方向。默认按 `skeleton-format.md` 修复格式、歧义、重复、矛盾及术语、Step、比价项和 JSON 约束，并仅依据本轮可信规则补齐遗漏；不得发明规则、阈值或映射。

> **格式优先级**：完整读取 [skeleton-format.md](skeleton-format.md)。新正文必须满足其结构和格式；基础正文不合规时一并修复，同时保留不冲突的业务规则、映射和用户修改，不得改变业务含义。

> **输出边界**：内部生成并校验全文；提案只展示校验返回的 `data.diff_content`。仅用户明确要求「展开完整提示词」时展示全文。

> **精确修改直达分支**：用户给出可唯一定位的增删改时，将其作为已授权修改目标，不执行 `[S2]`，不调用关系、规则或验证任务 MCP 复核。正文格式合规时只做该修改及其必需的机械一致性调整；不合规时同时按 `skeleton-format.md` 修复。删除带序号的母子品牌组或比价项后，必须把同一序列的后续条目从删除位置起连续重新编号，并同步更新数量标题；重新编号属于本次删除的必要组成，不算扩大修改范围。用户明确输入“新增母子品牌：<关系>”时直接加入该关系，即使加入后超过 100 个主品牌组也不得拒绝、裁剪或删除其他关系；同步更新标题中的实际组数。`writeMode` 始终为 `EDIT`。

## 回复前执行门禁

本节是必须实际完成的顺序，不是计划、建议或说明。除工具返回明确错误外，完成全部门禁前禁止输出任何用户可见答复：

1. 完整读取 [shared-steps.md](shared-steps.md)、[base-version-policy.md](base-version-policy.md) 和 [skeleton-format.md](skeleton-format.md)；缺少任一读取记录即不得继续或答复。
2. 使用业务上下文 `promptVersionId` 精确查询基础版本，固定 `version_name=""`、`query_online=false`；查询成功后锁定 `selectedPromptVersionId=promptVersionId>0` 和 `writeMode=EDIT`。禁止改读初始化 workflow。
3. 用户没有给出可唯一定位的精确增删改时，必须实际执行 `[S2]`，按 [rule-loading-policy.md](rule-loading-policy.md) 加载当前范围的规则和必要映射；禁止用格式模板中的占位符、示例或常识代替 MCP 结果。
4. 在内存中生成符合 `skeleton-format.md` 的候选完整正文。基础正文为 `111` 或其他残缺内容时必须重构，禁止把基础原文直接提交校验。
5. 必须实际调用 `tool_validate_prompt_skeleton`，其中 `prompt_content` 为第 4 步候选全文，`base_prompt_version_id=selectedPromptVersionId`。`valid=false` 时只按 errors 修正并在允许次数内重试；空 Diff 不构成停止理由。
6. 仅在 `valid=true` 且 `diff_record_id>0` 后输出“提示词修改提案”，原样展示同次调用的 `diff_content` 和 `diff_record_id`。ID 缺失或为 0 时不得展示提案。

禁止把“已加载 workflow / 下一步将查询 / 等你回复继续 / 请系统或产品先补骨架”作为正常结果。用户第一次请求修改时就完成上述读取、查询、生成、校验和提案；用户确认仅用于提案后的写入步骤。

## 查询与判断

1. 仅当业务上下文缺少 `ruleGroupId` 时执行 `[S1]`；已有则跳过。
2. 按 [base-version-policy.md](base-version-policy.md) 查询基础版本；记录 `selectedPromptVersionId=data.prompt_version.prompt_version_id>0` 并锁定 `writeMode=EDIT`。
3. 按 [skeleton-format.md](skeleton-format.md) 完整审计基础正文；任一要求不符均标记为“需要格式修复”并在本次修复。基础正文即使只有 `111` 或其他残缺内容，也只是需要重构的基础版本，不得把原文直接提交 `[S3]` 后因校验失败而停止。
4. 不调用验证任务或验证结果查询；需要分析验证数据时改走用户明确触发的 Badcase 流程。
5. 精确修改跳过本步；其他分支才按 `[S2]` 加载改动范围内的规则与必要映射。比价项名称无法对应规则或基础正文时先澄清，不改写成相近名称。
6. 精确修改只检查定位、重复和结构；其他分支再依据规则、映射和基础正文判断合理性。发现冲突、关键条件缺失或误判风险时说明并停止。
7. 格式合规时做最小修改；不合规时必须先按 [skeleton-format.md](skeleton-format.md) 重组并合入修改，保留可识别且不冲突的业务规则。基础正文没有可保留的有效业务内容时，依据 `[S2]` 已加载的可信规则生成完整骨架；不得要求外部先补一版骨架。仅需范式时读取 [rule-writing-examples.md](rule-writing-examples.md)。全文按格式规范控制在 1 万字内；基础正文已超出时不借机大幅删减。
8. 完成第 7 步候选全文后，实际执行 `[S3]` 校验这份内部完整正文，不向用户展示全文。严禁把未重构的基础正文（例如 `111`）作为 `prompt_content`。仅同一次调用返回 `resp_code=1`、`data.valid=true` 后进入提案，并原样记录 `data.diff_content` 和 `data.diff_record_id`；不得自行生成 Diff 或预演校验结果。`valid=false` 时忽略该次空 Diff，按校验错误修正候选全文并重试。

## 提案

按 `[S4]` 展示。以下模板即修改提案允许输出的完整范围，不得在模板后追加提示词全文或全文分段：

````markdown
## 提示词修改提案
- Agent：`<名称；查不到时写ID>`
- 基础提示词：`<名称>`（ID：`<ID>`，状态：`<线上/草稿/归档>`）
- 当前提示词来源：页面左侧选定的提示词（以当前业务上下文 `promptVersionId` 为准）
- 修改建议 ID（diff_id）：`<原样引用 S3 返回的非零 data.diff_record_id>`
- 状态：尚未保存
### 合理性判断
- 修改目标：<用户的原始要求或优化方向>
- 规则与映射依据：<支持本次修改的规则依据>
- 影响与风险：<适用范围、冲突和需要回归的项>
- 不修改部分：<明确保留未改动的规则>
### Diff
```diff
<原样引用 S3 返回的 data.diff_content>
```
完整提示词已生成并通过格式校验，未在此展开。需查看请回复「展开完整提示词」。
以上仅为修改提案。确认后创建新提示词草稿，不覆盖基础版本。
确认无误请回复：**确认创建提示词草稿**。
````

诉求明确时可简写「合理性判断」，但不得省略「影响与风险」和「不修改部分」。精确修改的依据写“用户明确指定”，不得声称来自未调用的 MCP。只有非零 `diff_record_id` 才能形成提案；只输出模板和服务端原样 Diff，不展示内部推理。

## 确认后

执行 `[S5]` 的 `EDIT` 路径，`prompt_version_id=selectedPromptVersionId`；缺失或为 0 时停止，不得改用 `save_prompt_draft`。
