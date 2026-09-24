*正在请求专家建议，优化解决方案，该过程耗时可能较长，请耐心等待...*

让我重新确认 05_EVALUATION.md 中关于 gold/silver/bronze 对 P0 门禁的原文：
用户明确指出了 v2 的 6 个剩余契约问题，需要逐一解决：

1. RuleResult 三态：PASS / FAIL / NOT_EVALUABLE
2. TABLE.INTERNAL 需要 Trusted TableSchema
3. AnchorDeclaration 必须精确引用 TokenOccurrence 的 span
4. ActualIssue 的唯一标识需要更强确定性
5. LOGIC.CONSISTENCY 的 status mapping 不能硬编码
6. 逐条定义每条 Rule 的 NOT_EVALUABLE 条件

让我先调用 Advisor 获取战略性指导，然后输出 v3。

我需要仔细重新阅读一些关键设计文档来回答这些问题，特别是：
- 05_EVALUATION.md 关于 gold/silver/bronze 的定义
- 05_EVALUATION.md 关于 P0 判定失败 vs 不可评估
- 03_CANONICAL_DATA_MODEL.md 关于 Evaluation 的 status 域
- 07_DECISIONS.md 关于已知的设计决策

让我先确认现有代码中 Evaluation 的 status 实际是什么样的，然后检查 RuleResult 的三态设计是否与 05_EVALUATION.md 兼容。
正在请求专家建议.......

已获取专家建议

好，Advisor 的指导非常精准。让我逐一落实 6 个修订点，然后输出 v3。

让我先快速重新确认一下 05_EVALUATION.md 中关于 gold/silver/bronze 对 P0 门禁的具体规定——特别是 NOT_EVALUABLE 在不同 case_level 下的行为边界。然后写 v3。

好，05_EVALUATION.md 明确了：
- gold：G1+G3+G4 齐全，承担门禁阈值
- silver：有部分 G1，G3 可从该部分推导，用于趋势监控
- bronze：仅有报告（G2+G4），用于趋势监控
- 门禁阈值只由 gold 用例承担；silver/bronze 不单独触发构建失败

现在让我继续看 §7 和 §9 的门禁具体规定：

好，现在我对 05_EVALUATION.md 关于门禁的理解更清晰了：

关键原文：
- "门禁阈值只由 gold 用例承担" —— silver/bronze 不触发构建失败
- P0 门禁规则：gold 用例必须 100%，否则构建失败
- 经济门禁：gold < 5 时，降级为"全量报错但不阻断构建"
- silver 只有"部分 G3"，P0 只有部分字段可测
- bronze 没有 G3

现在我需要把这些转换成 RuleResult 三态的设计：

**NOT_EVALUABLE 的触发条件**（一条 Rule 不能评估）：
- VALUE.CONSISTENCY：TrustedInput 中没有 G3 facts（bronze case）→ NOT_EVALUABLE
- TABLE.INTERNAL：没有 trusted table schema → NOT_EVALUABLE
- LOGIC.CONSISTENCY：没有 confirmed_criteria（没有受信答案）→ NOT_EVALUABLE
- ANCHOR.BINDING：如果没有 rendered_segments（渲染产物缺失）→ 可能是 System 端问题 → FAIL？还是 NOT_EVALUABLE？
- CONCLUSION.DIRECTION：没有 G4 direction（bronze case）→ NOT_EVALUABLE
- CONCLUSION.COVERAGE：没有 G4 required_evaluation_ids（bronze case）→ NOT_EVALUABLE
- EXPECTED.HIT：没有 expected_issues → NOT_EVALUABLE（不是 PASS！）

但有一个区分：
- **Trusted Answer 缺失**（G3 为空、G4 为空、confirmed_criteria 为空）→ NOT_EVALUABLE（case_level 决定了用例就没这个维度）
- **System Output 缺失**（rendered_segments 为空、system_evaluations 为空）→ 这是 System 问题 → FAIL（因为应该有 System Output 但没给出来）
- **System Output 存在但格式异常**（rendered_segments 有但 span 冲突）→ 可能 FAIL 或 precondition_error

对于 gold case：
- 如果 gold case 上某条 Rule 是 NOT_EVALUABLE，那应该是 precondition_error（gold 应该有全部 G3/G4/confirmed_criteria，如果没有说明评测用例构造出错）
- 但 Advisor 提醒了这一点——需要区分"这个 case type 确实没有这个维度"和"应该有但评测用例构造漏了"

让我把这些理清楚，然后输出 v3 Contract。

