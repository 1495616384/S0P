# Step 8 Coding Contract v1

> **状态**：**READY FOR CODING**（2026-10-09 Fixture Design Sprint 后 9/9 门禁 PASS；D-STEP8-13 处置决议＝最小路径不激活、保持 OPEN 非阻塞；D-CONFLICT-001 保持 OPEN / Deferred / 非当前路径；见 §0 / §19）
> **升级依据**：D-STEP8-02 / 03 / 06 / 07 / 09 证据化 CLOSED（fixture `case_step8_defect_v1` 已创建并验证）；门禁 1/2/3/5/6 由 FAIL → PASS
> **版本**：v1（Draft Revision 1，2026-10-09 Fixture Design Sprint 更新；结构 / OOS / 责任边界不变）
> **日期**：2026-10-08（原始）/ 2026-10-09（更新）
> **前置**：Step 7-E ACCEPTED + FROZEN（v11）；Step 7-F ACCEPTED + FROZEN（v1）；Step 8 Design Discovery COMPLETE；Step 8 Design Decision Closure COMPLETE（9/9 门禁 PASS）

---

## 0. Contract State Declaration

```text
本 Contract 当前为 READY FOR CODING。

升级记录（2026-10-09 Fixture Design Sprint）：
  D-STEP8-02 / 03 / 06 / 07 / 09 证据化 CLOSED：
    - D-STEP8-02：synthetic_business_fixture 已创建（tests/eval/fixtures/case_step8_defect_v1/
      inputs/inspection.xlsx + meta.json + notes.md；openpyxl 回读验证 7×6；Sprint Brief v2 §8 8/8）
    - D-STEP8-03：Q1-Q6 回答 = 结果 A（全部确定性；Sprint Brief v2 §6 / notes.md §6）
    - D-STEP8-06：6/6 fact_type 已注册、零 Registry 变更（registry.has 6/6 True）
    - D-STEP8-07：第一版统计 Compute = ∅；仅 unit_registry（Sprint Brief v2 §5）
    - D-STEP8-09：criterion-step8-defect-width-01 + step8_defect_slice_v1（Sprint Brief v2 §4）
  D-STEP8-13 处置决议：最小路径不激活 Conflict 裁决；resolve_conflict 第一版不提供
    （保持 OPEN、非 Coding-Blocking；Sprint Brief v2 §7）
  D-CONFLICT-001 保持 OPEN / Deferred / 非当前路径（未宣告解决）

READY FOR CODING 门禁（9 条，全部满足）：
  1. 第一份 Fixture 的设计和来源性质已明确（D-STEP8-02 CLOSED）              → PASS
  2. Mapper 的确定性边界已得到证据支持（D-STEP8-03 CLOSED，结果 A）          → PASS
  3. 第一版 FactType 和 Registry 要求已明确（D-STEP8-06 CLOSED，零变更）      → PASS
  4. Sample→Component 关联合规最小替代（D-CONFLICT-001 Deferred；非阻塞）     → PASS（替代路径）
  5. Compute 的输入 / 分组 / 输出和缺失冲突语义已明确（D-STEP8-07 CLOSED）    → PASS
  6. Criterion、ConclusionRule 与输出枚举已明确（D-STEP8-09 CLOSED）          → PASS
  7. SystemOutput projection 能按冻结 Schema 无歧义实现（D-STEP8-11 CLOSED）  → PASS
  8. 不存在未解决的 Coding-Blocking Design Conflict（当前路径）               → PASS
  9. 三份文档一致性（Closure §2/§14 与本 Contract §0/§19、Sprint Brief v2）    → PASS

其余状态语义不变：结构 / OOS 15 条 / 责任边界（Parsing ≠ Semantic Interpretation）/ 冻结契约
（Step 7-E/7-F 不可修改）均保持本 Contract 原状。

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
| 3 | Fact Store | `src/facts/store.py` | 状态机 + Conflict 管理（**Proposed Interface，见 §7**） | CDM 继承（裁决转换不激活：D-STEP8-13 处置，见 §7） |
| 4 | Conflict Resolver | `src/facts/conflict.py` | Conflict **记录**辅助（**不提供裁决入口**，见 §8） | CDM 继承（resolve_conflict 第一版不提供） |
| 5 | Unit Registry | `src/compute/unit_registry.py` | Unit → quantity_kind 映射 | 100% |
| 6 | Compute | `src/compute/calculate.py` | 统计计算 | **CLOSED（D-STEP8-07）：统计 Compute = ∅——第一版不创建 calculate.py（见 §9）** |
| 7 | Criterion Engine | `src/rules/criterion.py` | Fact + Criterion → Evaluation | 100%（CDM 继承） |
| 8 | ConclusionRule | `src/rules/conclude.py` | Evaluations → Conclusion | 100% |
| 9 | IR Builder | `src/ir/builder.py` | FactSet + Conclusion → Report IR | 100% |
| 10 | IR Materialization + Adapter | `src/ir/adapter.py` | Report IR → SystemOutput | 100% |
| 11 | synthetic_business_fixture | `tests/eval/fixtures/case_step8_defect_v1/inputs/inspection.xlsx` | 首份 fixture | **CLOSED（D-STEP8-02）：已创建并验证（见 §4）** |
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

**状态：CLOSED（D-STEP8-02，2026-10-09 Fixture Design Sprint）**

> **2026-10-09 决策更新（结论 B）**：fixture 方向由「混凝土抗压强度」调整为「缺陷判定切片」（决策报告 §4.2）；原混凝土方向 Deferred（D-CONFLICT-001 保持 OPEN / Deferred / 非当前路径）。D-STEP8-02 已证据化关闭。

### 已创建 fixture（唯一定稿）

| 项 | 值 |
|---|---|
| case_id | `case_step8_defect_v1` |
| 目录 | `tests/eval/fixtures/case_step8_defect_v1/` |
| 交付物 | `inputs/inspection.xlsx` + `meta.json` + `notes.md`（仅此三项） |
| Sheet / 规模 | 「构件缺陷检查记录」/ 1 header 行 + 6 数据行 × 6 列 |
| 列 | 楼栋编号 \| 构件编号 \| 缺陷编号 \| 最大宽度(mm) \| 长度(m) \| 形态描述 |
| 实例 key | B01 / K001-K003 / D001-D006（全局唯一；无连字符——冻结 tokenizer 兼容约束） |
| 单位 | mm（宽度）/ m（长度），quantity_kind 均 = length |
| 来源性质 | **synthetic_business_fixture = true**（`meta.json` 显式标记；绝不伪装历史真实数据） |
| 覆盖 | 6 缺陷：D002 = 0.30 边界等值 qualified；D003 = 0.35 超限 unqualified；其余 qualified |

### 验收证据（本次关闭依据）

- openpyxl 回读：`sheets=['构件缺陷检查记录']`、`7 × 6`（1 header + 6 数据），逐行与 Sprint Brief v2 §2 数据表一致
- `registry.has()` 6/6 True（零 Registry 变更）
- OOS-15 边界：fixture 目录内**无** ground_truth / expected_issues / known_issues；既有 7 个 case 目录零触碰
- Sprint Brief v2 §8 关闭条件 8/8 全部执行（含 git 状态核对：`src/cdm/registry.py` 无修改）

> **权威来源**：完整规格 / fact_type 映射 / Q1-Q6 回答 / 判定链 / 显示策略见 `Step_8_Fixture_Design_Sprint.md`（v2）§2-§6 与 fixture `notes.md`；本 Contract 不再重复。**ground_truth / expected_issues 属 Evaluation Case 设计（OOS-15），不是 Step 8 fixture 交付物。**

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

### Mapper 做什么（结构已冻结 + 确定性已裁定）

- **输入**：ParseResult（CellRaw list + header row 识别）
- **输出**：List[CandidateFact]（**瞬态桥接对象**，携带 fact_id / fact_type / value / unit / source_refs / method / provenance，但**不带合法 Fact.status**；不得伪装成 CDM Fact dataclass）
- **职责**：把 CellRaw 映射到 CandidateFact，设置 fact_id / fact_type / value / unit / source_refs。入库即成为合法 Fact（`status=filled, review_status=pending`，见 types.py 默认构造）

### Mapper 确定性程度（CLOSED — 结果 A，Q1-Q6 已由 fixture 验证）

| # | 问题 | 回答 | 影响 |
|---|---|---|---|
| Q1 | Excel 列名是否足够稳定？ | **是**——6 个固定列头逐一精确匹配（Sprint Brief v2 §3） | → 列名精确匹配 |
| Q2 | 是否需要 sheet/row/col context？ | **需要行级 context**：同行 6 列联合用于 L3 归属（host_component_id）与实例分组；单 sheet 无枚举歧义；context 源自 `source_refs` 位置（结构信息，非语义推断） | → 位置规则 |
| Q3 | 是否存在同义字段？ | **无**（每列唯一含义）；受控同义词表不建立（延后到出现真实变体） | → 受控同义词表（不启用） |
| Q4 | 是否存在无法无歧义映射的字段？ | **无**——6 列全部一行一义 | → 无 Design Finding |
| Q5 | 哪些映射是确定性规则？ | 列名精确匹配 + 行级 context 导出 L3 归属与分组 + 单位取列头声明 + 实例 key＝单元格文本 | → 冻结为 deterministic |
| Q6 | 哪些是不确定性问题？ | 本 fixture 内**无**；合并单元格 / 多级表头 / 跨行拼接不在本 fixture 内（边界如实声明，属后续工作） | → 后续工作 |

### 三种可能结果（已裁定）

**已裁定：结果 A（全部确定性）** → D-STEP8-03 = CLOSED（结构 + 确定性）。
（结果 B / C 为未采用分支：本 fixture 无需受控同义词表增强，亦无不可确定性映射字段。）

---

## 7. Fact Store

### 复用已有

CDM v0.3.1 已定义 Fact（`src/cdm/types.py`）和 FactTypeRegistry（`src/cdm/registry.py`）。Step 8 直接 import，不修改。

### Step 8 新增：FactStore

```python
class FactStore:
    """[Proposed Interface — 尚未实现] Fact 状态机（严格继承 CDM §15，禁止发明非法 status）。"""

    def __init__(self, registry: FactTypeRegistry):
        ...

    def ingest_candidates(self, candidates: List[CandidateFact]) -> None:
        """瞬态 CandidateFact → 合法 Fact(status=filled, review_status=pending) 入库。
        
        CandidateFact 是 Mapper 产出的瞬态桥接对象（非 CDM Fact dataclass，不带 status）。
        入库时使用 types.py 默认构造：status=filled, review_status=pending。
        """

    def get_confirmed_facts(self) -> List[Fact]:
        """返回 status=filled + review_status=confirmed 的 facts。"""

    def get_all_facts(self) -> List[Fact]:
        """返回所有 facts（含 conflict / missing / rejected）。"""

    def get_conflicts(self) -> List[Conflict]:
        """返回所有未解决的 conflicts。"""

    def mark_missing(self, fact_id: str) -> None:
        """标记为 missing（Fact.status=missing, value=None）。"""

    def mark_conflict(self, fact_id: str, candidates: List[ConflictCandidate]) -> None:
        """标记为 conflict + 记录候选值。"""

    def mark_rejected(self, fact_id: str, reason: str) -> None:
        """标记为 Fact.status=rejected。"""

    # resolve_conflict(fact_id, resolution_value, resolver_id) —— 第一版不提供
    #   （D-STEP8-13 处置决议：最小路径不激活；裁决后 Fact 状态转换在冻结 CDM 中未定义，禁止发明；
    #    见 Sprint Brief v2 §7）
