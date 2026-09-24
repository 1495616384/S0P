*正在请求专家建议，优化解决方案，该过程耗时可能较长，请耐心等待...*

先确认 Step 7-D 对重复 ExpectedIssue 的实际校验行为（Fix 3 依赖此证据）：
用户又提出了 9 个精修点，我必须先调用 AdvisorTool 再输出 v4 Contract。
正在请求专家建议....

已获取专家建议

Advisor 给出了 6 条具体建议，其中最关键的是 Fix 6（TrustedInput 提供 ExpectedEvaluationSnapshot，Validator 退化为纯比较器）和 Fix 3（确认 Step 7-D 是否已经在入口处拒绝重复 ExpectedIssue）。让我先确认 Step 7-D 对 duplicate ExpectedIssue 的实际行为：

Step 7-D 的 `validate_expected_issues()` 实现（validator.py:287-292）仅对每个 ExpectedIssue 做字段级校验（rule_id 非空、severity 合法、fact_id 语法），**没有检查重复**。这意味着：

- 若 EvaluationCase 里存在两条完全相同的 ExpectedIssue，Step 7-D 不会拦住
- Accuracy Validator 拿到 case 后，必须在入口显式检测 duplicate ExpectedIssue，触发 precondition_error
- 不能在 Accuracy Validator 里偷偷 deduplicate——Fix 3 冻结了这个行为

Fix 3 的结论明确：重复 ExpectedIssue 到达 Validator → precondition_error。
关键证据已确认：Step 7-D `validate_expected_issues()`（validator.py:287-292）**不检查 duplicate**。因此 Fix 3 的结论是：Accuracy Validator 入口必须自己检测重复 ExpectedIssue，检测到 → precondition_error，不做内部去重。

Fix 3 证据齐备。Fix 6 是最高杠杆决策——让 TrustedInput 直接提供 ExpectedEvaluationSnapshot，Validator 退化为纯比较器。现在输出 v4。

---

# Step 7-E Accuracy Validator — Coding Contract v4

---

## 修订基线（9 条定点）

| 编号 | v3 问题 | v4 修正 |
|---|---|---|
| 1 | Rule Applicability 未正式定义，三态传播规则缺明确矩阵 | §2 完整 7×6 决策矩阵，每格唯一 bucket |
| 2 | ANCHOR.BINDING 在 bronze 因 G3 为空被错误判 FAIL | §2 规则决策矩阵 ANCHOR.BINDING bronze=NOT_EVALUABLE |
| 3 | Accuracy Validator 偷偷 deduplicate ExpectedIssue | §F 取消内部去重；入口检测 duplicate → precondition_error（Fix 3 证据：validator.py:287-292 不做 duplicate 检查） |
| 4 | fact_id=None 语义未冻结 | §F 冻结：fact_id=None → rule-level wildcard；one-to-one matching |
| 5 | TrustedAggregateDef 用 row_range 不够精确 | §5 TrustedAggregateDef 用 explicit source_cells: list[TableCellRef]；恢复 D-025 relative error ≤ 1e-9 再 G3.precision round |
| 6 | Validator 可能膨胀成规则引擎 | TrustedInput 直接提供 ExpectedEvaluationSnapshot[]，Validator 退化为纯字符串精确比较器。CLOSE CriterionSnapshot.status_mapping 字段 |
| 7 | issue_id 用字符串拼接，有碰撞风险 | ActualIssue.identity 改为 frozen tuple + stable hash |
| 8 | Precision 四者未分层 | §3 四 precision 来源明确分工；Template display precision 错误不属于 P0 |
| 9 | 7 条 Rule × 6 种场景未全部明确 | §2 完整决策矩阵 |

---

## §1 RuleStatus 四态定义

```python
class RuleStatus(str, Enum):
    PASS = "PASS"                       # 验证通过：所有可比较项一致
    FAIL = "FAIL"                       # 验证失败：可比较项存在不一致
    NOT_EVALUABLE = "NOT_EVALUABLE"     # 无可信答案，按 case_level 合法豁免
    # precondition_error 不是 RuleStatus
    # 是 ValidationResult.precondition_error 顶层字段
```

**严格区分**：
- `precondition_error`：评测用例构造错误，**所有规则都不执行**，`ValidationResult.passed=False`
- `NOT_EVALUABLE`：规则按 case_level 合法豁免，**该规则执行但无可信答案**，不计入 FAIL

---

## §2 完整决策矩阵（7 Rule × 6 Condition × 3 case_level）

### 图例

```text
[ ]  = NOT_EVALUABLE
{ }  = precondition_error（短路，该 case 规则全部不执行）
FAIL = 规则 FAIL
PASS = 规则 PASS
```

### 决策矩阵表（每格唯一 bucket）

