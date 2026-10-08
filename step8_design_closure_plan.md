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

Candidate A（零 LLM M1-M3 Vertical Slice）仍然是唯一合理候选：

- **唯一能达成架构铁律 M3**（零 LLM 端到端）
- **唯一能形成真实验证循环**（真实 SystemOutput → Step 7-E）
- **唯一打通 6 个 P0 Blocker**
- 与 14 条架构原则完全对齐
- Candidate B（Parser Only）是 A 的子集，无法验证架构
- Candidate C（Pipeline Orchestrator）是空壳工程，违反 §8.1 结论

**Candidate A 保持为 PROPOSED，需在本 Session 冻结。**

### 需要关闭的 12 个决策（D-STEP8-01 ~ D-STEP8-12）

| 决策                                  | 核心问题                                                             | Frozen Document 是否有答案                                          |
| ------------------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------- |
| D-STEP8-01 Vertical Slice Scope       | 范围边界、模块清单                                                   | 部分（02 §17.2 M1-M3 有定义，但 Step 8 是 M1-M3 的最小切片）       |
| D-STEP8-02 Fixture Source             | 真实 Excel vs 人工构造                                               | **无**（需要决策）                                            |
| D-STEP8-03 RawSource → Fact 责任边界 | Parser 做什么？谁做语义映射？                                        | **无**（§五核心问题）                                        |
| D-STEP8-04 Fact Store 状态机          | candidate→confirmed→missing/conflict→resolved→rejected/confirmed | 部分（CDM §15 定义了状态，但转换逻辑未定义）                       |
| D-STEP8-05 Conflict resolution        | manual 闸门如何接入？                                                | 部分（CDM §36 说 manual，但接口未定义）                            |
| D-STEP8-06 First Fact Types           | Step 8 最小 fact_type 集合                                           | 部分（registry 已有 15+ 类型，但需确定 Step 8 首份 fixture 用哪些） |
| D-STEP8-07 First Compute set          | 1-3 个确定性计算                                                     | **无**（需根据 fixture 选择）                                 |
| D-STEP8-08 First Criterion operators  | 最小 operator 集合                                                   | 部分（CDN §25.1 有 rule 字段，但 operator 列表未冻结）             |
| D-STEP8-09 First ConclusionRule       | 最小聚合规则                                                         | 部分（CDM §27.1 有格式，但具体逻辑需根据 fixture）                 |
| D-STEP8-10 Minimal Report IR output   | 最小合法 IR 结构                                                     | 部分（04 §全文有 Schema，但"最小合法"的判断标准需定义）            |
| D-STEP8-11 SystemOutput projection    | IR → SystemOutput 的映射规则                                        | **无**（§十六核心问题）                                      |
| D-STEP8-12 E2E success criteria       | 12 条 S8-S-01 ~ S8-S-12                                              | 部分（Discovery 有 SC-1~SC-5，需细化为 12 条）                      |

### 架构风险复核

| 风险                             | 概率   | 缓解                                                           |
| -------------------------------- | ------ | -------------------------------------------------------------- |
| **R1 Parser 膨胀**         | MEDIUM | Parser 只做解析，语义映射留给独立 Mapper 层                    |
| **R2 Scope 膨胀**          | HIGH   | 严格 OOS 边界；每写一个模块前确认"在 Candidate A Scope 里吗？" |
| **R3 Synthetic E2E**       | HIGH   | 真实 Excel → 真实 SystemOutput，禁止中间换成 synthetic        |
| **R4 CDM/IR 二次建模**     | LOW    | 直接复用 src/cdm/types.py 的 Fact 和 SourceRef                 |
| **R5 IR 二次建模**         | LOW    | 直接复用 SystemOutput dataclass                                |
| **R6 Step 7-E/F 语义泄漏** | LOW    | F-INV-1/2 冻结，Step 8 只调用 evaluate_case()                  |

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

**Step 1: Executive Decision**

