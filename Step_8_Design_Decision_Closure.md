# Step 8 Design Decision Closure Report

> **日期**：2026-10-08（原始）/ 2026-10-09（Cross-Document Consistency Repair + Fixture Design Sprint 关闭）
> **前置**：Step 8 Design Discovery ✅（COMPLETE）→ Decision Closure → Cross-Document Consistency Repair → Fixture Design Sprint（缺陷判定切片方向，2026-10-09）
> **状态**：**READY FOR CODING**（D-STEP8-02 / 03 / 06 / 07 / 09 已证据化 CLOSED；D-STEP8-13 处置决议＝最小路径不激活、保持 OPEN 且不构成 Coding-Blocking；D-CONFLICT-001 保持 OPEN / Deferred / 非当前路径（未宣告解决）——9/9 门禁 PASS，见 §14）
> **本 Session 不创建任何生产代码**
> **Frozen Docs 不做任何修改**（02/03/04/05/07 全部保持原样）
> **Registry 不修改**（`src/cdm/registry.py` 保持原样；本切片零 Registry 变更）

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
PROPOSED → CLOSED（证据化后正式采纳）
Step 8 Design Status = READY FOR CODING（Fixture Design Sprint 后关闭全部关键决策；见 §14）
Candidate A 方向保持；配合缺陷判定切片（决策报告结论 B）
```

---

## 2. Decision Matrix（三态 INHERITED / CLOSED / OPEN）

| Decision | Status | Decision 内容 | Evidence |
|---|---|---|---|
| D-STEP8-01 Vertical Slice Scope | **CLOSED** | Step 8 = M1(parsers/xlsx + facts/store) + M2(compute + rules + conclude) + M3(ir/builder + ir/adapter)，但只做**一个最小真实 Excel 业务案例**的完整贯通 | 02 §17.2 + Discovery §10 + OOS 完整排除了 Pipeline/LLM/Template |
| D-STEP8-02 Fixture Source | **CLOSED** | workspace 无真实 Excel（Pre-Check 0a）→ 人工构造 synthetic_business_fixture。**已创建**：`tests/eval/fixtures/case_step8_defect_v1/`（缺陷判定切片）——`inputs/inspection.xlsx`（单 sheet「构件缺陷检查记录」7×6：header + 6 数据行；B01 / K001-K003 / D001-D006）+ `meta.json`（synthetic_business_fixture=true, case_level=bronze, registry_extensions=[]）+ `notes.md`（映射 / Q1-Q6 / 显示策略） | Fixture Design Sprint Brief v2 §2/§8（8/8 条件已执行）；openpyxl 回读验证 7×6；文件实际存在 |
| D-STEP8-03 RawSource → Fact Responsibility | **CLOSED（结构 + 确定性）** | Parser/Mapper/Store 三层分离结构明确（Parsing ≠ Semantic Interpretation 铁律）。Mapper 确定性程度经 fixture 验证 = **结果 A（全部确定性）**：Q1 列名精确匹配 6/6；Q2 行级 context 导出 L3 归属（源自 `source_refs` 位置，非语义推断）；Q3 无同义字段（受控同义词表不建立、延后）；Q4 无不可映射字段；Q5 确定性规则集 = 列名精确匹配 + 行级 context 导出 + 单位取列头声明 + 实例 key＝单元格文本；Q6 本 fixture 内无不确定性（合并单元格/多级表头/跨行拼接为后续工作，边界如实声明） | Sprint Brief v2 §6；fixture `notes.md` §6（Q1-Q6 正式回答）；结果 A 判定 |
| D-STEP8-04 Fact Store 状态机 | **CLOSED**（枚举继承） | FactStore 严格继承 CDM §15，**禁止发明非法 status**：CandidateFact（transient，非 CDM Fact）→ 入库后成为 `status=filled, review_status=pending` 的合法 Fact（types.py 默认构造）→ 经 Store 闸门可转入 confirmed（review_status=confirmed）/ missing（value=None）/ conflict / rejected。**`superseded` 只属于 revision 更正链（§15.5），不是 Conflict 裁决结果**；Conflict 裁决后的持久化转换未定义 → D-STEP8-13（OPEN；处置决议：第一版不激活，见 Sprint Brief v2 §7）。**不存在 `pending` / `resolved` 作为 Fact.status**（registry.py FACT_STATUSES） | CDM §15 + §15.5 revision 链 + registry.py FACT_STATUSES + types.py Fact 默认值 |
| D-STEP8-05 Conflict resolution | **CLOSED（policy）/ OPEN（裁决后转换）** | 第一阶段必须 manual；`Conflict.resolution_policy='manual'`（CDM §36 冻结）。裁决后如何生成合法 Fact 属 OPEN（D-STEP8-13；处置决议：第一版不提供 `resolve_conflict`，见 Sprint Brief v2 §7）。不实现自动裁决 | CDM §36 + project_memory hard constraint + types.py Conflict（字段为 `fact_id` / `resolved_by` / `resolution`） |
| D-STEP8-06 First Fact Types | **CLOSED** | 第一版采用集 = 本切片 6 个 fact_type：`building.id` / `component.id` / `defect.id` / `defect.width` / `defect.length` / `defect.pattern`——**全部已注册，零 Registry 变更**（registry.py L272-294）。无任何 Proposed / 未注册类型；无计算输出类型 | Sprint Brief v2 §3；`registry.has()` 6/6 True（回读验证）；决策报告结论 B |
| D-STEP8-07 First Compute set | **CLOSED** | 第一版统计 Compute set = **∅（明确排除）**：最小判定链 = 逐缺陷比较（Criterion）+ Evaluation 聚合（ConclusionRule，非统计）；任何统计/聚合输出 fact_type 未注册 → 不引入。唯一 compute 模块 = `src/compute/unit_registry.py`（mm→length、m→length）。`calculate.py` / `compute_statistic()` 第一版不创建。缺失/冲突语义 = N/A（无统计输入聚合）。S8-S-06 重化为单位换算/量纲一致性 | Sprint Brief v2 §5；registry.py 实际注册集 |
| D-STEP8-08 First Criterion operators | **INHERITED** | 直接引用 CDM §25.1。Criterion.rule 字段的 operator 取值（>、>=、<、<=、==、between）由冻结 CDM 定义 | CDM §25.1 |
| D-STEP8-09 First ConclusionRule | **CLOSED** | 第一版 Criterion = `criterion-step8-defect-width-01`（`defect.width <= 0.30` mm → qualified；> 0.30 → unqualified；**0.30 为本切片合成测试参数，非规范限值**）。ConclusionRule = `step8_defect_slice_v1`：逐构件按 `host_component_id` 聚合（任一 unqualified → 构件 unqualified；否则任一 not_evaluable → insufficient_evidence；否则 qualified）→ 全局同逻辑（空输入 → missing）；unqualified 优先；`covers[]` = 全部 6 条 evaluation ids。预期全局 direction = "unqualified"。direction 值域（切片声明）：qualified / unqualified / insufficient_evidence / missing | Sprint Brief v2 §4；fixture `notes.md` §4 |
| D-STEP8-10 Minimal Report IR output | **INHERITED** | 直接引用 04_REPORT_IR.md 全文 Schema。最小合法 IR = Document(version=0.3.1.1) + Section(semantic="inspection_result") + Table + Paragraph | 04_REPORT_IR.md |
| D-STEP8-11 SystemOutput projection | **CLOSED（Adapter 结构）** | IR→SystemOutput Adapter 需承担**IR Materialization**（Ref + Lit → 显示文本 + anchors 转换）。这不是 DOCX Renderer（OOS），而是 Step 8 确定性投影 | Pre-Check 0b 字段级分析 |
| D-STEP8-12 E2E success criteria | **CLOSED** | 12 条 S8-S-01 ~ S8-S-12（见 §八） | Discovery SC-1~SC-5 细化 |
| D-STEP8-13 Conflict 人工裁决 → Fact 状态转换 | **OPEN（处置决议：最小路径不激活；非 Coding-Blocking）** | 处置决议（Sprint Brief v2 §7）：最小路径不激活 Conflict 分支与 `resolve_conflict`——第一版 FactStore **不提供**该入口（避免发明「裁决后转换」）；`mark_conflict` 仅记录 candidates（resolved_by / resolution 保持 None），不构成裁决后转换；不假定、不实现、不伪造 conflict → superseded / revision / supersedes（CDM §15.5：二者不得混用）；S8-S-05 覆盖 candidate→filled/pending→confirmed / missing / rejected + conflict 记录，裁决转换验证延后。D-STEP8-13 保持 OPEN，不构成当前路径 Coding-Blocking（门禁 8 组成部分） | Sprint Brief v2 §7；CDM §15.5 |
| D-CONFLICT-001 Component–Sample 关联 | **OPEN（Deferred / 非当前路径，未解决）** | `component.concrete_strength` 假设一个 Component 一个强度值；GB/T 50081-2019 要求每构件 3 个平行试块。现有 CDM 无法无歧义表达 sample→component 归属（见 §10.2 证据化分析）。**2026-10-09 设计决策（结论 B）**：选定合规最小替代切片（缺陷判定切片）不依赖该关联 → 冲突保留 OPEN / Deferred / 非当前路径，未宣告解决；原混凝土场景 Deferred | 决策报告 `Step_8_Sample_Component_Relation_Design_Decision.md` §3/§4/§5；registry.py 实际注册集（仅 `sample.id`）+ src/cdm/domain.py 实际类（无 Sample）+ CDM §17/§23/§24 仅为概念骨架 |

### 决策分布

```text
INHERITED:       2  (D-STEP8-08, D-STEP8-10)
CLOSED:         10  (D-STEP8-01, 02, 03, 04, 05, 06, 07, 09, 11, 12)
OPEN:            1  (D-STEP8-13：处置决议＝最小路径不激活，保持 OPEN、非 Coding-Blocking)
Design Conflict: 1  (D-CONFLICT-001：OPEN / Deferred / 非当前路径（未解决）；当前路径无 Coding-Blocking Conflict)
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
| 12 | synthetic_business_fixture | `tests/eval/fixtures/case_step8_defect_v1/inputs/inspection.xlsx` | 首份 fixture（**已创建**：缺陷判定切片，7×6；见 §6 / §2 矩阵 D-STEP8-02） |

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
| OOS-13 | TemplateSpec / fact_display_spec | 属于 M4 Template Parser 产出；Step 8 的 IR Materialization 使用 **fixture-specific display policy**（临时、非系统级规则），不宣称是最终 TemplateSpec 规则 |
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

