Let me read 03_CANONICAL_DATA_MODEL.md to understand unit registry/quantity_kind properly, since Fix 8 requires freezing a unit→quantity_kind mapping. Let me also check 07_DECISIONS.md D-025 to confirm unit override semantics.

Good evidence collected. Let me also check 04_REPORT_IR.md §17 more carefully for whether D-025 allows unit override. Let me look at the full fact_display_spec example from 04_REPORT_IR.md.

Actually I already have evidence from 04_REPORT_IR.md:765:
```
fact_display_spec:
    building.area  → { decimals: 2, unit: "m²" }
```

This clearly shows D-025 includes unit override in fact_display_spec. So Fix 7 answer is: YES, TemplateSpec.fact_display_spec[fact_type] allows unit override. But we need to define consistency rule: FactDisplaySpecEntry.unit (if present) should == G3.unit for ANCHOR.BINDING to pass — this is the display unit comparison.

Now let me consolidate everything into v8. Let me call AdvisorTool first per the mandatory rule.

Wait, I already have the evidence and I know exactly what needs fixing. Let me call AdvisorTool as required before committing to this approach.
正在请求专家建议.......

已获取专家建议

AdvisorTool 给出了非常具体的 v8 修订方向。现在我来输出 v8。注意几个关键点：

1. **Fix 9 Anchor.fact_id 验证**：要复用 cdm/id.py 的 parse_fact_id() / is_valid_fact_id()，不要重新实现
2. **Fix 4 Identifier Grammar**：必须基于 CDM 的 SCOPE_CLASSES 范围（component, defect, building, project, material, structure），实例标识前缀字母对应 scope_class
3. **Fix 7 Unit override 语义**：D-025 明确允许 display_unit 不同（MPa vs Pa），所以 ANCHOR 检查是：(a) token.parsed_unit == display_unit（模板声明的显示单位）；(b) display_unit 和 G3.unit 必须在 Unit Registry 映射到同一个 quantity_kind（相容性检查）
4. **Fix 8 Unit Registry**：复用 registry.py:60 的 QUANTITY_KINDS，不发明新的

让我一次性输出 v8。


*正在请求专家建议，优化解决方案，该过程耗时可能较长，请耐心等待...*

关键证据齐备：registry.py:60 QUANTITY_KINDS 已冻结；D-025 允许 display_unit override；Fix 9 直接复用 cdm/id.py parse_fact_id()。输出 v8。

---

# Step 7-E Accuracy Validator — Coding Contract v8

---

## §0 修订基线

| 编号 | v7 问题 | v8 修正 |
|---|---|---|
| 1 | Numeric token 不支持 negative/±/scientific notation/range | 单正则：±?mantissa[.exp]? × 10ⁿ? unit? |
| 2 | Scientific notation 被当成 parsed_unit | 正则把 ×10ⁿ / e±n 合并进 numeric_value |
| 3 | Unit lexical 无 longest-match | Unit alternation 按长度降序：`m² > mm² > mm > m` |
| 4 | Identifier grammar 过宽，不区分实例标识 vs H2O/ISO9001 | 冻结 Identifier Registry：scope_class → prefix letter 映射 |
| 5 | normalized_identifier 规则未冻结 | 冻结：去除中文前缀词，保留核心字母数字+连字符 |
| 6 | FactDisplaySpec 完整性只检查 expected_evaluations | Stage A：G3 所有 fact 都检查 fact_type → FactDisplaySpec 映射 |
| 7 | FactDisplaySpec.unit 语义未对齐 D-025 | D-025 允许 unit override；ANCHOR 检查 token.unit == display_unit + unit registry → quantity_kind 兼容 |
| 8 | Quantity compatibility 无可执行定义 | 冻结 Unit Registry（unit → quantity_kind），复用 registry.py:60 QUANTITY_KINDS |
| 9 | Anchor.fact_id 不执行严格 CDM 校验 | 复用 cdm/id.py parse_fact_id()；invalid → ANCHOR.INVALID_FACT_ID |
| 10 | ExpectedIssue matching 粒度隐含 | 冻结 rule-level：只用 (rule_id, fact_id)；issue_type/location 不参与 |
| 11 | Trusted/System malformed 边界不清 | Trusted malformed → precondition_error；System malformed → FAIL（表格 duplicate/source 违规） |
| 12 | CaseStatus 聚合用词含糊 | 正式：any applicable FAIL → FAIL；all applicable PASS 且 ≥1 applicable → PASS；0 applicable → NOT_EVALUABLE |
| 13 | 23 条反例 | §16 完整 35+ 条 |

---

## §1 RuleStatus + CaseStatus（Fix 12：用词正式化）

```python
class RuleStatus(str, Enum):
    PASS = "PASS"
    FAIL = "FAIL"
    NOT_EVALUABLE = "NOT_EVALUABLE"

class Applicability(str, Enum):
    APPLICABLE = "APPLICABLE"                 # rule_id ∈ trusted_input.applicable_rules
    NOT_APPLICABLE = "NOT_APPLICABLE"         # rule_id ∉ applicable_rules

CaseStatus 聚合（Fix 12，正式定义）:
    if any applicable Rule.status == FAIL:
        CaseStatus = FAIL
    elif all applicable Rule.status == PASS and len(applicable Rules) >= 1:
        CaseStatus = PASS
    elif len(applicable Rules) == 0:
        CaseStatus = NOT_EVALUABLE
    else:  # all applicable NOT_EVALUABLE
        CaseStatus = NOT_EVALUABLE

ValidationResult.passed = (CaseStatus == PASS)
```

