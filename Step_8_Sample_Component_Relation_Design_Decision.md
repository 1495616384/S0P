# Step 8 Sample / Component 关联设计决策报告

> **日期**：2026-10-09
> **性质**：纯设计决策 Session（不产生生产代码、不创建 Fixture、不进入 Coding）
> **前置**：`Step_8_Design_Decision_Closure.md`（NEEDS DECISION）→ 本决策报告 → 条件式文档更新
> **后续状态更新（2026-10-09，Fixture Design Sprint 执行后）**：本报告为决策 Session（E6/E7）结束时的真实状态快照。此后同日执行的 Fixture Design Sprint 已完成：D-STEP8-02/03/06/07/09 证据化 CLOSED、D-STEP8-13 形成处置决议、9/9 门禁 PASS → **当前权威状态**见 `Step_8_Design_Decision_Closure.md` §14 / `Step_8_Coding_Contract_v1.md` §0 / `Step_8_Fixture_Design_Sprint.md` 尾部状态块（`STEP 8 CODING = READY`，尚未开始）。本报告内文（含 §8.2 状态块）保留决策时的历史记录，不作为当前状态依据。
> **执行计划**：`.trae/documents/step8_sample_component_relation_decision_plan.md`（v2 修订版）
> **本轮修改范围**：本报告（新建）+ 三份文档的条件式更新（见 §8.3）；Frozen Docs（02/03/04/05/07）/ `registry.py` / 生产代码 / 测试 / Fixture 一律未修改

---

## §0 问题界定

### 0.1 D-CONFLICT-001 重述

**业务场景**：混凝土抗压强度检测。GB/T 50081-2019 要求每个构件取 3 个平行试块，产生 3 个强度值；工程习惯需要再由构件级代表值（平均）做合格判定。

**Schema 缺口**：

| 能力 | 现状 | 证据 |
| ---- | ---- | ---- |
| 试块强度值本身的表达 | 无已注册类型（`sample` 仅注册 `sample.id`） | `src/cdm/registry.py` L300-302 |
| 「试块属于哪个构件」的归属表达 | 无合法载体 | 见下 |
| 构件级统计输出（如平均值）的类型 | 未注册；`03` §16.3 的 `fact:point.K001.average` 仅为示例 | `03_CANONICAL_DATA_MODEL.md` §16.3（L879） |

归属表达为何是无合法载体的——两条路都被冻结契约封死：

1. **作为普通 Fact 表达（如 `sample.host_component_id`）**：违反 `domain.py` L16 冻结意图「Relations are explicit L3 fields (B-class), **NOT Facts**」；且该 attribute 未注册，03 §8.1 规定 attribute 必须来自受控注册表、不得在抽取时临时发明。
2. **作为 L3 关系字段表达**：需要 Sample Domain Object 携带 `host_component_id` 一类字段；但 `domain.py` 仅实现 5 类（Project / Structure / Component / Defect / Evidence，L5-10），**没有 Sample 类**；`03` §23 MeasurementSet / §24 Measurement 仅为概念骨架（§24 明文「具体字段根据实际报告类型继续细化」，L1177），不构成可实施契约。

**结论（问题定性）**：D-CONFLICT-001 是**冻结 CDM 的 Schema 表达能力缺口**（component ↔ sample 归属），不是普通实现缺口，也不是单纯 Registry 覆盖不足。

### 0.2 与 D-STEP8-13 的边界分离

| 项 | D-CONFLICT-001 | D-STEP8-13 |
| ---- | ---- | ---- |
| 性质 | 横向：Schema 无法表达实体归属 | 纵向：Conflict 人工裁决后的 Fact 状态转换未定义 |
| 证据 | 03 §17/§23/§24；domain.py；registry.py | 03 §15.5（conflict ≠ superseded，不得混用）；§34 Conflict 仅 `resolved_by` / `resolution`；`types.py` Conflict 无 `conflict_id`、无转换字段 |
| 关联 | 无——两个问题独立 | 无——不得以本报告任何结论替代其独立判定 |

### 0.3 本报告授权边界

- 本轮**未获授权**修改 Frozen Docs（02/03/04/05/07）、`registry.py`、生产代码、测试、Fixture；未获授权进入 Coding 或开启下一阶段。
- 结论 C 成立时仅允许形成 D-045 提案；本报告不预选结论，A / B / C 由 §1-§3 证据裁定。
- **不因"类未实现"断言 CDM 不支持**（实现差距 ≠ Schema 缺口）；**不因"概念示例"断言字段与关联已冻结**。

---

## §1 证据核查（E1 + E2）

### 1.1 五分类矩阵