另外还有几个关键点：
1. TrustedTableSpec：必须从外部配置加载，Validator 用 trusted spec 中的 aggregate_kind 和 data cell 列表独立重算
2. AnchorDeclaration：必须引用 TokenOccurrence 的精确 span（segment_location + char_start + char_end），不能用 token_text 匹配
3. ActualIssue issue_id：用 rule_id + issue_type + location + (fact_id 或 evaluation_id 或 token span) 组成确定性复合键
4. CriterionSnapshot.status_mapping：{ "true": "qualified", "false": "unqualified" } 这样的冻结映射
5. 每条 Rule 的 NOT_EVALUABLE 触发条件表

现在开始写 v3。
# Step 7-E Accuracy Validator — Coding Contract v3

---

## 修订基线（对应 6 条定点修订）

| 编号 | v2 问题 | v3 修正 |
|---|---|---|
| 1 | RuleResult 单纯 passed: bool 无法区分"通过"和"无可信答案" | 三态 RuleStatus + 传播表 |
| 2 | StructuredTable 的 role 仍由 System 声明，不可信 | 新增 TrustedTableSpec（外部加载），Validator 独立重算 |
| 3 | AnchorDeclaration 用 token_text 模糊匹配 occurrence | AnchorDeclaration 精确引用 (segment_location, char_start, char_end) |
| 4 | issue_key 无 char span 和 issue_type 参与 | issue_id 确定性复合键：rule_id:issue_type:location:identity |
| 5 | LOGIC.CONSISTENCY 硬编码 qualified/pass | CriterionSnapshot 携带 frozen status_mapping dict |
| 6 | expected_total=0 隐式 PASS | 全部 7 条 Rule 逐条定义 PASS/FAIL/NOT_EVALUABLE 触发条件 |

---

## §1 RuleStatus 三态与传播规则

### RuleStatus（枚举）

```python
class RuleStatus(str, Enum):
    PASS = "PASS"                       # 验证通过：所有可比较项一致
    FAIL = "FAIL"                       # 验证失败：可比较项存在不一致
    NOT_EVALUABLE = "NOT_EVALUABLE"     # 无可信答案或 System Output 缺失，无法评估
```

### ValidationResult.passed 的语义（严格按 05_EVALUATION.md §2 分级 + §7 门禁）

**前置条件**：`precondition_error != None → passed=False`（短路，不进规则执行）。

**规则结果聚合**：

| case_level | RuleStatus 聚合规则 | ValidationResult.passed=True 条件 |
|---|---|---|
| **gold** | 所有 7 条规则必须 **PASS**；任何一条 FAIL → 整体 FAIL；**NOT_EVALUABLE 在 gold 上不允许**（视为评测用例构造错误 → precondition_error） | 全部 7 条 RuleResult.status == PASS |
| **silver** | 有 Trusted Answer 的规则必须 PASS；无 Trusted Answer 的规则自动 NOT_EVALUABLE（silver 允许部分维度无 G3/G4） | 有 Trusted Answer 的规则全部 PASS（NOT_EVALUABLE 规则不计入） |
| **bronze** | 无 G3 的规则全部 NOT_EVALUABLE；有 G3 的规则必须 PASS | 有 Trusted Answer 的规则全部 PASS |

### 传播边界（关键区分）

| 场景 | RuleStatus | 理由 |
|---|---|---|
| gold case 有 confirmed_criteria 但 System Evaluations 为空 | **FAIL** | gold 必须有 System Output，缺失是被测系统的问题 |
| gold case confirmed_criteria 外部配置漏了 | **precondition_error** | gold 应具备全部受信答案，漏了是评测用例构造错误，不是 NOT_EVALUABLE |
| silver case confirmed_criteria 不存在 | **NOT_EVALUABLE** | silver 只测部分字段，这个维度本就无答案 |
| bronze case 无 G3 facts | **NOT_EVALUABLE** | bronze 只有 G2+G4，不测数值 |

---

## §2 RuleResult 三态

```python
@dataclass
class RuleResult:
    rule_id: str
    status: RuleStatus                      # PASS / FAIL / NOT_EVALUABLE
    issues_count: int                       # 仅 FAIL 时有值
    issues: list[ActualIssue]               # 仅 FAIL 时有值
    # NOT_EVALUABLE 时填充原因
    not_evaluable_reason: Optional[str] = None
```

---

## §3 TrustedInput 与 SystemOutput（修订 1 / 2 / 5）

### TrustedInput（全部外部加载，不可由被测系统提供）

```python
@dataclass
class TrustedInput:
    case: EvaluationCase
    confirmed_criteria: list[CriterionSnapshot]     # 外部加载
    whitelist: WhitelistConfig                       # 外部加载（§B.2）
    fact_display_spec: FactDisplaySpecSnapshot       # 外部加载（§B.3）
    trusted_table_specs: list[TrustedTableSpec]     # 外部加载（§6.2）
```

### §3.1 CriterionSnapshot（修订 5：status_mapping 冻结）

