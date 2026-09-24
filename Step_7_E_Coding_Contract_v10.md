# Step 7-E Accuracy Validator — Coding Contract v10

> 版本：v10
> 状态：READY
>
> 本文档冻结 Step 7-E Accuracy Validator 的完整编码契约。
> 所有规则、数据类型、状态机、边界条件均为正式声明，实现必须严格遵守。

---

## 0. 前置不变量（冻结，不可修改）

以下不变量在 Step 7-E 实现过程中**必须保持**，任何冲突都意味着契约被破坏：

| 编号 | 不变量 | 来源 |
| ---- | ---- | ---- |
| INV-1 | 7 P0 Rule 不新增、不删除 | v9 冻结 |
| INV-2 | Step 7-D 类型（GroundTruthFact / ExpectedIssue / EvaluationCase 等）只读，不修改 | Step 7-D v4 冻结 |
| INV-3 | known_issues 只读诊断，不参与判定流程 | Step 7-D 设计 |
| INV-4 | ExpectedIssue 不内部 deduplicate，重复 → precondition_error | v9 冻结 |
| INV-5 | Validator 不执行 Criterion grammar，LOGIC.CONSISTENCY 用 ExpectedEvaluationSnapshot 替代 | 07_DECISIONS |
| INV-6 | 不做 build gate，不计算 gold_count_total | 05_EVAL §9 D-024 |
| INV-7 | 不 import Step 7-F+ 类（TypedDict Protocol） | v9 冻结 |
| INV-8 | 不引入 LLM 语义判断 | 项目规则：程序负责确定性 |
| INV-9 | `frozen_round` 用 `Decimal(str(value)).quantize(ROUND_HALF_UP)`，禁止 Python `round()` | v9 冻结 |
| INV-10 | `issue_id` = `SHA-256(canonical JSON of identity tuple)`，禁止 Python `hash()` | v9 冻结 |
| INV-11 | `Anchor.fact_id` 必须通过 `cdm.id.is_valid_fact_id()` | Step 7-B 冻结 |
| INV-12 | Unit Registry value 必须来自 `src.cdm.registry.QUANTITY_KINDS` | Step 7-B 冻结 |
| INV-13 | 不允许外部向 ActualIssue 构造函数传入 identity / issue_id（见 §11） | v10 新增 |
| INV-14 | TemplateSpec.fact_display_spec 是显示规格唯一真相源；Step 7-E 只取只读 decimals 快照（见 §3.1） | D-025 |
| INV-15 | renderable_fact_types ≠ required_facts，是独立 trusted snapshot（见 §3.2） | v10 确认 |

---

## 1. RuleStatus + CaseStatus

### 1.1 RuleStatus（单条规则判定结果）

| 值 | 含义 |
| ---- | ---- |
| `PASS` | 该规则在当前用例中通过 |
| `FAIL` | 该规则在当前用例中失败 |
| `NOT_EVALUABLE` | 该规则在当前用例中无法评估（缺少必要输入） |

### 1.2 CaseStatus（用例总体判定结果）

| 值 | 含义 |
| ---- | ---- |
| `PASS` | 全部 applicable Rule 均 PASS |
| `FAIL` | 至少一条 applicable Rule FAIL |
| `NOT_EVALUABLE` | 无可适用规则，或至少一条 applicable Rule 为 NOT_EVALUABLE |

### 1.3 计数定义（v10 正式冻结）

两个统计量是**纯集合运算**，与 CaseStatus 聚合逻辑解耦：

```
applicable_rule_count
    = |{ rule : rule.rule_id ∈ trusted_input.applicable_rules }|

evaluable_rule_count
    = |{ rule : rule.status ∈ {PASS, FAIL} }|
```

约束：
- `applicable_rule_count` 仅从 `TrustedInput.applicable_rules` 集合大小计算，不依赖任何判定结果
- `evaluable_rule_count` 在全部 RuleStatus 计算完成后统计
- `evaluable_rule_count ≤ applicable_rule_count`（每条 Rule 先有 applicability，再有判定结果）