| 类别 | 事项 | 证据（文件 / 条款 / 字段） |
| ---- | ---- | ---- |
| **冻结明确规定** | Fact 三段式 ID：`fact:<scope_class>.<instance_key>.<attribute>`；fact_type 不变式 `<scope_class>.<attribute>`；attribute 受控注册、不得发明 | 03 §8.1（L319+）；`registry.py` L15、L167-227（`validate_fact_type` / `validate_id_against_fact_type`） |
| | scope_class 枚举含 `sample`（7 值） | `registry.py` L16-24；03 §8.1 |
| | L3 关系铁律：关系是显式 L3 字段（B-class），不是 Fact | `domain.py` L16 |
| | `Component`：`id / parent_structure_id(必填) / defect_ids / fact_ids` | `domain.py` L137-171 |
| | `Defect`：弱实体，`host_component_id` **必填**；crack 等是 Defect 实例（由 fact_ids 区分），不是独立类 | `domain.py` L178-210（L9 注释、L189 字段） |
| | Criterion 强制字段（`criterion_id/value/unit/quantity_kind/condition/source_refs/review_status/rule`）+ 判定闸门（`review_status != confirmed` → Evaluation 必须 `not_evaluable`，禁止方向性判定） | 03 §25.1（L1198-1234；闸门 L1213-1217） |
| | Evaluation 判定结果「应由明确规则和确定性程序执行，而不是由 LLM 自由生成」 | 03 §26（L1254-1256） |
| | ConclusionRule 结构（`id/applies_to/inputs/logic/output/source_ref`）+ 聚合逻辑必须可执行、禁止 LLM 自由综合、必须可追溯（rule_id + 全部 Evaluation id） | 03 §27.1（L1307-1351） |
| | `conflict`（横向来源分歧）与 `superseded`（纵向版本更迭）不得混用 | 03 §15.5（L771+） |
| | Conflict 结构：`fact_id / candidates / resolution_policy / resolved_by / resolution`；第一阶段 `resolution_policy` 必须 `manual` | 03 §34-§36；`types.py`；`registry.py` `RESOLUTION_POLICIES`（L134-138） |
| | 数字锚定：数值 token 与实例标识 token 均须显式绑定 `fact_id` 且值相等；三类载体（Narrative/Table/Figure） | 04 §41.2；05 §6.4（L220-239）；`binding.py` BIND-7/BIND-8；`tokenizer.py`（照 tokenizer 提取 numeric + identifier） |
| | 7 条 P0 规则集与 Stage A~E 五阶段验证框架 | `validator_framework.py` `P0_RULE_IDS`（L78-86）；05 §6.1-6.6、§7 |
| **仅概念示例**（不得据以实施，不得断言已冻结） | MeasurementSet（「某一个检测项目的一组测量数据」，示例 K001=32.4 / K002=31.8 / K003=33.1）与 Measurement（「具体字段根据实际报告类型继续细化」）；无 ID 方案、无强制字段、无关系字段、无注册类型 | 03 §23（L1137-1157）、§24（L1161-1177） |
| | L3 组织示例（Component→MeasurementSet→Measurement 层级图） | 03 §29（L1399+） |
| | 程序计算事实示例 `fact:point.K001.average`——该 fact_type **未注册** | 03 §16.3（L879）；`registry.py` 实况无此条 |
| | 04 Computed 表达式示例（`avg(measurements)`，表达式语言「后续定义」） | 04 §23 |
| | 02 名义链路 XLSX → Measurement / MeasurementSet | 02 L283（名义）；02 L757 / L770（M3 = ir/schema + ir/builder；M3 硬标准表述） |
| **已定义未实现**（Step 8 Proposed 模块） | CellRaw / ParseResult（Parser 输出 Schema） | Coding Contract §5（Proposed） |
| | CandidateFact（瞬态桥接对象，非 CDM Fact）/ FactStore 状态机 / `resolve_conflict`（键=`fact_id`） | Coding Contract §7/§8（Proposed Interface） |
| | `compute_statistic`（签名与语义待定） | Coding Contract §9（Proposed） |
| | Criterion Engine `evaluate()` / `conclude()` / IR Builder / IR Adapter（Materialization） | Coding Contract §10-§13（Proposed） |
| **冻结与实现差距** | 无 Sample L3 类；无任何 sample↔component 关联字段 | `domain.py` 实况 |
| | `sample.concrete_strength` 未注册；无任何聚合（average 等）输出类型注册 | `registry.py` L255-304 实况 |
| | `domain_validate.py` 将引用完整性（含 `host_component_id` 指向存在性）**显式推迟**到 Store / Pipeline 层 | `domain_validate.py` L14-21 |
| **完全未定义** | sample → component 归属表达的机制（Registry 与冻结 L3 均无；概念层 MeasurementSet 亦无归属字段） | 综合上述——即 D-CONFLICT-001 本体 |
| | Conflict 人工裁决后的 Fact 状态转换（status / revision / supersedes） | 03 §15.5 / §34-36；`types.py`——即 D-STEP8-13 本体 |