```python
class CriterionSnapshot(TypedDict):
    criterion_id: str
    applicable_fact_ids: list[str]
    rule: str                                 # "value <= limit" / "value >= limit" / ...
    limit_value: Any
    limit_unit: Optional[str]
    limit_quantity_kind: Optional[str]
    status_mapping: dict[str, str]             # 修订 5：{ "true": "qualified", "false": "unqualified" }
                                              # 真/假 → Evaluation.status 的冻结映射
```

**关键**：Accuracy Validator 不硬编码任何 `qualified/pass` 字符串。只做：
1. 按 `rule` 字段计算布尔结果 `bool_result`
2. 查 `status_mapping[str(bool_result)]` 得到 expected_status
3. 与 System Evaluation.status 字符串精确比较

### §3.2 WhitelistConfig（不变，版本化外部配置）

```python
@dataclass
class WhitelistConfig:
    version: str                            # "2026-09-01.v1"
    entries: list[WhitelistEntry]

@dataclass
class WhitelistEntry:
    pattern: str
    reason: str
    scope: str                              # "global" | "semantic:xxx"
    confirmed_by: str
```

### §3.3 TrustedTableSpec（修订 2：外部加载的可信表格 schema）

```python
@dataclass
class TrustedTableSpec:
    """外部加载的可信表格 schema（修订 2：Validator 独立计算 aggregate 的依据）。

    与 SystemOutput.structured_tables 按 table_id 配对。
    Validator 交叉引用 system cell 的 role 与 trusted spec 的 cell_role，
    独立计算 aggregate 值——不信任 System 声明的 aggregate 值。
    """
    table_id: str

    # 明细数据 cells（有 row/col 坐标的集合）
    data_cells: list[TableCellRef]

    # 每个 aggregate cell 的：坐标 + 计算类型 + 它依赖哪些 data cells
    aggregates: list[TrustedAggregateDef]

@dataclass
class TableCellRef:
    row: int
    col: int

@dataclass
class TrustedAggregateDef:
    row: int                                    # aggregate cell 所在行
    col: int                                    # aggregate cell 所在列
    kind: str                                   # "sum" | "mean" | "min" | "max" | "count"
    data_row_range: Optional[tuple[int, int]] = None   # 可选：全部 data cells 的行列范围（简化版）
    data_rows: Optional[list[int]] = None       # 可选：精确行列表（data_row_range 优先）
```

**关键**：Validator 的独立计算流程：
```text
对每个 TrustedAggregateDef:
    1. 从 SystemOutput.structured_tables 中按 (row, col) 取 aggregate cell 的 numeric_value
    2. 从 system_structured_table 中按 data_rows/data_row_range 取所有 data cells 的 numeric_values
    3. Validator 独立计算: compute(kind, data_values) → expected_aggregate
    4. round(expected_aggregate, gt.precision) == round(system_aggregate, gt.precision)?
       - 不等 → ActualIssue(TABLE.{KIND}_MISMATCH)
```

### §3.4 SystemOutput（不变，但补充 TrustedTableSpec 交叉引用说明）

```python
@dataclass
class SystemOutput:
    report_ir: IReportIRSnapshot
    rendered_segments: list[RenderedSegment]
    structured_tables: list[StructuredTable]   # 必须有 table_id 匹配 TrustedTableSpec
    system_reported_issues: list[SystemReportedIssue]
    system_evaluations: list[IEvaluationSnapshot]
```

---

## §4 Anchor 契约（修订 3：精确 span 引用）

### RenderedSegment（补充 TokenOccurrence 提取契约）

```python
@dataclass
class RenderedSegment:
    segment_location: str               # 唯一标识，如 "semantic:inspection_result/table-t001/cell-r3-c2"
    segment_type: str                   # narrative_prose | table_cell | table_title | table_note | figure_caption
    raw_text: str
    anchor_declarations: list[AnchorDeclaration]  # 修订 3：不再用 token_text 模糊匹配

@dataclass
class AnchorDeclaration:
    """修订 3：精确引用单个 TokenOccurrence，不再用 token_text 匹配多个 occurrence。

    锚定精确身份 = (segment_location, char_start, char_end)。
    """
    segment_location: str
    char_start: int
    char_end: int
    fact_id: str

    @property
    def occurrence_id(self) -> str:
        """稳定哈希：用于日志和 issue_id 组成部分。"""
        return f"{self.segment_location}@{self.char_start}-{self.char_end}"
```

### TokenOccurrence（修订 3：与 AnchorDeclaration 同构）

```python
@dataclass
class TokenOccurrence:
    segment_location: str
    char_start: int
    char_end: int
    token_text: str                     # "32.4MPa" 或 "K002"
    token_kind: str                     # numeric | identifier

    @property
    def occurrence_id(self) -> str:
        return f"{self.segment_location}@{self.char_start}-{self.char_end}"
```