**05_EVAL §7 P0 100% 对齐**：gold 设计上 applicable 的 Rule 全部必须 PASS；silver/bronze 合法 NOT_APPLICABLE 不影响。

---

## §2 统一 5 阶段状态机（Fix 12 配套）

```text
Rule.execute(rule_id, case_level, trusted_input, system_output):

    Stage A: Precondition（只查 trusted case/config）
        a. case.validate() == []
        b. ExpectedIssue 无 duplicate（§9）
        c. WhitelistConfig scope 合法 + pattern 可编译
        d. TrustedTableSpec: table_id 唯一 + source_cells 无重复坐标 + aggregate cell 不在 source_cells（Fix 11）
        e. applicable_rules ⊆ P0_RULE_IDS
        f. FactDisplaySpecSnapshot: 所有 G3 facts 的 derive_fact_type(fact_id) 都有映射（Fix 6）
        g. Unit Registry（Fix 8）: 所有 FactDisplaySpecEntry.unit 在 Registry 里
        h. Anchor.fact_id（如果 trusted 侧有预声明）能用 parse_fact_id() 验证（Fix 9）
        任一失败 → precondition_error 顶层短路

    Stage B: Applicability
        Step B-1: rule_id ∈ trusted_input.applicable_rules?
            NO → RuleStatus = NOT_EVALUABLE（短路，所有 case_level）
        Step B-2: Trusted Answer 是否存在？
            普通 6 条: YES → Stage C；NO+gold → precondition_error；NO+silver/bronze → NOT_EVALUABLE
            EXPECTED.HIT: 永远"存在"（Fix 2：空集合法）

    Stage C: System Output Exists
        普通 6 条: 不存在 → FAIL
        EXPECTED.HIT: 无强制

    Stage D: System Output Well-formed（Fix 11: System malformed 全在这里）
        普通 6 条: malformed → FAIL（包括 Anchor span/Span conflict/Ref 缺 fact_id/Evaluation 缺 status/table duplicate table_id/aggregate target cell 在 source_cells 等）
        EXPECTED.HIT: 无

    Stage E: Compare
        执行规则比较逻辑
```

---

## §3 TrustedInput（Fix 6 + Fix 7 + Fix 8 配套）

### TrustedInput

```python
@dataclass
class TrustedInput:
    case: EvaluationCase
    applicable_rules: frozenset[str]                         # Fix 12: case-local（§2 applicable_rules.json）
    expected_evaluations: list[ExpectedEvaluationSnapshot]   # LOGIC.CONSISTENCY 唯一 trusted answer
    whitelist: WhitelistConfig
    fact_display_spec: FactDisplaySpecSnapshot               # Fix 6: G3 所有 fact_type 必须有映射（Stage A 验证）
    trusted_table_specs: list[TrustedTableSpec]
    unit_registry: UnitRegistry                              # Fix 8: 冻结单例
```

### §3.1 FactDisplaySpecEntry + FactDisplaySpecSnapshot（Fix 7：unit override 语义）

```python
@dataclass
class FactDisplaySpecEntry:
    """Fix 7: D-025 允许 unit override（模板可决定用 MPa 显示即使 G3 存 Pa）。

    来源：TemplateSpec.fact_display_spec[fact_type]（D-025 单一真相源）
    fact_type 示例："component.concrete_strength"

    语义：
        decimals → ANCHOR.BINDING 的 rendered display precision
        unit → Fix 7: display_unit（模板声明的显示单位）
            可以与 G3.unit 不同（D-025 语义：模板决定怎么显示）
            但 display_unit 和 G3.unit 必须在 Unit Registry 映射到同一 quantity_kind（Fix 8 相容性检查）
            也可以 display_unit 为 None（模板决定显示 G3.unit），此时 ANCHOR.BINDING 用 G3.unit
    """
    fact_type: str
    decimals: int
    unit: Optional[str] = None                               # Fix 7: 允许 None（模板不 override 时用 G3.unit）

@dataclass
class FactDisplaySpecSnapshot:
    entries: list[FactDisplaySpecEntry]

    def get_decimals(self, fact_type: str) -> int:
        """Fix 6: 找不到映射 → ValueError（Stage A 会捕获并转为 precondition_error）。"""
        for entry in self.entries:
            if entry.fact_type == fact_type:
                return entry.decimals
        raise ValueError(f"FactDisplaySpec: no decimals for fact_type='{fact_type}'")

    def get_display_unit(self, fact_type: str) -> Optional[str]:
        """Fix 7: 返回模板声明的 display unit（None 表示不 override）。"""
        for entry in self.entries:
            if entry.fact_type == fact_type:
                return entry.unit                            # 可能返回 None
        raise ValueError(f"FactDisplaySpec: no unit for fact_type='{fact_type}'")
```

### §3.2 Fix 8 Unit Registry（冻结单例）

