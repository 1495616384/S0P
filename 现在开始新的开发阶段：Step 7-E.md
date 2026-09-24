这是一个典型的设计审查任务——Step 7-E 的 Coding Contract。用户明确要求我：

1. 先阅读现有文档和代码
2. 只做设计审查，不写代码
3. 回答9个核心边界问题

让我先加载相关文件来理解当前状态。我需要并行读取多个设计文档和代码文件。

让我先规划 Todo 列表，然后开始系统性地阅读。

好的，让我开始系统性地阅读设计文档和代码。首先读取 05_EVALUATION.md，因为它是 Accuracy Validator 的核心规则来源。然后并行读取其他文档。

好，现在我需要并行读取所有关键文件。让我先读取设计文档（05_EVALUATION.md 是最核心的）和现有代码。

好，所有5份设计文档已经读完。现在需要读取当前的代码实现——Step 7-D 完成的 eval 模块和 Step 7-B/7-C 的 CDM 和 domain 实现。让我继续读代码。

好的，现在让我并行读取所有代码文件——src/eval/, src/cdm/ 和 tests/ 目录。

好，eval 模块全部读完了。现在需要读 Step 7-B/7-C 的核心 CDM 类型和 validator——特别是 Fact ID 格式、registry、domain 对象这些，因为 Accuracy Validator 会用到它们。让我继续读 CDM 的核心文件。

好的，现在让我读取 CDM 核心文件——types.py、registry.py、id.py、validate.py 和 domain.py。

好，所有代码也读完了。现在我已经完整读完了：

**设计文档**：
- 02_ARCHITECTURE.md v0.3（852行）
- 03_CANONICAL_DATA_MODEL.md v0.3.1（1872行）
- 04_REPORT_IR.md v0.3.1.1（1913行）
- 05_EVALUATION.md v0.3（423行）
- 07_DECISIONS.md v0.1（417行）

**代码实现**：
- Step 7-B：src/cdm/ (types, registry, id, validate, domain)
- Step 7-D：src/eval/ (types, validator, enums) + tests/eval/

现在我可以开始分析9个核心边界问题。让我调用 AdvisorTool 来获取战略性指导，然后再形成 Coding Contract。



*正在请求专家建议，优化解决方案，该过程耗时可能较长，请耐心等待...*

好，全部资料已消化，Advisor 关键建议已吸收。现在输出 Step 7-E Coding Contract，按9个核心边界问题逐一回答。

---

# Step 7-E Coding Contract — Accuracy Validator

## 先确立命名和概念基线

在回答具体问题前，必须先确立几个命名，否则后文全部会混乱。本 Contract 冻结以下命名，后续代码实现必须严格遵守。

### 三层 Issue 概念（绝对不能混淆）

| 命名 | 来源 | 含义 | 谁创建 | 用途 |
|---|---|---|---|---|
| **System Issue** | `src/eval/` 新增 `actual_issues.py` | 验证器实际发现的问题 | Accuracy Validator（本模块） | 与 `expected_issues` 对比、构建 IssueList、驱动 P0 判定 |
| **ExpectedIssue** | Step 7-D 已有 `eval.types.ExpectedIssue` | EvaluationCase 固件中登记的"系统应该发现的问题" | 逆向链路（构造期） | P0 `EXPECTED.HIT` 规则的判定依据 |
| **KnownIssue** | Step 7-D 已有 `eval.types.KnownIssue` | 历史报告中的已知错误（原文错、正确值已确认） | 逆向链路（构造期） | **只在比对时做值替换**，绝不放宽 P0 闸门 |

**关键 Advisor 洞察**：System Issue 有两条独立产生路径：
1. **路径 A**（`EXPECTED.HIT`）：Validator 检查 System IssueList 中是否存在 ExpectedIssue 要求的条目
2. **路径 B**（`VALUE.CONSISTENCY` 等 6 条）：Validator 对比 System Output 与 GroundTruth 产生的 Issue

这两条路径的输出类型相同（都是 ActualIssue），但产生逻辑完全独立。

---

## 1. 输入边界

### Accuracy Validator 接收什么

```text
输入（必须、不可缺失）:
├── EvaluationCase          —— Step 7-D 产物，含 meta + ground_truth + known_issues + expected_issues
├── SystemOutput            —— 系统运行时产物的协议对象（Step 7-E 新增，见下文 §A）
│   ├── report_ir: Document          —— Report IR（用于 Ref、结论 covers[]、表格结构检查）
│   ├── rendered_segments: list[RenderedSegment] —— 渲染产物片段（用于数字锚定 §41.2 R5）
│   │   └── 每个 RenderedSegment:
│   │       ├── segment_type: narrative_prose | table_cell | table_title | table_note | figure_caption
│   │       ├── text: str             —— 渲染后的纯文本
│   │       ├── anchors: list[Anchor]  —— 运行时声明的 anchor（dict-like: token→fact_id）
│   │       └── location: str         —— 在报告中的定位（Section.semantic + Block.id + 坐标）
│   ├── system_issue_list: list[SystemReportedIssue]  —— 生成管线自己报的问题（Step 6 validate 产出）
│   └── computed_aggregate_values: dict[str, Any] —— 程序计算的派生量（均值、合计、最大值等）
└── RuleRegistry            —— Step 7-E 定义的 rule_id 注册表（frozenset）

中间产物：
├── ActualIssue[]           —— 本 Validator 产生
├── RuleResult[]            —— 本 Validator 产生（每条 P0 规则一个结果）
└── ExpectedIssueCoverage   —— 本 Validator 产生

最终输出：
└── ValidationResult        —— 本 Validator 最终产出（见 §9）
```

### 绝对不做的事

- **Validator 不创建 GroundTruth**：GroundTruth 由逆向链路冻结为 JSON 固件，Validator 只读
- **Validator 不修改 EvaluationCase**：只读访问
- **Validator 不生成 SystemOutput**：SystemOutput 由 Pipeline Orchestrator 在调用 Validator 前准备好
- **Validator 不实现 parser / renderer**：只定义接口契约，解析渲染由调用方负责，Validator 消费解析好的 RenderedSegment

### SystemOutput 的接口契约（本 Step 只定契约）

```python
@dataclass
class SystemOutput:
    """Step 7-E Accuracy Validator 的输入协议对象。

    本对象只携带已解析好的结构化数据，不做解析。
    解析/渲染由 Pipeline Orchestrator 在 Step 6 调用 Accuracy Validator 前完成。
    """
    # Report IR —— 用于 Ref 检查、结论 covers[]、表格结构
    report_ir: AnyDocumentIR                    # Report IR Document 对象（具体类型在 Step 7-F IR 实现时冻结）

    # 渲染产物片段 —— 用于数字锚定校验（§41.2 R5 必须操作渲染产物）
    rendered_segments: list[RenderedSegment]

    # 生成管线自己报的问题 —— 用于 EXPECTED.HIT 规则
    system_reported_issues: list[SystemReportedIssue]

    # 程序计算的派生量 —— 用于 VALUE.CONSISTENCY（派生量先验正确性）和 TABLE.INTERNAL
    computed_aggregate_values: dict[str, Any]
```