### AnchorDeclaration 与 TokenOccurrence 的匹配方式

```text
tokenize(rendered_segment.raw_text) → TokenOccurrence[]     # 正则 + span 捕获
                                                              # 相同 token_text 出现两次 → 两个 TokenOccurrence，
                                                              # 靠 (segment_location, char_start, char_end) 区分

对每个 TokenOccurrence:
    找 AnchorDeclaration 中 (segment_location, char_start, char_end) 精确匹配
    - 没找到 → ActualIssue(ANCHOR.UNBOUND, token_identity=TokenOccurrence)
```

**修订 3 禁止的行为**：
- ❌ 用 token_text 在多个 AnchorDeclaration 中做模糊匹配
- ❌ 把 "32.4" 在同一段落出现两次时只声明一次 AnchorDeclaration
- ❌ 按 occurrence_id 的 hash 反推 AnchorDeclaration（必须精确坐标匹配）

---

## §5 ActualIssue（修订 4：确定性 issue_id）

```python
@dataclass
class ActualIssue:
    """Accuracy Validator 独立产生的问题。"""
    # ── 确定性唯一标识（修订 4） ──
    issue_id: str                       # f"{rule_id}:{issue_type}:{location}:{identity}"
                                        # identity = fact_id | evaluation_id | occurrence_id | ""

    # ── 来源定位 ──
    rule_id: str                        # P0_RULE_IDS 内
    severity: str                       # error | warning
    issue_type: str                     # VALUE.MISMATCH / ANCHOR.UNBOUND / ...

    # ── 维度定位 ──
    fact_id: Optional[str] = None
    evaluation_id: Optional[str] = None
    location: Optional[str] = None      # semantic path + block id
    token_identity: Optional[TokenOccurrence] = None  # ANCHOR.BINDING 专用

    # ── 问题描述 ──
    expected: Any = None
    actual: Any = None
    message: str = ""

    @staticmethod
    def make_issue_id(
        rule_id: str,
        issue_type: str,
        location: str = "",
        fact_id: Optional[str] = None,
        evaluation_id: Optional[str] = None,
        occurrence_id: Optional[str] = None,
    ) -> str:
        """确定性 issue_id 生成（修订 4）。

        保证同一报告中：
        - 相同 token 两次出现 → occurrence_id 不同 → issue_id 不同
        - 不同 issue_type 命中同一 fact_id → issue_type 不同 → issue_id 不同
        """
        identity = fact_id or evaluation_id or occurrence_id or ""
        return f"{rule_id}:{issue_type}:{location}:{identity}"
```

---

## §6 七条 P0 Rule 的 PASS/FAIL/NOT_EVALUABLE 完整触发条件（修订 6）

### §6.1 VALUE.CONSISTENCY

| 输入条件 | RuleStatus | 理由 |
|---|---|---|
| **Trusted Answer 缺失**：`len(G3.facts) == 0` | **NOT_EVALUABLE** | 无 G3 无可比对值；bronze case 必然触发 |
| **System Output 缺失**：System 无 report_ir Ref 段 | **FAIL** | 被测系统应产出引用但未产出（gold/silver 应有） |
| **所有 G3 fact_id 都有 Ref 且 round 后值+单位+量纲一致** | **PASS** | — |
| **任一 G3 fact_id 的 round 值/单位/量纲不一致** | **FAIL** | 产生 VALUE.MISMATCH / UNIT.MISMATCH / QUANTITY.MISMATCH |
| **G3 fact_id 存在但 System 完全没有此 Ref** | **FAIL** | 产生 VALUE.MISSING |
| **gold case 无 G3** | **precondition_error**（短路） | gold 应具备全部 G3，缺失是评测用例构造错误 |

### §6.2 TABLE.INTERNAL（修订 2）

| 输入条件 | RuleStatus | 理由 |
|---|---|---|
| **Trusted Answer 缺失**：`len(trusted_table_specs) == 0` | **NOT_EVALUABLE** | 无可信表格 schema，Validator 不知道哪些 cell 是 aggregate |
| **TrustedTableSpec 有但 System 无匹配 table_id 的 StructuredTable** | **FAIL** | 被测系统应产出此表但未产出 |
| **所有 TrustedAggregateDef 的独立重算值与 system_aggregate round 后一致** | **PASS** | — |
| **任一 aggregate 重算不一致** | **FAIL** | 产生 TABLE.{SUM/MEAN/MIN/MAX/COUNT}_MISMATCH |
| **TrustedTableSpec 的 aggregate 依赖的 data_rows 在 system table 中无数据** | **FAIL** | 产生 TABLE.DATA_MISSING |
| **gold case 有 StructuredTable 但无 TrustedTableSpec** | **precondition_error**（短路） | gold 应有完整可信 schema，漏了是评测用例构造错误 |

