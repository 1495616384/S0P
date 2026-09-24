用户要求对 Step 7-D 实施方案做最后一次修订（v3），严格处理 5 个问题：
1. 明确定义 G2 ↔ G3 跨文件关系
2. 统一 Optional / nullable / default 语义
3. 删除 GroundTruthFact.precision 的默认值 2
4. 明确 required_evaluation_ids 是 opaque stable identifier
5. 修正测试矩阵语义

在输出 v3 之前，先调用 AdvisorTool 获取策略指导，确保所有修订方向正确。这属于 "Before committing to an approach" 的场景。

ExperienceRecall 列出的经验主要涉及代码实现层面的边界控制和规则口径一致性，当前任务是纯设计修订，不涉及编码，所以这些经验的适用度有限。不过用户特别强调 "最终方案必须足够明确，可以直接作为下一轮 Coding Contract"，这提醒我设计层面的语义模糊如果不澄清，会直接导致实现阶段反复推翻。AdvisorTool 的输入会帮助我确认每个修订点的完整覆盖度。

用户的 5 个问题都是结构性的：G2/G3 跨文件关系、Optional 语义统一、precision 默认值删除、Evaluation ID 的 opaque 定义、测试矩阵语义修正。这些都是 validator 规则层的明确性问题，不是方向摇摆问题。AdvisorTool 的输入会帮助我确保每个修订点都被完整覆盖，不会在后续输出中遗漏边界情况。
正在请求专家建议.....

已获取专家建议

好，AdvisorTool 的输入确认了所有 5 项修订方向都是正确的，现在需要把它们完整地融入 v3。让我逐项过一遍用户的要求，确保每一条都被准确覆盖，没有遗漏。

第 1 点：G2 ↔ G3 跨文件关系。这是 Referential Integrity，和数据库外键约束是同一类问题。G3 引用 G2 必须声明过的 fact_id，这是 fixture 本身的结构完整性，在任何运行时系统接触它之前就应该被验证。如果 G3 的 fact_id 不在 G2 里，不管 Accuracy Validator 怎么实现，这个 fixture 就是结构上坏掉的。AdvisorTool 把这点定性为 "structural referential integrity"，完全同意。需要加到 validate_case 里，加到测试矩阵第四组，也需要在 fixture 描述里明确。

第 2 点：Optional / nullable / default 语义统一。AdvisorTool 明确了三个类别：required（无默认，必须提供，JSON 里缺了 error）、nullable-required（Optional[X] 但仍然必须在 JSON 里显式出现 null，validator 再按条件检查非空）、optional-default（有默认值，JSON 里可以省略）。关键区别是 KnownIssue.fact_id：它的类型是 Optional[str] = None，但在 issue_type='fact_value_error' 时是 nullable-required（必须显式给出 null 或非 null，validator 再检查非空），在 issue_type='conclusion_error' 时是 optional（可以省略）。这个区分很重要，因为 dataclass 本身的 Optional 类型只说明 Python 层面可以 None，不说明 JSON 序列化层面是否可以省略。

第 3 点：删除 precision 默认 2。完全同意。precision 是 Ground Truth 的显式元数据，由人工在逆向链路中确认（比如从原始 Excel 检测记录的小数位推断），不由 Step 7-D 猜测。如果默认 2，会掩盖"这个 GroundTruthFact 到底有没有被正确定义"——precision 缺失会被静默替换成 2，而不是暴露出来。改成 required 字段，dataclass 里 precision: int，没有默认值。

第 4 点：required_evaluation_ids 是 opaque stable identifier。Step 7-D 不定义 Evaluation 对象、不定义 Evaluation ID grammar、不创建 evaluation registry。validator 只检查 list 非空、每个 ID 非空、不重复。任何暗示 evaluation:xxx 是当前正式 grammar 的内容都要删掉，改用抽象示例 evaluation-001。

第 5 点：T15 的语义错误。这是 G2/fact_list 和 G3/facts 的直接混淆。T15 写的是 "bronze 的 GroundTruthFactList facts 为空数组"，但 GroundTruthFactList 只有 fact_ids 字段，没有 facts。正确应该是 bronze 的 GroundTruthBundle.facts == []，同时 GroundTruthFactList.fact_ids 非空。需要全面审计所有 fixture 描述和测试命名，确保 G2 = fact_list，G3 = facts 的对应关系没有被误写。

现在按照用户要求的 A-J 结构输出 v3，确保每个修订点都被清晰标注，方案足够明确可以直接作为 Coding Contract。

