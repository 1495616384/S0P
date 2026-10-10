# Step 8 独立技术审查报告（Independent Review Report）

> **日期**：2026-10-10
> **审查人**：独立架构/代码/验证证据审查员（非 Step 8 实施者）
> **审查依据**：`# S0P Step....txt`（S0P 独立审查任务）；`docs/Step_8_Coding_Contract_v1.md`（契约基线）；`reviews/Step_8_Independent_Review_Package.md`（审查包）
> **审查性质**：只读。未修改任何生产代码、测试、fixture、冻结契约。
> **审查结论**：**ACCEPT WITH NON-BLOCKERS**

---

## A. 审查范围

### 读取的权威文档
- `docs/02_ARCHITECTURE.md`、`docs/03_CANONICAL_DATA_MODEL.md`、`docs/04_REPORT_IR.md`、`docs/05_EVALUATION.md`、`docs/07_DECISIONS.md`
- `docs/Step_8_Coding_Contract_v1.md`、`docs/Step_8_Design_Decision_Closure.md`、`docs/Step_8_Fixture_Design_Sprint.md`、`docs/Step_8_Sample_Component_Relation_Design_Decision.md`、`docs/Step_7_F_Final_Freeze.md`
- `reviews/Step_8_Independent_Review_Package.md`

### 读取的生产代码（Step 8 全部 10 模块）
- `src/parsers/base.py`、`src/parsers/xlsx.py`
- `src/facts/mapper.py`、`src/facts/store.py`、`src/facts/conflict.py`
- `src/compute/unit_registry.py`
- `src/rules/criterion.py`、`src/rules/conclude.py`
- `src/ir/builder.py`、`src/ir/adapter.py`

### 读取的冻结验证层（交叉核对）
- `src/eval/accuracy/data_structures.py`、`src/eval/accuracy/validator_framework.py`、`src/eval/accuracy/__init__.py`、`src/eval/accuracy/binding.py`、`src/eval/accuracy/tokenizer.py`
- `src/eval/types.py`、`src/eval/run/runner.py`
- `src/cdm/types.py`、`src/cdm/registry.py`、`src/cdm/id.py`

### 读取的测试
- `tests/e2e/test_step8_e2e.py`（12 E2E；正向 6 + 负向）

### 读取的 fixture
- `tests/eval/fixtures/case_step8_defect_v1/meta.json` + `notes.md` + `inputs/inspection.xlsx`

### 实际独立运行的测试（原始输出见 F）
- `pytest tests/ --ignore=tests/eval/test_validator_framework.py` → **583 passed**
- `pytest tests/eval/test_step7f.py -q` → **139 passed**
- `pytest tests/e2e/test_step8_e2e.py -q` → **12 passed**
- `pytest tests/eval/test_validator_framework.py -q` → **1 error（P-1，预存）**

---

## B. 实现与 Coding Contract 追踪矩阵

判定约定：PASS / FAIL / PARTIAL / NOT VERIFIED / NOT APPLICABLE（附依据）。

