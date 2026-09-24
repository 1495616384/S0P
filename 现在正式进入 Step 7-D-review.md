好，纯设计任务。我先把所有相关文档、代码和Git状态读完，再出实施方案。
这是一个纯设计任务，用户明确要求不写代码、不修改文件，只输出实施方案。我需要先读取所有相关文档和代码，理解当前架构状态，然后输出Step 7-D的实施方案。

让我先读取所有需要的文档和代码文件，以及检查Git状态。

好，五份文档已读完。现在去读 Step 7-B / 7-C 的代码和 Git 状态。
好，五个设计文档都读完了。现在去读 Step 7-B 和 Step 7-C 的已完成代码，以及 Git 状态。

好，现在我需要读所有已完成的代码文件，以及检查 Git 状态和 tags。

现在看 Git 状态、tag、以及实际目录结构。
好，代码都读完了。现在检查 Git 状态和 tags。好，RunCommand 不可用，用 Shell 替代。

好，信息都收集完了。现在调用 AdvisorTool 做战略指导，然后输出 Step 7-D 的实施方案。

让我整理一下：

**当前仓库状态**：
- 3 个 commit，tags：design-baseline-m0, step-7b-complete, step-7c-complete
- Step 7-B 已完成：src/cdm/{types, id, registry, validate}（L1/L2）
- Step 7-C 已完成：src/cdm/{domain, domain_validate}（L3 Domain Objects）
- 工作区干净，只有未追踪的 reviews/ 和开发流水线.md

**已确认的文档基线**：
- 02_ARCHITECTURE.md v0.3
- 03_CANONICAL_DATA_MODEL.md v0.3.1
- 04_REPORT_IR.md v0.3.1.1
- 05_EVALUATION.md v0.3
- 07_DECISIONS.md v0.1

**05_EVALUATION.md §5 Evaluation Case 目录结构**：
```
cases/<case_id>/
├── meta.json
├── inputs/
├── ground_truth/
│   ├── facts.json
│   ├── fact_list.json
│   ├── structure.json
│   └── conclusion.json
├── known_issues.json
└── expected_issues.json
```

**核心约束分析**：

1. G1/G2/G3/G4 分离是铁律（D-012）
2. G3 必须来自 G1 独立推导，不能抄历史报告正文
3. Case Level（gold/silver/bronze）声明后不可提升
4. expected_issues 是正式验收对象
5. known_issues 登记历史报告已知错误
6. 本阶段只做数据结构 + 序列化 + 基础校验 + 单元测试

**方案 A vs B 分析**：

方案 A：大 EvaluationCase 对象，内部嵌套所有 G1/G2/G3/G4
- 优点：调用时一个对象搞定
- 缺点：
  - 边界耦合：G3 facts.json 其实就是 Fact 对象的 JSON，方案 A 会在内部复制一份 Fact 结构，和 Step 7-B 的 Fact 耦合到底
  - 不可测试性：想单独测 expected_issues 的 validator 必须构造一个完整 EvaluationCase
  - 扩展困难：后续加新的 ground_truth 类型要改大对象
  - 与 Facts/Domain Objects 耦合：G3 facts 里的 fact_id 必须符合 Step 7-B 的三段式，但方案 A 把这个耦合藏在内部

方案 B：EvaluationCase 作为外壳，各子结构独立
- 优点：
  - 边界稳定：每个子结构（Meta、GroundTruth、ExpectedIssues、KnownIssues）独立定义、独立验证
  - 可测试性：每个 validator 单独测
  - 避免与 Step 7-B/Fact 的耦合：GroundTruth.facts 是 List[Fact]，直接引用 Step 7-B 的 Fact dataclass，不重新定义
  - 后续扩展方便：加新的 ground_truth 类型只加子结构
- 缺点：多了一层组装逻辑
- 结论：方案 B 胜出

**与 Step 7-B/7-C 的边界**：
- GroundTruth.facts 直接使用 Step 7-B 的 Fact dataclass（复用 from_dict/to_dict）
- fact_list.json 和 fact_id 引用 Step 7-B 的三段式 ID grammar（复用 is_valid_fact_id）
- 本阶段不应该重新定义 Fact 结构

**目录结构设计**：
```
src/eval/
  __init__.py
  case.py          # EvaluationCase + Meta + KnownIssue + ExpectedIssue + GroundTruth
  validator.py     # validate_case + 各子 validator
  enums.py         # CASE_LEVELS, SEVERITIES, etc.
tests/eval/
  __init__.py
  test_case.py     # 结构 + 序列化
  test_validator.py # 校验
  fixtures/
    case_gold/     # gold 用例 fixture（最小样本）
    case_silver/   # silver 用例 fixture
    case_bronze/   # bronze 用例 fixture
```

**每个子结构的详细字段**：

Meta:
- case_id: str (必填)
- case_level: str ∈ {gold, silver, bronze} (必填)
- report_type: str (必填)
- traps: list[str] (可选，陷阱标注)
- source_project: str (必填)
- created_at: str (必填，ISO 8601)
- design_version: str (可选，指向哪版设计文档)

