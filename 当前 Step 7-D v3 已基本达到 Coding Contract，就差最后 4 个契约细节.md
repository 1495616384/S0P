用户要求做 v4，4 个明确的修订点，不重新设计架构。我先确认 advisor 意见，然后直接输出 v4。
正在请求专家建议....

已获取专家建议

好，v4 的所有修订点已经明确。我现在直接输出完整的 v4 Coding Contract，按照用户要求的 A-K 结构组织。


*正在请求专家建议，优化解决方案，该过程耗时可能较长，请耐心等待...*

好。4 项修订直接应用，不重新设计架构。输出 v4。

---

# Step 7-D 实施方案 v4：Evaluation Case 基础层 — Final Coding Contract

> **版本**：v4 — 可直接编码  
> **日期**：2026-09-24  
> **修订**：删除 nullable-required 术语 / 明确 from_dict 与 validate_* 的边界 / G3.value 不允许 None / fixture 数量修正为 3 正常 + 4 错误

---

## A. 最终设计目标

为评测体系建立**程序化、可版本化、可独立验证的评测用例数据结构层**。

Step 7-D 的唯一工作是：**在 fixture 被任何运行时系统接触之前，确保它结构健全**。

| 是 | 不是 |
| ---- | ---- |
| 9 个 dataclass 的定义 + to_dict / from_dict / to_json | 运行时数据流的一部分 |
| 7 个子 validator（单文件内字段语义检查） | 数值比对器 |
| 1 个顶层 validate_case（跨文件 referential integrity + case_level 一致性） | LLM / Store / Pipeline |
| inputs/ 目录存在性检查 | 解析 inputs/ 目录的文件内容 |
| 7 个 fixture（3 正常 + 4 错误） | 执行器 |

### 绝对不做的（不可谈判）

- 不复用 Step 7-B 的 `Fact` dataclass（只用它的 grammar 和枚举）
- 不调用 LLM / Adapter / Skill
- 不解析 inputs/ 目录下任何文件内容
- 不建立 Input Manifest
- 不实现 Store / Pipeline / Agent / Skill
- 不修改 Step 7-B / 7-C 任何代码
- 不修改任何设计文档
- 不创建 git commit

---

## B. 9 个 dataclass + 完整字段定义

### 字段语义分类体系（v4 统一，4 种）

| 类别 | 含义 | dataclass 实现 | JSON 缺失时行为 |
| ---- | ---- | ---- | ---- |
| **required** | JSON 中必须存在，值不能为 None | 无默认值 | `from_dict()` 抛 KeyError |
| **optional-default** | JSON 可以省略，Python 使用默认值 | `field(default=...)` | 省略时使用默认值 |
| **conditional-required** | 根据其他字段决定是否必须非 None | `field(default=None)` + validator 条件检查 | from_dict 允许 None；validator 再按条件强制 |
| **runtime-only** | 不进入持久化 JSON | `field(default="", metadata={"runtime_only": True})` | 参与 from_dir 加载，不参与 to_dict / to_json / from_dict |

---

### B-1. Meta（meta.json）

```python
@dataclass
class Meta:
    """05_EVALUATION.md §5.1"""
    case_id: str               # required
    case_level: str            # required — gold | silver | bronze
    report_type: str            # required — 报告类型标识
    source_project: str         # required — 来源项目标识
    created_at: str             # required — ISO 8601
    
    design_version: Optional[str] = None    # optional-default
    traps: list[str] = field(default_factory=list)  # optional-default
    notes: str = ""                             # optional-default
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "Meta": ...
    def to_json(self) -> str: ...
```

---

### B-2. GroundTruthFact（ground_truth/facts.json — G3）

```python
@dataclass
class GroundTruthFact:
    """G3 事实值 Ground Truth — 轻量答案密钥。
    
    语义区别于 Step 7-B 运行时 Fact：
        运行时 Fact 有 source_refs / method / provenance / review_status /
        status / revision / supersedes —— GroundTruthFact 全都没有。
    
    复用 Step 7-B 的：
        is_valid_fact_id()  — fact_id grammar 校验
        QUANTITY_KINDS      — quantity_kind 枚举校验
    
    precision 显式提供，不允许隐式默认 2（v3 第 3 项决定）。
    value 不允许 None（v4 第 3 项决定）—— CDM Fact missing 用 value=null
    表达，但 G3 是已确认的 Ground Truth，bronze 没有 G3 用 facts=[] 表达。
    """
    fact_id: str                    # required
    value: Any                      # required — 允许 int|float|str|bool，不允许 None（v4）
    unit: Optional[str] = None     # optional-default — 定量事实显式给值，定性事实省略
    quantity_kind: str              # required — QUANTITY_KINDS 枚举
    precision: int                  # required — 显式提供，无默认值
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "GroundTruthFact": ...
```

