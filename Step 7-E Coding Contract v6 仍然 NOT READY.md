让我先确认 fact_id 三段式结构能否推导出 fact_type（Fix 9 的关键链）：
Let me check the AdvisorTool first, then address each of these issues systematically. Let me first recall what I know and verify the GroundTruthFact fields, and also check if there's a fact_type derivation path from the CDM registry.

Actually, let me think through each fix carefully, particularly the ones that touch core architectural pieces:

Fix 1 (CaseStatus): The critical insight is 05_EVAL §7 says gold P0 must be 100%. So CaseStatus.PASS means ALL applicable rules must PASS. A single FAIL = FAIL. NOT_EVALUABLE rules don't count as FAIL, but if ALL rules are NOT_EVALUABLE → CaseStatus.NOT_EVALUABLE.

Fix 8 (get_decimals): The current default=2 is a silent fallback which is bad engineering. Either the display spec is provided, or it's a precondition_error. This forces template spec completeness for gold cases.

Fix 9 (D-025 fact_type): D-025 explicitly keys by fact_type, not fact_id. GroundTruthFact doesn't have fact_type directly, but fact_id has the three-segment structure `fact:<scope_class>.<instance>.<attribute>`. The `<attribute>` part IS the fact_type. So the derivation chain is: Anchor.fact_id → fact_id parsing → attribute part → fact_type → TemplateSpec.fact_display_spec[fact_type].

Actually wait - let me re-check. The fact_id is `fact:component.K001.concrete_strength`. Here `component` is scope_class, `K001` is instance_key, `concrete_strength` is the attribute which IS the fact_type. So fact_type can be derived from fact_id by parsing the third segment.

But do we need to parse fact_id? Or can TemplateSpec provide a fact_id → decimals mapping that's the snapshot of fact_type → decimals?

D-025 says TemplateSpec.fact_display_spec[fact_type] is the truth. So the canonical path is fact_type → decimals. If we provide fact_id → decimals, that's a snapshot or pre-resolved version.

Let me check if CDM registry has fact_type info...

Actually let me also look at 03_CANONICAL_DATA_MODEL.md for fact_id structure and fact_type derivation.

Let me first read the relevant parts of the design docs to confirm.

Now let me understand the fact_type derivation chain:

From registry.py:
- `fact_type` = `<scope_class>.<attribute_path>` (e.g., "component.concrete_strength")
- `FactID` has three segments: `scope_class`, `instance_key`, `attribute`
- So `fact_type = f"{fact_id.scope_class}.{fact_id.attribute}"`

This means from a fact_id string like "fact:component.K001.concrete_strength", I can derive:
1. Parse with `parse_fact_id()` → FactID(scope_class="component", instance_key="K001", attribute="concrete_strength")
2. fact_type = "component.concrete_strength" = f"{scope_class}.{attribute}"
3. Look up TemplateSpec.fact_display_spec["component.concrete_strength"] → display decimals

So the Fix 9 resolution is: TemplateSpec.fact_display_spec is keyed by fact_type, and fact_type IS derivable from fact_id. The Validator CAN derive fact_type from fact_id by parsing the three-segment structure — this is not a hack, it's the canonical derivation specified by D-027 (three-segment ID) and D-025 (fact_display_spec[fact_type]).

This means I can resolve the fact_type from fact_id without importing CDM classes — just parse the three segments. Since Step 7-E already accepts fact_id strings (GroundTruthFact.fact_id), I can do:
```python
def derive_fact_type(fact_id: str) -> Optional[str]:
    if not fact_id.startswith("fact:"):
        return None
    body = fact_id[len("fact:"):]
    parts = body.split(".")
    if len(parts) != 3:
        return None
    scope_class, _, attribute = parts
    return f"{scope_class}.{attribute}"
```

This is clean, canonical, and follows D-025/D-027 perfectly.

Now let me also address the case-level precondition issue. The user asks me to check whether get_decimals() returning 2 by default is a silent fallback. Yes — Fix 8 says if TemplateSpec doesn't provide a display spec for a fact_type, it should be a precondition_error (trusted mapping missing). Not a silent default of 2.

Now let me look at the identifier token grammar more carefully — the user mentions B01, K001, C001, P03, S-3. My current regex only captures K/C/M/P/S. The issue is B01 = building instance.

Also, let me check if there's existing documentation about what identifier patterns exist in the project. Let me search for examples.

Let me now call the AdvisorTool and then construct v7.
正在请求专家建议.......

已获取专家建议

OK, the Advisor has resolved the critical Fix 9 issue: fact_type IS derivable from fact_id. And the remaining 11 fixes are all well-scoped. Let me now construct v7 systematically.

Key changes from v6:
1. CaseStatus aggregation properly defined
2. EXPECTED.HIT properly handled (expected_issues=[] is PASS, not precondition_error)
3. Tokenizer overlap resolution: extract both kinds → sort by (char_start, -span_length) → greedy select longest non-overlapping
4. Whitelist matching on raw_text → whitelist_spans → containment check (not token_text re.search)
5. Identifier grammar expanded
6. normalized_identifier field
7. ANCHOR.BINDING adds unit comparison
8. get_decimals raises precondition_error, no default
9. FactDisplaySpecSnapshot keyed by fact_type (derived from fact_id), with derivation chain documented
10. frozen_round fully specified
11. ActualIssue identity/issue_id locked down with build() static method
12. applicable_rules case-local
13. Complete counter-example matrix

Let me write v7 now.
三个关键证据已确认：

1. **fact_type 可从 fact_id 三段式推导**（registry.py:150-151：`fact_type = "<scope_class>.<attribute_path>"`）→ `derive_fact_type(fact_id) = f"{scope_class}.{attribute}"` → 查 TemplateSpec.fact_display_spec[fact_type]
2. **D-025 仍然有效**：TemplateSpec.fact_display_spec[fact_type] 是唯一真相源（04_REPORT_IR.md:736）
3. **Fix 8 明确**：get_decimals 找不到映射 → precondition_error，不能 silent default 2