KnownIssue:
- fact_id: str (必填，三段式)
- original_value: Any (必填)
- correct_value: Any (必填)
- reason: str (必填)
- confirmed_by: str (必填)
- confirmed_at: str (必填，ISO 8601)

ExpectedIssue:
- rule_id: str (必填，如 "FACT.MISSING")
- fact_id: Optional[str] (可选，如果适用)
- severity: str ∈ {error, warning} (必填)
- note: str (可选)

GroundTruth:
- facts: list[Fact] (G3，从 Step 7-B 直接引用)
- fact_list: list[str] (G2，三段式 fact_id 列表)
- structure: dict (G4，章节路径集合、表格数等)
- conclusion: dict (G4，结论方向枚举)

**Validator 检查什么**：

必须检查：
1. meta.json 内部一致性：
   - case_level ∈ {gold, silver, bronze}
   - gold → ground_truth.facts 非空
   - silver → ground_truth.facts 部分非空
   - bronze → ground_truth.facts 应为空（或只有 fact_list）
2. fact_list.json 中的每个 fact_id 必须符合三段式 grammar（复用 Step 7-B）
3. GroundTruth.facts 中每个 Fact 必须通过 Step 7-B 的 validate_fact
4. known_issues 中 fact_id 必须通过三段式 grammar
5. known_issues 中 confirmed_at 必须是合法 ISO 8601
6. expected_issues 中 severity ∈ {error, warning}
7. expected_issues 中 fact_id（如果有）必须符合三段式 grammar
8. 跨文件一致性：fact_list 中的 fact_id 应与 GroundTruth.facts 中的 id 有重叠（gold 用例）

不应该检查：
1. Fact.value 的业务正确性（G3 推导正确性是逆向链路的事，不是本层）
2. rule_id 是否在某个 "正式 rule 注册表" 中（本阶段 rule_id 只是字符串，比对逻辑在 7-E）
3. expected_issues 与 known_issues 的重叠（允许同一个 fact 既是 expected_issue 又是 known_issue，角色不同）
4. 运行时比对逻辑（Step 7-E 以后）
5. inputs/ 目录的内容（inputs 是原始文件集合，本阶段只做路径登记）

让我把这些整理成完整的实施方案。
正在请求专家建议......

已获取专家建议

Advisor 给了一个关键提示：**不要复用 Step 7-B 的 Fact dataclass**，而是定义一个轻量的 GroundTruthFact。这个点很重要，让我重新想一下。

理由是：G3 ground truth 是「答案密钥」，不是「运行时 Fact」。运行时 Fact 有 source_refs、method、provenance、review_status、status、revision、supersedes——这些在 ground truth 里都没有。Ground truth 只关心：fact_id + value + unit + quantity_kind。

如果直接用 Fact 类，ground truth 的 Fact 对象会携带一堆空字段（source_refs=[]，status=filled，review_status=confirmed...），这既不 clean 也有误导性。

但是！fact_id 的 grammar 校验可以复用 Step 7-B 的 is_valid_fact_id。这是正确的复用方式——复用校验函数，不复用数据类。

好，现在我有了完整的实施方案。让我输出。
好，现在调用 Advisor 做方案评审。

*正在请求专家建议，优化解决方案，该过程耗时可能较长，请耐心等待...*

好，所有信息收集完毕。现在输出完整的 Step 7-D 实施方案。

---

# Step 7-D 实施方案：Evaluation Case 基础层

## A. 对 Step 7-D 的准确理解

**意图**：为评测体系建立一个**程序化、可版本化、可独立验证的评测用例数据结构层**。

这个层的定位是：
- 它**不是**运行时数据流的一部分（不像 Facts 被 Pipeline 消费）
- 它是**评测固件**——被 M7 eval/loader 加载、被 Regression Runner 读取的只读 JSON 数据载体
- 它的语义完全由 `05_EVALUATION.md` 定义，核心是 G1/G2/G3/G4 分离和 case level 分级
- 它必须在**没有任何 LLM、没有任何比对逻辑、没有任何执行器**的前提下就能独立校验通过

**一句话定位**：Step 7-D = `05_EVALUATION.md §5` 的程序化实现，且只实现其**数据结构 + Schema 校验**子集。

**明确不做的**（本阶段禁止）：
- 历史报告逆向抽取
- LLM 相关
- Accuracy Validator / Style Validator
- Regression Runner / Baseline / Run Snapshot
- 指标计算 / 报告评分
- 任何比对逻辑（IssueList 匹配、ground truth 与运行输出比对）
- DOCX/PDF 处理
- 数据库 / Agent / UI

---

## B. 当前架构中已确定的约束

从 5 份设计文档和 Step 7-B/7-C 已完成代码中抽取的硬约束：