| 契约条款 | 要求 | 实际文件/函数 | 验证证据 | 判定 |
|---|---|---|---|---|
| §2 责任边界 | Parsing ≠ Semantic Interpretation；Parser 不推断语义 | `src/parsers/xlsx.py` 仅做 openpyxl 读取/坐标/header 位置识别；`src/facts/mapper.py` 独立承担映射 | xlsx.py L1-13 注释明确"绝不进行 scope/fact_type/单位推断"；映射逻辑全在 mapper.py | PASS |
| §5 RawSource/CellRaw/ParseResult Schema | CellRaw(sheet,row,col,raw_value,raw_text,excel_address) | `src/parsers/base.py` L44-51、L55-70 | 字段与契约一一对应；excel_address 用 `{sheet}!{col}{row}` | PASS |
| §6 CandidateFact | 瞬态桥接对象，不带 Fact.status；Mapper 输出 List[CandidateFact] | `src/facts/mapper.py` `CandidateFact` L38-51；`map_cells_to_candidates` | 字段无 status/review_status/revision；入库时才由 Fact 默认构造补齐 | PASS |
| §6 列名精确匹配 / 确定性 | 固定 6 列表头精确匹配；未知列头/缺列/重复列→错误 | mapper.py `_COLUMNS` L77-84；L120-137 | 未知/重复/缺失表头均 `ValueError`，绝不静默 | PASS |
| §7 FactStore 状态机 | 继承 CDM §15；status ∈ {filled,missing,conflict,rejected,...}；review_status 独立 | `src/facts/store.py` FactStore | ingest→filled/pending；mark_missing→missing+value=None；mark_conflict→conflict+manual；mark_rejected；无 resolve_conflict | PASS |
| §7 禁止发明非法 status / pending 非 Fact.status | pending 仅是 review_status 合法值 | store.py 构造 Fact 时用 types.py 默认（filled/pending） | 未在任何地方将 pending 赋给 Fact.status | PASS |
| §8 Conflict 第一版仅记录 | resolution_policy=manual；resolved_by/resolution=None；**不提供 resolve_conflict** | `src/facts/store.py` L72-98、`src/facts/conflict.py` | new_conflict 固定 manual；store 无 resolve_conflict 方法 | PASS |
| §7/§8 禁止 conflict→superseded / 伪造 revision/supersedes | store/conflict 不得实现未经批准的转换或版本伪造 | store.py/conflict.py 全文检索 | 无 supersedes/revision 写操作；冲突仅记录候选值 | PASS |
| §9 Unit Registry | mm/m→length；同量纲换算；不存储换算值；未知单位报错 | `src/compute/unit_registry.py` `UNIT_TO_QUANTITY`、`convert` | mm→length、m→length；换算经基准 mm；未知单位 ValueError | PASS |
| §9 统计 Compute = ∅ | **不创建 calculate.py / compute_statistic()** | `src/compute/` 仅 `__init__.py`+`unit_registry.py` | 无 calculate.py；synthetic 统计行为不存在 | PASS |
| §10 Criterion | Fact+Criterion→Evaluation；review_status≠confirmed→not_evaluable；Decimal 精确比较 | `src/rules/criterion.py` `evaluate` L93-125 | 五重闸门（criterion/fact status、value 缺、量纲、未知单位）；operator 提取；between 显式报错 | PASS |
| §11 ConclusionRule | step8_defect_slice_v1：逐构件聚合+全局；unqualified 优先；空输入→missing；covers[]=6 | `src/rules/conclude.py` `conclude` | `_fold_evaluations`：unqualified>not_evaluable>qualified；空输入 missing；defect_host_map 分组 | PASS |
| §12 Report IR | 符合 04 Schema；只产 Heading/Paragraph(assertion)/Table/Signature；不产 Narrative | `src/ir/builder.py` `build` | Document 结构 + 3 Section；`_DIRECTION_TEXT` 受控枚举；无不受控自由文本 | PASS |
| §13 SystemOutput Adapter | Report IR→SystemOutput；Ref→显示值+AnchorDeclaration；display policy 与 P0-1 一致 | `src/ir/adapter.py` `to_system_output` | 构造冻结 data_structures.SystemOutput；2 位小数 ROUND_HALF_UP 与 fact_display_spec.decimals=2 一致；anchor 含 char_start/char_end | PASS |
| §13 显示精度 | 不得用 float 自然精度；数值→Decimal 2 位小数+单位 | adapter.py `_display_value` L51-72 | Decimal(str) + quantize(ROUND_HALF_UP)，禁 float 直接 str | PASS |
| §14 E2E 关键门禁 S8-S-10/11 | 真实 Excel → SystemOutput → evaluate_case() 消费，7 条 P0 | `tests/e2e/test_step8_e2e.py` TestPositiveE2E | 独立实测 12 passed；applicable_rule_count=7 | PASS |
| §15 测试覆盖 | 各模块单测/集成/E2E；回归不破坏 Step7-E/F | tests/parsers,facts,compute,rules,ir,integration,e2e | CMD1 全量 583 passed；CMD2 139 passed | PASS |
| §17 OOS 冻结 15 条 | 不引入 OOS 项 | 见 G 节遍检 | 无越界 | PASS |
| §20 偏差登记 | 不允许静默 deviation | 审查包 §7 登记 D-STEP8-14~23 | 已登记、未改契约文本（均未破坏冻结语义，见 C） | PASS |

**小结**：契约核心语义逐条有实现证据并一致，无 FAIL；14/15 项 PASS，1 项（§20 偏差登记）经 C 节裁决全部为非阻塞。

---