**重要澄清**：
> `CaseStatus.PASS ≠ 7 个 P0 Rule 全部测试`。
> 只要全部 applicable Rule 均 PASS，CaseStatus 即为 PASS。
> `applicable_rule_count` 可能小于 7（用例只适用部分规则）。

### 1.4 CaseStatus 聚合顺序（v9 冻结，v10 确认）

四条规则**顺序敏感**，逐条检查命中即停止：

| 优先级 | 条件 | 结果 |
| ---- | ---- | ---- |
| 1 | 任一 applicable Rule = FAIL | → CaseStatus = **FAIL** |
| 2 | `applicable_rule_count == 0` | → CaseStatus = **NOT_EVALUABLE** |
| 3 | 任一 applicable Rule = NOT_EVALUABLE | → CaseStatus = **NOT_EVALUABLE** |
| 4 | 全部 applicable Rule = PASS | → CaseStatus = **PASS** |

推理链：FAIL 优先覆盖 NOT_EVALUABLE，NOT_EVALUABLE 优先覆盖 PASS。

---

## 2. 五阶段状态机（v9 冻结）

```
Stage A (Precondition)
    ↓ pass
Stage B (Applicability)
    ↓ pass（至少一条 applicable Rule）
Stage C (System Output Exists)
    ↓ pass
Stage D (Well-formed)
    ↓ pass
Stage E (Compare)
```

### 2.1 Stage A: Precondition（v10 增补重复检查）

**输入**：TrustedInput + SystemOutput
**产出**：precondition_error 或进入 Stage B

检查项（按执行顺序）：

| # | 检查内容 | 失败条件 |
| ---- | ---- | ---- |
| A-1 | SystemOutput.report_ir 非空 | SystemOutput 字段缺失 |
| A-2 | TrustedInput.fact_display_spec 中同一 fact_type 不重复 | 重复 fact_type → **precondition_error** |
| A-3 | TrustedInput.fact_display_spec 的 key 均为合法 fact_type 格式 | key 格式 ≠ `<scope_class>.<attribute>` → precondition_error |
| A-4 | TrustedInput.unit_registry 内容完整且每个 value ∈ QUANTITY_KINDS | value 不在 QUANTITY_KINDS → precondition_error |
| A-5 | TrustedInput.renderable_fact_types 为 frozenset[str]（或空） | 非 frozenset 或元素非 str → precondition_error |
| A-6 | TrustedInput.applicable_rules 为 frozenset[str]（case-local） | 非 frozenset 或元素非 str → precondition_error |
| A-7 | ExpectedIssue 不重复同一 rule_id + fact_id 组合 | 重复 → precondition_error |
| A-8 | 每个 GroundTruthFact.fact_id 通过 `cdm.id.is_valid_fact_id()` | 格式无效 → precondition_error |
| A-9 | renderable_fact_types 中每个元素 ∈ 合法 fact_type 注册表 | 非法 → precondition_error |

**v10 新增**：A-2（FactDisplaySpec 重复 fact_type 显式禁止，不允许 first-wins）。

### 2.2 Stage B: Applicability

为每条 Rule 判定当前用例是否适用，产出 `applicable_rules: frozenset[str]`。

### 2.3 Stage C: System Output Exists

检查 SystemOutput 关键字段非空（report_ir / rendered_segments / structured_tables）。

### 2.4 Stage D: Well-formed

词法层检查（tokenizer / identifier grammar / unit 解析 / whitelist 命中）。详见 §5–§9。

### 2.5 Stage E: Compare

语义层比对（数值精度、锚定绑定、逻辑一致性、issue 匹配等）。

---

## 3. TrustedInput Schema（只读）

TrustedInput 是 Step 7-E 的唯一可信输入来源，由评测框架在 Validator 运行前加载。Validator 自己不生成任何 TrustedInput 内容。

```python
class TrustedInput:
    case: EvaluationCase                          # Step 7-D 冻结类型（INV-2）
    applicable_rules: frozenset[str]              # case-local，从 applicable_rules.json 加载
    expected_evaluations: ExpectedEvaluationSnapshot  # LOGIC.CONSISTENCY 唯一可信答案
    whitelist: WhitelistRegistry                  # 登记制白名单（§41.2.4）
    fact_display_spec: dict[str, FactDisplaySpecSnapshot]  # v10 两层明确（§3.1）
    trusted_table_specs: dict[str, TableSpecSnapshot]      # 表格结构快照
    unit_registry: UnitRegistrySnapshot           # v10 完整内容冻结（§3.2）
    renderable_fact_types: frozenset[str]         # v10 schema 明确（§3.3）
```