| 编号 | 约束 | 来源 |
| ---- | ---- | ---- |
| C-1 | G1/G2/G3/G4 四类信息必须**分离存储**，目录结构已由 05 §5 冻结 | 05_EVALUATION.md §5 / D-012 |
| C-2 | G3 facts 的值必须来自 G1 **独立推导**，禁止抄历史报告正文 | 05_EVALUATION.md §2 / D-012 |
| C-3 | Case level 分 **gold / silver / bronze**，且**声明后不可提升** | 05_EVALUATION.md §4 |
| C-4 | **gold** 要求 G1 + G3 + G4 齐全；**silver** 要求部分 G1 + 对应 G3；**bronze** 仅有 G2 + G4 | 05_EVALUATION.md §4 |
| C-5 | `expected_issues` 是**正式验收对象**，不是备注字段 | 05_EVALUATION.md §5.2 / §7 |
| C-6 | `known_issues` 必须登记历史报告的已知错误，保留 fact_id / original_value / correct_value / reason / confirmed_by / confirmed_at | 05_EVALUATION.md §3.2 |
| C-7 | fact_id 必须符合 Step 7-B 冻结的**三段式 grammar**：`fact:<scope_class>.<instance_key>.<attribute>` | 03_CDM v0.3.1 / D-027 |
| C-8 | Evaluation Case 不应该**重新定义 Fact**；如果引用 fact_id 或 domain object id，应**复用已有契约** | 项目规则 §7 |
| C-9 | Step 7-B / 7-C 已完成的代码**不得修改** | 用户明确要求 |
| C-10 | 本阶段只做：数据结构 + 序列化/反序列化 + 基础 schema validator + 单元测试 | 用户明确要求 |
| C-11 | 中间产物全部 JSON 落盘 | 02_ARCHITECTURE.md D-12 |
| C-12 | 第一阶段 Python 实现，dataclasses + typing + 标准 json + pytest | project_memory |

---

## C. 建议的数据结构

### 方案选择：方案 B（EvaluationCase 外壳 + 独立子结构）

**方案 A**（一个大 EvaluationCase 对象嵌套所有 G1/G2/G3/G4）vs **方案 B**（外壳 + 独立子结构）：

| 维度 | 方案 A | 方案 B | 胜出 |
| ---- | ---- | ---- | ---- |
| **可测试性** | 要测 expected_issues 验证，必须构造完整 EvaluationCase，包含 G1/G2/G3/G4 全部 | 每个子结构有独立 validator，可单独实例化、单独测 | B |
| **边界稳定性** | G3 facts 会直接/间接与 Step 7-B 的 Fact 类耦合；任何一方改字段，另一方被迫跟随 | 各子结构边界清晰：Meta 只管元信息，GroundTruth 只管答案密钥，ExpectedIssues/KnownIssues 只管异常登记。互不牵连 | B |
| **后续扩展** | 加新的 ground_truth 类型（比如 G3 的 computed_facts）需要改大对象 | 加一个新子结构或在现有子结构里加字段即可 | B |
| **与 Facts 耦合** | 很容易把 Step 7-B 的 Fact 类直接嵌套进来，复制一份运行时 Fact 作为 ground truth——语义错误 | GroundTruthFact 定义为**轻量答案密钥**（只有 fact_id + value + unit + quantity_kind），不复用运行时 Fact 类，但复用其 id grammar validator | B |

**核心设计判断**：GroundTruth.facts.json 不是「运行时 Fact 集合」，它是「评测答案密钥」。运行时 Fact 有 source_refs、method、provenance、review_status、status、revision、supersedes——这些在答案密钥里全部不存在。如果直接 import Fact 类来表示 ground truth，会产生大量空字段、语义混淆、以及与 Step 7-B 的隐式耦合。正确做法是：

- **复用 Step 7-B 的 `is_valid_fact_id()` 做 grammar 校验**（这是契约复用）
- **不复用 Step 7-B 的 `Fact` dataclass**（这是语义解耦）

### 数据结构详细定义

#### 1. Meta（meta.json）

```python
@dataclass
class Meta:
    case_id: str               # 必填，如 "case-001"
    case_level: str            # 必填，gold | silver | bronze
    report_type: str            # 必填，报告类型标识
    source_project: str         # 必填，来源项目标识
    created_at: str             # 必填，ISO 8601
    design_version: str         # 可选，指向哪版设计文档，如 "05_EVALUATION_v0.3"
    traps: list[str]            # 可选，陷阱标注，如 ["missing_data", "unit_anomaly"]
    notes: str                  # 可选，自由文本备注
```

**Validator 检查**：
- case_level ∈ {gold, silver, bronze}
- case_id 非空、非纯空白
- created_at 可解析为合法 ISO 8601 日期字符串
- 不允许 case_level 被修改（immutable 语义——但本阶段只做数据结构，immutability 在 Store 层保证）

#### 2. GroundTruth（ground_truth/ 下的四个 JSON）

四个文件对应四个独立的子结构：

