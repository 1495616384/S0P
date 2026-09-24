# Step 7-E Accuracy Validator — Coding Contract v10（回归修复版）

> 版本：v10（回归修复版）
> 状态：READY
>
> 本文档冻结 Step 7-E Accuracy Validator 的完整编码契约。
> 本轮为回归审计（REGRESSION FOUND）后的**恢复轮**：只恢复被意外破坏的既有冻结契约。
> 不新增业务能力、不新增 P0 Rule、不修改 Step 7-D、不扩大 Validator 职责、不写代码。

> **回归修复记录（R1–R12）**：
>
> | 编号 | 恢复项 | 位置 |
> | ---- | ---- | ---- |
> | R1 | ExpectedIssue matching 恢复 (rule_id, fact_id) 粒度 + 双向 one-to-one | §10 |
> | R2 | ActualIssue identity 恢复冻结六元组（保留构造修复） | §11 |
> | R3 | unit lexer 完整捕获注册表全部键 | §5.1 / §5.4 |
> | R4 | P0-1 精度唯一来源收敛到 Snapshot.decimals | §13 |
> | R5 | Stage B 恢复只读 membership test | §2.2 |
> | R6 | expected_evaluations 恢复多 Evaluation dict 结构 | §3 / §3.4 |
> | R7 | Unit Registry 冻结改为 exact equality，删除无值 hash 声明 | §3.2 / §2.1 A-4 |
> | R8 | 删除 identifier → scope_class 映射表 | §6.3 |
> | R9 | renderable_fact_types 恢复 allowed-to-render 语义 | §3.3 |
> | R10 | SystemOutput 恢复结构表达力 | §4 |
> | R11 | 补回 v10 静默丢失的冻结条款：range 逐 token 锚定 + whitelist span containment | §8.3 / §8.4 |
> | R12 | 自检补充恢复：§5.3 四行 candidate 对齐、§6.2 C30 对齐、BIND-7 标识同表绑定、INV-15 交叉引用修正 | §5.3 / §6.2 / §8.1 / §0 |
> | R13 | B1 修复：unit lexer 完整 token 边界（unit-start / unit-cont 文法 + 3 组新反例 + U-5） | §5.1 / §5.4 |
> | R14 | B2 修复：frozen_round 异常行为与契约一致（补齐检查，数学语义零改动） | §12 |
> | R15 | B3 修复：ActualIssue build-only 守卫（__post_init__ + object.__new__ build） | §11 |
> | R16 | 旧契约遗漏恢复：applicable_rules ⊆ P0_RULE_IDS（A-11；未知 Rule → precondition_error） | §2.1 / §2.2 / §13 / §0 |
> | R17 | B4 修复：numeric token → fact_id 显式绑定闭环（BIND-8 + §8.2 正式绑定与判定链 + INV-24 + T-18） | §8.1 / §8.2 / §0 / §14 |

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
| INV-13 | ActualIssue 为 build-only：identity / issue_id 不可作为 constructor 参数（TypeError）；任何 ActualIssue(...) 直接构造亦失败（__post_init__ 守卫）；唯一构造入口 = build()，产物同时具备完整 identity 与 issue_id（见 §11） | v10 新增，B3 加固 |
| INV-14 | TemplateSpec.fact_display_spec 是显示规格唯一真相源；Step 7-E 只取只读 decimals 快照（见 §3.1） | D-025 |
| INV-15 | renderable_fact_types ≠ required_facts，是独立 trusted snapshot（见 §3.3） | v10 确认 |
| INV-16 | matching key = (rule_id, fact_id)；severity / note 不参与 matching；one-to-one 双向（consumed_expected_indices + consumed_actual_indices） | 冻结恢复 R1 |
| INV-17 | ActualIssue.identity ≡ (issue_type, location, fact_id, evaluation_id, occurrence_id, table_aggregate)；变更 identity 语义必须先落 07_DECISIONS.md | 冻结恢复 R2 |
| INV-18 | UNIT_TO_QUANTITY 每个 key 必须被 numeric tokenizer 单位组完整捕获；加载 mapping 必须与冻结 dict 完全相等（A-4） | 冻结恢复 R3/R7 |
| INV-19 | P0 numeric comparison 精度唯一来源 = FactDisplaySpecSnapshot.decimals；G3.precision 不参与 | 冻结恢复 R4 |
| INV-20 | applicable_rules 唯一来源 = trusted_input.applicable_rules；Stage B 只读 membership test | 冻结恢复 R5 |
| INV-21 | renderable_fact_types = allowed-to-render（trusted 侧）；actual-rendered 不进入 TrustedInput | 冻结恢复 R9 |
| INV-22 | Anchor 标识比较 = normalized_identifier == parse_fact_id(anchor.fact_id).instance_key；禁止 scope_class 推断 | 冻结恢复 R8 |
| INV-23 | trusted_input.applicable_rules ⊆ P0_RULE_IDS（§13）；未知 / 非 7 P0 rule_id → precondition_error；不静默删除、不自动补入、不修改 trusted_input | 本轮恢复 R16 |
| INV-24 | 每个非 whitelist 豁免 numeric token 必须被唯一 AnchorDeclaration 绑定到合法 fact_id（BIND-8）；§8.2 比较链（token → fact_id → G3 → snapshot.decimals → P0-1）缺环即 FAIL / precondition_error | 本轮恢复 R17（04_REPORT_IR §41.2.2 / R2） |

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

### 1.3 计数定义（冻结）

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

### 1.4 CaseStatus 聚合顺序（冻结）

四条规则**顺序敏感**，逐条检查命中即停止：

| 优先级 | 条件 | 结果 |
| ---- | ---- | ---- |
| 1 | 任一 applicable Rule = FAIL | → CaseStatus = **FAIL** |
| 2 | `applicable_rule_count == 0` | → CaseStatus = **NOT_EVALUABLE** |
| 3 | 任一 applicable Rule = NOT_EVALUABLE | → CaseStatus = **NOT_EVALUABLE** |
| 4 | 全部 applicable Rule = PASS | → CaseStatus = **PASS** |

推理链：FAIL 优先覆盖 NOT_EVALUABLE，NOT_EVALUABLE 优先覆盖 PASS。

---

## 2. 五阶段状态机（冻结）

```
Stage A (Precondition)
    ↓ pass
Stage B (Applicability · 只读 membership test)
    ↓ pass（至少一条 applicable Rule）
Stage C (System Output Exists)
    ↓ pass
Stage D (Well-formed)
    ↓ pass
Stage E (Compare)
```

### 2.1 Stage A: Precondition

**输入**：TrustedInput + SystemOutput
**产出**：precondition_error 或进入 Stage B

检查项（按执行顺序）：

| # | 检查内容 | 失败条件 |
| ---- | ---- | ---- |
| A-1 | SystemOutput.report_ir 非空 | SystemOutput 字段缺失 |
| A-2 | TrustedInput.fact_display_spec 中同一 fact_type 不重复 | 重复 fact_type → **precondition_error** |
| A-3 | TrustedInput.fact_display_spec 的 key 均为合法 fact_type 格式 | key 格式 ≠ `<scope_class>.<attribute>` → precondition_error |
| A-4 | TrustedInput.unit_registry.mapping 与 §3.2 冻结 `UNIT_TO_QUANTITY` **完全相等（逐键逐值）** | 缺键 / 多键 / 任一值不等 → **precondition_error** |
| A-5 | TrustedInput.renderable_fact_types 为 frozenset[str]（或空） | 非 frozenset 或元素非 str → precondition_error |
| A-6 | TrustedInput.applicable_rules 为 frozenset[str]（case-local） | 非 frozenset 或元素非 str → precondition_error |
| A-7 | ExpectedIssue 不重复同一 rule_id + fact_id 组合 | 重复 → precondition_error |
| A-8 | 每个 GroundTruthFact.fact_id 通过 `cdm.id.is_valid_fact_id()` | 格式无效 → precondition_error |
| A-9 | renderable_fact_types 中每个元素 ∈ 合法 fact_type 注册表 | 非法 → precondition_error |
| A-10 | expected_evaluations 加载源中 evaluation_id 唯一 | 重复 → **precondition_error** |
| A-11 | trusted_input.applicable_rules ⊆ P0_RULE_IDS（7 条冻结 P0 Rule，见 §13） | 出现未知 / 非 7 P0 的 rule_id → **precondition_error** |

