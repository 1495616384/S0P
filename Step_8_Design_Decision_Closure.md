# Step 8 Design Decision Closure Report

> **日期**：2026-10-08
> **前置**：Step 8 Design Discovery ✅（COMPLETE）→ 本 Session Decision Closure
> **状态**：**NEEDS DECISION**（关键决策 D-STEP8-02 / 03 / 06 / 07 / 09 为 OPEN）
> **本 Session 不创建任何生产代码**
> **Frozen Docs 不做任何修改**（02/03/04/05/07 全部保持原样）

---

## 1. Executive Decision

### Candidate A 是否正式采用？

**正式提议采纳 Candidate A：Zero-LLM M1-M3 Vertical Slice。**

### 为什么？

| # | 理由 | 证据 |
|---|---|---|
| 1 | **唯一能达成架构铁律 M3** | 02 §17.2："M3 结束时，系统在零 LLM 状态下已能端到端产出可校验报告——这是判断架构是否成立的唯一硬标准"。当前 M0 ✅ → M1 ❌ → M2 ❌ → M3 ❌，架构是否成立从未被验证。 |
| 2 | **唯一打通全部 6 个 P0 Blocker** | Discovery §3.4：Parser / Fact Store / Compute / Criterion / ConclusionRule / IR Builder 全部不存在。Candidate B（Parser Only）只打第一个，Candidate C（Pipeline Orchestrator）是空壳 |
| 3 | **唯一能形成真实验证循环** | 真实 Excel → 真实 SystemOutput → Step 7-E 7 条 P0 → Step 7-F Gate。当前 7 个 fixture 的 inputs/ 全部为空，验证侧是"完美接收端但没有发射端" |
| 4 | **14 条架构原则完全对齐** | D-004（先做无 LLM 骨架）、D-001（四层模型）、D-003（missing/conflict 不静默）、D-008（原始值+原始单位+quantity_kind）、02 §8.1（Pipeline 不是 Agent） |

### 为什么拒绝 B 和 C

| Candidate | 拒绝理由 |
|---|---|
| **B: Parser Only** | 范围过小。Parser 产出的 RawSource 无处可去——独立价值弱，无法验证架构铁律。是 A 的子集，但 Step 8 需要 A 的完整能力跃迁。 |
| **C: Pipeline Orchestrator** | 空壳工程。违反 02 §8.1 结论"一个路由函数即可"。没有下层 Step 的真实实现，Orchestrator 只是调用空桩的循环。空壳无法验证架构。 |

### Candidate A 当前状态

```text
PROPOSED → CLOSED（本 Session 证据化后正式采纳）
但 Step 8 Design Status = NEEDS DECISION（关键决策 OPEN）
Candidate A 的目标方向保持不变，但需要 Step 8 Coding Round 前补齐 OPEN 决策
```

---

## 2. Decision Matrix（三态 INHERITED / CLOSED / OPEN）