```

### 状态转换（严格继承 CDM §15，禁止非法 status）

```
CellRaw → Mapper → CandidateFact（transient，非 CDM Fact）
                         ↓
                   Fact Store ingestion
                         ↓
                Fact(status=filled, review_status=pending)   ← types.py 默认构造
                         ↓ Store 闸门
           ┌─────────────┼──────────────┬───────────────┐
           ↓             ↓              ↓               ↓
     confirmed       missing         conflict        rejected
  (review_status   (value=null)     (candidates)     (status=rejected)
   =confirmed)                     (resolution
                                   _policy=manual)

Conflict 记录（第一版激活）→ mark_conflict(fact_id, candidates)
  → 仅记录 candidates + resolution_policy="manual"；resolved_by / resolution 保持 None
Conflict 人工裁决（第一版不激活；D-STEP8-13 处置决议）→ resolve_conflict 不提供
  → 裁决后 Fact 转换（status / revision / supersedes）在冻结 CDM 中未定义；禁止发明

Revision 更正（与 Conflict 无关）→ supersedes 链（CDM §15.5）
```

**关键禁令（保留）：**
- `pending` 是 `review_status` 的合法值，**绝不是** `Fact.status` 的合法值（registry.py FACT_STATUSES）
- `resolved` **不存在**于 Fact.status 枚举，也不得新增为持久化状态
- 不得假定「Conflict 人工裁决后原候选 Facts 转 `superseded`」——`conflict`（横向来源分歧）与 `superseded`（纵向版本更迭）是两套语义（CDM §15.5："二者不得混用"）
- 不得把 Conflict 裁决与 Fact 更正混为一谈；不得伪造 revision / supersedes 关系
- CandidateFact 是 Mapper→Store 之间的瞬态桥接对象，不得伪装成带非法 status 的 CDM Fact
- **第一版不提供 Conflict 裁决入口（`resolve_conflict`）**；`mark_conflict` 仅记录（D-STEP8-13 处置决议，见 Sprint Brief v2 §7）

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

### Step 8 接口（处置决议：第一版不提供裁决入口）

```text
resolve_conflict(fact_id, resolution_value, resolver_id)：
  第一版不提供（D-STEP8-13 处置决议——最小路径不激活；Sprint Brief v2 §7）。
  冻结 CDM 未定义裁决后 Fact 的 status / revision / supersedes 转换，禁止发明。
  第一版激活的 Conflict 行为仅为「记录」：
    mark_conflict(fact_id, candidates) → candidates 记录 + resolution_policy="manual"
    （resolved_by / resolution 保持 None）
  键约定：Conflict 主键是 fact_id（types.py 无 conflict_id 字段）——未来启用时的唯一合法键。
