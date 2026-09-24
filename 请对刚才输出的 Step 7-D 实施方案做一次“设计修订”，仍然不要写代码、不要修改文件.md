The user is asking me to produce a revised Step 7-D implementation plan (v2), not code, not file modifications. Let me first call AdvisorTool to get strategic guidance before committing to the revised approach.

Let me analyze each of the 6 mandatory corrections plus the 2 additional reviews:

1. **GroundTruthConclusion** - needs `required_evaluation_ids: list[str]` for P0 coverage check
2. **inputs** - validator only checks directory existence, no parsing
3. **bronze** - ground_truth/facts.json MUST be empty for bronze
4. **case_dir** - runtime only, not in to_dict/to_json
5. **dataclass count** - unified to 9
6. **GroundTruthConclusion.direction** - keep as string, no global enum

Additional reviews:
1. known_issues fact_id: should it be Optional?
2. ground_truth sub-files: which ones are required per level?

Let me also re-read the relevant sections of the design docs to make sure I get everything right. But actually I already read them all thoroughly. Let me call AdvisorTool first as required.
正在请求专家建议...

已获取专家建议

好，Advisor 的建议全部采纳。现在输出完整的 v2 修订方案，逐一对应用户提出的 6 个必须修正和 2 个额外检查项。

让我把每个修正点都落实清楚：

**必须修正 1：GroundTruthConclusion**
- 补充 `required_evaluation_ids: list[str]`
- 解释 answer key（声明什么必须覆盖）vs runtime covers[]（系统声称覆盖了什么）的语义区别

**必须修正 2：inputs**
- Meta 不带 input_files / input_manifest
- validator 只检查 `inputs/` 目录存在
- 不解析、不建立新数据模型

**必须修正 3：bronze**
- bronze 的 ground_truth/facts.json **必须为空数组**
- 这与 05_EVALUATION.md §4 完全一致（bronze 仅有 G2+G4）
- 不是自动提升机制，是声明与内容一致性检查

**必须修正 4：case_dir**
- 不参与 to_dict / to_json / from_dict
- from_dir() 使用它加载
- EvaluationCase 的 JSON 表示不含机器路径

**必须修正 5：dataclass 数量**
- 9 个，统一修正

**必须修正 6：direction**
- 保留为普通 str
- validator 只检查非空
- 业务取值域由 ConclusionRule / report type 决定

**额外检查 1：known_issues.fact_id**
- conclusion error 无法自然映射到 fact_id
- 改为 Optional[str] + issue_type 字段

**额外检查 2：ground_truth 四个子文件**
- 全部固定存在，内容的空/非空按 level 区分
- gold：全部非空
- silver：全部非空（"部分 G1" 指条目数少，不是空文件）
- bronze：facts.json 必须空数组 []；其余三个非空

好，现在整合输出完整的 v2 方案。


*正在请求专家建议，优化解决方案，该过程耗时可能较长，请耐心等待...*

好。现在输出完整的《Step 7-D 实施方案 v2》，每一处修订都显式标注。

---

# Step 7-D 实施方案 v2：Evaluation Case 基础层

> **版本**：v2 — 架构复核修订版  
> **修订时间**：2026-09-24  
> **修订来源**：架构复核（6 项必须修正 + 2 项额外检查）

---

## 修订摘要

| 编号 | 修订项 | v1 问题 | v2 修正 |
| ---- | ---- | ---- | ---- |
| R-1 | GroundTruthConclusion 缺覆盖度字段 | 只有 direction，缺少 P0 所需的 required_evaluation_ids | 补充 `required_evaluation_ids: list[str]`，明确语义区别 |
| R-2 | inputs/ 处理内部矛盾 | Meta 不带 input manifest 但又说"按声明检查" | Meta 不带 input 字段；validator **只检查 inputs/ 目录存在** |
| R-3 | bronze facts.json 语义模糊 | 允许 bronze 带非空 facts.json | bronze 的 facts.json **必须为空数组 `[]`**（与 05 §4 完全一致） |
| R-4 | EvaluationCase.case_dir 参与序列化 | case_dir 是机器路径，不应进 JSON | case_dir 转为 runtime-only，排除在 to_dict/to_json/from_dict 外 |
| R-5 | dataclass 计数错误 | 说 8 个实际列了 9 个 | 统一为 9 个 |
| R-6 | direction 业务取值域 | 想凭空建全局 conclusion enum | 保留 str，validator 只查非空；取值域由 ConclusionRule / report type 提供 |
| R-7 | known_issues.fact_id 无法覆盖结论错误 | 结论错误无法映射到 fact_id | fact_id 改为 Optional[str] + issue_type 字段区分两类错误 |
| R-8 | ground_truth 子文件存在性模糊 | 用了"至少一个子文件" | 四个子文件**全部固定存在**，内容空/非空按 level 明确区分 |

---