---

### B-3. GroundTruthFactList（ground_truth/fact_list.json — G2）

```python
@dataclass
class GroundTruthFactList:
    """G2 事实清单。
    
    与 GroundTruthFact 的语义边界：
        G2.fact_ids = 清单（哪些事实应该存在）
        G3.fact_id + value = 答案密钥（这些事实的值是多少）
        G3.fact_id ⊆ G2.fact_ids  —— 结构完整性（Step 7-D 检查）
    """
    fact_ids: list[str]            # required — 列表本身必须存在
                                   #   gold/silver 非空；bronze 非空
                                   #   列表内不允许重复
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "GroundTruthFactList": ...
```

---

### B-4. GroundTruthStructure（ground_truth/structure.json — G4）

```python
@dataclass
class GroundTruthStructure:
    """G4 报告形态 — 章节结构、表格、图片、必需要素。"""
    chapter_paths: list[str]        # required — 允许空列表（无章节路径）
    table_count: int                # required — >= 0
    table_headers: list[list[str]]  # required — 允许空列表（无表格）
    figure_count: int               # required — >= 0
    required_elements: list[str]    # required — 允许空列表
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "GroundTruthStructure": ...
```

---

### B-5. GroundTruthConclusion（ground_truth/conclusion.json — G4）

```python
@dataclass
class GroundTruthConclusion:
    """G4 结论 — 方向 + 覆盖度。
    
    required_evaluation_ids 是 opaque stable identifier：
        Step 7-D 不定义 Evaluation 对象。
        Step 7-D 不定义 Evaluation ID grammar。
        Step 7-D 不创建 evaluation registry。
        只检查：列表非空、每个 ID 非空、不重复。
        示例："evaluation-001" / "evaluation-002"
        不要使用 "evaluation:xxx" 格式。
    
    direction：
        不在 Step 7-D 定义全局 conclusion enum。
        业务取值域由 ConclusionRule + report type 提供。
        validator 只查非空。
    """
    direction: str                              # required
    required_evaluation_ids: list[str]         # required — opaque
    notes: str = ""                             # optional-default
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "GroundTruthConclusion": ...
```

---

### B-6. GroundTruthBundle（顶层容器）

```python
@dataclass
class GroundTruthBundle:
    """四个 ground_truth 子结构的容器。"""
    facts: list[GroundTruthFact]           # required — G3；bronze 时必须为 []
    fact_list: GroundTruthFactList          # required — G2
    structure: GroundTruthStructure          # required — G4
    conclusion: GroundTruthConclusion        # required — G4
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "GroundTruthBundle": ...
```

---

### B-7. KnownIssue（known_issues.json）

```python
@dataclass
class KnownIssue:
    """历史报告中的已知错误登记。
    
    issue_type 区分两类错误：
        fact_value_error → fact_id conditional-required（validator 强制非 None）
        conclusion_error  → fact_id 允许 None
    """
    issue_type: str                  # required — "fact_value_error" | "conclusion_error"
    fact_id: Optional[str] = None   # conditional-required
    original_value: Any              # required
    correct_value: Any               # required — original_value != correct_value
    reason: str                       # required
    confirmed_by: str                 # required
    confirmed_at: str                 # required — ISO 8601
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "KnownIssue": ...
```

---

### B-8. ExpectedIssue（expected_issues.json）

```python
@dataclass
class ExpectedIssue:
    """正式验收对象。"""
    rule_id: str                    # required
    severity: str                    # required — "error" | "warning"
    fact_id: Optional[str] = None   # optional-default
    note: str = ""                  # optional-default
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "ExpectedIssue": ...
```

---

### B-9. EvaluationCase（顶层外壳）