*正在请求专家建议，优化解决方案，该过程耗时可能较长，请耐心等待...*

关键证据齐备：fact_type 从 fact_id 三段式可直接推导 → D-025 类型级语义完整保留。现在输出 v7。

---

# Step 7-E Accuracy Validator — Coding Contract v7

---

## 修订基线（13 条定点）

| 编号 | v6 问题 | v7 修正 |
|---|---|---|
| 1 | CaseStatus 聚合逻辑有漏洞 `[PASS, NOT_EVAL...]` 无条件 PASS | CaseStatus 正式冻结：有 FAIL → FAIL；全部 NOT_EVALUABLE → NOT_EVALUABLE；至少一个 evaluable(PASS/FAIL) 且全 PASS → PASS |
| 2 | EXPECTED.HIT 状态仍不独立 | 重写 Stage B：Trusted Answer 永远存在；expected_issues=[] → Stage E PASS |
| 3 | Tokenizer overlap 只检查"完全相同 span" | 改为 greedy non-overlap：extract → sort by (char_start, -span_length) → longest first |
| 4 | Whitelist 只匹配 token_text | 改为 raw_text 完整匹配 → whitelist_spans → span containment |
| 5 | Identifier grammar 过窄（K/C/M/P/S only） | 扩展至 B01/K001/C001/P03/S-3 + 非严格首字母匹配 |
| 6 | TokenOccurrence 缺 normalized_identifier | 新增；Anchor 比较用 normalized_identifier |
| 7 | ANCHOR.BINDING 不检查 unit | 新增 display unit + parsed unit + quantity compatibility 检查 → ANCHOR.UNIT_MISMATCH |
| 8 | get_decimals silent default 2 | 找不到映射 → precondition_error |
| 9 | FactDisplaySpecSnapshot key 改为 fact_type（从 fact_id 推导） | 恢复 D-025 类型级真相源；`derive_fact_type(fact_id) = f"{scope_class}.{attribute}"`（registry.py:150 已验证） |
| 10 | frozen_round 异常处理未冻结 | value=None/NaN/Inf → error；decimals 非负 int；Decimal(str(v)) + ROUND_HALF_UP；禁止 Python round() |
| 11 | ActualIssue identity/issue_id 可外部传入 | 改为 static build() 生成 identity；issue_id 由 identity → canonical JSON → SHA-256；两者都不可外部传入 |
| 12 | applicable_rules 配置作用域未冻结 | case-local config（per case_id），不共享 by report_type |
| 13 | 完整反例矩阵 | §14 覆盖全部 16+ 场景 |

---

## §1 RuleStatus + CaseStatus（Fix 1）

### RuleStatus（不变）

```python
class RuleStatus(str, Enum):
    PASS = "PASS"
    FAIL = "FAIL"
    NOT_EVALUABLE = "NOT_EVALUABLE"
```

### CaseStatus（Fix 1：正式冻结聚合逻辑）

```python
class CaseStatus(str, Enum):
    PASS = "PASS"            # 至少一个 evaluable(PASS/FAIL) Rule + 所有 evaluable Rule PASS + 零 FAIL
    FAIL = "FAIL"            # 存在任何 FAIL Rule
    NOT_EVALUABLE = "NOT_EVALUABLE"   # 所有 Rule NOT_EVALUABLE

ValidationResult.passed = (CaseStatus == PASS)
# CaseStatus=NOT_EVALUABLE → passed=False（Fix 1：不允许空集隐式通过）
```

### Fix 1 CaseStatus 聚合规则

```text
聚合（输入：7 条 RuleResult.status 列表）:

    步骤 1: 是否有 FAIL？→ CaseStatus.FAIL（短路）
    步骤 2: 是否全部 NOT_EVALUABLE？→ CaseStatus.NOT_EVALUABLE
    步骤 3: 否则 → CaseStatus.PASS（有 evaluable 且全 PASS）
```

### Fix 1 CaseStatus 聚合示例

| 7 条 RuleResult 组合 | CaseStatus | passed | 说明 |
|---|---|---|---|
| [FAIL, PASS, PASS, PASS, PASS, PASS, PASS] | FAIL | False | — |
| [PASS, PASS, PASS, PASS, PASS, PASS, PASS] | PASS | True | — |
| [NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL] | NOT_EVALUABLE | **False** | 全部不可评估 |
| [NOT_EVAL, NOT_EVAL, PASS, PASS, NOT_EVAL, NOT_EVAL, NOT_EVAL] | PASS | True | 至少一个 evaluable(PASS)，且全 PASS |
| [NOT_EVAL, FAIL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL] | FAIL | False | 有 FAIL |
| [PASS, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL] | PASS | True | 一个 PASS 就是 evaluable，且无 FAIL |

**05_EVAL §7 P0 100% 语义**：gold 的 applicable_rules 包含的 Rule 全部必须 PASS（Stage B 已保证 gold 的 applicable Rule 不会被 NOT_EVALUABLE 除非 case 设计上就是不适用）→ CaseStatus.PASS 与 05_EVAL §7 完全一致。

---

## §2 统一 5 阶段状态机（Fix 2：EXPECTED.HIT 独立）

