# Step 8 Sample/Component 关联设计决策 — 执行计划（v2 修订版）

> 类型：纯设计决策 Session（非 Fixture、非 Coding）
> 修订：v2 — 去除预设结论，改为中立决策门禁；补齐字段级链路验证要求；三份 Step 8 文档更新改为条件式
> 输出：决策报告（无条件，必做）+ 三份 Step 8 文档更新（条件式，仅当门禁通过）
> 硬约束：不改 Frozen Docs（02/03/04/05/07）、不改 registry.py、不写生产代码/测试/Fixture、不做破坏性 Git 操作；不进入 Coding；不自动开启下一阶段

---

## 1. Summary

本 Session 通过证据化核查判定 D-CONFLICT-001（Component–Sample 关联）的真实状态，并在三个**互斥结论**间做出判定：

| 结论 | 含义 | 触发条件 |
|---|---|---|
| 结论 A | 现有 Frozen CDM 已足够表达「多个测量值属于同一 Component」 | E1 判定 MeasurementSet/Measurement 为正式可用契约，且 E2/E4 证明无需 Schema 变更 |
| 结论 B | 原混凝土问题仍未解决，但存在冻结 Schema 内合法的**最小替代 Vertical Slice** | 不满足 A；E4 证明替代切片全链无 BLOCKING |
| 结论 C | 必须进行 Frozen CDM Schema 变更 | A、B 均无法通过证据成立 |

**本计划不预设、不暗示任何结论。** 执行严格按序：

```text
E1 契约强度核验 → E2 实体字段核验 → E3 方案 A/B/C/D 比较
→ E4 字段级映射与 P0 链验证 → E5 门禁判定 → E6 决策报告（必做）→ E7 条件式文档动作 → E8 停止
```

D-STEP8-13（Conflict 人工裁决后的 Fact 状态转换）独立核查，不随本决策关闭。

---

## 2. Current State Analysis（已实际读取的核验事实 — 仅作分析起点，不作结论）

### 2.1 MeasurementSet / Measurement 的文档定义强度（待 E1 正式判定）

| 来源 | 实际内容 | 初步强度 |
|---|---|---|
| `03` §23（L1137-1157） | "某一个检测项目的一组测量数据"；示例 K001=32.4 / K002=31.8 / K003=33.1 | 概念描述，无 ID/字段/关系定义 |
| `03` §24（L1161-1177） | "一个具体测量值"；典型信息含 构件/测点/测值/单位/方法/时间/仪器；**"具体字段根据实际报告类型继续细化"** | 字段明确未冻结 |
| `03` §29（L1428-1439）/ §39 | Fact 组织为 Component / MeasurementSet / Measurement 的示意；MeasurementSet→程序计算→average 概念流程 | 组织/流程示意 |
| `03` §8.1（L345-355） | scope_class 含 `sample`；`measurement` 明确不是 scope_class（V3.1-5） | measurement 无主体类型 |
| `02` §6.1（L283）/ §17.2 | XLSX 标准化目标名义写 Measurement / MeasurementSet | 方向性描述 |
| `04` §24（L938-954） | Facts → MeasurementSet → 确定性程序 → TableSpec | 流程概念 |
| `src/cdm/types.py`、`domain.py` | 无 MeasurementSet / Measurement / Sample 类 | 未实现（**≠ 不支持**） |
| `registry.py` | 无 measurement 相关类型 | 未注册 |

### 2.2 实体与关系字段（已核验 — domain.py / domain_validate.py / types.py）