| Rule | Condition | gold | silver | bronze |
|---|---|---|---|---|
| **VALUE.CONSISTENCY** | Trusted Answer 缺失（G3 facts=0） | {precondition} — gold 应有 G3 | [NOT_EVALUABLE] | [NOT_EVALUABLE] |
|  | System Output 缺失（无 ref_occurrences） | FAIL | FAIL | FAIL |
|  | System 格式错误（Ref 缺 fact_id / value 类型不对） | FAIL | FAIL | FAIL |
|  | 正常比对且全部一致 | PASS | PASS | — |
|  | 正常比对但任一 mismatch / missing | FAIL | FAIL | — |
| **TABLE.INTERNAL** | Trusted Answer 缺失（trusted_table_specs=0） | {precondition} | [NOT_EVALUABLE] | [NOT_EVALUABLE] |
|  | TrustedTableSpec 有但 System 无匹配 table_id | FAIL | FAIL | — |
|  | System 格式错误（cell role 与 trusted spec 不一致） | FAIL | FAIL | — |
|  | 正常独立重算且一致 | PASS | PASS | — |
|  | 正常独立重算但任一 mismatch | FAIL | FAIL | — |
| **LOGIC.CONSISTENCY** | Trusted Answer 缺失（expected_evaluations=0） | {precondition} | [NOT_EVALUABLE] | [NOT_EVALUABLE] |
|  | System Output 缺失（system_evaluations=0 但 trusted 有） | FAIL | FAIL | — |
|  | System 格式错误（status 为空字符串） | FAIL | FAIL | — |
|  | 正常精确比较且全部 status 相等 | PASS | PASS | — |
|  | 正常比较但任一不等 | FAIL | FAIL | — |
| **ANCHOR.BINDING** | Trusted Answer 缺失（G3 facts=0） | {precondition} | [NOT_EVALUABLE] | [NOT_EVALUABLE] 🔴 Fix 2 |
|  | System Output 缺失（rendered_segments=0） | FAIL | FAIL | FAIL |
|  | AnchorDeclaration span 与 TokenOccurrence span 无匹配 | FAIL（全部 ANCHOR.UNBOUND） | FAIL | — |
|  | 正常扫描且全部 token 命中 whitelist 或正确锚定 | PASS | PASS | — |
|  | 正常扫描但任一 token 未命中 | FAIL（ANCHOR.UNBOUND 等） | FAIL | — |
| **CONCLUSION.DIRECTION** | Trusted Answer 缺失（G4.direction=None） | {precondition} | [NOT_EVALUABLE]（或按 05_EVAL §2 降级为 bronze） | [NOT_EVALUABLE] |
|  | System Output 缺失（无 conclusion block） | FAIL | FAIL | — |
|  | System 格式错误（direction 为空/不在枚举域） | FAIL | FAIL | — |
|  | 枚举精确相等 | PASS | PASS | — |
|  | 枚举不等 | FAIL | FAIL | — |
| **CONCLUSION.COVERAGE** | Trusted Answer 缺失（G4.required_evaluation_ids=0） | {precondition} | [NOT_EVALUABLE] | [NOT_EVALUABLE] |
|  | System Output 缺失（无 conclusion block） | FAIL | FAIL | — |
|  | conclusion block 无 covers[] 字段 | FAIL | FAIL | — |
|  | required_ids ⊆ covers_union | PASS | PASS | — |
|  | required_ids - covers_union ≠ ∅ | FAIL | FAIL（每条 gap 一个 ActualIssue） | — |
| **EXPECTED.HIT** | Trusted Answer 缺失（expected_issues=0） | [NOT_EVALUABLE] | [NOT_EVALUABLE] | [NOT_EVALUABLE] |
|  | Trusted Input 格式错误（重复 ExpectedIssue） | {precondition} 🔴 Fix 3 | {precondition} | {precondition} |
|  | System Output 缺失（system_reported_issues=0 但 expected_issues>0） | FAIL | FAIL | — |
|  | 全部 ExpectedIssue 被命中 | PASS | PASS | — |
|  | 存在未命中 | FAIL | FAIL | — |

### 关键边界说明

| 边界 | 规则 |
|---|---|
| Fix 2 明确：bronze G3=0 → ANCHOR.BINDING=NOT_EVALUABLE | 05_EVAL §2 bronze 无 G3，无可信 Fact 答案，**不能**报 ANCHOR.UNKNOWN_FACT / FAIL |
| Fix 1 明确：gold Trusted Answer 缺失 → precondition_error | gold 应具备全部可信答案，缺失是评测用例构造错误 |
| System Output 缺失 ≠ Trusted Answer 缺失 | System 本应产出但没产出 → FAIL；Trusted 本应存在但评测用例没提供 → NOT_EVALUABLE（silver/bronze）或 precondition_error（gold） |
| EXPECTED.HIT expected_issues=0 → NOT_EVALUABLE | 无可信 expected_issues 无法评测命中率，**不是隐式 PASS** |