## C. D-STEP8-14 ~ D-STEP8-23 裁决表

| 编号 | 实际偏差 | 信息/行为类 | 对契约语义影响 | 正确性/确定性 | 严重程度 | 审查裁决 |
|---|---|---|---|---|---|---|
| D-STEP8-14 | Mapper 落位 `src/facts/mapper.py` | 落位声明 | 无 | 无影响 | Informational | **接受**（建议契约补录路径） |
| D-STEP8-15 | `FactStore.mark_confirmed()` 为补充接口 | 接口补充 | 无变更（仅 filled→confirmed，继承 CDM） | 无 | Minor | **接受**：无此闸门判定链无法启动；建议并入 Proposed 清单 |
| D-STEP8-16 | `build(fact_set, domain_objects, conclusion)` 三参数 | 签名 | 无（Closure L373 即三参） | 无 | Informational | **接受**（Contract §12 片段未同步，建议修订注记） |
| D-STEP8-17 | `to_system_output(..., *, evaluations=())` | 签名增量 | 无（可选参，P0-3 所需） | 无 | Minor | **接受** |
| D-STEP8-18 | `conclude(*, defect_host_map, rule_id=...)` | 签名增量 | 无（分组键必须外部 L3 导出，禁命名推断） | 保证确定性 | Minor | **接受**（Contract §10 片段未同步，建议修订注记） |
| D-STEP8-19 | Anchor span = 显示值 span ∪ 相交 token span | 行为细化 | 无（token/fact_id 不变，仅扩 span） | 消除 BIND-7/8 假阳性 | Minor | **接受**（P0-4 PASS 验证） |
| D-STEP8-20 | `header_row=None`/`data_rows` 语义明确 | 语义细化 | 无（确定性声明） | 无 | Informational | **接受**（建议修订 Contract §5 注释） |
| D-STEP8-21 | `CellRaw(..., excel_address)` vs Closure `position` | 参数名 | 无 | 无 | Informational | **接受**（以 §5 为准） |
| D-STEP8-22 | `tests/facts/test_mapper.py` 落位 | 测试落位 | 无 | 无 | Informational | **接受** |
| D-STEP8-23 | `openpyxl==3.1.2` 未写入 requirements.txt | 依赖声明缺漏 | 无（环境已装） | 复现性缺口 | **Minor（需动作）** | **接受为非阻塞；需补录依赖**（见 I-IV） |

**逐项结论**：10 项中，D-STEP8-14/16/20/21/22 = Informational；D-STEP8-15/17/18/19 = Minor 实现补充；D-STEP8-23 = Minor 且需动作（依赖声明）。**均未改变已批准契约语义，不影响正确性/来源追溯/确定性/失败处理，不影响 P0，不影响历史可复现，不需为满足当前契约而改动代码，可通过正式决议接受为非阻塞。** 无需 REQUEST CHANGES。

---

## D. 五个待裁决问题（审查包 §9）

| # | 问题 | 证据 | 影响 | 是否违反契约 | 阻塞验收 | Non-Blocker | 推荐处理 | 需另行批准 |
|---|---|---|---|---|---|---|---|---|
| 1 | 是否接受 10 项偏差登记与处置 | 审查包 §7；本文 C 节 | 契约文本未同步的为 Informational/Minor | 否 | 否 | 是 | 接受；契约注记修订另立 Session | 是（契约修订需批准） |
| 2 | `openpyxl` 依赖落位 | D-STEP8-23；env 已装 3.1.2 | 复现性 | 否 | 否 | 是 | 建议补入 requirements.txt（或另立依赖清单） | 是（改动冻结文件需批准，故仅建议） |
| 3 | `mark_confirmed` 是否认可 | D-STEP8-15；store.py L123 | 无 | 否 | 否 | 是 | 认可并并入 Proposed 清单 | 建议（并入契约需批准） |
| 4 | P-1 处理路径 | OOS-8 + F 节复现 | 仅该测试文件无法整体收集 | 否（OOS-8 明确不修复） | 否 | 是 | 维持 OOS-8 排除；如需另开独立修复项 | 由负责人决定 |
| 5 | Step 8 Contract 是否具备 FROZEN 条件 | 本报告整体证据 | — | — | 否 | — | 建议按验收流程单独 Session 批准；本审查不自行宣告 FROZEN | 是（独立 Session） |