---

## 2. Validator 与 Step 7-D 的边界

| 职责 | Step 7-D | Step 7-E |
|---|---|---|
| EvaluationCase 结构正确性 | ✅ 负责 | ❌ 依赖已通过 |
| 文件加载 / 必需目录检查 | ✅ `from_dir()` 负责 | ❌ 不做 IO |
| G3 ⊆ G2 引用完整性 | ✅ `validate_case()` 负责 | ❌ 假设已成立 |
| 跨文件一致性（case_level ↔ G3 等） | ✅ 负责 | ❌ 假设已成立 |
| **System Output vs GroundTruth 一致性** | ❌ 不涉及 | ✅ 本模块全部职责 |
| **P0 判定规则执行** | ❌ 不涉及 | ✅ 本模块全部职责 |
| **expected_issues 命中率检查** | ❌ 不涉及 | ✅ `EXPECTED.HIT` 规则 |

**边界线**：
- Step 7-D 验证的是"**评测用例本身是否结构完整、逻辑自洽**"
- Step 7-E 验证的是"**系统产出的报告是否与评测用例的标准答案一致**"

Step 7-E 不调用 Step 7-D 的 `validate()` 函数，但会依赖 Step 7-D 已经执行过的结构完整性保证。如果传入了一个 Step 7-D 验证不通过的 EvaluationCase，Step 7-E 应该在入口处直接报错拒绝执行，而不是尝试校验。

**设计规则**：Step 7-E 入口检查 `len(case.validate()) == 0`，若失败立即返回 `ValidationResult(pass=False, rule_results=[], actual_issues=[], error=...)`，不尝试 P0 规则执行。

---

## 3. 七条 P0 判定规则程序化规格

以下 7 条规则严格依据 05_EVALUATION.md §6，不新增不删除。每条规则独立设计。

### Rule 注册表（冻结，第一阶段 7 条）

```python
P0_RULE_IDS: frozenset[str] = frozenset({
    "VALUE.CONSISTENCY",       # §6.1 数值一致性
    "TABLE.INTERNAL",          # §6.2 表内自洽
    "LOGIC.CONSISTENCY",       # §6.3 判定逻辑一致性
    "ANCHOR.BINDING",          # §6.4 数字锚定
    "CONCLUSION.DIRECTION",    # §6.5 结论方向一致性
    "CONCLUSION.COVERAGE",     # §6.6 结论覆盖度
    "EXPECTED.HIT",            # §5.2 + §7 expected_issues 命中率
})
```

**规则编号说明**：用 `PREFIX.SUFFIX` 格式，PREFIX 是规则的大类名，SUFFIX 是具体规则。这种命名方式让规则 id 在日志、metrics、known_issues 登记中可直接按大类过滤。

### 3.1 VALUE.CONSISTENCY — 数值一致性

| 维度 | 规格 |
|---|---|
| **rule_id** | `VALUE.CONSISTENCY` |
| **输入** | SystemOutput.report_ir 的 Ref 段 → 取 Fact.value 与 unit + GroundTruth.facts[]（G3） |
| **输出** | ActualIssue[] |
| **pass 条件** | 对每个 G3 fact_id：<br>1. System 的 Ref 段**存在该 fact_id 的引用**<br>2. `round(system_value, precision) == round(gt_value, precision)` 按 G3 声明的 precision<br>3. `system_unit == gt_unit`（单位必须精确相等）<br>4. `system_quantity_kind == gt_quantity_kind` |
| **fail 条件** | 上述任一条不满足 |
| **Issue 类型** | `VALUE.MISMATCH`（值不等）/ `UNIT.MISMATCH`（单位不等）/ `QUANTITY.MISMATCH`（量纲不等）/ `VALUE.MISSING`（G3 存在但 System 未引用） |
| **severity** | `error` |
| **P0** | ✅ |
| **容差** | **禁止**，不设容差百分比。仅 precision 控制舍入（由 G3 显式声明） |
| **降级** | 仅 `case_level != gold` 时，规则结果仍产生但**不触发门禁失败**。gold 用例必须 100% 通过 |
| **known_issues 参与** | 比对前执行值替换：若 `KnownIssue.fact_id == G3.fact_id` 且 `KnownIssue.issue_type == "fact_value_error"`，则用 `KnownIssue.correct_value` 替换 G3 的 original_value 参与比对。**known_issues 不用于放宽 pass 条件，只用于纠正 GroundTruth 本身的历史数据错误** |

### 3.2 TABLE.INTERNAL — 表内自洽

| 维度 | 规格 |
|---|---|
| **rule_id** | `TABLE.INTERNAL` |
| **输入** | SystemOutput.rendered_segments 中 type=table_cell 的数值 + SystemOutput.computed_aggregate_values（程序计算的合计/均值/最大最小） |
| **输出** | ActualIssue[] |
| **pass 条件** | 对每张表：<br>1. 合计行 ≈ 分项之和（容差由 precision 控制）<br>2. 最大值 ≥ 均值 ≥ 最小值<br>3. 计数行 == 明细条数 |
| **fail 条件** | 上述任一条不满足 |
| **Issue 类型** | `TABLE.SUM_MISMATCH` / `TABLE.ORDER_VIOLATION` / `TABLE.COUNT_MISMATCH` |
| **severity** | `error` |
| **P0** | ✅ |
| **容差** | 程序计算的派生量（合计/均值/统计量）允许相对误差 ≤ 1e-9（§6.1 规则） |
| **降级** | 无降级，gold/silver 都要过 |
| **known_issues 参与** | 不直接参与——known_issues 处理的是 G3 fact 值，本表内自洽是结构性检查 |

### 3.3 LOGIC.CONSISTENCY — 判定逻辑一致性

| 维度 | 规格 |
|---|---|
| **rule_id** | `LOGIC.CONSISTENCY` |
| **输入** | System 产生的 Evaluation / Criterion / Fact 值（来自 SystemOutput.report_ir 的 assertion 段 Ref） + GroundTruthConclusion |
| **输出** | ActualIssue[] |
| **pass 条件** | 每个 Evaluation 的判定结果（qualified / unqualified / level_A / ...）与限值规则一致：<br>推定值低于限值 → 合格<br>推定值高于限值 → 不合格<br>（判定逻辑由程序按 Criterion.rule 执行，本 Validator 只检查输出与输入的逻辑一致性，不检查限值本身是否正确） |
| **fail 条件** | 判定结论与输入值违反 Criterion.rule |
| **Issue 类型** | `LOGIC.MISMATCH` |
| **severity** | `error` |
| **P0** | ✅ |
| **容差** | 无——判定逻辑是布尔的 |
| **降级** | 无降级 |
| **known_issues 参与** | 若 `KnownIssue.issue_type == "conclusion_error"`，比对前先做方向替换 |

### 3.4 ANCHOR.BINDING — 数字锚定