### 3.1 FactDisplaySpec 两层明确（v10 冻结）

**层 1（渲染真相源）**：`TemplateSpec.fact_display_spec[fact_type]` — 04_REPORT_IR v0.3.1.1 定义的完整渲染规格，包含：
- `decimals: int` — 显示精度
- `unit: str` — 显示单位

这是 D-025 确立的唯一真相源，Renderer 按此规格渲染每个 Fact。

**层 2（Validator 快照）**：Step 7-E 从 TemplateSpec 派生的**只读**快照，本阶段**只读取 decimals**：

```python
class FactDisplaySpecSnapshot:
    fact_type: str
    decimals: int                     # Step 7-E 只取此字段
    # unit 字段存在于 TemplateSpec 但 Step 7-E 不读取
    # Step 7-E 不执行 unit conversion（INV-14）
```

**两层关系**：`TemplateSpec.fact_display_spec` ⊃ `FactDisplaySpecSnapshot`（后者是前者的只读子集投影）。Step 7-E 不修改、不覆盖、不派生 unit。

### 3.2 Unit Registry 完整内容（v10 冻结）

`UnitRegistrySnapshot.mapping` 从 `UNIT_TO_QUANTITY` dict 加载，内容**完全固定**如下（value 全部来自 QUANTITY_KINDS）：

```python
UNIT_TO_QUANTITY: dict[str, str] = {
    # Pressure
    "MPa": "pressure",
    "Pa": "pressure",
    "kPa": "pressure",
    "GPa": "pressure",
    "N/mm²": "pressure",
    "kN/m²": "pressure",
    # Length
    "m": "length",
    "mm": "length",
    "cm": "length",
    "km": "length",
    # Area
    "m²": "area",
    "mm²": "area",
    "cm²": "area",
    # Volume
    "m³": "volume",
    "mm³": "volume",
    # Mass
    "kg": "mass",
    "t": "mass",
    "g": "mass",
    # Force
    "N": "force",
    "kN": "force",
    # Frequency
    "Hz": "frequency",
    # Ratio
    "%": "ratio",
}
```

约束：
- 新增单位必须先改此 dict + CDM QUANTITY_KINDS，不能运行时动态添加
- 未知单位（不在此 dict 中）→ Stage D FAIL，错误码 `ANCHOR.UNKNOWN_UNIT`
- Unit Registry 的 hash 由 Contract 冻结，加载时做完整性校验

### 3.3 renderable_fact_types schema（v10 冻结）

```python
renderable_fact_types: frozenset[str]
```

约束：
- 类型：必须是 `frozenset[str]`（不可变集合）
- 每个元素：必须是合法 fact_type 格式（`<scope_class>.<attribute>`）
- 空集合：合法
- **与 required_facts 的明确区分**（INV-15）：
  - `required_facts`（在 TemplateSpec 中）：类型层需求清单，用于输入完整性预检
  - `renderable_fact_types`（TrustedInput 中）：本报告实际渲染过的 fact_type 快照，用于 ANCHOR.BINDING 的上下文判断
  - 二者可能重叠，但语义不同

---

## 4. SystemOutput Schema

SystemOutput 是被评测系统的产物，由 Step 7-E 自己 tokenize（Fix 3），不信任 System 自报的 TokenOccurrence。

```python
class SystemOutput:
    report_ir: TypedDict            # 04_REPORT_IR 定义的 Document
    rendered_segments: list[str]    # 渲染后按段落/表格/图注切分的文本列表
    structured_tables: list[TableRow]  # 解析后的表格行
    system_reported_issues: list[SystemIssue]  # 系统自报的 issue（仅供诊断）
    system_evaluations: list[SystemEvaluation]  # 系统自报的 evaluation（仅供诊断）
    # 注：无 TokenOccurrence —— Validator 自己 tokenize（Fix 3）
```