---

## E. 核心模块审查（D1-D9）

- **D1 Parser**（`src/parsers/xlsx.py`）：纯结构解析，无 `scope_class/fact_type/单位/key/同义` 推断（模块 docstring L12 明示）。used-range 矩形坐标完整产出（含矩形内空位，坐标稳定）。空白/未知列头/非法数据由 Mapper 边界显式 `ValueError`，Parser 层不静默。原始 `raw_value` 与 `raw_text` 均保真（`raw_text_of` 仅确定性字符串化，不改写值）。**PASS**
- **D2 Mapper**（`src/facts/mapper.py`）：6 列表头精确匹配；未知表头/重复/缺列 → `ValueError`（L128-137）。实例 key=CellRaw 文本原样（L166-167）；行级 context（sheet,row）导出 L3 归属（`_row_groups` L214），来源 = `source_refs` 位置（结构信息非推断）。`source_refs` 指向 `{sheet}!{col}{row}` 精确单元格（L194）。不注册新类型（fact_type 全部来自 `_COLUMNS` 冻结集）。**PASS**
- **D3 Fact Store**（`src/facts/store.py`）：CandidateFact→Fact 用默认构造（filled/pending）（L56-65）；未确认事实不进入 `get_confirmed_facts`（仅 filled+confirmed）（L102-107）；同值幂等、异值记录 Conflict 而非覆盖（L68-72，禁 last-writer-wins）。mark_missing→value=None；mark_conflict 仅记录（policy=manual）；mark_rejected；**无 resolve_conflict**。未伪造 revision/supersedes，未把 conflict 转 superseded。**PASS**
- **D4 Unit Registry**（`src/compute/unit_registry.py`）：mm/m→length 正确；同量纲换算经基准 mm（Decimal 精确，`convert` L45-55）；无跨量纲比较（Criterion 依赖量纲匹配闸门）；原始值不存换算结果；未知单位 `ValueError`；显示精度采用契约规定（adapter 2 位小数）。**PASS**
- **D5 统计 Compute**：`src/compute/` 仅 `unit_registry.py`，**无 calculate.py/compute_statistic**；单位换算未被误称统计；无隐藏统计、无缺失模块依赖。**PASS**
- **D6 Criterion**（`src/rules/criterion.py`）：Criterion 字段符合 CDM；量纲/单位比较经 `convert`；operator 提取得 `<=`；边际由 criterion 比较语义决定（0.30 边界 equal → qualified 实测）；未确认/缺值/量纲不符/未知单位 → `not_evaluable`（L95-113）；`not_evaluable` 不被误转 qualified（仅 `holds` 为真才 qualified）；`between` 显式报错。**PASS**
- **D7 ConclusionRule**（`src/rules/conclude.py`）：`conclude` 逐构件按 `host_component_id` 分组（L76-85），`_fold_evaluations` 实现「任一 unqualified→unqualified；否则任一 not_evaluable→insufficient_evidence；否则 qualified」（L49-54）；全局同理（L87-93）；空输入→missing（L67-74）。rule_id 记录；input_evaluation_ids 全量；cover 由 builder 排序后给出 6 条（D-STEP8-09 一致）。E2E 实测全局为 unqualified（D003 超限）。**PASS**
- **D8 IR Builder**（`src/ir/builder.py`）：Document(version 0.3.1.1)+3 Section，符合 04 Schema；方向与覆盖一致（conclusion.direction 与段文本来自受控枚举 `_DIRECTION_TEXT`）；引用 Fact 均在 fact_set（`_ref` 校验 L52-55）；数值经受控 Ref（非自由文本）；仅产 Heading/Paragraph(assertion)/Table，未新增未批准 IR 字段，未超范围生成 Narrative/DOCX/PDF。**PASS**
- **D9 IR Adapter**（`src/ir/adapter.py`）：SystemOutput 构造字段与冻结 `data_structures.SystemOutput` 完全匹配；Ref 物化为显示值 + anchor（`_materialize_parts`）；TableCell 逐格 `_materialize_cell`；anchor 绑定正确 fact_id 且 char_start/end 与文本严格匹配（含 D-STEP8-19 span 扩展）；多个数值 token 每 token 独立声明；标识符 token 走冻结 tokenizer；显示精度 2 位小数符合 P0-1 fact_display_spec.decimals=2；文本物化无遗漏/重复/错误绑定（E2E P0-4 PASS、确定性二次运行一致）。**PASS**