### 1.2 受控注册表实况（`registry.py` L255-304，逐条核验）

**缺陷相关全套已注册**：

| fact_type | quantity_kind | 说明 |
| ---- | ---- | ---- |
| `component.id` | qualitative | 构件标识（如 K001） |
| `component.concrete_strength` | pressure | 构件混凝土强度（MPa）——混凝土场景用 |
| `defect.id` | qualitative | 缺陷标识（如 C001） |
| `defect.width` | length | **单缺陷最大宽度（mm）**——注册描述明确「maximum defect width (mm)」 |
| `defect.length` | length | 缺陷长度（m） |
| `defect.pattern` | qualitative | 走向/形态（diagonal, transverse, ...） |
| `defect.judgement` | qualitative | 工程判断（定性） |
| `defect.present` | dimensionless | 存在性 |
| `sample.id` | qualitative | 试块标识——**sample 作用域仅此一条** |

**未注册**：`sample.concrete_strength`；任何 `average/max/min` 聚合输出类型；任何 sample↔component 关联类型。

### 1.3 L3 实体字段实况（`domain.py`）

- 五类对象：Project / Structure / Component / Defect / Evidence（L5-10）。
- `Component(id, parent_structure_id, defect_ids, fact_ids)`——`parent_structure_id` 必填；`defect_ids` 可为空。
- `Defect(id, host_component_id, fact_ids)`——`host_component_id` **必填**（弱实体）。**构件↔缺陷归属有合法且已冻结的 L3 载体**。
- 关系铁律（L16）：「Relations are explicit L3 fields (B-class), NOT Facts」。

### 1.4 判定 / 结论链契约（03 §25-§27）

- Criterion：`rule` 判定形式（示例 `value <= limit`）；单位与量纲规则与 Fact 一致（`value/unit/quantity_kind`）；`source_refs` 规范定位；`review_status` 闸门为红线（L1213-1217）。
- Evaluation：允许产生新判定信息的核心位置；必须确定性程序执行（L1256）。
- ConclusionRule：聚合逻辑的允许形式包括「单项否决 / 全部满足 / 分级映射 / 查表」（L1320-1327）；结论必须记录 `rule_id` 与全部参与聚合的 Evaluation id（L1349-1350）。
- 结论文字属 Report IR（受控模板句 + Ref），不属本链（§27 尾、§27.1 L1353-1357）。

### 1.5 验证侧契约（冻结，零修改）

- `data_structures.py`：`AnchorDeclaration(token, fact_id, char_start, char_end)`；`RenderedSegment(segment_location, raw_text, anchor_declarations)`；`TableCell(raw_text, fact_id?, anchor_declarations)`；`StructuredTable(table_id, headers, rows)`；`SystemOutput(report_ir, rendered_segments, structured_tables, system_reported_issues, system_evaluations)`。
- `validator_framework.py`：Stage A A-9（`renderable_fact_types` 每元素必须 ∈ FactTypeRegistry）；A-12（TABLE.INTERNAL 适用需 trusted `table_specs`）；P0-2 无启用检查 → NOT_EVALUABLE；P0-5 读 `report_ir.conclusion.direction`；P0-6 读 `report_ir.conclusion.covers`。
- 05 §6.5（结论方向枚举比对）/ §6.6（covers[] 覆盖度）/ §7（P0 必须 100%，gold 门禁）。

### 1.6 E1/E2 判定规则遵守声明

- 「无 Sample 类 / 无聚合注册」仅作为**实现差距**记录，未据此断言 CDM 不支持任何能力。
- MeasurementSet / Measurement / §29 组织图 / `point.K001.average` 示例均只作为**概念示例**记录，未据此断言字段与关联已冻结、可实施。

---

## §2 候选方案比较（E3）

### 2.1 候选定义（依计划 v2 §E3）

- **A**：复用 MeasurementSet + Measurement（先 E1 判定，再评估）
- **B**：引入 Sample Domain Object 与 Component 关联（涉及 Frozen Schema → 只能 D-045 级提案）
- **C**：关联关系作为普通 Fact
- **D**：修改 Vertical Slice 最小业务范围（严肃评估；不得为保住混凝土方案而排除）

### 2.2 六维比较表