说明：
- A-2：FactDisplaySpec 重复 fact_type 显式禁止，不允许 first-wins（v10 闭环保持）
- A-4：R7 恢复——`value ∈ QUANTITY_KINDS` 检查降为冗余防御，**不是充分条件**；缺 unit / 多 unit / 错 mapping 都必须被 A-4 发现
- A-10：R6 恢复——expected_evaluations 多 Evaluation 结构的查重入口
- A-11：R16 恢复（旧契约遗漏）——未知 Rule **不进入 applicable_rule_count**（§1.3 计数在 Stage A 之后执行）、**不静默删除**（不做 filter）、**不自动补入缺失 Rule**、**不修改 trusted_input.applicable_rules**（只读校验：非法即 fail，不做修正）

### 2.2 Stage B: Applicability Resolution（只读，R5 恢复冻结）

```
applicable_rules ≡ trusted_input.applicable_rules
```

来源：`eval_config/cases/<case_id>/applicable_rules.json`（case-local trusted config）

Stage B **仅做 membership test**：

```
rule_id ∉ trusted_input.applicable_rules  →  RuleStatus = NOT_EVALUABLE
rule_id ∈ trusted_input.applicable_rules  →  继续 Stage C–E
```

Validator **不推导、不生成、不修改、不覆盖** applicable_rules（INV-20）。
不存在第二个 applicability 真相源。
Stage B 的 membership test 在 A-11 校验后的集合上执行（`applicable_rules ⊆ P0_RULE_IDS` 已由 Stage A 保证）。

### 2.3 Stage C: System Output Exists

检查 SystemOutput 关键字段非空（report_ir / rendered_segments / structured_tables）。

### 2.4 Stage D: Well-formed

词法层检查（tokenizer / identifier grammar / unit 解析 / whitelist span containment / 标识绑定）。详见 §5–§9。

### 2.5 Stage E: Compare

语义层比对（数值精度、锚定级联、逻辑一致性、issue 匹配等）。

---

## 3. TrustedInput Schema（只读）

TrustedInput 是 Step 7-E 的唯一可信输入来源，由评测框架在 Validator 运行前加载。Validator 自己不生成任何 TrustedInput 内容。

```python
class TrustedInput:
    case: EvaluationCase                          # Step 7-D 冻结类型（INV-2）
    applicable_rules: frozenset[str]              # case-local trusted config（§2.2 只读 membership test 源）
    expected_evaluations: dict[str, ExpectedEvaluationEntry]  # key = evaluation_id（§3.4，R6 恢复）
    whitelist: WhitelistRegistry                  # 登记制白名单（§8.4）
    fact_display_spec: dict[str, FactDisplaySpecSnapshot]  # 两层明确（§3.1）
    trusted_table_specs: dict[str, TableSpecSnapshot]      # 表格结构快照
    unit_registry: UnitRegistrySnapshot           # mapping 必须 == §3.2 冻结 dict（A-4）
    renderable_fact_types: frozenset[str]         # allowed-to-render（§3.3，R9 恢复）
```

### 3.1 FactDisplaySpec 两层明确（冻结）

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

### 3.2 Unit Registry 完整内容（冻结，R7 恢复可验证性）

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

约束（冻结）：
- 上方 dict 即唯一真相源；新增单位必须先改此 dict + CDM QUANTITY_KINDS，不能运行时动态添加
- **Stage A A-4：加载后的 mapping 必须与本 dict 完全相等（逐键逐值）**；缺 unit / 多 unit / 错 mapping → precondition_error
- 未知单位（不在此 dict 中）→ Stage D FAIL，错误码 `ANCHOR.UNKNOWN_UNIT`（示例：`psi`、`abc`，见 §5.4）
- 每一个 dict key 必须能被 numeric tokenizer 单位组**完整捕获**（§5.4 不变量 U-1）
- v10 的「hash 由 Contract 冻结」声明已删除（R7）：无 hash 值的冻结声明不成立，完整性由 A-4 exact equality 保证

### 3.3 renderable_fact_types 语义（R9 恢复冻结：allowed-to-render）

```python
renderable_fact_types: frozenset[str]
```

**定义（冻结）**：allowed-to-render fact_type 集合。

来源链：

```
Template / Render Plan
    ↓
trusted snapshot（随用例配置冻结）
    ↓
TrustedInput.renderable_fact_types
```

**明确区分（冻结）**：

```
allowed-to-render ≠ actual-rendered
```

- `allowed-to-render`：trusted 侧声明的「本报告允许渲染哪些 fact_type」，**进入 TrustedInput**
- `actual-rendered`：SystemOutput 侧的实际渲染事实，只读诊断，**不进入 TrustedInput，不参与任何判定**

**Actual System Output 不能成为 TrustedInput 的来源。**

类型约束：
- 类型：必须是 `frozenset[str]`（不可变集合）
- 每个元素：必须是合法 fact_type 格式（`<scope_class>.<attribute>`）
- 空集合：合法

**与 required_facts 的明确区分**（INV-15）：
- `required_facts`（在 TemplateSpec 中）：类型层需求清单，用于输入完整性预检
- `renderable_fact_types`（TrustedInput 中）：allowed-to-render 快照，用于 ANCHOR.BINDING 的上下文判断
- 二者可能重叠，但语义不同

### 3.4 ExpectedEvaluationSnapshot（R6 恢复冻结：多 Evaluation 集合结构）

```python
expected_evaluations: dict[str, ExpectedEvaluationEntry]
```

- **key** = `evaluation_id`（不透明稳定 ID，与 `GroundTruthConclusion.required_evaluation_ids` 同一 ID 域）
- **value** = `ExpectedEvaluationEntry`

```python
class ExpectedEvaluationEntry:
    evaluation_id: str        # 同 key
    expected_status: str      # 期望判定值（受控值域由 ConclusionRule / report type 提供）
    note: str = ""
```

约束（冻结）：
- **支持多个 Evaluation**；LOGIC.CONSISTENCY 按 evaluation_id 做**确定性查找**
- 加载源中 evaluation_id 重复 → Stage A precondition_error（A-10）
- entry 字段为本阶段最小集；扩展 entry 字段必须先落 07_DECISIONS.md

---

## 4. SystemOutput Schema（R10 恢复结构表达力）

SystemOutput 是被评测系统的产物，由 Step 7-E 自己 tokenize（Fix 3），不信任 System 自报的 TokenOccurrence。