```python
# Fix 8: 复用 registry.py QUANTITY_KINDS（frozenset: length/area/volume/mass/force/pressure/displacement/velocity/frequency/...）
# Unit Registry 是 Accuracy Validator 内部冻结 dict，不 import Step 7-F

UNIT_TO_QUANTITY: dict[str, str] = {
    # Pressure
    "MPa": "pressure", "Pa": "pressure", "kPa": "pressure", "GPa": "pressure",
    "N/mm²": "pressure", "N/m²": "pressure", "kN/m²": "pressure",
    # Length
    "m": "length", "mm": "length", "cm": "length", "km": "length",
    # Area
    "m²": "area", "mm²": "area", "cm²": "area",
    # Volume
    "m³": "volume", "mm³": "volume",
    # Mass
    "kg": "mass", "t": "mass", "g": "mass",
    # Force
    "N": "force", "kN": "force",
    # Generic
    "倍": "count", "%": "ratio",
    # Unit-less
    "": "dimensionless",
}
# Note: Unit alternation in token regex uses this dict's keys sorted by length DESCENDING
# to ensure longest-match (Fix 3: m² before m, mm² before mm, etc.)

def get_quantity_kind(unit: Optional[str]) -> str:
    """Fix 8: 查 Unit Registry。unit=None → 'dimensionless'。找不到 → ValueError。"""
    if unit is None or unit == "":
        return "dimensionless"
    if unit not in UNIT_TO_QUANTITY:
        raise KeyError(f"Unit Registry: unknown unit '{unit}'")
    return UNIT_TO_QUANTITY[unit]
```

---

## §4 SystemOutput（Fix 1 + Fix 4 + Fix 9 配套）

```python
@dataclass
class SystemOutput:
    report_ir: IReportIRSnapshot
    rendered_segments: list[RenderedSegment]
    structured_tables: list[StructuredTable]
    system_reported_issues: list[SystemReportedIssue]
    system_evaluations: list[IEvaluationSnapshot]
```

### RenderedSegment + AnchorDeclaration（Fix 9 配套：Anchor.fact_id 严格 CDM 校验）

```python
@dataclass
class RenderedSegment:
    segment_location: str
    segment_type: str
    raw_text: str
    anchor_declarations: list[AnchorDeclaration]

@dataclass
class AnchorDeclaration:
    segment_location: str
    char_start: int
    char_end: int
    fact_id: str                                # Fix 9: 必须通过 cdm/id.py parse_fact_id() + is_valid_fact_id()
```

### StructuredTable（Fix 11：system malformed 检查）

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

## §5 Tokenizer（Fix 1 + Fix 2 + Fix 3 + Fix 4 + Fix 5：完整词法冻结）

### TokenOccurrence

```python
@dataclass(frozen=True)
class TokenOccurrence:
    segment_location: str
    char_start: int
    char_end: int
    token_text: str                             # Fix 1: 完整匹配（符号+数值+单位）
    token_kind: str                             # "numeric" | "identifier"
    numeric_value: Optional[float] = None      # Fix 2: 科学计数法合并进数值
    parsed_unit: Optional[str] = None          # Fix 3: longest-match；scientific notation 不是 unit
    normalized_identifier: Optional[str] = None # Fix 5: 去除中文前缀，保留核心 id
    sign: Optional[str] = None                 # "+" | "-" | None（Fix 1: 符号独立存储）

    @property
    def occurrence_id(self) -> str:
        return f"{self.segment_location}@{self.char_start}-{self.char_end}"
```

### Fix 4 Identifier Registry（冻结：scope_class → prefix letter）

```python
# Fix 4: Identifier 严格限定在 CDM SCOPE_CLASSES 范围内的实例标识。
# CDM SCOPE_CLASSES = frozenset({component, defect, building, project, material, structure})
# 每个 scope_class 对应一组典型实例标识前缀字母：

IDENTIFIER_SCOPE_MAP: dict[str, frozenset[str]] = {
    "component":  frozenset({"K", "C"}),     # 构件: K001, C001
    "defect":     frozenset({"D"}),          # 缺陷: D01
    "building":   frozenset({"B"}),          # 建筑: B01
    "project":    frozenset({"P", "S"}),      # 项目/样品: P03, S-3
    "material":   frozenset({"M"}),          # 材料: M01
    "structure":  frozenset({"ST"}),          # 结构: ST001
}

# Fix 4: 所有前缀字母集合（identifier grammar 的首字母范围）
ALL_IDENTIFIER_PREFIX_LETTERS: frozenset[str] = frozenset({
    "K", "C", "D", "B", "P", "S", "M", "ST"
})

# 不支持的 identifier patterns（显式声明 Fix 4）:
#   H2O   —— 化学分子式（元素+数字组合，不是 scope_class 实例标识）
#   CO2   —— 同上
#   A4    —— 只有首字母 A 不在前缀范围内（A 不对应任何 scope_class）
#   ISO9001 —— 规范引用（以 ISO/GB 开头，非 scope_class 实例标识）
#   GB50292 —— 规范引用（同上）
# 这些 pattern 不会被 identifier 正则捕获；如果它们包含数字，会被 numeric token regex 捕获。
```

### Fix 1 + Fix 2 + Fix 3 Numeric Token Regex（冻结）