- 正式采纳 Candidate A：Zero-LLM M1-M3 Vertical Slice
- 拒绝 B（范围过小）和 C（空壳工程）
- 理由：唯一达成 M3 里程碑 + 唯一真实验证循环

**Step 2: Decision Matrix**

- 12 个决策逐项关闭，标记 CLOSED/OPEN，记录决策内容和证据

**Step 3: Frozen Scope**

- In Scope：parsers/xlsx、facts/store、compute + rules + conclude、ir/builder + adapter
- 明确排除 Pipeline Orchestrator（§8.1 结论保留为后续）
- 冻结模块清单：`src/parsers/base.py`、`src/parsers/xlsx.py`、`src/facts/store.py`、`src/facts/conflict.py`、`src/compute/unit_registry.py`、`src/compute/calculate.py`、`src/rules/criterion.py`、`src/rules/conclude.py`、`src/ir/builder.py`、`src/ir/adapter.py`

**Step 4: RawSource → Fact Responsibility（单独一节）**

- **Parser 做什么**：Excel → RawSource + CellRaw（sheet + row + col + raw_value + raw_text + position）
- **Mapper 做什么**：CellRaw → Candidate Fact（确定性映射，不涉及语义推断）
- **Fact Store 做什么**：Candidate → confirmed/missing/conflict（状态机）
- **Rule 做什么**：Fact + Criterion → Evaluation + Conclusion
- **关键决策**：Mapper 是独立层，禁止 Parser 膨胀为 Parser + Semantic Interpreter + Rule Engine

**Step 5: First Vertical Slice（冻结完整数据流）**

```
Excel fixture
 ↓
xlsx parser
 ↓
RawSource + CellRaw list
 ↓
Deterministic Mapper（列名→fact_type registry 匹配）
 ↓
Candidate Facts
 ↓
Fact Store 状态机
 ↓
Confirmed FactSet
 ↓
Compute（average / max / min）
 ↓
Criterion（<= threshold 判定）
 ↓
Evaluation
 ↓
ConclusionRule（全部合格→合格；任一不合格→不合格）
 ↓
Report IR（最小合法结构）
 ↓
SystemOutput adapter（映射到 Step 7-E dataclass）
 ↓
Step 7-E evaluate_case()
 ↓
Step 7-F run_evaluation()
```

**Step 6: First Fixture Contract**

- D-STEP8-02 决策：优先寻找真实 Excel；若无则人工构造业务真实 fixture
- fixture 必须体现真实业务字段（component_id、concrete_strength、crack_width 等）
- fixture 必须体现真实单位（MPa、mm、m、%）
- fixture 必须有完整 provenance 链
- fixture 路径：`tests/eval/fixtures/<case_id>/inputs/inspection.xlsx`

**Step 7: Minimal Module Scope**

- 列出每个允许实现的模块、输入、输出、关键接口

**Step 8: Success Criteria（12 条 S8-S-01 ~ S8-S-12）**

- S8-S-01: 真实 Excel 可被 parser 读取
- S8-S-02: RawSource provenance 完整
- S8-S-03: Candidate Fact 正确
- S8-S-04: Fact Store 状态正确
- S8-S-05: Compute 结果与独立 expected truth 一致
- S8-S-06: Criterion Evaluation 正确
- S8-S-07: Conclusion 正确
- S8-S-08: IR 合法且 deterministic
- S8-S-09: SystemOutput 可被 Step 7-E 消费
- S8-S-10: Step 7-E 对真实 SystemOutput 成功执行
- S8-S-11: Step 7-F Run 可记录
- S8-S-12: 已有 Step 7-D/E/F regression 全部通过

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

如果全部 12 个决策关闭且无 Design Conflict：

```
STEP 8 DESIGN DISCOVERY = COMPLETE
STEP 8 DESIGN DECISION CLOSURE = COMPLETE
STEP 8 DESIGN STATUS = READY FOR CODING
STEP 8 CODING = NOT STARTED
```