```python
# ground_truth/facts.json — G3：事实值答案密钥
@dataclass
class GroundTruthFact:
    """轻量答案密钥 — 不是运行时 Fact。
    
    运行时 Fact 有 source_refs / method / provenance / review_status / 
    status / revision / supersedes / value_domain —— ground truth 全都没有。
    
    只保留评测比对所需的最小字段。
    """
    fact_id: str                    # 三段式 grammar，复用 Step 7-B 校验
    value: Any                      # 与运行时 Fact.value 类型相同
    unit: Optional[str]             # 与运行时 Fact.unit 语义相同
    quantity_kind: str              # 必须匹配 Step 7-B 已注册枚举
    precision: int                  # 显示精度，用于 P0 比对的 round 刻度

# ground_truth/fact_list.json — G2：事实清单（只有 fact_id 列表）
@dataclass
class GroundTruthFactList:
    fact_ids: list[str]             # 每个都必须通过三段式 grammar

# ground_truth/structure.json — G4：章节路径 + 表格 + 图片
@dataclass  
class GroundTruthStructure:
    chapter_paths: list[str]        # 章节路径集合
    table_count: int                # 表格数量
    table_headers: list[list[str]]  # 每个表格的列头
    figure_count: int               # 图片数量
    required_elements: list[str]    # 必需要素（如 "signature_block"）

# ground_truth/conclusion.json — G4：结论方向枚举
@dataclass
class GroundTruthConclusion:
    direction: str                  # 枚举值，如 "qualified" / "unqualified" / "level_B"
    coverage_required: list[str]    # 必须被 covers[] 引用的 Evaluation id 列表
    notes: Optional[str] = None
```

**Validator 检查**：
- 每个 GroundTruthFact.fact_id 通过 `is_valid_fact_id()`（复用 Step 7-B）
- 每个 GroundTruthFact.quantity_kind ∈ QUANTITY_KINDS（复用 Step 7-B registry）
- precision >= 0
- GroundTruthFactList 中每个 fact_id 通过 grammar 校验
- chapter_paths / table_count / figure_count 等非负
- GroundTruthConclusion.direction 非空

#### 3. KnownIssue（known_issues.json）

```python
@dataclass
class KnownIssue:
    """历史报告中的已知错误登记。"""
    fact_id: str                    # 必填，三段式 grammar
    original_value: Any             # 必填，历史报告中声称的值
    correct_value: Any              # 必填，正确值
    reason: str                     # 必填，错误原因
    confirmed_by: str               # 必填，确认人（评审方）
    confirmed_at: str               # 必填，ISO 8601
```

**Validator 检查**：
- fact_id 通过三段式 grammar
- confirmed_at 是合法 ISO 8601
- confirmed_by 非空
- original_value != correct_value（否则不是错误）

#### 4. ExpectedIssue（expected_issues.json）

```python
@dataclass
class ExpectedIssue:
    """正式验收对象 —— 系统应该发现但没发现的问题。"""
    rule_id: str                    # 必填，如 "FACT.MISSING" / "ANCHOR.FAIL"
    severity: str                    # 必填，error | warning
    fact_id: Optional[str] = None   # 可选，如适用
    note: str = ""                  # 可选，说明
```

**Validator 检查**：
- rule_id 非空
- severity ∈ {error, warning}
- fact_id（如果有）通过三段式 grammar

#### 5. EvaluationCase（顶层外壳）

```python
@dataclass
class EvaluationCase:
    case_dir: str                   # 路径（可选，runtime context）
    meta: Meta
    ground_truth: GroundTruthBundle
    known_issues: list[KnownIssue]
    expected_issues: list[ExpectedIssue]
    
    def to_dict(self) -> dict: ...
    def to_json(self) -> str: ...
    
    @classmethod
    def from_dir(cls, dir_path: str) -> "EvaluationCase": ...
    
    def validate(self) -> list[str]: ...
```

其中 GroundTruthBundle 是四个子结构的容器：

```python
@dataclass
class GroundTruthBundle:
    facts: list[GroundTruthFact]
    fact_list: GroundTruthFactList
    structure: GroundTruthStructure
    conclusion: GroundTruthConclusion
```

**顶层 Validator 还要检查的跨文件一致性**（这是单独子 validator 无法覆盖的）：

1. **gold 用例**：ground_truth.facts 非空 + fact_list.fact_ids 非空 + structure 非空 + conclusion 非空
2. **silver 用例**：ground_truth.facts 部分非空 + fact_list.fact_ids 部分非空（不要求全部）
3. **bronze 用例**：ground_truth.facts 应为空（或仅有 fact_list，无 facts）+ fact_list + structure + conclusion 应有内容
4. gold / silver 的 ground_truth.facts 中的 fact_id 应与 fact_list.fact_ids 有交集（或全部在 fact_list 中声明）
5. known_issues 中的 fact_id 若同时出现在 ground_truth.facts，应能找到对应条目（但这是建议性 warning，不是 error——known_issues 可以登记已被更正的历史错误，而 ground truth 是正确值）
6. case_level 与目录内容的一致性是**结构检查**，不做业务正确性检查（比如 bronze 用例声明了 gold 的内容——结构上允许，但 level 仍然是 bronze，不可静默提升）

---

## D. 建议的文件结构

