The user has asked me to produce Step 7-E Coding Contract v5 with 7 specific fixes. Let me first consult the AdvisorTool as required, then produce the revised contract.

The 7 fixes are:
1. Rule execution priority → formal state machine
2. Case-level prevents "empty set passes" → CaseStatus with PASS/FAIL/NOT_EVALUABLE
3. ExpectedIssue one-to-one matching → two-phase: exact fact_id first, then fact_id=None wildcards
4. ActualIssue issue_id → SHA-256 of canonical identity tuple serialization
5. TABLE.INTERNAL precision → TrustedAggregateDef carries its own precision
6. Whitelist scope → formal matching algorithm
7. Anchor conflict detection → formal rules

Let me call the Advisor first.
正在请求专家建议......

已获取专家建议

Advisor 核心洞察：v4 的条件重叠 bug 源于**没有统一的状态机**——bronze + G3=0（NOT_EVALUABLE）同时又 system output missing（FAIL），取决于检查顺序。Fix #1（统一 5 阶段状态机）是基石，同时解决 #2、#7 和最终审计。

关键边界：gold + EXPECTED.HIT + expected_issues=0 是**合法 NOT_EVALUABLE**（case 设计上就没有 expected issues），不是 precondition_error。gold 特例只在"本应存在的可信答案缺失"时触发。

Fix #4：SHA-256 必须冻结 canonical serialization 格式——`json.dumps(list(identity_tuple), ensure_ascii=False, separators=(',', ':'))`，明确禁止 Python `hash()`。

Fix #3：两阶段匹配 + consumed-index set，确定性可复现。

现在输出 v5。


*正在请求专家建议，优化解决方案，该过程耗时可能较长，请耐心等待...*

# Step 7-E Accuracy Validator — Coding Contract v5

---

## 修订基线（7 条定点）

| 编号 | v4 问题 | v5 修正 |
|---|---|---|
| 1 | Rule 执行优先级无统一状态机，条件重叠导致歧义（bronze+G3=0+system missing 可能读成 NOT_EVALUABLE 或 FAIL） | §2 统一 5 阶段状态机，短路语义冻结 |
| 2 | Case 层"全部 NOT_EVALUABLE 仍 passed=True"；gold 空集绕过 P0 | §3 CaseStatus 三态 + gold 特例范围精确化 |
| 3 | ExpectedIssue 匹配顺序无约束，wildcard 可能抢占 exact | §7 两阶段匹配 + consumed-index set |
| 4 | issue_id hash 不稳定（Python built-in hash 跨进程变化） | §4 SHA-256 + 冻结 canonical serialization |
| 5 | TrustedAggregateDef 无自己的 precision | §5 TrustedAggregateDef 新增 precision 字段 |
| 6 | whitelist scope 匹配算法未冻结 | §8 scope matching 算法 + 非法配置 → precondition_error |
| 7 | AnchorDeclaration 冲突/非法 span 无正式定义 | §9 6 类 anchor 冲突场景全部明确 → FAIL |

---

## §1 RuleStatus + CaseStatus

### RuleStatus（单条规则三态，不变）

```python
class RuleStatus(str, Enum):
    PASS = "PASS"                       # 规则适用，正常比较且全部一致
    FAIL = "FAIL"                       # 规则适用，存在不一致或 System Output 本应存在但缺失/malformed
    NOT_EVALUABLE = "NOT_EVALUABLE"     # 规则按 case_level 合法不适用（Trusted Answer 不存在）
```

### CaseStatus（单 Case 三态，Fix 2）

```python
class CaseStatus(str, Enum):
    PASS = "PASS"                       # 无 FAIL，且至少一个 PASS
    FAIL = "FAIL"                       # 存在任何 FAIL
    NOT_EVALUABLE = "NOT_EVALUABLE"     # 7 条 Rule 全部 NOT_EVALUABLE
```

### CaseStatus 聚合规则（Fix 2）