## A. 对 Step 7-D 的准确理解（v2 保留）

**意图**：为评测体系建立一个**程序化、可版本化、可独立验证的评测用例数据结构层**。

它的定位是：
- **不是**运行时数据流的一部分（不像 Facts 被 Pipeline 消费）
- 是**评测固件**——被 M7 eval/loader 加载、被 Regression Runner 读取的只读 JSON 数据载体
- 语义完全由 `05_EVALUATION.md` 定义，核心是 G1/G2/G3/G4 分离和 case level 分级
- 必须在**没有任何 LLM、没有任何比对逻辑、没有任何执行器**的前提下独立校验通过

**一句话定位**：Step 7-D = `05_EVALUATION.md §5` 的程序化实现，且只实现其**数据结构 + Schema 校验**子集。

---

## B. 当前架构中已确定的约束（v2 补充）

| 编号 | 约束 | 来源 |
| ---- | ---- | ---- |
| C-1 | G1/G2/G3/G4 四类信息必须**分离存储**，目录结构已由 05 §5 冻结 | 05_EVALUATION.md §5 / D-012 |
| C-2 | G3 facts 的值必须来自 G1 **独立推导**，禁止抄历史报告正文 | 05_EVALUATION.md §2 / D-012 |
| C-3 | Case level 分 **gold / silver / bronze**，且**声明后不可提升** | 05_EVALUATION.md §4 |
| C-4 | **gold**：G1 + G3 + G4 齐全；**silver**：部分 G1 + 对应 G3；**bronze**：**仅有 G2 + G4** | 05_EVALUATION.md §4 |
| C-5 | `expected_issues` 是**正式验收对象**，不是备注字段 | 05_EVALUATION.md §5.2 / §7 |
| C-6 | `known_issues` 登记历史报告已知错误，结论错误无法映射到 fact_id | 05_EVALUATION.md §3.2 / 本次额外检查 |
| C-7 | fact_id 必须符合 Step 7-B 冻结的**三段式 grammar** | 03_CDM v0.3.1 / D-027 |
| C-8 | Evaluation Case 不**重新定义** Fact；引用 fact_id 应**复用已有 grammar 契约** | 项目规则 §7 |
| C-9 | Step 7-B / 7-C 已完成的代码**不得修改** | 用户明确要求 |
| C-10 | 本阶段只做：数据结构 + 序列化 / 反序列化 + 基础 schema validator + 单元测试 | 用户明确要求 |
| C-11 | 中间产物全部 JSON 落盘 | 02_ARCHITECTURE.md D-12 |
| C-12 | Python 实现，dataclasses + typing + 标准 json + pytest | project_memory |
| **C-13（新）** | **bronze 的 ground_truth/facts.json 必须为空数组 `[]`** | 05_EVALUATION.md §4 明确定义 bronze = "仅有报告（G2 + G4）" |
| **C-14（新）** | **inputs/ 目录只需检查存在，Meta 不带 input manifest，不解析原始文件** | 本次 R-2 修正 |
| **C-15（新）** | **四个 ground_truth 子文件全部固定存在，内容空/非空按 level 区分** | 本次 R-8 修正 |

---

## C. 建议的数据结构（v2）

### 方案选择：方案 B（EvaluationCase 外壳 + 独立子结构）

四维度判断与 v1 相同，方案 B 胜出。理由不重复。

**核心设计判断（与 v1 一致，但重申一次）**：

GroundTruth.facts.json 是**评测答案密钥**，不是**运行时 Fact 集合**。运行时 Fact 有 source_refs / method / provenance / review_status / status / revision / supersedes / value_domain——答案密钥全都没有。因此：
- **复用 Step 7-B 的 `is_valid_fact_id()` / `parse_fact_id()` / `QUANTITY_KINDS` 枚举**（契约复用）
- **不复用 Step 7-B 的 `Fact` dataclass**（语义解耦）

---

### 1. Meta（meta.json）

```python
@dataclass
class Meta:
    case_id: str               # 必填，如 "case-001"
    case_level: str            # 必填，gold | silver | bronze
    report_type: str            # 必填，报告类型标识（如 "structure_safety_appraisal"）
    source_project: str         # 必填，来源项目标识
    created_at: str             # 必填，ISO 8601
    design_version: str         # 可选，指向哪版设计文档，如 "05_EVALUATION_v0.3"
    traps: list[str]            # 可选，陷阱标注
    notes: str                  # 可选，自由文本备注
```

**Meta 不带任何 input 文件字段**（修正 R-2）。inputs/ 的处理完全由目录存在性检查负责。

**Validator 检查**：
- case_level ∈ {gold, silver, bronze}
- case_id / report_type / source_project 非空、非纯空白
- created_at 可解析为合法 ISO 8601

---

### 2. GroundTruthFact（ground_truth/facts.json 的条目）