```

### 不变量

- resolution_policy 必须为 manual
- 没有人工 resolution 时，该 fact 不能进入 confirmed
- 第一版不存在裁决入口；`mark_conflict` 仅记录，不产生任何 Fact 状态转换
- **不得**声称旧 Fact 的 status 被设为 `superseded`——该转换在 Frozen CDM 中未定义（见 §7 关键禁令）

---

## 9. Compute

### Unit Registry（第一版唯一 compute 模块）

**状态：CLOSED（D-STEP8-07 裁定）**

Registry 沿用 CDM 的 QUANTITY_KINDS frozenset；Step 8 提供 Unit → quantity_kind 映射表（初始覆盖 fixture 涉及单位：mm → length、m → length）+ 同量纲换算支撑（供 Criterion 比较与显示一致性校验）；确定性纯函数。单位换算只在此发生，**不存储换算后的值**。

### Calculate

**状态：CLOSED（D-STEP8-07 裁定：第一版统计 Compute = ∅）**

- 第一版**不创建** `src/compute/calculate.py` / `compute_statistic()`（避免空壳模块）。
- 最小判定链 = 逐缺陷比较（Criterion）+ Evaluation 聚合（ConclusionRule）——均非统计 Compute。
- 任何统计 / 聚合输出 fact_type 均未注册；引入即需 Registry 授权 → 不在第一版。
- 统计能力（avg / max / min / count）延后：出现真实统计需求时按 D-STEP8-07 重新开启；**届时分组键必须为冻结 L3 字段**（如 `Defect.host_component_id`），不得靠命名推测。
- 缺失 / 冲突语义（compute 域）= **N/A**（无统计输入聚合）；Fact 级缺失/冲突的判定处置归 Store 闸门与 Criterion 冻结语义。
- D-CONFLICT-001 保持 OPEN / Deferred / 非当前路径——本裁定与其无耦合（当前路径不依赖该关联）。

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

**状态：CLOSED（D-STEP8-09 裁定，2026-10-09）**

第一版规则 `step8_defect_slice_v1`（确定性最小聚合）：

1. **逐构件聚合**（分组键 = `host_component_id`，冻结 L3 字段）：任一 Evaluation unqualified → 构件 unqualified；否则任一 not_evaluable → insufficient_evidence；否则全部 qualified → qualified。
2. **全局方向**：任一构件 unqualified → unqualified；否则任一 not_evaluable → insufficient_evidence；否则全部 qualified → qualified；输入为空 → missing。
3. 记录 `rule_id` + 全部 `input_evaluation_ids`；`covers[]` = 全部 6 条 evaluation ids（04 §42.1 / 05 §6.6）。
4. 优先级：unqualified 优先于 not_evaluable（单项否决的保守语义）。

预期（fixture `case_step8_defect_v1`）：K001 qualified；K002 unqualified；K003 qualified；全局 `direction = "unqualified"`。

Criterion：`criterion-step8-defect-width-01`（`defect.width <= 0.30` mm → qualified；> 0.30 → unqualified；review_status=confirmed；**0.30 为本切片合成测试参数，非规范限值**）。

接口：
```python
def conclude(evaluations: List[Evaluation], rule_id: str = "step8_defect_slice_v1") -> Conclusion:
    """多项 Evaluation → 单一 Conclusion（确定性最小聚合；分组键 = host_component_id）。"""