| 维度 | A（复用 MeasurementSet） | B（Sample 对象 + 关联，Frozen 变更） | C（关系作 Fact） | D（修改最小业务范围） |
| ---- | ---- | ---- | ---- | ---- |
| **合规性** | **不可实施**：概念未冻结（§24 L1177「继续细化」），无 ID / 字段 / 关系 / 注册类型可依据 | 可设计，但属 Frozen CDM Schema 变更，**仅 D-045 级提案可承载** | **不合规**：违反 `domain.py` L16 铁律；attribute 未注册、不得发明（03 §8.1） | **合规**：拟用全部 fact_type 均已注册；归属走 `Defect.host_component_id`（冻结必填 L3 字段） |
| **Schema 影响** | 若补完概念为正式契约 → 实为 03 修订（等同 B 的复杂度） | 高：03 §17/§23/§24 + `domain.py` + `registry.py` | 零表面修改、实为绕过治理（禁止） | **零** |
| **实现复杂度** | 高（需先冻结概念才能开工） | 中-高 | 低（技术可行但违规） | 中（复用同一组 Proposed 模块；无统计 Compute、无聚合输出类型） |
| **来源追溯** | 概念设想可追溯，无字段可挂 | 可设计（sample.* facts + 归属字段） | Fact 级可追溯但语义错位 | **完整**：Fact `source_refs` → Excel 单元格；数值与标识锚定齐备 |
| **Compute 确定性** | 分组键未定义 → 不可确定 | 分组键 = 新关系字段，可设计 | 分组键可从 Fact 读，但承载违规 | 最小路径无统计分组需求；Evaluation 聚合分组键 = `host_component_id`（L3 冻结字段，确定性）。若未来做缺陷统计聚合 → 需未注册类型（Proposed），不在最小路径 |
| **M3 价值** | 无法在零变更下实施 → 不可达 | 可达（混凝土场景），但需授权 | 以违规方式达成 → 不可接受 | **可达**：零 LLM 端到端可校验报告；覆盖 Parser / Store / Rules / IR / Adapter；统计 Compute 覆盖延后（如实） |

### 2.3 逐方案裁定

- **方案 A — 否决**。理由不是「类未实现」（那是实现差距），而是**契约不足**：MeasurementSet / Measurement 在冻结文档中仅为概念描述（03 §23/§24），无 ID 方案、无强制字段表、无关系字段、无注册类型；§24 明文「具体字段根据实际报告类型继续细化」。未冻结的概念不能作为实施契约；若将其补完，即是 03 修订本身（并入方案 B 的复杂度与授权要求）。
- **方案 B — 保留为结论 C 的触发路径，当前不选**。它是 Schema 级变更，只有在「不存在合规替代路径且必须修改 Frozen CDM Schema」时才应选择（用户约束 #1）。本轮 E4 证明合规替代路径存在（方案 D），故不触发。
- **方案 C — 否决**。硬性不合规（铁律 + 受控注册表），不因"技术可实现"而通过。
- **方案 D — 存活**。进入 E4 全链验证（§3）。

---

## §3 字段级链路验证（E4）

验证对象：**方案 D 的存活形态**（缺陷判定切片；定义见 §4.2）。逐环标注 `Frozen / Proposed / 未定义 / BLOCKING`；每条注明证据。

### 3.1 存活候选全链（十环）

