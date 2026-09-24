定点修复。先修复 from_dir 的 inputs/ 检查，然后运行所有命令拿真实结果。

直接定点修复 from_dir：在检查目录存在性之后、检查 required files 之前，加入 inputs/ 目录存在性检查。然后删掉 validate_case 里的 inputs 检查（因为 from_dir 已经提前保证了）。

现在把 validate_case 里的 inputs 检查也删掉（from_dir 已经在加载期保证了，validator 不该重复检查）。
from_dir 已修复：inputs/ 目录检查在 JSON 解析之前直接抛 FileNotFoundError。

现在还需要把 validate_case 里的 inputs 检查去掉，因为 from_dir 已经在加载期保证了。让我看一下 validator.py 里 validate_case 的 T-1 规则。

好，现在把 validate_case 里的 T-1 inputs/ 检查删掉——from_dir 已经在加载期保证了，validator 不该重复检查。

现在更新 test_loader.py 里的对应测试——from_dir 直接抛异常了，不能再走 validate 路径。同时先跑一轮看哪些测试被破坏了。
好，from_dir 和 validate_case 都修了。现在需要更新 test_loader.py 里的对应测试——之前的 `test_case_missing_inputs_dir_fails_validation` 现在应该变成 `test_from_dir_raises_on_missing_inputs_dir`，因为错误前移到了 from_dir 层面。

让我也运行所有测试拿真实结果，包括 collect-only。然后输出验收报告。

两个测试失败都是预期内的（修复前移导致的边界变化）。修复测试：第一个加 inputs/，第二个改成测试 from_dir 直接抛异常。
好，两个测试失败了，都是预期内的：
1. `test_from_dir_raises_keyerror_on_missing_required_key` — 这个测试之前只创建了 meta.json 但没创建 inputs/ 目录，现在 from_dir 先检查 inputs/ 就失败了。我需要给它加上 inputs/.gitkeep
2. `test_case_missing_inputs_dir_fails_validation` — 这个测试现在完全不成立了，因为 from_dir 直接抛 FileNotFoundError 了。我需要把它改成 `test_from_dir_raises_on_missing_inputs_dir`

让我也看看 fixture case_missing_inputs_dir——它确实没有 inputs/ 目录，现在应该直接抛异常。这个 fixture 不需要保留（因为 inputs/ 必须存在了），但我需要处理测试。

让我先修复测试文件，然后重新跑。

好，我需要修改 test_loader.py 里两个地方：

1. `test_from_dir_raises_keyerror_on_missing_required_key` (L67-87) — 需要给 tmp_path 里的 bad_case 创建 inputs/ 目录，否则 from_dir 先因为 inputs/ 缺失而抛 FileNotFoundError 而不是 KeyError

2. `test_case_missing_inputs_dir_fails_validation` (L112-116) — 这个 fixture 没有 inputs/ 目录，现在 from_dir 会直接抛 FileNotFoundError。应该改成 `test_from_dir_raises_on_missing_inputs_dir`

3. `test_from_dir_raises_on_missing_required_file` (L53-64) — 这个也需要加 inputs/ 目录

让我一起改。

删掉旧的 `test_case_missing_inputs_dir_fails_validation`（已被替代）。
好，现在还要删掉旧的 `test_case_missing_inputs_dir_fails_validation` 函数（已经被新的 test_from_dir_raises_on_missing_inputs_dir 替代）。让我看看那部分代码在哪里。

好。现在跑完整 pytest 命令矩阵。
好，现在先跑一遍 pytest -q 看看修复后的结果，然后跑 collect-only 和全量。

好，collect-only 收集到 79 个用例。现在跑 -q 看结果，然后跑全量。

79 passed / 207 passed / 零修改 cdm。现在跑 10 项专项检查。
全部 79 passed。现在跑那 10 项专项检查，拿真实结果。

PowerShell 在 Python inline 里反引号也有转义问题。让我把脚本写成一个临时 .py 文件然后执行。

现在跑这个临时验证脚本，然后删掉它，再跑 collect-only 确认新的总数（因为我加了这个脚本但它不是 pytest test）。

10 项全部 PASS。跑最终结果。
10 项全部 PASS。现在跑最终的 pytest -q 和 git status。

好，全部结果拿到了。现在输出最终验收报告。

让我确认一下测试数量：
- test_loader.py：之前 collect-only 列了 13 个函数。让我数一下：
  test_load_gold_case_success
  test_load_silver_case_success
  test_load_bronze_case_success
  test_from_dir_raises_on_missing_directory
  test_from_dir_raises_on_missing_required_file
  test_from_dir_raises_keyerror_on_missing_required_key
  test_from_dir_raises_on_missing_inputs_dir (新增)
  test_case_gold_minimal_validates_cleanly
  test_case_silver_minimal_validates_cleanly
  test_case_bronze_minimal_validates_cleanly
  test_case_bronze_facts_nonempty_fails_validation
  test_case_g3_not_in_g2_fails_validation
  test_case_g2_g3_duplicate_ids_fails_validation
  
  对，13 个（删掉了 test_case_missing_inputs_dir_fails_validation，新增了 test_from_dir_raises_on_missing_inputs_dir）