| 维度 | 规格 |
|---|---|
| **rule_id** | `ANCHOR.BINDING` |
| **输入** | SystemOutput.rendered_segments（**渲染后产物**，§41.2 R5 必须检查渲染文本而非 IR 声明） + SystemOutput.system_issue_list（生成管线报的锚定问题） + EvaluationCase 白名单（后续 Step 7-F 补充，当前 case 结构中预留） |
| **输出** | ActualIssue[] |
| **pass 条件** | 对每个 rendered_segment：<br>1. 提取全部数值 token 和实例标识 token（构件编号/测点编号等）<br>2. 每个 token 要么：(a) 显式绑定 fact_id 且 `round(token_value, precision) == round(fact_value, precision)`<br>要么：(b) 命中登记制白名单（模式级，由评审方确认）<br>3. **不允许**「值存在于全文 Fact 值集合」作为兜底（D-017 明确删除） |
| **fail 条件** | 任一 token 既未正确绑定 fact_id 也未命中白名单 |
| **Issue 类型** | `ANCHOR.UNBOUND`（未绑定）/ `ANCHOR.WRONG_FACT`（绑定了错误 fact_id 但值恰好匹配）/ `ANCHOR.VALUE_MISMATCH`（绑定正确 fact_id 但值不等） |
| **severity** | `error` |
| **P0** | ✅ |
| **容差** | 由 precision 控制（G3 声明的 precision） |
| **降级** | 无降级 |
| **白名单位置** | EvaluationCase 后续需在 `ground_truth/` 或顶层新增 `whitelist.json`（Step 7-E 先预留接口） |

### 3.5 CONCLUSION.DIRECTION — 结论方向一致性

| 维度 | 规格 |
|---|---|
| **rule_id** | `CONCLUSION.DIRECTION` |
| **输入** | SystemOutput.report_ir 的 conclusion assertion 段 → 解析出 direction + GroundTruthConclusion.direction（G4） |
| **输出** | ActualIssue[] |
| **pass 条件** | `system_direction == gt_direction`（**枚举精确相等**，不做语义相似度，§6.5 + D-023） |
| **fail 条件** | 枚举值不同 |
| **Issue 类型** | `CONCLUSION.DIRECTION_MISMATCH` |
| **severity** | `error` |
| **P0** | ✅ |
| **容差** | 无 |
| **降级** | 若 silver 用例的 G4 direction 只能从报告正文读出（D-023 规则），自动降级为 bronze → 不执行此规则 |
| **known_issues 参与** | 若 `KnownIssue.issue_type == "conclusion_error"` 且 direction 错误，比对前替换 correct_value |

### 3.6 CONCLUSION.COVERAGE — 结论覆盖度

| 维度 | 规格 |
|---|---|
| **rule_id** | `CONCLUSION.COVERAGE` |
| **输入** | SystemOutput.report_ir 的 conclusion assertion 段 → 取 `covers[]`（§42.1 + D-026） + GroundTruthConclusion.required_evaluation_ids（G4） |
| **输出** | ActualIssue[] |
| **pass 条件** | 对每个 `required_evaluation_ids`：至少被一个 conclusion assertion 的 `covers[]` 引用 |
| **fail 条件** | 存在 required evaluation_id 未被任何 covers[] 引用 |
| **Issue 类型** | `CONCLUSION.COVERAGE_GAP`（未覆盖的 evaluation_id 列表）/ `CONCLUSION.COVERS_MISSING`（结论段根本没有 covers[] 字段） |
| **severity** | `error` |
| **P0** | ✅ |
| **容差** | 无 |
| **降级** | 无降级 |
| **known_issues 参与** | 不直接参与——覆盖度是结构检查，不是值检查 |

### 3.7 EXPECTED.HIT — expected_issues 命中率

| 维度 | 规格 |
|---|---|
| **rule_id** | `EXPECTED.HIT` |
| **输入** | EvaluationCase.expected_issues[] + SystemOutput.system_reported_issues[]（**生成管线自己报的问题，不是本 Validator 产生的**） |
| **输出** | ActualIssue[]（每一条未命中的 ExpectedIssue 产生一个 ActualIssue） |
| **pass 条件** | **每一条** ExpectedIssue 都能在 system_reported_issues 中找到匹配项（匹配规则见 §7） |
| **fail 条件** | 存在 ExpectedIssue 未被任何 system_reported_issue 命中 |
| **Issue 类型** | `EXPECTED.NOT_HIT` |
| **severity** | 与 ExpectedIssue.severity 对齐（若 ExpectedIssue.severity="error" 则 ActualIssue.severity="error"） |
| **P0** | ✅（05_EVAL §7 明确"该报错时没报错是一等缺陷"） |
| **容差** | **零容差**——每条 ExpectedIssue 必须命中 |
| **降级** | gold 经济门禁（D-024）：gold < 5 时不阻断构建 |
| **known_issues 参与** | known_issues 只处理 GroundTruth 的值替换，不用于 EXPECTED.HIT 的匹配放宽 |

---

## 4. ActualIssue 正式建模

本 Step 新增的数据类型，与 Step 7-D 的 ExpectedIssue 和 KnownIssue **完全独立**，三者字段重合度最低化。

```python
@dataclass
class ActualIssue:
    """Accuracy Validator 实际发现的问题（Step 7-E 新增）。

    与 ExpectedIssue（答案钥匙）和 KnownIssue（历史排除）严格隔离。
    
    字段设计原则：问题定位优先、评测可追溯优先、JSON 可序列化。
    """
    # ── 唯一标识（运行时生成，不持久化） ──
    issue_id: str                    # 格式: "{rule_id}#{hashable_key}"，唯一

    # ── 来源定位 ──
    rule_id: str                     # VALUE.CONSISTENCY / ANCHOR.BINDING / ...
    severity: str                    # error | warning（来自 P0_RULES[rule_id].severity）
    issue_type: str                  # VALUE.MISMATCH / ANCHOR.UNBOUND / EXPECTED.NOT_HIT / ...
    fact_id: Optional[str] = None    # 若涉及具体 Fact
    evaluation_id: Optional[str] = None  # 若涉及 Evaluation（coverage 规则）

    # ── 报告定位 ──
    location: Optional[str] = None   # Section.semantic + Block.id + 渲染文本位置（"semantic:inspection_result/ph-2/assertion"）
    token: Optional[str] = None      # 若涉及具体 token（数字锚定规则）

    # ── 问题描述 ──
    expected: Any = None             # 期望值（来自 GroundTruth / 白名单）
    actual: Any = None               # 系统实际值（来自 SystemOutput）
    message: str = ""                # 人类可读的一句话描述
```

**字段设计决策**：
- **没有** `provenance` 字段——ActualIssue 只关心"是什么问题"，不关心问题从哪来的（那是 KnownIssue 负责的历史溯源）
- `issue_type` 比 `rule_id` 粒度更细：一条 rule 可以产生多种 issue_type（如 VALUE.CONSISTENCY 可以产生 VALUE.MISMATCH / UNIT.MISMATCH / QUANTITY.MISMATCH 三种）
- `expected` 和 `actual` 类型是 `Any`——因为 issue 类型不同时值类型也不同（数值对比是 float，方向对比是 str，覆盖度对比是 list）

---

## 5. 数字锚定（ANCHOR.BINDING）