| # | 环节 | 内容 | 状态 | 证据 |
| ---- | ---- | ---- | ---- | ---- |
| 1 | Excel 原始单元格 → RawSource / CellRaw | Parser 只做结构发现（不做 scope/fact_type/实例/单位推断） | **Proposed**（Coding Contract §5 CellRaw / ParseResult）；职责边界为冻结铁律 | Coding Contract §4/§5；`domain.py` 无关；RawSource 复用 `types.py` |
| 2 | CellRaw → CandidateFact | 列名 → fact_id / fact_type / value / unit / source_refs；确定性程度 Q1-Q6 待 fixture 验证 | **Proposed**（瞬态桥接对象）；**D-STEP8-03 确定性程度保持 OPEN** | Coding Contract §6；本报告 §4.2 列映射全部命中已注册类型（§1.2） |
| 3 | CandidateFact → Fact（Store ingestion） | 默认构造 `status=filled, review_status=pending`（`types.py` 默认值）；闸门 → confirmed | **Frozen（枚举）+ Proposed（接口）** | 03 §15；`registry.py` `FACT_STATUSES`；Coding Contract §7 |
| 4 | Fact → Domain Object | 切片实体：`Structure(building)` ← `Component.parent_structure_id`（必填）；`Component.defect_ids` ↔ `Defect.host_component_id`（必填） | **Frozen（类与字段）**；引用完整性保证层**需实现**（`domain_validate.py` L14-21 显式推迟到 Store/Pipeline 层——失败行为必须显式，不静默） | `domain.py` L137-171、L178-210；`domain_validate.py` L14-21 |
| 5 | Compute 输入 / 输出 | **最小路径无统计聚合**（不引入未注册输出类型）；单位换算与比较支撑走 unit registry | **Frozen（quantity_kind 体系）+ Proposed（unit_registry）**；统计 Compute **不在本切片**（如实，见 §4.4-①） | 03 §10；Coding Contract §9 |
| 6 | Criterion / Evaluation | `Criterion.rule`（如 `value <= limit`）+ 闸门（`review_status != confirmed` → `not_evaluable`）；阈值仅为本切片的**合成测试参数**，明确标注、不冒充工程验收规范 | **Frozen（契约）+ Proposed（引擎）** | 03 §25.1（L1198-1234）；执行计划 E4 检查 6 |
| 7 | Conclusion | ConclusionRule 最小聚合（单项否决/全部满足 + insufficient_evidence 分支）；记录 `rule_id` + 全部 Evaluation id；结论 `direction` 枚举 + `covers[]`（覆盖全部检测项 Evaluation，≥1 次引用） | **Frozen（契约）** | 03 §27.1（L1349-1350）；04 §42.1；05 §6.5/§6.6 |
| 8 | Report IR | Assertion 段 + 缺陷明细表 + 结论段（`direction` + `covers[]`）；数字与标识 token 显式锚定（数值 → `fact:defect.*.<attr>`；缺陷编号 → `fact:defect.*.id`；构件编号 → `fact:component.*.id`） | **Frozen（Schema）** | 04 §41.2；05 §6.4；`binding.py` BIND-7（`normalized_identifier == parse_fact_id(fact_id).instance_key`）；`tokenizer.py` |
| 9 | IR → SystemOutput | IR Materialization（Ref → 显示值 + `AnchorDeclaration` 位置计算 + Table cell 渲染）；fixture-specific display policy（显式声明，必须与 trusted `FactDisplaySpecSnapshot.decimals` 一致） | **Frozen（目标结构）+ Proposed（Adapter）** | `data_structures.py`；Coding Contract §13；04 §41.2 |
| 10 | SystemOutput → Step 7-E 七项 P0 → Step 7-F | Stage A（含 A-9 renderable / A-12 table_specs）→ … → Stage E 七条 P0；Run/Regression 消费 | **Frozen（验证侧，零修改）**；P0-2 在无表内自洽语义的明细表上为 **NOT_EVALUABLE**（见 §4.4-②） | `validator_framework.py`；05 §6.1-6.6、§7 |

**逐环结论**：十环均无 `BLOCKING`；不引入任何 Registry 变更、不引入任何 Frozen 变更、不引入任何未注册 fact_type。

### 3.2 混凝土场景（方案 A/B/C）阻塞环记录

| 环节 | 阻塞内容 | 状态 |
| ---- | ---- | ---- |
| Fact 环（试块值） | `sample.concrete_strength` 未注册 → 需 Registry 变更（**Proposed / Approval Required**，未获批） | 半阻塞（可通过授权解除，但不解决归属） |
| Domain / 归属环 | sample → component 归属无合法载体（Scheme 级） | **BLOCKING**（D-CONFLICT-001 本体） |
| Compute 环 | 分组键（归属）无法确定性解析；平均输出 fact_type 未注册 | **BLOCKING（依赖归属）** |
| IR/SystemOutput/验证环 | 无阻塞（与方案 D 相同） | 通过 |

**假设变更获批后的影响分析**（仅登记，不执行）：若未来批准 03 级方案（新增 Sample 类 + 归属字段 + `sample.concrete_strength` 注册 + 聚合输出类型注册），混凝土场景可完整实施；该路径的提案形态见 §7 预告。

### 3.3 确定性说明

- 本切片的确定性来源：列名精确匹配（待 Q1-Q6 验证，D-STEP8-03 保持 OPEN）+ 程序化判定/聚合 + 显式锚定；无 LLM 参与（OOS-3）。
- 命名约定不得作为归属依据（计划 E4 检查 3）：切片中不存在「由 `sample.id` 命名推断归属」的需求；归属一律由 `host_component_id` 显式承载。

---

## §4 最终结论

### 4.1 结论

> ## **结论 B：存在合规的最小替代 Vertical Slice ——「缺陷判定切片」**
>
> 原混凝土试块场景 **Deferred**；D-CONFLICT-001 **保持 OPEN / Deferred / 非当前路径，未宣告解决**（§5）。

### 4.2 替代切片定义（输入 / 事实类型 / 处理 / 判定 / IR / SystemOutput）

**输入（Excel，候选结构，待 Fixture Design 细化）**：

```text
Sheet1: 构件缺陷检查记录
行 1（Header）: 楼栋编号 | 构件编号 | 缺陷编号 | 最大宽度(mm) | 长度(m) | 形态描述
行 2+（Data）:  B01     | K001     | D001     | 0.15       | 2.3     | 斜向
               （候选规模：3-4 构件 × 每构件 1-2 缺陷；具体数据待 Fixture Design）
```

**事实类型（全部已注册，零 Registry 变更）**：