- 已实现 5 类 L3 对象：Project / Structure / Component / Defect / Evidence；**无 Sample**。
- Component：`id / parent_structure_id(必填) / defect_ids[](可空) / fact_ids[](仅 component scope)`。
- Defect：`id / host_component_id(必填，弱实体锚定) / fact_ids[](仅 defect scope)`。
- **引用完整性显式推迟**：`domain_validate.py` 不检查 host_component_id 是否指向真实 Component（L14-21，推迟到 Store / Pipeline 层）。
- 冻结规则：`Relations are explicit L3 fields (B-class), NOT Facts`（domain.py L16）。
- Evidence：`raw_source_id(必填) / source_refs(非空) / related_fact_ids(跨 scope 允许) / related_object_ids(可选)`。
- SourceRef = `(source_id, location)`；RawSource = `(source_id, source_type, location?, original_path?, metadata)`。
- Fact 全字段冻结（含 `method / provenance / status / review_status / supersedes / revision` 生命周期语义）。

### 2.3 Registry 实况（已核验 — registry.py L255-304）

已注册相关条目：`component.id`、`component.concrete_strength`(pressure)、`defect.id`、**`defect.width`(length，"maximum defect width (mm)"——单缺陷最大宽度语义)**、`defect.length`、`defect.pattern`、`defect.judgement`、`defect.present`、`point.id`、`sample.id`。

未注册：`sample.concrete_strength`、任何 sample↔component 关联类型、任何构件级缺陷聚合类型（如 component.max_defect_width）。

### 2.4 生产侧实现现状（已核验 — src 全树）

src 仅有 `src/cdm/**` 与 `src/eval/**`。以下均为三份 Step 8 文档中的 **Proposed（未实现）**：CellRaw / parse_xlsx / CandidateFact / FactStore / Compute / Criterion / Conclusion / IR Builder / IR Materialization Adapter。链路各环节的「Frozen / Proposed / 未定义」标注由 E1/E2/E4 逐一定案。

### 2.5 验证侧接口（已核验）

- 7 条 P0 规则（`validator_framework.py`）；SystemOutput / RenderedSegment / TableCell / StructuredTable / AnchorDeclaration 冻结（`data_structures.py`）。
- Step 7-F run 结构已实现（`src/eval/run/models.py`：RunSnapshot / RunRecord / RegressionReport / Baseline / GateDecision / JitterRegister）。
- 7 个评测 fixture 已存在，`inputs/` 全空（仅 .gitkeep）；仓库内无任何 .xlsx / .csv。

### 2.6 D-STEP8-13 相关（已核验）

- Conflict 身份键 = `fact_id`；字段 `candidates / resolution_policy="manual" / resolved_by / resolution`（`types.py` L117-152）。
- `03` §15.3 / §15.5：`conflict`（横向来源分歧）与 `superseded`（纵向 revision 更正链）不得混用。
- 无「裁决结果 → 最终 Fact」的 status / revision / supersedes 转换定义。

---

## 3. Execution Steps（严格按序执行）

### E1 契约强度核验与五分类矩阵（必做）

读取 `03` §2-§17、§23、§24、§29、§34-§36 及 `04` / `05` 相关章节，输出五分类差距矩阵：

```text
冻结明确规定 / 仅概念示例 / 已定义未实现 / 冻结与实现差距 / 完全未定义
```

判定规则（硬约束）：不得因类未实现断言 CDM 不支持；不得因概念示例断言字段与关联已冻结。

### E2 实体字段与约束核验（必做）

基于 §2.2 起点，补全核实 Component / Defect / Evidence / SourceRef / Fact / RawSource / Domain Object 关系的全部字段与约束，以及 `04` §41.2 与 `05` §6 的链路契约。输出字段级「可表达 / 需实现 / 需新增」清单。

### E3 候选方案 A/B/C/D 比较（必做）

对每方案给出六维对比：合规性 / Schema 影响 / 实现复杂度 / 来源追溯 / Compute 确定性 / M3 价值。

- A：复用 MeasurementSet + Measurement（先完成 E1 判定，再评估）
- B：引入 Sample Domain Object 与 Component 关联（涉及 Frozen Schema → 只能 D-045 级提案）
- C：关联关系作为普通 Fact（核查 "Relations are explicit L3 fields, NOT Facts" 与受控注册表约束，判定合规性）
- D：修改 Vertical Slice 最小业务范围（**严肃评估**；合理最小替代即作为候选；不得为保住混凝土试块方案而排除；若替代无法实现真实计算/可验证业务行为/端到端价值，须明确指出）