如果有未关闭决策：

```
STEP 8 DESIGN = NEEDS DECISION
```

---

### Phase 2: Step 8 Coding Contract（`docs/Step_8_Coding_Contract_v1.md`）

按用户指定的 21 节输出格式：

1. Objective
2. Scope
3. Architecture boundary
4. Input fixture
5. RawSource schema
6. Mapping responsibility
7. Fact Store
8. Conflict
9. Compute
10. Rules
11. Conclusion
12. Report IR
13. SystemOutput adapter
14. E2E flow
15. Tests
16. Success Criteria
17. Out of Scope
18. Risks
19. Decision Register
20. Contract Deviation Rules
21. Coding Stop Conditions

状态标记：

- 初始：**PROPOSED**
- 决策全部关闭后：**READY FOR CODING**
- 禁止标记：**FROZEN**（除非有正式冻结动作和证据）

---

## Dependencies and Considerations

### 已有 Frozen 依赖（只读消费）

| 依赖                            | 位置                                         | 说明                                                                              |
| ------------------------------- | -------------------------------------------- | --------------------------------------------------------------------------------- |
| CDM Fact + SourceRef + Conflict | `src/cdm/types.py`                         | Step 8 直接 import，不修改                                                        |
| FactTypeRegistry                | `src/cdm/registry.py`                      | Step 8 首份 fixture 必须使用 registry 中已有的 type（或先注册新的，但这不算修改） |
| SystemOutput dataclass          | `src/eval/accuracy/data_structures.py`     | IR adapter 必须产出此类型                                                         |
| Step 7-E evaluate_case()        | `src/eval/accuracy/validator_framework.py` | E2E 测试入口                                                                      |
| Step 7-F run_evaluation()       | `src/eval/run/runner.py`                   | 真实 Run 入口                                                                     |

### 需要冻结的新接口

| 接口                          | 说明                                                                     |
| ----------------------------- | ------------------------------------------------------------------------ |
| Parser 输出接口               | `ParseResult` = RawSource + List[CellRaw]                              |
| Mapper 输出接口               | `List[CandidateFact]`（直接是 CDM Fact 带 status="pending"）           |
| Fact Store 接口               | `add_candidate()` / `resolve_conflict()` / `get_confirmed_facts()` |
| Compute 接口                  | `compute_statistics(facts, statistic_type) → computed Fact`           |
| Criterion 引擎接口            | `evaluate(fact, criterion) → Evaluation`                              |
| ConclusionRule 接口           | `conclude(evaluations) → Conclusion`                                  |
| IR Builder 接口               | `build(fact_set, domain_objects, conclusion) → Report IR dict`        |
| IR→SystemOutput Adapter 接口 | `to_system_output(report_ir, fact_set) → SystemOutput`                |

### Unit 系统

- 沿用 CDM 的 `value + unit + quantity_kind` 三元组
- 单位换算只在 Compute 层发生
- 首份 fixture 涉及的单位：MPa、mm、m

### Design Conflict 检查

如果在 Closure 过程中发现以下情况：

1. CDM 无法覆盖某个 Step 8 需要的 Fact 模式 → D-045 决策
2. Report IR 无法表达某个 Step 8 需要的结构 → D-046 决策
3. Step 7-E 7 条 P0 有 Schema 空洞 → 登记但不静默修复

遇到任何 Conflict：标记 OPEN 状态，Step 8 DESIGN = NEEDS DECISION

---

## Validation

### 文档完整性验证

| 验证项                               | 方法                                        |
| ------------------------------------ | ------------------------------------------- |
| Decision Closure 14 节完整           | Checklist 逐项标记完成                      |
| Coding Contract 21 节完整            | Checklist 逐项标记完成                      |
| 12 个决策全部关闭（或明确标记 OPEN） | Decision Matrix 逐项检查                    |
| Design Conflict 登记                 | 如有 Conflict 必须有 Decision Register 条目 |
| Out of Scope 15 条完整               | OOS 表格逐项确认                            |