这是 Step 7-E 最复杂的规则，严格依据 05_EVALUATION.md §6.4 + 04_REPORT_IR.md §41.2 + 07_DECISIONS.md D-017 / D-018 / D-019。

### 输入来源（必须是渲染产物）

```text
SystemOutput.rendered_segments
    └── 每个 RenderedSegment 包含：
        ├── text: str              —— 渲染后的纯文本
        ├── anchors: list[Anchor] —— 运行时声明的 anchor（dict-like: {"token": "2.3m", "fact_id": "fact:defect.C001.length"}）
        ├── location: str         —— 定位
        └── segment_type: enum    —— NARRATIVE_PROSE / TABLE_CELL / TABLE_TITLE / TABLE_NOTE / FIGURE_CAPTION / TABLE_FIGURE_UNIT
```

### 判定流程

```text
对每个 RenderedSegment：
    1. 提取所有 token（含数值 token 和实例标识 token）
       - 数值 token 正则：(\d+(\.\d+)?)\s*([a-zA-Zμ]+)?   —— 捕获值 + 可选单位
       - 实例标识 token：构件编号（K\d+ / C\d+）、测点编号、样品编号等（正则可配置）
    
    2. 对每个 token：
       a. 先查白名单（登记制，模式级）
          - 若命中 → 通过（白名单模式如 r"表\s*\d+-\d+" / r"第\s*\d+\s*章" / r"GB\s*\d+-\d+"）
          - 白名单结构：{ pattern: str, reason: str, scope: str, confirmed_by: str }
       b. 查 anchors[] 声明
          - 找 fact_id
          - 若找不到 → ANCHOR.UNBOUND
          - 找到 fact_id 后，查 GroundTruth.facts[fact_id].value
          - 若 fact_id 不在 G3 中 → ANCHOR.UNKNOWN_FACT（fact_id 未在评测声明）
          - `round(token_value, precision) == round(gt_value, precision)`？
            - 是 → 通过
            - 否 → ANCHOR.VALUE_MISMATCH
       c. 实例标识 token 额外检查：
          - 比如 token "K002" 绑定到 fact:component.K003.id → 错配
          - 实例标识必须绑定到对应的 scope_class 实例的 id 属性

    3. 特别注意：不做 token 覆盖度检查（即 anchors[] 声明了但 text 中没出现的 token）——这属于 IR 完整性检查，由 IR Validator 负责（§41.2 R6）
```

### Narrative 段 vs Table cell vs Figure caption 的处理差异

| 载体 | 差异 |
|---|---|
| Narrative prose | 自由文本，所有数值/标识必须声明 anchor |
| Table cell (Ref 渲染) | 若来自 Ref，理论上渲染前已校验；但渲染后仍需扫描——防止绕过 Ref 直接硬编码 |
| Table cell (Computed 渲染) | 同上 |
| Table cell (Lit 中的数值) | **最危险**——Lit 可以放任意文本，必须做锚定检查 |
| Table title / table note | 容易被遗忘，必须扫描（D-018 强制纳入） |
| Figure caption | 同 table title，必须扫描 |

### 边界场景处理

| 场景 | 处理 |
|---|---|
| "约 2.3m"（近似） | 提取 token "2.3m"，锚定到 fact:xxx.length；"约" 是 Lit 不检查 |
| "2.0～2.5m"（区间） | 两个 token 独立锚定，各自值相等 |
| 白名单正则匹配失败 | P0 error——注册制，没匹配到就是没登记，登记后下次跑 |
| 实例标识 token 绑定到错误 fact_id | ANCHOR.BINDING 产生 ANCHOR.WRONG_FACT（D-018 强制检查） |
| assertion 段 Ref 渲染后 | 仍需扫描——Ref 渲染后的数字在 text 中，必须存在 anchor 声明 |

---

## 6. 结论验证（CONCLUSION.DIRECTION + CONCLUSION.COVERAGE）

### CONCLUSION.DIRECTION（§6.5）

```text
System Conclusion → program parse → direction_str
     ↓
GroundTruthConclusion.direction
     ↓
枚举精确比较 ==
```

**严格不做语义相似度**。方向值的枚举域由 `ConclusionRule.output` 定义（在具体报告类型的规则文件中，不在本模块），Validator 只做字符串精确相等。

若 system 产生了新的方向值不在业务枚举中 → 这是 IR Schema 的问题，应由 IR Validator 拦截，本 Validator 报 IR.INVALID_DIRECTION 然后拒绝执行。

### CONCLUSION.COVERAGE（§42.1 + D-026）

```text
1. 找所有 semantic="conclusion" 的 assertion 段
2. 收集所有 covers[] 条目 → covered_ids: set[str]
3. expected_ids = GroundTruthConclusion.required_evaluation_ids
4. missing = expected_ids - covered_ids
5. 若 missing 非空 → 每一条 missing 产生一个 ActualIssue(issue_type="CONCLUSION.COVERAGE_GAP", evaluation_id=missing_id)
6. 若 conclusion 段根本没有 covers[] → 一个 ActualIssue(issue_type="CONCLUSION.COVERS_MISSING")
```

**边界情况**：
- 有多个 conclusion 段？收集所有 covers[] 的并集
- covers[] 中的 evaluation_id 在 G4 required_evaluation_ids 中不存在？**不报错**——多覆盖不是错，漏覆盖才是
- G4 的 required_evaluation_ids 是 opaque ID？Step 7-D 已验证非空、无重复。本 Validator 直接做 set difference 比较。

---

## 7. expected_issues 匹配规则（EXPECTED.HIT）

从评测独立性、防作弊、可重复判定三原则推导，采用**精确匹配、零模糊**策略。

### 匹配键设计

```text
对每条 ExpectedIssue：
    1. 先找 system_reported_issues 中 rule_id 相同的条目集
    2. 若 ExpectedIssue.fact_id 非 None：
        在 rule_id 匹配集中找 fact_id 精确相等的条目
        命中 → ✅
        未命中 → 该 ExpectedIssue 判定失败（即使存在 rule_id 相同但 fact_id 不同的条目也不算）
    3. 若 ExpectedIssue.fact_id 为 None：
        只要存在 rule_id 相同的条目 → ✅
        但不能用 rule_id+fact_id 的条目替代 rule_id-only 的 ExpectedIssue（必须严格相等）
```

### 匹配决策表

| ExpectedIssue 特征 | System 侧匹配条件 |
|---|---|
| rule_id=A, fact_id=None | 存在任意一条 rule_id=A 的 reported issue 即命中 |
| rule_id=A, fact_id=fact:xxx.xxx | 必须存在 rule_id=A **且** fact_id=fact:xxx.xxx 的条目 |

### 防作弊机制

| 作弊手法 | 为什么匹配规则能防 |
|---|---|
| 系统乱报一堆 error 来蒙混 expected_issues | ExpectedIssue 有精确 fact_id 约束，乱报不会命中具体 fact_id 的 expected_issue |
| 把一个 expected_issue 拆成多条 system issues | 一条 expected_issue 可以被多条 actual issues 命中（first match 算），但反过来一条 actual issue 不能覆盖多条 expected_issue |
| 故意不报某个 fact_id 的问题 | 若 ExpectedIssue 明确 fact_id，未命中直接产出 ActualIssue |
| 用 warning 代替 error 报错 | EXPECTED.HIT 不检查 severity，因为 severity 是 Validator 内部规则的输出强度。ExpectedIssue.severity 只用于本 Validator 产出的 ActualIssue.severity |