```text
CaseStatus 聚合（从 7 条 RuleResult.status 列表推导）：

    存在 FAIL → CaseStatus = FAIL
    无 FAIL 且至少一个 PASS → CaseStatus = PASS
    7 条全部 NOT_EVALUABLE → CaseStatus = NOT_EVALUABLE

ValidationResult.passed = (CaseStatus == PASS)
    即：CaseStatus.NOT_EVALUABLE 时 passed=False（Fix 2：不允许空集隐式通过）

Gold 特例（精确范围，Fix 2）：
    gold + precondition_error 已在顶层短路，不进入 CaseStatus 聚合
    gold 的 NOT_EVALUABLE 仅允许：EXPECTED.HIT + expected_issues=0（合法——case 设计上就无 expected issues）
    gold 的 NOT_EVALUABLE 不允许：任何本应具备 Trusted Answer 的规则（VALUE/TABLE/LOGIC/ANCHOR/CONCLUSION*）
    → 这些规则若 NOT_EVALUABLE，说明评测用例构造错误，应在 precondition 阶段报错（不是在 CaseStatus 聚合时处理）
```

---

## §2 统一 5 阶段状态机（Fix 1 — 解决条件重叠歧义）

### 状态机定义（所有 7 条 Rule 共用）

```text
Rule.execute(case_level, trusted_input, system_output):

    ┌─────────────────────────────────────────┐
    │ Stage A: Precondition                    │
    │ 检查：case.validate() == []              │
    │ 检查：ExpectedIssue 无 duplicate         │
    │ 检查：WhitelistConfig pattern 合法       │
    │ 检查：TrustedTableSpec source_cells 有效 │
    │ 检查：AnchorDeclaration span 合法        │
    │ （上述任一失败 → precondition_error 顶层短路）│
    └─────────────────────────────────────────┘
                    │ 通过
                    ▼
    ┌─────────────────────────────────────────┐
    │ Stage B: Applicability（Fix 1 短路）    │
    │ 检查：Trusted Answer 是否存在？          │
    │ 检查：case_level 是否允许该规则豁免？    │
    │                                          │
    │ 若 Trusted Answer 不存在 AND             │
    │    case_level 合法豁免 → 短路，          │
    │    RuleStatus = NOT_EVALUABLE            │
    │                                          │
    │ 若 Trusted Answer 不存在 AND             │
    │    case_level 不允许（gold 本应有）       │
    │    → precondition_error 顶层短路         │
    │                                          │
    │ 若 Trusted Answer 存在 → 进入 Stage C     │
    └─────────────────────────────────────────┘
                    │ Trusted Answer 存在
                    ▼
    ┌─────────────────────────────────────────┐
    │ Stage C: System Output Exists            │
    │ 检查：System Output 本规则所需数据是否存在？│
    │                                          │
    │ 不存在 → RuleStatus = FAIL               │
    │ （因为 Stage B 已确认规则适用，系统应产出）│
    │                                          │
    │ 存在 → 进入 Stage D                       │
    └─────────────────────────────────────────┘
                    │ 存在
                    ▼
    ┌─────────────────────────────────────────┐
    │ Stage D: System Output Well-formed       │
    │ 检查：格式合法？                          │
    │ （如：Ref 有 fact_id；status 非空；        │
    │   Anchor span 在 raw_text 范围内等）       │
    │                                          │
    │ 不合法 → RuleStatus = FAIL               │
    │ 合法 → 进入 Stage E                       │
    └─────────────────────────────────────────┘
                    │ 合法
                    ▼
    ┌─────────────────────────────────────────┐
    │ Stage E: Compare                         │
    │ 执行规则比较逻辑 → PASS 或 FAIL          │
    │ FAIL 时产生 ActualIssue[]                 │
    └─────────────────────────────────────────┘
```

### Fix 1 短路语义（解决条件重叠）

**Stage B 是关键短路点**：一旦判定规则 NOT_EVALUABLE，**绝不进入 Stage C/D/E**。

| 场景 | Stage B 判定 | 最终 RuleStatus | 理由 |
|---|---|---|---|
| bronze + G3=0 + system_output 缺失 | G3=0，bronze 合法豁免 → **短路** | NOT_EVALUABLE | 不因 system 缺失而 FAIL |
| gold + G3=0 | G3=0，gold 不豁免 → **precondition_error 短路** | — | 评测用例构造错误 |
| silver + confirmed_criteria=0 + system_evaluations 缺失 | confirmed_criteria=0，silver 合法豁免 → **短路** | NOT_EVALUABLE | 不因 system 缺失而 FAIL |
| silver + confirmed_criteria 有 + system_evaluations 缺失 | confirmed_criteria 有 → 进入 Stage C | FAIL | 系统应产出但缺失 |
| gold + confirmed_criteria 有 + system_evaluations 缺失 | confirmed_criteria 有 → 进入 Stage C | FAIL | 系统应产出但缺失 |
| bronze + expected_issues=0 | expected_issues=0，bronze 合法豁免 → **短路** | NOT_EVALUABLE | 不因 system 缺失而 FAIL |