但 **Mapper 是否能完全确定性**必须通过第一份 Excel fixture 的真实结构验证后才能冻结。**[2026-10-09 更新：fixture `case_step8_defect_v1` 已创建并验证 → Q1-Q6 回答 = 结果 A（全部确定性）→ D-STEP8-03 CLOSED（结构 + 确定性）；原「当前 workspace 无任何真实 Excel，无法实证」状态已解除。]**（见 §2 矩阵 / Sprint Brief v2 §6 / fixture `notes.md` §6）

#### Q1-Q6（**已由 fixture 验证 = 结果 A**，Pre-Check 0a 扩展）

| # | 问题 | 验证结果（2026-10-09） | 影响（本 fixture 下） |
|---|---|---|---|
| Q1 | Excel 列名是否足够稳定，可以直接匹配到 fact_type registry？ | **是**——6 个固定列头逐一精确匹配（Sprint Brief v2 §3） | 列名精确匹配（已冻结） |
| Q2 | 是否需要 sheet / row / column context 才能正确映射？ | **需要行级 context**——同行 6 列联合用于 L3 归属与实例分组；来源 = `source_refs` 位置（结构信息，非语义推断） | 位置规则（已冻结） |
| Q3 | 是否存在同义字段（如"混凝土强度"/"抗压强度"/"砼强度"都指向同一 fact_type）？ | **无**（每列唯一含义）；受控同义词表**不建立**（延后到出现真实变体） | 同义词表（未启用，延后） |
| Q4 | 是否存在无法无歧义映射的字段？ | **无**——6 列全部一行一义 | 无 Design Finding |
| Q5 | 哪些映射属于确定性规则（列名精确匹配、受控词表匹配、位置规则）？ | **(a) 列名精确匹配；(b) 行级 context 导出 L3 归属与分组；(c) 原始单位取自列头声明；(d) 实例 key = 单元格文本** | 已冻结为 deterministic |
| Q6 | 哪些情况属于不确定性问题（需要语义理解）？ | **本 fixture 内无**；边界如实声明：合并单元格 / 多级表头 / 跨行拼接不在本 fixture 内（后续工作） | 无（边界已声明） |