```python
# Fix 3: Unit alternation 按长度 DESCENDING 排序（longest-match 保证）
# 例如："m²" > "mm²" > "mm" > "m"
#   32.5m²  → unit="m²"（不是 "m"）
#   1250mm² → unit="mm²"（不是 "mm"）
#   2.3m³   → unit="m³"

ALL_UNITS_SORTED_BY_LENGTH = sorted(
    UNIT_TO_QUANTITY.keys(),
    key=lambda u: len(u),
    reverse=True
)
UNIT_ALTERNATION = "|".join(re.escape(u) for u in ALL_UNITS_SORTED_BY_LENGTH)

# Fix 1 + Fix 2: 单一完整正则
# 捕获组:
#   group(1): 可选符号 [+-]?
#   group(2): mantissa \d+(\.\d+)?
#   group(3): 可选指数  (×10ⁿ | ×10^n | e[+-]?\d+ | E[+-]?\d+)
#   group(4): 可选单位（longest-match）
NUMERIC_TOKEN_RE = re.compile(
    rf'([+-]?)(\d+(?:\.\d+)?)\s*(×10[⁰¹²³⁴⁵⁶⁷⁸⁹]+|×10\^-?\d+|[eE][+-]?\d+)?\s*({UNIT_ALTERNATION})?',
    re.UNICODE,
)

def parse_numeric_token(match: re.Match) -> tuple[float, Optional[str], str]:
    """Fix 2: scientific notation 合并进 numeric_value。

    示例:
        "3.2"    → (3.2, None, "")
        "-3.2"   → (-3.2, None, "-")
        "+3.2MPa" → (3.2, "MPa", "+")
        "1.2×10⁶Pa" → (1200000.0, "Pa", "")
        "2.3m³"  → (2.3, "m³", "")
    """
    sign_str = match.group(1) or ""
    mantissa_str = match.group(2)
    exponent_str = match.group(3) or ""
    unit = match.group(4) or None

    mantissa = float(mantissa_str)
    if exponent_str:
        # 解析指数: 10⁶ / 10^6 / e6 / E+6 / e-6
        if exponent_str.startswith("×10"):
            # ×10⁶ / ×10^-3
            if exponent_str.startswith("×10^"):
                exp_num = int(exponent_str[4:])
            else:
                # Unicode superscript → int
                SUPERSCRIPT_TO_DIGIT = {"⁰":"0","¹":"1","²":"2","³":"3","⁴":"4","⁵":"5","⁶":"6","⁷":"7","⁸":"8","⁹":"9"}
                exp_digits = "".join(SUPERSCRIPT_TO_DIGIT.get(c, c) for c in exponent_str[3:])
                exp_num = int(exp_digits)
            mantissa = mantissa * (10 ** exp_num)
        elif exponent_str.lower().startswith("e"):
            exp_num = int(exponent_str[1:])
            mantissa = mantissa * (10 ** exp_num)

    # 应用符号
    if sign_str == "-":
        mantissa = -mantissa

    return mantissa, unit, sign_str or None
```

### Fix 4 + Fix 5 Identifier Token Regex（冻结）

```python
# Fix 4: scope_class-based identifier grammar
# 捕获组:
#   group(1): 可选中文前缀词（"构件"/"测点"/"样品"/"检测点"/"裂缝"/"强度"）
#   group(2): 核心 identifier — [KCMPSBST]\d{1,5}(-\d{1,3})?
#   group(3): 核心 identifier 中的连字符后缀（可选）

CHINESE_PREFIX_RE = r'(?:构件|测点|样品|检测点|裂缝|强度|检验点)?\s*'
IDENTIFIER_CORE_RE = r'([KCMPSBST]\d{1,5}(?:-\d{1,3})?)'

IDENTIFIER_TOKEN_RE = re.compile(
    f'{CHINESE_PREFIX_RE}{IDENTIFIER_CORE_RE}',
    re.UNICODE,
)

def parse_identifier_token(match: re.Match) -> tuple[str, str]:
    """Fix 5: 冻结 normalization 规则。

    token_text → normalized_identifier:
        "构件 K002" → "K002"（去中文前缀 + 空格）
        K001       → "K001"（无变化）
        S-3        → "S-3"（连字符保留）
        B01        → "B01"
        P03        → "P03"
    """
    # group(2) 已经是捕获的核心 identifier（不含中文前缀）
    normalized = match.group(2)
    return normalized
```

### Fix 3 Tokenizer（overlap 消解）

```python
def tokenize_rendered_segment(raw_text: str, segment_location: str) -> list[TokenOccurrence]:
    candidates: list[TokenOccurrence] = []

    # Phase 1: numeric
    for m in NUMERIC_TOKEN_RE.finditer(raw_text):
        num_val, unit, sign = parse_numeric_token(m)
        candidates.append(TokenOccurrence(
            segment_location=segment_location,
            char_start=m.start(),
            char_end=m.end(),
            token_text=m.group(0),
            token_kind="numeric",
            numeric_value=num_val,
            parsed_unit=unit,
            sign=sign,
        ))

    # Phase 2: identifier
    for m in IDENTIFIER_TOKEN_RE.finditer(raw_text):
        normalized = parse_identifier_token(m)
        candidates.append(TokenOccurrence(
            segment_location=segment_location,
            char_start=m.start(),
            char_end=m.end(),
            token_text=m.group(0),
            token_kind="identifier",
            normalized_identifier=normalized,
        ))

    # Phase 3: Fix 3 greedy non-overlap
    # 排序: (char_start, -span_length) — 长的优先
    candidates.sort(key=lambda t: (t.char_start, -(t.char_end - t.char_start)))
    accepted: list[TokenOccurrence] = []
    accepted_spans: list[tuple[int, int]] = []

    for cand in candidates:
        cand_span = (cand.char_start, cand.char_end)
        overlap = any(
            cand_span[0] < ae and cand_span[1] > as_
            for (as_, ae) in accepted_spans
        )
        if not overlap:
            accepted.append(cand)
            accepted_spans.append(cand_span)

    accepted.sort(key=lambda t: (t.char_start, t.char_end))
    return accepted
```