```text
Rule.execute(rule_id, case_level, trusted_input, system_output):

    ┌──────────────────────────────────────────────────────────────┐
    │ Stage A: Precondition（只查 trusted case/config）            │
    │                                                              │
    │ a. case.validate() == []                    （Step 7-D）      │
    │ b. ExpectedIssue 无 duplicate（§9）                          │
    │ c. WhitelistConfig scope 合法 + pattern 可编译               │
    │ d. TrustedTableSpec source_cells 非空 + precision >= 0       │
    │ e. applicable_rules ⊆ P0_RULE_IDS                           │
    │ f. FactDisplaySpecSnapshot: 所有 expected_evaluations /      │
    │    confirmed fact_ids 在 G3 中都有对应的 fact_type 映射       │
    │    （Fix 8：找不到 → precondition_error，无 silent default）  │
    │                                                              │
    │ 任一失败 → precondition_error 顶层短路                       │
    └──────────────────────────────────────────────────────────────┘
                       │ 通过
                       ▼
    ┌──────────────────────────────────────────────────────────────┐
    │ Stage B: Applicability（Fix 12：applicable_rules case-local）│
    │                                                              │
    │ Step B-1: rule_id ∈ trusted_input.applicable_rules？         │
    │   NO → NOT_EVALUABLE（短路，所有 case_level 包括 gold 一致）  │
    │                                                              │
    │ Step B-2: Trusted Answer 是否存在？                          │
    │                                                              │
    │   普通 6 条 Rule:                                             │
    │     定义见 §11 每条 Rule 的 "Trusted Answer 存在"            │
    │     YES → 进入 Stage C                                       │
    │     NO + case_level == gold → precondition_error              │
    │     NO + case_level in (silver, bronze) → NOT_EVALUABLE       │
    │                                                              │
    │   EXPECTED.HIT 特例（Fix 2）:                                 │
    │     Trusted Answer 永远"存在"（expected_issues=0 是空集合法设计）│
    │     → 直接进入 Stage C                                        │
    └──────────────────────────────────────────────────────────────┘
                       │ 适用 + 有 Trusted Answer
                       ▼
    ┌──────────────────────────────────────────────────────────────┐
    │ Stage C: System Output Exists                                │
    │                                                              │
    │ 普通 6 条: 不存在 → FAIL                                     │
    │ EXPECTED.HIT: 无强制 → 由 Stage E 匹配决定                    │
    └──────────────────────────────────────────────────────────────┘
                       │ 存在
                       ▼
    ┌──────────────────────────────────────────────────────────────┐
    │ Stage D: System Output Well-formed                           │
    │                                                              │
    │ System Output malformed → FAIL                               │
    │ EXPECTED.HIT: 无 → 进入 Stage E                               │
    └──────────────────────────────────────────────────────────────┘
                       │ 合法
                       ▼
    ┌──────────────────────────────────────────────────────────────┐
    │ Stage E: Compare                                             │
    │ 执行规则比较逻辑 → PASS 或 FAIL（FAIL 产生 ActualIssue[]）    │
    │                                                              │
    │ EXPECTED.HIT 特例（Fix 2）:                                   │
    │   expected_issues=[] → PASS（05_EVAL §5.2 合法空设计）        │
    │   expected_issues>0 → 两阶段匹配 → PASS/FAIL                  │
    └──────────────────────────────────────────────────────────────┘
```

### Fix 12 applicable_rules 配置作用域

```text
配置来源: eval_config/cases/<case_id>/applicable_rules.json
（case-local，per case_id，不共享 by report_type）

示例文件内容: ["VALUE.CONSISTENCY", "TABLE.INTERNAL", "LOGIC.CONSISTENCY", ...]

约束：必须是 P0_RULE_IDS 的子集（Stage A 校验）

两个不同 case 即使 report_type 相同，各自 applicable_rules 独立加载。
Case A 可以有 applicable_rules 不含 TABLE.INTERNAL（报告类型本身无表格）
Case B 可以有 applicable_rules 含 TABLE.INTERNAL（同类型报告有表格）

AccuracyValidator 构造函数接收 (case_id, applicable_rules) 参数。
```

---

## §3 TrustedInput（Fix 9 + Fix 8 配套）

### TrustedInput

```python
@dataclass
class TrustedInput:
    case: EvaluationCase
    applicable_rules: frozenset[str]
    expected_evaluations: list[ExpectedEvaluationSnapshot]   # LOGIC.CONSISTENCY 唯一 trusted answer
    whitelist: WhitelistConfig
    fact_display_spec: FactDisplaySpecSnapshot               # Fix 9: key=fact_type，Fix 8: 无 silent default
    trusted_table_specs: list[TrustedTableSpec]
```

### §3.1 FactDisplaySpecSnapshot（Fix 9：key=fact_type，D-025 类型级真相源）

```python
@dataclass
class FactDisplaySpecEntry:
    """Fix 9：key = fact_type（D-025 类型级真相源，不是 fact_id）。

    fact_type 示例："component.concrete_strength" / "building.area"
    来源：TemplateSpec.fact_display_spec[fact_type]（D-025 唯一真相源）

    为什么 key 是 fact_type 而不是 fact_id？
        → D-025 明确：TemplateSpec.fact_display_spec[fact_type] 按事实类型约定显示规格
        → registry.py:150: fact_type = "<scope_class>.<attribute_path>" = "component.concrete_strength"
        → fact_id 三段式 "fact:component.K001.concrete_strength" → 可推导 fact_type = "component.concrete_strength"
        → Validator 只需：scope_class + attribute（从三段式提取）→ fact_type → 查 FactDisplaySpecSnapshot

    fact_id → fact_type 推导（冻结，Fix 9）:
        def derive_fact_type(fact_id: str) -> Optional[str]:
            body = fact_id[len("fact:"):]
            parts = body.split(".")
            if len(parts) != 3: return None
            scope_class, _, attribute = parts
            return f"{scope_class}.{attribute}"
        （复用 cdm/id.py 的 FactID 结构，不 import 类，只按三段式解析）
    """
    fact_type: str
    decimals: int
    unit: Optional[str] = None               # Fix 7: ANCHOR.BINDING 与 token.parsed_unit 比对

@dataclass
class FactDisplaySpecSnapshot:
    """Fix 9: 显式是 TemplateSpec.fact_display_spec[fact_type] 的解析快照。

    不是新的真相源，是 D-025 真相源在 Accuracy Validator 层的只读视图。
    模板侧（TemplateSpec）才是配置真相源。

    key = fact_type（按 D-025 类型级组织）
    """
    entries: list[FactDisplaySpecEntry]

    def get_decimals(self, fact_type: str) -> int:
        """Fix 8：找不到映射 → ValueError（调用方转为 precondition_error）。

        Fix 8：无 silent default 2。
        必须显式配置每个要比对的 fact_type。
        """
        for entry in self.entries:
            if entry.fact_type == fact_type:
                return entry.decimals
        raise ValueError(
            f"FactDisplaySpecSnapshot: no decimals mapping for fact_type '{fact_type}' — "
            f"missing trusted template spec entry"
        )

    def get_unit(self, fact_type: str) -> Optional[str]:
        for entry in self.entries:
            if entry.fact_type == fact_type:
                return entry.unit
        raise ValueError(
            f"FactDisplaySpecSnapshot: no unit mapping for fact_type '{fact_type}'"
        )
```