```

**direction 值域（本切片声明）**：`qualified / unqualified / insufficient_evidence / missing`——Step 7-D 不限定全局结论枚举（`src/eval/types.py` 仅查非空）；P0-5 以 trusted direction 比对 `report_ir.conclusion.direction`。

### Conclusion 数据结构（CDM §27）

```python
@dataclass
class Conclusion:
    conclusion_id: str
    status: str  # qualified | unqualified | insufficient_evidence | missing
    rule_id: str  # "step8_defect_slice_v1"
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

### 结论覆盖度（P0-6 入口）

结论段必须携带 `covers[]`（04_REPORT_IR.md §42.1），列出本结论覆盖的 evaluation / fact 标识，供 Step 7-E 的 P0-6 CONCLUSION.COVERAGE 判定。`covers[]` 取值已随 D-STEP8-09 裁定：= 全部 6 条 evaluation ids（本切片）。

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

### 精度规则（fixture-specific display policy — 仅本 Vertical Slice）

Step 8 无 TemplateSpec（OOS-13），使用 **fixture-specific display policy**——临时、确定性、**仅属于本 Vertical Slice**：
- 为本次 fixture 涉及的每个 fact_type 显式指定小数位数（`defect.width` → 2 位小数 + `" mm"`；`defect.length` → 2 位小数 + `" m"`），并显式指定单位后缀显示
- 数值格式化**不得**以 Python float 的内部表示或自然精度为最终格式规则（不得直接 `str(value)`）
- 单位显示 = fact.unit（不强制 SI 转换）