### 7 条 Rule 各自的 Stage B 判定条件

| Rule | Stage B：什么是"Trusted Answer 存在"？ | 合法 NOT_EVALUABLE case_level |
|---|---|---|
| VALUE.CONSISTENCY | `len(G3.facts) > 0` | silver / bronze |
| TABLE.INTERNAL | `len(trusted_table_specs) > 0` | silver / bronze |
| LOGIC.CONSISTENCY | `len(expected_evaluations) > 0` | silver / bronze |
| ANCHOR.BINDING | `len(G3.facts) > 0` | silver / bronze 🔴（bronze 无 G3 → NOT_EVALUABLE，不因 span 问题 FAIL） |
| CONCLUSION.DIRECTION | `G4.direction is not None` | silver（降级） / bronze |
| CONCLUSION.COVERAGE | `len(G4.required_evaluation_ids) > 0` | silver / bronze |
| EXPECTED.HIT | **无**——expected_issues=0 是合法设计 | gold / silver / bronze（全部 case_level 都可以 NOT_EVALUABLE） |

### 7 条 Rule 各自的 Stage C 检查项（System Output 缺失）

| Rule | Stage C：System Output 应提供什么？ |
|---|---|
| VALUE.CONSISTENCY | `report_ir.ref_occurrences` 非空 |
| TABLE.INTERNAL | `structured_tables` 有匹配 trusted_table_specs 的 table_id |
| LOGIC.CONSISTENCY | `system_evaluations` 非空 |
| ANCHOR.BINDING | `rendered_segments` 非空 |
| CONCLUSION.DIRECTION | `report_ir.conclusion_blocks` 非空 |
| CONCLUSION.COVERAGE | `report_ir.conclusion_blocks` 非空 + conclusion_block 有 covers[] |
| EXPECTED.HIT | **无强制要求**——system_reported_issues 为空在 Stage B 后由匹配逻辑决定（不是 Stage C） |

---

## §3 ValidationResult + RuleResult（不变，配套 CaseStatus）

### ValidationResult

```python
@dataclass
class ValidationResult:
    case_id: str
    case_level: str
    case_status: CaseStatus                    # Fix 2：三态
    passed: bool                               # Fix 2：= (case_status == CaseStatus.PASS)
    rule_results: list[RuleResult]             # 固定 7 条
    actual_issues: list[ActualIssue]
    expected_coverage: ExpectedIssueCoverage
    summary: ValidationSummary
    precondition_error: Optional[str] = None   # 非 None → case_status=FAIL, rule_results=[]
```

### RuleResult（三态，不变）

```python
@dataclass
class RuleResult:
    rule_id: str
    status: RuleStatus
    issues_count: int
    issues: list[ActualIssue]
    not_evaluable_reason: Optional[str] = None
```

### ExpectedIssueCoverage（Fix 3：不做去重）

```python
@dataclass
class ExpectedIssueCoverage:
    expected_total: int                        # 原样来自 case.expected_issues
    expected_hit: int                          # two-phase matching 命中数
    expected_miss: int                         # expected_total - expected_hit
    hit_rate: float                            # expected_total=0 → 0.0
    unmatched: list[ExpectedIssue]
```

---

## §4 ActualIssue（Fix 4：SHA-256 稳定 hash）

```python
@dataclass(frozen=True)
class ActualIssue:
    """Accuracy Validator 独立产生的问题。

    Fix 4：issue_id = SHA-256(canonical_serialization(identity_tuple))
    跨进程、跨机器、跨运行稳定。Python built-in hash() 禁止使用。
    """
    identity: tuple                   # 不可变元组：(rule_id, issue_type, location, fact_id, evaluation_id, occurrence_id, table_aggregate)
    issue_id: str                      # SHA-256 hex digest

    rule_id: str
    severity: str
    issue_type: str
    fact_id: Optional[str] = None
    evaluation_id: Optional[str] = None
    location: Optional[str] = None
    token_identity: Optional[TokenOccurrence] = None
    expected: Any = None
    actual: Any = None
    message: str = ""

    @staticmethod
    def build_identity(
        rule_id: str,
        issue_type: str,
        location: str = "",
        fact_id: Optional[str] = None,
        evaluation_id: Optional[str] = None,
        occurrence_id: Optional[str] = None,
        table_aggregate: Optional[str] = None,
    ) -> tuple:
        return (
            rule_id,
            issue_type,
            location,
            fact_id or "",
            evaluation_id or "",
            occurrence_id or "",
            table_aggregate or "",
        )

    @staticmethod
    def compute_issue_id(identity: tuple) -> str:
        """Fix 4：SHA-256 of canonical JSON serialization.

        Canonical serialization 格式（冻结）：
            json.dumps(list(identity), ensure_ascii=False, separators=(',', ':'))
            → UTF-8 encode
            → SHA-256
            → hex digest (64 chars)

        禁止：Python built-in hash()（跨进程随机化）
        禁止：string concatenation with delimiters（delimiter collision risk）
        禁止：任何加盐或随机因子
        """
        import hashlib
        import json
        canonical = json.dumps(list(identity), ensure_ascii=False, separators=(',', ':'))
        return hashlib.sha256(canonical.encode('utf-8')).hexdigest()
```