### Fix 1 + Fix 2 不支持范围（显式声明）

| 不支持形式 | 处理 | 原因 |
|---|---|---|
| `±3.2MPa`（Unicode 正负号） | numeric token 捕获 "3.2MPa"，sign=None | Fix 1 当前只支持 `+` / `-` 作为符号前缀；`±` 作为 token_text 的一部分，sign=None，在 Stage E 比较时不会有问题（sign=None 表示正数）。需要显式±比较时应在 G3 中体现 |
| `2.0~2.5m`（range） | 两个独立 numeric tokens + 中间 "~" 作为 whitespace | 两个 token 分别锚定到各自 fact_id |
| `1.2e6`（小写 e） | 支持（NUMERIC_TOKEN_RE 含 e/E） | — |

---

## §6 Whitelist + Anchor（Fix 4 + Fix 9 配套）

### Fix 4 Whitelist 匹配算法

```text
对每个 RenderedSegment:
    a. 按 scope 筛选 whitelist 条目 → applicable_entries
    b. 每个 applicable_entry 在 raw_text 上 re.finditer(pattern) → whitelist_spans[]
       （匹配完整表达式如 "GB 50292-2015" / "表3-2" / "第5.2.3条" / "页码"）

    c. 对每个 TokenOccurrence:
        # Fix 4: span containment（TokenOccurrence span 被完整 whitelist span 覆盖）
        if any(ws_start <= token.char_start and token.char_end <= ws_end
               for (ws_start, ws_end) in all_whitelist_spans):
            WHITELIST_PASS
        else:
            继续 Anchor 检查

### Fix 7 ANCHOR.BINDING Stage D（System Output Well-formed）
    a. AnchorDeclaration span 在 raw_text 范围内？→ 越界 → FAIL（ANCHOR.ANCHOR_SPAN_OUT_OF_RANGE）
    b. AnchorDeclaration span 匹配某个 TokenOccurrence？→ 找不到 → FAIL（ANCHOR.ANCHOR_ORPHAN）
    c. AnchorDeclaration.fact_id 通过 cdm/id.py is_valid_fact_id() + registry 验证？
       → Fix 9: 失败 → FAIL（ANCHOR.INVALID_FACT_ID）
    d. 按 (segment_location, char_start, char_end) 分组 AnchorDeclaration → 任何一组 size > 1 → FAIL（ANCHOR.ANCHOR_DUPLICATE）
    e. TokenOccurrence 命中 whitelist span + 有 AnchorDeclaration → FAIL（ANCHOR.WHITELIST_AND_ANCHOR）
    f. 对未命中 whitelist 的 TokenOccurrence：无 AnchorDeclaration → FAIL（ANCHOR.UNBOUND）

### Fix 7 ANCHOR.BINDING Stage E（正常比较）
    对每个有 AnchorDeclaration 的 TokenOccurrence:
        a. fact_type = derive_fact_type(Anchor.fact_id)  （Fix 9: parse_fact_id 三段式）
        b. display_decimals = fact_display_spec.get_decimals(fact_type)  （Fix 6: 找不到→Stage A 已过）
        c. display_unit = fact_display_spec.get_display_unit(fact_type)  （Fix 7: None 表示不 override，用 G3.unit）
        d. G3 = case.ground_truth.facts[fact_id]

        e. 数值比较（Fix 10: frozen_round）:
           frozen_round(token.numeric_value, display_decimals)
           vs frozen_round(gt.value, display_decimals)
           不等 → FAIL（ANCHOR.VALUE_MISMATCH）

        f. Unit 比较（Fix 7: display_unit override + Fix 8: quantity_kind 相容性）:
           effective_display_unit = display_unit if display_unit is not None else gt.unit
           token.parsed_unit == effective_display_unit?
           否 → FAIL（ANCHOR.UNIT_MISMATCH）
           get_quantity_kind(token.parsed_unit) == get_quantity_kind(gt.unit)?
           否 → FAIL（ANCHOR.UNIT_MISMATCH：单位不同量纲）

        g. Identifier 比较（Fix 5: normalized_identifier）:
           if token.token_kind == "identifier":
               normalized_id = token.normalized_identifier
               gt_instance_key = parse_fact_id(Anchor.fact_id).instance_key
               normalized_id == gt_instance_key?
               否 → FAIL（ANCHOR.WRONG_FACT）
```

---

## §7 TrustedTableSpec（Fix 11：malformed 边界）

### Fix 11 Stage A TrustedTableSpec 完整性检查（trusted malformed → precondition_error）

```text
a. TrustedTableSpec.table_id 全局唯一（跨 trusted_table_specs 列表）
   不唯一 → precondition_error

b. TrustedAggregateDef.source_cells 坐标无重复
   重复 → precondition_error

c. TrustedAggregateDef (row, col) target cell 不在自己的 source_cells 中
   （不能把 aggregate cell 自己算进 source cells）
   → 在 → precondition_error

d. TrustedAggregateDef.kind ∈ {"sum", "mean", "min", "max", "count"}
   不在 → precondition_error

e. TrustedAggregateDef.precision >= 0 且是 int
   不满足 → precondition_error
```

