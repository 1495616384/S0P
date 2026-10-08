# Step 8 Design Decision Closure Plan

## Repository Research

### 当前状态（已确认）

| 里程碑                | 状态               | 说明                                        |
| --------------------- | ------------------ | ------------------------------------------- |
| Step 1-7-D            | ✅ ACCEPTED        | CDM Schema + 7 条 P0 + Run/Regression/Gate  |
| Step 7-E              | ✅ FROZEN          | 1650 行，7 条 P0，412 回归全绿              |
| Step 7-F              | ✅ FROZEN          | 1260 行，139 专属测试全绿                   |
| **生成侧**      | 🔴**零实现** | 无 parsers/ facts/ compute/ rules/ ir/ 目录 |
| fixtures/inputs/      | 🔴**全空**   | 全部 .gitkeep                               |
| SystemOutput 生产路径 | 🔴**不存在** | 无代码能生成真实 SystemOutput               |

### 已冻结的硬约束（不可修改）

| 约束                                        | 来源                                     | 影响                           |
| ------------------------------------------- | ---------------------------------------- | ------------------------------ |
| Conflict resolution_policy = manual         | CDM §36 + project_memory                | Step 8 只能实现 manual         |
| Fact ID 三段式`fact:<scope>.<key>.<attr>` | CDM v0.3.1                               | 禁止发明新格式                 |
| fact_type 不变式 =`<scope>.<attr>`        | CDM v0.3.1                               | 第一段必须等于 scope_class     |
| RawSource 只读                              | CDM §3                                  | Parser 不得修改 RawSource      |
| SystemOutput dataclass                      | `src/eval/accuracy/data_structures.py` | Step 8 必须精确产出此类型      |
| Step 7-E 唯一 P0 truth source               | F-INV-1                                  | Step 8 不得自己验证            |
| Step 7-F 不得重算 P0                        | F-INV-2                                  | Step 8 只消费 evaluate_case()  |
| 禁止修改 02/03/04/05/07 docs                | project_memory                           | 任何 schema 缺口需 D-045+ 决策 |

### Discovery Candidate A 复核结论

Candidate A（零 LLM M1-M3 Vertical Slice）仍然是当前推荐候选，但**必须经过本轮 Decision Closure 的证据化后才能正式采用**：

- **唯一能达成架构铁律 M3**（零 LLM 端到端）
- **唯一能形成真实验证循环**（真实 SystemOutput → Step 7-E）
- **唯一打通 6 个 P0 Blocker**
- 与 14 条架构原则完全对齐
- Candidate B（Parser Only）是 A 的子集，无法验证架构
- Candidate C（Pipeline Orchestrator）是空壳工程，违反 §8.1 结论

**Candidate A = PROPOSED，本轮目标：INHERITED / CLOSED 证据化。**

### 决策状态定义（三态）

| 状态                | 含义                               | 处理                   |
| ------------------- | ---------------------------------- | ---------------------- |
| **INHERITED** | Frozen Design 已明确，无需重新决策 | 引用 Frozen doc 章节号 |
| **CLOSED**    | 本轮有充分证据，正式决定           | 记录决策内容和证据     |
| **OPEN**      | 当前证据不足，需后续决策           | 明确列出需要什么证据   |

严禁为了达到"12/12 CLOSED"而强行关闭。

### 需要关闭的 12 个决策（D-STEP8-01 ~ D-STEP8-12）