```python
@dataclass
class EvaluationCase:
    """评测用例的完整表示。"""
    # ── 持久化字段 ──
    meta: Meta                           # required
    ground_truth: GroundTruthBundle       # required
    known_issues: list[KnownIssue]       # required — 允许 []
    expected_issues: list[ExpectedIssue] # required — 允许 []
    
    # ── Runtime-only 字段 ──
    case_dir: str = field(
        default="", 
        repr=False, 
        metadata={"runtime_only": True}
    )
    
    # ── 序列化（显式排除 case_dir）──
    def to_dict(self) -> dict: ...
    def to_json(self) -> str: ...
    
    # ── Loader（case_dir 只在 from_dir 中使用）──
    @classmethod
    def from_dir(cls, dir_path: str) -> "EvaluationCase": ...
    
    # ── 顶层 Validator ──
    def validate(self) -> list[str]: ...
```

---

## C. 字段语义分类速查表

### Meta

| 字段 | 类别 |
| ---- | ---- |
| case_id | required |
| case_level | required |
| report_type | required |
| source_project | required |
| created_at | required |
| design_version | optional-default |
| traps | optional-default |
| notes | optional-default |

### GroundTruthFact

| 字段 | 类别 |
| ---- | ---- |
| fact_id | required |
| value | required（不允许 None） |
| unit | optional-default |
| quantity_kind | required |
| precision | required |

### GroundTruthFactList

| 字段 | 类别 |
| ---- | ---- |
| fact_ids | required |

### GroundTruthStructure

| 字段 | 类别 |
| ---- | ---- |
| chapter_paths | required |
| table_count | required |
| table_headers | required |
| figure_count | required |
| required_elements | required |

### GroundTruthConclusion

| 字段 | 类别 |
| ---- | ---- |
| direction | required |
| required_evaluation_ids | required |
| notes | optional-default |

### KnownIssue

| 字段 | 类别 |
| ---- | ---- |
| issue_type | required |
| fact_id | conditional-required |
| original_value | required |
| correct_value | required |
| reason | required |
| confirmed_by | required |
| confirmed_at | required |

### ExpectedIssue

| 字段 | 类别 |
| ---- | ---- |
| rule_id | required |
| severity | required |
| fact_id | optional-default |
| note | optional-default |

### EvaluationCase

| 字段 | 类别 |
| ---- | ---- |
| meta | required |
| ground_truth | required |
| known_issues | required |
| expected_issues | required |
| case_dir | runtime-only |

---

## D. 文件结构

```
src/
  eval/
    __init__.py
    enums.py                       # CASE_LEVELS / SEVERITIES / ISSUE_TYPES
    types.py                       # 9 个 dataclass
    validator.py                   # 7 个子 validator + validate_case

tests/
  eval/
    __init__.py
    fixtures/                       # 7 个 fixture —— 3 正常 + 4 错误（v4）
      # ── 正常 3 个 ──
      case_gold_minimal/
        meta.json
        inputs/                     # 目录存在，不解析
        ground_truth/
          facts.json                # 非空，每条有 precision + value != None
          fact_list.json            # 非空 fact_ids，无重复，每条在 facts 里有对应
          structure.json
          conclusion.json           # direction 非空，required_evaluation_ids 非空无重复
        known_issues.json           # [] 或正常条目
        expected_issues.json        # 至少 1 条陷阱
      case_silver_minimal/
        ...
      case_bronze_minimal/
        ...
        ground_truth/
          facts.json                # **必须为空数组 []**
          fact_list.json            # **必须非空**
          ...
      # ── 错误 4 个（v4 修正）──
      case_missing_inputs_dir/
        ...                         # 缺 inputs/ 目录 → validate_case 报错
      case_bronze_facts_nonempty/
        ...                         # bronze 但 facts.json 非空 → validate_case 报错
      case_g3_not_in_g2/
        ...                         # gold，G3 fact_id 不在 G2 → validate_case 报错
      case_g2_g3_duplicate_ids/
        ...                         # gold，G2 或 G3 有重复 fact_id → validate_case 报错
    test_types.py                   # 构造 + roundtrip + from_dict KeyError 测试
    test_validator.py               # 7 个子 validator + validate_case
    test_loader.py                  # from_dir 集成测试（含文件不存在 / JSON 解析失败）
```