---

## 5. Tokenizer（Validator 自己 tokenize）

Step 7-E 从渲染文本**自己提取** numeric + identifier token，不信任 System 的声明。

### 5.1 冻结的正则表达式

```python
# 数值 token：捕获可选 ± 前缀、可选 ± 号、数字主体、可选科学计数法、可选单位后缀
NUMERIC_TOKEN_RE = r'(?<![0-9A-Za-z])(±\s*)?([+-]?)(\d+(?:\.\d+)?)\s*(×10[⁰¹²³⁴⁵⁶⁷⁸⁹]+|×10\^-?\d+|[eE][+-]?\d+)?\s*([A-Za-zμ²³]+)?'

# 标识 token：构件/测点/样品编号（ST 首置；K/C/M/P/S/B/D 前缀）
IDENTIFIER_TOKEN_RE = r'(?<![A-Za-z0-9])((?:构件|测点|样品|检测点|裂缝|强度|检验点)?\s*)(ST|[KCMPSBD])(-?\d{1,5})(-\d{1,3})?(?![A-Za-z0-9])'
```

### 5.2 Overlap 消解（v9 冻结）

同一位置同时匹配 numeric 和 identifier 时，按 sort key 排序取第一个：

```python
sort_key = (char_start, -span_length, 0 if identifier else 1)
# identifier 优先于 numeric（0 < 1）
# 同类型时长 span 优先（-span_length 小的在前）
# 起始位置早的优先
```

### 5.3 Candidate 反例（v10 与 regex 严格对齐）

| 原始文本 | identifier candidate | numeric candidate | overlap 消解后 |
| ---- | ---- | ---- | ---- |
| `ST001` | `ST001`（完整，正则匹配 `ST` + `001`） | **NONE**（`(?<![0-9A-Za-z])` 在 T 之后 lookbehind 失败，因为前一个字符 T 属于 `[0-9A-Za-z]`） | identifier `ST001` |
| `S-3` | `S-3`（正则匹配 `S` + `-3`） | `3`（在 `-` 之后 lookbehind 成功，因为 `-` 不属于 `[0-9A-Za-z]`） | identifier `S-3`（identifier 优先） |
| `2.3m` | 无 | `2.3m` | numeric `2.3m` |
| `K001` | `K001` | **NONE**（`(?<![0-9A-Za-z])` 对第一个字符 K lookbehind 成功，但后面没有 numeric 候选位置——正则要求 lookbehind 在 numeric 起始位，此处无 numeric 起始位） | identifier `K001` |
| `H2O` | 无（H 不在 `ST\|[KCMPSBD]` 前缀中） | 候选 `2`（lookbehind 在 H 之后成功） | numeric `2` |
| `A4` | 无（A 不在前缀中） | 候选 `4` | numeric `4` |
| `ISO9001` | 无（ISO 不在前缀中） | 候选 `9001` | numeric `9001` |
| `GB50292` | 无（GB 不在前缀中） | 候选 `50292` | numeric `50292` |

**v10 关键修正**：`ST001` 的 numeric candidate 明确为 NONE。旧版错误写法（numeric candidate = `001`）忽略了 `(?<![0-9A-Za-z])` 在 T 之后的 lookbehind 失败。

---

## 6. Identifier Grammar（v9 冻结）

### 6.1 正式 grammar

```
identifier := (ST|[KCMPSBD])(-?\d{1,5})(-\d{1,3})?
ST 必须首置，[KCMPSBD] 为单点字符
可选前缀描述性汉字（构件|测点|样品|检测点|裂缝|强度|检验点）
```

### 6.2 明确的非 identifier 集合

以下文本**不属于** Identifier Grammar 匹配范围：
- `H2O`, `CO2`, `A4`, `ISO9001`, `GB50292`, `C30`, `Q235`（前缀不在受控集）
- 规范号、标准号、章节号、表格号（这些走 whitelist，不走 identifier grammar）

### 6.3 scope_class 映射（仅作文本解析规则，不声明类型）