| 决策                                  | 核心问题                                                             | 预期状态                           | 证据来源                                                   |
| ------------------------------------- | -------------------------------------------------------------------- | ---------------------------------- | ---------------------------------------------------------- |
| D-STEP8-01 Vertical Slice Scope       | 范围边界、模块清单                                                   | CLOSED                             | 02 §17.2 + Discovery §10 + OOS 列表                      |
| D-STEP8-02 Fixture Source             | 真实 Excel vs 人工构造                                               | **OPEN → 需调查**           | 检查 workspace 是否有真实 Excel fixture                    |
| D-STEP8-03 RawSource → Fact 责任边界 | Parser 做什么？谁做语义映射？                                        | CLOSED（结构）+ OPEN（确定性程度） | 02 §7 + §11.3 + 证据：Mapper 是否完全确定性              |
| D-STEP8-04 Fact Store 状态机          | candidate→confirmed→missing/conflict→resolved→rejected/confirmed | CLOSED                             | CDM §15 + §15.5 revision 链                              |
| D-STEP8-05 Conflict resolution        | manual 闸门如何接入？                                                | CLOSED                             | CDM §36 + project_memory hard constraint                  |
| D-STEP8-06 First Fact Types           | Step 8 最小 fact_type 集合                                           | **OPEN → 需 fixture**       | 依赖 D-STEP8-02 的 fixture 字段                            |
| D-STEP8-07 First Compute set          | 1-3 个确定性计算                                                     | **OPEN → 需 fixture**       | 依赖 fixture 的数据类型                                    |
| D-STEP8-08 First Criterion operators  | 最小 operator 集合                                                   | INHERITED                          | CDM §25.1 有 rule 字段定义                                |
| D-STEP8-09 First ConclusionRule       | 最小聚合规则                                                         | **OPEN → 需 fixture**       | 依赖 fixture 的业务判定逻辑                                |
| D-STEP8-10 Minimal Report IR output   | 最小合法 IR 结构                                                     | INHERITED                          | 04_REPORT_IR.md 全文 Schema                                |
| D-STEP8-11 SystemOutput projection    | IR → SystemOutput 的字段级映射                                      | **OPEN → 需字段分析**       | 04_REPORT_IR.md vs`src/eval/accuracy/data_structures.py` |
| D-STEP8-12 E2E success criteria       | 12 条 S8-S-01 ~ S8-S-12                                              | CLOSED                             | Discovery §10 SC-1~SC-5 细化                              |

### 架构风险复核

| 风险                             | 概率   | 缓解                                                                             |
| -------------------------------- | ------ | -------------------------------------------------------------------------------- |
| **R1 Parser 膨胀**         | MEDIUM | Parsing ≠ Semantic Interpretation 铁律；但 Mapper 是否完全确定性需 fixture 证据 |
| **R2 Scope 膨胀**          | HIGH   | 严格 OOS 边界；每写一个模块前确认"在 Candidate A Scope 里吗？"                   |
| **R3 Synthetic E2E**       | HIGH   | 真实 Excel → 真实 SystemOutput，禁止中间换成 synthetic                          |
| **R4 CDM/IR 二次建模**     | LOW    | 直接复用 src/cdm/types.py 的 Fact 和 SourceRef                                   |
| **R5 IR 二次建模**         | LOW    | 直接复用 SystemOutput dataclass                                                  |
| **R6 Step 7-E/F 语义泄漏** | LOW    | F-INV-1/2 冻结，Step 8 只调用 evaluate_case()                                    |

---

## Files and Modules

### 需要生成的文档（本 Session 唯一产出）

| 文件                                       | 性质                    | 状态           |
| ------------------------------------------ | ----------------------- | -------------- |
| `docs/Step_8_Design_Decision_Closure.md` | Decision Closure Report | **新建** |
| `docs/Step_8_Coding_Contract_v1.md`      | 正式 Coding Contract    | **新建** |

### 禁止修改的文件

```text
docs/02_ARCHITECTURE.md        — FROZEN
docs/03_CANONICAL_DATA_MODEL.md — FROZEN
docs/04_REPORT_IR.md           — FROZEN
docs/05_EVALUATION.md          — FROZEN
docs/07_DECISIONS.md           — 不主动修改
src/cdm/**                     — 只读
src/eval/**                    — 只读（冻结）
src/**                         — 零生产代码创建（parser/facts/compute/rules/ir）
tests/**                       — 零新增测试代码（本 Session 只做设计文档）
```

---

## Implementation Steps

### Phase 1: Decision Closure Report（`docs/Step_8_Design_Decision_Closure.md`）

按用户指定的 14 节输出格式逐步完成：

---

**Step 0: Pre-Check — 收集必需证据**

在写 Decision Matrix 之前，必须先做以下调查：