A. 设计目标 - 保留 v2 的定位，明确 Step 7-D 是 fixture 的结构完整性验证者
B. 数据结构 - 9 个 dataclass，每个字段标注 required / nullable / optional-default，precision 去掉默认值，fact_id 在 KnownIssue 里明确 nullable-required / optional 的条件区别
C. 文件结构 - 与 v2 相同，但 fixture 描述里明确 G2/G3 对应关系
D. Validator 完整规则 - 分两组：子 validator（只做单个文件的结构），顶层 validate_case（跨文件一致性 + G2→G3 referential integrity）
E. G2/G3/G4 关系 - 单独一节讲清楚：G2 是 fact_id 清单，G3 是答案值，G3 的每个 fact_id 必须出现在 G2，两边都不允许重复
F. gold/silver/bronze 完整规则 - 四个 ground_truth 子文件的存在性 + 内容要求，按级别逐项列
G. 测试矩阵 - 三组基础 + 第四组跨文件一致性（包含 G2→G3 referential integrity 的 4 个新测试），修正 T15 和所有 G2/G3 混淆的命名
H. 与 7-B/7-C 的依赖 - 复用的是 import 的函数和枚举，不复用 Fact dataclass
I. 明确留给 Step 7-E 及以后 - 不变
J. 最终最小实现范围 - 不变，验收标准明确

现在开始输出。


*正在请求专家建议，优化解决方案，该过程耗时可能较长，请耐心等待...*

好。5 项修订全部确认正确，现在直接输出 v3。

---

# Step 7-D 实施方案 v3：Evaluation Case 基础层

> **版本**：v3 — Coding Contract 就绪版本  
> **修订时间**：2026-09-24  
> **本轮修订来源**：5 项结构完整性要求（G2↔G3 referential integrity / Optional 语义统一 / precision 去默认 / Evaluation ID opaque / 测试矩阵语义修正）

---

## A. Step 7-D v3 设计目标

**定位**：为评测体系建立一个**程序化、可版本化、可独立验证的评测用例数据结构层**。

Step 7-D 是**评测固件的结构完整性守卫者**——它验证 fixture 自身是否内部一致、文件是否齐全、字段是否非空、跨文件引用是否合法。它的工作边界是：

| 它是 | 它不是 |
| ---- | ---- |
| fixture 结构完整性验证者 | 运行时数据流的一部分 |
| G2 / G3 跨文件 referential integrity 检查者 | 数值比对器 |
| JSON schema 合规守卫者 | LLM / Store / Pipeline |
| 只读数据结构定义 | 执行器 |

**一句话**：Step 7-D 确保一份 fixture 是**结构健全的评测固件**，后续 Step 7-E Accuracy Validator 才有资格使用它。

**明确不做的**（与 v2 相同，重申一次）：
- 历史报告逆向抽取
- LLM / Adapter / Skill
- Accuracy Validator / Style Validator
- Regression Runner / Baseline / Run Snapshot / 指标计算
- DOCX / PDF / 数据库 / Agent / UI
- 修改 Step 7-B / 7-C 任何代码
- 修改任何设计文档

---

## B. 数据结构（9 个 dataclass）

### 字段语义统一规则（v3 新增）

每个字段在 dataclass 定义中显式标注三类之一：

| 类别 | 含义 | dataclass 定义方式 | JSON 缺失时 from_dict 行为 |
| ---- | ---- | ---- | ---- |
| **required** | 必须在 JSON 中提供，值不能为 None | 无默认值 | KeyError 异常（由调用方捕获） |
| **nullable-required** | Optional[X] 类型，但 JSON 中**必须显式出现**（值可以为 null） | `field(default=None)` —— 仍有默认，但 validator 会在条件下要求非 None | 使用 .get() 取 None；validator 再按条件检查是否应为非 None |
| **optional-default** | 有 Python 默认值，JSON 中**可以省略** | `field(default_factory=list)` 或 `field(default="")` | 省略时使用默认值 |

以下所有 dataclass 逐字段标注此三类。

---

### 1. Meta（meta.json）

```python
@dataclass
class Meta:
    """05_EVALUATION.md §5.1 定义的用例元信息。"""
    case_id: str               # required — 如 "case-001"
    case_level: str            # required — gold | silver | bronze
    report_type: str            # required — 报告类型标识
    source_project: str         # required — 来源项目标识
    created_at: str             # required — ISO 8601
    
    design_version: Optional[str] = None    # optional-default — 指向哪版设计文档
    traps: list[str] = field(default_factory=list)  # optional-default — 陷阱标注
    notes: str = ""                             # optional-default — 自由文本备注
    
    # —— 序列化 ——
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "Meta": ...
    def to_json(self) -> str: ...
```

**Validator 检查**：
- case_level ∈ {gold, silver, bronze}
- case_id / report_type / source_project 非空、非纯空白
- created_at 可解析为合法 ISO 8601

---

### 2. GroundTruthFact（ground_truth/facts.json 的条目 — G3）

```python
@dataclass
class GroundTruthFact:
    """G3 事实值 Ground Truth — 轻量答案密钥。
    
    语义区别于运行时 Fact（Step 7-B）：
    运行时 Fact 有 source_refs / method / provenance / review_status /
    status / revision / supersedes —— GroundTruthFact 全都没有。
    
    只复用 Step 7-B 的：
        - is_valid_fact_id() 做 grammar 校验
        - QUANTITY_KINDS 枚举
    
    precision 是 required 字段（v3 第 3 项修订）：
        数值比较遵守 G3 的原始精度（05 §6.1），
        precision 不由 Step 7-D 猜测，必须显式提供。
    """
    fact_id: str                    # required — 三段式 grammar
    value: Any                      # required — float|int|str|bool|None
    unit: Optional[str] = None     # nullable-required for 定量事实（显式给 null），
                                   #   optional-default 允许省略（省略时默认 None）
                                   #   定性事实正常给 None
    quantity_kind: str              # required — QUANTITY_KINDS 枚举
    precision: int                  # required — v3 修订：显式提供，无默认值
    
    # —— 序列化 ——
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "GroundTruthFact": ...
```