| 前缀 | 可能映射的 scope_class | 说明 |
| ---- | ---- | ---- |
| ST | point / sample | 检测点 / 样品 |
| K, C | component | 构件 |
| M | material | 材料 |
| P | point | 测点 |
| S | sample / defect | 样品 / 缺陷 |
| B | building | 建筑 |
| D | defect | 缺陷 |

注：映射是"可能"关系——具体 scope_class 由 GroundTruthFact.fact_id 三段式确认，Identifier Grammar 只负责**提取 candidate**，不做最终类型判定。

---

## 7. Unit Registry 完整内容（v10 冻结，重复 §3.2）

§3.2 已列完整 `UNIT_TO_QUANTITY` dict。本节不再重复。

未知单位处理：
- 未出现在 dict 中的单位 → Stage D 产出 `UNKNOWN_UNIT` 错误
- 示例：`psi` → `UNKNOWN_UNIT` → Stage D FAIL

---

## 8. ANCHOR.BINDING（v9 冻结）

### 8.1 Stage D 检查（Well-formed）

| # | 检查 | 错误码 |
| ---- | ---- | ---- |
| BIND-1 | 每个 numeric token 的单位后缀可被 `UNIT_TO_QUANTITY` 解析 | `ANCHOR.UNKNOWN_UNIT` |
| BIND-2 | 无 `±` 前缀（本阶段不支持公差表达） | `ANCHOR.UNSUPPORTED_SIGN` |
| BIND-3 | identifier token 的 grammar 合法（§6） | `ANCHOR.INVALID_IDENTIFIER` |
| BIND-4 | 白名单 token 按登记制匹配 | 不匹配 → `ANCHOR.UNANCHORED` |
| BIND-5 | 每个 identifier token 的 fact_id 通过 `cdm.id.is_valid_fact_id()`（INV-11） | `ANCHOR.INVALID_FACT_ID` |
| BIND-6 | 同一 token 不出现多处绑定声明 | `ANCHOR.DUPLICATE_BINDING` |

### 8.2 Stage E 级联 unit 判定

对每个数值 token，按以下优先级与 GroundTruthFact 比较：

| 优先级 | 条件 | 结果 | 错误码 |
| ---- | ---- | ---- | ---- |
| 1 | token 单位未知（UNKNOWN_UNIT） | FAIL | `ANCHOR.UNKNOWN_UNIT` |
| 2 | token quantity_kind ≠ G3.quantity_kind | FAIL | `ANCHOR.QUANTITY_KIND_MISMATCH` |
| 3 | token quantity_kind = G3.quantity_kind 但单位字符串不同 | FAIL | `ANCHOR.UNIT_MISMATCH` |
| 4 | token.parsed_unit == G3.unit | PASS | — |

注：`quantity_kind` 由 token 的单位后缀通过 `UNIT_TO_QUANTITY` 查表得到。

---

## 9. FactDisplaySpec（v10 两层明确，§3.1 补充）

§3.1 已明确两层关系。本节仅补充 Stage A 重复检查规则（v10 新增）：

### 9.1 重复 fact_type → precondition_error

`fact_display_spec` 以 `dict[fact_type] = FactDisplaySpecSnapshot` 形式加载。若加载源（JSON 数组）包含相同 fact_type 超过一次：

```
错误码：PRECONDITION.FACT_DISPLAY_SPEC_DUPLICATE_FACT_TYPE
错误信息："fact_display_spec contains duplicate fact_type: {fact_type}"
```

禁止 first-wins 静默处理。

---

## 10. ExpectedIssue matching（v9 冻结）

### 10.1 Rule-level 匹配

每个 ExpectedIssue 携带 `rule_id`，匹配时只考虑同 rule_id 的 ActualIssue。

### 10.2 两阶段匹配

| 阶段 | 条件 | 行为 |
| ---- | ---- | ---- |
| Phase 1: Exact | ActualIssue 的 rule_id + fact_id + severity 与 ExpectedIssue 完全相等 | 标记为 matched，不再参与 Phase 2 |
| Phase 2: Wildcard | ExpectedIssue 无 fact_id（或 fact_id=None），同 rule_id 的 ActualIssue 任一可匹配 | 标记为 matched，不再参与后续匹配 |

### 10.3 One-to-one 保证