---

## 8. known_issues 的位置

### 唯一合法用途

```python
def _apply_known_issue_correction(
    case: EvaluationCase,
    fact_id: str,
    original_value: Any,
) -> Any:
    """在比对前做值替换，只影响 G3 GroundTruth 的值。
    
    known_issues 是为了纠正 GroundTruth 本身的历史数据错误，
    不是为了放宽 P0 闸门。"""
    for ki in case.known_issues:
        if ki.issue_type == "fact_value_error" and ki.fact_id == fact_id:
            if original_value == ki.original_value:
                return ki.correct_value
    return original_value
```

### 绝对禁止的用途

❌ 不能因为 known_issues 中登记了某 fact_id，就自动放宽对该 fact_id 的 VALUE.CONSISTENCY 检查
❌ 不能因为 known_issues 中有 conclusion_error，就跳过 CONCLUSION.DIRECTION 的比较
❌ 不能用 known_issues 来"容忍" ANCHOR.UNBOUND 或 EXPECTED.NOT_HIT

### 原因

known_issues 登记的是**历史报告本身的错误**，这些错误被写进了 GroundTruth（逆向链路从历史报告逆向抽取时，先把原文错抄了进来）。known_issues 的作用是**纠正 GroundTruth**，让评测对比的基准本身正确，而不是让评测规则变松。

---

## 9. 结果对象设计

### ValidationResult

```python
@dataclass
class ValidationResult:
    """Accuracy Validator 的最终输出（Step 7-E 新增）。

    必须可 JSON 序列化（用于 Run Snapshot），但不含运行时引用。
    """
    # ── 顶层判定 ──
    case_id: str                          # 来自 EvaluationCase.meta.case_id
    case_level: str                       # 来自 EvaluationCase.meta.case_level
    pass: bool                            # 顶层 pass/fail（所有 P0 rule 都 pass 才 True）

    # ── P0 规则逐条结果 ──
    p0_rule_results: list[RuleResult]     # 每条 P0 规则一个

    # ── Validator 实际发现的问题 ──
    actual_issues: list[ActualIssue]      # 本 Validator 产生的全部 Issue

    # ── expected_issues 覆盖矩阵 ──
    expected_issue_coverage: ExpectedIssueCoverage

    # ── 汇总指标（用于 Run Snapshot 对比） ──
    summary_metrics: dict

    # ── 错误（当 EvaluationCase 结构校验失败时设置） ──
    error: Optional[str] = None           # 非 None 时 pass=False
```

### RuleResult

```python
@dataclass
class RuleResult:
    """单条规则的执行结果。"""
    rule_id: str
    pass: bool                            # 本规则 pass/fail
    issues_count: int                     # 本规则产生的 ActualIssue 数
    issues: list[ActualIssue]             # 本规则产生的 ActualIssue 列表
    degraded: bool = False                # 是否因 case_level 降级（非 gold 时某些规则不阻断）
    degraded_reason: Optional[str] = None
```

### ExpectedIssueCoverage

```python
@dataclass
class ExpectedIssueCoverage:
    """expected_issues 命中率的详细矩阵。"""
    expected_total: int
    expected_hit: int
    expected_miss: int                   # 未命中的 ExpectedIssue 数
    hit_rate: float                      # expected_hit / expected_total，expected_total=0 时为 1.0
    unmatched_expected: list[ExpectedIssue]  # 未命中的 ExpectedIssue 完整副本
```

### summary_metrics 结构

```python
{
    "case_level": "gold",
    "gold_count_total": 1,           # 当前评测集中的 gold 用例数（M7 阶段完整统计，当前单 case 为 1）
    "expected_issues_hits": 2,
    "expected_issues_total": 2,
    "value_consistency_pass_rate": 1.0,
    "anchor_binding_pass_rate": 1.0,
    "conclusion_direction_match": True,
    "conclusion_coverage_pass": True,
    "p0_total_rules": 7,
    "p0_failed_rules": 0,
    "total_actual_issues": 0,
    "severity_error_count": 0,
    "severity_warning_count": 0,
}
```

---

## 模块文件结构

```text
src/eval/
├── __init__.py            # 新增导出
├── types.py               # 已有，不动
├── enums.py               # 已有，不动
├── validator.py           # 已有，不动
│
├── contracts.py           # Step 7-E 新增
│   ├── P0_RULE_IDS: frozenset[str]     # 7 条 rule_id 冻结
│   ├── ACCEPTED_SEVERITIES: frozenset  # error | warning（已有 enums 复用）
│   ├── SystemOutput (protocol/interface)
│   ├── RenderedSegment (dataclass)
│   ├── SystemReportedIssue (dataclass)
│   └── WhitelistEntry (dataclass，白名单条目)
│
├── actual_issues.py       # Step 7-E 新增
│   ├── ActualIssue (dataclass)
│   ├── ActualIssueType 注册表（frozenset）
│   └── make_actual_issue(rule_id, issue_type, ...) —— 工厂函数
│
├── results.py             # Step 7-E 新增
│   ├── ValidationResult (dataclass)
│   ├── RuleResult (dataclass)
│   ├── ExpectedIssueCoverage (dataclass)
│   └── to_json() 序列化方法
│
└── accuracy_validator.py  # Step 7-E 新增，核心实现
    ├── AccuracyValidator (class)
    ├── validate(case, system_output) -> ValidationResult
    ├── _run_value_consistency(case, system_output) -> RuleResult
    ├── _run_table_internal(case, system_output) -> RuleResult
    ├── _run_logic_consistency(case, system_output) -> RuleResult
    ├── _run_anchor_binding(case, system_output) -> RuleResult
    ├── _run_conclusion_direction(case, system_output) -> RuleResult
    ├── _run_conclusion_coverage(case, system_output) -> RuleResult
    └── _run_expected_hit(case, system_output) -> RuleResult
```

---

## 与现有代码的完全隔离声明

本 Step **不修改**以下已有文件（测试除外）：

- `src/eval/types.py` —— 9 dataclass 全部保持 v4 冻结契约
- `src/eval/enums.py` —— CASE_LEVELS / SEVERITIES / ISSUE_TYPES 保持不变
- `src/eval/validator.py` —— 所有 validate_* 函数保持不动

**只新增**4 个文件：`contracts.py`、`actual_issues.py`、`results.py`、`accuracy_validator.py`。

`__init__.py` 需要追加导出（import 新增模块），但不删除任何已有导出。

---

## 验收标准（Step 7-E 完成时必须全部满足）