#### 三种可能结果（**裁定：结果 A**）

**结果 A：全部 Q1-Q6 回答为"确定性足够"** ✅ **已采用（2026-10-09）**
- Mapper 可以冻结为完全确定性层
- D-STEP8-03 = CLOSED（结构 + 确定性）
- Candidate A 的 Zero-LLM 目标保持不变

**结果 B：大部分可以确定性，但有少量需要规则增强**（未触发）
- Mapper 可以冻结为确定性层 + 受控同义词表 + 位置规则
- 仍然 Zero-LLM
- D-STEP8-03 = CLOSED（结构）+ OPEN（细节规则）

**结果 C：存在无法确定性映射的字段**（未触发）
- 记录为 Design Finding
- D-STEP8-03 = CLOSED（结构）+ OPEN（部分字段映射）
- **影响 Candidate A 的 Zero-LLM 目标**——可能需要 Step 8.5 引入有限 LLM 或人工介入层

### Fact Store 的职责（严格继承 CDM，禁止非法 status）

```text
CellRaw → Mapper → CandidateFact（transient，非 CDM Fact，不带合法 status）
                          ↓
                    Fact Store ingestion
                          ↓
                   Fact(status=filled, review_status=pending)   ← types.py 默认构造
                          ↓ Store 闸门
              ┌───────────┼────────────┬─────────────┐
              ↓           ↓            ↓             ↓
      confirmed      missing       conflict     rejected
   (review_status   (value=null)   (candidates)  (status=rejected)
    =confirmed)                     (resolution
                                    _policy=manual)
                                    
Conflict 记录 → mark_conflict(fact_id, candidates)（第一版**仅记录**；policy=manual）
  → resolved_by / resolution 保持 None；不构成「裁决后转换」
  → resolve_conflict 第一版**不提供**（D-STEP8-13 处置决议：最小路径不激活）
  → 裁决后如何生成合法 Fact（status / revision / supersedes）在冻结 CDM 中未定义
    → **D-STEP8-13 OPEN**；不得假定原候选转 superseded

Revision 更正（与 Conflict 无关）→ supersedes 链（CDM §15.5）
  → NEW(revision=N, supersedes=[...]) 写入；OLD(revision=N-1) 的 status=superseded
```