#### 0a. 检查 workspace 是否有真实 Excel fixture

```text
搜索范围：
g:\workspace\zixun4\tests\eval\fixtures\*\inputs\*.xlsx
g:\workspace\zixun4\**\*.xlsx（排除 .git 和 node_modules）
其他可能的 Excel 来源目录
```

如果找到真实 Excel：

- 读取其 sheet/header/row/字段结构
- 记录实际字段名
- 记录实际单位
- 评估是否能映射到 registry 中已有的 fact_type

如果找不到真实 Excel：

- D-STEP8-02 = OPEN（标记"需人工构造业务真实 fixture"）
- 后续决策（D-STEP8-06/07/09）也保持 OPEN 直到 fixture 设计完成

#### 0b. Report IR vs SystemOutput 字段级映射分析（为 D-STEP8-11 准备）

逐字段比对 `04_REPORT_IR.md` 的 Report IR Schema vs `data_structures.py` 的 SystemOutput dataclass：

| SystemOutput 字段                              | Report IR 对应                                          | 映射方式                             | 问题                                                                                                        |
| ---------------------------------------------- | ------------------------------------------------------- | ------------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| `report_ir: dict`                            | Report IR Document Schema（Section/Block/Inline/Table） | 直接透传                             | IR Builder 必须产出完全符合 04 的 dict                                                                      |
| `rendered_segments: List[RenderedSegment]`   | 04 §11-18 的 Assertion/Narrative 段落                  | **需要 projection**            | Assertion → rendered_segments？Narrative → rendered_segments？AnchorDeclaration 如何从 IR anchors 转换？  |
| `structured_tables: List[StructuredTable]`   | 04 §TableSpec                                          | **需要 projection**            | TableSpec → StructuredTable 的映射规则？TableCell.anchor_declarations 如何从 IR anchor_declarations 转换？ |
| `system_reported_issues: List[SystemIssue]`  | 无直接对应                                              | 可能是诊断性输出                     | Step 8 可以留空或产出基本诊断                                                                               |
| `system_evaluations: List[SystemEvaluation]` | 无直接对应                                              | 可能需要 Rule/Criterion 引擎同步产出 | Evaluation 是 Step 8 内部计算的，如何投影到 SystemEvaluation？                                              |

如果发现语义断层严重（例如 IR → rendered_segments 需要大量重新解释）：

- 标记 Design Conflict
- 不得修改 04 或 data_structures.py
- 记录问题，D-STEP8-11 = OPEN

---

**Step 1: Executive Decision**

- 正式提议采纳 Candidate A：Zero-LLM M1-M3 Vertical Slice
- 拒绝 B（范围过小）和 C（空壳工程）
- 本轮不直接冻结，需 Evidence 支撑

**Step 2: Decision Matrix（三态 INHERITED/CLOSED/OPEN）**

| Decision                                    | Status         | Decision                     | Evidence                                   |
| ------------------------------------------- | -------------- | ---------------------------- | ------------------------------------------ |
| D-STEP8-01 Vertical Slice Scope             | CLOSED         | ...                          | 02 §17.2 M1-M3 + Discovery §10 Scope     |
| D-STEP8-02 Fixture Source                   | **OPEN** | 调查后决定                   | Pre-Check 0a 结果                          |
| D-STEP8-03 RawSource → Fact Responsibility | CLOSED（结构） | Parser/Mapper/Store 三层分离 | 02 §7 §11.3；但 Mapper 确定性程度 = OPEN |
| D-STEP8-04 Fact Store 状态机                | CLOSED         | ...                          | CDM §15 + §15.5                          |
| D-STEP8-05 Conflict resolution              | CLOSED         | ...                          | CDM §36                                   |
| D-STEP8-06 First Fact Types                 | **OPEN** | 依赖 fixture                 | D-STEP8-02                                 |
| D-STEP8-07 First Compute set                | **OPEN** | 依赖 fixture                 | D-STEP8-02                                 |
| D-STEP8-08 First Criterion operators        | INHERITED      | 直接引用 CDM §25.1          | Frozen                                     |
| D-STEP8-09 First ConclusionRule             | **OPEN** | 依赖 fixture                 | D-STEP8-02                                 |
| D-STEP8-10 Minimal Report IR output         | INHERITED      | 直接引用 04                  | Frozen                                     |
| D-STEP8-11 SystemOutput projection          | **OPEN** | 字段级分析后决定             | Pre-Check 0b                               |
| D-STEP8-12 E2E success criteria             | CLOSED         | ...                          | Discovery SC-1~SC-5 细化                   |