**Validator 检查**：
- fact_id 通过三段式 grammar（`is_valid_fact_id()`）
- quantity_kind ∈ QUANTITY_KINDS（Step 7-B）
- precision >= 0
- （bronze 用例由顶层 validate_case 强制整个 facts 列表为空数组）

---

### 3. GroundTruthFactList（ground_truth/fact_list.json — G2）

```python
@dataclass
class GroundTruthFactList:
    """G2 事实清单 — 报告应当涉及哪些事实项。
    
    与 GroundTruthFact（G3）的语义边界：
        G2.fact_ids  = 清单（哪些事实应该存在）
        G3.fact_id + value = 答案密钥（这些事实的值是多少）
        G3.fact_id 必须 出现在 G2.fact_ids 中（v3 第 1 项修订）
    """
    fact_ids: list[str]            # required — 列表本身必须存在
                                   #   元素在 bronze/gold/silver 有不同要求
                                   #   列表内不允许重复（v3 第 1 项修订）
    
    # —— 序列化 ——
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "GroundTruthFactList": ...
```

**Validator 检查**：
- 每个 fact_id 通过三段式 grammar
- 列表内无重复 fact_id（`len(fact_ids) == len(set(fact_ids))`）

---

### 4. GroundTruthStructure（ground_truth/structure.json — G4）

```python
@dataclass
class GroundTruthStructure:
    """G4 报告形态 — 章节结构、表格、图片、必需要素。"""
    chapter_paths: list[str]        # required
    table_count: int                # required — >= 0
    table_headers: list[list[str]]  # required — 允许空列表（无表格时）
    figure_count: int               # required — >= 0
    required_elements: list[str]    # required — 允许空列表
    
    # —— 序列化 ——
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "GroundTruthStructure": ...
```

**Validator 检查**：
- table_count >= 0, figure_count >= 0
- chapter_paths 元素非空
- table_headers 子列表元素非空
- required_elements 元素非空

---

### 5. GroundTruthConclusion（ground_truth/conclusion.json — G4）

```python
@dataclass
class GroundTruthConclusion:
    """G4 结论 — 方向 + 覆盖度。
    
    required_evaluation_ids 是 opaque stable identifier（v3 第 4 项修订）：
        Step 7-D 不定义 Evaluation 对象。
        Step 7-D 不定义 Evaluation ID grammar。
        Step 7-D 不创建 evaluation registry。
        Step 7-D 只做三件事：列表非空、每个 ID 非空、不重复。
        正式 Evaluation ID schema / grammar / registry
        留给后续 Evaluation / Conclusion 领域模型阶段。
    
    direction 的业务取值域：
        不在 Step 7-D 定义全局 conclusion enum。
        direction 是纯 str，validator 只查非空。
        业务取值域由 ConclusionRule + report type 提供（v2 已决定，重申）。
    """
    direction: str                              # required
    required_evaluation_ids: list[str]         # required — opaque，不暗示 grammar
    notes: str = ""                             # optional-default
    
    # —— 序列化 ——
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "GroundTruthConclusion": ...
```

**Opaque identifier 示例**（v3 第 4 项修订，避免暗示 grammar）：

```text
"evaluation-001"
"evaluation-002"
"evaluation-003"
```

**不要使用**任何 `evaluation:xxx` 格式——当前阶段没有 Evaluation 对象的正式 grammar。

**Validator 检查**：
- direction 非空
- required_evaluation_ids 非空列表
- 每个元素非空
- 列表内无重复（`len(ids) == len(set(ids))`）

---

### 6. GroundTruthBundle（顶层容器）

```python
@dataclass
class GroundTruthBundle:
    """四个 ground_truth 子结构的容器。
    
    G2 / G3 / G4 跨文件一致性由顶层 validate_case 执行，
    不在子 validator 中执行。
    """
    facts: list[GroundTruthFact]   # required — G3；bronze 时必须为 []
    fact_list: GroundTruthFactList  # required — G2
    structure: GroundTruthStructure  # required — G4
    conclusion: GroundTruthConclusion  # required — G4
    
    # —— 序列化 ——
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "GroundTruthBundle": ...
```

---

### 7. KnownIssue（known_issues.json）