| Decision | Status | Decision 内容 | Evidence |
|---|---|---|---|
| D-STEP8-01 Vertical Slice Scope | **CLOSED** | Step 8 = M1(parsers/xlsx + facts/store) + M2(compute + rules + conclude) + M3(ir/builder + ir/adapter)，但只做**一个最小真实 Excel 业务案例**的完整贯通 | 02 §17.2 + Discovery §10 + OOS 完整排除了 Pipeline/LLM/Template |
| D-STEP8-02 Fixture Source | **OPEN** | workspace 中无任何真实 Excel/CSV 文件（Pre-Check 0a Glob 确认）。**需要人工构造 business fixture**，但 fixture 设计（sheet/header/字段/单位/fact_type 映射）尚待完成 | Pre-Check 0a：Glob `g:\workspace\zixun4\**\*.xlsx` → 0 个文件 |
| D-STEP8-03 RawSource → Fact Responsibility | **CLOSED（结构）+ OPEN（确定性程度）** | Parser/Mapper/Store 三层分离结构已明确（Parsing ≠ Semantic Interpretation 铁律）。但 Mapper 是否能完全确定性（Q1-Q6）无法在没有真实 fixture 的情况下证据化 | Pre-Check 0a 无 fixture；Pre-Check 0b 确认 Adapter 部分可确定性；§四 Q1-Q6 需 fixture 验证 |
| D-STEP8-04 Fact Store 状态机 | **CLOSED** | FactStore 状态机继承 CDM §15：candidate(pending) → confirmed/filled / missing / conflict → resolved → superseded；Conflict resolution_policy = manual | CDM §15 + §15.5 revision 链 |
| D-STEP8-05 Conflict resolution | **CLOSED** | 第一阶段必须 manual；Store 接口：`resolve_conflict(conflict_id, resolution_value, resolver_id)`；不实现自动裁决 | CDM §36 + project_memory hard constraint |
| D-STEP8-06 First Fact Types | **OPEN** | 依赖 fixture 字段。registry 已注册 15+ 类型（component.concrete_strength、defect.width 等），但 fixture 实际用哪些尚待 fixture 设计 | D-STEP8-02 |
| D-STEP8-07 First Compute set | **OPEN** | 依赖 fixture 的数据类型。预候选：average / max / min（统计量）；但需确认 fixture 是否有一组同类型数据需要聚合 | D-STEP8-02 |
| D-STEP8-08 First Criterion operators | **INHERITED** | 直接引用 CDM §25.1。Criterion.rule 字段的 operator 取值（>、>=、<、<=、==、between）由冻结 CDM 定义 | CDM §25.1 |
| D-STEP8-09 First ConclusionRule | **OPEN** | 依赖 fixture 的业务判定逻辑。预候选：单项合格 → 合格；任一不合格 → 不合格；missing → insufficient evidence | D-STEP8-02 |
| D-STEP8-10 Minimal Report IR output | **INHERITED** | 直接引用 04_REPORT_IR.md 全文 Schema。最小合法 IR = Document(version=0.3.1.1) + Section(semantic="inspection_result") + Table + Paragraph | 04_REPORT_IR.md |
| D-STEP8-11 SystemOutput projection | **CLOSED（Adapter 结构）** | IR→SystemOutput Adapter 需承担**IR Materialization**（Ref + Lit → 显示文本 + anchors 转换）。这不是 DOCX Renderer（OOS），而是 Step 8 确定性投影 | Pre-Check 0b 字段级分析 |
| D-STEP8-12 E2E success criteria | **CLOSED** | 12 条 S8-S-01 ~ S8-S-12（见 §八） | Discovery SC-1~SC-5 细化 |

### 决策分布

```text
INHERITED:    2  (D-STEP8-08, D-STEP8-10)
CLOSED:       4  (D-STEP8-01, D-STEP8-04, D-STEP8-05, D-STEP8-12)
OPEN:         6  (D-STEP8-02, D-STEP8-03 确定性程度, D-STEP8-06, D-STEP8-07, D-STEP8-09, + D-STEP8-03 完整)
```

---

## 3. Frozen Scope

### In Scope（Step 8 必须做的）