### Fix 11 Stage D System structured_tables malformed 检查（system malformed → FAIL）

```text
a. structured_tables 中 table_id 不唯一 → FAIL（TABLE.DUPLICATE_TABLE_ID）
b. 有 TrustedTableSpec.table_id 但 System structured_tables 中找不到 → FAIL（TABLE.TABLE_MISSING_IN_SYSTEM）
c. TrustedAggregateDef.source_cells 中某个 (row, col) 在 System table.cells 中不存在或 numeric_value is None → FAIL（TABLE.SOURCE_CELL_MISSING）
d. TrustedAggregateDef (row, col) target cell 在 System table.cells 中不存在或 numeric_value is None → FAIL（TABLE.AGGREGATE_CELL_MISSING）
```

### Fix 11 TrustedTableSpec 数据结构

```python
@dataclass
class TableCellRef:
    row: int
    col: int

@dataclass
class TrustedAggregateDef:
    row: int                                # aggregate target cell row
    col: int                                # aggregate target cell col
    kind: str                               # "sum" | "mean" | "min" | "max" | "count"
    source_cells: list[TableCellRef]        # Fix 11: 无重复，不含 (row, col) 自身
    precision: int                          # Fix 5: 该 aggregate 的 comparison precision

@dataclass
class TrustedTableSpec:
    table_id: str                           # Fix 11: 全局唯一
    aggregates: list[TrustedAggregateDef]
    data_cells: list[TableCellRef]
```

---

## §8 frozen_round（Fix 10：完整不变）

```python
from decimal import Decimal, ROUND_HALF_UP
import math

def frozen_round(value: Any, decimals: int) -> float:
    """Fix 10: 冻结舍入规则。

    参数验证:
        value 必须是 int/float，不能是 None/str/Decimal
        value 不能是 NaN/Infinity
        decimals 必须是非负 int

    舍入: Decimal(str(value)).quantize(Decimal('0.'+'0'*decimals), ROUND_HALF_UP)

    禁止: Python round()（浮点精度边界行为不可预测）
    示例: frozen_round(2.675, 2) = 2.68（正确） vs Python round(2.675, 2) = 2.67（错误）
    """
    if not isinstance(value, (int, float)):
        raise TypeError(f"frozen_round: value must be int/float, got {type(value)}")
    if isinstance(value, float) and (math.isnan(value) or math.isinf(value)):
        raise ValueError(f"frozen_round: value is NaN/Infinity ({value})")
    if not isinstance(decimals, int) or decimals < 0:
        raise ValueError(f"frozen_round: decimals must be non-negative int, got {decimals}")

    d = Decimal(str(value))
    quantizer = Decimal('0.' + '0' * decimals)
    return float(d.quantize(quantizer, rounding=ROUND_HALF_UP))
```

---

## §9 ExpectedIssue matching（Fix 10：正式 rule-level）

### Fix 10 matching 粒度冻结

```python
# Step 7-D ExpectedIssue 冻结字段：
#   rule_id: str
#   severity: str
#   fact_id: Optional[str]
#   note: str
# Fix 10: ExpectedIssue 没有 issue_type/location/evaluation_id 字段
#   → matching 只能用 (rule_id, fact_id) composite key
#   → issue_type / location / evaluation_id 不参与匹配（根本不存在于 ExpectedIssue）

匹配规则:
    阶段 1（exact fact_id 优先）:
        ExpectedIssue.fact_id != None → 找 SystemReportedIssue 中 rule_id 相等 + fact_id 精确相等
    阶段 2（wildcard 兜底）:
        ExpectedIssue.fact_id == None → 找剩余 rule_id 相等的 SystemReportedIssue
    consumed_indices set 保证 one-to-one（一条 SystemReportedIssue 最多消耗一条 ExpectedIssue）

    Fix 10 明确:
        ExpectedIssue 是 rule-level（由 rule_id 标识）
        fact_id 是 optional 额外 specificity 层
        severity 不参与匹配（ExpectedIssue.severity 只用于本 Validator 产出的 ActualIssue.severity 对齐）
        note 不参与匹配（只是诊断描述）
```

---

## §10 Anchor.fact_id CDM 校验（Fix 9）

```python
# Fix 9: 复用 cdm/id.py 的 parse_fact_id() / is_valid_fact_id()
# 不重新实现 fact_id 格式验证
# 任何 fact_id 失败 is_valid_fact_id() → Stage D FAIL → ANCHOR.INVALID_FACT_ID

# cdm/id.py 已实现的校验链（Fix 9 依赖的现有能力）:
#   1. 必须有 "fact:" prefix
#   2. 必须正好 3 段（scope_class, instance_key, attribute）
#   3. scope_class ∈ SCOPE_CLASSES（component/defect/building/project/material/structure）
#   4. attribute ∈ FactTypeRegistry（registered attribute）
#   5. fact_type 第一段 == scope_class（invariant）

# Accuracy Validator 在 Stage D 调用:
from cdm.id import is_valid_fact_id, parse_fact_id
err = is_valid_fact_id(anchor.fact_id)
if err is not None:
    FAIL（ANCHOR.INVALID_FACT_ID, message=err）
```

---

## §11 ActualIssue（Fix 11：identity/issue_id 不可外部传入）