```python
@dataclass
class GroundTruthFact:
    """轻量答案密钥 — 不是运行时 Fact。

    运行时 Fact 有 source_refs / method / provenance / review_status /
    status / revision / supersedes / value_domain —— ground truth 全都没有。
    只保留评测比对所需的最小字段。

    复用 Step 7-B 的 is_valid_fact_id() 做 grammar 校验，
    不复用 Step 7-B 的 Fact dataclass。
    """
    fact_id: str                    # 三段式 grammar，复用 Step 7-B 校验
    value: Any                      # 与运行时 Fact.value 类型相同（float|int|str|bool|None）
    unit: Optional[str]             # 与运行时 Fact.unit 语义相同
    quantity_kind: str              # 必须匹配 Step 7-B 已注册枚举
    precision: int                  # 显示精度，用于 P0 比对的 round 刻度；默认 2
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "GroundTruthFact": ...
```

**Validator 检查**：
- fact_id 通过三段式 grammar（`is_valid_fact_id()`）
- quantity_kind ∈ QUANTITY_KINDS（Step 7-B registry）
- precision >= 0
- （bronze 用例：整个 facts 列表必须为空数组——由顶层 validate_case 执行）

---

### 3. GroundTruthFactList（ground_truth/fact_list.json）

```python
@dataclass
class GroundTruthFactList:
    """G2 事实清单 — 这份报告应当涉及哪些事实项。
    
    与 GroundTruthFact（G3 答案密钥）的区别：
    - GroundTruthFactList.fact_ids 只回答"应当有哪些事实"
    - GroundTruthFact.value 回答"这些事实的值是多少"
    """
    fact_ids: list[str]
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "GroundTruthFactList": ...
```

**Validator 检查**：
- 每个 fact_id 通过三段式 grammar

---

### 4. GroundTruthStructure（ground_truth/structure.json）

```python
@dataclass
class GroundTruthStructure:
    """G4 报告形态 — 章节结构、表格、图片、必需要素。"""
    chapter_paths: list[str]        # 章节路径集合
    table_count: int                # 表格数量
    table_headers: list[list[str]]  # 每个表格的列头
    figure_count: int               # 图片数量
    required_elements: list[str]    # 必需要素（如 "signature_block"）
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "GroundTruthStructure": ...
```

**Validator 检查**：
- table_count >= 0, figure_count >= 0
- chapter_paths / required_elements 列表元素非空
- table_headers 的每个子列表元素非空

---

### 5. GroundTruthConclusion（ground_truth/conclusion.json）

```python
@dataclass
class GroundTruthConclusion:
    """G4 结论 — 方向 + 覆盖度（v2 修正 R-1）。"""
    direction: str                  # 必填，业务取值域由 ConclusionRule / report type 提供
    required_evaluation_ids: list[str]   # 必填，P0 覆盖度比对的答案密钥
    notes: Optional[str] = None     # 可选
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "GroundTruthConclusion": ...
```

**v2 新增字段的语义说明**（R-1 修正）：

`required_evaluation_ids` 与运行时的 `covers[]` 是**两个不同角色**：

| 角色 | 所有者 | 语义 | 什么时候填 |
| ---- | ---- | ---- | ---- |
| `required_evaluation_ids` | Ground Truth（评测固件） | **答案密钥**："这份用例有 N 个 Evaluation，每一个都必须被覆盖" | 逆向链路构建评测用例时，人工确认后冻结 |
| `covers[]` | 运行时 Report IR 结论段 | **系统的声称**："我在这条结论里覆盖了这些 Evaluation" | 系统在 Step5 compose 时由程序生成 |

Step 7-E Accuracy Validator 的比对逻辑（不在本阶段实现）：

```text
评测比对（由 Step 7-E 实现，不是 Step 7-D）：
    每个 required_evaluation_id
        必须出现在某个 Report IR 结论段的 covers[] 中
        否则 → P0 error
```

**Validator 检查**：
- direction 非空（v2 R-6 修正：不做全局 enum，业务取值域在 ConclusionRule 层）
- required_evaluation_ids 非空列表，每个元素非空

---

### 6. GroundTruthBundle（顶层容器）

```python
@dataclass
class GroundTruthBundle:
    """四个 ground_truth 子结构的容器。"""
    facts: list[GroundTruthFact]
    fact_list: GroundTruthFactList
    structure: GroundTruthStructure
    conclusion: GroundTruthConclusion
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "GroundTruthBundle": ...
```

---

### 7. KnownIssue（known_issues.json）