```
src/
  eval/                            # Step 7-D 新建目录
    __init__.py                    # 包入口，export 所有公开类 + validator
    enums.py                       # CASE_LEVELS / SEVERITIES 等受控枚举
    types.py                       # 所有 dataclass：Meta / GroundTruthFact / 
                                   #   GroundTruthFactList / GroundTruthStructure / 
                                   #   GroundTruthConclusion / GroundTruthBundle /
                                   #   KnownIssue / ExpectedIssue / EvaluationCase
    validator.py                   # 所有 validate_* 函数
                                   #   - validate_meta
                                   #   - validate_ground_truth_fact(s)
                                   #   - validate_ground_truth_fact_list
                                   #   - validate_ground_truth_structure
                                   #   - validate_ground_truth_conclusion
                                   #   - validate_known_issue(s)
                                   #   - validate_expected_issue(s)
                                   #   - validate_case（顶层，跨文件一致性）

tests/
  eval/                            # Step 7-D 新建目录
    __init__.py
    fixtures/                      # 最小 fixture JSON（不依赖真实历史数据）
      case_gold_minimal/
        meta.json
        ground_truth/
          facts.json
          fact_list.json
          structure.json
          conclusion.json
        known_issues.json          # 空数组 []
        expected_issues.json       # 包含 1-2 条陷阱
      case_silver_minimal/
        ...
      case_bronze_minimal/
        ...
      case_gold_missing_file/      # 故意缺文件，测试 validator 能否检出
        ...
      case_gold_level_mismatch/    # 声明 gold 但内容只有 bronze 级
        ...
    test_types.py                  # 所有 dataclass 的构造 + JSON roundtrip
    test_validator.py              # 所有 validator 的 pass/fail 覆盖
    test_loader.py                 # from_dir / 从 fixture 目录加载
```

---

## E. Validator 应检查什么、不应该检查什么

### 应该检查的（结构性 / schema 级）

| 编号 | 检查项 | 用哪个 validator |
| ---- | ---- | ---- |
| V-1 | Meta.case_level ∈ {gold, silver, bronze} | validate_meta |
| V-2 | Meta.case_id / report_type / source_project 非空 | validate_meta |
| V-3 | Meta.created_at 是合法 ISO 8601 字符串 | validate_meta |
| V-4 | 目录必需文件存在（meta.json / ground_truth/ / 至少一个子文件） | validate_case |
| V-5 | GroundTruthFact.fact_id 通过三段式 grammar | validate_ground_truth_facts（复用 Step 7-B `is_valid_fact_id()`） |
| V-6 | GroundTruthFact.quantity_kind ∈ QUANTITY_KINDS | validate_ground_truth_facts（复用 Step 7-B registry） |
| V-7 | GroundTruthFact.precision >= 0 | validate_ground_truth_facts |
| V-8 | fact_list.fact_ids 中每个通过三段式 grammar | validate_ground_truth_fact_list |
| V-9 | KnownIssue.fact_id 通过三段式 grammar | validate_known_issues |
| V-10 | KnownIssue.confirmed_at 合法 ISO 8601 | validate_known_issues |
| V-11 | KnownIssue.original_value != correct_value | validate_known_issues |
| V-12 | ExpectedIssue.rule_id 非空 | validate_expected_issues |
| V-13 | ExpectedIssue.severity ∈ {error, warning} | validate_expected_issues |
| V-14 | ExpectedIssue.fact_id（如有）通过三段式 grammar | validate_expected_issues |
| V-15 | GroundTruthConclusion.direction 非空 | validate_ground_truth_conclusion |
| V-16 | Case Level 与目录内容的一致性（gold 缺 G3 → error；silver 缺 G3 → error；bronze 有 G3 → 允许但加 warning） | validate_case（顶层） |
| V-17 | gold / silver 的 ground_truth.facts 与 fact_list.fact_ids 有交集 | validate_case（顶层，建议性） |

### 不应该检查的（留给后续步骤）

| 编号 | 检查项 | 留给谁 |
| ---- | ---- | ---- |
| X-1 | GroundTruthFact.value 的**业务正确性**（G3 是否真的从 G1 独立推导） | 逆向链路 + 人工确认（不是程序能判断的） |
| X-2 | rule_id 是否在某个"正式 rule 注册表"中 | Step 7-E（Accuracy Validator 会有自己的 rule 注册表） |
| X-3 | expected_issues 与 known_issues 是否重叠（允许重叠，角色不同） | 不检查 |
| X-4 | fact_id 是否在某个 Fact Store 中存在 | Store 层（本层只有轻量密钥，没有 Store） |
| X-5 | inputs/ 目录的文件内容、文件格式 | 本阶段只登记路径，不解析原始文件 |
| X-6 | 运行时 IssueList 与 expected_issues 的比对 | Step 7-E / Step 7-F |
| X-7 | 数值比对逻辑（G3 value vs 运行输出 value 的比较） | Step 7-E Accuracy Validator |
| X-8 | 结论方向枚举值的业务含义 | 业务层，validator 只检查非空 |
| X-9 | 任何 LLM 相关的东西 | Step 7-H 以后 |

---