---

## E. 完整 Validator 规则

### 关键边界（v4 第 2 项）

```
JSON 文本
    ↓  json.load()
dict
    ↓  from_dict()   ←── required key 缺失直接抛 KeyError
对象构造完成
    ↓
validate_*()          ←── 字段语义检查（单文件内）
validate_case()       ←── 跨文件一致性 + case_level 合规性
```

**validate_case 不捕获 KeyError**。KeyError 的捕获和测试在 test_types.py / test_loader.py 中完成。

---

### 7 个子 validator

| validator | 检查对象 | 检查项 |
| ---- | ---- | ---- |
| validate_meta | Meta | (1) case_level ∈ {gold, silver, bronze} (2) case_id/report_type/source_project 非空非纯空白 (3) created_at 可解析为 ISO 8601 |
| validate_ground_truth_fact | GroundTruthFact | (1) fact_id 通过三段式 grammar（复用 Step 7-B `is_valid_fact_id()`）(2) quantity_kind ∈ QUANTITY_KINDS（复用 Step 7-B）(3) precision >= 0 (4) **value is not None**（v4 第 3 项）(5) **isinstance(value, (int, float, str, bool))**（v4 第 3 项） |
| validate_ground_truth_fact_list | GroundTruthFactList | (1) 每个 fact_id 通过三段式 grammar (2) **列表内无重复**（`len(ids) == len(set(ids))`） |
| validate_ground_truth_structure | GroundTruthStructure | (1) table_count >= 0 (2) figure_count >= 0 (3) chapter_paths 元素非空 (4) table_headers 子列表元素非空 (5) required_elements 元素非空 |
| validate_ground_truth_conclusion | GroundTruthConclusion | (1) direction 非空 (2) required_evaluation_ids 非空列表 (3) 每个元素非空 (4) **列表内无重复**（`len(ids) == len(set(ids))`） |
| validate_known_issue | KnownIssue | (1) issue_type ∈ {"fact_value_error", "conclusion_error"} (2) issue_type="fact_value_error" → **fact_id 必须非 None + 通过 grammar**（conditional-required）(3) confirmed_at 合法 ISO 8601 (4) confirmed_by 非空 (5) original_value != correct_value |
| validate_expected_issue | ExpectedIssue | (1) rule_id 非空 (2) severity ∈ {"error", "warning"} (3) fact_id（如有）通过三段式 grammar |

---

### 顶层 validate_case（跨文件一致性）

| 编号 | 规则 | 适用 |
| ---- | ---- | ---- |
| T-1 | inputs/ 目录存在 | 所有级别 |
| T-2 | 四个 ground_truth 子文件存在 | 所有级别（不论空非空） |
| T-3 | G2 → G3 referential integrity（见 F 节） | gold / silver |
| T-4 | case_level 与内容一致性（见 G 节） | 所有级别 |

---

## F. G2 / G3 / G4 关系

### G2 ↔ G3 referential integrity（三条规则）

这是**结构完整性**，不是**值正确性**。类比数据库 foreign key——引用不存在 ID 的表，即使内容对也是坏表。

| 编号 | 规则 | gold | silver | bronze |
| ---- | ---- | ---- | ---- | ---- |
| R-1 | **G3.fact_id ⊆ G2.fact_ids** | ✅ 必须 | ✅ 必须 | N/A（G3 为空） |
| R-2 | **G2.fact_ids 不允许重复** | ✅ 必须 | ✅ 必须 | ✅ 必须 |
| R-3 | **G3.fact_id 不允许重复** | ✅ 必须 | ✅ 必须 | N/A（G3 为空） |

---

## G. gold / silver / bronze 完整规则

### 四个 ground_truth 子文件的存在性 + 内容要求

| ground_truth 子文件 | 语义 | gold | silver | bronze |
| ---- | ---- | ---- | ---- | ---- |
| facts.json | G3 答案值 | 存在 + **非空** | 存在 + **非空** | 存在 + **必须为空数组 `[]`** |
| fact_list.json | G2 事实清单 | 存在 + **非空** | 存在 + **非空** | 存在 + **非空** |
| structure.json | G4 报告形态 | 存在 + 完整 | 存在 + 完整 | 存在 + 完整 |
| conclusion.json | G4 结论 | 存在 + **非空** | 存在 + **非空** | 存在 + **非空** |