**关键声明（本次 Session 修正）：**
- 此规则 **不宣称是最终 TemplateSpec 规范**；仅为本次 Fixture 的临时显示策略
- 正式显示规格的唯一真相源是 04_REPORT_IR.md §17 定义的 `TemplateSpec.fact_display_spec`（OOS-13，M4 负责）
- 渲染出的显示值必须与 Step 7-E P0-1 所用的可信 `FactDisplaySpecSnapshot.decimals` 一致（INV-19），保证按显示精度值相等可精确比对
- Report IR Materialization 与 DOCX Rendering 必须分离（Materialization in scope / Rendering OOS）
- 数字 AnchorDeclaration 必须与实际显示文本、对应 Fact 及 Frozen P0 验证规则一致
- 不得因 Fixture 专用格式规则而修改 Frozen Report IR 或 Step 7-E

### AnchorDeclaration 生成规则

```text
Ref("fact:defect.D003.width")
  ↓ 查询 Fact
  ↓ fact.value = 0.35, fact.unit = "mm"
  ↓ 显示 = "0.35 mm"（2 位小数 + 单位后缀）
  ↓ 原始文本位置 = char_start, char_end（拼接后位置计算）
  ↓ AnchorDeclaration(token="0.35 mm", fact_id="fact:defect.D003.width",
                       char_start=X, char_end=Y)
```