## F. 测试矩阵

### 第一组：数据结构构造 + JSON roundtrip（test_types.py）

| 编号 | 测试 | 覆盖 |
| ---- | ---- | ---- |
| T1 | Meta 正常构造 + to_dict / from_dict / to_json roundtrip | Meta |
| T2 | GroundTruthFact 正常构造（定量） | GroundTruthFact |
| T3 | GroundTruthFact 正常构造（定性，unit=None） | GroundTruthFact |
| T4 | GroundTruthFactList 构造 + roundtrip | GroundTruthFactList |
| T5 | GroundTruthStructure 构造 + roundtrip | GroundTruthStructure |
| T6 | GroundTruthConclusion 构造 + roundtrip | GroundTruthConclusion |
| T7 | KnownIssue 完整构造 + roundtrip | KnownIssue |
| T8 | ExpectedIssue（带 fact_id）+ roundtrip | ExpectedIssue |
| T9 | ExpectedIssue（不带 fact_id）+ roundtrip | ExpectedIssue |
| T10 | GroundTruthBundle 组装 + roundtrip | GroundTruthBundle |
| T11 | EvaluationCase 完整组装 + roundtrip | EvaluationCase |
| T12 | 所有 Optional 字段为 None 时的 roundtrip | 各 dataclass 的 from_dict 默认值处理 |

### 第二组：Validator 基本 pass 路径

| 编号 | 测试 | 覆盖 |
| ---- | ---- | ---- |
| V1 | validate_meta：gold / silver / bronze 合法 case_level 各一个 | validate_meta |
| V2 | validate_ground_truth_facts：列表全部合法 | validate_ground_truth_facts |
| V3 | validate_ground_truth_fact_list：列表全部合法 | validate_ground_truth_fact_list |
| V4 | validate_ground_truth_structure：全部合法 | validate_ground_truth_structure |
| V5 | validate_ground_truth_conclusion：全部合法 | validate_ground_truth_conclusion |
| V6 | validate_known_issues：空数组（无已知错误）+ 正常数组 | validate_known_issues |
| V7 | validate_expected_issues：空数组 + 正常数组 | validate_expected_issues |
| V8 | validate_case：gold fixture 全量通过 | validate_case（顶层） |
| V9 | validate_case：silver fixture 全量通过 | validate_case（顶层） |
| V10 | validate_case：bronze fixture 全量通过 | validate_case（顶层） |

### 第三组：Validator 失败路径（每个子 validator 至少覆盖一个错误项）

| 编号 | 错误 | 期望 |
| ---- | ---- | ---- |
| F1 | Meta.case_level = "platinum"（不在枚举） | validate_meta 报错 |
| F2 | Meta.case_id = "" | validate_meta 报错 |
| F3 | Meta.created_at = "not-a-date" | validate_meta 报错 |
| F4 | GroundTruthFact.fact_id = "not_a_valid_id" | validate_ground_truth_facts 报错（复用 Step 7-B grammar） |
| F5 | GroundTruthFact.fact_id = "fact:unknown.K001.field"（scope_class 非法） | validate_ground_truth_facts 报错 |
| F6 | GroundTruthFact.quantity_kind = "made_up" | validate_ground_truth_facts 报错 |
| F7 | GroundTruthFact.precision = -1 | validate_ground_truth_facts 报错 |
| F8 | fact_list.fact_ids 含空字符串 | validate_ground_truth_fact_list 报错 |
| F9 | KnownIssue.fact_id grammar 非法 | validate_known_issues 报错 |
| F10 | KnownIssue.confirmed_at 非法 | validate_known_issues 报错 |
| F11 | KnownIssue.original_value == correct_value | validate_known_issues 报错 |
| F12 | KnownIssue.confirmed_by = "" | validate_known_issues 报错 |
| F13 | ExpectedIssue.rule_id = "" | validate_expected_issues 报错 |
| F14 | ExpectedIssue.severity = "fatal"（不在枚举） | validate_expected_issues 报错 |
| F15 | ExpectedIssue.fact_id grammar 非法 | validate_expected_issues 报错 |
| F16 | GroundTruthConclusion.direction = "" | validate_ground_truth_conclusion 报错 |

### 第四组：顶层 Validator 跨文件一致性（validate_case）

| 编号 | 场景 | 期望 |
| ---- | ---- | ---- |
| C1 | gold 用例缺 ground_truth/facts.json | validate_case 报错 |
| C2 | gold 用例缺 ground_truth/fact_list.json | validate_case 报错 |
| C3 | gold 用例缺 ground_truth/structure.json | validate_case 报错 |
| C4 | gold 用例缺 ground_truth/conclusion.json | validate_case 报错 |
| C5 | silver 用例缺 ground_truth/facts.json（虽然只需要"部分"，但本阶段用 silver 最小 fixture 也带 facts） | validate_case 报错 |
| C6 | bronze 用例带了 ground_truth/facts.json | **不报错**（内容多于级别声明是合法的——只是级别声明不因此提升） |
| C7 | gold 用例的 ground_truth.facts 与 fact_list.fact_ids 完全不交集 | validate_case 给 warning（建议性） |
| C8 | 目录缺 meta.json | validate_case 报错 |
| C9 | EvaluationCase.from_dir 加载 fixture 目录 | 所有子文件正确解析 |