```python
class AnchorDeclaration:
    token: str              # 声明绑定的 token 文本（与 04_REPORT_IR §13.1 anchors {token, fact_id} 对齐）
    fact_id: str            # 绑定目标（必须通过 cdm.id.is_valid_fact_id()，INV-11）
    char_start: int         # 在 RenderedSegment.raw_text 中的起始偏移
    char_end: int           # 结束偏移（不含）

class RenderedSegment:
    segment_location: str   # 定位（semantic path + block id）
    raw_text: str           # 渲染后文本 —— Validator 对其自行 tokenize
    anchor_declarations: list[AnchorDeclaration]

class TableCell:
    raw_text: str
    fact_id: Optional[str] = None    # 来自 Ref 的单元格携带

class StructuredTable:
    table_id: str
    headers: list[TableCell]
    rows: list[list[TableCell]]

class SystemOutput:
    report_ir: TypedDict                          # 04_REPORT_IR 定义的 Document
    rendered_segments: list[RenderedSegment]      # 结构化渲染段（禁止退化为 list[str]）
    structured_tables: list[StructuredTable]
    system_reported_issues: list[SystemIssue]     # 仅供诊断
    system_evaluations: list[SystemEvaluation]    # 仅供诊断
    # 无 TokenOccurrence —— Validator 自己 tokenize（Fix 3 不变）
```

约束（冻结）：
- `rendered_segments` 禁止退化为 `list[str]`——否则 AnchorDeclaration 与 char 定位无处表达，ANCHOR.BINDING / TABLE.INTERNAL 均不可执行
- AnchorDeclaration 的 char 位置仅用于定位与诊断，**不替代 Validator 自行 tokenize**（Fix 3 不变）
- AnchorDeclaration 结构与 04_REPORT_IR §13.1 / §41.2 的 anchors（token + fact_id）对齐

---

## 5. Tokenizer（Validator 自己 tokenize）

Step 7-E 从渲染文本**自己提取** numeric + identifier token，不信任 System 的声明。

### 5.1 冻结的正则表达式（R3 修正单位组，B1 加固完整 token 边界）

```python
# 数值 token：捕获可选 ± 前缀、可选 ± 号、数字主体、可选科学计数法、可选单位后缀
# 单位组（B1 修复）：完整 token 边界 —— unit candidate = [A-Za-zμ%][A-Za-z0-9μ²³%/]*
# （unit-start 起始 + unit-cont 延续的极大连续 run；必须完整捕获 UNIT_TO_QUANTITY 全部键，§5.4）
NUMERIC_TOKEN_RE = r'(?<![0-9A-Za-z])(±\s*)?([+-]?)(\d+(?:\.\d+)?)\s*(×10[⁰¹²³⁴⁵⁶⁷⁸⁹]+|×10\^-?\d+|[eE][+-]?\d+)?\s*([A-Za-zμ%][A-Za-z0-9μ²³%/]*)?'

# 标识 token：构件/测点/样品编号（ST 首置；K/C/M/P/S/B/D 前缀）
IDENTIFIER_TOKEN_RE = r'(?<![A-Za-z0-9])((?:构件|测点|样品|检测点|裂缝|强度|检验点)?\s*)(ST|[KCMPSBD])(-?\d{1,5})(-\d{1,3})?(?![A-Za-z0-9])'
```

### 5.2 Overlap 消解（冻结）

同一位置同时匹配 numeric 和 identifier 时，按 sort key 排序取第一个：

```python
sort_key = (char_start, -span_length, 0 if identifier else 1)
# identifier 优先于 numeric（0 < 1）
# 同类型时长 span 优先（-span_length 小的在前）
# 起始位置早的优先
```

### 5.3 Candidate 反例（R12 修正：与 regex 严格对齐）

| 原始文本 | identifier candidate | numeric candidate | overlap 消解后 |
| ---- | ---- | ---- | ---- |
| `ST001` | `ST001`（正则匹配 `ST` + `001`） | **NONE**（前置 T ∈ [0-9A-Za-z]，lookbehind 失败） | identifier `ST001` |
| `S-3` | `S-3`（正则匹配 `S` + `-3`） | `3`（前置 `-` ∉ [0-9A-Za-z]，lookbehind 成功） | identifier `S-3`（identifier 优先） |
| `2.3m` | 无 | `2.3`（unit = `m`） | numeric `2.3m` |
| `K001` | `K001` | **NONE**（前置 K ∈ [0-9A-Za-z]，lookbehind 失败） | identifier `K001` |
| `H2O` | 无（H 不在受控前缀集） | **NONE**（前置 H ∈ [0-9A-Za-z]，lookbehind 失败） | 无 numeric token |
| `A4` | 无（A 不在受控前缀集） | **NONE**（前置 A ∈ [0-9A-Za-z]，lookbehind 失败） | 无 numeric token |
| `ISO9001` | 无（I 不在受控前缀集） | **NONE**（前置 O ∈ [0-9A-Za-z]，lookbehind 失败） | 无 numeric token |
| `GB50292` | 无（G 不在受控前缀集） | **NONE**（前置 B ∈ [0-9A-Za-z]，lookbehind 失败） | 无 numeric token |

**统一机制**：前置字母一律阻断 numeric candidate（与 `ST001` 同一 lookbehind 机制）。
对照：`GB 50292-2015`（空格分隔）中 `50292` 前置空格 → numeric candidate **存在**，走 whitelist span containment（§8.4）判定。

**R12 修正说明**：v10 表中 H2O / A4 / ISO9001 / GB50292 四行误写 numeric candidate 存在，违反已冻结的「candidate 描述与 regex 一致」要求，本次恢复对齐。

### 5.4 Unit lexer 完整捕获不变量（R3 恢复冻结，B1 加固完整 token 边界）

单位候选文法（冻结）：

```
unit_candidate := [A-Za-zμ%][A-Za-z0-9μ²³%/]*
    unit-start（起始字符）：[A-Za-zμ%]
    unit-cont （延续字符）：[A-Za-z0-9μ²³%/]
```

即：数值结束后，从第一个 unit-start 字符开始，取**极大连续 unit-like run** 作为**完整 unit candidate**。

不变量（冻结）：

| # | 不变量 |
| ---- | ---- |
| U-1 | `UNIT_TO_QUANTITY` 的每一个 key 必须能被单位组**完整捕获**（完整 unit token）（INV-18） |
| U-2 | 禁止前缀截断**与尾部静默截断**：`m3 → m`、`N/mm2 → N/mm`、`N/mm²x → N/mm²` 均为违规 |
| U-3 | 完整 candidate ∉ `UNIT_TO_QUANTITY` → `UNKNOWN_UNIT` → Stage D FAIL（不得静默置 None） |
| U-4 | 数值后无 unit-start 字符（普通标点 / 空格 / 中文说明等）→ unit = None（合法：纯数值 / 无量纲） |
| U-5 | **禁止用 negative lookahead 把错误单位变成「没有 numeric token」**——错误单位必须以 UNKNOWN_UNIT 显式识别，numeric 主体必须保留 |

回归反例（全部必须闭合）：

| 输入 | numeric | unit candidate | quantity_kind | 判定 |
| ---- | ---- | ---- | ---- | ---- |
| `12.5N/mm²` | 12.5 | `N/mm²` | pressure | 完整捕获 |
| `12.5kN/m²` | 12.5 | `kN/m²` | pressure | 完整捕获 |
| `32.5%` | 32.5 | `%` | ratio | 完整捕获 |
| `32.4psi` | 32.4 | `psi` | — | UNKNOWN_UNIT → Stage D FAIL |
| `32.4abc` | 32.4 | `abc` | — | UNKNOWN_UNIT → Stage D FAIL |
| `32.4m3` | 32.4 | `m3` | — | UNKNOWN_UNIT → Stage D FAIL（B1 新增） |
| `32.4N/mm2` | 32.4 | `N/mm2` | — | UNKNOWN_UNIT → Stage D FAIL（B1 新增） |
| `12.5N/mm²x` | 12.5 | `N/mm²x` | — | UNKNOWN_UNIT → Stage D FAIL（B1 新增） |