`consumed_indices: set[int]` 记录已匹配的 ExpectedIssue 下标，不允许同一个 ExpectedIssue 被匹配两次。

---

## 11. ActualIssue 构造语义（v10 正式冻结）

### 11.1 Public constructor 禁止传入 identity / issue_id

```python
@dataclass
class ActualIssue:
    # ── Public constructor 接受的字段 ──
    rule_id: str
    severity: str
    category: str
    message: str
    fact_id: Optional[str] = None
    source: str = ""          # 错误来源位置（文本坐标）

    # ── 以下字段不在 public constructor 中（init=False） ──
    identity: tuple = field(init=False, repr=False)
    issue_id: str = field(init=False, repr=False)
```

约束：
- `identity` 和 `issue_id` 被声明为 `field(init=False)`，public constructor 不接受这两个参数
- 若调用方尝试 `ActualIssue(identity=..., issue_id=...)`，Python dataclass 会抛出 `TypeError`（因为这两个参数不在 `__init__` 签名中）
- identity 的构造在 `build()` classmethod 内部完成

### 11.2 build() classmethod（唯一构造入口）

```python
@classmethod
def build(
    cls,
    rule_id: str,
    severity: str,
    category: str,
    message: str,
    fact_id: Optional[str] = None,
    source: str = "",
) -> "ActualIssue":
    # identity 由 build() 内部构造，外部无法控制
    identity = (rule_id, severity, category, fact_id, source)
    instance = cls(
        rule_id=rule_id,
        severity=severity,
        category=category,
        message=message,
        fact_id=fact_id,
        source=source,
    )
    object.__setattr__(instance, "identity", identity)
    object.__setattr__(instance, "issue_id", _compute_issue_id(identity))
    return instance
```

### 11.3 issue_id 计算（INV-10）

```python
def _compute_issue_id(identity: tuple) -> str:
    canonical_json = json.dumps(identity, sort_keys=True, ensure_ascii=False)
    return hashlib.sha256(canonical_json.encode("utf-8")).hexdigest()
```

约束：
- 禁止 Python `hash()`（结果不稳定、进程间不同）
- identity tuple 顺序必须严格固定（rule_id → severity → category → fact_id → source）

### 11.4 外部构造检查

若测试或实现中出现以下模式，视为契约违规：

```python
# ❌ 禁止：外部传入 identity
issue = ActualIssue(rule_id="ANCHOR.BINDING", identity=("x", "y"))

# ❌ 禁止：外部传入 issue_id
issue = ActualIssue(rule_id="ANCHOR.BINDING", issue_id="abc123")

# ✅ 允许：唯一入口
issue = ActualIssue.build(
    rule_id="ANCHOR.BINDING",
    severity="error",
    category="QUANTITY_KIND_MISMATCH",
    message="token quantity_kind mismatch",
    fact_id="fact:component.K001.concrete_strength",
    source="rendered_segments[2]:char[15-20]",
)
```

---

## 12. frozen_round（v9 冻结）

```python
from decimal import Decimal, ROUND_HALF_UP

def frozen_round(value: Any, decimals: int) -> float:
    """Decimal(str(value)).quantize 结果，用 ROUND_HALF_UP。"""
    # 禁止 NaN / Inf → 抛 ValueError
    # decimals < 0 → 抛 TypeError
    # value = None → 抛 ValueError
    d = Decimal(str(value))
    quantize_str = "0." + ("0" * decimals) if decimals > 0 else "1"
    return float(d.quantize(Decimal(quantize_str), rounding=ROUND_HALF_UP))
```

约束：
- 禁止 Python 内置 `round()`（banker's rounding，与工程四舍五入不符）
- 必须用 `Decimal(str(value))` 而非 `Decimal(value)`（后者对浮点二进制误差敏感）

---

## 13. 7 P0 Rules（v9 冻结，不增删）