---

## G. 与 Step 7-B / Step 7-C 的依赖关系

| 依赖 | 如何复用 | 为什么 |
| ---- | ---- | ---- |
| Step 7-B `is_valid_fact_id()` | **直接 import 调用** | 三段式 grammar 是冻结的契约，不能重写 |
| Step 7-B `parse_fact_id()` | **直接 import 调用** | 跨文件一致性检查需要解析 fact_id 的 scope_class / instance_key / attribute |
| Step 7-B `QUANTITY_KINDS` 枚举 | **直接 import 引用** | ground truth 的 quantity_kind 必须与运行时 Fact 的 quantity_kind 使用同一个受控枚举 |
| Step 7-B `SCOPE_CLASSES` 枚举 | 间接（通过 is_valid_fact_id 已包含） | 不在本层直接使用 |
| Step 7-B `Fact` dataclass | **不复用** | GroundTruthFact 是轻量答案密钥，运行时 Fact 有 source_refs/method/provenance/review_status/status/revision/supersedes——两者语义不同 |
| Step 7-B `FactTypeRegistry` | 可选（用于更深层校验） | GroundTruthFact 的 attribute 是否在 registry 中可做 optional warning，但不强制——答案密钥应该比运行时更宽容 |
| Step 7-C Domain Objects | **无直接依赖** | Evaluation Case 的 ground truth 不引用 Domain Object id（只有 fact_id），本阶段也没有 Domain Object 级的 ground truth |

**Step 7-B / 7-C 不得修改什么**：
- 不新增字段到 Step 7-B 的 Fact / SourceRef / Conflict
- 不修改 registry.py 中的枚举值（除非扩展枚举值，那是合理的 registry 扩展，不是修改）
- 不修改 validate.py / validate_fact / validate_conflict
- 不修改 test_types.py / test_domain.py 中的任何已有测试
- Step 7-B 已有的 `step-7b-complete` tag 不动

---

## H. 明确留给 Step 7-E 以后的内容

| 编号 | 内容 | 留到哪一步 |
| ---- | ---- | ---- |
| L-1 | IssueList 与 expected_issues 的命中率比对 | Step 7-E Accuracy Validator |
| L-2 | 数值比对逻辑（G3 value vs 运行输出 value，按显示精度 round） | Step 7-E |
| L-3 | 判定逻辑一致性比对 | Step 7-E |
| L-4 | 数字锚定比对 | Step 7-E |
| L-5 | 结论覆盖度比对 | Step 7-E |
| L-6 | 结构一致性比对（P1） | Step 7-F Style Validator |
| L-7 | 文本一致性比对（P2） | Step 7-F |
| L-8 | 版式一致性比对（P3） | Step 7-F |
| L-9 | Run Snapshot | Step 7-G Regression Runner |
| L-10 | Baseline | Step 7-G |
| L-11 | 指标计算 / 门禁判定 | Step 7-G |
| L-12 | 回归报告 | Step 7-G |
| L-13 | 历史报告逆向抽取 | Step 7-I（逆向链路） |
| L-14 | LLM / Adapter / Skill | Step 7-H |
| L-15 | Template / Renderer / DOCX | Step 7-J |
| L-16 | 任何数据库 / Agent / UI | 永远不在 Step 7-D 范围内 |

---

## I. 可能存在的设计歧义

### 歧义 1：GroundTruthFact 的 precision 字段

05_EVALUATION.md §6.1 说"按显示精度进行，不按浮点相等"。但 precision 是来自 fact_type 注册表（`TemplateSpec.fact_display_spec[fact_type]`）的信息——ground truth 是否需要自己带 precision？

**两种选择**：
- 选择 A：GroundTruthFact 自带 precision
  - 优点：ground truth 自包含，评测比对时不依赖外部 TemplateSpec
  - 缺点：与 TemplateSpec 有潜在不一致（ground truth 注册 precision=2，TemplateSpec 注册 precision=3）
- 选择 B：GroundTruthFact 不带 precision，比对时从 TemplateSpec 查
  - 优点：单一真相源（TemplateSpec）
  - 缺点：ground truth 不是完全自包含，评测时需要额外加载 TemplateSpec

**本阶段建议**：选择 A，GroundTruthFact 带 precision（可选字段，默认 None 表示"按 TemplateSpec"）。理由是 Step 7-D 是自包含的数据层，不应与尚未实现的 TemplateSpec 产生编译期依赖。比对时 Step 7-E 会处理这个优先级（ground truth 的 precision 优先，缺省则查 TemplateSpec）。

### 歧义 2：bronze 用例的 ground_truth/facts.json 是否为空

05_EVALUATION.md §4 说 bronze "仅有报告（G2 + G4）"。但如果人工在构造 bronze 用例时顺便把 G3 也填了（因为方便），程序是否应该报错？