**关键门禁条件**：READY FOR CODING 要求：

- D-STEP8-02 不是 OPEN（fixture 至少明确到可以执行）
- D-STEP8-03 的确定性程度不是 OPEN（或至少结构边界明确）
- D-STEP8-11 不是 OPEN（或 Adapter 至少定义到能编码接口）
- **没有未解决 Design Conflict**

如果以上任何条件不满足 → STEP 8 DESIGN = NEEDS DECISION

**Step 3: Frozen Scope**

- In Scope：parsers/xlsx、facts/store、compute + rules + conclude、ir/builder + adapter
- 明确排除 Pipeline Orchestrator（§8.1 结论保留为后续）
- 冻结模块清单（与 02 §17.2 对齐，但只列 Step 8 允许实现的）

**Step 4: RawSource → Fact Responsibility（单独一节，核心问题）**

#### 铁律（必须坚持）

> Parsing ≠ Semantic Interpretation

#### Parser 的严格职责（不涉及任何语义推断）

```text
文件解析
sheet / row / column 结构发现
cell.raw_value + cell.raw_text + cell position
原始 provenance 记录（source_refs）
列头识别（识别，但不解释）
```

**Parser 绝对不做**：

- scope_class 推断
- fact_type 推断
- 实例 key 推断
- 单位判断
- 字段同义识别
- 任何"这个列看起来像是..."的判断

#### Mapper 的推荐架构（但确定性程度需 fixture 证据）

Mapper 作为独立层是推荐架构方向，但"映射是否完全确定性"必须通过第一份 Excel fixture 的真实结构验证后才能冻结。

##### 需要分析的问题（Pre-Check 0a 完成后逐项回答）

| #  | 问题                                                                         | 回答                | 影响                                                       |
| -- | ---------------------------------------------------------------------------- | ------------------- | ---------------------------------------------------------- |
| Q1 | Excel 列名是否足够稳定，可以直接匹配到 fact_type registry？                  | （待 fixture 证据） | 如果列名混乱 → 不能完全 deterministic                     |
| Q2 | 是否需要 sheet / row / column context 才能正确映射？                         | （待 fixture 证据） | 如果需要上下文 → Mapper 更复杂                            |
| Q3 | 是否存在同义字段（如"混凝土强度"/"抗压强度"/"砼强度"都指向同一 fact_type）？ | （待 fixture 证据） | 如果有同义 → 需要受控词表，仍然 deterministic             |
| Q4 | 是否存在无法无歧义映射的字段？                                               | （待 fixture 证据） | 如果存在 → 记录为 Design Finding，可能需要 LLM 或人工介入 |
| Q5 | 哪些映射属于确定性规则（列名精确匹配、受控词表匹配、位置规则）？             | （待 fixture 证据） | 这部分可以冻结为 deterministic                             |
| Q6 | 哪些情况属于不确定性问题（需要语义理解）？                                   | （待 fixture 证据） | 这部分不能硬做，需要记录为后续工作                         |

##### 三种可能结果

**结果 A：全部 Q1~Q6 回答为"确定性足够"**

- Mapper 可以冻结为完全确定性层
- Candidate A 的 Zero-LLM 目标保持不变
- D-STEP8-03 = CLOSED（结构 + 确定性）

**结果 B：大部分可以确定性，但有少量需要规则增强**

- Mapper 可以冻结为确定性层 + 受控同义词表 + 位置规则
- 仍然 Zero-LLM
- D-STEP8-03 = CLOSED（结构）+ OPEN（细节规则）

**结果 C：存在无法确定性映射的字段**