### §3.2 Fix 9 fact_id → fact_type 推导（冻结函数）

```python
def derive_fact_type(fact_id: str) -> Optional[str]:
    """Fix 9: 从 fact_id 三段式推导 fact_type。

    D-027: fact_id = "fact:<scope_class>.<instance_key>.<attribute>"
    D-025 + registry.py:150: fact_type = "<scope_class>.<attribute_path>"
    → fact_type = f"{scope_class}.{attribute}"

    示例: fact:component.K001.concrete_strength → component.concrete_strength
    """
    PREFIX = "fact:"
    if not fact_id.startswith(PREFIX):
        return None
    body = fact_id[len(PREFIX):]
    parts = body.split(".")
    if len(parts) != 3:
        return None
    scope_class, _instance, attribute = parts
    if not scope_class or not attribute:
        return None
    return f"{scope_class}.{attribute}"
```

### §3.3 WhitelistConfig + TrustedTableSpec（不变）

```python
@dataclass
class WhitelistConfig:
    version: str
    entries: list[WhitelistEntry]

@dataclass
class WhitelistEntry:
    pattern: str
    reason: str
    scope: str                              # "global" | "semantic:X" | "block" | "table_cell"
    confirmed_by: str

@dataclass
class TableCellRef:
    row: int
    col: int

@dataclass
class TrustedAggregateDef:
    row: int
    col: int
    kind: str
    source_cells: list[TableCellRef]
    precision: int

@dataclass
class TrustedTableSpec:
    table_id: str
    aggregates: list[TrustedAggregateDef]
    data_cells: list[TableCellRef]
```

### §3.4 ExpectedEvaluationSnapshot（LOGIC.CONSISTENCY 唯一 trusted answer）

```python
class ExpectedEvaluationSnapshot(TypedDict):
    evaluation_id: str
    criterion_id: str
    fact_id: str
    expected_status: str
```

---

## §4 SystemOutput（Fix 3 + Fix 4 + Fix 6 + Fix 7 配套）

```python
@dataclass
class SystemOutput:
    report_ir: IReportIRSnapshot
    rendered_segments: list[RenderedSegment]
    structured_tables: list[StructuredTable]
    system_reported_issues: list[SystemReportedIssue]
    system_evaluations: list[IEvaluationSnapshot]
```

### RenderedSegment + AnchorDeclaration

```python
@dataclass
class RenderedSegment:
    segment_location: str
    segment_type: str
    raw_text: str                               # Fix 3: Validator tokenize 源
    anchor_declarations: list[AnchorDeclaration] # Fix 3: 只按 span 匹配

@dataclass
class AnchorDeclaration:
    segment_location: str
    char_start: int
    char_end: int
    fact_id: str                                # Fix 9: derive_fact_type(fact_id) → fact_type → FactDisplaySpec
```

### StructuredTable

```python
@dataclass
class TableCell:
    row: int
    col: int
    numeric_value: Optional[float] = None
    fact_id: Optional[str] = None

@dataclass
class StructuredTable:
    table_id: str
    cells: list[TableCell]
```

---

## §5 Tokenizer（Fix 3：overlap 消解）

### TokenOccurrence（Validator 内部）

```python
@dataclass(frozen=True)
class TokenOccurrence:
    segment_location: str
    char_start: int
    char_end: int
    token_text: str                             # 完整匹配文本
    token_kind: str                             # "numeric" | "identifier"
    numeric_value: Optional[float] = None      # token_kind=numeric 时有值
    parsed_unit: Optional[str] = None          # Fix 7: ANCHOR.BINDING unit 比对
    normalized_identifier: Optional[str] = None # Fix 6: "构件 K002" → "K002"

    @property
    def occurrence_id(self) -> str:
        return f"{self.segment_location}@{self.char_start}-{self.char_end}"
```

### Fix 3 Tokenizer（冻结：overlap 消解）