**建议**：bronze 用例 ground_truth/facts.json 可以为空也可以非空。validator 只在 **gold** 级别强制要求非空，bronze 只检查 fact_list + structure + conclusion 有内容即可。理由是 case_level 是**声明**，不是内容的自动提升器——bronze 带了 facts 不自动变成 gold，也不应该因为"内容超标"而报错。

### 歧义 3：GroundTruthFact 的 fact_id 引用 Step 7-B 哪个版本的 grammar

Step 7-B 冻结的是 v0.3.1 三段式（无 `.vN` 后缀，scope_class 含 defect）。如果后续 Step 7-D 实现时 Step 7-B 又变了怎么办？

**建议**：在 `GroundTruthFact` 模块头部注释里写 "Reuses Step 7-B v0.3.1 `is_valid_fact_id()` — grammar contract frozen at step-7b-complete tag"。如果 Step 7-B 后续有不兼容变更（极不可能，因为已打 step-7b-complete），Step 7-D 需要手动 review 再升级 import。

### 歧义 4：inputs/ 目录的处理

05_EVALUATION.md §5 定义了 `inputs/` 目录，但里面是原始文件（Excel、PDF、图片、docx），不是 JSON。Step 7-D 的 validator 是否要检查 inputs/ 目录存在？

**建议**：validator 只做"meta.json 声明了什么，就检查什么"——即 gold 用例的 meta.json 里如果声明了 inputs 文件列表（或只声明 "inputs/" 目录存在），就检查；如果没有声明，validator 跳过。本阶段 inputs/ 目录的处理最小化：只登记路径名，不解析文件内容。实际文件解析由 Pipeline Step1（parse_xlsx / parse_docx_template / parse_docx）负责。

### 歧义 5：expected_issues.rule_id 的格式

05_EVALUATION.md §5.2 示例用的是 `"FACT.MISSING"` 这种点分段格式。但目前没有正式的 rule_id 注册表。

**建议**：本阶段 rule_id 就是普通 str，不强制任何格式。在 validator 里只检查非空。Step 7-E Accuracy Validator 实现时再建立 rule_id 注册表，届时可以做"rule_id 是否在注册表中"的更深层校验——但那是 Step 7-E 的 validator 的事，不是 Step 7-D 的事。

---

## J. 最终推荐的 Step 7-D 最小实施范围

### 必须做的（不可裁剪）

1. **`src/eval/enums.py`**：CASE_LEVELS（gold/silver/bronze）、SEVERITIES（error/warning）两个受控枚举
2. **`src/eval/types.py`**：8 个 dataclass——Meta、GroundTruthFact、GroundTruthFactList、GroundTruthStructure、GroundTruthConclusion、GroundTruthBundle、KnownIssue、ExpectedIssue、EvaluationCase
3. **`src/eval/validator.py`**：7 个子 validator + 1 个顶层 validate_case（跨文件一致性），全部函数式风格，与 Step 7-B/7-C 一致
4. **`src/eval/__init__.py`**：export 所有公开类和函数
5. **`tests/eval/test_types.py`**：所有 dataclass 的构造 + JSON roundtrip（约 15 个 test class）
6. **`tests/eval/test_validator.py`**：三个分组——基本 pass（~10）、失败路径（~15）、跨文件一致性（~10），合计 ~35+ 个测试
7. **`tests/eval/fixtures/`**：5 个最小 fixture 目录——case_gold_minimal、case_silver_minimal、case_bronze_minimal、case_gold_missing_file、case_gold_level_mismatch
8. **`tests/eval/test_loader.py`**：from_dir 加载 fixture 目录的集成测试

### 可选但推荐的

1. 在 `types.py` 的每个 dataclass 头部注释里标注对应的 05_EVALUATION.md 章节
2. 在复用 Step 7-B 的 import 处加行注释标注具体复用函数（便于后续维护时快速定位依赖链）
3. 从 fixture 生成一版 fixture.json（用于人工 review）——但**不**做 fixture 到 gold 的自动提升

### 明确不做的（请再确认一遍）

- **不复用** Step 7-B 的 Fact dataclass（只用它的 id grammar 和枚举）
- 不定义任何比对逻辑
- 不调用任何 LLM
- 不解析 inputs/ 目录的真实内容
- 不实现任何 Store / Pipeline / Agent / Skill
- 不修改 Step 7-B / 7-C 的任何代码文件
- 不修改 05_EVALUATION.md 或其他设计文档
- 不创建 git commit

### 验收标准（Step 7-D 完成的判定）

1. `pytest tests/eval/` 全部通过（新增的测试，加上不影响 Step 7-B/7-C 的 128 个测试）
2. `python -c "from eval import *"` 无 ImportError
3. validator 对 5 个 fixture 目录的判定符合预期（gold fixture pass，missing_file fixture fail 且给出明确错误信息）
4. 无任何设计文档或 Step 7-B/7-C 代码被修改
5. 打 tag `step-7d-complete`

---

以上是完整的 Step 7-D 实施方案。等你确认后再进入执行。