### 附加跨文件一致性（validate_case 执行）

| 附加规则 | gold | silver | bronze |
| ---- | ---- | ---- | ---- |
| inputs/ 目录存在 | ✅ | ✅ | ✅ |
| G2 fact_ids 无重复 | ✅ | ✅ | ✅ |
| G3.fact_id ⊆ G2.fact_ids | ✅ | ✅ | N/A |
| G3.fact_id 无重复 | ✅ | ✅ | N/A |

**bronze 定义依据**：05_EVALUATION.md §4 明确定义 bronze = "仅有报告（G2 + G4）"，G3 缺失。validator 检测到 "bronze + 非空 G3" 报错——不是等级提升，而是阻止语义模糊。

---

## H. 测试矩阵

### 第一组：test_types.py（构造 + roundtrip + KeyError）

| 编号 | 测试 | 覆盖 |
| ---- | ---- | ---- |
| T1 | Meta 正常构造 + roundtrip | Meta |
| T2 | GroundTruthFact 定量构造（precision 显式） + roundtrip | GroundTruthFact |
| T3 | GroundTruthFact 定性构造（unit=None） + roundtrip | GroundTruthFact |
| T4 | GroundTruthFactList 构造 + roundtrip | GroundTruthFactList |
| T5 | GroundTruthStructure 构造 + roundtrip | GroundTruthStructure |
| T6 | GroundTruthConclusion 构造（opaque required_evaluation_ids） + roundtrip | GroundTruthConclusion |
| T7 | KnownIssue fact_value_error + roundtrip | KnownIssue |
| T8 | KnownIssue conclusion_error（fact_id=None） + roundtrip | KnownIssue |
| T9 | ExpectedIssue 带 fact_id + roundtrip | ExpectedIssue |
| T10 | ExpectedIssue 不带 fact_id + roundtrip | ExpectedIssue |
| T11 | GroundTruthBundle 组装 + roundtrip | GroundTruthBundle |
| T12 | EvaluationCase 完整组装 + roundtrip | EvaluationCase |
| T13 | EvaluationCase.to_dict() **不含 case_dir** | runtime-only 字段不序列化 |
| T14 | **from_dict 三类字段行为测试**：(a) required 字段缺 → KeyError (b) optional-default 字段缺 → 使用默认值 (c) conditional-required → 允许 None 进入对象，validator 后续检查 | 字段语义一致性 |
| T15 | bronze fixture roundtrip：**GroundTruthBundle.facts=[]，GroundTruthFactList.fact_ids 非空** | G2/G3 正确对应 |

### 第二组：test_validator.py —— 7 个子 validator 基本 pass

| 编号 | 测试 |
| ---- | ---- |
| V1 | validate_meta：gold / silver / bronze 各一个合法 |
| V2 | validate_ground_truth_fact：列表全部合法 |
| V3 | validate_ground_truth_fact_list：列表全部合法 |
| V4 | validate_ground_truth_structure：全部合法 |
| V5 | validate_ground_truth_conclusion：direction + required_evaluation_ids（opaque） |
| V6 | validate_known_issues：fact_value_error + conclusion_error 各一个 |
| V7 | validate_expected_issues：空数组 + 正常数组 |

### 第三组：test_validator.py —— 7 个子 validator 失败路径

