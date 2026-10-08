# Step 8 Coding Contract v1

> **状态**：**PROPOSED**（决策不全，见 §19 Decision Register — 6 个 OPEN）
> **正式升级 READY FOR CODING 的条件**：关闭 D-STEP8-02 / 03 / 06 / 07 / 09 后重新检查 READY FOR CODING 门禁
> **版本**：v1（Draft Revision 0，由 Step 8 Design Decision Closure Report 驱动）
> **日期**：2026-10-08
> **前置**：Step 7-E ACCEPTED + FROZEN（v11）；Step 7-F ACCEPTED + FROZEN（v1）；Step 8 Design Discovery COMPLETE；Step 8 Design Decision Closure INCOMPLETE（关键决策 OPEN）

---

## 0. Contract State Declaration

```text
本 Contract 当前为 PROPOSED。

PROPOSED 含义：
  1. 整体架构方向（Candidate A）已确立
  2. 核心责任边界（Parsing ≠ Semantic Interpretation）已确立
  3. 6 个决策为 CLOSED，2 个为 INHERITED，6 个为 OPEN
  4. OPEN 决策关闭后（D-STEP8-02/03/06/07/09）→ Contract 升级为 READY FOR CODING

READY FOR CODING 门禁（全部满足）：
  a. D-STEP8-02 Fixture Source 不是 OPEN
  b. D-STEP8-03 Mapper 确定性程度 不是 OPEN
  c. D-STEP8-06 First Fact Types 不是 OPEN
  d. D-STEP8-07 First Compute set 不是 OPEN
  e. D-STEP8-09 First ConclusionRule 不是 OPEN
  f. 无 Design Conflict

FROZEN 状态保留给后续正式冻结动作（需独立 Session 批准）。
```

---

## 1. Objective

**Step 8 的目标**：在**零 LLM** 条件下，从**一份真实 Excel fixture** 出发，打通 Input → CDM → Report IR → SystemOutput 的确定性管线，产出第一个能被 Step 7-E 真实消费的 SystemOutput，**达成架构铁律 M3 里程碑**。

这不是为了增加代码数量，而是为了：
- 第一次把 S0P 的"生产侧"接到已经冻结的"验证侧"
- 用一个最小、真实、可验证的 Vertical Slice 验证 02/03/04/05 的设计是否成立
- 形成第一个真正意义上的 **END-TO-END VERIFIED VERTICAL SLICE**

---

## 2. Scope

### In Scope

| # | 模块 | 路径 | 职责 | 确定性 |
|---|---|---|---|---|
| 1 | XLSX Parser | `src/parsers/xlsx.py` | Excel → RawSource + CellRaw list | 100%（纯解析） |
| 2 | RawSource Schema | `src/parsers/base.py` | CellRaw dataclass | 100% |
| 3 | Fact Store | `src/facts/store.py` | 状态机 + Conflict 管理 | 100%（CDM 继承） |
| 4 | Conflict Resolver | `src/facts/conflict.py` | manual resolution 接口 | 100% |
| 5 | Unit Registry | `src/compute/unit_registry.py` | Unit → quantity_kind 映射 | 100% |
| 6 | Compute | `src/compute/calculate.py` | 统计计算 | 100% |
| 7 | Criterion Engine | `src/rules/criterion.py` | Fact + Criterion → Evaluation | 100%（CDM 继承） |
| 8 | ConclusionRule | `src/rules/conclude.py` | Evaluations → Conclusion | 100% |
| 9 | IR Builder | `src/ir/builder.py` | FactSet + Conclusion → Report IR | 100% |
| 10 | IR Materialization + Adapter | `src/ir/adapter.py` | Report IR → SystemOutput | 100% |
| 11 | synthetic_business_fixture | `tests/eval/fixtures/<case>/inputs/` | 首份 fixture | **OPEN（D-STEP8-02）** |
| 12 | 各模块单元测试 + 集成测试 + E2E 测试 | `tests/parsers/`, `tests/facts/`, ... | 覆盖上述所有模块 | 测试确定性 |

### Out of Scope（冻结 15 条）

| 编号 | 排除项 | 理由 |
|---|---|---|
| OOS-1 | DOCX Template Parser + Renderer（M4） | Step 8 止于 IR |
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

## 3. Architecture Boundary

### Parsing ≠ Semantic Interpretation（铁律）