禁止结果（违规）：
- `m3` → `m`；`N/mm2` → `N/mm`；`N/mm²x` → `N/mm²`（前缀 / 尾部截断）
- `32.4m3` / `32.4N/mm2` / `12.5N/mm²x` → unit=None（静默退化）
- `32.4m3` / `32.4N/mm2` / `12.5N/mm²x` → 无 numeric token（lookahead 吞 token，违反 U-5）

边界澄清（文法推论，非新规则）：
- `/` 属 unit-cont 不属 unit-start → `1/2` 仍解析为两个 numeric token（各自独立锚定），分数不被吞为单位
- `-` 不在 unit 字符集 → range「2.0-2.5m」行为不变（§8.3）
- 数字属 unit-cont 不属 unit-start → `32.4m3` 完整捕获 `m3`，纯数字主体解析不受影响

---

## 6. Identifier Grammar（冻结）

### 6.1 正式 grammar

```
identifier := (ST|[KCMPSBD])(-?\d{1,5})(-\d{1,3})?
ST 必须首置，[KCMPSBD] 为单点字符
可选前缀描述性汉字（构件|测点|样品|检测点|裂缝|强度|检验点）
```

### 6.2 明确的非 identifier 集合（R12 修正对齐）

以下文本**不产生 identifier candidate**（首字母不在受控前缀集，或前缀后非数字）：
- `H2O`, `CO2`, `A4`, `ISO9001`, `GB50292`, `Q235`
- 规范号、标准号、章节号、表格号（这些走 whitelist，不走 identifier grammar）

边界澄清（R12）：`C30` / `C45` 等混凝土强度等级——首字母 C ∈ 受控前缀集，identifier candidate **存在**；词法层不做混凝土等级 / 构件编号的语义区分（禁止 scope_class 推断，§6.3），最终由锚定绑定判定（BIND-7）：未绑定或 instance_key 不等 → FAIL。

### 6.3 标识解析边界（R8 恢复冻结）

Identifier 解析**只负责**：
- text parsing
- 产出 `normalized_identifier`

**不做** identifier → scope_class 推断（v10 的前缀映射表 ST→point/sample、K/C→component、M→material、P→point、S→sample/defect、B→building、D→defect 已整体删除）。

Anchor 标识比较**只做**：

```
normalized_identifier == parse_fact_id(anchor.fact_id).instance_key
```

绝不推断 scope_class（INV-22）。

---

## 7. Unit Registry 完整内容（冻结，重复 §3.2）

§3.2 已列完整 `UNIT_TO_QUANTITY` dict。本节不再重复。

未知单位处理：
- 未出现在 dict 中的单位 → Stage D 产出 `UNKNOWN_UNIT` 错误
- 示例：`psi` → `UNKNOWN_UNIT` → Stage D FAIL；`abc` → `UNKNOWN_UNIT` → Stage D FAIL

---

## 8. ANCHOR.BINDING（冻结）

### 8.1 Stage D 检查（Well-formed）

| # | 检查 | 错误码 |
| ---- | ---- | ---- |
| BIND-1 | 每个 numeric token 的单位后缀可被 `UNIT_TO_QUANTITY` 解析（§5.4 U-1/U-3） | `ANCHOR.UNKNOWN_UNIT` |
| BIND-2 | 无 `±` 前缀（本阶段不支持公差表达；`±3.2MPa` / `± 3.2MPa` 均捕获 `±` 后判 FAIL） | `ANCHOR.UNSUPPORTED_SIGN` |
| BIND-3 | identifier token 的 grammar 合法（§6） | `ANCHOR.INVALID_IDENTIFIER` |
| BIND-4 | 白名单 token 按 span containment 匹配（§8.4） | 不匹配 → `ANCHOR.UNANCHORED` |
| BIND-5 | 每个 AnchorDeclaration.fact_id 通过 `cdm.id.is_valid_fact_id()`（INV-11） | `ANCHOR.INVALID_FACT_ID` |
| BIND-6 | 同一 token 不出现多处绑定声明 | `ANCHOR.DUPLICATE_BINDING` |
| BIND-7 | 每个 identifier token 必须被至少一个 AnchorDeclaration 覆盖（token span 包含于声明 token span 或文本相等），且 `normalized_identifier == parse_fact_id(fact_id).instance_key`（§6.3） | 未覆盖 → `ANCHOR.UNANCHORED`；instance_key 不等 → `ANCHOR.IDENTIFIER_BINDING_MISMATCH` |
| BIND-8 | 每一个未经 whitelist 豁免的 numeric token 必须被至少一个 AnchorDeclaration 覆盖（span 规则同 BIND-7），且覆盖声明提供合法 fact_id（BIND-5）；同一 numeric token 不允许存在多个不同 fact_id 的绑定（多重声明按 BIND-6） | 未覆盖 → `ANCHOR.UNANCHORED`；fact_id 非法 → `ANCHOR.INVALID_FACT_ID`；多个不同 fact_id → `ANCHOR.DUPLICATE_BINDING` |

说明：
- BIND-7 恢复 04_REPORT_IR §41.2.3 / D-018 的「实例标识 token 同表绑定」冻结要求（v10 P0-4 核心判定中的「标识同表绑定」在 Stage D 无对应检查项，本条补齐）
- 被 whitelist span 完全包含的 identifier token 豁免 BIND-7（§8.4）
- BIND-8 恢复 04_REPORT_IR §41.2.2 / R2 的冻结要求（「每一个数字 token 都必须显式绑定到一个 fact_id」）——修复「numeric token → fact_id 无正式来源」的可执行性缺口（B4）；仍属 P0-4 ANCHOR.BINDING，不新增 P0 Rule
- BIND-8 与 BIND-7 使用同一冻结 span 规则；部分重叠（相交但不包含）**不算覆盖**
- BIND-8 作用域 = Validator 对 rendered_segments（prose / 图注）tokenize 出的 numeric token；structured_tables 单元格 numeric token 的逐 token 绑定机制为已登记候选（结尾扫描第 6 项），本轮不设计
- 由此 §8.2 比较链获得正式输入：token → 唯一 fact_id → G3

### 8.2 Stage E：Numeric Token 绑定与判定链（B4 修复，正式闭环）

对每个**未经 whitelist 豁免**的 numeric token（BIND-8 已保证唯一有效绑定）：

```
numeric token
    ↓ 唯一 AnchorDeclaration（BIND-8：span 覆盖 + 唯一 fact_id）
fact_id
    ├─→ G3 查找（case.ground_truth.facts 按 fact_id）
    │       缺 → FAIL（ANCHOR.INVALID_FACT_ID：绑定目标不存在于用例）
    └─→ parse_fact_id() → derive_fact_type() → fact_type
            ↓ fact_display_spec[fact_type]
            │       缺条目 → precondition_error（missing display spec，v9 冻结保持）
        FactDisplaySpecSnapshot.decimals
            ↓
        frozen_round(token_value, decimals) == G3.value   （P0-1 VALUE.CONSISTENCY）
```

Unit 级联（与上述链并行执行，按优先级与 G3 比较）：

| 优先级 | 条件 | 结果 | 错误码 |
| ---- | ---- | ---- | ---- |
| 1 | token 单位未知（UNKNOWN_UNIT） | FAIL | `ANCHOR.UNKNOWN_UNIT` |
| 2 | token quantity_kind ≠ G3.quantity_kind | FAIL | `ANCHOR.QUANTITY_KIND_MISMATCH` |
| 3 | token quantity_kind = G3.quantity_kind 但单位字符串不同 | FAIL | `ANCHOR.UNIT_MISMATCH` |
| 4 | token.parsed_unit == G3.unit | PASS | — |