```python
@dataclass
class KnownIssue:
    """历史报告中的已知错误登记。
    
    issue_type 区分两类错误（v2 R-7，v3 保留）：
        fact_value_error  → fact_id required（nullable-required 有条件非 None）
        conclusion_error  → fact_id 可空（optional-default 允许省略）
    """
    issue_type: str                  # required — "fact_value_error" | "conclusion_error"
    fact_id: Optional[str] = None   # nullable-required（有条件）：
                                    #   - issue_type="fact_value_error" → validator 强制非 None
                                    #   - issue_type="conclusion_error"  → 允许 None
    original_value: Any              # required
    correct_value: Any               # required — original_value != correct_value
    reason: str                       # required
    confirmed_by: str                 # required
    confirmed_at: str                 # required — ISO 8601
    
    # —— 序列化 ——
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "KnownIssue": ...
```

**Validator 检查**：
- issue_type ∈ {"fact_value_error", "conclusion_error"}
- issue_type="fact_value_error" → fact_id 必填、非 None、通过三段式 grammar
- issue_type="conclusion_error" → fact_id 可空
- confirmed_at 合法 ISO 8601
- confirmed_by 非空
- original_value != correct_value

---

### 8. ExpectedIssue（expected_issues.json）

```python
@dataclass
class ExpectedIssue:
    """正式验收对象 — 系统应该发现但没发现的问题。
    
    rule_id 的业务注册表留给 Step 7-E Accuracy Validator。
    """
    rule_id: str                    # required
    severity: str                    # required — "error" | "warning"
    fact_id: Optional[str] = None   # optional-default — 有就检查 grammar，没有跳过
    note: str = ""                  # optional-default
    
    # —— 序列化 ——
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "ExpectedIssue": ...
```

**Validator 检查**：
- rule_id 非空
- severity ∈ {error, warning}
- fact_id（如有）通过三段式 grammar

---

### 9. EvaluationCase（顶层外壳）

```python
@dataclass
class EvaluationCase:
    """评测用例的完整表示。
    
    case_dir 是 Runtime-only（v2 R-4，v3 保留）：
        不属于 Evaluation Case 的语义数据。
        不参与 to_dict / to_json / from_dict。
        from_dir() 使用它加载数据，
        但 JSON 表示不包含机器相关路径。
    """
    # —— 持久化字段（参与序列化）────
    meta: Meta                           # required
    ground_truth: GroundTruthBundle       # required
    known_issues: list[KnownIssue]       # required — 允许 []
    expected_issues: list[ExpectedIssue] # required — 允许 []
    
    # —— Runtime-only 字段（不参与序列化）────
    case_dir: str = field(default="", repr=False, metadata={"runtime_only": True})
    
    # —— 序列化（显式排除 case_dir）────
    def to_dict(self) -> dict: ...
    def to_json(self) -> str: ...
    
    # —— Loader（case_dir 只在 from_dir 中使用）────
    @classmethod
    def from_dir(cls, dir_path: str) -> "EvaluationCase": ...
    
    # —— 顶层 Validator ──
    def validate(self) -> list[str]: ...
```

---

## C. 文件结构

```
src/
  eval/
    __init__.py                    # export 所有公开类和函数
    enums.py                       # CASE_LEVELS / SEVERITIES / ISSUE_TYPES
    types.py                       # 9 个 dataclass
    validator.py                   # 8 个 validate_* 函数
    
tests/
  eval/
    __init__.py
    fixtures/
      case_gold_minimal/
        meta.json
        inputs/                    # 目录只存在，不解析内容
        ground_truth/
          facts.json               # G3 — 非空数组（至少 1 条 GroundTruthFact，每个有 precision）
          fact_list.json           # G2 — 非空 fact_ids，无重复，每条必须在 G3 里有对应
          structure.json           # G4
          conclusion.json          # G4 — direction 非空，required_evaluation_ids 非空，opaque 无重复
        known_issues.json          # [] 或正常条目
        expected_issues.json       # 至少 1 条陷阱
      case_silver_minimal/
        ...                        # 四个 ground_truth 子文件全部非空
                                   # silver 的 "部分 G1" = facts 条目数少 ≠ 空文件
      case_bronze_minimal/
        ...
        ground_truth/
          facts.json               # G3 — **必须为空数组 []**
          fact_list.json           # G2 — **必须非空**（bronze 只有 G2+G4，G2 是事实清单）
          structure.json           # G4 — 非空
          conclusion.json          # G4 — 非空
      case_missing_inputs_dir/     # inputs/ 目录缺 → validate_case 报错
      case_bronze_facts_nonempty/  # bronze 但 facts.json 非空 → validate_case 报错
      case_g3_not_in_g2/           # gold，G3 fact_id 不在 G2 → validate_case 报错
      case_g2_g3_duplicate_ids/    # gold，G2/G3 有重复 fact_id → validate_case 报错
    test_types.py                  # 构造 + JSON roundtrip（含 case_dir 不序列化 / precision 必须显式 / issue_type 两个分支）
    test_validator.py              # 7 个子 validator + validate_case 顶层（含 G2→G3 referential integrity）
    test_loader.py                 # from_dir 集成测试
```

---

## D. Validator 完整规则

### 7 个子 validator（单个文件内的结构检查）