---

## §3 Precision 四者分层（Fix 8）

### 四个 precision 来源

```text
A. GroundTruthFact.precision        —— 来自 Step 7-D EvaluationCase.ground_truth.facts[].precision
                                      含义：G3 值的规范记录精度
                                      用途：VALUE.CONSISTENCY / ANCHOR.BINDING 的 round 控制

B. TemplateSpec.fact_display_spec    —— 模板声明的显示小数位（D-025）
                                      含义：渲染时模板对某 fact_type 声明的显示格式
                                      用途：**不属于 P0 Accuracy Validator**（Fix 8 结论）
                                            Template display 格式合规性属于未来 P1/P2 检查

C. Canonical value comparison        —— VALUE.CONSISTENCY 的比较公式
                                      = round(system_canonical_value, G3.precision)
                                        vs round(gt_value, G3.precision)

D. Rendered token comparison         —— ANCHOR.BINDING 的 token 数值比较
                                      = round(token_numeric, G3.precision)
                                        vs round(gt_value, G3.precision)
```

### 各 P0 Rule 的 precision 使用

| Rule | precision 来源 | 附加公式 |
|---|---|---|
| VALUE.CONSISTENCY | G3.precision | `round(system_val, p) == round(gt_val, p)` |
| ANCHOR.BINDING | G3.precision | `round(token_val, p) == round(gt_val, p)` |
| TABLE.INTERNAL | 先 relative error ≤ 1e-9（派生量先验正确性），再用 G3.precision round | 两步：① `abs(recomputed - system) / max(abs(recomputed), 1e-12) ≤ 1e-9`；② `round(recomputed, p) == round(system, p)` |
| LOGIC.CONSISTENCY | 不涉及（布尔比较） | `expected_status == system_status`（字符串精确相等） |
| CONCLUSION.DIRECTION | 不涉及（枚举比较） | `system_direction == gt_direction` |
| CONCLUSION.COVERAGE | 不涉及（集合比较） | `required_ids ⊆ covers_union` |
| EXPECTED.HIT | 不涉及（字符串匹配） | one-to-one composite key 精确匹配 |

### Fix 8 结论

TemplateSpec.fact_display_spec 的渲染精度合规性**不属于 P0 Accuracy Validator 的职责**，属于未来 P1/P2 模板合规检查。Accuracy Validator 只负责 canonical 值与 G3.precision 控制的 round 比较。

---

## §4 TrustedInput（修订后）

### TrustedInput（全部外部加载）

```python
@dataclass
class TrustedInput:
    """Accuracy Validator 的受信输入集合。

    全部来自 EvaluationCase 固件 + 外部配置。
    不包含任何来自被测系统的字段。
    """
    case: EvaluationCase
    expected_evaluations: list[ExpectedEvaluationSnapshot]  # 🔴 Fix 6：直接提供 expected status
    confirmed_criteria: list[CriterionSnapshot]            # 外部加载
    whitelist: WhitelistConfig                              # 外部加载
    fact_display_spec: FactDisplaySpecSnapshot              # 外部加载（D-025）
    trusted_table_specs: list[TrustedTableSpec]            # 外部加载
```

### §4.1 ExpectedEvaluationSnapshot（Fix 6：Validator 退化为纯比较器）

```python
class ExpectedEvaluationSnapshot(TypedDict):
    """Fix 6：TrustedInput 直接携带 expected evaluation status。

    由评测用例构造器在外部预先计算（confirmed_criteria + G3 facts → expected_status）。
    Accuracy Validator 只做字符串精确比较，不执行任何 rule evaluation，不膨胀成规则引擎。
    """
    evaluation_id: str
    criterion_id: str
    fact_id: str
    expected_status: str                    # 冻结的 expected status（如 "qualified" / "unqualified"）
```

**Fix 6 关键决策**：Accuracy Validator 不再包含任何"按 rule 计算布尔值 → 查 status_mapping → 得到 expected_status"的逻辑。这些逻辑在评测用例构造期由外部执行，TrustedInput 只给最终的 `expected_status` 字符串。

好处：
- Validator 退化为纯比较器，零业务语义解释
- 消除 Validator 和 System 对同一条 rule 的不同解释风险
- 消除未来"rule grammar 扩展"变成通用规则引擎的可能

### §4.2 CriterionSnapshot（保留但缩减）

```python
class CriterionSnapshot(TypedDict):
    """外部加载的 Criterion 信息（Fix 6 后不再需要 status_mapping —— 已由评测用例构造器消费）。

    Accuracy Validator 只读这些字段用于定位和诊断（Issue 消息中引用）。
    """
    criterion_id: str
    applicable_fact_ids: list[str]
    rule: str                               # "value <= limit"（诊断用）
    limit_value: Any                        # 诊断用
    limit_unit: Optional[str]
    limit_quantity_kind: Optional[str]
    # 🔴 status_mapping 字段已移除（Fix 6）—— 不在 Validator 中做映射
```