注：
- `quantity_kind` 由 token 的单位后缀通过 `UNIT_TO_QUANTITY` 查表得到
- **缺任一环 → FAIL / precondition_error**；错误码均属现有 ANCHOR.* 体系，不新增 P0 Rule
- `ANCHOR.INVALID_FACT_ID` 两级定义：① `is_valid_fact_id()` 失败（语法，Stage D / BIND-5）；② fact_id ∉ 当前用例 GroundTruthFact 集合（绑定目标不存在，Stage E 链）
- 本节不改变 P0-1 比较语义（R4：精度唯一来源 = snapshot.decimals；G3.precision 不参与）

### 8.3 区间（range）token（R11 补回冻结）

区间 / 近似表达（如「2.0～2.5m」「约 2.3m」）**不做整体锚定**：

- 区间内每个数值 token 独立提取、独立绑定 fact_id（可各不相同）、独立值相等
- 禁止用区间整体作为一次锚定

（与 04_REPORT_IR §41.2.5 区间条款一致。）

### 8.4 Whitelist span containment（R11 补回冻结）

白名单豁免判定采用 **span containment**：

```
token 的 [char_start, char_end) ⊆ 某条登记白名单 pattern 的匹配 span
    → 豁免（跳过 Stage E 级联判定，仍输出诊断记录）
```

- **部分重叠（相交但不包含）→ 不豁免** → 按正常判定（UNANCHORED / UNKNOWN_UNIT 等）
- 示例：文本「GB 50292-2015」整体命中白名单 pattern，内部 numeric token `50292` / `2015` 的 span 均被包含 → 豁免
- 被豁免的 identifier token 同步豁免 BIND-7

---

## 9. FactDisplaySpec 重复唯一性（冻结）

### 9.1 重复 fact_type → precondition_error

`fact_display_spec` 以 `dict[fact_type] = FactDisplaySpecSnapshot` 形式加载。若加载源（JSON 数组）包含相同 fact_type 超过一次：

```
错误码：PRECONDITION.FACT_DISPLAY_SPEC_DUPLICATE_FACT_TYPE
错误信息："fact_display_spec contains duplicate fact_type: {fact_type}"
```

禁止 first-wins 静默处理。

---

## 10. ExpectedIssue matching（R1 恢复冻结）

### 10.1 Matching key（冻结）

```
matching key = (rule_id, fact_id)
```

- **severity 不参与 matching**（仅随双方记录，用于报告分级）
- **note 不参与 matching**

### 10.2 两阶段匹配（冻结）

| 阶段 | 条件 | 行为 |
| ---- | ---- | ---- |
| Phase 1: Exact | ExpectedIssue.fact_id != None：ActualIssue 与 ExpectedIssue 的 **(rule_id, fact_id) 完全相等** | 标记 matched，双方进入对应 consumed 集合 |
| Phase 2: Wildcard | ExpectedIssue.fact_id == None：同 rule_id 的任一**未被消费**的 ActualIssue 可匹配 | 标记 matched，双方进入对应 consumed 集合 |

### 10.3 One-to-one 双向保证（R1 恢复冻结）

必须**分别维护**两个消费集合：

```
consumed_expected_indices: set[int]   # 每个 ExpectedIssue 至多匹配一个 ActualIssue
consumed_actual_indices: set[int]    # 每个 ActualIssue 至多被一个 ExpectedIssue 消费
```

- **禁止只保护 ExpectedIssue 侧**
- 已消费的 ActualIssue 不得再参与任何阶段的匹配
- 已消费的 ExpectedIssue 不得再匹配任何 ActualIssue

### 10.4 ExpectedIssue 不内部 deduplicate（保持）

重复 ExpectedIssue（同 rule_id + fact_id 组合）→ Stage A precondition_error（§2.1 A-7）。matching 语义的恢复不改变该规则。

---

## 11. ActualIssue 构造语义（R2 恢复 identity 冻结 + 保持构造修复）

### 11.1 Identity 定义（恢复冻结，六元组，INV-17）

```
identity ≡ (
    issue_type,        # identity[0] — 问题类型（由产出该 issue 的 Rule 判定逻辑给出）
    location,          # identity[1] — 定位（渲染文本/表格坐标，如 rendered_segments[2]:char[15-20]）
    fact_id,           # identity[2] — 绑定事实（可 None）
    evaluation_id,     # identity[3] — 关联判定（可 None）
    occurrence_id,     # identity[4] — 关联出现次（可 None）
    table_aggregate,   # identity[5] — 表格聚合标识（可 None）
)
```

- identity 顺序严格固定，不得增删改字段
- **变更 identity 语义必须先在 07_DECISIONS.md 落新决策；禁止在契约修订中静默替换**
- v10 曾将 identity 静默替换为 (rule_id, severity, category, fact_id, source)，本轮回归修复恢复冻结六元组

### 11.2 build-only 构造守卫（B3 加固；identity / issue_id 保持 init=False）

```python
@dataclass
class ActualIssue:
    # ── Public constructor 接受的字段 ──
    rule_id: str                              # 归属 P0 Rule（不参与 identity）
    severity: str                             # error | warning（不参与 matching，仅报告分级）
    issue_type: str                           # identity[0]
    location: str                             # identity[1]
    message: str                              # 诊断信息（不参与 identity）

    fact_id: Optional[str] = None             # identity[2]
    evaluation_id: Optional[str] = None       # identity[3]
    occurrence_id: Optional[str] = None       # identity[4]
    table_aggregate: Optional[str] = None     # identity[5]

    # ── 以下字段不在 public constructor 中（init=False，保持 v10 修复） ──
    identity: tuple = field(init=False, repr=False)
    issue_id: str = field(init=False, repr=False)

    def __post_init__(self) -> None:
        # build-only 守卫（B3 修复）：
        # dataclass 生成的 __init__ 在字段赋值后必调用本方法
        #   → 任何 ActualIssue(...) 直接构造在此失败，「identity / issue_id 未初始化的
        #     半成品」不可能逸出；
        # build() 走 object.__new__ 绕过 __init__，不触发本方法。
        raise TypeError("ActualIssue is build-only; use ActualIssue.build(...)")
```

约束（B3 加固）：
- `ActualIssue(identity=...)` → `TypeError`（参数不在 `__init__` 签名）
- `ActualIssue(issue_id=...)` → `TypeError`（同上）
- `ActualIssue(rule_id=..., severity=..., ...)` **任何直接构造** → `TypeError`（`__post_init__` build-only 守卫）
- 唯一构造入口 = `build()`；build() 产物**同时具备**完整 identity 与 issue_id
- 字段语义、六元 identity（§11.1）、SHA-256 issue_id 规则（§11.4）**均未改动**——仅构造机制加固

### 11.3 build() classmethod（唯一构造入口，B3 加固为真 build-only）