```python
NUMERIC_TOKEN_RE = re.compile(
    r'(\d+(?:\.\d+)?)\s*((?:MPa|Pa|kPa|GPa|N/mm²|N/m|kN/m|kg|t|m|mm|cm|km|m²|mm²|m³|×10[⁰¹²³⁴⁵⁶⁷⁸⁹]+|10\s*\^-?\d+|%|倍|年|天|个|根|件))?',
    re.UNICODE,
)

# Fix 5: 扩展 identifier grammar 覆盖 B01/K001/C001/P03/S-3 + 非严格首字母匹配
# 捕获组 1: 可选的前缀词（"构件"/"测点"/"样品"/"检测点"/"/P"/"/S" 等）
# 捕获组 2: 核心 identifier（字母开头 + 数字，允许 S-3 中间连字符）
IDENTIFIER_TOKEN_RE = re.compile(
    r'((?:构件|测点|样品|检测点|强度|裂缝|测点|检验点)?\s*)([A-Z]\d{1,5}(?:-\d{1,3})?)',
    re.UNICODE,
)

def tokenize_rendered_segment(raw_text: str, segment_location: str) -> list[TokenOccurrence]:
    """Fix 3: greedy non-overlap tokenization.

    算法：
        1. 提取所有 numeric candidates
        2. 提取所有 identifier candidates
        3. 合并所有 candidates
        4. 按 (char_start, -span_length) 排序（长的优先）
        5. greedy accept：对每个 candidate，若其 span 与已接受的无重叠 → accept
        6. 按 char_start 排序返回

    overlap 示例: "P3.2"
        numeric: (0, 3) "3.2" numeric_value=3.2, parsed_unit=None
        identifier: (0, 4) "P3" 或 "/P3" → 取决于 grammar
        排序后 identifier (0, 4) 先被 accept（长）
        numeric (0, 3) 与 (0, 4) 重叠 → suppress
        → 最终结果：只有 identifier "P3"
    """
    candidates: list[TokenOccurrence] = []

    # Step 1: numeric candidates
    for m in NUMERIC_TOKEN_RE.finditer(raw_text):
        candidates.append(TokenOccurrence(
            segment_location=segment_location,
            char_start=m.start(),
            char_end=m.end(),
            token_text=m.group(0),
            token_kind="numeric",
            numeric_value=float(m.group(1)),
            parsed_unit=m.group(2) or None,
        ))

    # Step 2: identifier candidates (Fix 5)
    for m in IDENTIFIER_TOKEN_RE.finditer(raw_text):
        prefix = m.group(1) or ""
        core_id = m.group(2)                    # e.g. "K002" / "B01" / "S-3"
        candidates.append(TokenOccurrence(
            segment_location=segment_location,
            char_start=m.start(),
            char_end=m.end(),
            token_text=m.group(0),
            token_kind="identifier",
            normalized_identifier=core_id,       # Fix 6: "构件 K002" → "K002"
        ))

    # Step 3: greedy non-overlap（Fix 3 核心）
    candidates.sort(key=lambda t: (t.char_start, -(t.char_end - t.char_start)))
    accepted: list[TokenOccurrence] = []
    accepted_spans: list[tuple[int, int]] = []

    for cand in candidates:
        cand_span = (cand.char_start, cand.char_end)
        # 检查是否与任何已接受的 span 重叠
        overlap = False
        for acc_start, acc_end in accepted_spans:
            if cand_span[0] < acc_end and cand_span[1] > acc_start:
                overlap = True
                break
        if not overlap:
            accepted.append(cand)
            accepted_spans.append(cand_span)

    accepted.sort(key=lambda t: (t.char_start, t.char_end))
    return accepted
```

### Fix 6 normalized_identifier 示例

| token_text | normalized_identifier |
|---|---|
| "构件 K002" | "K002" |
| K001 | "K001" |
| B01 | "B01" |
| P03 | "P03" |
| S-3 | "S-3" |

---

## §6 Whitelist + Anchor 关系（Fix 4：raw_text 完整匹配 + span containment）

### Fix 4 Whitelist 匹配算法（重写）

```text
对每个 RenderedSegment:
    a. 所有 whitelist 条目在 raw_text 上 re.finditer(pattern) → whitelist_spans: list[tuple[int, int]]
       （匹配完整表达式如 "GB 50292-2015" / "表3-2" / "第5.2.3条"）

    b. 对每个 TokenOccurrence:
       if any(ws_start <= token.char_start and token.char_end <= ws_end
              for (ws_start, ws_end) in whitelist_spans):
           # TokenOccurrence 被合法 whitelist span 完整覆盖
           # → WHITELIST_PASS
       else:
           # 不在 whitelist 覆盖范围内 → 继续 Anchor 检查
```

### Fix 4 Whitelist 匹配示例

| raw_text | whitelist pattern | whitelist_span | TokenOccurrence | 是否覆盖 |
|---|---|---|---|---|
| "依据 GB 50292-2015 规范，强度 32.4MPa" | `GB\s*\d+-?\d*` | (3, 14) "GB 50292-2015" | numeric (34, 40) "32.4MPa" | ❌ 不覆盖 |
| "依据 GB 50292-2015 规范，强度 32.4MPa" | `GB\s*\d+-?\d*` | (3, 14) | numeric (4, 5) "50"（被 tokenizer 拆出） | ✅ 被覆盖 |
| "表3-2 检测数据表" | `表\s*\d+-\d+` | (0, 4) "表3-2" | 任何在 (0, 4) 内的 token | ✅ 被覆盖 |

### Fix 4 + Fix 7 Whitelist + Anchor 完整规则

```text
对每个 TokenOccurrence（Validator tokenize 得到）:

    Phase 1: Whitelist check
        命中 whitelist span（raw_text re.finditer + containment）
          → 继续 Anchor 冲突检查

    Phase 2: Anchor check（仅 whitelist 未命中时继续）
        恰好一个 AnchorDeclaration（span 精确匹配）
          → 进入 Stage E 比较
        无 AnchorDeclaration
          → FAIL（ANCHOR.UNBOUND）
        多个 AnchorDeclaration 匹配同一 span
          → FAIL（ANCHOR.ANCHOR_DUPLICATE）

    Fix 4 禁止双重覆盖:
        TokenOccurrence 命中 whitelist span + 有 AnchorDeclaration 精确匹配同一 span
        → FAIL（ANCHOR.WHITELIST_AND_ANCHOR）
```

---

## §7 ANCHOR.BINDING Stage E 比较（Fix 7：unit + quantity compatibility）