**关键禁令：**
- `pending` 是 `review_status` 的合法值，**绝不是** `Fact.status` 的合法值（registry.py FACT_STATUSES）
- `resolved` **不存在**于 `Fact.status` 枚举
- CandidateFact 是 Mapper→Store 之间的**瞬态桥接对象**，不得伪装成带非法 status 的 CDM Fact
- **不得**假定 Conflict 人工裁决后原候选 Fact 自动变为 `superseded`（CDM §15.5：`conflict` 是横向来源分歧、`superseded` 是纵向版本更迭，「二者不得混用」）。裁决后的持久化转换未定义 → D-STEP8-13（OPEN；处置决议：最小路径不激活，见 Sprint Brief v2 §7）

### CandidateFact 与 FactStore 接口（**Proposed Interface** — 尚未实现）

以下对象/接口是 Mapper 与 Fact Store 之间的**设计提案**；当前 workspace 中不存在对应实现（无 `src/facts/`）。D-STEP8-13 处置决议：最小路径不激活 Conflict 裁决转换，`resolve_conflict` 第一版**不提供**（见 §2 / Sprint Brief v2 §7）。

```python
@dataclass
class CandidateFact:
    """[Proposed] Mapper → Store 的瞬态桥接对象。不是 CDM Fact。"""
    fact_id: str            # 三段式；满足 fact_type 不变式
    fact_type: str          # 必须已注册（registry.py）
    value: FactValue
    unit: Optional[str]
    quantity_kind: str
    source_refs: List[SourceRef]
    method: str             # measured / observed / computed / quoted / inferred
    provenance: str         # program / ai / human
    # 不带 status / review_status / revision —— 入库时由 types.py Fact 默认构造补齐

class FactStore:
    """[Proposed] 状态机接口；严格继承 CDM §15 / §15.5。"""
    def ingest_candidates(self, candidates: List[CandidateFact]) -> None: ...
    def get_confirmed_facts(self) -> List[Fact]: ...      # status=filled 且 review_status=confirmed
    def get_all_facts(self) -> List[Fact]: ...
    def get_conflicts(self) -> List[Conflict]: ...        # 键 = Conflict.fact_id
    def mark_missing(self, fact_id: str) -> None: ...
    def mark_conflict(self, fact_id: str, candidates: List[ConflictCandidate]) -> None: ...
    def mark_rejected(self, fact_id: str, reason: str) -> None: ...
    # resolve_conflict(fact_id, resolution_value, resolver_id) —— 第一版不提供
    #   （D-STEP8-13 处置决议：最小路径不激活；裁决后 Fact 状态转换在冻结 CDM 中未定义，禁止发明）
```

> 接口命名以冻结数据类为准：`Conflict` 的实例键是 `fact_id`（`src/cdm/types.py` 无 `conflict_id` 字段）——若未来启用裁决入口，统一使用 `fact_id` 作为键；任何 `resolve()` / `resolve_conflict(conflict_id, ...)` 的旧写法均已废弃。

### Rule 层的职责

```text
Fact + Criterion → Evaluation（status ∈ qualified / unqualified / not_evaluable）
Criterion.review_status != confirmed → 自动 not_evaluable（02 §11.4 闸门）
Evaluations + ConclusionRule → Conclusion（最小聚合）
```

---

## 5. First Vertical Slice（数据流）