```python
@dataclass
class KnownIssue:
    """历史报告中的已知错误登记（v2 R-7 修正）。
    
    05_EVALUATION.md §3.2 同时提到数值错误和结论错误。
    数值错误 → fact_id 自然映射；结论错误 → 无 fact_id。
    
    因此 fact_id 改为 Optional[str]，增加 issue_type 字段区分两类。
    """
    issue_type: str           # 必填，"fact_value_error" | "conclusion_error"
    fact_id: Optional[str]    # 可选，fact_value_error 时必填；conclusion_error 时可空
    original_value: Any       # 必填，历史报告中声称的值
    correct_value: Any        # 必填，正确值
    reason: str               # 必填，错误原因
    confirmed_by: str         # 必填，确认人（评审方）
    confirmed_at: str         # 必填，ISO 8601
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "KnownIssue": ...
```

**issue_type 的两个取值**（R-7）：

| issue_type | 含义 | fact_id 是否必填 | original_value / correct_value |
| ---- | ---- | ---- | ---- |
| `fact_value_error` | 历史报告某条事实的值笔误 | 必填（三段式 grammar） | fact 的原始值 vs 正确值 |
| `conclusion_error` | 历史报告结论方向错误（如 qualified 应为 unqualified） | 可空 | 结论方向的原始值 vs 正确值（如 "qualified" vs "unqualified"） |

**Validator 检查**：
- issue_type ∈ {"fact_value_error", "conclusion_error"}
- issue_type="fact_value_error" 时：fact_id 必填且通过三段式 grammar
- issue_type="conclusion_error" 时：fact_id 可空
- confirmed_at 合法 ISO 8601
- confirmed_by 非空
- original_value != correct_value

---

### 8. ExpectedIssue（expected_issues.json）

```python
@dataclass
class ExpectedIssue:
    """正式验收对象 —— 系统应该发现但没发现的问题。
    
    Step 7-D 只做结构检查。
    rule_id 的业务注册表留给 Step 7-E Accuracy Validator。
    """
    rule_id: str                    # 必填
    severity: str                    # 必填，error | warning
    fact_id: Optional[str] = None   # 可选
    note: str = ""                  # 可选
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "ExpectedIssue": ...
```

**Validator 检查**：
- rule_id 非空
- severity ∈ {error, warning}
- fact_id（如果有）通过三段式 grammar

---

### 9. EvaluationCase（顶层外壳）

```python
@dataclass
class EvaluationCase:
    """评测用例的完整表示。v2 R-4 修正 case_dir。"""
    # ── 持久化字段 ──
    meta: Meta
    ground_truth: GroundTruthBundle
    known_issues: list[KnownIssue]
    expected_issues: list[ExpectedIssue]
    
    # ── Runtime-only 字段（v2 R-4 修正）────
    case_dir: str = field(default="", repr=False)
    #     case_dir 是 Loader 运行时上下文，不属于 Evaluation Case 的语义数据。
    #     不参与 to_dict / to_json / from_dict。
    #     from_dir() 使用它加载数据，但 JSON 表示不包含机器相关路径。
    
    # ── 序列化（v2 R-4：显式排除 case_dir）──
    def to_dict(self) -> dict:
        return {
            "meta": self.meta.to_dict(),
            "ground_truth": {
                "facts": [f.to_dict() for f in self.ground_truth.facts],
                "fact_list": self.ground_truth.fact_list.to_dict(),
                "structure": self.ground_truth.structure.to_dict(),
                "conclusion": self.ground_truth.conclusion.to_dict(),
            },
            "known_issues": [ki.to_dict() for ki in self.known_issues],
            "expected_issues": [ei.to_dict() for ei in self.expected_issues],
        }
    
    def to_json(self) -> str: ...
    
    # ── Loader（v2 R-4：case_dir 只在 from_dir 中使用）──
    @classmethod
    def from_dir(cls, dir_path: str) -> "EvaluationCase": ...
    
    # ── 顶层 Validator ──
    def validate(self) -> list[str]: ...
```

---

## D. 建议的文件结构（v2，与 v1 相同）

```
src/
  eval/
    __init__.py
    enums.py                       # CASE_LEVELS / SEVERITIES / ISSUE_TYPES
    types.py                       # 9 个 dataclass（v2 R-5：统一为 9 个）
    validator.py                   # 8 个 validate_* 函数
    
tests/
  eval/
    __init__.py
    fixtures/
      case_gold_minimal/
        meta.json
        inputs/                    # 只存在，内容不管（v2 R-2）
        ground_truth/
          facts.json               # 非空
          fact_list.json           # 非空 fact_ids
          structure.json           # 非空
          conclusion.json          # 非空 required_evaluation_ids + direction
        known_issues.json          # [] 或正常条目
        expected_issues.json       # 至少 1 条陷阱
      case_silver_minimal/
        ...                        # 四个 ground_truth 子文件全部非空
                                   # (silver 的"部分 G1" = facts 条目少 ≠ 空文件)
      case_bronze_minimal/
        ...
        ground_truth/
          facts.json               # **必须为空数组 []**（v2 R-3 / R-8）
          fact_list.json           # 非空
          structure.json           # 非空
          conclusion.json          # 非空
      case_missing_inputs_dir/     # inputs/ 目录缺，validator 应检出
        ...
      case_bronze_facts_nonempty/  # bronze 但 facts.json 非空，validator 应检出
        ...
    test_types.py
    test_validator.py
    test_loader.py
```