| validator | 检查对象 | 检查项 |
| ---- | ---- | ---- |
| validate_meta | Meta | case_level 枚举 / 所有 required 非空 / created_at ISO 8601 |
| validate_ground_truth_fact | GroundTruthFact | fact_id grammar / quantity_kind 枚举 / precision >= 0 |
| validate_ground_truth_fact_list | GroundTruthFactList | 每个 fact_id grammar / 列表内无重复 |
| validate_ground_truth_structure | GroundTruthStructure | table_count >= 0 / figure_count >= 0 / 元素非空 |
| validate_ground_truth_conclusion | GroundTruthConclusion | direction 非空 / required_evaluation_ids 非空 / 每个 ID 非空 / 列表内无重复 |
| validate_known_issue | KnownIssue | issue_type 枚举 / fact_id 条件检查 / confirmed_at ISO 8601 / confirmed_by 非空 / original_value != correct_value |
| validate_expected_issue | ExpectedIssue | rule_id 非空 / severity 枚举 / fact_id（如有）grammar |

### 顶层 validate_case（跨文件一致性 + case_level 合规性）

| 编号 | 检查项 | 规则 | 来源 |
| ---- | ---- | ---- | ---- |
| T-1 | inputs/ 目录存在 | 所有级别 | 05 §5 |
| T-2 | 四个 ground_truth 子文件存在 | 所有级别（不论空非空） | 05 §5 固定目录结构 |
| T-3 | **G2 / G3 跨文件 referential integrity** | **见 E 节完整规则** | v3 第 1 项修订 |
| T-4 | case_level 与内容的一致性 | 见 F 节完整规则 | 05 §4 |
| T-5 | 所有 required 字段无 KeyError | 所有子文件 | Python from_dict 的 KeyError 由 validate_case 捕获并转为 error 信息 |

---

## E. G2 / G3 / G4 关系（v3 第 1 项修订）

### G2 ↔ G3 referential integrity 规则

**为什么属于 Step 7-D 而不是 Step 7-E**：

G2 → G3 外键约束是**结构完整性**，不是**值正确性**。类比数据库：一张引用了不存在 ID 的外键表，即使数据内容对，也是坏表。这个验证发生在 fixture 被任何运行时系统使用**之前**——如果 G3 引用了 G2 没声明的 fact_id，不管 Accuracy Validator 怎么实现，这个 fixture 就是结构上坏掉的。

```
Step 7-D 验证：
    fixture 内部是否自洽（G3 引用 G2 必须声明过的 fact_id）
    → 结构完整性

Step 7-E 验证：
    G3 的值 vs 运行时输出的值（按 precision round 比对）
    → 值正确性
```

### 三条规则（apply to gold / silver）

| 编号 | 规则 | 实现 |
| ---- | ---- | ---- |
| R-1 | **G3.fact_id ⊆ G2.fact_ids** | `set(g3_ids).issubset(set(g2_ids))` |
| R-2 | **G2.fact_ids 不允许重复** | `len(g2_ids) == len(set(g2_ids))` |
| R-3 | **G3.fact_id 不允许重复** | `len(g3_ids) == len(set(g3_ids))` |

### bronze 的特殊规则

bronze 没有 G3（"仅有报告（G2+G4）"，05 §4），因此：
- R-1 / R-3 在 bronze 中**不适用**（G3 为空）
- R-2 仍然适用（G2.fact_ids 仍然不允许重复）
- 额外约束：**G3 facts 必须为空数组 `[]`**（由 F 节 bronze 规则强制执行）

### 规则总结表

| 规则 | gold | silver | bronze |
| ---- | ---- | ---- | ---- |
| G3.fact_id ⊆ G2.fact_ids | ✅ 必须 | ✅ 必须 | N/A（G3 为空） |
| G2.fact_ids 不重复 | ✅ 必须 | ✅ 必须 | ✅ 必须 |
| G3.fact_id 不重复 | ✅ 必须 | ✅ 必须 | N/A（G3 为空） |

---

## F. gold / silver / bronze 完整规则

### 四个 ground_truth 子文件的存在性 + 内容要求

| ground_truth 子文件 | 语义 | gold | silver | bronze |
| ---- | ---- | ---- | ---- | ---- |
| facts.json | G3 答案值 | **存在 + 非空**（至少 1 条 GroundTruthFact） | **存在 + 非空**（至少 1 条） | **存在 + 必须为空数组 `[]`** |
| fact_list.json | G2 事实清单 | **存在 + 非空**（fact_ids 至少 1 个） | **存在 + 非空** | **存在 + 非空** |
| structure.json | G4 报告形态 | **存在 + 完整** | **存在 + 完整** | **存在 + 完整** |
| conclusion.json | G4 结论 | **存在 + 非空** | **存在 + 非空** | **存在 + 非空** |

**所有级别都必须同时存在四个子文件**。区别只在内容空/非空。

### 附加的跨文件一致性（由 validate_case 强制执行）

| 附加规则 | gold | silver | bronze |
| ---- | ---- | ---- | ---- |
| inputs/ 目录存在 | ✅ | ✅ | ✅ |
| G2 fact_ids 无重复 | ✅ | ✅ | ✅ |
| G3 fact_id ⊆ G2.fact_ids | ✅ | ✅ | N/A |
| G3 fact_id 无重复 | ✅ | ✅ | N/A |