### Fix 4 canonical serialization 示例

```python
identity = ("ANCHOR.BINDING", "ANCHOR.UNBOUND", "semantic:x/block-y", "", "", "loc@12-17", "")
canonical = json.dumps(list(identity), ensure_ascii=False, separators=(',', ':'))
# → ["ANCHOR.BINDING","ANCHOR.UNBOUND","semantic:x/block-y","","","loc@12-17",""]
# → SHA-256 → "a1b2c3d4..." (64 chars)
```

---

## §5 TrustedInput（不变，配套 Fix 5）

```python
@dataclass
class TrustedInput:
    case: EvaluationCase
    expected_evaluations: list[ExpectedEvaluationSnapshot]   # Fix 6：外部预计算
    confirmed_criteria: list[CriterionSnapshot]
    whitelist: WhitelistConfig
    fact_display_spec: FactDisplaySpecSnapshot
    trusted_table_specs: list[TrustedTableSpec]
```

### §5.1 TrustedAggregateDef（Fix 5：自带 precision）

```python
@dataclass
class TableCellRef:
    row: int
    col: int

@dataclass
class TrustedAggregateDef:
    """Fix 5：explicit source_cells + 自带 precision。"""
    row: int
    col: int
    kind: str                               # "sum" | "mean" | "min" | "max" | "count"
    source_cells: list[TableCellRef]        # 精确坐标，无 wildcards
    precision: int                          # Fix 5：该 aggregate 自己的 comparison precision

@dataclass
class TrustedTableSpec:
    table_id: str
    aggregates: list[TrustedAggregateDef]
    data_cells: list[TableCellRef]
```

### Fix 5 TABLE.INTERNAL 精度使用

```text
对每个 TrustedAggregateDef:
    1. 从 source_cells 坐标取 data_values
    2. Validator 独立计算 recomputed = compute(kind, data_values)
    3. relative error 检查：
       if abs(recomputed - system_aggregate) / max(abs(recomputed), 1e-12) > 1e-9
           → TABLE.{KIND}_MISMATCH (FAIL)
    4. Fix 5：用 TrustedAggregateDef.precision 做 round 比较（不是 G3.precision）
       if round(recomputed, aggregate.precision) != round(system_aggregate, aggregate.precision)
           → TABLE.{KIND}_MISMATCH (FAIL)
```

### §5.2 ExpectedEvaluationSnapshot + CriterionSnapshot（Fix 6，不变）

```python
class ExpectedEvaluationSnapshot(TypedDict):
    """Fix 6：TrustedInput 直接提供 expected status，Validator 退化为纯比较器。"""
    evaluation_id: str
    criterion_id: str
    fact_id: str
    expected_status: str

class CriterionSnapshot(TypedDict):
    criterion_id: str
    applicable_fact_ids: list[str]
    rule: str
    limit_value: Any
    limit_unit: Optional[str]
    limit_quantity_kind: Optional[str]
    # status_mapping 已移除（Fix 6）
```

---

## §6 SystemOutput（不变）

```python
@dataclass
class SystemOutput:
    report_ir: IReportIRSnapshot
    rendered_segments: list[RenderedSegment]
    structured_tables: list[StructuredTable]
    system_reported_issues: list[SystemReportedIssue]
    system_evaluations: list[IEvaluationSnapshot]
```

### Anchor 契约（Fix 7 配套）

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
    fact_id: str

    @property
    def occurrence_id(self) -> str:
        return f"{self.segment_location}@{self.char_start}-{self.char_end}"