- 记录为 Design Finding
- D-STEP8-03 = CLOSED（结构）+ OPEN（部分字段映射）
- 可能需要后续引入有限 LLM 或人工介入（但这会影响 Candidate A 的 Zero-LLM 目标）

#### Fact Store 的职责（继承 CDM）

```text
Candidate → confirmed / missing / conflict（状态机）
Conflict resolution_policy = manual
review_status 人工闸门
```

#### Rule 层的职责

```text
Fact + Criterion → Evaluation
Evaluations + ConclusionRule → Conclusion
```

**Step 5: First Vertical Slice（数据流草案，关键接口待定）**

```
Excel fixture（待 D-STEP8-02 确认）
 ↓
xlsx parser（确定性，不涉及语义）
 ↓
RawSource + CellRaw list
 ↓
Mapper（确定性程度待 fixture 证据；见 Step 4 Q1-Q6）
 ↓
Candidate Facts（status=pending, provenance=program）
 ↓
Fact Store 状态机（CDM §15）
 ↓
Confirmed FactSet（status=filled, review_status=confirmed）
 ↓
Compute（average / max / min — 待 fixture 确认）
 ↓
Criterion（operator 继承 CDM §25.1）
 ↓
Evaluation（status=qualified / unqualified / not_evaluable）
 ↓
ConclusionRule（最小聚合 — 待 fixture 确认）
 ↓
Report IR（04 Schema 最小合法结构）
 ↓
SystemOutput adapter（字段级映射 — 待 D-STEP8-11 确认）
 ↓
Step 7-E evaluate_case()
 ↓
Step 7-F run_evaluation()
```

**Step 6: First Fixture Contract（D-STEP8-02 决策后填充）**

- fixture 来源：Pre-Check 0a 结果
- 如果真实 Excel：记录 sheet / header / row / 字段 / 单位 / scope / fact_type 映射
- 如果人工构造：明确标记 `synthetic_business_fixture = true`
- 绝对不能伪装为 `historical_real_data`
- fixture 路径：`tests/eval/fixtures/<case_id>/inputs/inspection.xlsx`

**Step 7: Minimal Module Scope**

- 列出每个允许实现的模块、输入、输出、关键接口

**Step 8: Success Criteria（12 条 S8-S-01 ~ S8-S-12）**

**Step 9: Out of Scope（逐条冻结）**

- OOS-1: DOCX Template Parser
- OOS-2: DOCX Renderer
- OOS-3: PDF Parser
- OOS-4: Image Parser
- OOS-5: LLM Adapter
- OOS-6: LLM Skills
- OOS-7: RAG
- OOS-8: Knowledge Graph
- OOS-9: Vector Database
- OOS-10: Multi-Agent
- OOS-11: Kubernetes
- OOS-12: MQ/Redis
- OOS-13: Pipeline Orchestrator
- OOS-14: Step 7-E modification
- OOS-15: Step 7-F modification

**Step 10: Risks（只列真正风险）**

**Step 11: Known Non-Blockers（保留 P-1/N-1/N-2/N-3）**

**Step 12: Contract Deviation（NONE 或逐条列出）**

**Step 13: Step 8 Coding Contract（引用或内嵌）**

**Step 14: Final Status**

最终 Status 判定规则：

| 条件                                                                                | Status                                 |
| ----------------------------------------------------------------------------------- | -------------------------------------- |
| D-STEP8-02/03(确定性)/06/07/09/11 中有任何 OPEN + 无 Design Conflict                | NEEDS DECISION                         |
| 关键决策都不是 OPEN + 无 Design Conflict + Scope/Fixture/Boundary/Projection 都明确 | **READY FOR CODING**             |
| 有未解决 Design Conflict                                                            | NEEDS DECISION + 列出 CONFLICT-1/2/... |

---

### Phase 2: Step 8 Coding Contract（`docs/Step_8_Coding_Contract_v1.md`）

按用户指定的 21 节输出格式：

