用户指出了 Python dataclass 的硬语法错误：有默认值的字段必须出现在没有默认值的字段之后。这是正确的 Python dataclass 语法规则。让我做一次机械检查，然后给出修正后的结果。

我需要检查 v4 中全部 9 个 dataclass 的字段顺序：

1. Meta - 让我检查
2. GroundTruthFact - 有问题（unit 在 quantity_kind/precision 前面）
3. GroundTruthFactList - 只有 fact_ids，没问题
4. GroundTruthStructure - 检查
5. GroundTruthConclusion - 检查
6. GroundTruthBundle - 检查
7. KnownIssue - 有问题（fact_id 在 original_value 前面）
8. ExpectedIssue - 检查
9. EvaluationCase - 检查

让我逐一过一遍，确保每个 dataclass 都是"所有无默认字段在前，有默认字段在后"。
机械检查开始，逐个过 9 个 dataclass。

---

## 1. 修正后的两个 dataclass

### GroundTruthFact

```python
@dataclass
class GroundTruthFact:
    """G3 事实值 Ground Truth — 轻量答案密钥。"""
    fact_id: str                    # required
    value: Any                      # required — 允许 int|float|str|bool，不允许 None
    quantity_kind: str              # required — QUANTITY_KINDS 枚举
    precision: int                  # required — 显式提供，无默认值
    unit: Optional[str] = None     # optional-default — 定量事实显式给值，定性事实省略
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "GroundTruthFact": ...
```

**字段顺序调整**：unit（有默认值）移到末尾。语义完全不变。

### KnownIssue

```python
@dataclass
class KnownIssue:
    """历史报告中的已知错误登记。"""
    issue_type: str                  # required — "fact_value_error" | "conclusion_error"
    original_value: Any              # required
    correct_value: Any               # required — original_value != correct_value
    reason: str                       # required
    confirmed_by: str                 # required
    confirmed_at: str                 # required — ISO 8601
    fact_id: Optional[str] = None   # conditional-required — Python 层默认 None，
                                    # validator 根据 issue_type 决定是否必须非 None
    
    def to_dict(self) -> dict: ...
    @classmethod
    def from_dict(cls, data: dict) -> "KnownIssue": ...
```

**字段顺序调整**：fact_id（有默认值）移到末尾。语义完全不变——conditional-required 的判断逻辑由 validator 层的 `issue_type="fact_value_error" → fact_id 必须非 None` 规则执行，Python dataclass 层只是允许 None 进入对象。

---

## 2. 9 个 dataclass 字段顺序机械检查结果

检查规则：所有无默认值的字段必须出现在所有有默认值字段之前。

### 检查清单

| # | Dataclass | 字段序列（default 状态） | 结果 |
| ---- | ---- | ---- | ---- |
| 1 | **Meta** | case_id(r) → case_level(r) → report_type(r) → source_project(r) → created_at(r) → design_version(d) → traps(d) → notes(d) | ✅ 合法 |
| 2 | **GroundTruthFact** | fact_id(r) → value(r) → quantity_kind(r) → precision(r) → **unit(d)** | ✅ 合法（已修正） |
| 3 | **GroundTruthFactList** | fact_ids(r) | ✅ 合法（只有 1 个无默认字段） |
| 4 | **GroundTruthStructure** | chapter_paths(r) → table_count(r) → table_headers(r) → figure_count(r) → required_elements(r) | ✅ 合法（全部无默认） |
| 5 | **GroundTruthConclusion** | direction(r) → required_evaluation_ids(r) → **notes(d)** | ✅ 合法 |
| 6 | **GroundTruthBundle** | facts(r) → fact_list(r) → structure(r) → conclusion(r) | ✅ 合法（全部无默认） |
| 7 | **KnownIssue** | issue_type(r) → original_value(r) → correct_value(r) → reason(r) → confirmed_by(r) → confirmed_at(r) → **fact_id(d)** | ✅ 合法（已修正） |
| 8 | **ExpectedIssue** | rule_id(r) → severity(r) → **fact_id(d)** → **note(d)** | ✅ 合法 |
| 9 | **EvaluationCase** | meta(r) → ground_truth(r) → known_issues(r) → expected_issues(r) → **case_dir(d)** | ✅ 合法 |

**检查结论：9/9 合法。无其他需要调整的 dataclass。**

### 被调整的字段对 `from_dict` / `to_dict` 的影响

字段顺序调整**不影响** to_dict / from_dict 的实现。JSON key 顺序不影响 `json.load()` 的解析，`dataclass.asdict()` 默认按定义顺序输出，但 JSON 消费方也不关心 key 顺序。from_dict 只需用 `data["fact_id"]`、`data["value"]` 按 key 取，与 dataclass 字段定义顺序无关。

唯一需要注意的是 **Python 代码中直接构造实例时的位置参数顺序**——例如 `GroundTruthFact(fact_id="...", value=42, quantity_kind="length", precision=3, unit="mm")` 这个调用现在的位置参数顺序和之前不同。但这不影响 JSON roundtrip 和 validator。

---

## 3. Step 7-D v4 Coding Contract 是否可以直接编码

**可以。**

- 9 个 dataclass 的字段顺序**全部合法**（已修正 GroundTruthFact 和 KnownIssue）
- 没有使用 `kw_only` / `InitVar` / 自定义 `__init__` / Pydantic / 第三方 dataclass 库
- 所有数据模型语义**零变化**（只是位置参数顺序变了，这不是设计变更）
- 其余所有设计（validator、测试矩阵、fixture 结构、跨文件一致性规则、gold/silver/bronze 规则）**保持 v4 不变**

直接进入编码阶段即可。