@dataclass
class TokenOccurrence:
    segment_location: str
    char_start: int
    char_end: int
    token_text: str
    token_kind: str

    @property
    def occurrence_id(self) -> str:
        return f"{self.segment_location}@{self.char_start}-{self.char_end}"
```

### StructuredTable（不变）

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

## §7 expected_issues 两阶段匹配（Fix 3 — 确定性）

### 算法（冻结）

```text
输入：
    expected_issues: list[ExpectedIssue]     # Fix 3：原样来自 case.expected_issues，不去重
    system_reported_issues: list[SystemReportedIssue]

辅助：
    consumed_system_indices: set[int] = set()   # 已被消耗的 SystemReportedIssue 索引

阶段 1：Exact fact_id 匹配（Fix 3：exact 永远优先）
    for ei in expected_issues:
        if ei.fact_id is None:
            continue                               # 跳过 wildcard，阶段 2 处理
        for idx, sri in enumerate(system_reported_issues):
            if idx in consumed_system_indices:
                continue
            if sri.rule_id == ei.rule_id and sri.fact_id == ei.fact_id:
                consumed_system_indices.add(idx)
                ei.matched = True
                break
        # 内层循环结束没找到 → ei.matched 保持 False（不是 continue 跳过外层）

阶段 2：Rule-level wildcard 匹配（fact_id=None）
    for ei in expected_issues:
        if ei.fact_id is not None or ei.matched:
            continue                               # 只处理未匹配的 wildcard
        for idx, sri in enumerate(system_reported_issues):
            if idx in consumed_system_indices:
                continue
            if sri.rule_id == ei.rule_id:
                consumed_system_indices.add(idx)
                ei.matched = True
                break

产出：
    expected_hit = sum(1 for ei in expected_issues if ei.matched)
    expected_miss = expected_total - expected_hit
    unmatched = [ei for ei in expected_issues if not ei.matched]
```

### Fix 3 匹配约束

| 约束 | 实现 |
|---|---|
| 一条 SystemReportedIssue 最多消耗一次 | consumed_system_indices set |
| 一条 ExpectedIssue 最多命中一次 | ei.matched flag |
| 不修改 expected_issues 原始数量 | expected_total = len(case.expected_issues) |
| 不去重 | 入口检测 duplicate → precondition_error（§2 Stage A） |
| 确定性 | 按 expected_issues 原始顺序、system_reported_issues 原始顺序遍历 |
| exact 永远优先于 wildcard | 阶段 1 先跑，阶段 2 只处理未匹配的 wildcard |

### Fix 3 反例测试（必须通过）

```python
def test_wildcard_cannot_steal_exact_match():
    # 两个 ExpectedIssue：一个 exact，一个 wildcard，同一个 rule_id
    # 只有一条 SystemReportedIssue（fact_id 精确匹配 exact 的那个）
    case = EvaluationCase(expected_issues=[
        ExpectedIssue(rule_id="FACT.MISSING", severity="error", fact_id="fact:component.K001.length"),  # exact
        ExpectedIssue(rule_id="FACT.MISSING", severity="error", fact_id=None),                          # wildcard
    ])
    system = SystemOutput(system_reported_issues=[
        SystemReportedIssue(rule_id="FACT.MISSING", fact_id="fact:component.K001.length"),  # 精确匹配第一个
    ])
    result = validator.validate(case, system, trusted_input)
    # 阶段 1：exact 命中（consumed={0}）
    # 阶段 2：wildcard 找不到未消耗的 → miss
    assert result.expected_coverage.expected_hit == 1
    assert result.expected_coverage.expected_miss == 1
    assert result.case_status == CaseStatus.FAIL
```

---

## §8 Whitelist（Fix 6：scope matching）

### WhitelistConfig + WhitelistEntry（scope 字段明确化）

```python
@dataclass
class WhitelistConfig:
    version: str
    entries: list[WhitelistEntry]

@dataclass
class WhitelistEntry:
    pattern: str                               # 正则模式
    reason: str
    scope: str                                 # Fix 6：scope matching 规则
    confirmed_by: str