### 为什么 bronze 的 facts.json 必须为空数组

05_EVALUATION.md §4 明确定义 bronze = "仅有报告（G2 + G4）"，G3 是缺失的。如果 bronze fixture 带了非空 facts.json，那么它**实际上有 G3 答案密钥**，与声明的 case_level 矛盾。validator 在检测到 "bronze + 非空 G3" 时报错，不是为了做等级提升，而是为了阻止"声明有 G3 还是没 G3"这件事变得模糊。

---

## G. 测试矩阵

### 第一组：数据结构构造 + JSON roundtrip（test_types.py）

| 编号 | 测试 | 覆盖 |
| ---- | ---- | ---- |
| T1 | Meta 正常构造 + roundtrip（required + optional-default 字段混合） | Meta |
| T2 | GroundTruthFact 定量构造 + **precision 显式提供**（v3 第 3 项） | GroundTruthFact |
| T3 | GroundTruthFact 定性构造（unit=None） + precision 显式提供 | GroundTruthFact |
| T4 | GroundTruthFactList 构造 + roundtrip | GroundTruthFactList |
| T5 | GroundTruthStructure 构造 + roundtrip | GroundTruthStructure |
| T6 | GroundTruthConclusion 构造（direction + required_evaluation_ids） + **opaque ID 示例**（v3 第 4 项） | GroundTruthConclusion |
| T7 | KnownIssue fact_value_error 构造 + roundtrip（fact_id 非 None） | KnownIssue |
| T8 | KnownIssue conclusion_error 构造 + roundtrip（fact_id=None） | KnownIssue |
| T9 | ExpectedIssue 带 fact_id + roundtrip | ExpectedIssue |
| T10 | ExpectedIssue 不带 fact_id + roundtrip | ExpectedIssue |
| T11 | GroundTruthBundle 组装 + roundtrip | GroundTruthBundle |
| T12 | EvaluationCase 完整组装 + roundtrip | EvaluationCase |
| T13 | **EvaluationCase.to_dict() 不含 case_dir**（v2 R-4，v3 保留） | case_dir 不参与序列化 |
| T14 | **from_dict 三类字段行为测试**（v3 第 2 项）：<br>(a) required 字段缺失 → KeyError<br>(b) nullable-required 字段缺 → validator 条件检查<br>(c) optional-default 字段缺 → 使用默认值 | 三类字段语义一致性 |
| T15 | **bronze fixture roundtrip：GroundTruthBundle.facts 为空数组，GroundTruthFactList.fact_ids 非空**（v3 第 5 项修正命名） | G2/G3 正确对应 |

### 第二组：子 validator 基本 pass 路径

| 编号 | 测试 | 覆盖 |
| ---- | ---- | ---- |
| V1 | validate_meta：gold / silver / bronze 各一个合法 | validate_meta |
| V2 | validate_ground_truth_fact：列表全部合法（每个有 precision） | validate_ground_truth_fact |
| V3 | validate_ground_truth_fact_list：列表全部合法 | validate_ground_truth_fact_list |
| V4 | validate_ground_truth_structure：全部合法 | validate_ground_truth_structure |
| V5 | validate_ground_truth_conclusion：direction + required_evaluation_ids（opaque） | validate_ground_truth_conclusion |
| V6 | validate_known_issues：fact_value_error + conclusion_error 各一个 | validate_known_issues |
| V7 | validate_expected_issues：空数组 + 正常数组 | validate_expected_issues |

### 第三组：子 validator 失败路径

| 编号 | 错误 | 期望 |
| ---- | ---- | ---- |
| F1 | Meta.case_level = "platinum" | validate_meta 报错 |
| F2 | Meta.case_id = "" | validate_meta 报错 |
| F3 | Meta.created_at = "not-a-date" | validate_meta 报错 |
| F4 | GroundTruthFact.fact_id grammar 非法 | validate_ground_truth_fact 报错 |
| F5 | GroundTruthFact.fact_id scope_class 非法 | validate_ground_truth_fact 报错 |
| F6 | GroundTruthFact.quantity_kind 非法 | validate_ground_truth_fact 报错 |
| F7 | GroundTruthFact.precision = -1 | validate_ground_truth_fact 报错 |
| **F8** | **GroundTruthFact 缺 precision 字段**（v3 第 3 项：precision 是 required 无默认） | **from_dict KeyError / validate_case 捕获** |
| F9 | fact_list.fact_ids 含空字符串 | validate_ground_truth_fact_list 报错 |
| **F10** | **fact_list.fact_ids 有重复**（v3 第 1 项 R-2） | **validate_ground_truth_fact_list 报错** |
| F11 | KnownIssue.issue_type = "other" | validate_known_issues 报错 |
| F12 | KnownIssue fact_value_error 但 fact_id=None | validate_known_issues 报错 |
| F13 | KnownIssue confirmed_at 非法 | validate_known_issues 报错 |
| F14 | KnownIssue original_value == correct_value | validate_known_issues 报错 |
| F15 | ExpectedIssue.rule_id = "" | validate_expected_issues 报错 |
| F16 | ExpectedIssue.severity = "fatal" | validate_expected_issues 报错 |
| F17 | GroundTruthConclusion.direction = "" | validate_ground_truth_conclusion 报错 |
| F18 | GroundTruthConclusion.required_evaluation_ids = [] | validate_ground_truth_conclusion 报错 |
| F19 | GroundTruthConclusion.required_evaluation_ids 有重复 | validate_ground_truth_conclusion 报错 |