| Excel 列 | fact_type | quantity_kind | 说明 |
| ---- | ---- | ---- | ---- |
| 楼栋编号 | `building.id` | qualitative | 支撑 `Component.parent_structure_id`（必填） |
| 构件编号 | `component.id` | qualitative | 缺陷归属目标 |
| 缺陷编号 | `defect.id` | qualitative | 实例标识 token 锚定对象 |
| 最大宽度(mm) | `defect.width` | length | 单缺陷最大宽度——**不得与计算代表值混同** |
| 长度(m) | `defect.length` | length | 与 width 同量纲不同单位，验证单位语义 |
| 形态描述 | `defect.pattern` | qualitative | 定性 Fact |

**处理规则（程序执行，确定性）**：

1. Mapper：列名精确映射 → CandidateFact（Q1-Q6 待 fixture 验证，D-STEP8-03 保持 OPEN）。
2. Store：ingestion → `filled/pending` → 闸门 → `confirmed`。
3. Criterion 判定（合成测试参数，明确标注）：`defect.width`（或其组合数据）与限值比较（`value <= limit` → qualified / unqualified）；`review_status != confirmed` → `not_evaluable`。
4. ConclusionRule 最小聚合（按 `host_component_id` 分组到构件）：任一缺陷 unqualified → 构件 unqualified；全部 qualified → qualified；任一 not_evaluable → insufficient_evidence；空/缺失 → missing（凭 03 §27.1 约束「规则缺失时结论只能 missing」，不臆断分支细节）。

**判定输出**：Evaluation（`qualified / unqualified / not_evaluable`）+ Conclusion（`direction` 枚举 + `rule_id` + `input_evaluation_ids`）。

**Report IR**：Assertion 段（引 Ref）+ 缺陷明细表（单元格 Ref/Lit）+ 结论段（`direction` + `covers[]` 覆盖全部检测项 Evaluation）。

**SystemOutput**：Adapter Materialization → `report_ir` + `rendered_segments`（含 AnchorDeclaration）+ `structured_tables`（含 cell anchor_declarations）+ `system_evaluations`，完全匹配 `data_structures.py`。

### 4.3 业务行为证明（用户约束 #2 五要素）

| 要素 | 内容 | 为什么不是「解析/搬运/输出」 |
| ---- | ---- | ---- |
| **输入** | 构件缺陷检查记录 Excel（原始数据） | — |
| **处理规则** | 程序化 Criterion 判定 + ConclusionRule 聚合（03 §25.1 / §27.1 契约），非 LLM | 判定与聚合是**规则判断**与**逻辑合成**（项目规则「程序负责确定性：规则判断」），不是搬运 |
| **预期输出** | Evaluation.status（逐缺陷）+ Conclusion.direction（构件级，含 covers[]） | 输出是**新判定信息**（03 §26 L1254 明确允许产生新判定信息的核心位置），不是报告文本本身 |
| **来源追溯** | 每 Fact `source_refs` → Excel 单元格位置；数值 token 与实例标识 token 显式绑定 fact_id（04 §41.2） | 可追溯性是判定可信的前提，锚定链路独立于生成侧（05 §6.4） |
| **可验证业务结果** | S8-S-07（判定正确）/ S8-S-08（聚合正确）/ S8-S-10/11（Step 7-E 消费 + 七项 P0 产出结果）/ S8-S-12（回归） | 结果可由第三方（评测侧）独立验证，不依赖生成侧自证 |

**证据边界声明**：本轮为设计决策 Session，上述证明的性质是**契约级（字段 / 条款 / 链路）成立性论证**；运行级证据（真实 SystemOutput 与 P0 结果）将在 Coding / E2E 阶段产生（S8-S-10/11 为关键门禁）。两者不混同。

### 4.4 三项覆盖边界（如实记录，不掩盖）

1. **不覆盖统计 Compute**：切片最小路径无 average / max / min 统计聚合；M2 的统计计算能力验证延后。D-STEP8-07 保持 OPEN（关闭路径由 Fixture Design 重化为：「最小切片是否需要统计量」——若不引入聚合输出类型，则无需新注册）。
2. **P0-2 TABLE.INTERNAL 适用性边界**：明细表无合计/统计语义时，P0-2 依 v11 语义为 NOT_EVALUABLE（适用性由 trusted 侧显式声明，`validator_framework.py` A-12）。切片在其余 6 条 P0 上可正向验证；P0-2 的正向覆盖需含表内自洽语义的表格（依赖聚合计数值——最小路径不含）。
3. **混凝土场景 Deferred**：D-CONFLICT-001 不在当前路径；一旦恢复混凝土场景，该冲突与依赖它的分组 / 输出类型问题重新生效。

### 4.5 为什么不是 A / 为什么不是 C