### §6.3 LOGIC.CONSISTENCY（修订 5）

| 输入条件 | RuleStatus | 理由 |
|---|---|---|
| **Trusted Answer 缺失**：`len(confirmed_criteria) == 0` | **NOT_EVALUABLE** | 无 confirmed_criteria 无可判定依据 |
| **System Output 缺失**：`len(system_evaluations) == 0` | **FAIL** | 被测系统应产出 Evaluation 但未产出（gold/silver） |
| **所有 confirmed_criteria 的 expected_status（Validator 独立计算）与 system_evaluation.status 精确相等** | **PASS** | — |
| **任一 confirmed_criteria 的 expected_status 与 system_status 不一致** | **FAIL** | 产生 LOGIC.MISMATCH |
| **gold case confirmed_criteria 为空** | **precondition_error**（短路） | gold 应有全部 confirmed_criteria，缺失是评测用例构造错误 |

**独立计算流程（修订 5）**：
```python
def _run_logic_consistency(self, criterion, gt_fact_value, system_evaluation):
    # 1. 按 rule 计算布尔值
    bool_result = self._eval_rule(criterion.rule, gt_fact_value, criterion.limit_value)
    #    例：rule="value <= limit", gt_fact_value=2.3, limit_value=5.0 → bool_result=True

    # 2. 查 frozen status_mapping（修订 5：不硬编码 qualified/pass）
    expected_status = criterion.status_mapping[str(bool_result)]
    #    例：status_mapping={"true": "qualified", "false": "unqualified"} → expected_status="qualified"

    # 3. 与 System Evaluation.status 精确比较
    return expected_status == system_evaluation["status"]
```

### §6.4 ANCHOR.BINDING（修订 3）

| 输入条件 | RuleStatus | 理由 |
|---|---|---|
| **System Output 缺失**：`len(rendered_segments) == 0` | **FAIL** | 渲染产物应存在但未产出（§41.2 R5 必须操作渲染产物） |
| **所有 TokenOccurrence 要么命中 whitelist 要么正确锚定 fact_id 且 round 值相等** | **PASS** | — |
| **任一 TokenOccurrence 未锚定且未命中 whitelist** | **FAIL** | 产生 ANCHOR.UNBOUND |
| **锚定存在但 fact_id 不在 G3** | **FAIL** | 产生 ANCHOR.UNKNOWN_FACT |
| **锚定存在且 fact_id 在 G3 但 round 值不等** | **FAIL** | 产生 ANCHOR.VALUE_MISMATCH |
| **锚定存在且 fact_id 在 G3 且值相等但 instance_id 不匹配（如 K002 token 锚到 fact:component.K003）** | **FAIL** | 产生 ANCHOR.WRONG_FACT |

**注意**：ANCHOR.BINDING 没有"Trusted Answer 缺失"的正常 NOT_EVALUABLE 情况——whitelist 是外部配置（永远加载），G3 facts 如果在 bronze case 为空则所有 token 会报 ANCHOR.UNKNOWN_FACT（FAIL，不是 NOT_EVALUABLE）。

### §6.5 CONCLUSION.DIRECTION

| 输入条件 | RuleStatus | 理由 |
|---|---|---|
| **Trusted Answer 缺失**：`G4.direction is None`（bronze case） | **NOT_EVALUABLE** | 无可信方向答案 |
| **System Output 缺失**：System report_ir 无 conclusion 段 | **FAIL** | 被测系统应产出结论但未产出 |
| **system_direction == gt_direction（枚举精确相等）** | **PASS** | — |
| **system_direction != gt_direction** | **FAIL** | 产生 CONCLUSION.DIRECTION_MISMATCH |
| **silver case G4 direction 来源于正文抄录**（05_EVAL §2 V2.1-7） | **NOT_EVALUABLE** | 降级处理，不是本 Validator 决定的 |
| **gold case G4.direction is None** | **precondition_error**（短路） | gold 应有完整 G4，缺失是评测用例构造错误 |

### §6.6 CONCLUSION.COVERAGE

| 输入条件 | RuleStatus | 理由 |
|---|---|---|
| **Trusted Answer 缺失**：`len(G4.required_evaluation_ids) == 0` | **NOT_EVALUABLE** | 无可信覆盖答案 |
| **System Output 缺失**：System 无 conclusion 段 | **FAIL** | — |
| **System 无 covers[] 字段** | **FAIL** | 产生 CONCLUSION.COVERS_MISSING |
| **`G4.required_ids ⊆ system.covers_union`** | **PASS** | — |
| **存在 required_ids - covers_union ≠ ∅** | **FAIL** | 每一个未覆盖 id 产生一个 CONCLUSION.COVERAGE_GAP |
| **gold case required_evaluation_ids 为空** | **precondition_error**（短路） | gold 应有完整 G4 |