```
Excel → Parser → CellRaw（有位置有原始值，但无语义）
                    ↓
                  Mapper（把 CellRaw 映射到 CandidateFact）
                    ↓
              CandidateFact（有 fact_id 有 fact_type）
                    ↓
              Fact Store 状态机
```

Parser 只做确定性文件结构发现，**绝对不做**：scope_class 推断、fact_type 推断、实例 key 推断、单位判断、字段同义识别。

### 四层结构（继承 CDM v0.3.1）

```
L1 RawSource / CellRaw（Parser 输出）
    ↓
L2 Candidate → Fact（Mapper + Store）
    ↓
L3 Domain Objects（由 Fact 组织；Step 8 最小覆盖）
    ↓
L4 Report IR（IR Builder 产出，符合 04 Schema）
    ↓
SystemOutput（Adapter Materialization 后，Step 7-E 可消费）
```

### Step 8 ↔ Step 7-E/F 边界（冻结）

```
Step 8 负责：
  生成真实 SystemOutput

Step 7-E 负责：
  判定这个 SystemOutput 是否正确（7 条 P0）

Step 7-F 负责：
  Run / Regression / Baseline / Gate（消费 Step 7-E 结果）

接口：
  Step 8 → SystemOutput → Step 7-E evaluate_case() → CaseEvaluationResult
  Step 7-F runner 编排 Step 7-E，不被 Step 8 反向依赖
```

---

## 4. Input Fixture

**状态：OPEN（D-STEP8-02）**

### 当前证据

- workspace 中无任何真实 Excel/CSV 文件（Glob 确认）
- 无法使用真实 Excel

### 决策方向

需要创建 **synthetic_business_fixture**，明确标记 `synthetic_business_fixture = true`。

### Fixture 设计任务清单（OPEN）

| # | 工作 | 产出 | 状态 |
|---|---|---|---|
| 1 | 选择 fixture 覆盖的 Scope | component + concrete_strength | OPEN |
| 2 | 设计 Excel 结构 | sheet / header / rows | OPEN |
| 3 | 确定 fact_type 映射 | 每个列头 → registry type | OPEN |
| 4 | 确定单位 | MPa（压力） | OPEN |
| 5 | 确定 scope_class | component | OPEN |
| 6 | 设计实例 key 规则 | K001 / K002 / ... | OPEN |
| 7 | 补充 fixture 周边 | ground_truth / expected_issues / meta | OPEN |
| 8 | 决定 case_id | tests/eval/fixtures/<case_id>/ | OPEN |

---

## 5. RawSource Schema

### 复用已有

CDM v0.3.1 已定义 RawSource（`src/cdm/types.py`）：
```python
@dataclass
class RawSource:
    source_id: str
    source_type: str  # "xlsx"（from RAW_SOURCE_TYPES）
    location: Optional[str] = None
    original_path: Optional[str] = None
    metadata: dict = field(default_factory=dict)
```

### Step 8 新增：CellRaw

CellRaw 是 Parser 产出的最小单元格单元，**不涉及任何语义推断**：

```python
@dataclass
class CellRaw:
    """Parser 产出的原始单元格。无任何语义解释。"""
    sheet: str            # Sheet 名称
    row: int              # 行号（1-based）
    col: int              # 列号（1-based）
    raw_value: Any        # openpyxl 原始值（int/float/str/datetime/None）
    raw_text: str         # 转为字符串的值（用于显示/记录）
    excel_address: str    # Excel 引用（如 "Sheet1!F23"）
```

### ParseResult

```python
@dataclass
class ParseResult:
    raw_source: RawSource
    header_row: Optional[int]  # header 在第几行；None = 第一行为 header
    data_rows: int             # 总行数
    cells: List[CellRaw]       # 全部原始单元格
```

---

## 6. Mapping Responsibility

### 铁律

> **Parsing ≠ Semantic Interpretation**

### Parser 做什么（已冻结）

- 文件打开 + Sheet 枚举
- Cell 原始值提取
- Cell 位置坐标
- 列头行识别（纯位置，不解释列头含义）
- RawSource + CellRaw 构建

### Mapper 做什么（结构已冻结，确定性程度 OPEN）

- **输入**：ParseResult（CellRaw list + header row 识别）
- **输出**：List[CandidateFact]（CDM Fact dataclass with `status="pending"`, `provenance="program"`, `method="quoted"`）
- **职责**：把 CellRaw 映射到 CandidateFact，设置 fact_id / fact_type / value / unit / source_refs