- **不是 A**：MeasurementSet / Measurement 未构成可实施契约（§2.3），无任何已冻结字段可挂接；「类未实现」不是理由，「契约不足」才是。
- **不是 C**：选 C 的**前置条件是「不存在合规替代路径且必须修改 Frozen CDM Schema」**（用户约束 #1）。E4 已证明存在全链合规、零 Schema 变更的替代路径（方案 D），故 C 的触发条件不被满足；D-045 不提交（§7）。

---

## §5 D-CONFLICT-001 状态与适用范围

### 5.1 状态

```text
D-CONFLICT-001 = OPEN / DEFERRED / 非当前路径（未解决）
```

- **未解决**：归属表达能力的 Schema 缺口未消除；未通过改名、未通过换对象绕过治理、未修改任何 Registry 或 CDM 使其「看似消失」。
- **非当前路径**：当前选定路径（缺陷判定切片）不依赖该关联（`Defect.host_component_id` 承担构件↔缺陷归属），因此该冲突不再对当前路径构成 Coding-Blocking。
- **Deferred**：原混凝土试块场景（3 试块 → 平均代表值）延期；不在本轮或下一 Coding Round 实现。

### 5.2 适用范围与未来关闭条件（不变）

- **适用范围**：一切需要「多个 Sample 测量值归属同一 Component 并支持聚合」的场景（首例：混凝土抗压强度平行试块）。
- **关闭条件**（沿用 Closure §10.2，未放宽）：
  1. 由独立决策批准 03 层面的关联表达（Sample Domain Object / 关系字段 / 经授权的关联类型），或
  2. 在混凝土场景内给出**无需该关联**的合规最小替代方案并经证据验证。
- 恢复混凝土场景时，本冲突自动回到「当前路径阻塞」状态，须先走上述关闭条件。

### 5.3 禁止事项复述（继承并重申）

- 不得通过修改 `registry.py` 或 03 使该冲突看似消失；
- 不得伪造 `revision` / `supersedes`；
- 不得把 Conflict 裁决与 Fact 更正混为一谈；
- 不得把「改用其他业务对象」当作「原冲突已解决」。

---

## §6 D-STEP8-13 独立核查结论

### 6.1 证据

| 证据 | 内容 |
| ---- | ---- |
| `types.py` `Conflict` | 实例键 = `fact_id`；字段仅 `fact_id / candidates / resolution_policy / resolved_by / resolution`；**无 `conflict_id`、无状态转换字段** |
| 03 §34-§36 | Conflict 结构仅定义候选记录与人工裁决记录；未定义裁决后 Fact 的持久化转换 |
| 03 §15.5 | 「`conflict` 是横向来源分歧，`superseded` 是纵向版本更迭，**二者不得混用**」——裁决 ≠ 更正 |
| 核心禁令 | 不得假定原候选 Fact 转 `superseded`；不得伪造 revision / supersedes；`resolved` 不是合法 `Fact.status` |

### 6.2 对当前路径的影响

- 缺陷判定切片最小路径可设计为**单源数据、无冲突场景**（不激活 conflict 分支与 `resolve_conflict`）→ 该决策**不阻塞当前路径实施**。
- 但 S8-S-05 中 Fact Store 状态机的 conflict 分支完整性仍受该决策限制；Fact Store 的完整状态机验证延后。

### 6.3 状态

```text
D-STEP8-13 = OPEN（独立保持；本报告未构造、未暗示任何冲突裁决后的 Fact 状态转换）
```

---

## §7 D-045 状态

```text
D-045 = 不需要提交
PROPOSED / APPROVAL REQUIRED / FROZEN CDM MODIFIED = NO
```

- 不选结论 C，故不形成 D-045 提案，Frozen CDM 未修改。
- **预告**（仅登记未来触发条件）：若未来决定恢复混凝土场景且确定必须通过 Schema 级关联表达解决，则触发 D-045 级提案流程（问题 / 证据 / 不足论证 / 最小变更 / 影响 / 迁移 / 追溯 / 确定性 Compute / 验证策略 / 风险 / 待审批）。当前不提交。

---

## §8 自检 12 项 + 状态块 + 文件清单

### 8.1 自检 12 项