### 架构一致性验证

| 验证项                                                           | 方法                                      |
| ---------------------------------------------------------------- | ----------------------------------------- |
| Candidate A 符合架构铁律 M3                                      | 阅读 02 §17.2                            |
| Parser/Mapper/Store 边界符合"Parsing ≠ Semantic Interpretation" | 对照 §五原则                             |
| 不修改 Frozen docs                                               | Grep 搜索 docs/02~05/07 是否有修改        |
| 复用 CDM Fact schema 不发明第二套                                | 对照 src/cdm/types.py                     |
| 复用 SystemOutput dataclass                                      | 对照 src/eval/accuracy/data_structures.py |
| Conflict resolution_policy = manual                              | 对照 CDM §36                             |

### Success Criteria 可行性验证

| SC                                | 可行性                   | 验证方式                     |
| --------------------------------- | ------------------------ | ---------------------------- |
| S8-S-01 真实 Excel 可读取         | ✅ 可行（openpyxl 成熟） | 端到端测试                   |
| S8-S-02 provenance 完整           | ✅ 可行                  | 每个 Fact 带 source_refs     |
| S8-S-03 Candidate Fact 正确       | ✅ 可行                  | 单元测试                     |
| S8-S-04 Fact Store 状态正确       | ✅ 可行                  | 状态机单元测试               |
| S8-S-05 Compute 结果一致          | ✅ 可行                  | 独立计算 expected truth      |
| S8-S-06 Criterion Evaluation 正确 | ✅ 可行                  | 单元测试                     |
| S8-S-07 Conclusion 正确           | ✅ 可行                  | 单元测试                     |
| S8-S-08 IR 合法且 deterministic   | ✅ 可行                  | Schema 校验器 + 重复运行一致 |
| S8-S-09 SystemOutput 可消费       | ✅ 可行                  | 类型检查                     |
| S8-S-10 Step 7-E 成功执行         | ✅ 可行                  | 端到端测试                   |
| S8-S-11 Step 7-F Run 可记录       | ✅ 可行                  | runner 接口                  |
| S8-S-12 regression 通过           | ✅ 可行                  | pytest 全量回归              |

---

## Risks

| 风险                                        | 概率   | 影响                  | 缓解                         |
| ------------------------------------------- | ------ | --------------------- | ---------------------------- |
| **真实 Excel fixture 不可得**         | MEDIUM | D-STEP8-02 需重新决策 | 允许人工构造业务真实 fixture |
| **CDM Schema 边界空洞**               | MEDIUM | 需 D-045 决策         | 发现即登记，不静默修复       |
| **Report IR Schema 边界空洞**         | LOW    | 需 D-046 决策         | 发现即登记，不静默修复       |
| **Step 7-E P0 Schema 空洞**           | LOW    | 登记但不修 Step 7-E   | 设计发现→登记→后续决策     |
| **Scope Creep 忍不住加 Pipeline/LLM** | HIGH   | 违反 OOS              | 严格对照 OOS 列表            |

---

## Exit Criteria

### 成功出口

```
docs/Step_8_Design_Decision_Closure.md 存在且完整（14 节全部关闭）
docs/Step_8_Coding_Contract_v1.md 存在且完整（21 节，状态 = READY FOR CODING）
12 个 D-STEP8 决策全部 CLOSED
无未解决 Design Conflict
不修改任何 Frozen docs
不创建任何生产代码
```

### 失败出口

```
存在关键 Decision OPEN
存在未解决 Design Conflict
发现 Frozen doc 必须修改才能继续
→ STEP 8 DESIGN = NEEDS DECISION
→ 列出 OPEN-1 / OPEN-2 / ...
```

### 明确不算完成

```
只写出 Scope 不写 Decision
只写 Contract 不关闭决策
为了"推进"强行关闭有缺口的决策
静默修改 Frozen docs
```