### Mapper 确定性程度（OPEN — 需 fixture 验证 Q1-Q6）

| # | 问题 | 回答 | 影响 |
|---|---|---|---|
| Q1 | Excel 列名是否足够稳定？ | **待 fixture** | → 列名精确匹配 |
| Q2 | 是否需要 sheet/row/col context？ | **待 fixture** | → 位置规则 |
| Q3 | 是否存在同义字段？ | **待 fixture** | → 受控同义词表 |
| Q4 | 是否存在无法无歧义映射的字段？ | **待 fixture** | → 记录 Design Finding |
| Q5 | 哪些映射是确定性规则？ | **待 fixture** | → 冻结为 deterministic |
| Q6 | 哪些是不确定性问题？ | **待 fixture** | → 后续工作 |

### 三种可能结果

**结果 A**：全部确定性 → D-STEP8-03 = CLOSED（结构 + 确定性）
**结果 B**：大部分确定性 + 规则增强 → D-STEP8-03 = CLOSED（结构）+ OPEN（细节规则）
**结果 C**：存在无法确定性映射 → D-STEP8-03 = CLOSED（结构）+ OPEN（部分映射）+ 可能影响 Zero-LLM

---

## 7. Fact Store

### 复用已有

CDM v0.3.1 已定义 Fact（`src/cdm/types.py`）和 FactTypeRegistry（`src/cdm/registry.py`）。Step 8 直接 import，不修改。

### Step 8 新增：FactStore

```python
class FactStore:
    """Fact 状态机（继承 CDM §15）。"""

    def __init__(self, registry: FactTypeRegistry):
        ...

    def add_candidates(self, candidates: List[Fact]) -> None:
        """添加 candidate facts（status 必须为 pending）。"""

    def get_confirmed_facts(self) -> List[Fact]:
        """返回 status=filled + review_status=confirmed 的 facts。"""

    def get_all_facts(self) -> List[Fact]:
        """返回所有 facts（含 conflict / missing / rejected）。"""

    def get_conflicts(self) -> List[Conflict]:
        """返回所有未解决的 conflicts。"""

    def mark_missing(self, fact_id: str) -> None:
        """标记为 missing（value=None）。"""

    def mark_conflict(self, fact_id: str, candidates: List[ConflictCandidate]) -> None:
        """标记为 conflict + 记录候选值。"""

    def mark_rejected(self, fact_id: str, reason: str) -> None:
        """标记为 rejected。"""
```

### 状态转换（继承 CDM §15）

```
Candidate(pending) → confirmed(filled, review_status=confirmed)
                  → missing(status=missing, value=null)
                  → conflict(status=conflict, candidates=[...], resolution_policy=manual)
                  → rejected(status=rejected, review_status=rejected)

Conflict resolved(manual) → confirmed(选定值) → superseded(旧值)
Revision 更正 → supersedes 链（CDM §15.5）
```

### 不变量

- Conflict resolution_policy **必须**为 `"manual"`（CDM §36 冻结）
- 不允许静默选择"最可信"、最新值、平均值、或任何自动策略
- revision > 1 必须携带 supersedes
- superseded Fact 不得携带 supersedes 字段（方向约束）

---

## 8. Conflict

### 继承 CDM §36

```python
@dataclass
class Conflict:  # 已在 src/cdm/types.py 定义
    fact_id: str
    candidates: List[ConflictCandidate]
    resolution_policy: str = "manual"
    resolved_by: Optional[str] = None
    resolution: Optional[str] = None
```

### Step 8 接口

```python
def resolve_conflict(
    conflict_id: str,
    resolution_value: Any,
    resolver_id: str,
) -> Fact:
    """Manual 解决冲突。返回选定值作为新 confirmed Fact。"""
```

### 不变量

- resolution_policy 必须为 manual
- 没有人工 resolution 时，该 fact 不能进入 confirmed
- 旧 Fact 的 status 设为 superseded

---

## 9. Compute

### Unit Registry

**状态：INHERITED（CDM §10 定义 quantity_kind）**

Registry 沿用 CDM 的 QUANTITY_KINDS frozenset，Step 8 提供 Unit → quantity_kind 映射表（初始只覆盖 fixture 涉及的单位）。

### Calculate