```
case_step8_defect_v1/inputs/inspection.xlsx（缺陷判定切片 fixture，已创建）
 ↓
xlsx parser（openpyxl，确定性，不涉及语义）
 ↓
RawSource + List[CellRaw]（每个 cell 有 sheet/row/col/raw_value/raw_text/position）
 ↓
Mapper（确定性程度已验证 = 结果 A；D-STEP8-03 CLOSED，见 §2）
 ↓
List[CandidateFact]（**瞬态桥接对象**，携带 fact_id/fact_type/value/unit/source_refs，但**不带合法 status**；不得伪装成 CDM Fact）
 ↓
Fact Store ingestion（transient → 合法 Fact）
 ↓
List[Fact](status=filled, review_status=pending)   ← 入库即用 types.py 默认构造
 ↓
Fact Store 闸门（review_status 人工确认 / missing 标记 / conflict 识别）
 ↓
Confirmed FactSet（status=filled, review_status=confirmed）
 ↓
Compute（2026-10-09 决策更新：按替代切片方向由 Fixture Design 裁定最小 Compute set——当前路径无统计聚合；原混凝土方向 average / max / min 预候选 Deferred，分组依赖 D-CONFLICT-001 已解除）
 ↓
Criterion Engine（operator 继承 CDM §25.1；review_status != confirmed → not_evaluable）
 ↓
List[Evaluation]
 ↓
ConclusionRule（`step8_defect_slice_v1` — host_component_id 逐构件聚合 → 全局；D-STEP8-09 CLOSED）
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

显示精度：Step 8 无 TemplateSpec（OOS-13），使用 **fixture-specific display policy**——一份**显式声明**的临时策略（按 `fact_type` 规定小数位与单位后缀），**不得**以 Python `float` 的 `repr` / 自然精度作为格式规则。该策略：
- 仅对本次 Vertical Slice 生效；
- 明确指定数值精度、单位显示与显示格式；
- **不宣称**是最终 TemplateSpec 规范；正式显示规格的唯一真相源是 04_REPORT_IR.md §17 的 `TemplateSpec.fact_display_spec`（OOS-13，M4 负责）；
- 必须与 Step 7-E P0-1 使用的 trusted `FactDisplaySpecSnapshot.decimals` 一致（INV-19），否则 P0-1 FAIL。

`AnchorDeclaration` 的数字锚定必须与实际显示文本、对应 Fact 及冻结 P0 规则（04 §41.2）一致。Report IR Materialization 与 DOCX Rendering 必须分离。

---

## 6. First Fixture Contract

### 状态：**CLOSED**（2026-10-09 Fixture Design Sprint）

> **2026-10-09 决策更新（结论 B）**：fixture 方向由「混凝土抗压强度」调整为「缺陷判定切片」（决策报告 §4.2）；原混凝土方向内容随结论 B Deferred（不再承载实施）。D-STEP8-02 经 Fixture Design Sprint 证据化关闭。

workspace 中无真实 Excel 文件（Pre-Check 0a Glob 确认）→ fixture 为**人工构造**，必须明确标记 `synthetic_business_fixture = true`，绝对不能伪装为 `historical_real_data`。

### 已创建 fixture（三件交付物，唯一定稿）

| 项 | 值 |
|---|---|
| case_id | `case_step8_defect_v1` |
| 目录 | `tests/eval/fixtures/case_step8_defect_v1/` |
| 交付物 | `inputs/inspection.xlsx` + `meta.json` + `notes.md`（仅此三项） |
| Sheet / 规模 | 「构件缺陷检查记录」/ 1 header 行 + 6 数据行 × 6 列 |
| 列 | 楼栋编号 \| 构件编号 \| 缺陷编号 \| 最大宽度(mm) \| 长度(m) \| 形态描述 |
| 实例 key | B01 / K001-K003 / D001-D006（全局唯一；无连字符——冻结 tokenizer 兼容约束） |
| 单位 | mm（宽度）/ m（长度），quantity_kind 均 = length |
| 覆盖 | 结果行含 qualified（含 D002 = 0.30 边界等值）与 unqualified（D003 超限） |

### fixture 设计交付物工作项（全部已执行）

| # | 工作 | 产出 | 状态 |
|---|---|---|---|
| 1 | fixture 覆盖 Scope | defect 判定切片（building / component / defect 三层实例） | ✅ 已执行 |
| 2 | Excel 结构 | 单 sheet / header 行 1 / 6 数据行 / 6 列 | ✅ 已执行 |
| 3 | fact_type 映射 | 6/6 列头 → 全部已注册 fact_type（**零 Registry 变更**） | ✅ 已执行（Sprint Brief v2 §3） |
| 4 | 单位 | mm（宽度）/ m（长度） | ✅ 已执行 |
| 5 | scope_class | building / component / defect | ✅ 已执行 |
| 6 | 实例 key 规则 | B01 / K001-K003 / D001-D006（无连字符） | ✅ 已执行 |
| 7 | fixture 周边文件 | `meta.json` + `notes.md`（**无** ground_truth / expected_issues——OOS-15） | ✅ 已执行 |
| 8 | case_id | `case_step8_defect_v1` | ✅ 已执行 |

> **权威来源**：完整规格 / Q1-Q6 回答 / 判定链 / 显示策略见 `Step_8_Fixture_Design_Sprint.md`（v2）与 fixture `notes.md`；本 Report 不再重复。验收证据：openpyxl 回读 7×6 与规格一致；`registry.has()` 6/6 True；OOS-15 边界（目录内无 ground_truth / expected_issues / known_issues）。

### 不允许的做法（保留；本次关闭的合规性声明）

- 为了"推进"强行把 D-STEP8-02 标记为 CLOSED——**本次关闭的依据是实际创建并回读验证的文件**（非声称）
- 使用完全虚构的字段——**本 fixture 使用受控词表与已注册 fact_type**，非脱离工程场景的杜撰
- 伪装成 historical real data——**meta.json 显式 `synthetic_business_fixture=true`**

---

## 7. Minimal Module Scope

### 允许实现的模块

| 模块 | 输入 | 输出 | 关键接口 | 确定性 |
|---|---|---|---|---|
| `src/parsers/base.py` | 无（Schema 定义） | CellRaw dataclass | `CellRaw(sheet, row, col, raw_value, raw_text, position)` | 100% |
| `src/parsers/xlsx.py` | xlsx 文件路径 | ParseResult(RawSource, List[CellRaw]) | `parse_xlsx(path) → ParseResult` | 100% |
| `src/facts/store.py` | List[CandidateFact] | FactStore（confirmed / missing / conflict / rejected） | **Proposed**：`ingest_candidates()`, `get_confirmed_facts()`, `get_all_facts()`, `get_conflicts()`, `mark_missing()`, `mark_conflict()`, `mark_rejected()`；`resolve_conflict` **第一版不提供**（D-STEP8-13 处置） | 枚举 100%（CDM 继承）；裁决转换不激活（D-STEP8-13） |
| `src/facts/conflict.py` | Conflict 记录辅助 | Conflict 记录（candidates + policy=manual） | **不新增独立 API**；仅作内部辅助（Conflict 记录构造/校验）；**不提供裁决入口**（D-STEP8-13 处置） | — |
| `src/compute/unit_registry.py` | 无（Registry 定义） | Unit → quantity_kind 映射 | `get_quantity_kind(unit)` | 100%（deterministic） |
| `src/compute/calculate.py` | — | — | **第一版不创建**（D-STEP8-07 裁定：统计 Compute = ∅；避免空壳模块） | N/A |
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
| 1 | S8-S-01 | synthetic_business_fixture.xlsx 存在于 `tests/eval/fixtures/<case>/inputs/` | 文件存在 | ✅ 已满足（`case_step8_defect_v1`） |
| 2 | S8-S-02 | 真实 Excel 可被 xlsx parser 读取，产出 RawSource + CellRaw list | 单元测试 | fixture 已就绪（Coding 验证） |
| 3 | S8-S-03 | RawSource provenance 完整（source_id + location）；每个 CellRaw 精确对应 Excel 位置 | 单元测试 + 手动检查 JSON | 确定性（CDM 继承） |
| 4 | S8-S-04 | Mapper 产出的 Candidate Fact：fact_id 三段式正确 + fact_type 不变式成立 + registry 验证通过 | 单元测试（需 fixture 后） | fixture 已就绪 + 映射规则冻结（Coding 验证） |
| 5 | S8-S-05 | Fact Store 状态机正确：candidate→confirmed / missing / conflict 转换；Conflict resolution_policy = manual | 单元测试（状态转换矩阵） | CDM 继承（裁决后转换验证延后——D-STEP8-13 处置） |
| 6 | S8-S-06 | 单位换算 / 量纲一致性 vs 独立 expected truth（**S8-S-06 重化**；统计计算延后——D-STEP8-07 裁定） | 集成测试 | 已重化（D-STEP8-07） |
| 7 | S8-S-07 | Criterion Evaluation 正确（qualified / unqualified / not_evaluable 判定） | 单元测试 | CDM 继承 |
| 8 | S8-S-08 | Conclusion 聚合逻辑正确 | 单元测试 | ConclusionRule 已裁定（`step8_defect_slice_v1`；Coding 验证） |
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
| R1 | ~~真实 Excel fixture 不可得~~ | **CLOSED（已缓解）** | D-STEP8-02 CLOSED：synthetic fixture 已创建并回读验证 | 人工构造 business fixture 并明确标记 synthetic（已执行） |
| R2 | ~~Mapper 无法完全确定性~~ | **CLOSED（已验证）** | 结果 A：全部确定性（D-STEP8-03 CLOSED） | 证据：Sprint Brief v2 §6 + notes.md §6；边界（合并单元格/多级表头）如实记录为后续工作 |
| R3 | **CDM Schema 边界空洞** | **HIGH**（已发生） | D-CONFLICT-001 = OPEN / Deferred / 非当前路径（原 Coding-Blocking 已由合规最小替代切片解除；混凝土场景恢复时重新生效） | 已登记 §10.1/§10.2 + 2026-10-09 决策报告；不静默修改 03；需独立设计决策 |
| R4 | **Report IR → SystemOutput 语义断层** | **LOW** | 已证伪——IR Materialization 可确定性承担 | Pre-Check 0b 确认 Adapter 可确定性投影 |
| R5 | **Scope Creep（忍不住加 Pipeline/LLM）** | **HIGH** | 违反 OOS；延迟交付 | 严格对照 OOS 15 条；每写一个模块前确认 |
| R6 | ~~Fixture 需要未注册的 fact_type~~ | **CLOSED（已排除）** | 6/6 列头 fact_type 全部已注册（零 Registry 变更） | registry.has 6/6 True；Registry 扩展仍需独立授权，**不得自行修改 registry.py** |

---

## 10.1 Design Conflict Registry

| ID | 发现 | 影响范围 | 状态 / 处理 |
|---|---|---|---|
| **D-CONFLICT-001** | `component.concrete_strength` 假设一个 Component 一个强度值；GB/T 50081-2019 要求每构件 3 个平行试块 → 需要 sample→component 归属关系 | Fixture 设计、Mapper、Compute、FactType 选择（混凝土场景） | **OPEN（Deferred / 非当前路径，未解决）**。现有 CDM 无法无歧义表达该关系（见 §10.2）。**2026-10-09 决策（结论 B）**：合规最小替代切片（缺陷判定切片）不依赖该关联，冲突不再对当前路径构成 Coding-Blocking；未宣告解决、关闭条件不变（决策报告 §5）。不得静默采用任何路径；不得修改 Registry/CDM 让问题看似消失 |

### 10.2 D-CONFLICT-001 证据化 Schema 可表达性分析

**依据**（实际读取本地文件，非引用以往总结）：

| 证据来源 | 实际内容 |
|---|---|
| `src/cdm/domain.py` | 已实现 Domain Object 仅 5 类：Project / Structure / Component / Defect / Evidence。**没有 Sample 类**；`Component` 字段为 `id / parent_structure_id / defect_ids / fact_ids`，**没有 sample_ids / 测量集合字段** |
| `src/cdm/registry.py` | `sample` 作用域只注册 `sample.id`；**未注册 `sample.concrete_strength`**；无任何 sample↔component 关联类型 |
| `src/cdm/domain.py` 模块说明 | 「Relations are explicit L3 fields (B-class), **NOT Facts**」——关系应由 L3 字段承载，而非 Fact |
| `03_CDM §17/§23/§24/§29` | 概念骨架含 MeasurementSet / Measurement，但**仅为概念**：未实现类、无注册类型、无关系字段 |
| `03_CDM §8.1` | `fact_type` 必须来自受控注册表；新增类型必须先改注册表，不允许抽取时临时发明 |

**判断**：

1. 用现有 CDM 的**普通 Fact** 表达 sample→component 归属：**不合法**。`sample.host_component_id` 之类的 Fact 既未注册，又违反「关系是 L3 字段而非 Fact」的冻结意图；且 `sample` 无对应 L3 Domain Object，无处存放该关系。
2. 仅**扩展受控 Registry**（如 +`sample.concrete_strength`）：只能表达"试块有强度值"，**不能表达"该试块属于哪个构件"**，**不足以**关闭 D-CONFLICT-001。
3. 必须**改变 Frozen CDM Schema**（新增 Sample Domain Object / 关系字段，或授权关系型 FactType）才能表达该归属；超出 Step 8 授权，需独立设计决策（`03` 变更 / D-045 级）。

**结论**（2026-10-09 复核后更新状态，实质结论不变）：
- D-CONFLICT-001 保持 **OPEN**；经设计决策裁定为 **Deferred / 非当前路径**（不再对本 Step 的当前路径构成 Coding-Blocking）；
- 缺少的能力：一个合法的「sample → component」归属表达（L3 关系字段或经授权的关联类型），以及 Compute 分组键的确定性解析依据——**该能力缺口未消除**；
- 关闭条件（不变）：由独立决策批准 `03` 层面的关联表达，或在既有模型内给出**无需该关联**的合规最小替代方案并经证据验证；
- **2026-10-09 复核**：合规最小替代方案已被找到并经字段级链路验证（缺陷判定切片，归属由 `Defect.host_component_id` 冻结字段承载）——见 `Step_8_Sample_Component_Relation_Design_Decision.md` §3/§4。该方案**不宣告原冲突解决**：原混凝土场景（sample→component 归属 + 统计聚合）仍无法在冻结 CDM 内表达，保持 Deferred。

> **禁止**：不得通过修改 `src/cdm/registry.py` 或 `03_CANONICAL_DATA_MODEL.md` 使该冲突看似消失；不得伪造 `revision` / `supersedes`；不得把 Conflict 裁决与 Fact 更正混为一谈。

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

状态标记：**READY FOR CODING**（9/9 门禁 PASS，见 §14；`D-CONFLICT-001` 保持 OPEN / Deferred / 非当前路径；`D-STEP8-13` 处置决议＝最小路径不激活——均不构成当前路径 Coding-Blocking）

---

## 14. Final Status

### 决策分布

```text
INHERITED:       2  (D-STEP8-08, D-STEP8-10)
CLOSED:         10  (D-STEP8-01, 02, 03, 04, 05, 06, 07, 09, 11, 12)
OPEN:            1  (D-STEP8-13：处置决议＝最小路径不激活，保持 OPEN、非 Coding-Blocking)
Design Conflict: 1  (D-CONFLICT-001：OPEN / Deferred / 非当前路径（未解决）；当前路径无 Coding-Blocking Conflict)
```

### READY FOR CODING 门禁检查

| # | 门禁 | 状态 | 详情 |
|---|---|---|---|
| 1 | 第一份 Fixture 的设计和来源性质已明确 | **PASS** | D-STEP8-02 CLOSED：synthetic_business_fixture `case_step8_defect_v1` 已创建（xlsx + meta.json + notes.md；回读验证 7×6；Sprint Brief v2 §8 8/8） |
| 2 | Mapper 确定性边界有证据支持 | **PASS** | D-STEP8-03 CLOSED：Q1-Q6 = 结果 A（全部确定性；Sprint Brief v2 §6 / notes.md §6） |
| 3 | 第一版 FactType 与 Registry 要求已明确 | **PASS** | D-STEP8-06 CLOSED：6/6 fact_type 已注册、零 Registry 变更（registry.has 6/6 True） |
| 4 | Sample→Component 关联已合法确定，或有合规最小替代 | **PASS（替代路径）** | 合规最小替代切片（缺陷判定切片）不依赖该关联，经字段级链路验证（决策报告 §3/§4）；D-CONFLICT-001 本身保持 OPEN / Deferred，未宣告解决 |
| 5 | Compute 输入 / 分组 / 输出 / 缺失冲突语义已明确 | **PASS** | D-STEP8-07 CLOSED：第一版统计 Compute = ∅；仅 unit_registry（Sprint Brief v2 §5）；缺失/冲突语义 = N/A |
| 6 | Criterion / ConclusionRule 与输出枚举已明确 | **PASS** | D-STEP8-09 CLOSED：criterion-step8-defect-width-01 + step8_defect_slice_v1 + direction 值域声明（Sprint Brief v2 §4） |
| 7 | SystemOutput projection 可按冻结 Schema 无歧义实现 | **PASS** | D-STEP8-11 CLOSED；Pre-Check 0b 字段级确认 |
| 8 | 不存在未解决的 Coding-Blocking Design Conflict | **PASS（当前路径）** | 当前路径（缺陷判定切片）无 Coding-Blocking Design Conflict；D-CONFLICT-001 保持 OPEN / Deferred / 非当前路径（未解决，恢复混凝土场景时重新生效） |
| 9 | 三份文档的决策矩阵 / 接口 / 流程 / 风险 / 状态一致 | **PASS** | Fixture Design Sprint 三文档同批更新后一致（Closure §2/§14 ↔ Contract §0/§19 ↔ Sprint Brief v2） |

### OPEN 决策清单（保留项，均不构成当前路径 Coding-Blocking）

| OPEN | 关键程度 | 关闭条件 |
|---|---|---|
| D-STEP8-13 Conflict 裁决 → Fact 转换 | **OPEN（非 Coding-Blocking）** | 处置决议（Sprint Brief v2 §7）：第一版不提供 `resolve_conflict`；`mark_conflict` 仅记录；不伪造 conflict→superseded 转换。关闭条件不变：冻结 CDM 明确裁决后转换，或授权新决策 |
| **D-CONFLICT-001** Component–Sample 关联 | **OPEN / Deferred / 非当前路径**（不再是当前路径的 Coding-Blocking） | 独立决策批准 `03` 层面的关联表达，或给出无需该关联的合规最小替代方案（当前路径已由替代切片解除阻塞，**未宣告解决**；混凝土场景恢复时重新生效） |

### 最终状态判定

```text
STEP 8 DESIGN DISCOVERY = COMPLETE
STEP 8 DESIGN DECISION CLOSURE = COMPLETE（D-STEP8-02/03/06/07/09 证据化 CLOSED；D-STEP8-13 处置登记；9/9 门禁 PASS）
STEP 8 DESIGN STATUS = READY FOR CODING
STEP 8 CODING CONTRACT = READY FOR CODING
STEP 8 FIXTURE DESIGN = COMPLETE（case_step8_defect_v1 三件交付物已创建并验证）
STEP 8 CODING = READY（可启动 Coding Round；尚未开始）
```

### 下一阶段

```text
Step 8 Coding Round（新 Session）
  ↓ 按 Contract v1（READY FOR CODING）实施 src/parsers | facts | compute(unit_registry) | rules | ir
  ↓ fixture = tests/eval/fixtures/case_step8_defect_v1（已就绪；映射规则 / 判定链 / 显示策略冻结于 Sprint Brief v2 与 notes.md）
  ↓ D-CONFLICT-001 保持 OPEN / Deferred（非前置；恢复混凝土场景时重新生效）
  ↓ D-STEP8-13 保持 OPEN（最小路径不激活 Conflict 分支）
  ↓ S8-S-01 ~ S8-S-12 验证 + Step 7-E/F 回归（S8-S-12）
```

> 门禁已全部满足（9/9 PASS）；三份文档一致输出 `READY FOR CODING`。状态升级以实际门禁检查为依据；不得虚假升级。

---

*本 Report 到此完成决策关闭与 Fixture Design。不创建任何生产代码。下一步：Step 8 Coding Round（Contract = READY FOR CODING）。*