---

## E. Validator 应检查什么、不应该检查什么（v2 修订）

### 应该检查的

| 编号 | 检查项 | 用哪个 validator | 修订说明 |
| ---- | ---- | ---- | ---- |
| V-1 | Meta.case_level ∈ {gold, silver, bronze} | validate_meta | |
| V-2 | Meta.case_id / report_type / source_project 非空 | validate_meta | |
| V-3 | Meta.created_at 是合法 ISO 8601 字符串 | validate_meta | |
| V-4 | **inputs/ 目录存在** | validate_case | **v2 R-2 新增** |
| V-5 | **四个 ground_truth 子文件全部存在**（不论 level） | validate_case | **v2 R-8 修正**（不再"至少一个"） |
| V-6 | GroundTruthFact.fact_id 通过三段式 grammar | validate_ground_truth_facts | 复用 Step 7-B `is_valid_fact_id()` |
| V-7 | GroundTruthFact.quantity_kind ∈ QUANTITY_KINDS | validate_ground_truth_facts | 复用 Step 7-B registry |
| V-8 | GroundTruthFact.precision >= 0 | validate_ground_truth_facts | |
| V-9 | **bronze 的 facts 列表必须为空数组** | validate_case | **v2 R-3 / R-13 新增** |
| V-10 | fact_list.fact_ids 中每个通过三段式 grammar | validate_ground_truth_fact_list | |
| V-11 | KnownIssue.issue_type ∈ {"fact_value_error", "conclusion_error"} | validate_known_issues | **v2 R-7 新增** |
| V-12 | KnownIssue.fact_id：fact_value_error 时必填且 grammar 合法；conclusion_error 时可空 | validate_known_issues | **v2 R-7 修订** |
| V-13 | KnownIssue.confirmed_at 合法 ISO 8601 | validate_known_issues | |
| V-14 | KnownIssue.original_value != correct_value | validate_known_issues | |
| V-15 | ExpectedIssue.rule_id 非空 | validate_expected_issues | |
| V-16 | ExpectedIssue.severity ∈ {error, warning} | validate_expected_issues | |
| V-17 | ExpectedIssue.fact_id（如有）通过三段式 grammar | validate_expected_issues | |
| V-18 | GroundTruthConclusion.direction 非空 | validate_ground_truth_conclusion | **v2 R-6：只查非空，不做全局 enum** |
| V-19 | GroundTruthConclusion.required_evaluation_ids 非空，元素非空 | validate_ground_truth_conclusion | **v2 R-1 新增** |
| V-20 | **case_level 与内容的跨文件一致性** | validate_case | **v2 R-8 / C-13 统一**，详细规则见下表 |

### 四个 ground_truth 子文件的固定存在 + 内容要求（v2 R-8 统一）

| ground_truth 子文件 | gold | silver | bronze |
| ---- | ---- | ---- | ---- |
| facts.json | **必须非空**（至少 1 条 GroundTruthFact） | **必须非空**（至少 1 条——silver 的"部分 G1" = 条目数少 ≠ 空文件） | **必须为空数组 `[]`**（只有 G2+G4，无 G3 答案密钥） |
| fact_list.json | **必须非空**（fact_ids 列表至少 1 个） | **必须非空** | **必须非空**（G2 事实清单） |
| structure.json | **必须完整非空**（chapter_paths / table_count 等都有值） | **必须完整非空** | **必须完整非空** |
| conclusion.json | **必须非空**（direction + required_evaluation_ids） | **必须非空** | **必须非空**（G4 结论方向） |

所有级别都**必须同时存在四个子文件**（R-8）。区别只在内容空/非空。

### 不应该检查的（明确排除）

| 编号 | 检查项 | 留给谁 |
| ---- | ---- | ---- |
| X-1 | GroundTruthFact.value 的业务正确性（G3 是否真从 G1 独立推导） | 逆向链路 + 人工确认 |
| X-2 | rule_id 是否在某个正式 rule 注册表中 | Step 7-E Accuracy Validator |
| X-3 | fact_id 是否在某个 Fact Store 中存在 | Store 层 |
| X-4 | inputs/ 目录下 Excel/PDF/DOCX 的文件内容 | 不解析，只查目录存在 |
| X-5 | 任何比对逻辑（IssueList 匹配、G3 vs 运行输出） | Step 7-E 以后 |
| X-6 | direction 的业务取值域（qualified / unqualified / level_B 等） | ConclusionRule + report type 提供，validator 只查非空（v2 R-6） |
| X-7 | 任何 LLM / Store / Pipeline / Agent / Skill / Database | 后续步骤 |
| X-8 | case_level 的"静默提升" | 不做自动提升——case_level 是声明字段，validator 只检查声明与内容的**一致性**（bronze 带非空 facts → error，因为"仅有 G2+G4"与"有 G3"矛盾；gold 带空 facts → error，因为声明有 G3 但实际没有） |