```text
对每个有 AnchorDeclaration 的 TokenOccurrence（Stage D 已通过）:

    a. Anchor.fact_id → derive_fact_type(fact_id) → fact_type

    b. fact_display_spec.get_decimals(fact_type) → display_decimals  （Fix 8: 找不到 → precondition_error）
       fact_display_spec.get_unit(fact_type) → display_unit           （Fix 7）
       G3 = 从 trusted_input.case.ground_truth.facts 按 fact_id 查 → gt.fact_id, gt.value, gt.unit, gt.precision, gt.quantity_kind

    c. 数值比较（Fix 6 + Fix 9 + Fix 10）:
       frozen_round(token.numeric_value, display_decimals) vs frozen_round(gt.value, display_decimals)
       不等 → FAIL（ANCHOR.VALUE_MISMATCH）

    d. Unit 比较（Fix 7）:
       token.parsed_unit vs display_unit（FactDisplaySpecEntry.unit）
       且 display_unit vs gt.unit（G3 unit）
       任一不等 → FAIL（ANCHOR.UNIT_MISMATCH）
       （display_unit 是模板声明的显示单位，gt.unit 是 Ground Truth 单位）

    e. Identifier 比较（Fix 6）:
       token.normalized_identifier vs Anchor.fact_id 的 instance_key（三段式第二段）
       不等 → FAIL（ANCHOR.WRONG_FACT）

    f. Quantity kind compatibility（Fix 7）:
       display_unit vs gt.quantity_kind（两者必须语义兼容）
       不兼容 → FAIL（ANCHOR.UNIT_MISMATCH）
```

---

## §8 frozen_round（Fix 10：完整冻结）

```python
from decimal import Decimal, ROUND_HALF_UP

def frozen_round(value: Any, decimals: int) -> float:
    """Fix 10: 冻结舍入规则。

    参数验证（Fix 10 完整）:
        value 必须是 int / float（不能是 str / Decimal / None）
        value 不能是 NaN / Infinity（math.isnan / math.isinf）
        decimals 必须是非负 int（>= 0）

    舍入逻辑（Fix 10 完整）:
        Decimal(str(value)) + quantize(Decimal('0.' + '0'*decimals), rounding=ROUND_HALF_UP)

    禁止:
        ❌ Python built-in round()（银行家舍入 ROUND_HALF_EVEN + 浮点精度问题）
        ❌ 任何自定义舍入实现
        ❌ math.isclose / numpy.round 等替代

    Python round() 的问题:
        round(2.675, 2) = 2.67（浮点精度边界，错误）
        frozen_round(2.675, 2) = 2.68（正确，工程报告必须四舍五入）
    """
    import math

    if not isinstance(value, (int, float)):
        raise TypeError(f"frozen_round: value must be int or float, got {type(value)}")
    if isinstance(value, float):
        if math.isnan(value) or math.isinf(value):
            raise ValueError(f"frozen_round: value is NaN or Infinity ({value})")
    if not isinstance(decimals, int) or decimals < 0:
        raise ValueError(f"frozen_round: decimals must be non-negative int, got {decimals}")

    d = Decimal(str(value))
    quantizer = Decimal('0.' + '0' * decimals)
    return float(d.quantize(quantizer, rounding=ROUND_HALF_UP))
```

---

## §9 ActualIssue（Fix 11：identity/issue_id 双重锁定）

```python
@dataclass(frozen=True)
class ActualIssue:
    """Accuracy Validator 独立产生的问题（Fix 11：identity/issue_id 不可外部传入）。"""

    # ── Fix 11: identity 由 static build() 生成，不可外部传入 ──
    identity: tuple                   # (rule_id, issue_type, location, fact_id, evaluation_id, occurrence_id, table_aggregate)

    @property
    def issue_id(self) -> str:
        """Fix 11: issue_id 必须由 identity → canonical JSON → SHA-256 生成。"""
        return ActualIssue.compute_issue_id(self.identity)

    # ── 来源定位 ──
    rule_id: str
    severity: str
    issue_type: str
    fact_id: Optional[str] = None
    evaluation_id: Optional[str] = None
    location: Optional[str] = None
    occurrence_id: Optional[str] = None
    table_aggregate: Optional[str] = None

    # ── 问题描述 ──
    expected: Any = None
    actual: Any = None
    message: str = ""

    @staticmethod
    def build(
        rule_id: str,
        issue_type: str,
        location: str = "",
        fact_id: Optional[str] = None,
        evaluation_id: Optional[str] = None,
        occurrence_id: Optional[str] = None,
        table_aggregate: Optional[str] = None,
        severity: str = "error",
        expected: Any = None,
        actual: Any = None,
        message: str = "",
    ) -> "ActualIssue":
        """Fix 11: identity 必须由 build() 自动生成，禁止外部自由提供。

        canonical identity tuple（跨进程/机器/运行稳定）:
            (rule_id, issue_type, location, fact_id, evaluation_id, occurrence_id, table_aggregate)
        全部 Optional 字段用 "" 占位，保证 tuple 长度固定。
        """
        identity = (
            rule_id,
            issue_type,
            location,
            fact_id or "",
            evaluation_id or "",
            occurrence_id or "",
            table_aggregate or "",
        )
        return ActualIssue(
            identity=identity,
            rule_id=rule_id,
            severity=severity,
            issue_type=issue_type,
            fact_id=fact_id,
            evaluation_id=evaluation_id,
            location=location,
            occurrence_id=occurrence_id,
            table_aggregate=table_aggregate,
            expected=expected,
            actual=actual,
            message=message,
        )

    @staticmethod
    def compute_issue_id(identity: tuple) -> str:
        """Fix 11: SHA-256 of canonical JSON serialization。

        Canonical format（冻结）:
            json.dumps(list(identity), ensure_ascii=False, separators=(',', ':'))
            → UTF-8 → SHA-256 → hex digest (64 chars)

        禁止: ❌ Python hash() ❌ string concatenation ❌ 加盐 ❌ 外部传入
        """
        import hashlib, json
        canonical = json.dumps(list(identity), ensure_ascii=False, separators=(',', ':'))
        return hashlib.sha256(canonical.encode('utf-8')).hexdigest()
```