---

## 14. E2E Flow

```text
case_step8_defect_v1/inputs/inspection.xlsx（已创建；D-STEP8-02 CLOSED）
 ↓ parse_xlsx()
RawSource + List[CellRaw]
 ↓ map_cells_to_candidates()（Mapper 确定性程度＝结果 A；D-STEP8-03 CLOSED）
List[CandidateFact]（**瞬态桥接对象**，携带 fact_id/fact_type/value/unit/source_refs，但不带合法 Fact.status）
 ↓ FactStore ingestion
FactStore（Fact(status=filled, review_status=pending) → 闸门 → confirmed / missing / conflict（记录））
 ↓ FactStore.get_confirmed_facts()
List[Fact]
 ↓ unit_registry 换算（D-STEP8-07 裁定：统计 Compute = ∅——第一版无 compute_statistic）
Facts（原值 + 单位换算支撑）
 ↓ evaluate()（Criterion Engine）
List[Evaluation]
 ↓ conclude()（ConclusionRule）
Conclusion（step8_defect_slice_v1；D-STEP8-09 CLOSED）
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
| Conflict resolver | `tests/facts/test_conflict.py` | Conflict 记录（candidates + policy="manual"）；**resolve_conflict 不提供（D-STEP8-13 处置决议）** |
| Unit Registry | `tests/compute/test_unit_registry.py` | unit → quantity_kind 映射 |
| Compute | —（第一版不创建 calculate.py；D-STEP8-07 裁定：统计 ∅） | 单位换算 / quantity_kind 一致性由 unit_registry 测试覆盖 |
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
| 1 | S8-S-01 | synthetic_business_fixture.xlsx 存在于 `tests/eval/fixtures/<case>/inputs/` | 文件存在 | ✅ 已满足（`case_step8_defect_v1`） |
| 2 | S8-S-02 | Excel 可被 parser 读取，产出 RawSource + CellRaw list | 单元测试 | fixture 已就绪（Coding 验证） |
| 3 | S8-S-03 | RawSource provenance 完整；每个 CellRaw 精确对应 Excel 位置 | 手动检查 JSON | 可验证 |
| 4 | S8-S-04 | Candidate Fact fact_id 三段式正确 + fact_type 不变式成立 + registry 验证通过 | 单元测试 | fixture + 映射规则已冻结（Coding 验证） |
| 5 | S8-S-05 | Fact Store 状态机正确；Conflict resolution_policy = manual | 单元测试 | CDM 继承（裁决转换不激活：D-STEP8-13 处置） |
| 6 | S8-S-06 | 单位换算 / 量纲一致性 vs 独立 expected truth（**S8-S-06 重化**；统计计算延后） | 集成测试 | **已重化（D-STEP8-07 裁定：统计 ∅）** |
| 7 | S8-S-07 | Criterion Evaluation 正确 | 单元测试 | CDM 继承 |
| 8 | S8-S-08 | Conclusion 聚合逻辑正确 | 单元测试 | ConclusionRule 已裁定（`step8_defect_slice_v1`；Coding 验证） |
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
| R1 | ~~真实 Excel fixture 不可得~~ | **CLOSED（已缓解）** | D-STEP8-02 CLOSED：synthetic fixture 已创建并验证 | synthetic_business_fixture 明确标记（已执行） |
| R2 | ~~Mapper 无法完全确定性~~ | **CLOSED（已验证）** | 结果 A：全部确定性（D-STEP8-03 CLOSED） | 证据：Sprint Brief v2 §6 + notes.md §6 |
| R3 | **CDM Schema 边界空洞（已实际发生）** | **HIGH** | D-CONFLICT-001 = OPEN / Deferred / 非当前路径（原 Coding-Blocking 已由合规最小替代切片解除；混凝土场景恢复时重新生效） | 已登记 Design Conflict；不静默改 Schema，不放行 Coding |
| R4 | **Scope Creep** | **HIGH** | 违反 OOS | 严格对照 OOS 15 条 |
| R5 | ~~Fixture 引入不在 registry 的 fact_type~~ | **CLOSED（已排除）** | 6/6 列头 fact_type 已注册（零 Registry 变更） | registry.has 6/6 True；Registry 扩展仍不得自行修改 registry.py |
| R6 | **人工裁决后 Fact 状态转换无冻结依据** | **CLOSED（已裁决：第一版不激活）** | 处置决议：resolve_conflict 不提供（D-STEP8-13 处置） | 禁止伪造 revision/supersedes；裁决转换验证延后 |

---

## 19. Decision Register

### 已决策（INHERITED / CLOSED）

| 决策 | Status | 内容 | Evidence |
|---|---|---|---|
| D-STEP8-01 Vertical Slice Scope | CLOSED | M1(parsers/xlsx + facts/store) + M2(compute + rules + conclude) + M3(ir/builder + ir/adapter) 的一个最小真实 Excel 业务案例 | 02 §17.2 + Discovery §10 |
| D-STEP8-02 Fixture Source | CLOSED | synthetic_business_fixture 已创建：`tests/eval/fixtures/case_step8_defect_v1/`（`inputs/inspection.xlsx` 7×6 + `meta.json` + `notes.md`） | Sprint Brief v2 §2/§8（8/8 条件已执行）；openpyxl 回读验证 7×6 |
| D-STEP8-03 RawSource → Fact Responsibility | CLOSED（结构 + 确定性） | 三层分离结构（Parsing ≠ Semantic Interpretation）+ Mapper 确定性 = **结果 A（全部确定性）**：Q1-Q6 正式回答（列名精确匹配 / 行级 context 导出 L3 归属 / 无同义字段 / 无不可映射字段 / 确定性规则集 / 本 fixture 内无不确定性） | Sprint Brief v2 §6；fixture `notes.md` §6 |
| D-STEP8-04 Fact Store 状态机 | CLOSED | Fact.status 继承 CDM §15（filled/missing/conflict/rejected/superseded）；review_status（pending/confirmed/rejected）独立；CandidateFact 为瞬态桥接对象。**Conflict 裁决后的 Fact 状态转换不属于本决策**（见 D-STEP8-13） | CDM §15 + §15.5 |
| D-STEP8-05 Conflict resolution | CLOSED（政策）/ 裁决入口第一版不提供（D-STEP8-13 处置） | 第一阶段必须 manual（resolution_policy="manual"）；`resolve_conflict()` 第一版**不提供**；`mark_conflict` 仅记录 candidates（不构成裁决后转换） | CDM §36 + hard constraint + Sprint Brief v2 §7 |
| D-STEP8-06 First Fact Types | CLOSED | 第一版采用集 = 6 个 fact_type：`building.id` / `component.id` / `defect.id` / `defect.width` / `defect.length` / `defect.pattern`——**全部已注册，零 Registry 变更** | Sprint Brief v2 §3；`registry.has()` 6/6 True；git 无 registry.py 变更 |
| D-STEP8-07 First Compute set | CLOSED | 第一版统计 Compute = **∅（明确排除）**；唯一 compute 模块 = `unit_registry.py`（mm→length、m→length）；`calculate.py` 不创建；S8-S-06 重化为单位换算 / 量纲一致性 | Sprint Brief v2 §5；registry.py 实际注册集 |
| D-STEP8-08 First Criterion operators | INHERITED | 直接引用 CDM §25.1 定义 | CDM §25.1 |
| D-STEP8-09 First ConclusionRule | CLOSED | Criterion = `criterion-step8-defect-width-01`（`defect.width <= 0.30` mm → qualified；> 0.30 → unqualified；0.30 为合成测试参数）；ConclusionRule = `step8_defect_slice_v1`（逐构件按 `host_component_id` 聚合 → 全局；unqualified 优先；covers[] = 全部 6 条 evaluation ids）；direction 值域声明 | Sprint Brief v2 §4；fixture `notes.md` §4 |
| D-STEP8-10 Minimal Report IR output | INHERITED | 直接引用 04_REPORT_IR.md 全文 Schema | 04 |
| D-STEP8-11 SystemOutput projection | CLOSED（结构） | IR Materialization（Ref→显示值 + anchors + Table cell 渲染）；不是 DOCX Renderer | Pre-Check 0b |
| D-STEP8-12 E2E success criteria | CLOSED | 12 条 S8-S-01 ~ S8-S-12 | Discovery SC-1~SC-5 细化 |

### 未决策（OPEN）

| 决策 | Status | 内容 / 处置 | 影响 |
|---|---|---|---|
| D-STEP8-13 Conflict 裁决后 Fact 状态转换 | **OPEN（处置决议：最小路径不激活；非 Coding-Blocking）** | Frozen CDM 未定义 conflict→最终 Fact 的 status/revision/supersedes 转换。处置：第一版 FactStore 不提供 `resolve_conflict`；`mark_conflict` 仅记录 candidates；不假定、不实现、不伪造 conflict → superseded / revision（CDM §15.5）；裁决转换验证延后 | 不构成当前路径 Coding-Blocking（门禁 8 组成部分） |
| D-CONFLICT-001 Component-Sample Relation | **OPEN / Deferred / 非当前路径**（未解决） | 冻结 CDM 无 Component↔Sample 关联的合法表达（混凝土场景）；当前路径已由合规最小替代切片（缺陷判定切片）解除阻塞（决策报告 §3/§4/§5） | 混凝土场景恢复时重新生效；不阻断当前路径 |

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
| Fixture 设计发现 registry 缺少必要 fact_type | 先判定：仅需在受控 Registry 补充允许类型（需设计授权，D-STEP8-06）**还是**须改 Frozen CDM Schema（登记 D-045）→ 授权前**不得自行修改 registry.py** |
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

*本 Coding Contract 为 READY FOR CODING 状态（2026-10-09 Fixture Design Sprint；9/9 门禁 PASS，见 §0）。D-STEP8-13 保持 OPEN（最小路径不激活、非 Coding-Blocking）；D-CONFLICT-001 保持 OPEN / Deferred / 非当前路径。*

```text
STEP 8 DESIGN STATUS   = READY FOR CODING
STEP 8 CODING CONTRACT = READY FOR CODING
STEP 8 CODING          = READY（可启动；尚未开始）
```

状态升级以实际门禁检查为依据；不得虚假升级状态。