### §4.3 WhitelistConfig（不变）

```python
@dataclass
class WhitelistConfig:
    version: str
    entries: list[WhitelistEntry]

@dataclass
class WhitelistEntry:
    pattern: str
    reason: str
    scope: str
    confirmed_by: str
```

### §4.4 TrustedTableSpec（Fix 5：explicit source cells）

```python
@dataclass
class TableCellRef:
    row: int
    col: int

@dataclass
class TrustedAggregateDef:
    """Fix 5：explicit source_cells，不用 row_range。

    必须精确声明每个 aggregate cell 的坐标、类型、以及它依赖的 data cells 坐标列表。
    """
    row: int                                # aggregate cell 所在行
    col: int                                # aggregate cell 所在列
    kind: str                               # "sum" | "mean" | "min" | "max" | "count"
    source_cells: list[TableCellRef]        # Fix 5：精确 source cells 列表，无 wildcards

@dataclass
class TrustedTableSpec:
    table_id: str
    aggregates: list[TrustedAggregateDef]
    data_cells: list[TableCellRef]          # 完整明细 cells 列表（可用于 TABLE.COUNT_MISMATCH）
```

**Fix 5 Validator 独立计算流程**：
```text
对每个 TrustedAggregateDef:
    1. 从 SystemOutput.structured_tables 按 (row, col) 取 aggregate cell 的 numeric_value → system_aggregate
    2. 按 TrustedAggregateDef.source_cells 坐标列表取所有 data cells 的 numeric_values → data_values
    3. Validator 独立计算:
       a. if len(data_values) == 0 → TABLE.DATA_MISSING (FAIL)
       b. 计算 recomputed = compute(kind, data_values)
       c. 先 relative error 检查：abs(recomputed - system_aggregate) / max(abs(recomputed), 1e-12) ≤ 1e-9  （05_EVAL §6.1 要求）
       d. 再 G3.precision round 检查：round(recomputed, p) == round(system_aggregate, p)
       e. 任一检查未通过 → TABLE.{KIND}_MISMATCH (FAIL)
```

---

## §5 SystemOutput（不变，与 TrustedTableSpec 交叉引用）

```python
@dataclass
class SystemOutput:
    report_ir: IReportIRSnapshot
    rendered_segments: list[RenderedSegment]
    structured_tables: list[StructuredTable]   # 按 table_id 匹配 TrustedTableSpec
    system_reported_issues: list[SystemReportedIssue]
    system_evaluations: list[IEvaluationSnapshot]
```

### Anchor 契约（Fix 2 + Fix 4 + Fix 3 配套）

```python
@dataclass
class RenderedSegment:
    segment_location: str
    segment_type: str
    raw_text: str
    anchor_declarations: list[AnchorDeclaration]

@dataclass
class AnchorDeclaration:
    """Fix 2：精确引用单个 TokenOccurrence span。"""
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

### StructuredTable（System 产出，按 TrustedTableSpec 交叉验证）

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

## §6 ActualIssue（Fix 7：canonical identity tuple）

```python
@dataclass(frozen=True)
class ActualIssue:
    """Accuracy Validator 独立产生的问题（Fix 7：frozen tuple identity）。"""
    # ── Canonical Identity（Fix 7：frozen tuple + stable hash） ──
    identity: tuple                   # 不可变元组，精确表达问题身份
    issue_id: str                      # hash(identity) 的稳定表示（如 base64(hash(tuple))）

    # ── 来源定位 ──
    rule_id: str
    severity: str
    issue_type: str
    fact_id: Optional[str] = None
    evaluation_id: Optional[str] = None
    location: Optional[str] = None
    token_identity: Optional[TokenOccurrence] = None

    # ── 问题描述 ──
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
        """Fix 7：canonical identity tuple，满足 7 种场景不碰撞。

        identity tuple 组成：
            (rule_id, issue_type, location, fact_id, evaluation_id, occurrence_id, table_aggregate)
        全部 Optional 字段用 "" 占位，保证 tuple 长度固定。
        """
        return (
            rule_id,
            issue_type,
            location,
            fact_id or "",
            evaluation_id or "",
            occurrence_id or "",
            table_aggregate or "",
        )