---

## F. E2E 与七项 P0 证据

### F1. 正向 E2E（独立复跑，`tests/e2e/test_step8_e2e.py` 12 passed）
实测 `result.rule_statuses` 与实施者记录完全一致：

| P0 | 状态 | 证据类型 |
|---|---|---|
| VALUE.CONSISTENCY | PASS | 正向（绑定无 value_mismatch） |
| TABLE.INTERNAL | **NOT_EVALUABLE** | trusted 仅声明 detail_row_range 无启用检查 → 不带 issues（test_p0_2_not_evaluable_without_checks） |
| LOGIC.CONSISTENCY | PASS | 正向（system_evaluations 与 expected 对齐） |
| ANCHOR.BINDING | PASS | 正向（BIND-7/8 无假阳性） |
| CONCLUSION.DIRECTION | PASS | 正向 |
| CONCLUSION.COVERAGE | PASS | 正向 |
| EXPECTED.HIT | PASS | 正向 |
| **case_status** | NOT_EVALUABLE | 分母含 not_evaluable：pass 6 + not_evaluable 1 |
| evaluable_rule_count | 6 | — |
| all_actual_issues | [] | — |

**重要判定（S0P Task E1/E3）**：`6 项 PASS + TABLE.INTERNAL NOT_EVALUABLE` **不等于**"七项 P0 全通过"。TABLE.INTERNAL 的 NOT_EVALUABLE 是可信输入契约允许的状态（未声明启用 P0-2 合计/统计检查），**不得计入正向通过证据**，但也**不构成缺陷**。case_status=NOT_EVALUABLE 的聚合正确（not_evaluable 计入分母）。**当前 Coding Contract（S8-S-10/11）接受该覆盖边界**，不要求 P0-2 必须 PASS。

### F2. 负向路径（逐条核对实际测试代码）
- **Direction 篡改**（`conclusion_status="qualified"`）→ P0-5 FAIL（`conclusion_direction_mismatch`，location=`report_ir.conclusion.direction`），其余规则不受影响。✓
- **value 篡改**（D001.width 0.12→0.13）→ P0-1 FAIL（`value_mismatch`，fact_id=`fact:defect.D001.width`），ANCHOR 保持 PASS、LOGIC 保持 PASS；显示值已重化 `"0.13 mm"`。✓
- **ExpectedIssue 命中/未命中** → P0-7 PASS / FAIL（`expected_issue_not_hit`）。✓
- **缺失**：mark_missing → eval `not_evaluable`（gate:fact_status=missing）→ 全局 `insufficient_evidence` → Adapter 拒绝渲染（ValueError 缺失）。✓
- **冲突**：注入 0.33 → Conflict(policy=manual, candidates{0.35,0.33}, resolved_by/resolution=None) → 不入 confirmed → eval `not_evaluable`（gate:fact_status=conflict）→ `build(confirmed-only)` 显式拒绝。✓
- **未确认事实**不进入方向性判定；锚点偏移/错误 fact_id/数值不一致被冻结 P0 检出（P0-4 绑定链路）。✓

### F3. P0 覆盖边界结论
- 正向 PASS 证据：P0-1/2(NE)/3/4/5/6/7（P0-2 为 NOT_EVALUABLE）
- 负向 FAIL 证据：P0-1、P0-5、P0-7（E2E 显式）
- P0-3、P0-4 在 Step 8 E2E 内仅有正向 PASS；负向通过冻结 Step 7-E 既有规则单元测试覆盖（本审查不重复验证 Step 7-E 负例）

**无被意外跳过/静默改 PASS 的规则**。

---

## G. 独立测试结果（原始输出）

环境：Windows / Python 3.11.7 / pytest 8.4.1 / openpyxl 3.1.2 / rfc8785 OK；`pytest.ini` pythonpath=src。

| 命令 | 实测结果 | 实施者记录 | 差异 |
|---|---|---|---|
| `python -m pytest tests/ --ignore=tests/eval/test_validator_framework.py -q` | **583 passed in 1.30s** | 583 passed | 无 |
| `python -m pytest tests/eval/test_step7f.py -q` | **139 passed in 0.31s** | 139 passed | 无 |
| `python -m pytest tests/e2e/test_step8_e2e.py -q` | **12 passed in 0.56s** | 12 passed | 无 |
| `python -m pytest tests/eval/test_validator_framework.py -q` | **1 error（collection）** | 1 error | 无 |