### §6.7 EXPECTED.HIT

| 输入条件 | RuleStatus | 理由 |
|---|---|---|
| **Trusted Answer 缺失**：`len(expected_issues) == 0` | **NOT_EVALUABLE** | 修订 6：没有 expected_issues 无法评测命中率，**不是隐式 PASS** |
| **System Output 缺失**：`len(system_reported_issues) == 0` 但 expected_issues 非空 | **FAIL** | 被测系统应报错但一个都没报 |
| **所有去重后的 ExpectedIssue 都被一条 SystemReportedIssue 命中** | **PASS** | — |
| **存在 ExpectedIssue 未被命中** | **FAIL** | 每一条未命中产生 EXPECTED.NOT_HIT |

---

## §7 SystemReportedIssue（不变，修订 6 确认两域独立）

```python
@dataclass
class SystemReportedIssue:
    """生成管线自己产出的问题。

    P0_RULE_IDS（Accuracy Validator 的 7 条）与 SystemReportedIssue.rule_id（生成系统自己的规则域）
    是两个完全独立的集合。本 Validator 不做 rule_id 枚举检查，只做字符串匹配。
    """
    rule_id: str
    fact_id: Optional[str] = None
    evaluation_id: Optional[str] = None
    issue_type: str = ""
    severity: str = "error"
    message: str = ""
```

---

## §8 expected_issues 匹配（不变，修订 6 配套）

### 匹配规则（去重 + 精确键）

```text
1. 对 expected_issues 按 (rule_id, fact_id) 去重
   （避免多条 ExpectedIssue 完全相同导致同一个 SystemReportedIssue 被重复计数）

2. 对每条去重后的 ExpectedIssue:
   找 SystemReportedIssue 中:
   - rule_id 精确相等
   - fact_id 精确相等（ExpectedIssue.fact_id 非 None 时）
   - 或两边 fact_id 都为 None（两边同 None 时匹配）
   
   找到 → 命中；没找到 → 消耗一条 SystemReportedIssue（一 Issue 只消耗一条 ExpectedIssue）

3. EXPECTED.HIT RuleResult:
   expected_total == 0 → NOT_EVALUABLE（修订 6：无 expected_issues 无法评测，不是隐式 PASS）
   全部命中 → PASS
   有未命中 → FAIL
```

---

## §9 known_issues 边界（不变）

Accuracy Validator 对 known_issues 的处理：**只读、不参与任何 pass/fail/NOT_EVALUABLE 判定**。

Step 7-D `case.validate()` 已完成 known_issues 的结构检查。本 Validator 仅在 ValidationResult 中记录诊断信息（若 case.known_issues 非空），不影响规则执行。

---

## §10 ValidationResult（三态 + 修订 7）

```python
@dataclass
class ValidationResult:
    """单 Case 结果。不含构建门禁判断。"""
    case_id: str
    case_level: str

    # ── 三态聚合（核心修订 1） ──
    passed: bool                    # gold: 全部 PASS；silver: 有答案的全部 PASS；bronze: 有答案的全部 PASS
                                    # 等价于：没有 RuleResult.status == FAIL；NOT_EVALUABLE 不计入 FAIL

    # ── 规则逐条结果 ──
    rule_results: list[RuleResult]   # 固定 7 条（P0_RULE_IDS 冻结）

    # ── ActualIssues ──
    actual_issues: list[ActualIssue]  # FAIL 的 RuleResult 产生的 ActualIssue

    # ── expected_issues 覆盖矩阵 ──
    expected_coverage: ExpectedIssueCoverage

    # ── 汇总指标 ──
    summary: ValidationSummary

    # ── 执行阶段错误 ──
    precondition_error: Optional[str] = None  # 非 None → passed=False, rule_results=[]
```

### ExpectedIssueCoverage（修订 6 配套）

```python
@dataclass
class ExpectedIssueCoverage:
    expected_total: int                 # 去重后
    expected_hit: int
    expected_miss: int
    hit_rate: float                     # expected_total == 0 时 → 0.0（NOT_EVALUABLE 场景，不是隐式 1.0）
    unmatched: list[ExpectedIssue]
```

### ValidationSummary（修订 7：单 Case，无跨 Case 聚合）

```python
@dataclass
class ValidationSummary:
    total_actual_issues: int
    severity_error_count: int
    severity_warning_count: int
    rules_passed: int                   # RuleStatus.PASS 的数量
    rules_failed: int                   # RuleStatus.FAIL 的数量
    rules_not_evaluable: int            # RuleStatus.NOT_EVALUABLE 的数量
    # gold_count_total / build_gate_result 已删除（跨 Case 聚合）
```