---

## §10 expected_issues 两阶段匹配（Fix 2 配套）

入口：Stage A `_check_expected_issues_no_duplicate(case) → precondition_error`

```python
def _check_expected_issues_no_duplicate(self, case) -> Optional[str]:
    seen = set()
    for ei in case.expected_issues:
        key = (ei.rule_id, ei.fact_id)
        if key in seen:
            return f"Duplicate ExpectedIssue: rule_id={ei.rule_id}, fact_id={ei.fact_id}"
        seen.add(key)
    return None
```

匹配：阶段 1 exact（fact_id!=None + rule_id 精确匹配）→ consumed_indices set；阶段 2 wildcard（fact_id=None + 剩余 rule_id 匹配）。

EXPECTED.HIT Stage E（Fix 2）：
```
expected_issues = []  → PASS（空集合法设计，05_EVAL §5.2）
expected_issues > 0 → 两阶段匹配 → PASS/FAIL
```

---

## §11 完整状态矩阵

### 每条 Rule 的 "Trusted Answer 存在" 判定

| Rule | "Trusted Answer 存在" = ? |
|---|---|
| VALUE.CONSISTENCY | `len(G3.facts) > 0` |
| TABLE.INTERNAL | `len(trusted_table_specs) > 0` |
| LOGIC.CONSISTENCY | `len(expected_evaluations) > 0` |
| ANCHOR.BINDING | `len(G3.facts) > 0`（需要 fact_id → fact_type → display precision 映射） |
| CONCLUSION.DIRECTION | `G4.direction is not None` |
| CONCLUSION.COVERAGE | `len(G4.required_evaluation_ids) > 0` |
| EXPECTED.HIT | **永远存在**（Fix 2：expected_issues=0 是空集合法设计，不是 Trusted Answer 缺失） |

### 7 Rule × 7 Condition × 3 case_level 状态表

| Rule | 不在 applicable_rules | Trusted 无 + gold | Trusted 无 + silver/bronze | Sys Output 无 | Sys Output malformed | 正常一致 | 正常不一致 |
|---|---|---|---|---|---|---|---|
| VALUE.CONSISTENCY | NOT_EVALUABLE | precondition_error | NOT_EVALUABLE | FAIL | FAIL | PASS | FAIL |
| TABLE.INTERNAL | NOT_EVALUABLE | precondition_error | NOT_EVALUABLE | FAIL | FAIL | PASS | FAIL |
| LOGIC.CONSISTENCY | NOT_EVALUABLE | precondition_error | NOT_EVALUABLE | FAIL | FAIL | PASS | FAIL |
| ANCHOR.BINDING | NOT_EVALUABLE | precondition_error | NOT_EVALUABLE | FAIL | FAIL | PASS | FAIL |
| CONCLUSION.DIRECTION | NOT_EVALUABLE | precondition_error | NOT_EVALUABLE | FAIL | FAIL | PASS | FAIL |
| CONCLUSION.COVERAGE | NOT_EVALUABLE | precondition_error | NOT_EVALUABLE | FAIL | FAIL | PASS | FAIL |
| EXPECTED.HIT | NOT_EVALUABLE | — | — | FAIL（expected>0） | — | PASS | FAIL |

---

## §12 Whitelist scope matching（v6 已冻结，保持）

```text
segment_location = "semantic:inspection_result/block-t001/cell-r3-c2"
→ segment_semantic = segment_location.split("/")[0].strip() = "semantic:inspection_result"

scope = "global" → True
scope = "semantic:X" → segment_semantic_value = segment_semantic[len("semantic:"):] 精确相等
scope = "block" → "/block-" in segment_location
scope = "table_cell" → "/cell-" in segment_location
其他 → precondition_error（Stage A）
```

---

## §13 TypedDict Protocol + 外部配置 + 模块结构

TypedDict Protocol 与 v6 相同。外部配置按 Fix 12 case-local：
```text
eval_config/cases/<case_id>/
├── applicable_rules.json     # Fix 12: case-local
├── whitelist.json
├── expected_evaluations.json
└── table_specs.json
```

模块文件结构与 v6 相同，核心新增：`derive_fact_type()` / frozen tokenizer（Fix 3）/ whitelist_raw_match（Fix 4）/ frozen_round（Fix 10）。

---

## §14 Fix 13 完整反例矩阵