| # | 模块 | 路径 | 职责 |
|---|---|---|---|
| 1 | XLSX Parser | `src/parsers/xlsx.py` | Excel → RawSource + CellRaw list（纯解析，不涉及语义） |
| 2 | RawSource Schema | `src/parsers/base.py` | CellRaw dataclass（从 CDM 继承 RawSource，补充 cell 层级） |
| 3 | Fact Store | `src/facts/store.py` | 状态机 + Conflict 管理 + review_status 闸门 |
| 4 | Compute | `src/compute/unit_registry.py` + `src/compute/calculate.py` | 单位注册表 + 统计计算 |
| 5 | Criterion Engine | `src/rules/criterion.py` | Fact + Criterion → Evaluation |
| 6 | ConclusionRule | `src/rules/conclude.py` | Evaluations → Conclusion（最小聚合） |
| 7 | IR Builder | `src/ir/builder.py` | FactSet + Conclusion → Report IR dict |
| 8 | IR Materialization + Adapter | `src/ir/adapter.py` | Report IR → SystemOutput（Ref→显示值 + anchors 转换 + Table cell 渲染） |
| 9 | Unit Tests | `tests/parsers/`, `tests/facts/`, `tests/compute/`, `tests/rules/`, `tests/ir/` | 各模块单元测试 |
| 10 | Integration Test | `tests/integration/` | Excel → FactSet → Evaluation → IR 集成测试 |
| 11 | E2E Test | `tests/e2e/` | 真实 Excel → SystemOutput → Step 7-E evaluate_case() 消费 |
| 12 | synthetic_business_fixture | `tests/eval/fixtures/<case_id>/inputs/inspection.xlsx` | 首份 fixture（待 fixture 设计完成后创建） |

### Out of Scope（必须排除的）