---

## §11 TypedDict Protocol（修订 8，零 import 未来类）

```python
from typing import TypedDict, List, Optional, Any

# Report IR 快照（Step 7-F 填充）
class ConclusionBlockSnapshot(TypedDict):
    semantic: str                                    # "conclusion"
    direction: str
    covers: List[str]

class RefOccurrence(TypedDict):
    fact_id: str
    value: Any                                       # canonical Fact.value
    unit: Optional[str]

class IReportIRSnapshot(TypedDict):
    conclusion_blocks: List[ConclusionBlockSnapshot]
    ref_occurrences: List[RefOccurrence]

# Criterion 快照（外部加载）
class CriterionSnapshot(TypedDict):
    criterion_id: str
    applicable_fact_ids: List[str]
    rule: str
    limit_value: Any
    limit_unit: Optional[str]
    limit_quantity_kind: Optional[str]
    status_mapping: dict[str, str]                   # 修订 5

# Evaluation 快照（被测系统产出）
class IEvaluationSnapshot(TypedDict):
    evaluation_id: str
    criterion_id: str
    fact_id: str
    status: str                                      # qualified / unqualified / level_A / ...

# GroundTruth Fact 引用（从 EvaluationCase 直接读，无需额外 Protocol）
# Step 7-D GroundTruthFact frozen fields: fact_id, value, unit, quantity_kind, precision
```

---

## §12 外部配置文件约定

```text
eval_config/
├── whitelist.json          # §B.2 版本化白名单
├── criteria.json           # §B.1 confirmed_criteria 列表
└── table_specs.json        # §3.3 trusted_table_specs 列表
```

加载时机：AccuracyValidator 构造函数中读取，注入 TrustedInput。

---

## §13 模块文件结构

```text
src/eval/
├── __init__.py                # 追加导出（不改已有导出）
├── types.py                   # Step 7-D v4 冻结，不动
├── enums.py                   # Step 7-D 冻结，不动
├── validator.py               # Step 7-D 冻结，不动
│
├── contracts.py               # 新增
│   ├── P0_RULE_IDS: frozenset[str]         # 7 条，冻结
│   ├── RuleStatus (enum)                    # PASS / FAIL / NOT_EVALUABLE
│   ├── TrustedInput (dataclass)
│   ├── WhitelistConfig + WhitelistEntry
│   ├── FactDisplaySpecEntry
│   └── TypedDict Protocols (§11)
│
├── actual_issues.py           # 新增
│   ├── ActualIssue (dataclass)
│   ├── ActualIssue.make_issue_id()
│   └── ISSUE_TYPES (frozenset)
│
├── results.py                 # 新增
│   ├── RuleResult (dataclass, 三态 status)
│   ├── ExpectedIssueCoverage (dataclass)
│   ├── ValidationSummary (dataclass)
│   └── ValidationResult (dataclass)
│
├── inputs.py                  # 新增（合并 v2 的 contracts.py 中更偏输入的 dataclass）
│   ├── SystemOutput (dataclass)
│   ├── RenderedSegment + AnchorDeclaration + TokenOccurrence
│   ├── StructuredTable + TableCell
│   ├── TrustedTableSpec + TrustedAggregateDef + TableCellRef
│   └── SystemReportedIssue (dataclass)
│
└── accuracy_validator.py      # 新增，核心实现
    ├── AccuracyValidator (class)
    └── validate(case, system_output, trusted_input) -> ValidationResult
```

---

## §14 明确不做

| 编号 | 不做 | 原因 |
|---|---|---|
| M1 | 不修改 `types.py` / `enums.py` / `validator.py` | Step 7-D 冻结 |
| M2 | 不实现 parser / renderer | 只定义 SystemOutput 契约 |
| M3 | 不 import 未来 Step 7-F class | 用 TypedDict |
| M4 | known_issues 不参与 pass/fail/NOT_EVALUABLE | §G 原则 |
| M5 | whitelist 放进 EvaluationCase | §B.2 版本化外部配置 |
| M6 | 不新增 P0 Rule 处理前置条件失败 | 7 条冻结，前置失败走 error |
| M7 | 不判断构建门禁 / 不计算 gold_count_total | §§10 M7，跨 Case 聚合是 M7 eval runner 责任 |
| M8 | 不硬编码 qualified/pass | §6.3 status_mapping 冻结映射 |
| M9 | 不做 LLM 语义相似度 | CONCLUSION.DIRECTION 枚举精确相等 |
| M10 | 不实现模板渲染 | FactDisplaySpec 外部注入 |

---

## §15 风险审查