| 编号 | 错误 | 期望 |
| ---- | ---- | ---- |
| F1 | Meta.case_level = "platinum" | validate_meta 报错 |
| F2 | Meta.case_id = "" | validate_meta 报错 |
| F3 | Meta.created_at = "not-a-date" | validate_meta 报错 |
| F4 | GroundTruthFact.fact_id grammar 非法 | validate_ground_truth_fact 报错 |
| F5 | GroundTruthFact.fact_id scope_class 非法 | validate_ground_truth_fact 报错 |
| F6 | GroundTruthFact.quantity_kind 非法 | validate_ground_truth_fact 报错 |
| F7 | GroundTruthFact.precision = -1 | validate_ground_truth_fact 报错 |
| **F8** | **GroundTruthFact.value = None**（v4 第 3 项） | **validate_ground_truth_fact 报错** |
| **F9** | **GroundTruthFact.value = [1,2,3]**（非 int/float/str/bool）（v4 第 3 项） | **validate_ground_truth_fact 报错** |
| F10 | fact_list.fact_ids 含空字符串 | validate_ground_truth_fact_list 报错 |
| **F11** | **fact_list.fact_ids 有重复** | **validate_ground_truth_fact_list 报错** |
| F12 | KnownIssue.issue_type = "other" | validate_known_issue 报错 |
| F13 | KnownIssue fact_value_error 但 fact_id=None | validate_known_issue 报错（conditional-required） |
| F14 | KnownIssue confirmed_at 非法 | validate_known_issue 报错 |
| F15 | KnownIssue original_value == correct_value | validate_known_issue 报错 |
| F16 | ExpectedIssue.rule_id = "" | validate_expected_issue 报错 |
| F17 | ExpectedIssue.severity = "fatal" | validate_expected_issue 报错 |
| F18 | GroundTruthConclusion.direction = "" | validate_ground_truth_conclusion 报错 |
| F19 | GroundTruthConclusion.required_evaluation_ids = [] | validate_ground_truth_conclusion 报错 |
| F20 | GroundTruthConclusion.required_evaluation_ids 有重复 | validate_ground_truth_conclusion 报错 |

### 第四组：test_validator.py —— 顶层 validate_case（跨文件一致性）

| 编号 | 场景 | 期望 |
| ---- | ---- | ---- |
| C1 | gold fixture 全量通过 | validate_case 返回 [] |
| C2 | silver fixture 全量通过 | validate_case 返回 [] |
| C3 | bronze fixture 全量通过（facts.json=[] + fact_list 非空） | validate_case 返回 [] |
| C4 | 任意级别缺 inputs/ 目录 | validate_case 报错 |
| C5 | gold 缺 ground_truth/facts.json | validate_case 报错 |
| C6 | gold 缺 ground_truth/fact_list.json | validate_case 报错 |
| C7 | gold 缺 ground_truth/structure.json | validate_case 报错 |
| C8 | gold 缺 ground_truth/conclusion.json | validate_case 报错 |
| C9 | gold facts.json 为空数组 | validate_case 报错 |
| C10 | silver facts.json 为空数组 | validate_case 报错 |
| **C11** | **bronze facts.json 非空** | **validate_case 报错** |
| **C12** | **gold：G3 fact_id 不在 G2.fact_ids** | **validate_case 报错（R-1）** |
| **C13** | **silver：G3 fact_id 不在 G2.fact_ids** | **validate_case 报错（R-1）** |
| **C14** | **gold：G2.fact_ids 有重复** | **validate_case 报错（R-2）** |
| **C15** | **gold：G3.fact_id 有重复** | **validate_case 报错（R-3）** |
| C16 | 任意级别缺 meta.json | validate_case 报错 |
| C17 | bronze fixture facts.json=`[]` 正确解析 | from_dir 测试 |
| C18 | from_dir to_dict 不含 case_dir | case_dir runtime-only 测试 |

### 第五组：test_loader.py

| 编号 | 测试 |
| ---- | ---- |
| L1 | from_dir 正常加载 gold fixture，case_dir 正确设置 |
| L2 | from_dir 正常加载 silver fixture |
| L3 | from_dir 正常加载 bronze fixture（facts.json=`[]` 正确解析为 []） |
| **L4** | **from_dir 加载缺失 inputs/ 目录的 fixture → 直接抛异常（不走到 validate_case）**（v4 第 2 项） |
| **L5** | **from_dir 加载缺失 ground_truth/facts.json 的 fixture → 直接抛异常**（v4 第 2 项） |
| **L6** | **from_dir 加载 meta.json 缺 case_id 的 fixture → KeyError**（v4 第 2 项） |

### 测试总数统计

| 组 | 数量 |
| ---- | ---- |
| T（test_types.py） | 15 |
| V（validator pass） | 7 |
| F（validator fail） | 20（含 v4 新增 F8 / F9） |
| C（validate_case） | 18 |
| L（test_loader.py） | 6 |
| **合计** | **66 个** |

### Fixture 总数

**7 个 fixture = 3 正常 + 4 错误**（v4 第 4 项修正）