**状态：部分 OPEN（具体统计量依赖 fixture）**

接口：
```python
def compute_statistic(
    facts: List[Fact],
    statistic_type: str,  # "avg" | "max" | "min" | "sum" | "count"
    result_fact_id: str,
    result_fact_type: str,
) -> Fact:
    """对同类型 Fact 列表计算统计值。产出 computed Fact（method="computed", provenance="program"）。"""
```

初始支持的统计量（OPEN → 待 fixture 确认）：
- `avg`（平均值）— 最常见
- `max` / `min` — 工程检测常用
- `count` — 计数（构件数、测点总数）

单位换算规则：
- 所有输入 Fact 必须有相同 quantity_kind（否则返回 missing）
- 单位换算只在此发生；**不存储换算后的值**
- 输出 Fact 使用原始单位（不强制 SI）

---

## 10. Rules

### Criterion Engine

**状态：INHERITED（CDM §25.1 定义）**

```python
def evaluate(fact: Fact, criterion: Criterion) -> Evaluation:
    """Fact + Criterion → Evaluation。"""
    # 闸门：criterion.review_status != confirmed → not_evaluable
    # operator: > >= < <= == between
    # 比较：fact.value vs criterion.value（需单位换算）
```

### Criterion 数据结构（CDM §25.1）

```python
# CDM 已定义 criterion_id / value / unit / quantity_kind / condition
# source_refs / review_status / rule

# Step 8 补充 Evaluation 数据结构
@dataclass
class Evaluation:
    evaluation_id: str
    criterion_id: str
    fact_id: str
    status: str  # qualified | unqualified | not_evaluable
    method: str  # how judgment was derived
```

### 闸门

```
criterion.review_status != confirmed → Evaluation.status = "not_evaluable"
禁止输出方向性判定
```

---

## 11. Conclusion

### ConclusionRule

**状态：OPEN（D-STEP8-09，依赖 fixture 业务逻辑）**

最小聚合规则（预候选，待 fixture 确认）：

| 逻辑 | 输出 |
|---|---|
| 全部 Evaluation qualified → Conclusion.status = "qualified" |
| 任一 Evaluation unqualified → Conclusion.status = "unqualified" |
| 任一 Evaluation not_evaluable（无 confirmed criterion）→ Conclusion.status = "insufficient_evidence" |
| 所有 Evaluation missing → Conclusion.status = "missing" |

接口：
```python
def conclude(evaluations: List[Evaluation], rule_id: str = "minimal") -> Conclusion:
    """多项 Evaluation → 单一 Conclusion。Rule 为最小聚合逻辑。"""
```

### Conclusion 数据结构（CDM §27）

```python
@dataclass
class Conclusion:
    conclusion_id: str
    status: str  # qualified | unqualified | insufficient_evidence | missing
    rule_id: str  # "minimal" 等
    input_evaluation_ids: List[str]  # 聚合了哪些判定
    description: str  # 简短文字描述（程序生成，非 LLM）
```

---

## 12. Report IR

**状态：INHERITED（04_REPORT_IR.md 全文 Schema）**

### IR Builder

```python
def build(
    fact_set: List[Fact],
    conclusion: Conclusion,
) -> dict:
    """FactSet + Conclusion → Report IR dict（符合 04 Schema）。"""
```

### Step 8 最小合法 IR 结构

```text
Document
  version = "0.3.1.1"
  sections:
    Section(id="overview", semantic="project_overview")
      Block: Heading("工程概况")
      Block: Paragraph(kind="assertion", segments=[Lit, Ref, Lit])
    Section(id="inspection_result", semantic="inspection_result")
      Block: Table(headers=[Lit, Lit], rows=[[Ref, Ref], ...])
    Section(id="conclusion", semantic="conclusion")
      Block: Paragraph(kind="assertion", segments=[Lit, Ref/Lit, Lit])
```

### Block 类型限制

Step 8 只产出：
- Heading（章节标题）
- Paragraph（kind="assertion" — 数值陈述；不产出 Narrative，OOS-14）
- Table（检测结果表）
- Signature（签字位，占位）

不产出：
- List（后续）
- Figure（后续）
- Formula（后续）
- Toc（后续）
- PageBreak（后续）
- Narrative 段（OOS-14）

### 不创建 IR Validator

Step 8 的 IR Builder 必须直接产出符合 04 Schema 的 dict。不需要额外 Validator 模块。