---

## F. 测试矩阵（v2 修订）

### 第一组：数据结构构造 + JSON roundtrip（test_types.py）

| 编号 | 测试 | 覆盖 |
| ---- | ---- | ---- |
| T1 | Meta 正常构造 + roundtrip | Meta |
| T2 | GroundTruthFact 定量构造 + roundtrip | GroundTruthFact |
| T3 | GroundTruthFact 定性构造（unit=None） | GroundTruthFact |
| T4 | GroundTruthFactList 构造 + roundtrip | GroundTruthFactList |
| T5 | GroundTruthStructure 构造 + roundtrip | GroundTruthStructure |
| T6 | GroundTruthConclusion 构造（direction + required_evaluation_ids） + roundtrip | GroundTruthConclusion **v2 R-1 新增字段** |
| T7 | KnownIssue fact_value_error 构造 + roundtrip | KnownIssue **v2 R-7 issue_type=fact_value_error** |
| T8 | KnownIssue conclusion_error 构造（fact_id=None）+ roundtrip | KnownIssue **v2 R-7 issue_type=conclusion_error** |
| T9 | ExpectedIssue 带 fact_id + roundtrip | ExpectedIssue |
| T10 | ExpectedIssue 不带 fact_id + roundtrip | ExpectedIssue |
| T11 | GroundTruthBundle 组装 + roundtrip | GroundTruthBundle |
| T12 | EvaluationCase 完整组装 + roundtrip | EvaluationCase |
| T13 | **EvaluationCase.to_dict() 不含 case_dir** | **v2 R-4 新测试** |
| T14 | 所有 Optional 字段为 None 时 from_dict 默认值正确 | 各 dataclass |
| T15 | bronze 的 GroundTruthFactList facts 为空数组 + roundtrip | GroundTruthBundle（bronze fixture） |

### 第二组：Validator 基本 pass 路径

| 编号 | 测试 | 覆盖 |
| ---- | ---- | ---- |
| V1 | validate_meta：gold / silver / bronze 各一个合法 | validate_meta |
| V2 | validate_ground_truth_facts：列表全部合法 | validate_ground_truth_facts |
| V3 | validate_ground_truth_fact_list：列表全部合法 | validate_ground_truth_fact_list |
| V4 | validate_ground_truth_structure：全部合法 | validate_ground_truth_structure |
| V5 | validate_ground_truth_conclusion：direction + required_evaluation_ids 全合法 | validate_ground_truth_conclusion |
| V6 | validate_known_issues：fact_value_error + conclusion_error 各一个 | validate_known_issues **v2 R-7** |
| V7 | validate_expected_issues：空数组 + 正常数组 | validate_expected_issues |
| V8 | validate_case：**case_gold_minimal fixture 全量通过** | validate_case（顶层） |
| V9 | validate_case：**case_silver_minimal fixture 全量通过** | validate_case |
| V10 | validate_case：**case_bronze_minimal fixture 全量通过**（facts.json 空数组） | validate_case **v2 R-3** |

### 第三组：Validator 失败路径

| 编号 | 错误 | 期望 |
| ---- | ---- | ---- |
| F1 | Meta.case_level = "platinum" | validate_meta 报错 |
| F2 | Meta.case_id = "" | validate_meta 报错 |
| F3 | Meta.created_at = "not-a-date" | validate_meta 报错 |
| F4 | GroundTruthFact.fact_id grammar 非法 | validate_ground_truth_facts 报错 |
| F5 | GroundTruthFact.fact_id scope_class 非法 | validate_ground_truth_facts 报错 |
| F6 | GroundTruthFact.quantity_kind 非法 | validate_ground_truth_facts 报错 |
| F7 | GroundTruthFact.precision = -1 | validate_ground_truth_facts 报错 |
| F8 | fact_list.fact_ids 含空字符串 | validate_ground_truth_fact_list 报错 |
| F9 | KnownIssue.issue_type = "other" | validate_known_issues 报错 **v2 R-7** |
| F10 | KnownIssue fact_value_error 但 fact_id 为空 | validate_known_issues 报错 **v2 R-7** |
| F11 | KnownIssue conclusion_error 但 fact_id 有值（**不报错**——conclusion_error fact_id 可空） | 验证 validator 不做无意义报错 |
| F12 | KnownIssue original_value == correct_value | validate_known_issues 报错 |
| F13 | KnownIssue confirmed_at 非法 | validate_known_issues 报错 |
| F14 | KnownIssue confirmed_by = "" | validate_known_issues 报错 |
| F15 | ExpectedIssue.rule_id = "" | validate_expected_issues 报错 |
| F16 | ExpectedIssue.severity = "fatal" | validate_expected_issues 报错 |
| F17 | ExpectedIssue.fact_id grammar 非法 | validate_expected_issues 报错 |
| F18 | GroundTruthConclusion.direction = "" | validate_ground_truth_conclusion 报错 **v2 R-6** |
| F19 | GroundTruthConclusion.required_evaluation_ids = [] | validate_ground_truth_conclusion 报错 **v2 R-1** |
| F20 | GroundTruthConclusion.required_evaluation_ids 元素为空 | validate_ground_truth_conclusion 报错 |