```

### Fix 7 各场景 identity tuple 示例

| Issue 类型 | identity tuple 示例 |
|---|---|
| ANCHOR.UNBOUND 同一 token 两次出现 | `("ANCHOR.BINDING", "ANCHOR.UNBOUND", "semantic:x/block-y", "", "", "loc@12-17", "")` vs `("ANCHOR.BINDING", "ANCHOR.UNBOUND", "semantic:x/block-y", "", "", "loc@45-50", "")` —— **occurrence_id 不同 → 不碰撞** |
| TABLE.SUM_MISMATCH 两个不同 aggregate cell | `("TABLE.INTERNAL", "TABLE.SUM_MISMATCH", "table-t001", "", "", "", "r5c8")` vs `("TABLE.INTERNAL", "TABLE.SUM_MISMATCH", "table-t001", "", "", "", "r7c3")` —— **table_aggregate 不同 → 不碰撞** |
| CONCLUSION.COVERAGE_GAP 两个不同未覆盖 evaluation_id | `("CONCLUSION.COVERAGE", "CONCLUSION.COVERAGE_GAP", "semantic:conclusion/block-c1", "", "eval-001", "", "")` vs `("CONCLUSION.COVERAGE", "CONCLUSION.COVERAGE_GAP", "semantic:conclusion/block-c1", "", "eval-002", "", "")` —— **evaluation_id 不同 → 不碰撞** |
| VALUE.MISMATCH 同一 fact 的 UNIT.MISMATCH vs VALUE.MISMATCH | `("VALUE.CONSISTENCY", "UNIT.MISMATCH", "", "fact:component.K001.length", "", "", "")` vs `("VALUE.CONSISTENCY", "VALUE.MISMATCH", "", "fact:component.K001.length", "", "", "")` —— **issue_type 不同 → 不碰撞** |

---

## §7 expected_issues 匹配（Fix 3 + Fix 4）

### Fix 3：禁止内部去重，入口检测 duplicate → precondition_error

```python
# AccuracyValidator.validate() 入口（在 case.validate() 之后）
def _check_expected_issues_not_duplicate(self, case: EvaluationCase) -> Optional[str]:
    seen = set()
    for ei in case.expected_issues:
        key = (ei.rule_id, ei.fact_id)
        if key in seen:
            return f"Duplicate ExpectedIssue: rule_id={ei.rule_id}, fact_id={ei.fact_id}"
        seen.add(key)
    return None
```

**证据**：Step 7-D `validate_expected_issues()`（validator.py:287-292）仅对每条 ExpectedIssue 做字段级校验，**不检查 duplicate**。因此 Accuracy Validator 必须在入口自己检测。

### Fix 4：fact_id=None 语义冻结 + one-to-one matching

```text
输入（AccuracyValidator 收到的 expected_issues 已通过 duplicate 检测，无重复）:
    expected_issues: list[ExpectedIssue]       # 原样来自 case.expected_issues，不去重
    system_reported_issues: list[SystemReportedIssue]

匹配规则（one-to-one，确定性，无模糊）:

1. 按 ExpectedIssue 自身条件分类匹配：
   a. ExpectedIssue.fact_id is not None:
      在 system_reported_issues 中找 rule_id 精确相等 AND fact_id 精确相等 的条目
      （fact_id 必须精确匹配——不能 wildcard 替代 exact fact_id）
   
   b. ExpectedIssue.fact_id is None:
      在 system_reported_issues 中找 rule_id 精确相等 的条目
      （rule-level wildcard——但只能消耗一条 SystemReportedIssue）

2. one-to-one 约束：
   - 一条 SystemReportedIssue 最多只能消耗一条 ExpectedIssue
   - 遍历顺序：按 expected_issues 原始顺序，逐条找未被消耗的 SystemReportedIssue

3. EXPECTED.HIT RuleResult:
   - 全部命中 → PASS
   - 存在未命中 → FAIL（每条未命中的 ExpectedIssue 产生一个 ActualIssue）
   - expected_issues=0 → NOT_EVALUABLE（不是隐式 PASS）
```

### 为什么这样设计

| 约束 | 推导 |
|---|---|
| Fix 3 明确禁止内部去重 | 不修改 EvaluationCase 的原始基数——评测用例声明了几条 ExpectedIssue 就必须命中几条 |
| Fix 4 fact_id=None → rule-level wildcard | ExpectedIssue 明确不指定 fact_id 时，就是只要求该 rule_id 报错即可 |
| Fix 4 fact_id!=None → exact fact_id | 防作弊：指定 fact_id 的 ExpectedIssue 必须精确命中 |
| one-to-one | 一条 SystemReportedIssue 不能覆盖多条 ExpectedIssue；反之亦然 |

---

## §8 SystemReportedIssue（不变）

```python
@dataclass
class SystemReportedIssue:
    rule_id: str                              # 生成系统自己的规则域，与 P0_RULE_IDS 独立
    fact_id: Optional[str] = None
    evaluation_id: Optional[str] = None
    issue_type: str = ""
    severity: str = "error"
    message: str = ""