| 编号 | 排除项 | 理由 |
|---|---|---|
| OOS-1 | DOCX Template Parser + Renderer（M4） | Step 8 止于 IR，DOCX 渲染是 Step 9 |
| OOS-2 | Pipeline Orchestrator + Step 框架 | 下层 Step 不存在时空壳；违反 02 §8.1 结论"一个路由函数即可" |
| OOS-3 | LLM Adapter + 任何 LLM Skill（M8） | D-004 铁律"先做无 LLM 骨架" |
| OOS-4 | Style Validator（M6） | 依赖 DOCX 渲染 |
| OOS-5 | PDF / 图片 / 自由文字 Parser | Step 8 只覆盖 xlsx |
| OOS-6 | 修改 src/eval/run/**（Step 7-F） | 冻结契约 |
| OOS-7 | 修改 src/eval/accuracy/**（Step 7-E） | 冻结契约 |
| OOS-8 | 修复 P-1 export defect | PRE-EXISTING Step 7-E 缺陷 |
| OOS-9 | 修改 docs/02/03/04/05/07 | project_memory hard constraint |
| OOS-10 | Multi-Agent / RAG / 向量数据库 / 知识图谱 / 微服务 / K8s / MQ | 过度工程化 |
| OOS-11 | 自动 Conflict 解决策略 | hard constraint |
| OOS-12 | 任何 Narrative 段落生成（需 LLM） | M9 内容；依赖 M8 |
| OOS-13 | TemplateSpec / fact_display_spec | 属于 M4 Template Parser 产出，Step 8 的 IR Materialization 使用默认精度（如无 TemplateSpec，用 CDM Fact 的原始单位 + 自然精度） |
| OOS-14 | Report IR Narrative 段生成 | Step 8 零 LLM；首份 Vertical Slice 只用 Assertion 段 + Table，不用 Narrative |
| OOS-15 | Ground Truth / expected_issues 设计 | 属于 Evaluation Case 设计（Step 6 已冻结），Step 8 只实现生成侧 |

---

## 4. RawSource → Fact Responsibility（核心问题）

### 铁律（必须坚持）

> **Parsing ≠ Semantic Interpretation**

### Parser 的严格职责

Parser 只做确定性的文件结构发现：

| 职责 | 说明 | 确定性？ |
|---|---|---|
| 文件打开 + Sheet 枚举 | openpyxl 直接读 | ✅ 100% |
| Cell 原始值提取 | cell.value / cell.data_type | ✅ 100% |
| Cell 位置坐标 | sheet_name + row + col + excel_address | ✅ 100% |
| 原始文本获取 | cell.value 转字符串 | ✅ 100% |
| 列头行识别 | 第一行非空行或显式 header_row 参数 | ✅ 100% |
| RawSource 构建 | source_id = filename, source_type = "xlsx" | ✅ 100% |

**Parser 绝对不做**（任何一项都属于语义推断）：
- scope_class 推断（"这个列是构件号 → component"）
- fact_type 推断（"这个列在数值列后面 → concrete_strength"）
- 实例 key 推断（"这个列是 K001 → 是 component 的 id"）
- 单位判断（"MPa 是压力单位"）
- 字段同义识别（"强度" = "抗压强度" = "砼强度"）
- 任何"这个列看起来像是..."的判断

### Mapper 的推荐架构 + 确定性程度问题

Mapper 作为独立层是推荐架构方向——这是 Parsing ≠ Semantic Interpretation 的自然延伸：Parser 产出 CellRaw（有位置有原始值但无语义），Mapper 把 CellRaw 映射到 CandidateFact（有 fact_id 有 fact_type）。

但 **Mapper 是否能完全确定性**必须通过第一份 Excel fixture 的真实结构验证后才能冻结。当前 workspace 无任何真实 Excel，无法实证。

#### 待 fixture 验证的 6 个问题（Pre-Check 0a 扩展）

| # | 问题 | 预期答案 | 影响 |
|---|---|---|---|
| Q1 | Excel 列名是否足够稳定，可以直接匹配到 fact_type registry？ | **待 fixture** | 如果列名混乱 → 不能完全 deterministic |
| Q2 | 是否需要 sheet / row / column context 才能正确映射？ | **待 fixture** | 如果需要上下文 → Mapper 更复杂但仍可确定性 |
| Q3 | 是否存在同义字段（如"混凝土强度"/"抗压强度"/"砼强度"都指向同一 fact_type）？ | **待 fixture** | 如果有同义 → 建立受控同义词表，仍然 deterministic |
| Q4 | 是否存在无法无歧义映射的字段？ | **待 fixture** | 如果存在 → 记录为 Design Finding，可能需要后续引入有限 LLM 或人工介入 |
| Q5 | 哪些映射属于确定性规则（列名精确匹配、受控词表匹配、位置规则）？ | **待 fixture** | 这部分可以冻结为 deterministic |
| Q6 | 哪些情况属于不确定性问题（需要语义理解）？ | **待 fixture** | 这部分不能硬做，需要记录为后续工作 |

#### 三种可能结果（取决于 fixture）

**结果 A：全部 Q1-Q6 回答为"确定性足够"**
- Mapper 可以冻结为完全确定性层
- D-STEP8-03 = CLOSED（结构 + 确定性）
- Candidate A 的 Zero-LLM 目标保持不变

**结果 B：大部分可以确定性，但有少量需要规则增强**
- Mapper 可以冻结为确定性层 + 受控同义词表 + 位置规则
- 仍然 Zero-LLM
- D-STEP8-03 = CLOSED（结构）+ OPEN（细节规则）

**结果 C：存在无法确定性映射的字段**
- 记录为 Design Finding
- D-STEP8-03 = CLOSED（结构）+ OPEN（部分字段映射）
- **影响 Candidate A 的 Zero-LLM 目标**——可能需要 Step 8.5 引入有限 LLM 或人工介入层

### Fact Store 的职责（继承 CDM）

```text
Candidate(pending) → confirmed(filled, review_status=confirmed)
                   → missing(status=missing, value=null)
                   → conflict(status=conflict, candidates=[...], resolution_policy=manual)
                   → rejected(status=rejected, review_status=rejected)
Conflict 解决（manual）→ resolve_conflict(conflict_id, resolution_value, resolver_id)
Revision 更正 → supersedes 链（CDM §15.5）
```

### Rule 层的职责

```text
Fact + Criterion → Evaluation（status ∈ qualified / unqualified / not_evaluable）
Criterion.review_status != confirmed → 自动 not_evaluable（02 §11.4 闸门）
Evaluations + ConclusionRule → Conclusion（最小聚合）
```

---

## 5. First Vertical Slice（数据流）

```
synthetic_business_fixture.xlsx（待 D-STEP8-02 设计完成）
 ↓
xlsx parser（openpyxl，确定性，不涉及语义）
 ↓
RawSource + List[CellRaw]（每个 cell 有 sheet/row/col/raw_value/raw_text/position）
 ↓
Mapper（确定性程度待 fixture 验证；见 §四 Q1-Q6）
 ↓
List[CandidateFact]（CDM Fact dataclass with status="pending", provenance="program"）
 ↓
Fact Store 状态机（CDM §15）
 ↓
Confirmed FactSet（status=filled, review_status=confirmed）
 ↓
Compute（average / max / min — 待 fixture 确认）
 ↓
Criterion Engine（operator 继承 CDM §25.1；review_status != confirmed → not_evaluable）
 ↓
List[Evaluation]
 ↓
ConclusionRule（最小聚合 — 待 fixture 确认逻辑）
 ↓
Conclusion
 ↓
IR Builder（FactSet + Conclusion → Report IR dict；符合 04 Schema）
 ↓
IR Materialization + Adapter（Ref → 显示值 + anchors 转换 + Table cell 渲染）
 ↓
SystemOutput（完全匹配 src/eval/accuracy/data_structures.py dataclass）
 ↓
Step 7-E evaluate_case(trusted, system) → CaseEvaluationResult
 ↓
Step 7-F run_evaluation() → RunSnapshot
```

### IR Materialization 是什么

IR Materialization 是 Step 8 Adapter 必须承担的确定性投影工作——**不是 DOCX Renderer**：

| IR Materialization | DOCX Renderer（OOS） |
|---|---|
| Ref("fact:component.K001.concrete_strength") → "32.4 MPa"（数值 + 单位 + 精度） | "32.4 MPa" → Word 段落 + 字体 + 字号 + 行距 |
| 把 Narrative prose + anchors 组装成 RenderedSegment.raw_text + AnchorDeclaration 列表 | 段落 → Word Paragraph + Style |
| 把 Table cell 的 Ref/Lit 变成 StructuredTable 的 TableCell.raw_text | Table → Word Table + 列宽 + 边框 |
| 产出 SystemOutput dataclass | 产出 DOCX 文件 |
| **Step 8 In Scope** | **OOS（M4）** |

精度规则：Step 8 无 TemplateSpec（OOS-13），使用默认精度——从 Fact.value 的 Python 精度推导（如 32.4 → 1 位小数）。这是确定性的，不涉及 LLM。

---

## 6. First Fixture Contract

### 当前状态：**OPEN**

workspace 中无真实 Excel 文件（Pre-Check 0a Glob 确认），无法使用真实 Excel。

### 决策方向：**需要人工构造 business fixture**

必须明确标记为 `synthetic_business_fixture = true`，绝对不能伪装为 `historical_real_data`。

### fixture 设计需要完成的工作

| # | 工作 | 产出 | 状态 |
|---|---|---|---|
| 1 | 选择 fixture 覆盖的 Scope | component + concrete_strength（混凝土抗压强度检测） | OPEN |
| 2 | 设计 Excel 结构 | sheet 名称 / header 行 / 数据行 / 字段名 | OPEN |
| 3 | 确定 fact_type 映射 | 每个列头对应 registry 中已注册的哪个 fact_type | OPEN |
| 4 | 确定单位 | MPa（压力） | OPEN（预计为 MPa，但需确认 fixture 字段） |
| 5 | 确定 scope_class | component（构件） | OPEN |
| 6 | 设计实例 key 规则 | K001 / K002 / ... 或 B01-K001 / ... | OPEN |
| 7 | 补充 fixture 周边文件 | ground_truth/facts.json + expected_issues.json + meta.json | OPEN |
| 8 | 决定 fixture 用哪个 case_id | tests/eval/fixtures/<case_id>/inputs/inspection.xlsx | OPEN |

### 不允许的做法

- 为了"推进"强行把 D-STEP8-02 标记为 CLOSED（没有 fixture 就没有证据）
- 使用完全虚构的字段（必须体现真实工程检测场景）
- 伪装成 historical real data

---

## 7. Minimal Module Scope

### 允许实现的模块

| 模块 | 输入 | 输出 | 关键接口 | 确定性 |
|---|---|---|---|---|
| `src/parsers/base.py` | 无（Schema 定义） | CellRaw dataclass | `CellRaw(sheet, row, col, raw_value, raw_text, position)` | 100% |
| `src/parsers/xlsx.py` | xlsx 文件路径 | ParseResult(RawSource, List[CellRaw]) | `parse_xlsx(path) → ParseResult` | 100% |
| `src/facts/store.py` | List[CandidateFact] | FactStore(confirmed, missing, conflict, rejected) | `add_candidates()`, `get_confirmed_facts()`, `get_conflicts()`, `resolve_conflict()` | 100%（CDM 继承） |
| `src/facts/conflict.py` | Conflict 对象 | 人工决策结果 | `resolve(conflict_id, decision, resolver_id)` | 100%（manual） |
| `src/compute/unit_registry.py` | 无（Registry 定义） | Unit → quantity_kind 映射 | `get_quantity_kind(unit)` | 100%（deterministic） |
| `src/compute/calculate.py` | List[Fact], statistic_type | computed Fact | `compute(facts, "avg"/"max"/"min") → Fact` | 100% |
| `src/rules/criterion.py` | Fact, Criterion | Evaluation | `evaluate(fact, criterion) → Evaluation` | 100% |
| `src/rules/conclude.py` | List[Evaluation], ConclusionRule | Conclusion | `conclude(evaluations, rule) → Conclusion` | 100% |
| `src/ir/builder.py` | FactSet, DomainObjects, Conclusion | Report IR dict | `build(fact_set, domain_objects, conclusion) → dict` | 100% |
| `src/ir/adapter.py` | Report IR dict, FactSet | SystemOutput | `to_system_output(report_ir, fact_set) → SystemOutput` | 100% |

### 禁止实现的模块

```text
pipeline/orchestrator   — OOS-2
llm/adapter             — OOS-3
llm/skills/*            — OOS-3
parsers/pdf             — OOS-5
parsers/image           — OOS-5
parsers/text            — OOS-5
render/docx             — OOS-1
ir/validate             — 后续 Step（Step 8 IR Builder 必须直接产出符合 04 的 IR，不需要额外 Validator）
```

---

## 8. Success Criteria（12 条）

| # | 编号 | 标准 | 验证方式 | 状态 |
|---|---|---|---|---|
| 1 | S8-S-01 | synthetic_business_fixture.xlsx 存在于 `tests/eval/fixtures/<case>/inputs/` | 文件存在 | 需 fixture 设计 |
| 2 | S8-S-02 | 真实 Excel 可被 xlsx parser 读取，产出 RawSource + CellRaw list | 单元测试 | 依赖 fixture |
| 3 | S8-S-03 | RawSource provenance 完整（source_id + location）；每个 CellRaw 精确对应 Excel 位置 | 单元测试 + 手动检查 JSON | 确定性（CDM 继承） |
| 4 | S8-S-04 | Mapper 产出的 Candidate Fact：fact_id 三段式正确 + fact_type 不变式成立 + registry 验证通过 | 单元测试（需 fixture 后） | 依赖 fixture |
| 5 | S8-S-05 | Fact Store 状态机正确：candidate→confirmed / missing / conflict 转换；Conflict resolution_policy = manual | 单元测试（状态转换矩阵） | CDM 继承 |
| 6 | S8-S-06 | Compute 结果与独立 expected truth 一致（人工计算对比） | 集成测试 | 依赖 fixture |
| 7 | S8-S-07 | Criterion Evaluation 正确（qualified / unqualified / not_evaluable 判定） | 单元测试 | CDM 继承 |
| 8 | S8-S-08 | Conclusion 聚合逻辑正确 | 单元测试 | 依赖 fixture |
| 9 | S8-S-09 | IR 合法且 deterministic（重复运行产出相同 IR；符合 04 Schema） | IR Schema 校验器 + 重复运行对比 | 100% 可验证 |
| 10 | S8-S-10 | SystemOutput 可被 Step 7-E 真实消费（evaluate_case() 不抛出类型错误） | E2E 测试 | **关键门禁** |
| 11 | S8-S-11 | Step 7-E 对真实 SystemOutput 跑 7 条 P0 规则，产出 CaseEvaluationResult | E2E 测试 | **关键门禁** |
| 12 | S8-S-12 | 已有 Step 7-D/E/F regression 全部通过（pytest） | pytest 全量运行 | 冻结回归 |

---

## 9. Out of Scope（冻结）

| 编号 | 排除项 | 理由 |
|---|---|---|
| OOS-1 | DOCX Template Parser + Renderer（M4） | Step 8 止于 IR，DOCX 渲染是 Step 9 |
| OOS-2 | Pipeline Orchestrator + Step 框架 | 违反 02 §8.1 |
| OOS-3 | LLM Adapter + LLM Skills（M8） | D-004 铁律 |
| OOS-4 | Style Validator（M6） | 依赖 DOCX 渲染 |
| OOS-5 | PDF / 图片 / 自由文字 Parser | Step 8 只覆盖 xlsx |
| OOS-6 | 修改 src/eval/run/**（Step 7-F） | 冻结契约 |
| OOS-7 | 修改 src/eval/accuracy/**（Step 7-E） | 冻结契约 |
| OOS-8 | 修复 P-1 export defect | PRE-EXISTING |
| OOS-9 | 修改 docs/02/03/04/05/07 | hard constraint |
| OOS-10 | Multi-Agent / RAG / 向量数据库 / 微服务 | 过度工程化 |
| OOS-11 | 自动 Conflict 解决 | hard constraint |
| OOS-12 | Narrative 段落生成（需 LLM） | M9 |
| OOS-13 | TemplateSpec / fact_display_spec | 属于 M4 |
| OOS-14 | Narrative 段 IR 生成 | Step 8 只做 Assertion + Table |
| OOS-15 | Ground Truth / expected_issues 设计 | Step 6 已冻结 |

---

## 10. Risks

| # | 风险 | 概率 | 影响 | 缓解 |
|---|---|---|---|---|
| R1 | **真实 Excel fixture 不可得** | MEDIUM | D-STEP8-02 = OPEN；Fixture 设计需额外 Session | 允许人工构造 business fixture，但必须明确标记 synthetic |
| R2 | **Mapper 无法完全确定性** | MEDIUM | Candidate A Zero-LLM 目标需调整；可能需要后续有限 LLM | 如实记录 Design Finding；结果 A/B/C 三种路径 |
| R3 | **CDM Schema 边界空洞** | LOW | 需 D-045 决策 | 发现即登记；不静默修改 03 |
| R4 | **Report IR → SystemOutput 语义断层** | **LOW** | 已证伪——IR Materialization 可确定性承担 | Pre-Check 0b 确认 Adapter 可确定性投影 |
| R5 | **Scope Creep（忍不住加 Pipeline/LLM）** | **HIGH** | 违反 OOS；延迟交付 | 严格对照 OOS 15 条；每写一个模块前确认 |
| R6 | **Fixture 设计引入不在 registry 的 fact_type** | MEDIUM | 需要先扩展 registry 才能注册 | D-STEP8-06 必须检查 registry 覆盖；如缺类型 → 先扩展 registry 不算修改 CDM Schema |

---

## 11. Known Non-Blockers（保留）

### P-1: Step 7-E export defect（PRE-EXISTING）
- **性质**：`src/eval/accuracy/__init__.py` 未导出 RuleStatus 等符号
- **是否影响 Step 8**：不影响。Step 8 只 import `data_structures.py` 的 SystemOutput 类型和 `validator_framework.py` 的 evaluate_case() 函数，不依赖根包导出

### N-1: D-041~D-044 未登记到 07_DECISIONS.md
- **性质**：Documentation Governance
- **是否影响 Step 8**：不影响技术实现

### N-2: _compute_four_hashes artifact byte_size=0
- **性质**：minimal implementation（Runner 层有意设计）
- **是否影响 Step 8**：可能关联——如果 Step 8 引入真实 artifact 可自然填充

### N-3: Fixture expected_issues 覆盖度有限
- **性质**：Fixture 质量（当前 7 case 仅 1 有 expected_issues）
- **是否影响 Step 8**：Step 8 引入真实 fixture 后自然扩展

---

## 12. Contract Deviation

**NONE**

本 Report 未偏离任何 Frozen Design。

---

## 13. Step 8 Coding Contract

参见配套文档：`docs/Step_8_Coding_Contract_v1.md`

状态标记：**PROPOSED**（决策不全，待 Step 8 Coding Round 前补齐 OPEN 决策后升级）

---

## 14. Final Status

### 决策分布

```text
INHERITED:     2  (D-STEP8-08, D-STEP8-10)
CLOSED:        4  (D-STEP8-01, D-STEP8-04, D-STEP8-05, D-STEP8-12)
OPEN:          6  (D-STEP8-02, D-STEP8-03 完整, D-STEP8-06, D-STEP8-07, D-STEP8-09)
Design Conflict: 0
```

### READY FOR CODING 门禁检查

| 门禁 | 状态 | 详情 |
|---|---|---|
| 关键决策非 OPEN | **FAIL** | D-STEP8-02(Fixture) + D-STEP8-03(Mapper 确定性) + D-STEP8-06(First Fact Types) + D-STEP8-07(Compute) + D-STEP8-09(ConclusionRule) 为 OPEN |
| 无 Design Conflict | PASS | Pre-Check 0b 确认 Adapter 可确定性承担 IR Materialization |
| Scope 已明确 | **部分通过** | D-STEP8-01 CLOSED，但具体 fixture 驱动的 fact types / compute / conclusion 尚 OPEN |
| Fixture 至少明确到可执行 | **FAIL** | D-STEP8-02 OPEN（无真实 Excel，synthetic fixture 设计未开始） |
| 责任边界已明确 | **部分通过** | Parsing ≠ Semantic Interpretation 铁律明确，但 Mapper 确定性程度待 fixture |
| Projection 已明确 | **通过** | D-STEP8-11 Adapter 结构 CLOSED（IR Materialization 可确定性） |

### OPEN 决策清单

| OPEN | 关键程度 | 关闭条件 |
|---|---|---|
| D-STEP8-02 Fixture Source | **CRITICAL** | 设计并创建 synthetic_business_fixture（8 个子工作项） |
| D-STEP8-03 Mapper 确定性 | **CRITICAL** | fixture 完成后回答 Q1-Q6 |
| D-STEP8-06 First Fact Types | **HIGH** | 依赖 fixture 字段 |
| D-STEP8-07 First Compute set | **HIGH** | 依赖 fixture 数据类型 |
| D-STEP8-09 First ConclusionRule | **HIGH** | 依赖 fixture 业务逻辑 |

### 最终状态判定

```text
STEP 8 DESIGN DISCOVERY = COMPLETE
STEP 8 DESIGN DECISION CLOSURE = INCOMPLETE（关键决策 OPEN）
STEP 8 DESIGN STATUS = NEEDS DECISION
STEP 8 CODING = NOT STARTED
```

### 下一阶段

```text
Step 8 Fixture Design Round（新 Session）
  ↓ 关闭 D-STEP8-02 + D-STEP8-06 + D-STEP8-07 + D-STEP8-09
  ↓ 回答 D-STEP8-03 的 Q1-Q6
  ↓ 重新跑本 Report 的 READY FOR CODING 门禁检查
  ↓ 如全部通过 → Step 8 Coding Round
```

---

*本 Session 到此停止。不创建任何生产代码。等待下一个 Session 关闭 OPEN 决策后进入 Coding。*