---

## 13. SystemOutput Adapter（IR Materialization）

**状态：CLOSED（结构），确定性已确认**

### 为什么叫 Materialization 不是 Renderer

IR Materialization 把 IR 的结构化表示（Ref / Lit / Table cell）转换成带显示值的文本表示——这是确定性操作（Ref 替换 + 数值格式化），不涉及 DOCX 排版。

| IR Materialization | DOCX Renderer（OOS） |
|---|---|
| Ref → "32.4 MPa" | "32.4 MPa" → Word 段落 + 字体 + 字号 |
| Lit + Ref + Lit → RenderedSegment.raw_text | Paragraph → Word Paragraph |
| Table cell Ref → TableCell.raw_text | Table → Word Table + 列宽 + 边框 |
| SystemOutput dataclass | DOCX 文件 |
| **Step 8 In Scope** | **OOS（M4）** |

### 字段级映射（Pre-Check 0b 结果）

| SystemOutput 字段 | Report IR 对应 | 映射方式 |
|---|---|---|
| `report_ir: dict` | Report IR Document Schema | **直接透传** — IR Builder 产出的 dict 原样填入 |
| `rendered_segments: List[RenderedSegment]` | Paragraph(kind="assertion") + Table | **Materialization** — Lit + Ref → 拼接 raw_text；同时生成 AnchorDeclaration（token + fact_id + char_start + char_end） |
| `structured_tables: List[StructuredTable]` | Table | **Materialization** — Table(headers/rows) → StructuredTable(headers/rows)；每个 cell 的 Ref → raw_text + fact_id；Table cell 的 anchors → cell.anchor_declarations |
| `system_reported_issues: List[SystemIssue]` | 无直接对应 | 诊断性输出；Step 8 可以产出 missing/conflict facts 的基本诊断 |
| `system_evaluations: List[SystemEvaluation]` | 无直接对应 | Rule 引擎同步产出；每个 Evaluation → SystemEvaluation |

### 精度规则

Step 8 无 TemplateSpec（OOS-13），使用默认精度：
- fact.value 是 float → 保留原始 Python 精度
- fact.value 是 int → 整数显示
- 单位 = fact.unit（不强制转换）

### AnchorDeclaration 生成规则

```text
Ref("fact:component.K001.concrete_strength")
  ↓ 查询 Fact
  ↓ fact.value = 32.4, fact.unit = "MPa"
  ↓ 显示 = "32.4 MPa"
  ↓ 原始文本位置 = char_start, char_end（拼接后位置计算）
  ↓ AnchorDeclaration(token="32.4 MPa", fact_id="fact:component.K001.concrete_strength",
                       char_start=X, char_end=Y)
```

---

## 14. E2E Flow

```text
synthetic_business_fixture.xlsx（待 D-STEP8-02）
 ↓ parse_xlsx()
RawSource + List[CellRaw]
 ↓ map_cells_to_candidates()（Mapper 确定性程度待 D-STEP8-03 Q1-Q6）
List[CandidateFact]（CDM Fact with status="pending", provenance="program"）
 ↓ FactStore.add_candidates()
FactStore（confirmed / missing / conflict）
 ↓ FactStore.get_confirmed_facts()
List[Fact]
 ↓ compute_statistics()（待 D-STEP8-07）
computed Facts + original Facts
 ↓ evaluate()（Criterion Engine）
List[Evaluation]
 ↓ conclude()（ConclusionRule）
Conclusion（待 D-STEP8-09）
 ↓ ir_builder.build()
Report IR dict（符合 04 Schema）
 ↓ ir_adapter.to_system_output()（IR Materialization）
SystemOutput（完全匹配 src/eval/accuracy/data_structures.py）
 ↓ Step 7-E evaluate_case(trusted, system)
CaseEvaluationResult
 ↓ Step 7-F run_evaluation()（可选）
RunSnapshot
```

---

## 15. Tests

### 单元测试