```

### Fix 6 scope matching 算法（冻结）

```text
对每个 (TokenOccurrence, RenderedSegment) 对:

    for whitelist_entry in whitelist_config.entries:
        if not scope_matches(whitelist_entry.scope, rendered_segment.segment_location):
            continue                           # scope 不匹配 → 这条 entry 不适用

        if re.search(whitelist_entry.pattern, token_occurrence.token_text):
            return WHITELIST_HIT               # scope + pattern 都匹配 → 通过

    return NO_WHITELIST_MATCH                  # 无 entry 匹配 → 继续 ANCHOR 检查

scope_matches 算法（Fix 6，冻结）:
    whitelist_scope = entry.scope.strip()

    if whitelist_scope == "global":
        return True                            # 所有 segment_location 都匹配

    if whitelist_scope.startswith("semantic:"):
        required_semantic = whitelist_scope[len("semantic:"):]
        # 标准化 segment_location：取第一个 "/" 之前的部分为 semantic
        segment_semantic = rendered_segment.segment_location.split("/")[0]
        # 精确匹配（不是前缀——防止 "semantic:inspiration" 匹配 "semantic:inspiration_result"）
        return segment_semantic == required_semantic

    # 非法 scope 值 → precondition_error（在 Stage A 检测）
    raise ValueError(f"Invalid whitelist scope: {whitelist_scope}")
```

### Fix 6 WhitelistConfig 入口校验（Stage A）

```python
def _validate_whitelist_config(self, whitelist: WhitelistConfig) -> Optional[str]:
    for idx, entry in enumerate(whitelist.entries):
        # scope 合法值检查
        if entry.scope != "global" and not entry.scope.startswith("semantic:"):
            return f"Whitelist[{idx}]: invalid scope '{entry.scope}' — must be 'global' or 'semantic:X'"
        # pattern 正则可编译检查
        try:
            re.compile(entry.pattern)
        except re.error as e:
            return f"Whitelist[{idx}]: pattern '{entry.pattern}' not a valid regex: {e}"
    return None
```

---

## §9 Anchor 冲突检测（Fix 7 — 6 类场景全部明确）

### Fix 7 Stage D（System Output Well-formed）Anchor 检查

在 ANCHOR.BINDING 的 Stage D，执行以下 6 类格式/冲突检查，任何一个失败 → FAIL：

| # | 冲突场景 | 判定 | 检测方式 |
|---|---|---|---|
| A7-1 | AnchorDeclaration span 越界 | FAIL | `char_start < 0 or char_end > len(raw_text) or char_start >= char_end` |
| A7-2 | AnchorDeclaration span 与任何 TokenOccurrence 不完全一致 | FAIL | 对每个 AnchorDeclaration：必须存在 TokenOccurrence（segment_location + char_start + char_end 精确相等） |
| A7-3 | 一个 TokenOccurrence 被多个 AnchorDeclaration 占用 | FAIL | 按 (segment_location, char_start, char_end) 分组，任何一组 size > 1 → 冲突 |
| A7-4 | 一个 AnchorDeclaration 找不到对应 TokenOccurrence | FAIL | 与 A7-2 同一个检查（双向覆盖） |
| A7-5 | 同一 token_text 出现多次只声明一次 AnchorDeclaration | FAIL | 对每个 TokenOccurrence（按 token_text 分组）：同组的每个 occurrence 都必须有自己的 AnchorDeclaration |
| A7-6 | AnchorDeclaration 使用 token_text fallback 匹配多个 occurrence | FAIL | **Fix 7：AnchorDeclaration 不携带 token_text 字段**——从数据结构层面禁止这种 fallback |

### Fix 7 Stage B 的 ANCHOR.BINDING 短路

```text
Stage B (Applicability) for ANCHOR.BINDING:
    if len(G3.facts) == 0:
        if case_level in ("silver", "bronze"):
            RuleStatus = NOT_EVALUABLE（短路，不进入 Stage C/D/E）
            # Fix 7 关键：bronze 因 G3=0 NOT_EVALUABLE
            # 不因 Anchor span 越界或 token 问题而 FAIL
        else:  # gold
            precondition_error（gold 本应有 G3）
```

### Fix 7 Stage E 的正常比较（Anchor 通过 Stage D 后）

```text
对每个 TokenOccurrence（Stage D 已确认每个都有 AnchorDeclaration）:

    a. 查 whitelist（§8 算法）
       命中 → 通过

    b. 用 AnchorDeclaration.fact_id 查 G3
       fact_id 不在 G3 → ActualIssue(ANCHOR.UNKNOWN_FACT)

    c. round(token_numeric, G3.precision) == round(gt_value, G3.precision)?
       否 → ActualIssue(ANCHOR.VALUE_MISMATCH)

    d. identifier token（构件编号等）额外检查：
       Anchor.fact_id 的 instance_key == token_text?
       否 → ActualIssue(ANCHOR.WRONG_FACT)