| 类型 | 数量 | 列表 |
| ---- | ---- | ---- |
| 正常 | 3 | case_gold_minimal / case_silver_minimal / case_bronze_minimal |
| 错误 | 4 | case_missing_inputs_dir / case_bronze_facts_nonempty / case_g3_not_in_g2 / case_g2_g3_duplicate_ids |

---

## I. 7-B / 7-C 依赖

### 复用内容（直接 import）

| Step | 复用对象 | 用途 | 位置 |
| ---- | ---- | ---- | ---- |
| 7-B | `is_valid_fact_id()` | 所有 fact_id grammar 校验 | `src/cdm/id.py` |
| 7-B | `parse_fact_id()` | G2→G3 辅助（如需要） | `src/cdm/id.py` |
| 7-B | `QUANTITY_KINDS` 枚举 | GroundTruthFact.quantity_kind 枚举 | `src/cdm/registry.py` |

### 不复用的内容

| 对象 | 原因 |
| ---- | ---- |
| Step 7-B `Fact` dataclass | GroundTruthFact 语义完全不同 |
| Step 7-B `FactTypeRegistry` | validator 不强制 |
| Step 7-C Domain Objects | Evaluation Case 不引用 Domain Object id |

### 不得修改

- **绝对不修改**：`src/cdm/` 下所有已有代码 / `tests/cdm/` 下所有已有测试 / `step-7b-complete` / `step-7c-complete` tag

---

## J. 明确留给 Step 7-E 及以后

| 编号 | 内容 | 步骤 |
| ---- | ---- | ---- |
| L-1 | IssueList vs expected_issues 命中率比对 | 7-E Accuracy Validator |
| L-2 | G3.value vs 运行输出 value（按 precision round） | 7-E |
| L-3 | 判定逻辑一致性比对 | 7-E |
| L-4 | 数字锚定比对 | 7-E |
| L-5 | 结论覆盖度比对（required_evaluation_ids vs 运行时 covers[]） | 7-E |
| L-6 | 结构 / 文本 / 版式一致性（P1 / P2 / P3） | 7-F |
| L-7 | Run Snapshot / Baseline / 回归报告 / 指标计算 | 7-G |
| L-8 | 历史报告逆向抽取 | 7-I |
| L-9 | LLM / Adapter / Skill / Template / Renderer / DOCX | 7-H / 7-J |
| L-10 | Evaluation / Conclusion 对象的正式 grammar / registry / schema | 后续领域模型步骤 |
| L-11 | 数据库 / Agent / UI | 永远不在 Step 7-D 范围 |

---

## K. 最终 Coding Contract（可直接编码）

### 文件清单

| 编号 | 路径 | 内容 |
| ---- | ---- | ---- |
| 1 | `src/eval/__init__.py` | export 所有公开类和函数 |
| 2 | `src/eval/enums.py` | 三个 `frozenset` 常量：CASE_LEVELS = frozenset({"gold","silver","bronze"})、SEVERITIES = frozenset({"error","warning"})、ISSUE_TYPES = frozenset({"fact_value_error","conclusion_error"}) |
| 3 | `src/eval/types.py` | **9 个 dataclass**（§B-1 ~ B-9） |
| 4 | `src/eval/validator.py` | **7 个子 validator + 1 个 validate_case**（§E） |
| 5 | `tests/eval/__init__.py` | 空文件 |
| 6 | `tests/eval/test_types.py` | **15 个测试** |
| 7 | `tests/eval/test_validator.py` | **7 pass + 20 fail + 18 validate_case = 45 个测试** |
| 8 | `tests/eval/test_loader.py` | **6 个测试** |
| 9 | `tests/eval/fixtures/` | **7 个 fixture 目录** |

### 验收标准

1. `pytest tests/eval/` 全部通过（66 新增测试 + 128 个 Step 7-B/7-C 已有测试不受影响）
2. `python -c "from eval import *"` 无 ImportError
3. 7 个 fixture validator 判定正确：3 个正常全量 pass + 4 个错误全量 fail 且报错信息指向具体规则
4. **零修改** 05_EVALUATION.md / 03_CANONICAL_DATA_MODEL.md / 02_ARCHITECTURE.md / Step 7-B / Step 7-C
5. 打 tag `step-7d-complete`

---

以上是 v4 Final Coding Contract。等你确认后再进入编码阶段。