### E4 字段级数据映射与 P0 验证链分析（必做 — 对每个存活候选执行）

对 E3 中未被否决的候选执行完整链路逐环验证；对因 Schema 变更需求被阻塞的候选，记录阻塞环 + 假设变更获批后的影响分析。链路：

```text
Excel 原始单元格 → RawSource / CellRaw → CandidateFact / Fact → Domain Object
→ Compute 输入 → Compute 输出 Fact（如需要）→ Criterion / Evaluation → Conclusion
→ Report IR → IR Materialization → SystemOutput → Step 7-E 七项 P0 → Step 7-F Run / Regression
```

逐项检查（每条须记录证据：文件 / 字段 / 条款）：

1. `defect.id` / `defect.width` / `component.id` 注册定义与语义；`defect.width` 是单缺陷最大宽度，**不得与计算代表值混同**（若候选切片不依赖 defect，则对相应对象做同类核验）。
2. `Defect.host_component_id` 必填性与校验方式；引用完整性在 domain_validate 被显式推迟 → 明确切片由哪一层保证、失败时的行为（不静默）。
3. Compute 分组键的确定性来源必须是 L3 关系字段，不得靠命名约定推测归属。
4. Compute 输出 Fact（如需要）逐字段定案：`fact_id` / `fact_type`（**未注册 → 标 Proposed + 记录授权要求**）/ `unit` / `quantity_kind` / `method=computed` / `provenance=program` / `source_refs`（与输入事实的可追溯关系）/ `status` / `review_status` / `revision`。
5. 若无需 Compute：说明如何验证确定性业务处理；不得为保留 Compute 模块强行增加计算；若因此丧失 M3 价值，明确说明。
6. Criterion 阈值与规则仅作为**明确标注的合成测试参数**，不得冒充工程验收规范。
7. Conclusion：`direction` 枚举 / `covers[]` 覆盖度 / Evaluation 映射符合 `05` §6.5 / §6.6 与 GroundTruthConclusion 契约。
8. 数字锚定：实例标识 token 显式绑定 fact_id（如 `fact:component.K002.id`），符合 `04` §41.2 / `05` §6.4 / AnchorDeclaration 契约。
9. Report IR / SystemOutput / Step 7-E 接口零 Schema 变更可行性（对照冻结结构逐一验证）。
10. 任一环需要未获批准的 Schema / Registry / 类型变更 → 该环标 BLOCKING，进入结论 C 判定。

### E5 决策门禁判定（必做）

```text
若现有 MeasurementSet / Measurement 契约足够：
    选 A —— 明确已有契约与实现缺口清单
否则若替代切片能在冻结 Schema 内合法完成端到端验证（E4 全链无 BLOCKING）：
    选 B —— 原场景 Deferred；D-CONFLICT-001 定义为 OPEN / DEFERRED / 非当前路径 + 定义未来关闭条件
否则：
    选 C —— D-CONFLICT-001 保持 Coding-Blocking；仅形成 D-045 提案，不改 Frozen CDM
```

同时按三种结论区分 D-CONFLICT-001 状态：

```text
① 原混凝土试块关联问题已解决（需 E1 证据支持）
② 原问题仍未解决，但替代切片不依赖该关系 → 有条件推进（Deferred / 非当前路径）
③ 替代切片仍存在冻结契约无法解决的阻塞 → 不解除
```

**不得仅因选用某对象就宣布 D-CONFLICT-001 已解决**；必须解释选择依据，不机械套用顺序。

D-STEP8-13 独立判定：保持 OPEN；分析其是否影响当前切片路径；不得假定旧候选 Fact 变成 `superseded`，不得自行构造 revision / supersedes。

### E6 产出决策报告（必做，无条件）

新建 `docs/Step_8_Sample_Component_Relation_Design_Decision.md`：