| 测试对象 | 文件 | 覆盖 |
|---|---|---|
| xlsx parser | `tests/parsers/test_xlsx.py` | 真实 fixture 读取；sheet 枚举；cell 提取；header 识别 |
| FactStore | `tests/facts/test_store.py` | 状态转换矩阵；conflict 添加；missing 标记；confirmed 查询 |
| Conflict resolver | `tests/facts/test_conflict.py` | manual resolution；candidate 选择；supersedes 链 |
| Unit Registry | `tests/compute/test_unit_registry.py` | unit → quantity_kind 映射 |
| Compute | `tests/compute/test_calculate.py` | avg / max / min 计算；单位换算；quantity_kind 一致性 |
| Criterion Engine | `tests/rules/test_criterion.py` | qualified / unqualified / not_evaluable；review_status 闸门 |
| ConclusionRule | `tests/rules/test_conclude.py` | 最小聚合逻辑；qualified / unqualified / insufficient_evidence |
| IR Builder | `tests/ir/test_builder.py` | IR Schema 校验；minimal IR 结构；determinism（重复运行一致） |
| IR Adapter | `tests/ir/test_adapter.py` | IR Materialization；AnchorDeclaration 位置计算；SystemOutput 类型匹配 |

### 集成测试

| 测试 | 覆盖 |
|---|---|
| Excel → FactSet → Evaluation | Parser + Mapper + Store + Compute + Criterion 串联 |
| FactSet → IR → SystemOutput | IR Builder + Adapter 串联 |

### E2E 测试

```text
real_excel → RawSource → FactSet → Evaluation → Conclusion → Report IR → SystemOutput → Step 7-E evaluate_case() 消费
```

**S8-S-10/11 是关键门禁**。Step 7-E 必须成功对这个 SystemOutput 跑 7 条 P0，产出 CaseEvaluationResult。

### 回归测试

```text
pytest tests/（全部，含 Step 7-D/E/F 回归）
```

S8-S-12 要求已有 412 回归全部通过。

---

## 16. Success Criteria（12 条）

| # | 编号 | 标准 | 验证方式 | 状态 |
|---|---|---|---|---|
| 1 | S8-S-01 | synthetic_business_fixture.xlsx 存在于 `tests/eval/fixtures/<case>/inputs/` | 文件存在 | OPEN |
| 2 | S8-S-02 | Excel 可被 parser 读取，产出 RawSource + CellRaw list | 单元测试 | OPEN |
| 3 | S8-S-03 | RawSource provenance 完整；每个 CellRaw 精确对应 Excel 位置 | 手动检查 JSON | 可验证 |
| 4 | S8-S-04 | Candidate Fact fact_id 三段式正确 + fact_type 不变式成立 + registry 验证通过 | 单元测试 | OPEN |
| 5 | S8-S-05 | Fact Store 状态机正确；Conflict resolution_policy = manual | 单元测试 | CDM 继承 |
| 6 | S8-S-06 | Compute 结果与独立 expected truth 一致 | 集成测试 | OPEN |
| 7 | S8-S-07 | Criterion Evaluation 正确 | 单元测试 | CDM 继承 |
| 8 | S8-S-08 | Conclusion 聚合逻辑正确 | 单元测试 | OPEN |
| 9 | S8-S-09 | IR 合法且 deterministic（重复运行一致） | Schema 校验 + 重复运行 | 100% 可验证 |
| 10 | S8-S-10 | SystemOutput 可被 Step 7-E 真实消费 | E2E 测试 | **关键门禁** |
| 11 | S8-S-11 | Step 7-E 对真实 SystemOutput 跑 7 条 P0 产出 CaseEvaluationResult | E2E 测试 | **关键门禁** |
| 12 | S8-S-12 | 已有 Step 7-D/E/F regression 全部通过 | pytest | 冻结回归 |

---

## 17. Out of Scope

见 §2 Scope Out of Scope 表格（冻结 15 条）。

---

## 18. Risks

| # | 风险 | 概率 | 影响 | 缓解 |
|---|---|---|---|---|
| R1 | **真实 Excel fixture 不可得** | MEDIUM | D-STEP8-02 = OPEN；Fixture 设计需额外 Session | synthetic_business_fixture 明确标记 |
| R2 | **Mapper 无法完全确定性** | MEDIUM | 可能影响 Zero-LLM 目标 | 如实记录 Design Finding；结果 A/B/C |
| R3 | **CDM Schema 边界空洞** | LOW | 需 D-045 决策 | 发现即登记 |
| R4 | **Scope Creep** | **HIGH** | 违反 OOS | 严格对照 OOS 15 条 |
| R5 | **Fixture 引入不在 registry 的 fact_type** | MEDIUM | 需先扩展 registry | D-STEP8-06 检查 registry 覆盖 |

---