```

---

## §9 known_issues 边界（不变）

Accuracy Validator 对 known_issues 的处理：**只读、不参与任何判定**。Step 7-D 已完成 known_issues 结构正确性检查。本 Validator 仅记录诊断信息。

---

## §10 ValidationResult（不变）

```python
@dataclass
class ValidationResult:
    case_id: str
    case_level: str
    passed: bool                              # 无 RuleResult.status == FAIL；NOT_EVALUABLE 不计入 FAIL
    rule_results: list[RuleResult]            # 固定 7 条（P0_RULE_IDS 冻结）
    actual_issues: list[ActualIssue]          # FAIL 的 RuleResult 产生
    expected_coverage: ExpectedIssueCoverage
    summary: ValidationSummary
    precondition_error: Optional[str] = None  # 非 None → passed=False, rule_results=[]
```

### RuleResult

```python
@dataclass
class RuleResult:
    rule_id: str
    status: RuleStatus                       # PASS / FAIL / NOT_EVALUABLE
    issues_count: int                        # 仅 FAIL 时有值
    issues: list[ActualIssue]                # 仅 FAIL 时有值
    not_evaluable_reason: Optional[str] = None  # 仅 NOT_EVALUABLE 时有值
```

### ExpectedIssueCoverage

```python
@dataclass
class ExpectedIssueCoverage:
    expected_total: int                       # 原样来自 case.expected_issues（Fix 3：无内部去重）
    expected_hit: int
    expected_miss: int
    hit_rate: float                           # expected_total=0 时 → 0.0（NOT_EVALUABLE 场景）
    unmatched: list[ExpectedIssue]
```

### ValidationSummary

```python
@dataclass
class ValidationSummary:
    total_actual_issues: int
    severity_error_count: int
    severity_warning_count: int
    rules_passed: int
    rules_failed: int
    rules_not_evaluable: int
```

---

## §11 TypedDict Protocol

```python
from typing import TypedDict, List, Optional, Any

# Report IR
class ConclusionBlockSnapshot(TypedDict):
    semantic: str                                       # "conclusion"
    direction: str
    covers: List[str]

class RefOccurrence(TypedDict):
    fact_id: str
    value: Any                                          # canonical Fact.value
    unit: Optional[str]

class IReportIRSnapshot(TypedDict):
    conclusion_blocks: List[ConclusionBlockSnapshot]
    ref_occurrences: List[RefOccurrence]

# Expected Evaluation（Fix 6：外部预计算，Validator 退化为纯比较器）
class ExpectedEvaluationSnapshot(TypedDict):
    evaluation_id: str
    criterion_id: str
    fact_id: str
    expected_status: str                                # 冻结字符串

# System Evaluation（被测系统产出）
class IEvaluationSnapshot(TypedDict):
    evaluation_id: str
    criterion_id: str
    fact_id: str
    status: str

# GroundTruth Fact（直接从 EvaluationCase 读）
# Step 7-D GroundTruthFact: fact_id, value, unit, quantity_kind, precision
```

---

## §12 外部配置文件约定

```text
eval_config/
├── whitelist.json          # 版本化白名单
├── criteria.json           # confirmed_criteria 列表（Fix 6：评测用例构造器已据此预计算 ExpectedEvaluationSnapshot）
├── expected_evaluations.json  # Fix 6：外部预计算的 expected evaluations（或直接嵌入 TrustedInput）
└── table_specs.json        # trusted_table_specs 列表
```

加载时机：AccuracyValidator 构造函数中读取，注入 TrustedInput。

---

## §13 模块文件结构

```text
src/eval/
├── __init__.py                # 追加导出
├── types.py                   # Step 7-D v4 冻结，不动
├── enums.py                   # Step 7-D 冻结，不动
├── validator.py               # Step 7-D 冻结，不动
│
├── contracts.py               # 新增
│   ├── P0_RULE_IDS: frozenset[str]      # 7 条，冻结
│   ├── RuleStatus (enum)                  # PASS / FAIL / NOT_EVALUABLE
│   ├── TrustedInput (dataclass)
│   ├── WhitelistConfig + WhitelistEntry
│   ├── FactDisplaySpecEntry
│   └── TypedDict Protocols (§11)
│
├── actual_issues.py           # 新增
│   ├── ActualIssue (frozen dataclass, Fix 7 identity tuple)
│   ├── ActualIssue.build_identity()
│   └── ISSUE_TYPES (frozenset)
│
├── results.py                 # 新增
│   ├── RuleResult (dataclass, 三态 status)
│   ├── ExpectedIssueCoverage (dataclass, Fix 3 不做去重)
│   ├── ValidationSummary (dataclass)
│   └── ValidationResult (dataclass)
│
├── inputs.py                  # 新增
│   ├── SystemOutput (dataclass)
│   ├── RenderedSegment + AnchorDeclaration + TokenOccurrence
│   ├── StructuredTable + TableCell
│   ├── TrustedTableSpec + TrustedAggregateDef + TableCellRef
│   └── SystemReportedIssue (dataclass)
│
└── accuracy_validator.py      # 新增，核心实现
    ├── AccuracyValidator (class)
    ├── validate(case, system_output, trusted_input) -> ValidationResult
    ├── _check_preconditions(case, system_output, trusted_input) -> Optional[str]  # case.validate() + ExpectedIssue duplicate
    └── _run_rule(rule_id, ...) -> RuleResult   # 每条 Rule 按 §2 决策矩阵执行