```python
@classmethod
def build(
    cls,
    rule_id: str,
    severity: str,
    issue_type: str,
    location: str,
    message: str,
    fact_id: Optional[str] = None,
    evaluation_id: Optional[str] = None,
    occurrence_id: Optional[str] = None,
    table_aggregate: Optional[str] = None,
) -> "ActualIssue":
    # B3：build() 是唯一绕过 __post_init__ 守卫的入口
    # （object.__new__ 绕过 dataclass __init__），独家完成
    # 字段装配 + identity + issue_id，产物完整。
    instance = object.__new__(cls)
    object.__setattr__(instance, "rule_id", rule_id)
    object.__setattr__(instance, "severity", severity)
    object.__setattr__(instance, "issue_type", issue_type)
    object.__setattr__(instance, "location", location)
    object.__setattr__(instance, "message", message)
    object.__setattr__(instance, "fact_id", fact_id)
    object.__setattr__(instance, "evaluation_id", evaluation_id)
    object.__setattr__(instance, "occurrence_id", occurrence_id)
    object.__setattr__(instance, "table_aggregate", table_aggregate)

    # identity 由 build() 内部按 §11.1 冻结六元组构造，外部无法控制
    identity = (
        issue_type,
        location,
        fact_id,
        evaluation_id,
        occurrence_id,
        table_aggregate,
    )
    object.__setattr__(instance, "identity", identity)
    object.__setattr__(instance, "issue_id", _compute_issue_id(identity))
    return instance
```

### 11.4 issue_id 计算（INV-10，保持）

```python
def _compute_issue_id(identity: tuple) -> str:
    canonical_json = json.dumps(identity, sort_keys=True, ensure_ascii=False)
    return hashlib.sha256(canonical_json.encode("utf-8")).hexdigest()
```

约束：
- 禁止 Python `hash()`（结果不稳定、进程间不同）
- identity 六元组顺序严格固定（issue_type → location → fact_id → evaluation_id → occurrence_id → table_aggregate）

### 11.5 外部构造检查（B3 加固）

```python
# ❌ 禁止：外部传入 identity → TypeError（参数不在 __init__ 签名）
issue = ActualIssue(rule_id="ANCHOR.BINDING", identity=("x", "y"))

# ❌ 禁止：外部传入 issue_id → TypeError（参数不在 __init__ 签名）
issue = ActualIssue(rule_id="ANCHOR.BINDING", issue_id="abc123")

# ❌ 禁止：任何直接构造 → TypeError（__post_init__ build-only 守卫，B3）
issue = ActualIssue(
    rule_id="ANCHOR.BINDING",
    severity="error",
    issue_type="quantity_kind_mismatch",
    location="rendered_segments[2]:char[15-20]",
    message="token quantity_kind mismatch",
)

# ✅ 允许：唯一入口（产物同时具备完整 identity 与 issue_id）
issue = ActualIssue.build(
    rule_id="ANCHOR.BINDING",
    severity="error",
    issue_type="quantity_kind_mismatch",
    location="rendered_segments[2]:char[15-20]",
    message="token quantity_kind mismatch",
    fact_id="fact:component.K001.concrete_strength",
)
```

---

## 12. frozen_round（冻结，B2 修复异常行为一致性）

```python
from decimal import Decimal, ROUND_HALF_UP

def frozen_round(value: Any, decimals: int) -> float:
    """frozen_round — 工程四舍五入（Decimal(str(value)) + ROUND_HALF_UP）。

    异常行为（冻结，B2：代码与契约完全一致）：
        ① value is None              → ValueError
        ② value 为 NaN / +Inf / -Inf → ValueError
        ③ decimals 非 int 或 < 0     → TypeError
    多条件同时非法时，按 ①②③ 顺序抛出首个异常（确定性）。
    """
    # ①
    if value is None:
        raise ValueError("frozen_round: value must not be None")

    d = Decimal(str(value))      # 禁止 Decimal(value)（浮点二进制误差敏感）

    # ②
    if d.is_nan() or d.is_infinite():
        raise ValueError("frozen_round: value must be finite (NaN / Inf forbidden)")

    # ③
    if not isinstance(decimals, int) or decimals < 0:
        raise TypeError("frozen_round: decimals must be a non-negative int")

    # ④ 合法输入：数学语义不变（禁止 Python round()）
    if decimals == 0:
        quantize_str = "1"
    else:
        quantize_str = "0." + ("0" * decimals)
    return float(d.quantize(Decimal(quantize_str), rounding=ROUND_HALF_UP))
```

约束：
- 禁止 Python 内置 `round()`（banker's rounding，与工程四舍五入不符）
- 必须用 `Decimal(str(value))` 而非 `Decimal(value)`（后者对浮点二进制误差敏感）
- B2：原 v10 注释声明了异常行为但代码未实现——本轮补齐实现；数学语义（`Decimal(str(value))` + `quantize` + `ROUND_HALF_UP`）**零改动**

---

## 13. 7 P0 Rules（冻结，不增删）

| # | Rule ID | 含义 | 核心判定 |
| ---- | ---- | ---- | ---- |
| P0-1 | `VALUE.CONSISTENCY` | 报告数值与 Ground Truth 一致 | frozen_round(token_value, snapshot.decimals) == G3.value（精度链见下） |
| P0-2 | `TABLE.INTERNAL` | 表格内部数值自洽（合计、统计量与分项一致） | 程序计算值 vs 报告声明值（输入 = StructuredTable，§4） |
| P0-3 | `LOGIC.CONSISTENCY` | 判定逻辑与 expected_evaluations 一致 | Validator 不执行 Criterion grammar，按 evaluation_id 确定性查找并比对（§3.4） |
| P0-4 | `ANCHOR.BINDING` | 每个数字锚定正确 | Unit 解析 + quantity_kind 级联 + 标识同表绑定（§8） |
| P0-5 | `CONCLUSION.DIRECTION` | 结论方向与 GroundTruthConclusion.direction 一致 | 字符串精确匹配（case-sensitive） |
| P0-6 | `CONCLUSION.COVERAGE` | 结论覆盖全部检测项 | required_evaluation_ids ⊆ conclusion.covers[] |
| P0-7 | `EXPECTED.HIT` | ExpectedIssues 被全部命中 | 两阶段 matching（(rule_id, fact_id) exact + rule-level wildcard）+ 双向 one-to-one（§10） |

**P0_RULE_IDS（冻结，A-11 校验基准）**：

```python
P0_RULE_IDS: frozenset[str] = frozenset({
    "VALUE.CONSISTENCY",
    "TABLE.INTERNAL",
    "LOGIC.CONSISTENCY",
    "ANCHOR.BINDING",
    "CONCLUSION.DIRECTION",
    "CONCLUSION.COVERAGE",
    "EXPECTED.HIT",
})
```

- Stage A A-11：`trusted_input.applicable_rules ⊆ P0_RULE_IDS`，否则 precondition_error（R16）
- 与上表 7 条一一对应；P0 Rule 增删必须同步修改此集合（INV-1）

**P0-1 精度唯一来源链（R4 恢复冻结，INV-19）**：

```
Anchor.fact_id
    ↓ derive_fact_type()：parse_fact_id(fact_id) 的第一段 + "." + 第三段
fact_type
    ↓ fact_display_spec[fact_type]
FactDisplaySpecSnapshot
    ↓
decimals
```

- P0 numeric comparison **唯一使用 snapshot.decimals**
- **G3.precision 不参与 Step 7-E P0 numeric comparison**（仅为 Step 7-D 数据字段，INV-2 只读）
- snapshot 缺该 fact_type 条目 → precondition_error（missing display spec，v9 冻结保持）
- 禁止形成 `FactDisplaySpec.decimals` + `G3.precision` 双精度真相源

---

## 14. 测试契约

### 14.1 必须覆盖的反例