1. Objective
2. Scope
3. Architecture boundary
4. Input fixture
5. RawSource schema
6. Mapping responsibility（含确定性程度分析）
7. Fact Store
8. Conflict
9. Compute
10. Rules
11. Conclusion
12. Report IR
13. SystemOutput adapter（字段级映射表）
14. E2E flow
15. Tests
16. Success Criteria
17. Out of Scope
18. Risks
19. Decision Register
20. Contract Deviation Rules
21. Coding Stop Conditions

状态标记（严格按条件）：

- **PROPOSED**：初始状态，决策不全
- **READY FOR CODING**：Phase 1 全部关键决策 CLOSED，无 OPEN，无 Design Conflict
- 禁止标记：**FROZEN**（除非有正式冻结动作和证据）

---

## Dependencies and Considerations

### 已有 Frozen 依赖（只读消费）

| 依赖                            | 位置                                         | 说明                                                                              |
| ------------------------------- | -------------------------------------------- | --------------------------------------------------------------------------------- |
| CDM Fact + SourceRef + Conflict | `src/cdm/types.py`                         | Step 8 直接 import，不修改                                                        |
| FactTypeRegistry                | `src/cdm/registry.py`                      | Step 8 首份 fixture 必须使用 registry 中已有的 type；如需新 type 可注册但不算修改 |
| SystemOutput dataclass          | `src/eval/accuracy/data_structures.py`     | IR adapter 必须产出此类型                                                         |
| Step 7-E evaluate_case()        | `src/eval/accuracy/validator_framework.py` | E2E 测试入口                                                                      |
| Step 7-F run_evaluation()       | `src/eval/run/runner.py`                   | 真实 Run 入口                                                                     |

### 需要冻结的新接口（接口定义本身不涉及确定性假设）

| 接口                          | 说明                                                                     | 确定性程度                                   |
| ----------------------------- | ------------------------------------------------------------------------ | -------------------------------------------- |
| Parser 输出接口               | `ParseResult` = RawSource + List[CellRaw]                              | 100% 确定（Parser 是纯解析）                 |
| Mapper 输出接口               | `List[CandidateFact]`（CDM Fact with status="pending"）                | **待 fixture 验证**（见 Step 4 Q1-Q6） |
| Fact Store 接口               | `add_candidate()` / `resolve_conflict()` / `get_confirmed_facts()` | 100% 确定（CDM 继承）                        |
| Compute 接口                  | `compute_statistics(facts, statistic_type) → computed Fact`           | 100% 确定（程序计算）                        |
| Criterion 引擎接口            | `evaluate(fact, criterion) → Evaluation`                              | 100% 确定（CDM 继承）                        |
| ConclusionRule 接口           | `conclude(evaluations) → Conclusion`                                  | 待 fixture 确认聚合逻辑                      |
| IR Builder 接口               | `build(fact_set, domain_objects, conclusion) → Report IR dict`        | **待 D-STEP8-11 字段级分析**           |
| IR→SystemOutput Adapter 接口 | `to_system_output(report_ir, fact_set) → SystemOutput`                | **待 D-STEP8-11 字段级分析**           |

### Unit 系统

- 沿用 CDM 的 `value + unit + quantity_kind` 三元组
- 单位换算只在 Compute 层发生
- 首份 fixture 涉及的单位：**待 fixture 确认**

### Design Conflict 检查

如果在 Closure 过程中发现以下情况：

1. CDM 无法覆盖某个 Step 8 需要的 Fact 模式 → D-045 决策
2. Report IR 无法表达某个 Step 8 需要的结构 → D-046 决策
3. Step 7-E 7 条 P0 有 Schema 空洞 → 登记但不静默修复
4. Report IR → SystemOutput 存在无法无损衔接的语义断层 → CONFLICT-X

遇到任何 Conflict：标记 Conflict 状态，Step 8 DESIGN = NEEDS DECISION

---

## Validation

### 证据收集完整性验证

| 验证项                       | 方法                                |
| ---------------------------- | ----------------------------------- |
| Pre-Check 0a 真实 Excel 检查 | Glob 全 workspace                   |
| Pre-Check 0b 字段级映射分析  | 逐字段对比 04 vs data_structures.py |
| Step 4 Q1-Q6 全部回答        | Checklist 逐项填充                  |