| 编号 | 验收项 | 说明 |
|---|---|---|
| A1 | 新增 4 文件、修改 0 个已有文件（除 __init__.py 追加导出） | 隔离性保证 |
| A2 | 7 条 P0 rule_id 注册到 `P0_RULE_IDS` frozenset | 规则完整性 |
| A3 | AccuracyValidator.validate() 入口检查 `case.validate() == []` | 依赖 Step 7-D 完整性 |
| A4 | ActualIssue / ValidationResult / RuleResult / ExpectedIssueCoverage 可 JSON 序列化 | Run Snapshot 兼容 |
| A5 | ANCHOR.BINDING 的白名单执行登记制（pattern + reason + scope + confirmed_by） | D-019 合规 |
| A6 | known_issues 仅参与值替换，不影响 pass/fail 判定 | §8 原则 |
| A7 | expected_issues 匹配采用 (rule_id, fact_id) 精确组合键 | 防作弊 |
| A8 | 全部测试 pass（Step 7-E 自身的测试） | 质量门槛 |

---

以上就是 Step 7-E Coding Contract 完整设计。没有进入代码实现，没有修改任何已有文件，所有回答严格基于 02/03/04/05/07 五份设计文档和现有 Step 7-B/7-C/7-D 实现。

请确认这份 Contract 的方向是否正确，确认后再开始代码实现。
好的，现在我需要调用 AdvisorTool 在形成 Step 7-E Coding Contract 之前获得战略指导。这是一个典型的"before committing to an approach"场景——我需要在正式输出 Coding Contract 之前确保方向正确。

让我调用 AdvisorTool。
正在请求专家建议............

已获取专家建议

AdvisorTool 的建议非常关键——特别是关于两个 Issue 流分离的洞察。让我现在基于所有设计文档、代码实现和 Advisor 建议，系统性地构建 Step 7-E Coding Contract。

让我按照用户提出的9个核心问题逐一回答，同时结合 Advisor 的战略指导。

好，Contract 已经完整输出了。现在我需要调用 AdvisorTool 在声明任务完成之前做最终审查。
正在请求专家建议.......

已获取专家建议

Advisor 提出了三个缺口，我需要在最终 Contract 中修正它们：

1. SystemOutput 需要增加 domain_objects / evaluations 字段——LOGIC.CONSISTENCY 和 TABLE.INTERNAL 需要 L3 的 Criterion 和 Evaluation 对象
2. SystemReportedIssue 需要正式定义 dataclass
3. `pass` 是 Python 关键字，需要改成 `passed`

让我在最终回复中补充这三个修正。
Coding Contract 已完整输出。在最终交付前调用 Advisor 做一次整体审查：

*正在请求专家建议，优化解决方案，该过程耗时可能较长，请耐心等待...*

Advisor 提出了三个实际会导致实现问题的缺口。以下是最终修正后的完整 Coding Contract（三个缺口的修正点用 **🔴修正** 标注）：

---

# Step 7-E Coding Contract（修正版）

## 修正汇总（来自 Advisor 终审）

| 编号 | 问题 | 修正 |
|---|---|---|
| C1 | SystemOutput 缺少 domain_objects 字段——LOGIC.CONSISTENCY / TABLE.INTERNAL / CONCLUSION.COVERAGE 需要 L3 对象不在 Report IR 中 | 新增 `domain_objects` 字段 |
| C2 | SystemReportedIssue 被引用但未定义 | 在 §A 正式定义 dataclass |
| C3 | `pass` 是 Python 保留关键字，作为 dataclass 字段名会引发序列化和代码风格问题 | 改为 `passed` |

---

## 先确立命名和概念基线

### 三层 Issue 概念（绝对不能混淆）

| 命名 | 来源 | 含义 | 谁创建 | 用途 |
|---|---|---|---|---|
| **ActualIssue** | Step 7-E 新增 `actual_issues.py` | Accuracy Validator 实际发现的问题 | Accuracy Validator | 与 expected_issues 对比、构建 ValidationResult、驱动 P0 判定 |
| **ExpectedIssue** | Step 7-D 已有 `eval.types.ExpectedIssue` | EvaluationCase 固件中登记的"系统应该发现的问题" | 逆向链路（构造期） | P0 `EXPECTED.HIT` 规则的判定依据 |
| **KnownIssue** | Step 7-D 已有 `eval.types.KnownIssue` | 历史报告中的已知错误（原文错、正确值已确认） | 逆向链路（构造期） | **只在比对时做值替换**，绝不放宽 P0 闸门 |
| **SystemReportedIssue** | 生成管线 Step 6 validate 自己产出的问题 | 系统在生成过程中报的问题 | 生成管线 | EXPECTED.HIT 规则的比对对象 |

**关键洞察**：ActualIssue 有两条独立产生路径：
1. **路径 A**（`EXPECTED.HIT`）：Validator 检查 SystemReportedIssue 中是否存在 ExpectedIssue 要求的条目
2. **路径 B**（`VALUE.CONSISTENCY` 等 6 条）：Validator 对比 System Output 与 GroundTruth 产生的 Issue

---

## 1. 输入边界

### Accuracy Validator 接收什么

```text
输入（必须、不可缺失）:
├── EvaluationCase              —— Step 7-D 产物
│
├── SystemOutput                —— 🔴修正 C1：增加 domain_objects
│   ├── report_ir: AnyDocumentIR
│   ├── rendered_segments: list[RenderedSegment]
│   ├── system_reported_issues: list[SystemReportedIssue]
│   ├── computed_aggregate_values: dict[str, Any]
│   └── domain_objects: DomainObjectSet       🔴新增
│       ├── evaluations: list[Evaluation]     —— L3 Evaluation 对象（LOGIC.CONSISTENCY 需要）
│       ├── criteria: list[Criterion]         —— L3 Criterion 对象（LOGIC.CONSISTENCY 需要）
│       └── conclusions: list[Conclusion]     —— L3 Conclusion 对象（CONCLUSION.COVERAGE 需要）
│
└── RuleRegistry (frozenset)

中间产物：
├── ActualIssue[]
├── RuleResult[]
└── ExpectedIssueCoverage

最终输出：
└── ValidationResult
```

### 🔴修正 C1 详细说明

Advisor 指出：Report IR 只携带 `Ref` 引用，不携带 Criterion.rule（判定逻辑）、Criterion.value（限值）、Evaluation.status（判定结果）、Conclusion.rule_id（聚合规则）这些 L3 领域对象。

- **LOGIC.CONSISTENCY** 需要 `Criterion.rule`（判定形式如 `"value <= limit"`）和 `Criterion.value`（限值）来检查 Evaluation.status 是否与 Fact 值一致
- **TABLE.INTERNAL** 需要 `computed_aggregate_values`（已在 SystemOutput 中），但同时也需要确认派生量的计算是否与程序规则一致——这部分数据来自 System 运行时，不是 IR
- **CONCLUSION.COVERAGE** 已确认 `covers[]` 在 Report IR 的结论段（§42.1 + D-026），可以从 IR 中取；但 L3 Conclusion 对象携带的 `required_evaluation_ids` 可以作为校验——G4 用例的 G4.conclusion.required_evaluation_ids 应包含在 L3 Conclusion 的 required_evaluation_ids 中

**SystemOutput 接口修正后**：