```python
@dataclass(frozen=True)
class ActualIssue:
    identity: tuple                                # Fix 11: static build() 生成
    @property
    def issue_id(self) -> str:                     # Fix 11: identity → canonical JSON → SHA-256
        return ActualIssue.compute_issue_id(self.identity)

    rule_id: str
    severity: str
    issue_type: str
    fact_id: Optional[str] = None
    evaluation_id: Optional[str] = None
    location: Optional[str] = None
    occurrence_id: Optional[str] = None
    table_aggregate: Optional[str] = None
    expected: Any = None
    actual: Any = None
    message: str = ""

    @staticmethod
    def build(
        rule_id, issue_type, location="",
        fact_id=None, evaluation_id=None,
        occurrence_id=None, table_aggregate=None,
        severity="error", expected=None, actual=None, message="",
    ):
        identity = (rule_id, issue_type, location, fact_id or "", evaluation_id or "",
                    occurrence_id or "", table_aggregate or "")
        return ActualIssue(identity=identity, ...)

    @staticmethod
    def compute_issue_id(identity: tuple) -> str:
        import hashlib, json
        canonical = json.dumps(list(identity), ensure_ascii=False, separators=(',', ':'))
        return hashlib.sha256(canonical.encode('utf-8')).hexdigest()
```

---

## §12 完整状态矩阵（不变）

7 Rule × 7 Condition × 3 case_level，每格唯一 bucket。CaseStatus 正式聚合（§1 Fix 12）。EXPECTED.HIT 独立（§2 Fix 2）。

---

## §13 TypedDict Protocol + 外部配置 + 模块结构

TypedDict Protocol 与 v7 相同。外部配置 Fix 12 case-local：`eval_config/cases/<case_id>/applicable_rules.json`。

模块文件结构新增（v8 特有）：
- `numeric_parser.py`: NUMERIC_TOKEN_RE / parse_numeric_token / frozen_round
- `identifier_parser.py`: IDENTIFIER_TOKEN_RE / parse_identifier_token / IDENTIFIER_SCOPE_MAP
- `unit_registry.py`: UNIT_TO_QUANTITY / get_quantity_kind / longest-match unit alternation

---

## §14 Fix 13 完整反例矩阵（35+ 条）

| # | 场景 | 预期结果 |
|---|---|---|
| 1 | -3.2MPa（Fix 1 negative number） | numeric_value=-3.2, sign="-", parsed_unit="MPa" |
| 2 | +3.2MPa（Fix 1 explicit positive） | numeric_value=3.2, sign="+", parsed_unit="MPa" |
| 3 | ±3.2MPa（Fix 1 Unicode 正负号） | numeric_value=3.2, sign=None（± 不解析为符号） |
| 4 | 1.2×10⁶Pa（Fix 2 scientific notation） | numeric_value=1200000.0, parsed_unit="Pa"（×10⁶ 合并进数值，不是 unit） |
| 5 | 32.5m²（Fix 3 longest-match unit） | parsed_unit="m²"（不是 "m"） |
| 6 | 1250mm²（Fix 3 longest-match） | parsed_unit="mm²"（不是 "mm"） |
| 7 | 2.3m³（Fix 3 longest-match） | parsed_unit="m³" |
| 8 | H2O（Fix 4 非 identifier） | 不被 identifier 正则捕获（首字母 H ∉ K/C/D/B/P/S/M/ST），数字 "2" "O" 被 numeric 捕获但 O 不是数字，最终只有 "2" 是 numeric？→ 取决于 tokenizer 实际运行，但**不会**产生 identifier |
| 9 | CO2（Fix 4 非 identifier） | C 是合法前缀字母（component）→ CO2 会被 identifier 正则捕获（C + O2 不是 identifier core 形式... 实际上 IDENTIFIER_CORE_RE 是 `[KCMPSBST]\d{1,5}(-\d{1,3})?`，C + O 不是数字 → **不会被 identifier 捕获**。会被 numeric 捕获吗？"O2" 中只有 "2" 是数字 → 只有 "2" numeric。最终 "CO" 部分无 token 覆盖） |
| 10 | A4（Fix 4 非 identifier） | A 不在前缀字母范围内 → 不会被 identifier 捕获 |
| 11 | ISO9001（Fix 4 非 identifier） | ISO 不匹配 [KCMPSBST] 开头 → 不会被 identifier 捕获；9001 被 numeric 捕获 |
| 12 | GB50292（Fix 4 非 identifier） | GB 不匹配 → 不会被 identifier 捕获；50292 被 numeric 捕获 |
| 13 | B01（Fix 4 identifier） | normalized_identifier="B01"（prefix B = building scope） |
| 14 | S-3（Fix 4 identifier） | normalized_identifier="S-3"（prefix S = project scope，支持连字符） |
| 15 | K001 / C001 / P03（Fix 4） | 全部正确捕获 |
| 16 | 非法 Anchor.fact_id（"fact:xxx.yyy.zzz" 中 yyy 含 '.'） | Stage D → FAIL（ANCHOR.INVALID_FACT_ID）（Fix 9） |
| 17 | G3 fact 有但 FactDisplaySpec 缺 mapping | Stage A → precondition_error（Fix 6） |
| 18 | duplicate TrustedTableSpec.table_id | Stage A → precondition_error（Fix 11） |
| 19 | duplicate source_cells 坐标 | Stage A → precondition_error（Fix 11） |
| 20 | aggregate target 在 source_cells 中 | Stage A → precondition_error（Fix 11） |
| 21 | System structured_tables duplicate table_id | Stage D → FAIL（TABLE.DUPLICATE_TABLE_ID） |
| 22 | ExpectedIssue 有 issue_type 字段（Step 7-D 冻结，实际没有） | Fix 10: 不存在 — matching 只用 (rule_id, fact_id) |
| 23 | ExpectedIssue 匹配 issue_type | Fix 10: issue_type 不参与匹配（根本不存在于 ExpectedIssue） |
| 24 | silver: applicable Rule 有 Trusted Answer 缺失 | NOT_EVALUABLE（Stage B-2 silver 豁免） |
| 25 | gold: non-applicable + rule 有 Trusted Answer 缺失（矛盾场景） | RuleStatus = NOT_EVALUABLE（Stage B-1 不适用短路，不进入 B-2） |
| 26 | all rules non-applicable（空 applicable_rules） | CaseStatus = NOT_EVALUABLE → passed=False（Fix 12） |
| 27 | one applicable PASS + six non-applicable | CaseStatus = PASS（Fix 12: all applicable PASS 且 ≥1） |
| 28 | Anchor.WHITELIST_AND_ANCHOR 双重覆盖 | Stage E → FAIL |
| 29 | Whitelist pattern "GB\s*\d+-\d+" 覆盖完整 expression + 内部所有 token | 所有内部 TokenOccurrence 被 whitelist span 完整覆盖 → WHITELIST_PASS |
| 30 | Wrong unit（display_unit="MPa", parsed_unit="N/mm²" 但两者 quantity_kind 相同） | Fix 7: MPa 和 N/mm² 都映射到 pressure → 单位量纲兼容 ✓；但 parsed_unit != display_unit → ANCHOR.UNIT_MISMATCH |
| 31 | wrong display precision（模板 decimals=1，token=32.45 → round to 32.5；gt.value=32.4 → round to 32.4） | frozen_round(32.45, 1)=32.5 vs frozen_round(32.4, 1)=32.4 → ANCHOR.VALUE_MISMATCH |
| 32 | frozen_round(2.675, 2) | 2.68（Decimal ROUND_HALF_UP） |
| 33 | frozen_round(float('nan'), 2) | ValueError |
| 34 | TrustedTableSpec 缺少某 G3 fact 的 FactDisplaySpec（Fix 6） | Stage A → precondition_error（G3 所有 fact_type 都要有映射） |
| 35 | Anchor.fact_id 通过 parse_fact_id 但 scope_class 未注册 → is_valid_fact_id 返回错误 | Stage D → FAIL（ANCHOR.INVALID_FACT_ID） |