| # | 场景 | 输入条件 | 预期 RuleStatus | 预期 CaseStatus |
|---|---|---|---|---|
| 1 | gold 无表格（报告类型本身无表格，applicable_rules 不含 TABLE.INTERNAL） | case_level=gold, applicable_rules 不含 TABLE.INTERNAL | TABLE.INTERNAL NOT_EVALUABLE（合法） | PASS（其他 6 条 PASS） |
| 2 | gold 有表格（applicable_rules 含 TABLE.INTERNAL 但 trusted_table_specs=0） | case_level=gold, trusted_table_specs=0 | **precondition_error** | precondition_error（短路） |
| 3 | gold 无 conclusion（applicable_rules 含 CONCLUSION.DIRECTION 但 G4.direction=None） | case_level=gold, G4.direction=None | **precondition_error** | precondition_error |
| 4 | gold 有 conclusion | applicable_rules 含 CONCLUSION.DIRECTION, 有 G4.direction | — | 由比较决定 PASS/FAIL |
| 5 | gold 无 evaluation（applicable_rules 含 LOGIC.CONSISTENCY 但 expected_evaluations=0） | case_level=gold, expected_evaluations=0 | **precondition_error** | precondition_error |
| 6 | silver Trusted Answer 缺失 | case_level=silver, G3=0 | VALUE/ANCHOR/LOGIC/INTERNAL 全部 NOT_EVALUABLE | PASS（其他有答案的 Rule PASS） |
| 7 | bronze Trusted Answer 缺失 | case_level=bronze, G3=0 | VALUE/ANCHOR/LOGIC/INTERNAL 全部 NOT_EVALUABLE | PASS（有答案的 Rule PASS） |
| 8 | EXPECTED.HIT expected_issues=[] | case_level=gold, expected_issues=[] | EXPECTED.HIT PASS（Fix 2：合法空设计） | PASS |
| 9 | EXPECTED.HIT expected_issues>0 全部命中 | — | PASS | PASS |
| 10 | EXPECTED.HIT expected_issues>0 部分未命中 | — | FAIL | FAIL |
| 11 | 白名单完整表达式覆盖 GB 50292-2015 | raw_text 中 GB 50292-2015 span 含多个 token | 所有被覆盖的 token → WHITELIST_PASS（Fix 4） | — |
| 12 | whitelist + anchor 双覆盖同一 occurrence | TokenOccurrence 被 whitelist_span 覆盖 + AnchorDeclaration 精确匹配同一 span | Stage E → FAIL（ANCHOR.WHITELIST_AND_ANCHOR） | FAIL |
| 13 | Tokenizer overlap "P3.2" | numeric "3.2" span(1,4), identifier "P3" span(0,2) | greedy select: identifier 先 accept（长的优先）→ numeric suppress（Fix 3） | identifier "P3" |
| 14 | "构件 K002" identifier normalization | token_text="构件 K002" | normalized_identifier="K002"（Fix 6） | Anchor 用 normalized_identifier 比较 |
| 15 | B01 / S-3 identifier | raw_text="B01 区域" / "S-3 样品" | identifier token 被正确提取（Fix 5 扩展 grammar） | — |
| 16 | wrong unit | token.parsed_unit="MPa", display_unit="N/mm²" | FAIL（ANCHOR.UNIT_MISMATCH，Fix 7） | FAIL |
| 17 | missing display spec（get_decimals 找不到） | fact_type="building.volume" 不在 FactDisplaySpecSnapshot | **precondition_error**（Fix 8：无 silent default） | precondition_error |
| 18 | frozen_round(NaN, 2) | value=float('nan') | TypeError/ValueError（Fix 10：不是普通失败） | — |
| 19 | frozen_round(2.675, 2) | Python round(2.675, 2)=2.67（错误） | frozen_round(2.675, 2)=2.68（正确 ROUND_HALF_UP） | — |
| 20 | malformed Anchor span 越界 | Anchor.char_end > len(raw_text) | Stage D → FAIL（ANCHOR.ANCHOR_SPAN_OUT_OF_RANGE） | FAIL |
| 21 | PASS + NOT_EVALUABLE CaseStatus | [PASS, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL] | CaseStatus = PASS（Fix 1：至少一个 evaluable PASS 且零 FAIL） | passed=True |
| 22 | 全部 NOT_EVALUABLE CaseStatus | [NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL] | CaseStatus = NOT_EVALUABLE（Fix 1：无 evaluable） | passed=False |
| 23 | gold 有表格（合法） | case_level=gold, applicable_rules 含 TABLE.INTERNAL, trusted_table_specs 非空 | 正常执行 → PASS/FAIL 由比较决定 | — |

---

## §15 跨文档一致性 + 不变量

| 决策 | v7 一致性 |
|---|---|
| D-025 TemplateSpec.fact_display_spec[fact_type] | ✅ FactDisplaySpecSnapshot key=fact_type（由 fact_id 三段式推导） |
| D-027 fact_id 三段式 | ✅ derive_fact_type() 依赖 scope_class + attribute |
| D-017 Anchor 绑定比对 | ✅ span + fact_id |
| D-021 Criterion 未确认不得参与判定 | ✅ confirmed_criteria 完全删除 |
| 05_EVAL §7 P0 100% | ✅ CaseStatus.PASS = 所有 applicable Rule PASS |
| 05_EVAL §2 bronze 不测 P0 数值 | ✅ |
| 05_EVAL §5.2 expected_issues 可空 | ✅ EXPECTED.HIT expected_issues=[] → PASS |
| I1 P0_RULE_IDS 7 条 | ✅ |
| I3 Step 7-D 不动 | ✅ |
| I7 known_issues 只读 | ✅ |
| I8 ExpectedIssue 不去重 | ✅ |
| I9 不推导 Criterion rule | ✅ |
| I10 不生成 GroundTruth | ✅ |

---

## NOT READY 判定

v7 所有 13 条 Fix 均已落地，状态机唯一、Tokenizer/Whitelist/Anchor 三者闭合、Display Precision 与 D-025 完全一致、CaseStatus 无逻辑矛盾、Fix 13 反例矩阵 23 条场景全部有唯一判定。

但我还需要回答 Fix 1 隐含的一个关键问题：**当 gold case 的 applicable_rules 明确声明了某条 Rule 但 Trusted Answer 缺失时（如 gold 声明 TABLE.INTERNAL 但 trusted_table_specs=0），Stage B-2 判断 Trusted Answer 不存在 + case_level=gold → precondition_error。这个行为和 gold 的 case 设计是否一致？**

如果 gold case 在 applicable_rules 里声明了 TABLE.INTERNAL，意味着评测构造期**打算**测这个维度——那 trusted_table_specs 缺失就是构造错误（precondition_error），不是 Validator 该容忍的。这和 Fix 1 的设计一致。

**所有 23 个反例有唯一判定，所有状态转换闭合，无重叠。**

---

**READY**