```python
@dataclass
class SystemOutput:
    """Step 7-E Accuracy Validator 的输入协议对象。

    本对象只携带已解析好的结构化数据，不做解析。
    解析/渲染由 Pipeline Orchestrator 在 Step 6 调用 Accuracy Validator 前完成。
    """
    report_ir: AnyDocumentIR
    rendered_segments: list[RenderedSegment]
    system_reported_issues: list[SystemReportedIssue]
    computed_aggregate_values: dict[str, Any]
    
    # 🔴修正 C1：L3 Domain Objects
    evaluations: list[Evaluation] = field(default_factory=list)  # 判定结果对象
    criteria: list[Criterion] = field(default_factory=list)      # 判定依据对象
    conclusions: list[Conclusion] = field(default_factory=list)  # 聚合结论对象
```

### 🔴修正 C2：SystemReportedIssue 正式定义

```python
@dataclass
class SystemReportedIssue:
    """生成管线 Step 6 validate 产出的问题（EXPECTED.HIT 的比对对象）。

    与本 Validator 产生的 ActualIssue 类型相同但来源不同——
    ActualIssue 是本 Validator 用 P0 规则独立判定的；
    SystemReportedIssue 是被评测系统自己报的问题。
    """
    rule_id: str                      # 必须在 P0_RULE_IDS 中
    fact_id: Optional[str] = None     # 可选（某些 issue 不涉及具体 Fact）
    evaluation_id: Optional[str] = None
    issue_type: str = ""
    severity: str = "error"
    message: str = ""
```

---

## 2. Validator 与 Step 7-D 的边界（不变）

| 职责 | Step 7-D | Step 7-E |
|---|---|---|
| EvaluationCase 结构正确性 | ✅ | ❌ 依赖已通过 |
| 文件加载 / 必需目录 | ✅ | ❌ 不做 IO |
| G3 ⊆ G2 引用完整性 | ✅ | ❌ 假设已成立 |
| case_level ↔ G3 / G4 一致性 | ✅ | ❌ 假设已成立 |
| **System Output vs GroundTruth** | ❌ | ✅ |
| **P0 判定规则执行** | ❌ | ✅ |
| **expected_issues 命中率** | ❌ | ✅ EXPECTED.HIT |

**边界线**：Step 7-D 验证评测用例本身，Step 7-E 验证系统产出的报告与标准答案是否一致。

---

## 3. 七条 P0 判定规则程序化规格（修正 pass → passed）

### Rule 注册表

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

### 3.1 VALUE.CONSISTENCY

| 维度 | 规格 |
|---|---|
| rule_id | `VALUE.CONSISTENCY` |
| 输入 | SystemOutput.report_ir 的 Ref 段 → Fact.value + GroundTruth.facts[]（G3） |
| 输出 | ActualIssue[] |
| **passed 条件**（🔴修正 C3） | 对每个 G3 fact_id：<br>1. System Ref 段存在该 fact_id 的引用<br>2. `round(system_value, precision) == round(gt_value, precision)`<br>3. `system_unit == gt_unit`<br>4. `system_quantity_kind == gt_quantity_kind` |
| fail 条件 | 上述任一条不满足 |
| Issue 类型 | `VALUE.MISMATCH` / `UNIT.MISMATCH` / `QUANTITY.MISMATCH` / `VALUE.MISSING` |
| severity | `error` |
| P0 | ✅ |
| 容差 | **禁止**，precision 由 G3 显式声明 |
| 降级 | 仅 `case_level != gold` 时不触发门禁失败 |
| known_issues 参与 | 值替换（`KnownIssue.correct_value` 替换 G3 original_value），不放宽 pass 条件 |

### 3.2 TABLE.INTERNAL

| 维度 | 规格 |
|---|---|
| rule_id | `TABLE.INTERNAL` |
| 输入 | SystemOutput.rendered_segments(type=table_cell) + SystemOutput.computed_aggregate_values |
| **passed 条件**（🔴修正 C3） | 对每张表：合计 ≈ 分项之和（相对误差 ≤ 1e-9）；最大值 ≥ 均值 ≥ 最小值；计数行 == 明细条数 |
| fail 条件 | 上述任一条不满足 |
| Issue 类型 | `TABLE.SUM_MISMATCH` / `TABLE.ORDER_VIOLATION` / `TABLE.COUNT_MISMATCH` |
| severity | `error` |
| P0 | ✅ |
| 容差 | 派生量允许相对误差 ≤ 1e-9 |
| 降级 | 无 |
| known_issues 参与 | 不直接参与 |

### 3.3 LOGIC.CONSISTENCY

| 维度 | 规格 |
|---|---|
| rule_id | `LOGIC.CONSISTENCY` |
| 输入 | 🔴修正 C1：SystemOutput.evaluations[] + SystemOutput.criteria[] + SystemOutput.report_ir 的 assertion Ref 段 |
| **passed 条件** | 每个 Evaluation.status 与 Criterion.rule 检查一致：<br>- `rule="value <= limit"` + `fact_value <= criterion.value` → `status in {qualified, pass}`<br>- `rule="value <= limit"` + `fact_value > criterion.value` → `status in {unqualified, fail}` |
| fail 条件 | 判定结果与输入值违反 rule |
| Issue 类型 | `LOGIC.MISMATCH` |
| severity | `error` |
| P0 | ✅ |
| 容差 | 布尔比较，无容差 |
| 降级 | 无 |
| known_issues 参与 | conclusion_error 做方向替换 |

### 3.4 ANCHOR.BINDING

| 维度 | 规格 |
|---|---|
| rule_id | `ANCHOR.BINDING` |
| 输入 | SystemOutput.rendered_segments（**渲染后产物**）+ 白名单（模式级，注册制） |
| **passed 条件** | 对每个 rendered_segment：每个数值 token 和实例标识 token 要么 (a) 绑定 fact_id 且值相等；要么 (b) 命中登记制白名单 |
| fail 条件 | 任一 token 未绑定且未命中白名单 |
| Issue 类型 | `ANCHOR.UNBOUND` / `ANCHOR.WRONG_FACT` / `ANCHOR.VALUE_MISMATCH` |
| severity | `error` |
| P0 | ✅ |
| 容差 | precision 控制 |
| 降级 | 无 |
| 白名单位置 | EvaluationCase 后续需新增 `ground_truth/whitelist.json`（Step 7-E 先预留接口） |

### 3.5 CONCLUSION.DIRECTION

| 维度 | 规格 |
|---|---|
| rule_id | `CONCLUSION.DIRECTION` |
| 输入 | SystemOutput.report_ir → conclusion direction + GroundTruthConclusion.direction（G4） |
| **passed 条件** | `system_direction == gt_direction`（枚举精确相等，不做语义相似度） |
| fail 条件 | 枚举值不同 |
| Issue 类型 | `CONCLUSION.DIRECTION_MISMATCH` |
| severity | `error` |
| P0 | ✅ |
| 容差 | 无 |
| 降级 | silver 若 G4 只能从正文读出则自动降级 bronze（不执行） |
| known_issues 参与 | conclusion_error 做方向替换 |

### 3.6 CONCLUSION.COVERAGE