| # | 反例类别 | 测试重点 |
| ---- | ---- | ---- |
| T-1 | Counting | applicable_rule_count = 0 时 CaseStatus = NOT_EVALUABLE；applicable_rule_count < 7 时仍然 PASS |
| T-2 | Unit Registry | loaded mapping == 冻结 dict（缺 / 多 / 错 → precondition_error）；unit lexer 反例：12.5N/mm²→N/mm²/pressure、12.5kN/m²→kN/m²/pressure、32.5%→%/ratio、32.4psi / 32.4abc→UNKNOWN_UNIT（详见 §5.4 全表含 B1 新增三例） |
| T-3 | FactDisplaySpec 两层 | TemplateSpec dict 含 decimals+unit；Snapshot 只含 decimals；Step 7-E 不 import unit 字段 |
| T-4 | ActualIssue constructor | ActualIssue(identity=...) → TypeError；ActualIssue(issue_id=...) → TypeError；**任何 ActualIssue(...) 直接构造 → TypeError（B3）**；build() 正常且产物同时具备完整 identity 与 issue_id；identity 六元组内容与顺序正确 |
| T-5 | FactDisplaySpec 重复 | JSON 数组含重复 fact_type → precondition_error；dict 含唯一 key → 正常 |
| T-6 | Tokenizer candidate | ST001 numeric=NONE；S-3 numeric=3；H2O / A4 / ISO9001 / GB50292 numeric=NONE（前置字母阻断） |
| T-7 | renderable_fact_types | frozenset[str] 正常；list[str] → TypeError；空集合合法；非法 fact_type → precondition_error；allowed-to-render 语义（actual-rendered 不进入 TrustedInput） |
| T-8 | 冻结正则 | NUMERIC_TOKEN_RE 和 IDENTIFIER_TOKEN_RE 逐段验证所有反例（含单位组完整捕获） |
| T-9 | Matching | severity / note 不参与匹配；(rule_id, fact_id) exact + rule-level wildcard；双向 one-to-one（单条 ActualIssue 被两条 ExpectedIssue 争用 → 仅第一条成功） |
| T-10 | identity / issue_id | 相同 identity → 相同 issue_id；六元组任一字段变化 → issue_id 变化 |
| T-11 | Precision truth source | G3.precision 与 snapshot.decimals 不一致时，P0-1 按 snapshot.decimals 判定；snapshot 缺条目 → precondition_error |
| T-12 | Stage B | trusted_input.applicable_rules 之外无法使规则变为 applicable；∉ → NOT_EVALUABLE |
| T-13 | ExpectedEvaluations | 重复 evaluation_id → precondition_error；多 Evaluation 全部可按 id 确定性查找 |
| T-14 | Range / Whitelist | 区间逐 token 锚定；span 部分重叠不豁免；完全包含才豁免 |
| T-15 | frozen_round 异常（B2） | frozen_round(None, 2) → ValueError；frozen_round(float("nan"), 2) → ValueError；frozen_round(float("inf"), 2) → ValueError（-Inf 同）；frozen_round(1.2, -1) → TypeError；frozen_round(2.345, 2) → 2.35 |
| T-16 | Unit 完整 token 边界（B1） | 32.4m3 → unit=`m3` → UNKNOWN_UNIT；32.4N/mm2 → unit=`N/mm2` → UNKNOWN_UNIT；12.5N/mm²x → unit=`N/mm²x` → UNKNOWN_UNIT；三者均不得截断、不得 unit=None、不得被 lookahead 吞掉 numeric token |
| T-17 | applicable_rules ⊆ P0_RULE_IDS（R16） | applicable_rules 含未知 rule_id（如 "NOT_A_P0"）→ precondition_error；不被计入 applicable_rule_count；不被静默删除；trusted_input.applicable_rules 不被修改 |
| T-18 | Numeric Binding（B4） | Case A：numeric token 有合法 fact_id → 正常进入 P0-1 比对；Case B：无 AnchorDeclaration → ANCHOR.UNANCHORED → FAIL；Case C：AnchorDeclaration.fact_id 非法 → ANCHOR.INVALID_FACT_ID → FAIL；Case D：同一 numeric token 同时绑定两个不同 fact_id → ANCHOR.DUPLICATE_BINDING → FAIL；Case E：numeric token 完全处于 whitelist span → 豁免 numeric binding；Case F：token 与 AnchorDeclaration 仅部分重叠 → 不算绑定（span 规则）→ UNANCHORED；Case G：已绑定但 token value 与 G3 按 snapshot.decimals 不相等 → 走 P0-1 → FAIL |

### 14.2 实现阶段划分

```
Phase 1: 契约自检（contracts.py）
    → 数据结构 + 枚举 + 构造函数语义

Phase 2: 词法层（numeric_parser.py + identifier_parser.py）
    → regex + tokenize + overlap 消解 + unit 完整捕获

Phase 3: 单元层（unit_registry.py + anchor.py）
    → UNIT_TO_QUANTITY exact equality + quantity_kind 级联

Phase 4: 比对层（compare.py + actual_issues.py）
    → frozen_round + 双向 one-to-one matching + identity 构造

Phase 5: 状态机（accuracy_validator.py）
    → 五阶段 + CaseStatus 聚合
```

---

## 结尾判定

**回归恢复（R1–R12，已完成）**：

| # | 回退项 | 恢复位置 |
| ---- | ---- | ---- |
| 一 | ExpectedIssue matching → (rule_id, fact_id) + 双向 one-to-one | §10 |
| 二 | ActualIssue identity → 冻结六元组 | §11 |
| 三 | unit lexer 完整捕获注册表全部键 | §5.1 / §5.4 |
| 四 | P0-1 精度唯一来源 → Snapshot.decimals | §13 |
| 五 | Stage B → 只读 membership test | §2.2 |
| 六 | expected_evaluations → dict 多 Evaluation 结构 | §3 / §3.4 / A-10 |
| 七 | Unit Registry → exact equality（hash 声明删除） | §3.2 / A-4 |
| 八 | identifier → scope_class 映射表删除 | §6.3 |
| 九 | renderable_fact_types → allowed-to-render | §3.3 |
| 十 | SystemOutput → 结构表达力恢复 | §4 |

补充恢复：range 逐 token 锚定（§8.3）、whitelist span containment（§8.4）、§5.3 / §6.2 candidate 对齐、BIND-7 标识同表绑定。

**执行层阻塞项闭合（B1–B3）**：

| # | 阻塞项 | 修复位置 | 状态 |
| ---- | ---- | ---- | ---- |
| B1 | Unit Lexer 完整 token 边界 | §5.1 / §5.4（unit-start / unit-cont 文法 + U-5 + 3 组新反例） | ✅ 闭合 |
| B2 | frozen_round 行为一致性 | §12（异常检查补齐，数学语义零改动） | ✅ 闭合 |
| B3 | ActualIssue build-only | §11（__post_init__ 守卫 + object.__new__ build） | ✅ 闭合 |

**数字绑定闭环（B4，本轮）**：

| # | 阻塞项 | 修复位置 | 状态 |
| ---- | ---- | ---- | ---- |
| B4 | Numeric Token → fact_id 显式绑定缺失 | §8.1 BIND-8 / §8.2 正式绑定与判定链 / INV-24 / T-18 | ✅ 闭合 |

**旧契约遗漏恢复与扫描（本轮）**：

| 项 | 内容 | 位置 | 状态 |
| ---- | ---- | ---- | ---- |
| R16 | applicable_rules ⊆ P0_RULE_IDS；未知 / 非 7 P0 rule_id → precondition_error；不静默删除、不自动补入、不修改 trusted_input | §2.1 A-11 / §2.2 / §13 / INV-23 / T-17 | ✅ 闭合 |

旧契约遗漏扫描（5 项）+ B4 轮新增候选（第 6 项）：以下各项无法从磁盘证据确证为**确定性回归 / 确定性冲突**，一律标记 **REGRESSION CANDIDATE / EXECUTABILITY CANDIDATE**、不直接修改（第 6 项按 B4 轮指令明确不重新设计 TableCell）；待冻结登记确认——确认任一成立即回退 NOT READY 并在下轮恢复：