### 第四组：顶层 validate_case（跨文件一致性 —— v3 第 1 项核心）

| 编号 | 场景 | 期望 | 类别 |
| ---- | ---- | ---- | ---- |
| C1 | gold fixture 全量通过 | validate_case 返回 [] | pass |
| C2 | silver fixture 全量通过 | validate_case 返回 [] | pass |
| C3 | **bronze fixture 全量通过**（facts.json=[] + fact_list 非空） | validate_case 返回 [] | pass |
| C4 | 任意级别缺 inputs/ 目录 | validate_case 报错 | 文件存在性 |
| C5 | gold 缺 ground_truth/facts.json | validate_case 报错（四文件固定存在） | 文件存在性 |
| C6 | gold 缺 ground_truth/fact_list.json | validate_case 报错 | 文件存在性 |
| C7 | gold 缺 ground_truth/structure.json | validate_case 报错 | 文件存在性 |
| C8 | gold 缺 ground_truth/conclusion.json | validate_case 报错 | 文件存在性 |
| C9 | gold 用例 facts.json 为空数组 | validate_case 报错（gold 要求非空） | case_level 一致性 |
| C10 | silver 用例 facts.json 为空数组 | validate_case 报错 | case_level 一致性 |
| C11 | **bronze 用例 facts.json 非空** | **validate_case 报错** | **case_level 一致性（v3 第 1 项）** |
| C12 | **gold fixture：G3 fact_id 不在 G2.fact_ids** | **validate_case 报错** | **G2→G3 referential integrity R-1（v3 第 1 项）** |
| C13 | **silver fixture：G3 fact_id 不在 G2.fact_ids** | **validate_case 报错** | **G2→G3 referential integrity R-1** |
| C14 | **gold fixture：G2.fact_ids 有重复** | **validate_case 报错** | **G2/G3 referential integrity R-2** |
| C15 | **gold fixture：G3.fact_id 有重复** | **validate_case 报错** | **G2/G3 referential integrity R-3** |
| C16 | gold fixture 缺 meta.json | validate_case 报错 | 文件存在性 |
| C17 | EvaluationCase.from_dir 加载 bronze fixture（facts.json=`[]`） | 正确解析为空列表 | Loader |
| C18 | EvaluationCase.from_dir to_dict 不含 case_dir | case_dir 不进入序列化 | Loader |

**第四组合计：18 个测试，其中 G2→G3 referential integrity 相关 4 个是 v3 新增（C12 / C13 / C14 / C15）**。

### 测试总数统计

| 组 | 数量 |
| ---- | ---- |
| T（构造 + roundtrip） | 15 |
| V（子 validator pass） | 7 |
| F（子 validator fail） | 15（含 v3 新增 F8 / F10） |
| C（顶层 validate_case） | 18（含 v3 新增 C11–C15 共 5 个） |
| **合计** | **55+** |

---

## H. 与 Step 7-B / Step 7-C 的依赖

### 复用内容（直接 import）

| Step | 复用对象 | 用途 | 位置 |
| ---- | ---- | ---- | ---- |
| 7-B | `is_valid_fact_id()` | GroundTruthFact.fact_id grammar 校验、KnownIssue.fact_id 条件校验、ExpectedIssue.fact_id 校验 | `src/cdm/id.py` |
| 7-B | `parse_fact_id()` | G2→G3 referential integrity 辅助（如有需要） | `src/cdm/id.py` |
| 7-B | `QUANTITY_KINDS` 枚举 | GroundTruthFact.quantity_kind 枚举校验 | `src/cdm/registry.py` |
| 7-B | `SCOPE_CLASSES` 枚举 | 间接（通过 is_valid_fact_id） | `src/cdm/registry.py` |

### 不复用的内容

| 对象 | 为什么 |
| ---- | ---- |
| Step 7-B `Fact` dataclass | GroundTruthFact 语义完全不同——答案密钥 vs 运行时事实 |
| Step 7-B `FactTypeRegistry` | 可选做 optional warning，但 Step 7-D validator 不强制 |
| Step 7-C 所有 Domain Objects | Evaluation Case 不引用 Domain Object id |

### Step 7-B / 7-C 不得修改什么

- **绝对不修改**：`src/cdm/types.py` / `validate.py` / `registry.py` / `id.py` / `domain.py` / `domain_validate.py` 中的任何已有代码
- **绝对不修改**：`tests/cdm/test_types.py` / `test_domain.py` 中的任何已有测试
- **绝对不触碰**：`step-7b-complete` / `step-7c-complete` tag