- test_types.py：collect-only 列了 18 个
- test_validator.py：collect-only 列了 48 个（TestValidateMetaPass: 3, TestValidateGroundTruthFactPass: 3, TestValidateGroundTruthFactListPass: 2, TestValidateGroundTruthStructurePass: 2, TestValidateGroundTruthConclusionPass: 2, TestValidateKnownIssuePass: 2, TestValidateExpectedIssuePass: 2, TestValidateMetaFail: 3, TestValidateGroundTruthFactFail: 6, TestValidateGroundTruthFactListFail: 2, TestValidateKnownIssueFail: 5, TestValidateExpectedIssueFail: 3, TestValidateGroundTruthConclusionFail: 3, TestValidateCase: 10 → 总共 3+3+2+2+2+2+2+3+6+2+5+3+3+10 = 48）

13 + 18 + 48 = 79 ✓

- 全量 79 + 128（cdm 原有）= 207 ✓

好，数据全部一致。
---

# Step 7-D 最终实施验收报告

## A. 实际修改的文件

| 文件 | 操作 | 修改内容 |
| ---- | ---- | ---- |
| `src/eval/types.py` | 修改 | `from_dir()` 新增 inputs/ 目录存在性检查，直接抛 `FileNotFoundError`（在任何 JSON 解析之前） |
| `src/eval/validator.py` | 修改 | `validate_case()` 删除 T-1 inputs/ 检查（前移到 from_dir）；规则编号重新整理 |
| `tests/eval/test_loader.py` | 修改 | `test_from_dir_raises_on_missing_required_file` 和 `test_from_dir_raises_keyerror_on_missing_required_key` 增加 inputs/ 目录创建（满足新边界）；**新增** `test_from_dir_raises_on_missing_inputs_dir`（直接测 from_dir 抛异常）；**删除** 旧的 `test_case_missing_inputs_dir_fails_validation` |

**新增未变**：`src/eval/__init__.py`、`src/eval/enums.py`、7 个 fixture 目录。

**零修改**：`src/cdm/` 任何文件、`tests/cdm/` 任何文件、任何设计 Markdown。

## B. from_dir 修复前后行为

| 场景 | 修复前 | 修复后 |
| ---- | ---- | ---- |
| `path/inputs/` 不存在 | from_dir 成功返回对象；validate_case 延迟报错 | **from_dir 直接抛 `FileNotFoundError`**（在所有 JSON 读取之前） |
| `path/ground_truth/facts.json` 不存在 | from_dir 抛 FileNotFoundError | 不变（仍然直接抛） |
| meta.json 缺 required key | from_dict 抛 KeyError | 不变 |
| 测试矩阵 L4（inputs 缺失） | 写在 validate_case 里测 | 前移到 from_dir 层面直接测抛异常 |

## C. 10 项专项检查结果

| # | 检查 | 结果 |
| ---- | ---- | ---- |
| 1 | from_dir case_gold_minimal → 成功 | **PASS** |
| 2 | from_dir case_silver_minimal → 成功 | **PASS** |
| 3 | from_dir case_bronze_minimal → 成功，facts=[] | **PASS** |
| 4 | from_dir case_missing_inputs_dir → **FileNotFoundError** | **PASS** |
| 5 | 缺 ground_truth/facts.json → **FileNotFoundError** | **PASS** |
| 6 | meta.json 缺 case_id → **KeyError** | **PASS** |
| 7 | `to_dict()` **不含 case_dir** | **PASS** |
| 8 | `GroundTruthFact.value=None` → validator error | **PASS** |
| 9 | bronze + 非空 facts → validate_case error | **PASS** |
| 10 | G3 fact_id 不在 G2 → validate_case error | **PASS** |

## D. pytest tests/eval/ 实际结果

```
79 passed in 0.20s
```

## E. pytest tests/ 全量结果

```
207 passed in 0.50s
```

79（Step 7-D）+ 128（Step 7-B 74 + Step 7-C 54）= 零回归。

## F. pytest collect-only 实际用例数量

```
79 tests collected in 0.13s
```

| 测试文件 | pytest 收集用例数 |
| ---- | ---- |
| test_loader.py | **13** |
| test_types.py | **18** |
| test_validator.py | **48** |
| **Step 7-D 合计** | **79** |

（无 parametrize——所有用例都是显式 `def test_xxx` 函数或 `class TestXxx` 方法，函数数量 = 收集用例数量。）

全量收集：207 tests（79 + 128）。

## G. git status --short

```
?? reviews/
?? src/eval/
?? tests/eval/
?? *.md   （之前就存在，非 Step 7-D 产物）
```

所有 Step 7-D 新增文件均为 untracked。**没有修改任何已有文件**。

## H. 是否还有契约冲突

**零契约冲突。**

- from_dir / validate_case 边界已按 v4 Coding Contract 调整到位
- inputs/ 目录检查从 validate_case 前移到 from_dir，行为与 v4 完全一致
- Step 7-B / Step 7-C 接口零冲突
- 10 项专项检查全部 PASS
- pytest 全量 207 passed

---

**Step 7-D 实施完成。等待验收。**