```

---

## §14 明确不做

| 编号 | 不做 | 原因 |
|---|---|---|
| M1 | 不修改 `types.py` / `enums.py` / `validator.py` | Step 7-D 冻结 |
| M2 | 不实现 parser / renderer | 只定义 SystemOutput 契约 |
| M3 | 不 import 未来 Step 7-F class | 用 TypedDict |
| M4 | known_issues 不参与任何判定 | 诊断参考 |
| M5 | whitelist 放进 EvaluationCase | 版本化外部配置 |
| M6 | 不新增 P0 Rule | 7 条冻结 |
| M7 | 不判断构建门禁 / gold_count_total | M7 eval runner 责任 |
| M8 | 不执行 rule evaluation | Fix 6：TrustedInput 给 ExpectedEvaluationSnapshot，Validator 退化为纯比较器 |
| M9 | 不做 LLM 语义相似度 | 枚举精确相等 |
| M10 | 不处理 Template display precision 合规 | Fix 8：属于 P1/P2，不属于 P0 |
| M11 | 不内部 deduplicate ExpectedIssue | Fix 3：不修改原始基数 |
| M12 | 不推导 rule grammar | Fix 6：不在 Validator 内做 rule evaluation |

---

## §15 风险审查

| 编号 | 风险 | 缓解 |
|---|---|---|
| R1 | ExpectedEvaluationSnapshot 暂无现成 gold/silver 用例支撑 | silver/bronze 自动 NOT_EVALUABLE；gold 必须有否则 precondition_error |
| R2 | TrustedAggregateDef 的 source_cells 配置工作量 | 先用最简单的表配置（每个表 aggregate 数有限）；配置错误在 precondition 阶段发现 |
| R3 | AnchorDeclaration 与 TokenOccurrence span 不匹配 | 两者使用相同 segment_location 和 raw_text 提取，构造时交叉验证 |
| R4 | one-to-one matching 顺序敏感 | Fix 4 规定按 expected_issues 原始顺序匹配，确定性可复现 |
| R5 | bronze ANCHOR.BINDING NOT_EVALUABLE 掩盖渲染质量 | bronze 本就不测数值——渲染质量是模板/IR 层问题，在 IR Validator 中独立检查 |

---

## §16 测试契约

### 7 Rule × 6 Condition × 3 case_level 完整断言（Fix 9）

| Rule | Condition | gold 断言 | silver 断言 | bronze 断言 |
|---|---|---|---|---|
| VALUE.CONSISTENCY | G3=0 | precondition_error | NOT_EVALUABLE | NOT_EVALUABLE |
|  | 无 ref_occurrences | FAIL | FAIL | FAIL |
|  | 全部 round 值/单位/量纲一致 | PASS | PASS | — |
|  | 任一 mismatch | FAIL | FAIL | — |
| TABLE.INTERNAL | trusted_table_specs=0 | precondition_error | NOT_EVALUABLE | NOT_EVALUABLE |
|  | TrustedTableSpec 有但 System 无匹配 table_id | FAIL | FAIL | — |
|  | 独立重算一致 | PASS | PASS | — |
|  | 独立重算任一 mismatch | FAIL | FAIL | — |
| LOGIC.CONSISTENCY | expected_evaluations=0 | precondition_error | NOT_EVALUABLE | NOT_EVALUABLE |
|  | system_evaluations=0 但 trusted 有 | FAIL | FAIL | — |
|  | 全部 status 精确相等 | PASS | PASS | — |
|  | 任一不等 | FAIL | FAIL | — |
| ANCHOR.BINDING | G3=0 | precondition_error | NOT_EVALUABLE | **NOT_EVALUABLE** 🔴Fix2 |
|  | rendered_segments=0 | FAIL | FAIL | FAIL |
|  | 全部 token 命中 whitelist 或正确锚定 | PASS | PASS | — |
|  | 任一 token 未命中 | FAIL | FAIL | — |
| CONCLUSION.DIRECTION | G4.direction=None | precondition_error | NOT_EVALUABLE | NOT_EVALUABLE |
|  | system_direction == gt_direction | PASS | PASS | — |
|  | 枚举不等 | FAIL | FAIL | — |
| CONCLUSION.COVERAGE | required_evaluation_ids=0 | precondition_error | NOT_EVALUABLE | NOT_EVALUABLE |
|  | required_ids ⊆ covers_union | PASS | PASS | — |
|  | 存在 gap 或 covers[] 缺失 | FAIL | FAIL | — |
| EXPECTED.HIT | expected_issues=0 | NOT_EVALUABLE | NOT_EVALUABLE | NOT_EVALUABLE |
|  | 重复 ExpectedIssue | precondition_error | precondition_error | precondition_error |
|  | system_reported_issues=0 但 expected_issues>0 | FAIL | FAIL | — |
|  | 全部命中 | PASS | PASS | — |
|  | 存在未命中 | FAIL | FAIL | — |

### Fix 7 identity 断言

```python
def test_actual_issue_identity_no_collision():
    # 同一 fact 的不同 issue_type 必须不同 identity
    a = ActualIssue.build_identity("VALUE.CONSISTENCY", "UNIT.MISMATCH", "", "fact:xxx.xxx")
    b = ActualIssue.build_identity("VALUE.CONSISTENCY", "VALUE.MISMATCH", "", "fact:xxx.xxx")
    assert a != b

    # 同一 token 两次出现必须不同 identity
    c = ActualIssue.build_identity("ANCHOR.BINDING", "ANCHOR.UNBOUND", "loc", occurrence_id="seg@12-17")
    d = ActualIssue.build_identity("ANCHOR.BINDING", "ANCHOR.UNBOUND", "loc", occurrence_id="seg@45-50")
    assert c != d

    # 不同 aggregate cell 必须不同 identity
    e = ActualIssue.build_identity("TABLE.INTERNAL", "TABLE.SUM_MISMATCH", "t001", table_aggregate="r5c8")
    f = ActualIssue.build_identity("TABLE.INTERNAL", "TABLE.SUM_MISMATCH", "t001", table_aggregate="r7c3")
    assert e != f