---

## I. 明确留给 Step 7-E 及以后

| 编号 | 内容 | 留到哪一步 |
| ---- | ---- | ---- |
| L-1 | IssueList 与 expected_issues 命中率比对 | Step 7-E Accuracy Validator |
| L-2 | 数值比对逻辑（G3.value vs 运行输出 value，按 precision round） | Step 7-E |
| L-3 | 判定逻辑一致性比对 | Step 7-E |
| L-4 | 数字锚定比对 | Step 7-E |
| L-5 | 结论覆盖度比对（required_evaluation_ids vs 运行时 covers[]） | Step 7-E |
| L-6 | 结构一致性比对（P1） | Step 7-F |
| L-7 | 文本一致性比对（P2） | Step 7-F |
| L-8 | 版式一致性比对（P3） | Step 7-F |
| L-9 | Run Snapshot / Baseline / 回归报告 | Step 7-G |
| L-10 | 历史报告逆向抽取 | Step 7-I |
| L-11 | LLM / Adapter / Skill / Template / Renderer / DOCX | Step 7-H / 7-J |
| L-12 | Evaluation / Conclusion 对象的正式 grammar / registry / schema | 后续 Evaluation 领域模型步骤 |
| L-13 | 任何数据库 / Agent / UI | 永远不在 Step 7-D 范围 |

---

## J. 最终最小实现范围

### 必须做的

| 编号 | 产物 | 具体内容 |
| ---- | ---- | ---- |
| P-1 | `src/eval/enums.py` | CASE_LEVELS（gold/silver/bronze）、SEVERITIES（error/warning）、ISSUE_TYPES（fact_value_error/conclusion_error）三个 `frozenset` 常量 |
| P-2 | `src/eval/types.py` | **9 个 dataclass**（Meta / GroundTruthFact / GroundTruthFactList / GroundTruthStructure / GroundTruthConclusion / GroundTruthBundle / KnownIssue / ExpectedIssue / EvaluationCase），每个字段明确标注 required / nullable / optional-default |
| P-3 | `src/eval/validator.py` | 7 个子 validator（validate_meta / validate_ground_truth_fact / validate_ground_truth_fact_list / validate_ground_truth_structure / validate_ground_truth_conclusion / validate_known_issue / validate_expected_issue）+ 1 个顶层 validate_case（含 inputs/ 目录存在性 / 四子文件存在性 / case_level 内容一致性 / **G2→G3 referential integrity R-1 R-2 R-3** / **bronze facts 必须为空**） |
| P-4 | `src/eval/__init__.py` | export 所有公开类和函数 |
| P-5 | `tests/eval/test_types.py` | **15 个测试**：构造 + roundtrip + case_dir 不序列化 + precision 显式提供 + T14 三类字段行为测试 + T15 bronze G2/G3 正确对应 |
| P-6 | `tests/eval/test_validator.py` | **40+ 个测试**：7 个子 validator 的 pass（~7）+ fail（~15）+ validate_case 跨文件一致性（18 个含 G2→G3 4 个新增） |
| P-7 | `tests/eval/fixtures/` | **7 个 fixture 目录**（case_gold_minimal / case_silver_minimal / case_bronze_minimal / case_missing_inputs_dir / case_bronze_facts_nonempty / case_g3_not_in_g2 / case_g2_g3_duplicate_ids） |
| P-8 | `tests/eval/test_loader.py` | from_dir 集成测试 |

### 可选但推荐的

| 编号 | 产物 | 说明 |
| ---- | ---- | ---- |
| Q-1 | 每个 dataclass 头部注释标注对应的 05_EVALUATION.md 章节 | 方便后续维护追溯 |
| Q-2 | import Step 7-B 处加注释标注具体复用函数 | 便于定位依赖链 |

### 明确不做的

- 不复用 Step 7-B 的 Fact dataclass
- 不调用 LLM / Adapter / Skill
- 不解析 inputs/ 目录下任何文件内容
- 不建立 Input Manifest 或新的 RawSource / Input 数据模型
- 不实现任何 Store / Pipeline / Agent / Skill
- 不修改 Step 7-B / 7-C 任何代码
- 不修改 05_EVALUATION.md 或其他设计文档
- 不创建 git commit

### 验收标准（Step 7-D 完成的判定）

1. `pytest tests/eval/` 全部通过（55+ 新增测试 + 不影响 Step 7-B/7-C 的 128 个已有测试）
2. `python -c "from eval import *"` 无 ImportError
3. 7 个 fixture 目录的 validator 判定全部符合预期：gold/silver 全量 pass；bronze 全量 pass（facts=[]）；3 个 error fixture（missing_inputs / bronze_facts_nonempty / g3_not_in_g2 / g2_g3_duplicate_ids）全量 fail 且报错信息明确指向具体规则
4. 无任何设计文档或 Step 7-B/7-C 代码被修改
5. 打 tag `step-7d-complete`

---

以上是最终的 v3 Coding Contract。等你确认后再进入执行。