| # | 自检项 | 结果 |
| ---- | ---- | ---- |
| 1 | 决策报告已创建（E6 无条件产出） | ✅ 本文件 |
| 2 | 五分类矩阵完整（E1）；未因「类未实现」断言不支持、未因「概念示例」断言已冻结 | ✅ §1.1 / §1.6 |
| 3 | 方案 A/B/C/D 六维比较完整（E3） | ✅ §2.2 |
| 4 | 字段级链路逐环验证（E4）；每条引用文件 / 字段 / 条款；缺证据项如实标注 | ✅ §3.1（十环，无 BLOCKING） |
| 5 | 结论由证据决定，无预设（A 否决理由 = 契约不足；C 未触发；B 由 E4 支撑） | ✅ §2.3 / §4 |
| 6 | 结论 B 业务行为五要素证明完整（输入 / 规则 / 输出 / 追溯 / 可验证结果）+ 证据边界声明 | ✅ §4.3 |
| 7 | D-CONFLICT-001 保持 OPEN / Deferred / 非当前路径，未宣告解决 | ✅ §5 |
| 8 | D-STEP8-13 独立保持 OPEN；未构造任何 Fact 状态转换 | ✅ §6 |
| 9 | Frozen Docs（02/03/04/05/07）/ `registry.py` / 生产代码 / 测试 / Fixture 未修改 | ✅ §8.3（逐项核对） |
| 10 | 三份文档更新严格限定已证明结论；未将 Proposed 改为 Approved；未升级 READY FOR CODING | ✅ §8.3（E7 范围裁剪） |
| 11 | 未进入 Coding；未创建 Fixture；未自动开启下一阶段 | ✅ §8.4 |
| 12 | 最终 A-F 报告输出 + 如实报告 git 工作区状态（不以 `git diff` 为空证明无变化） | ✅ 会话末尾报告 |

### 8.2 最终状态块

```text
STEP 8 DESIGN STATUS     = NEEDS DECISION
STEP 8 CODING CONTRACT   = PROPOSED
STEP 8 FIXTURE DESIGN    = NOT STARTED
STEP 8 CODING            = BLOCKED
D-CONFLICT-001 STATUS    = OPEN / DEFERRED / 非当前路径（未解决）
D-STEP8-13 STATUS        = OPEN
FROZEN CDM MODIFIED      = NO
REGISTRY MODIFIED        = NO
PRODUCTION CODE MODIFIED = NO
FIXTURE CREATED          = NO
```

> 说明：门禁 4 / 8 因本轮结论更新为 PASS（基于合规替代路径证据），但门禁 1/2/3/5/6 仍 FAIL（D-STEP8-02/03/06/07/09 OPEN）；**整体状态不升级、不得进入 Coding。**

### 8.3 文件修改清单

**新建（本 Session 无条件产出）**：

| 文件 | 动作 |
| ---- | ---- |
| `docs/Step_8_Sample_Component_Relation_Design_Decision.md` | 新建（本报告） |

**条件式更新（E7，严格限定已证明结论；条件核验：结论 B 获证据支持 ✅ / 当前路径无未解除 Coding-Blocking 冲突 ✅（D-CONFLICT-001 移出当前路径、D-STEP8-13 不在最小路径激活）/ 所需决策已获授权 ✅（任务书三结论框架 + 「明确原混凝土试块方案是否延期」））**：

| 文件 | 更新点 | 边界 |
| ---- | ---- | ---- |
| `docs/Step_8_Design_Decision_Closure.md` | 头部状态、§2 矩阵 D-CONFLICT-001 行、§10.1/§10.2 复核结论、§14 门禁 4/8、OPEN 清单该行、状态块注记、下一阶段 | 不改其他决策行；不放宽关闭条件；整体状态保持 NEEDS DECISION |
| `docs/Step_8_Coding_Contract_v1.md` | §0 门禁 4/8、§4 Fixture 方向说明、§9 Compute 分组键注记、§19 Decision Register 两行、状态块（不变） | 不改为 READY FOR CODING；不将候选写成已批准 |
| `docs/Step_8_Fixture_Design_Sprint.md` | §1 Domain Direction（混凝土 Deferred + 缺陷方向）、§2/§3 冲突与 Compute 注记、§5 候选结构、§6 关闭条件前置、§8 门禁 d/h、尾部停止条件 | 不设计完整 fixture；不创建任何文件 |

**未修改（逐项核对）**：

| 对象 | 状态 |
| ---- | ---- |
| `docs/02_ARCHITECTURE.md` / `docs/03_CANONICAL_DATA_MODEL.md` / `docs/04_REPORT_IR.md` / `docs/05_EVALUATION.md` / `docs/07_DECISIONS.md` | 未修改 |
| `src/cdm/registry.py` | 未修改（`sample.concrete_strength`、聚合类型均未添加） |
| `src/**` 生产代码（含 `src/cdm/*`、`src/eval/accuracy/*`、`src/eval/run/*`） | 未修改 |
| `tests/**`（含全部 fixture） | 未修改；未创建任何 `.xlsx` / fixture 文件 |

### 8.4 停止声明

- 本轮止于 E6（+ 条件式 E7）+ E8 验证；**不进入 Coding、不创建 Fixture、不开启下一阶段**。
- 下一步（供后续 Session 决策，不自动执行）：按本报告结论 B 筹备 Fixture Design Session（缺陷判定切片方向），并处理 D-STEP8-02/03/06/07/09/13 的关闭路径。

---

*本报告的全部结论可由证据复核：每项判断均引用具体文件 / 字段 / 条款；未证实事项已标注为未证实，未推断为已支持。*