| # | 扫描项 | 是否为既有冻结规则 | v10 当前状态 | 是否确定回归 | 是否修改 | 最终结论 |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| 1 | case.validate() 纳入 Stage A | 指认为既有约束；v9 原文缺失，无法独立证实 | Stage A 无整体 case 校验；仅 A-7 / A-8 / A-10 局部覆盖（ExpectedIssue 去重、fact_id 格式、evaluation_id 去重）。Step 7-D `case.validate()` 覆盖 8 类校验（meta / G3 值与量纲 / G2 / structure / conclusion / known_issues / expected_issues / G3⊆G2），且 `from_dir()` 不自动调用 | 否——REGRESSION CANDIDATE | 否 | 待冻结登记确认；确认成立 → 下轮以 A 项检查恢复（不重设计） |
| 2 | whitelist precondition（登记结构：pattern / reason / scope / confirmed_by） | 指认为既有约束；无法独立证实 | §8.4 仅定义匹配语义（span containment）；Stage A 无 whitelist 登记结构校验 | 否——REGRESSION CANDIDATE | 否 | 待冻结登记确认 |
| 3 | TrustedTableSpec precondition | 指认为既有约束；无法独立证实 | trusted_table_specs 出现在 TrustedInput 但 TableSpecSnapshot 类型未定义；Stage A 无对应检查 | 否——REGRESSION CANDIDATE | 否 | 待冻结登记确认；另注：类型未定义本身构成 P0-2 可执行性缺口（与 B 轮 SystemOutput 同类），确认后一并处理 |
| 4 | renderable_fact_types ⊆ fact_display_spec.keys() | 指认为既有约束；无法独立证实 | 存在**部分替代**：A-9（元素 ∈ CDM fact_type 注册表）+ P0-1 惰性检查（snapshot 缺条目 → precondition_error，标注 v9 冻结保持） | 否——REGRESSION CANDIDATE（部分覆盖 ≠ 明确替代） | 否 | 待确认；若确认，需先裁决「Stage A 集合级校验」与「惰性检查」的替代关系 |
| 5 | G3.unit ∈ UNIT_TO_QUANTITY（或 None）合法性 | 指认为既有约束；无法独立证实（Step 7-D validate_ground_truth_fact 亦不检查 unit） | A-8 仅校验 fact_id 格式；G3.unit 无注册表合法性检查 | 否——REGRESSION CANDIDATE | 否 | 待冻结登记确认 |
| 6 | TABLE-NUMERIC-BINDING（TableCell 级 numeric token 逐 token 绑定） | 04_REPORT_IR §41.2.1 明确将 Table 单元格 / 表标题 / 表注纳入锚定范围（冻结依据明确） | TableCell.fact_id 为 cell 级单值：单 numeric token 的 cell 可绑定；一个 cell 含多个 numeric token（尤其各属不同 fact）时无法逐 token 绑定（B4 仅闭环 rendered_segments 侧，§8.1 BIND-8 作用域注明） | 否——REGRESSION CANDIDATE / EXECUTABILITY CANDIDATE | 否（B4 轮指令：不重新设计 TableCell） | 待裁决；若确认需逐 token 绑定 → 下轮以最小扩展恢复（cell 内 token 级绑定表达），本轮不擅自设计 |

**内部交叉检查（B1–B3 + B4 修复轮，零新冲突）**：

| 检查项 | 结果 | 位置 |
| ---- | ---- | ---- |
| 7 P0 完全不变 | ✅ | §13（7 条未增删；B4 仅补 BIND-8，仍属 P0-4） |
| Step 7-D 完全未修改 | ✅ | 本轮零改动（仅契约文档） |
| ActualIssue 六元 identity 完全不变 | ✅ | §11.1（B3 仅改构造机制；B4 未触碰） |
| ExpectedIssue matching 仍为 (rule_id, fact_id) | ✅ | §10.1（B4 未触碰） |
| applicable_rules 仍只来自 TrustedInput | ✅ | §2.2（B4 未触碰） |
| renderable_fact_types 仍为 allowed-to-render | ✅ | §3.3（B4 未触碰） |
| precision 唯一来源仍为 Snapshot.decimals | ✅ | §13 / §8.2（B4 强化：token→fact_id→fact_type→snapshot.decimals→P0-1 完整链） |
| UNIT_TO_QUANTITY 仍 exact equality | ✅ | §2.1 A-4（B4 未触碰） |
| identifier 仍禁止 scope_class 推断 | ✅ | §6.3（B4 未触碰） |
| range / whitelist span containment 保持 | ✅ | §8.3 / §8.4（B4 未触碰；BIND-8 复用同一 span 规则） |
| ST001 / S-3 行为保持 | ✅ | §5.3（B4 未触碰标识 regex） |
| ±3.2MPa / ± 3.2MPa 保持 | ✅ | §5.1 `(±\s*)?` + BIND-2（B4 未触碰 ± 组） |
| P0-4 numeric binding 闭环 | ✅ | §8.1 BIND-8 + §8.2 判定链（B4 本轮补齐） |
| P0-4 identifier binding 闭环 | ✅ | §8.1 BIND-7（R12 补齐，B4 未触碰） |
| P0-4 whitelist exemption 闭环 | ✅ | §8.4（BIND-8/BIND-7 同样依赖 whitelist 豁免） |
| P0-4 range token 行为 | ✅ | §8.3（区间逐 token 独立锚定，B4 未触碰） |
| P0-4 display precision 来源 | ✅ | §8.2 → §13（snapshot.decimals，B4 强化链） |
| P0-4 G3 comparison | ✅ | §8.2（unit 级联 + P0-1 value 比较，B4 闭环输入） |

**READY 条件核对**：

| 条件 | 结果 |
| ---- | ---- |
| 所有回退全部恢复 | ✅（R1–R12） |
| B1–B3 全部闭合 | ✅ |
| B4 numeric-token binding 完整闭环 | ✅（BIND-8 + §8.2 判定链 + INV-24 + T-18 Case A–G） |
| applicable_rules ⊆ P0_RULE_IDS 修复完成 | ✅（R16 / A-11 / INV-23 / T-17） |
| 无未处理的确定性回归 | ✅（遗漏扫描 6 项均为 CANDIDATE，非确定性回归；第 6 项 TABLE-NUMERIC-BINDING 已按 B4 轮指令标记为 EXECUTABILITY CANDIDATE，不重新设计 TableCell） |
| 所有 Unit lexer 反例闭合 | ✅（8/8，§5.4，含 B1 新增三例） |
| 所有 truth source 唯一 | ✅（precision / applicability / unit registry / renderable 各单一来源） |
| SystemOutput schema 可执行 | ✅（RenderedSegment / AnchorDeclaration / StructuredTable / TableCell，§4；TableCell 多 numeric token 场景已登记候选） |
| 没有重新设计其他部分 | ✅（B4 仅补 BIND-8 + §8.2 判定链；不新增 P0 Rule、不改 AnchorDeclaration 结构、不改 ActualIssue identity / ExpectedIssue matching / Unit Registry / identifier scope / ± 组 / 标识 regex / frozen_round 数学语义 / 7 P0 / Step 7-D 只读 / known_issues 只读 / ExpectedIssue 不去重 / 不执行 Criterion grammar / 不做 build gate / 不 import Step 7-F+ / 不引入 LLM 全部保持） |

B4 修复期间本文档状态为 NOT READY；B4 闭合、P0-4 完整闭环、交叉检查零新冲突后，最终状态如下。

**Step 7-E Coding Contract v10 = READY**