- 三个集合存在覆盖重叠（step7f/e2e 均被包含在 tests/ 全量），**不将三者简单相加**。
- **已知 P-1 复现确认**：`ImportError: cannot import name 'RuleStatus' from 'eval.accuracy'`。实证定位：`RuleStatus` 定义于 `src/eval/accuracy/validator_framework.py`，但 `src/eval/accuracy/__init__.py` 未导出；`src/eval/run/runner.py`（L23）与 E2E 测试均直接自 `validator_framework` 导入 RuleStatus，不依赖根包导出，故不受影响。`docs/Step_7_F_Final_Freeze.md` 已将该缺陷定性为 **PRE-EXISTING Step 7-E export defect / OOS-8**，且本审查核实该文件 mtime=2026-09-30（Step 8 实施前），**与本次改动无关，非回归**。按 OOS-8 不修复，符合契约。
- 未执行的测试及原因：`test_validator_framework.py` 因 P-1 无法整体收集，按任务规定 ignore。

---

## H. 冻结契约和 OOS 检查

| 核验项 | 证据 | 判定 |
|---|---|---|
| 冻结 docs/02/03/04/05/07 未修改 | mtime ≤ 2026-09-30；git 无 M/D | PASS |
| `src/cdm/registry.py` 未扩展 | mtime=2026-09-22；`src/cdm/*` 全部 tracked 无修改 | PASS |
| `src/eval/accuracy/**` 未修改 | binding/tokenizer/frozen_round/data_structures mtime ≤09-29；validator_framework.py 为 Step7-E 既有（09-30，非 Step8 新增内容） | PASS |
| 未用改 P0/Validator 通过测试 | 全量 583 为既有+新增断言，实跑通过 | PASS |
| 未改 Step 7-F JCS/哈希/运行契约 | `src/eval/run/hash_utils.py` mtime=10-08（Step7-F 收口范围），Step8 文件均 10-09；step7f 139 复现 | PASS |
| 未新增 calculate.py 等排除计算 | `src/compute/` 仅 unit_registry.py | PASS |
| 未引入 LLM/DOCX-PDF Renderer/Pipeline | Step8 模块仅 10 个，无此类 | PASS |
| 未创建 OOS-15 ground_truth/expected_issues/known_issues | fixtures 全仓检索无此类文件；meta.json 明示"NOT an Evaluation Case (no ground_truth/expected_issues by OOS-15)" | PASS |
| 未改既有 7 个 fixture | `tests/eval/fixtures/` 仅新增 case_step8_defect_v1 | PASS |
| 无跨模块重复实现 | 模块职责单一对应契约模块表 | PASS |
| 无测试配置跳过关键验证 | E2E applicable_rule_count=7，7 条全跑 | PASS |

---

## I. Blocking / Major / Minor / Informational 问题清单

### Blocking（无）
未发现违反冻结契约、导致核心数据语义错误、可生成错误结论、绕过 P0 或使 E2E 验收不成立的问题。

### Major（无）
未发现重要契约实现不完整、关键边界条件错误或可追溯/确定性严重不足。

### Minor（3）
- **M1**：D-STEP8-23 —— `openpyxl==3.1.2` 未写入 `requirements.txt`（当前环境已装 3.1.2，功能不受影响，但存在**复现性缺口**：全新环境无法据依赖清单安装）。建议补录。
- **M2**：D-STEP8-19 —— Anchor span 由「显示值 span」扩展为「显示值 span ∪ 相交 token span」，属行为细化，需在契约注记中固化（当前仅以偏差登记存在）。不影响正确性（P0-4 PASS 验证）。
- **M3**：P0-3（LOGIC.CONSISTENCY）/ P0-4（ANCHOR.BINDING）在 Step 8 E2E 内仅具正向 PASS 证据，负向由冻结 Step 7-E 既有规则测试覆盖。建议在后续 E2E 补显式负例以增强可追溯（非当前验收阻塞）。

### Informational（5）
- D-STEP8-14/16/20/21/22 为落位/签名注释/参数名/测试落位层面声明，均不改变语义，建议随契约修订统一补录。