### 文档完整性验证

| 验证项                                     | 方法                                        |
| ------------------------------------------ | ------------------------------------------- |
| Decision Closure 14 节完整                 | Checklist 逐项标记完成                      |
| Coding Contract 21 节完整                  | Checklist 逐项标记完成                      |
| 12 个决策都有状态（INHERITED/CLOSED/OPEN） | Decision Matrix 逐项检查                    |
| Design Conflict 登记                       | 如有 Conflict 必须有 Decision Register 条目 |
| Out of Scope 15 条完整                     | OOS 表格逐项确认                            |

### 架构一致性验证

| 验证项                              | 方法                                      |
| ----------------------------------- | ----------------------------------------- |
| Candidate A 符合架构铁律 M3         | 阅读 02 §17.2                            |
| Parsing ≠ Semantic Interpretation  | 对照 Step 4 Q1-Q6 + 02 §11.3             |
| 不修改 Frozen docs                  | Grep 搜索 docs/02~05/07 是否有修改        |
| 复用 CDM Fact schema 不发明第二套   | 对照 src/cdm/types.py                     |
| 复用 SystemOutput dataclass         | 对照 src/eval/accuracy/data_structures.py |
| Conflict resolution_policy = manual | 对照 CDM §36                             |

### READY FOR CODING 门禁

| 门禁                     | 条件                                 |
| ------------------------ | ------------------------------------ |
| 关键决策非 OPEN          | D-STEP8-02/03(确定性)/11 都不是 OPEN |
| 无 Design Conflict       | CONFLICT-X 列表为空                  |
| Scope 已明确             | D-STEP8-01 CLOSED + OOS 完整         |
| Fixture 至少明确到可执行 | D-STEP8-02 有具体来源/设计           |
| 责任边界已明确           | Parsing/Mapping/Store 三层           |
| Projection 已明确        | IR → SystemOutput 字段级映射        |

---

## Risks

| 风险                                         | 概率   | 影响                                | 缓解                                          |
| -------------------------------------------- | ------ | ----------------------------------- | --------------------------------------------- |
| **真实 Excel fixture 不可得**          | MEDIUM | D-STEP8-02 = OPEN                   | 允许人工构造 business fixture，但必须明确标记 |
| **Mapper 无法完全确定性**              | MEDIUM | Candidate A Zero-LLM 目标可能需调整 | 如实记录 Design Finding，不强行               |
| **CDM Schema 边界空洞**                | MEDIUM | 需 D-045 决策                       | 发现即登记，不静默修复                        |
| **Report IR → SystemOutput 语义断层** | MEDIUM | D-STEP8-11 = OPEN                   | 字段级分析后如实记录；不修改 Frozen docs      |
| **Scope Creep 忍不住加 Pipeline/LLM**  | HIGH   | 违反 OOS                            | 严格对照 OOS 列表                             |

---

## Exit Criteria

### READY FOR CODING 出口（严格满足全部）

```
docs/Step_8_Design_Decision_Closure.md 存在且完整（14 节）
docs/Step_8_Coding_Contract_v1.md 存在且完整（21 节，状态 = READY FOR CODING）
关键决策（D-STEP8-02/03/11）都不是 OPEN
无 Design Conflict
不修改任何 Frozen docs
不创建任何生产代码
```

### NEEDS DECISION 出口

```
存在任何关键 Decision OPEN（D-STEP8-02/03 确定性程度/11）
存在未解决 Design Conflict（CONFLICT-X）
→ STEP 8 DESIGN = NEEDS DECISION
→ 列出 OPEN-1 / OPEN-2 / ... 和 CONFLICT-1 / CONFLICT-2 / ...
```

### 明确不算完成

```
为了"推进"强行关闭有缺口的决策（标记 OPEN 为 CLOSED）
Mapper 没有 fixture 证据就假定完全确定性
D-STEP8-11 没有字段级分析就假定 IR→SystemOutput 可以直接衔接
静默修改 Frozen docs
```