| 编号 | 风险 | 缓解 |
|---|---|---|
| R1 | confirmed_criteria 暂无现有评测用例支撑 | silver/bronze 自动 NOT_EVALUABLE；gold 必须有否则 precondition_error |
| R2 | TrustedTableSpec 的 data_rows 配置精度 | 先用 data_row_range（简化版），精细场景逐步补充 |
| R3 | TokenOccurrence 正则覆盖不全 | 冻结正则 `(\d+(\.\d+)?)\s*([a-zA-Zμ]+)?`；实例标识另行配置；覆盖率测试驱动 |
| R4 | AnchorDeclaration 与 TokenOccurrence span 不匹配 | Token 提取和 Anchor 提取使用同一个 render 后文本和相同正则；构造函数注入时交叉验证 |
| R5 | status_mapping 配置错误导致 LOGIC.CONSISTENCY 误判 | 外部评审确认 + 单测覆盖；配置错误在 precondition 阶段发现 |
| R6 | ExpectedIssueCoverage expected_total=0 时 hit_rate=0.0 | 明确这是 NOT_EVALUABLE 场景，不是隐式 PASS；与 RuleStatus 三态一致 |

---

## §16 测试契约

### 每条 Rule 的独立测试断言

| Rule | PASS 断言 | FAIL 断言 | NOT_EVALUABLE 断言 |
|---|---|---|---|
| VALUE.CONSISTENCY | 所有 G3 fact 都有 Ref + round 值/单位/量纲一致 | 任一 mismatch/missing | 无 G3 facts（bronze） |
| TABLE.INTERNAL | 所有 TrustedAggregateDef 独立重算与 system_aggregate round 后一致 | 任一 aggregate mismatch | 无 TrustedTableSpec |
| LOGIC.CONSISTENCY | 所有 confirmed_criteria 的 expected_status（Validator 独立计算 + status_mapping）与 system_status 精确相等 | 任一 mismatch | 无 confirmed_criteria |
| ANCHOR.BINDING | 所有 TokenOccurrence 要么命中 whitelist 要么正确锚定 fact_id + round 值相等 | 任一 unbound/unknown_fact/value_mismatch/wrong_fact | **无正常 NOT_EVALUABLE**（渲染产物缺失 → FAIL） |
| CONCLUSION.DIRECTION | system_direction == gt_direction 枚举精确相等 | 枚举不等 | G4.direction 为空（bronze） |
| CONCLUSION.COVERAGE | required_ids ⊆ covers_union | 存在未覆盖 id 或 covers[] 缺失 | required_evaluation_ids 为空 |
| EXPECTED.HIT | 所有去重 ExpectedIssue 被命中 | 存在未命中 | expected_issues=[] |

### case_level 三态传播测试

| case_level | 场景 | passed 预期 |
|---|---|---|
| gold | 所有 RuleStatus=PASS | True |
| gold | 有一条 FAIL | False |
| gold | confirmed_criteria 为空 | precondition_error 短路 |
| silver | confirmed_criteria 为空（NOT_EVALUABLE），有 G3 的规则 PASS | True |
| bronze | G3 为空（VALUE.CONSISTENCY NOT_EVALUABLE） | True（有答案的规则 PASS） |

### Anchor span 精确性测试

| 场景 | 预期 |
|---|---|
| TokenOccurrence "32.4MPa" 在同一段出现两次 | 两个不同的 TokenOccurrence（span 不同），各自检查 AnchorDeclaration |
| AnchorDeclaration 按 token_text="32.4MPa" 模糊匹配多个 occurrence | AnchorDeclaration 必须精确匹配 span——不允许这种写法 |

---

## READY 判定

逐条检查：

| 检查项 | 结论 |
|---|---|
| RuleStatus 三态（PASS/FAIL/NOT_EVALUABLE）定义清晰 | ✅ |
| ValidationResult.passed 按 case_level 传播三态 | ✅ |
| TrustedTableSpec 外部加载、Validator 独立重算 | ✅ |
| AnchorDeclaration 精确引用 TokenOccurrence span | ✅ |
| ActualIssue.issue_id 确定性复合键（含 issue_type + span） | ✅ |
| CriterionSnapshot.status_mapping 冻结映射 | ✅ |
| 7 条 Rule 每条 PASS/FAIL/NOT_EVALUABLE 触发条件 | ✅ 每条独立表格 |
| 禁止隐式 PASS（expected_total=0 → NOT_EVALUABLE, hit_rate=0.0） | ✅ |
| P0_RULE_IDS 与 SystemReportedIssue.rule_id 两域独立 | ✅ |
| known_issues 不参与任何判定 | ✅ |
| TypedDict Protocol 零 import 未来类 | ✅ |
| 全部 7 条 Rule NOT_EVALUABLE 条件可直接写断言 | ✅ |
| gold/silver/bronze 行为严格符合 05_EVALUATION.md | ✅ |
| 无新 P0 Rule 混入 | ✅ |

---

**READY**