---

## J. 最终独立审查结论

**ACCEPT WITH NON-BLOCKERS**

理由：
1. **实现与 Coding Contract 一致**：核心 16 项追踪矩阵无 FAIL；Parsing≠Semantic Interpretation、FactStore 状态机、Conflict 仅记录无裁决、统计 Compute=∅、Unit Registry、Criterion 五重闸门、ConclusionRule 聚合、IR Builder/Adapter、SystemOutput 与冻结 data_structures 匹配，均逐条有真实文件/函数证据。
2. **独立复跑与实施者记录完全一致**：583 / 139 / 12 passed；P-1 复核确认为同一预存 OOS-8 缺陷（非回归）。
3. **负向证据真实有效**：P0-1/P0-5/P0-7 失败路径、缺失/冲突不确定性保留在 E2E 中显式验证，未为凑全绿改动任何被跳过逻辑。
4. **冻结契约与 OOS 边界零越界**：mtime + git status 双证据独立一致；未改冻结 docs/registry/accuracy/run；未引入 OOS 项；未创建 OOS-15 文件。
5. **10 项偏差全部为 Informational/Minor 非阻塞**，均不改变已批准契约语义，不产生 Blocking/Major 问题。

**注意事项**：`6 项 PASS + TABLE.INTERNAL NOT_EVALUABLE` 不构成"七项全通过"，但该覆盖边界为当前 Coding Contract 所明确接受，不阻塞验收；验收时不得将 P0-2 NOT_EVALUABLE 计作正向通过。

---

## K. 后续动作

本轮为只读审查，**不实施任何修复**。以下为进入正式验收前的建议动作，交由独立修复/后续 Session 处理：

1. **低风险必做**：将 `openpyxl==3.1.2`（及 Step 8 运行依赖）补录依赖声明（写入 `requirements.txt` 或另立清单）——需获批后修改（涉及冻结范围边界，按流程审批）。
2. **建议**：随契约修订统一 D-STEP8-14/16/20/21/22 的落位/签名/注释；将 D-STEP8-15/17/18/19 并入 Proposed 接口清单与契约注记（需独立 Session 批准）。
3. **建议**：为 P0-3/P0-4 增补显式负例（增强可追溯，非阻塞）。
4. **P-1**：维持 OOS-8 排除；如需正式修复探测器缺口，另开独立修复项，不在 Step 8 范围内。
5. **验收前置**：审查结论为 ACCEPT WITH NON-BLOCKERS，具备进入正式验收的条件；但**不自行修改 Coding Contract 状态、不自行宣告 Step 8 FROZEN**——FROZEN 升级与契约文本修订须由独立 Session 批准。

---

## L. 可复现证据

### 审查命令
```text
工作目录：g:\workspace\zixun4
(1) python -m pytest tests/ --ignore=tests/eval/test_validator_framework.py -q   → 583 passed in 1.30s
(2) python -m pytest tests/eval/test_step7f.py -q                                → 139 passed in 0.31s
(3) python -m pytest tests/e2e/test_step8_e2e.py -q                              → 12 passed in 0.56s
(4) python -m pytest tests/eval/test_validator_framework.py -q                   → 1 error (P-1: ImportError RuleStatus)
(5) git status --short（全部 ?? untracked，无 M/D）+ 冻结文件 mtime 核验
```

### 关键文件定位
- 契约：`docs/Step_8_Coding_Contract_v1.md`
- 审查包：`reviews/Step_8_Independent_Review_Package.md`（登记 D-STEP8-14~23、5 个待裁决问题）
- 生产代码：`src/parsers/{base,xlsx}.py`、`src/facts/{mapper,store,conflict}.py`、`src/compute/unit_registry.py`、`src/rules/{criterion,conclude}.py`、`src/ir/{builder,adapter}.py`
- 冻结验证层：`src/eval/accuracy/{data_structures,validator_framework,binding,tokenizer}.py`
- 测试：`tests/e2e/test_step8_e2e.py`（12 E2E）
- fixture：`tests/eval/fixtures/case_step8_defect_v1/{meta.json,notes.md,inputs/inspection.xlsx}`

### 复现步骤
按 §G 命令逐条执行即可复现。全量回归须排除 `test_validator_framework.py`（P-1 OOS-8）。