### 第四组：顶层 Validator 跨文件一致性（validate_case）

| 编号 | 场景 | 期望 |
| ---- | ---- | ---- |
| C1 | gold 用例**缺 inputs/ 目录** | validate_case 报错 **v2 R-2** |
| C2 | gold 用例缺 ground_truth/facts.json | validate_case 报错 **v2 R-8（四文件固定存在）** |
| C3 | gold 用例缺 ground_truth/fact_list.json | validate_case 报错 |
| C4 | gold 用例缺 ground_truth/structure.json | validate_case 报错 |
| C5 | gold 用例缺 ground_truth/conclusion.json | validate_case 报错 |
| C6 | gold 用例 facts.json 为空数组 | validate_case 报错 **v2 R-8（gold facts 必须非空）** |
| C7 | bronze 用例**带非空 facts.json** | validate_case 报错 **v2 R-3 / R-13 关键路径** |
| C8 | bronze 用例缺 inputs/ 目录 | validate_case 报错 **v2 R-2（bronze 也要求 inputs/存在）** |
| C9 | silver 用例 facts.json 为空数组 | validate_case 报错 **v2 R-8（silver facts 也必须非空）** |
| C10 | 任意级别缺 meta.json | validate_case 报错 |
| C11 | EvaluationCase.from_dir 加载 gold fixture | 所有子文件正确解析，case_dir 不进入 JSON |
| C12 | EvaluationCase.from_dir 加载 bronze fixture（facts.json 为 `[]`） | 正确解析为空列表 |
| C13 | **bronze_facts_nonempty fixture → validate_case 报错** | **v2 R-3 / R-13 的专门 fixture** |

---

## G. 与 Step 7-B / Step 7-C 的依赖关系（v2 不变）

| 依赖 | 如何复用 | 为什么 |
| ---- | ---- | ---- |
| Step 7-B `is_valid_fact_id()` | 直接 import | 三段式 grammar 是冻结的契约 |
| Step 7-B `parse_fact_id()` | 直接 import | 跨文件一致性检查需要解析 fact_id |
| Step 7-B `QUANTITY_KINDS` 枚举 | 直接 import | ground truth quantity_kind 必须与运行时统一 |
| Step 7-B `SCOPE_CLASSES` | 间接（通过 is_valid_fact_id） | 不在本层直接使用 |
| Step 7-B `Fact` dataclass | **不复用** | GroundTruthFact 语义不同（答案密钥 vs 运行时事实） |
| Step 7-B `FactTypeRegistry` | 可选做 optional warning | 答案密钥可以比运行时更宽容 |
| Step 7-C Domain Objects | **无依赖** | Evaluation Case 不引用 Domain Object id |

**Step 7-B / 7-C 不得修改的内容**：不变——所有已有代码、测试、tag（step-7b-complete、step-7c-complete）。

---

## H. 明确留给 Step 7-E 以后的内容（v2 不变）

| 编号 | 内容 | 留到哪一步 |
| ---- | ---- | ---- |
| L-1 | IssueList 与 expected_issues 命中率比对 | Step 7-E Accuracy Validator |
| L-2 | 数值比对逻辑（G3 vs 运行输出，按 precision round） | Step 7-E |
| L-3 | 判定逻辑一致性比对 | Step 7-E |
| L-4 | 数字锚定比对 | Step 7-E |
| L-5 | 结论覆盖度比对（required_evaluation_ids vs covers[]） | Step 7-E |
| L-6 | 结构一致性比对（P1） | Step 7-F |
| L-7 | 文本一致性比对（P2） | Step 7-F |
| L-8 | 版式一致性比对（P3） | Step 7-F |
| L-9 | Run Snapshot / Baseline / 回归报告 | Step 7-G |
| L-10 | 历史报告逆向抽取 | Step 7-I |
| L-11 | LLM / Adapter / Skill / Template / Renderer / DOCX | Step 7-H / 7-J |
| L-12 | 任何数据库 / Agent / UI | 永远不在 Step 7-D 范围 |

---

## I. 可能存在的设计歧义（v2 补充）

### 歧义 1（v1 已识别，v2 保留）：GroundTruthFact.precision

选择 A（GroundTruthFact 自带 precision），理由不变——Step 7-D 自包含，不与尚未实现的 TemplateSpec 产生编译期依赖。