## 19. Decision Register

### 已决策（INHERITED / CLOSED）

| 决策 | Status | 内容 | Evidence |
|---|---|---|---|
| D-STEP8-01 Vertical Slice Scope | CLOSED | M1(parsers/xlsx + facts/store) + M2(compute + rules + conclude) + M3(ir/builder + ir/adapter) 的一个最小真实 Excel 业务案例 | 02 §17.2 + Discovery §10 |
| D-STEP8-04 Fact Store 状态机 | CLOSED | candidate(pending) → confirmed/missing/conflict/rejected；revision 链 | CDM §15 + §15.5 |
| D-STEP8-05 Conflict resolution | CLOSED | 第一阶段必须 manual；Store.resolve_conflict() 接口 | CDM §36 + hard constraint |
| D-STEP8-08 First Criterion operators | INHERITED | 直接引用 CDM §25.1 定义 | CDM §25.1 |
| D-STEP8-10 Minimal Report IR output | INHERITED | 直接引用 04_REPORT_IR.md 全文 Schema | 04 |
| D-STEP8-11 SystemOutput projection | CLOSED（结构） | IR Materialization（Ref→显示值 + anchors + Table cell 渲染）；不是 DOCX Renderer | Pre-Check 0b |
| D-STEP8-12 E2E success criteria | CLOSED | 12 条 S8-S-01 ~ S8-S-12 | Discovery SC-1~SC-5 细化 |

### 未决策（OPEN）

| 决策 | Status | 需要的证据 | 影响 |
|---|---|---|---|
| D-STEP8-02 Fixture Source | **OPEN** | synthetic_business_fixture 设计完成（8 个子工作项） | 关键门禁 |
| D-STEP8-03 RawSource → Fact Responsibility | **OPEN（完整）** | fixture 完成后回答 Q1-Q6（确定性程度） | 关键门禁 |
| D-STEP8-06 First Fact Types | **OPEN** | fixture 字段 → registry type 映射 | S8-S-04 |
| D-STEP8-07 First Compute set | **OPEN** | fixture 数据类型（是否有一组同类型聚合） | S8-S-06 |
| D-STEP8-09 First ConclusionRule | **OPEN** | fixture 业务判定逻辑 | S8-S-08 |

---

## 20. Contract Deviation Rules

本 Contract 不允许静默 deviation。

任何 deviation 必须：
1. 登记为 Decision Register 新条目（D-STEP8-13+）
2. 注明 deviation 内容 + 理由 + 不这样做的代价
3. 如果涉及修改 Frozen docs → 必须登记 Design Conflict + D-045+ 决策请求
4. 如果偏离 Candidate A 方向 → 必须重新跑 Decision Closure

---

## 21. Coding Stop Conditions

### 绝对停止（立即）

| 条件 | 动作 |
|---|---|
| 发现需要修改 docs/02/03/04/05/07 才能继续 | STOP → 登记 Design Conflict → 暂停 |
| 发现需要修改 src/eval/accuracy/** 才能继续 | STOP → 登记 → 暂停 |
| 发现需要修改 src/eval/run/** 才能继续 | STOP → 登记 → 暂停 |
| 发现 CDM Fact schema 无法覆盖 Step 8 需要的模式 | STOP → 登记 → D-045 决策请求 |
| 发现 Report IR Schema 无法表达 Step 8 需要的结构 | STOP → 登记 → D-046 决策请求 |
| Mapper 遇到无法确定性映射的字段且必须继续 | STOP → 登记 Design Finding → 暂停 |

### 停止并重新决策

| 条件 | 动作 |
|---|---|
| Fixture 设计发现 registry 缺少必要 fact_type | 扩展 registry（不算修改 CDM Schema）→ 继续 |
| Scope 开始膨胀到 OOS 项 | STOP → 重新确认 OOS → 回到边界 |

### 明确禁止（任何时候）

```text
静默修改 Frozen docs
为了"推进"强行关闭 OPEN 决策
Mapper 没有 fixture 证据就假定完全确定性
IR→SystemOutput 没有字段级分析就假定可以直接衔接
静默添加 Pipeline/LLM/Template Parser/Renderer
Mapper 膨胀为 Parser + Semantic Interpreter + Rule Engine
```

---

*本 Coding Contract 为 PROPOSED 状态。关闭 OPEN 决策后重新检查 READY FOR CODING 门禁。*