```

### Fix 3 duplicate ExpectedIssue 断言

```python
def test_duplicate_expected_issues_precondition_error():
    case = EvaluationCase(expected_issues=[
        ExpectedIssue(rule_id="FACT.MISSING", severity="error", fact_id="fact:component.K001.length"),
        ExpectedIssue(rule_id="FACT.MISSING", severity="error", fact_id="fact:component.K001.length"),  # 重复
    ])
    result = validator.validate(case, system_output, trusted_input)
    assert result.precondition_error is not None
    assert "Duplicate ExpectedIssue" in result.precondition_error
```

---

## READY 判定

逐条检查：

| 检查项 | 结论 |
|---|---|
| 7 Rule × 6 Condition × 3 case_level 决策矩阵全覆盖，每格唯一 bucket | ✅ |
| Fix 1: Rule Applicability 正式定义（PASS/FAIL/NOT_EVALUABLE/precondition_error） | ✅ |
| Fix 2: ANCHOR.BINDING bronze 行为 = NOT_EVALUABLE，不因 G3=0 报 FAIL | ✅ |
| Fix 3: 禁止内部去重，入口检测 duplicate → precondition_error（证据：validator.py:287-292 不做检查） | ✅ |
| Fix 4: fact_id=None 语义冻结（rule-level wildcard）；one-to-one matching | ✅ |
| Fix 5: TrustedAggregateDef 用 explicit source_cells: list[TableCellRef]；relative error ≤ 1e-9 → G3.precision round | ✅ |
| Fix 6: TrustedInput 提供 ExpectedEvaluationSnapshot[]，Validator 退化为纯比较器；CriterionSnapshot.status_mapping 字段已移除 | ✅ |
| Fix 7: ActualIssue.identity = frozen tuple（rule_id, issue_type, location, fact_id, evaluation_id, occurrence_id, table_aggregate）；7 种场景不碰撞 | ✅ |
| Fix 8: Precision 四者分层明确；Template display precision 合规不属于 P0 | ✅ |
| Fix 9: 每条 Rule × 6 Condition × 3 case_level 测试断言全覆盖 | ✅ |
| NOT_EVALUABLE 不会被隐式算成 PASS | ✅（决策矩阵无空 cell） |
| P0_RULE_IDS 仍然只有 7 条 | ✅ |
| 不引入新的 P0 Rule | ✅ |
| 不修改 Step 7-D 已冻结的数据结构 | ✅ |
| gold 不会通过缺失 TrustedInput 绕过 P0（→ precondition_error） | ✅ |
| silver 不会因未覆盖维度被误判 | ✅（NOT_EVALUABLE 合法豁免） |
| bronze 不会因缺 G3 被错误判 P0 FAIL | ✅ |
| 全部断言可直接写成测试 | ✅ |

---

**READY**