### 歧义 2（v2 新增）：bronze 用例为什么还要求 inputs/ 目录存在

05_EVALUATION.md §4 定义 bronze = "仅有报告（G2 + G4）"，意思是**系统没有 G1 输入可供评测**，不是说**评测用例目录里连 inputs/ 文件夹都不应该有**。实际历史项目目录里总会有一个 inputs/ 占位。validator 检查 inputs/ 目录存在，不解析其内容——与 bronze 的语义不冲突。

### 歧义 3（v2 新增）：GroundTruthConclusion.required_evaluation_ids 的 grammar

Step 7-D 只检查元素非空。但 `evaluation:component.K001.status` 这类 Evaluation id 有自己的 grammar（前缀 `evaluation:` + 三段式结构？）。本阶段 Evaluation 还没有正式 grammar 定义（那是 Step 7-C 的 domain.py 后续要加的），所以 validator 只查非空。这是**合理的前向兼容**——当 Evaluation grammar 在后续步骤冻结后，可以升级 validator。

### 歧义 4（v2 新增）：KnownIssue 的 conclusion_error 条目里 original_value / correct_value 存什么

05_EVALUATION.md §3.2 的 known_issues 示例只展示了 fact_value_error 场景。conclusion_error 场景下 original_value / correct_value 存的是结论方向枚举值（如 `"qualified"` vs `"unqualified"`），不是数值。validator 只做 `!=` 检查，不做类型检查（值可以是 str、int、任何可比对象）。

### 歧义 5（v2 新增）：silver 的 facts.json "部分"是什么意思

v2 把 silver 与 gold 放在一起要求 facts.json 非空。silver 的"部分 G1" = facts.json **条目数少**（比如 gold 有 15 条 GroundTruthFact，silver 只有 3 条可从该部分 G1 独立推导的），不是文件空。这样 silver 与 gold 的**validator 规则完全相同**（都要求 facts.json 非空），区别是内容量，不是结构规则。

---

## J. 最终推荐的 Step 7-D 最小实施范围（v2）

### 必须做的

1. **`src/eval/enums.py`**：CASE_LEVELS（gold/silver/bronze）、SEVERITIES（error/warning）、ISSUE_TYPES（fact_value_error/conclusion_error）三个受控枚举
2. **`src/eval/types.py`**：**9 个 dataclass**（v2 R-5 统一）——Meta、GroundTruthFact、GroundTruthFactList、GroundTruthStructure、GroundTruthConclusion、GroundTruthBundle、KnownIssue、ExpectedIssue、EvaluationCase
3. **`src/eval/validator.py`**：7 个子 validator + 1 个顶层 validate_case（含 v2 R-2 inputs/目录检查、R-3 bronze facts 空数组检查、R-8 四个子文件存在性检查）
4. **`src/eval/__init__.py`**：export 所有公开类和函数
5. **`tests/eval/test_types.py`**：所有 dataclass 构造 + JSON roundtrip（包含 v2 R-4 的 case_dir 不序列化测试、R-7 的 conclusion_error 构造测试）
6. **`tests/eval/test_validator.py`**：基本 pass（~10）、失败路径（~20）、跨文件一致性（~13），合计 **~45+ 测试**
7. **`tests/eval/fixtures/`**：**5 个 fixture 目录**——case_gold_minimal、case_silver_minimal、case_bronze_minimal（facts.json=`[]`）、case_missing_inputs_dir、case_bronze_facts_nonempty
8. **`tests/eval/test_loader.py`**：from_dir 加载 fixture 的集成测试

### 可选但推荐的

1. 在每个 dataclass 头部注释标注对应的 05_EVALUATION.md 章节
2. 在复用 Step 7-B 的 import 处加注释标注具体复用函数
3. 从 fixture 生成一版 fixture.json 供人工 review（但**不**做 fixture 到 gold 的自动提升）

### 明确不做的

- 不复用 Step 7-B 的 Fact dataclass
- 不调用 LLM / Adapter / Skill
- 不解析 inputs/ 目录下任何文件内容
- 不建立 Input Manifest 或新的 RawSource / Input 数据模型
- 不实现任何 Store / Pipeline / Agent / Skill
- 不修改 Step 7-B / 7-C 的任何代码文件
- 不修改 05_EVALUATION.md 或其他设计文档
- 不创建 git commit

### 验收标准

1. `pytest tests/eval/` 全部通过（新增测试 + 不影响 Step 7-B/7-C 已有的 128 个测试）
2. `python -c "from eval import *"` 无 ImportError
3. 5 个 fixture 目录的 validator 判定全部符合预期
4. 无任何设计文档或 Step 7-B/7-C 代码被修改
5. 打 tag `step-7d-complete`

---

以上是完整的 v2 修订方案。等你确认后再进入执行。