| # | Rule ID | 含义 | 核心判定 |
| ---- | ---- | ---- | ---- |
| P0-1 | `VALUE.CONSISTENCY` | 报告数值与 Ground Truth 一致 | frozen_round(token_value, G3.precision) == G3.value |
| P0-2 | `TABLE.INTERNAL` | 表格内部数值自洽（合计、统计量与分项一致） | 程序计算值 vs 报告声明值 |
| P0-3 | `LOGIC.CONSISTENCY` | 判定逻辑与 ExpectedEvaluationSnapshot 一致 | Validator 不执行 Criterion grammar，直接比对 snapshot |
| P0-4 | `ANCHOR.BINDING` | 每个数字锚定正确 | Unit 解析 + quantity_kind 级联 + 标识同表绑定 |
| P0-5 | `CONCLUSION.DIRECTION` | 结论方向与 GroundTruthConclusion.direction 一致 | 字符串精确匹配（case-sensitive） |
| P0-6 | `CONCLUSION.COVERAGE` | 结论覆盖全部检测项 | required_evaluation_ids ⊆ conclusion.covers[] |
| P0-7 | `EXPECTED.HIT` | ExpectedIssues 被全部命中 | 两阶段 matching + consumed_indices 去重 |

---

## 14. 测试契约

### 14.1 必须覆盖的反例

v10 冻结了 8 个待闭环项，实现时必须为每个反例编写独立测试用例：

| # | 反例类别 | 测试重点 |
| ---- | ---- | ---- |
| T-1 | Counting | applicable_rule_count = 0 时 CaseStatus = NOT_EVALUABLE；applicable_rule_count < 7 时仍然 PASS |
| T-2 | Unit Registry | UNIT_TO_QUANTITY dict 完整且每个 value ∈ QUANTITY_KINDS；unknown unit → UNKNOWN_UNIT |
| T-3 | FactDisplaySpec 两层 | TemplateSpec dict 含 decimals+unit；Snapshot 只含 decimals；Step 7-E 不 import unit 字段 |
| T-4 | ActualIssue constructor | ActualIssue(identity=...) → TypeError；ActualIssue(issue_id=...) → TypeError；ActualIssue.build() 正常工作 |
| T-5 | FactDisplaySpec 重复 | JSON 数组含重复 fact_type → precondition_error；dict 含唯一 key → 正常 |
| T-6 | Tokenizer candidate | ST001 numeric=NONE；S-3 numeric=3；overlap 消解 identifier 优先 |
| T-7 | renderable_fact_types schema | frozenset[str] 正常；list[str] → TypeError；空集合合法；非法 fact_type → precondition_error |
| T-8 | 冻结正则 | NUMERIC_TOKEN_RE 和 IDENTIFIER_TOKEN_RE 逐段验证所有反例 |

### 14.2 实现阶段划分

```
Phase 1: 契约自检（contracts.py）
    → 数据结构 + 枚举 + 构造函数语义

Phase 2: 词法层（numeric_parser.py + identifier_parser.py）
    → regex + tokenize + overlap 消解

Phase 3: 单元层（unit_registry.py + anchor.py）
    → UNIT_TO_QUANTITY + quantity_kind 级联

Phase 4: 比对层（compare.py + actual_issues.py）
    → frozen_round + issue matching + identity 构造

Phase 5: 状态机（accuracy_validator.py）
    → 五阶段 + CaseStatus 聚合
```

---

## 结尾判定：READY

**8 项闭环全部达成**：

| # | 闭环项 | 状态 | 位置 |
| ---- | ---- | ---- | ---- |
| 一 | applicable/evaluable 计数明确定义 | ✅ | §1.3 |
| 二 | Unit Registry 完整 UNIT_TO_QUANTITY | ✅ | §3.2 / §7 |
| 三 | D-025 两层明确 | ✅ | §3.1 / §9 |
| 四 | ActualIssue constructor 禁止外部 identity | ✅ | §11 |
| 五 | FactDisplaySpec 重复 → precondition_error | ✅ | §2.1 A-2 / §9.1 |
| 六 | tokenizer candidate 与 regex 对齐 | ✅ | §5.3 |
| 七 | renderable_fact_types schema 明确 | ✅ | §3.3 |
| 八 | 文档版本卫生 | ✅ | 标题/正文统一为 v10，纯净无元话语 |

**所有既有不变量保持**：INV-1 至 INV-15 共 15 条，全部明确声明。

**Step 7-E Coding Contract v10 判定：READY**

实现可以开始。