1. §0 问题界定（D-CONFLICT-001 重述；与 D-STEP8-13 边界分离）
2. §1 证据核查（E1 + E2 输出，含五分类矩阵）
3. §2 方案 A/B/C/D 比较表（E3 输出）
4. §3 字段级链路验证（E4 输出，逐环 Frozen / Proposed / 未定义 / BLOCKING 标注）
5. §4 最终结论（A / B / C 之一 + 依据说明）
6. §5 D-CONFLICT-001 状态与适用范围（含未来关闭条件，如适用）
7. §6 D-STEP8-13 独立核查结论（保持 OPEN + 对当前路径的影响分析）
8. §7 D-045 状态（结论 C 时附完整提案：问题 / 证据 / 不足论证 / 最小变更 / 影响 / 迁移 / 追溯 / 确定性 Compute / 验证策略 / 风险 / 待审批）
9. §8 自检 12 项 + 最终状态块 + 文件修改清单

### E7 条件式文档动作

- **仅当结论 A 或 B 被证据支持、且当前路径无 Coding-Blocking 冲突时**，才允许同步更新三份 Step 8 文档；更新内容严格限定已证明结论，不得把「候选方案」改写为「已批准方案」，**不得自动升级为 READY FOR CODING**。
- 结论 C：三份正式文档保持原状（除非修正明确的事实错误或状态不一致）；仅形成 D-045 提案。
- A / B 时的更新范围（按结论裁剪）：
  - `Step_8_Design_Decision_Closure.md`：决策矩阵 / 门禁 4、8 / §10 复核结论 / 状态块
  - `Step_8_Coding_Contract_v1.md`：fixture 候选 / Compute 分组键 / Decision Register / 状态块
  - `Step_8_Fixture_Design_Sprint.md`：Domain Direction / 候选结构 / 关闭条件
- 任何情况下：D-CONFLICT-001 若非「已解决」则保留 OPEN；D-STEP8-13 保持 OPEN。

### E8 停止

完成 E6（+ 条件性 E7）+ 验证后立即停止。

---

## 4. Assumptions & Decisions（过程性决策，不含预设结论）

| # | 条目 | 依据 |
|---|---|---|
| 1 | 不预设结论；A / B / C 由 E1-E5 证据判定 | 用户修订要求 §1 |
| 2 | 决策报告无条件创建（E6）；三份文档更新条件式（E7） | 用户修订要求 §4 |
| 3 | E4 字段级链路验证为必做步骤；每条断言引用文件 / 字段 / 条款 | 用户修订要求 §2 |
| 4 | D-STEP8-13 独立核查；不得伪造状态转换 | 用户修订要求 §5 |
| 5 | Compute 输出类型未注册时标 Proposed（属 Registry 受控词表扩展，非 Frozen 03 修改） | registry.py 实况 |
| 6 | Criterion 阈值仅作合成测试参数明确标注 | 用户修订要求 §2 |
| 7 | 结论 C 时 Frozen 变更仅以 D-045 提案形式，标注 `PROPOSED / APPROVAL REQUIRED / FROZEN CDM MODIFIED = NO` | 用户任务书 §6 |

---

## 5. Verification（完成后执行）

1. 重读全部修改文件，逐项核对用户 §9 的 12 项验收清单
2. `git status` 如实报告（含未跟踪文件；`git diff` 为空 ≠ 无变化）
3. 确认 Frozen Docs / registry.py / src / tests / fixtures 均未修改
4. 按用户 §10 格式输出 A-F 报告（证据核查 / 方案比较 / 最终架构建议 / 决策状态 / 文件修改 / 最终状态）

---

## 6. Stop Conditions

- 完成 E6（+ 条件性 E7）+ 验证后立即停止。
- 不创建 Fixture；不改 Registry / 生产代码 / Frozen CDM；不进入 Step 8 Coding；不自动开启下一阶段。
- 结论 C 时不得为推进项目强行选择 A 或 B。