| 维度 | 规格 |
|---|---|
| rule_id | `CONCLUSION.COVERAGE` |
| 输入 | SystemOutput.report_ir 的 conclusion assertion 段 → `covers[]` + GroundTruthConclusion.required_evaluation_ids（G4） |
| **passed 条件** | 每个 required_evaluation_id 至少被一个 covers[] 引用 |
| fail 条件 | 存在未覆盖的 required_evaluation_id |
| Issue 类型 | `CONCLUSION.COVERAGE_GAP` / `CONCLUSION.COVERS_MISSING` |
| severity | `error` |
| P0 | ✅ |
| 容差 | 无 |
| 降级 | 无 |
| known_issues 参与 | 不直接参与 |

### 3.7 EXPECTED.HIT

| 维度 | 规格 |
|---|---|
| rule_id | `EXPECTED.HIT` |
| 输入 | EvaluationCase.expected_issues[] + SystemOutput.system_reported_issues[]（🔴修正 C2：正式类型定义） |
| **passed 条件** | 每一条 ExpectedIssue 都能在 system_reported_issues 中找到匹配项（§7 匹配规则） |
| fail 条件 | 存在 ExpectedIssue 未命中 |
| Issue 类型 | `EXPECTED.NOT_HIT` |
| severity | 与 ExpectedIssue.severity 对齐 |
| P0 | ✅ |
| 容差 | 零容差 |
| 降级 | gold 经济门禁（gold < 5 时不阻断） |
| known_issues 参与 | 不参与（值替换不属于此规则） |

---

## 4. ActualIssue 正式建模（不变）

```python
@dataclass
class ActualIssue:
    """Accuracy Validator 实际发现的问题（Step 7-E 新增）。"""
    issue_id: str                  # 运行时生成："{rule_id}#{hashable_key}"
    rule_id: str                   # VALUE.CONSISTENCY / ANCHOR.BINDING / ...
    severity: str                  # error | warning
    issue_type: str                # VALUE.MISMATCH / ANCHOR.UNBOUND / ...
    fact_id: Optional[str] = None
    evaluation_id: Optional[str] = None
    location: Optional[str] = None # Section.semantic + Block.id
    token: Optional[str] = None    # 数字锚定规则用
    expected: Any = None
    actual: Any = None
    message: str = ""
```

---

## 5. 数字锚定（不变）

关键设计点保持：
- 输入必须是 **渲染后产物**（§41.2 R5），不是 IR 声明
- 不做"全文值集合兜底"（D-017）
- 实例标识 token（构件编号等）必须与数值 token 同表绑定（D-018）
- 白名单是 **登记制**（pattern + reason + scope + confirmed_by），不是逐条放行（D-019）
- 正则匹配未命中 → P0 error

---

## 6. 结论验证（不变）

- **CONCLUSION.DIRECTION**：枚举精确相等，不做语义相似度（§6.5 + D-023）
- **CONCLUSION.COVERAGE**：从 conclusion assertion 的 `covers[]` 取（§42.1 + D-026），set difference 比较

---

## 7. expected_issues 匹配规则（不变）

| ExpectedIssue 特征 | SystemReportedIssue 匹配条件 |
|---|---|
| rule_id=A, fact_id=None | 存在任意一条 rule_id=A → 命中 |
| rule_id=A, fact_id=fact:xxx.xxx | 必须 rule_id=A **且** fact_id=fact:xxx.xxx → 命中 |

**零模糊匹配**——防作弊 + 可重复判定。

---

## 8. known_issues 的位置（不变）

**唯一合法用途**：比对前做值替换（correction of GroundTruth 本身的历史数据错误）。

**绝对禁止**：放宽 P0 闸门 / 跳过规则 / 容忍 ANCHOR.UNBOUND 或 EXPECTED.NOT_HIT。

---

## 9. 结果对象设计（🔴修正 C3：pass → passed）

### ValidationResult

```python
@dataclass
class ValidationResult:
    case_id: str
    case_level: str
    passed: bool                          # 🔴修正 C3：passed，不是 pass
    p0_rule_results: list[RuleResult]
    actual_issues: list[ActualIssue]
    expected_issue_coverage: ExpectedIssueCoverage
    summary_metrics: dict
    error: Optional[str] = None           # evaluation case 结构校验失败时设置
```

### RuleResult

```python
@dataclass
class RuleResult:
    rule_id: str
    passed: bool                          # 🔴修正 C3
    issues_count: int
    issues: list[ActualIssue]
    degraded: bool = False
    degraded_reason: Optional[str] = None
```

### ExpectedIssueCoverage（不变）

```python
@dataclass
class ExpectedIssueCoverage:
    expected_total: int
    expected_hit: int
    expected_miss: int
    hit_rate: float
    unmatched_expected: list[ExpectedIssue]
```

---

## 模块文件结构（不变）

```text
src/eval/
├── __init__.py            # 追加导出
├── types.py               # 不动（Step 7-D v4 冻结）
├── enums.py               # 不动
├── validator.py           # 不动
│
├── contracts.py           # 新增
│   ├── P0_RULE_IDS: frozenset[str]
│   ├── SystemOutput (dataclass)
│   ├── RenderedSegment (dataclass)
│   ├── SystemReportedIssue (dataclass)  🔴修正 C2：正式定义
│   └── WhitelistEntry (dataclass)
│
├── actual_issues.py       # 新增
│   ├── ActualIssue (dataclass)
│   └── ISSUE_TYPES_ACCURACY (frozenset)
│
├── results.py             # 新增
│   ├── ValidationResult (dataclass)
│   ├── RuleResult (dataclass)          🔴修正 C3：passed 字段
│   └── ExpectedIssueCoverage (dataclass)
│
└── accuracy_validator.py  # 新增，核心实现
    ├── AccuracyValidator (class)
    └── validate(case, system_output) -> ValidationResult
```

---

## 验收标准

| 编号 | 验收项 | 说明 |
|---|---|---|
| A1 | 新增 4 文件、修改 0 个已有文件（除 __init__.py 追加导出） | 隔离性 |
| A2 | 7 条 rule_id 注册到 `P0_RULE_IDS` frozenset | 规则完整性 |
| A3 | 入口检查 `case.validate() == []` | 依赖 Step 7-D |
| A4 | ActualIssue / ValidationResult / RuleResult / ExpectedIssueCoverage 可 JSON 序列化 | Run Snapshot 兼容 |
| A5 | 🔴修正 C2：SystemReportedIssue 正式定义在 contracts.py | EXPECTED.HIT 可用 |
| A6 | 🔴修正 C1：SystemOutput 携带 evaluations + criteria + conclusions | LOGIC.CONSISTENCY 可用 |
| A7 | 🔴修正 C3：所有 `pass` 字段改为 `passed` | Python 关键字规避 |
| A8 | known_issues 仅做值替换 | §8 原则 |
| A9 | expected_issues 匹配精确（rule_id, fact_id） | 防作弊 |
| A10 | 全部测试 pass | 质量门槛 |

---

修正版 Coding Contract 完整输出完毕。三个 Advisor 指出的实现障碍已在 Contract 中落地修正。确认后即可进入 Step 7-E 代码实现。