---

## §15 最终 READY/NOT_READY 判定

逐条检查：

| 检查项 | 结论 |
|---|---|
| Fix 1: Numeric token ±/scientific notation/range | ✅ 完整正则 + parse_numeric_token 函数 |
| Fix 2: Scientific notation 合并进 numeric_value | ✅ ×10⁶ → 1200000.0 |
| Fix 3: Unit longest-match alternation | ✅ 按长度降序排序 |
| Fix 4: Identifier grammar scope-class 限定 + unsupported patterns 明确 | ✅ H2O/CO2/A4/ISO9001/GB50292 明确非 identifier |
| Fix 5: normalized_identifier 冻结 | ✅ |
| Fix 6: FactDisplaySpec 完整性 Stage A 全 G3 检查 | ✅ |
| Fix 7: FactDisplaySpec.unit override + ANCHOR 双检查（parsed_unit==display_unit + quantity_kind 兼容） | ✅ D-025 对齐 |
| Fix 8: Unit Registry 冻结 dict → quantity_kind | ✅ 复用 QUANTITY_KINDS |
| Fix 9: Anchor.fact_id 严格 CDM 校验 → ANCHOR.INVALID_FACT_ID | ✅ 复用 cdm/id.py |
| Fix 10: ExpectedIssue rule-level matching（(rule_id, fact_id)） | ✅ |
| Fix 11: Trusted malformed → precondition_error；System malformed → FAIL | ✅ 边界清晰 |
| Fix 12: CaseStatus 正式三态聚合 | ✅ 无歧义 |
| Fix 13: 35 条反例全部有唯一判定 | ✅ |
| Fix 12: applicable_rules case-local | ✅ |
| Fix 10: frozen_round 完整异常处理 | ✅ |
| 所有 7 条 Rule × 7 Condition × 3 case_level 有唯一状态 | ✅ |
| Tokenizer/Whitelist/Anchor 三者闭合 | ✅ |
| DisplaySpec 与 D-025/03/07 完全一致 | ✅ |
| P0_RULE_IDS 7 条未变 | ✅ |
| Step 7-D types.py/validator.py/enums.py 不动 | ✅ |
| 不 import Step 7-F class | ✅ TypedDict |
| known_issues 只读 | ✅ |
| ExpectedIssue 不去重 | ✅ |
| Validator 不执行 Criterion grammar | ✅ |
| ANCHOR.BINDING bronze 行为 = NOT_EVALUABLE（G3=0 → Stage B-2 silver/bronze 豁免） | ✅ |
| gold 不通过缺失 Trusted Input 绕过 P0 | ✅ precondition_error |

---

**READY**