```

---

## §10 TypedDict Protocol + 外部配置（不变）

```python
from typing import TypedDict, List, Optional, Any

class ConclusionBlockSnapshot(TypedDict):
    semantic: str                                       # "conclusion"
    direction: str
    covers: List[str]

class RefOccurrence(TypedDict):
    fact_id: str
    value: Any
    unit: Optional[str]

class IReportIRSnapshot(TypedDict):
    conclusion_blocks: List[ConclusionBlockSnapshot]
    ref_occurrences: List[RefOccurrence]

class ExpectedEvaluationSnapshot(TypedDict):
    evaluation_id: str
    criterion_id: str
    fact_id: str
    expected_status: str

class IEvaluationSnapshot(TypedDict):
    evaluation_id: str
    criterion_id: str
    fact_id: str
    status: str
```

---

## §11 完整最终重审（Fix 9 — 每条 Rule × 5 Stage × 3 case_level）

| Rule | Stage B (Applicability) | Stage C (Sys Output) | Stage D (Well-formed) | Stage E (Compare) | case_level |
|---|---|---|---|---|---|
| VALUE.CONSISTENCY | G3=0 → silver/bronze NOT_EVALUABLE；gold precondition | ref_occurrences 缺失 → FAIL | Ref 缺 fact_id/value → FAIL | round 值/单位/量纲一致 → PASS；任一 mismatch/missing → FAIL | gold/silver/bronze |
| TABLE.INTERNAL | trusted_table_specs=0 → silver/bronze NOT_EVALUABLE；gold precondition | 无匹配 table_id → FAIL | source_cells 在 system 中无数据 → FAIL | rel.err ≤ 1e-9 + precision round 一致 → PASS；任一 mismatch → FAIL | gold/silver/bronze |
| LOGIC.CONSISTENCY | expected_evaluations=0 → silver/bronze NOT_EVALUABLE；gold precondition | system_evaluations 缺失 → FAIL | status 为空 → FAIL | expected_status == system_status → PASS；任一不等 → FAIL | gold/silver/bronze |
| ANCHOR.BINDING | G3=0 → silver/bronze **NOT_EVALUABLE**（Fix 2）；gold precondition | rendered_segments 缺失 → FAIL | §9 6 类冲突 → FAIL（包括 span 越界/重复/缺失） | 全部 token 命中 whitelist 或正确锚定 + round 值相等 → PASS；任一 → FAIL | gold/silver/bronze |
| CONCLUSION.DIRECTION | G4.direction=None → silver/bronze NOT_EVALUABLE；gold precondition | 无 conclusion block → FAIL | direction 为空/不在枚举域 → FAIL | 枚举精确相等 → PASS；不等 → FAIL | gold/silver/bronze |
| CONCLUSION.COVERAGE | required_evaluation_ids=0 → silver/bronze NOT_EVALUABLE；gold precondition | 无 conclusion block → FAIL | covers[] 缺失 → FAIL | required_ids ⊆ covers_union → PASS；gap → FAIL | gold/silver/bronze |
| EXPECTED.HIT | expected_issues=0 → **全部 case_level NOT_EVALUABLE**（Fix 2 合法） | system_reported_issues 缺失但 expected_issues>0 → FAIL | — | 全部命中 → PASS；任一未命中 → FAIL | gold/silver/bronze |

### Fix 2 CaseStatus 聚合验证

| 7 条 RuleResult 组合 | CaseStatus | passed |
|---|---|---|
| [FAIL, PASS, PASS, PASS, PASS, PASS, PASS] | FAIL | False |
| [PASS, PASS, PASS, PASS, PASS, PASS, PASS] | PASS | True |
| [NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL] | NOT_EVALUABLE | **False** |
| [NOT_EVAL, NOT_EVAL, PASS, PASS, NOT_EVAL, NOT_EVAL, NOT_EVAL] | PASS | True |
| [FAIL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL, NOT_EVAL] | FAIL | False |

---

## §12 明确不做

| 编号 | 不做 | 原因 |
|---|---|---|
| M1 | 不修改 `types.py` / `enums.py` / `validator.py` | Step 7-D 冻结 |
| M2 | 不实现 parser / renderer | 只定义 SystemOutput 契约 |
| M3 | 不 import 未来 Step 7-F class | TypedDict |
| M4 | known_issues 不参与判定 | 诊断参考 |
| M5 | whitelist 放进 EvaluationCase | 版本化外部配置 |
| M6 | 不新增 P0 Rule | 7 条冻结 |
| M7 | 不判断构建门禁 | M7 eval runner 责任 |
| M8 | 不执行 rule evaluation | Fix 6：ExpectedEvaluationSnapshot 外部预计算 |
| M9 | 不做 LLM 语义相似度 | 枚举精确相等 |
| M10 | 不处理 Template display precision | 不属于 P0 |
| M11 | 不 deduplicate ExpectedIssue | Fix 3：入口检测 duplicate → precondition_error |
| M12 | 不推导 rule grammar | Fix 6 |
| M13 | AnchorDeclaration 不携带 token_text 字段 | Fix 7：从数据结构层面禁止 token_text fallback |

---

## §13 风险审查

| 编号 | 风险 | 缓解 |
|---|---|---|
| R1 | ExpectedEvaluationSnapshot 暂无 gold/silver 用例 | silver/bronze 自动 NOT_EVALUABLE；gold 必须有否则 precondition_error |
| R2 | TrustedAggregateDef precision 配置工作量 | 每个 aggregate 在 TrustedTableSpec 中单独配置，数量有限；配置错误 precondition 阶段发现 |
| R3 | SHA-256 digest 长度（64 chars） | 可截断为前 16 chars（仍安全）；冻结规则避免未来截断不一致 |
| R4 | one-to-one matching 对 ExpectedIssue 顺序敏感 | 按 expected_issues 原始顺序遍历；测试覆盖顺序敏感性 |
| R5 | gold EXPECTED.HIT expected_issues=0 合法 NOT_EVALUABLE | Fix 2 明确：这是唯一允许 gold 出现 NOT_EVALUABLE 的 Rule |

---

## READY 判定

逐条检查：

| 检查项 | 结论 |
|---|---|
| Fix 1：统一 5 阶段状态机（precondition → applicability → sys output exists → well-formed → compare）+ 短路语义 | ✅ |
| Fix 2：CaseStatus 三态 + CaseStatus=NOT_EVALUABLE 时 passed=False | ✅ |
| Fix 2：gold 特例精确范围（仅 EXPECTED.HIT + expected_issues=0 合法 NOT_EVALUABLE） | ✅ |
| Fix 3：两阶段匹配（exact first → wildcard）+ consumed-index set + 反例测试 | ✅ |
| Fix 4：SHA-256 冻结序列化格式 + Python hash() 禁止 | ✅ |
| Fix 5：TrustedAggregateDef 自带 precision + relative error ≤ 1e-9 再 round | ✅ |
| Fix 6：scope matching 算法（global / semantic:X 精确匹配）+ 非法 entry → precondition_error | ✅ |
| Fix 7：Anchor 6 类冲突场景全部明确 → FAIL | ✅ |
| Fix 7：AnchorDeclaration 无 token_text 字段（数据结构层面禁止 fallback） | ✅ |
| bronze 不会因 G3=0 被错误判 P0 FAIL（ANCHOR.BINDING NOT_EVALUABLE） | ✅ |
| silver 不会因未覆盖维度被误判（NOT_EVALUABLE 合法豁免） | ✅ |
| gold 不会通过缺失 Trusted Input 绕过 P0（→ precondition_error） | ✅ |
| NOT_EVALUABLE 不会被隐式算成 PASS（CaseStatus=NOT_EVALUABLE → passed=False） | ✅ |
| ExpectedIssue 数量不会被偷偷修改（原样 len(case.expected_issues)） | ✅ |
| P0_RULE_IDS 仍然只有 7 条 | ✅ |
| 不引入新的 P0 Rule | ✅ |
| 不修改 Step 7-D types.py / validator.py / enums.py | ✅ |
| Validator 不执行 Criterion rule | ✅ |
| Validator 不生成 GroundTruth | ✅ |
| Validator 不修改 known_issues | ✅ |
| Validator 不自己 deduplicate ExpectedIssue | ✅ |
| 不产生 build gate | ✅ |
| 不依赖未来 Step 7-F Python class | ✅ |
| 所有状态转换、匹配算法、唯一 ID、precision、scope、anchor 冲突处理都可直接写成测试 | ✅ |

---

**READY**
