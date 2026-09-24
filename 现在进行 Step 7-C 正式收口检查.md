并行执行全部验收动作：
用户要求做最后的验收检查。我需要并行执行：读取关键文件 + 运行 pytest + 检查 git 状态。让我一次把所有工具调用都发出。

读取完成，代码全部可见。现在重新运行 pytest 和 git 命令：

全部信息已收集完毕。让我逐一核查用户的 6 个验收项：

## 1. None/Type Hint 检查

用户指出：`structure_type: str` 这个 type hint 在 Python dataclass 中**不会自动拒绝 None**。这是正确的 —— dataclass 的 type hint 没有运行时强制。`Structure(structure_type=None)` 会成功构造，validator `validate_struct()` 才会拒绝它。

看 validator 代码 line 168: `if not struct.structure_type or not struct.structure_type.strip():` — None 进入这里会触发 `not None` = True，会报 non-empty 错误。✅

测试 S-5 用的是空字符串，没有直接测 None 构造。不过用户说的是"如果当前测试已经覆盖 validator 对 None 的拒绝"。让我快速验证一下 None 的行为...

实际上测试没有直接构造 Structure(structure_type=None)，但 validator 逻辑 `if not struct.structure_type` 对 None 和 "" 都触发。所以 None 构造 → validator 拒绝 → 有错误。测试 S-5 虽然用 "" 但覆盖了同一条代码路径。

## 2. 真实代码检查

通读了全部 4 个文件（共 ~1130 行）：

### ✅ 没有 Building class
`class Building` 在 domain.py 中不存在。Search 确认。

### ✅ 没有新增 scope_class
domain.py 没有触及 registry.py。test_domain.py line 70-71 的断言验证了 `"structure" not in SCOPE_CLASSES`。

### ✅ 没有修改 Step 7-B 核心代码
git diff 显示只有 `src/cdm/__init__.py` 被修改，其他 3 个源文件都是 **untracked**（新建）。`types.py` / `registry.py` / `id.py` / `validate.py` 都在 git status 里没有任何标记 — 确认零改动。

### ✅ Domain Object 没有 value 字段
逐个核对 5 个 dataclass：
- Project: id, fact_ids, metadata — 无 value
- Structure: id, structure_type, project_id, parent_id, fact_ids — 无 value
- Component: id, parent_structure_id, defect_ids, fact_ids — 无 value
- Defect: id, host_component_id, fact_ids — 无 value
- Evidence: id, raw_source_id, source_refs, related_fact_ids, related_object_ids — 无 value

全部符合 —— 值从 Fact 层读取。

### ✅ Structure 的 scope 规则正确
domain_validate.py 的 STRUCTURE_ALLOWED_SCOPES:
- "building" → frozenset({"building"})
- floor/axis/zone/structure_unit → frozenset() (必须空)
对应 test_s4（floor + fact_ids → INVALID）、test_s2（building + fact:building.* → VALID）、test_s8（building + component → INVALID）全部通过。

### ✅ Component.defect_ids 可以为空
test_c1 显式验证 defect_ids=[] 的 Component 通过 validate_component()。validator 代码 line 237 的 `_check_list_elements_non_empty` 只检查 list **元素**非空，不检查 list 本身非空。

### ✅ Defect.host_component_id 必填
domain.py line 189: `host_component_id: str` 没有 Optional、没有 default。
validator line 269: 空值直接报错。
test_d3 / test_d4 覆盖。

### ✅ Evidence.source_refs 非空
domain.py line 233: `source_refs: list[SourceRef] = field(default_factory=list)` — 默认空列表
validator line 314: `if len(ev.source_refs) == 0` 直接报错。
test_e6 覆盖。

### ✅ Evidence.related_fact_ids 可以跨 scope
domain_validate.py line 324-326: `_check_fact_ids_scope(..., allowed_scopes=None)` — None 表示**跳过 scope 检查**。只检查 grammar。
test_e2 显式构造跨 scope（defect + component + project 三个 scope 的 Facts）→ validate_evidence 返回 []。

### ✅ Store 存在性检查没有提前实现
通读 domain_validate.py 333 行，没有任何地方检查"某个 ID 引用的对象是否真实存在"。docstring line 14-21 明确列出了推迟的 7 类检查。

## 3. 测试结果
- 全量 tests/: **128 passed in 0.25s** ✅
- test_domain.py: **54 passed in 0.12s** ✅
- test_types.py (Step 7-B 回归): **74 passed** (在 128 里面)

## 4. Git 状态
```
 M src/cdm/__init__.py        — 唯一修改
?? reviews/                    — pre-existing, 排除
?? src/cdm/domain.py           — 纳入 Step 7-C
?? src/cdm/domain_validate.py  — 纳入 Step 7-C
?? tests/cdm/test_domain.py    — 纳入 Step 7-C
?? 开发流水线.md                — pre-existing, 排除
```
Diff stat: `__init__.py | 42 +++++++++++++++++++++++++++++++++++++-----` — +37/-5，精确对应本次导出增量。没有意外文件。

## 5. commit 前准备
准备纳入 commit 的 4 个文件：
1. `src/cdm/domain.py` — 新建
2. `src/cdm/domain_validate.py` — 新建
3. `src/cdm/__init__.py` — 修改
4. `tests/cdm/test_domain.py` — 新建

排除的 pre-existing 过程文件：
- `reviews/` 目录
- `开发流水线.md`
- 其他一切

## 6. 是否可以 commit

可以。理由：
- 128/128 测试通过（74 Step 7-B + 54 Step 7-C）
- 设计文档零改动
- Step 7-B 核心代码零改动
- scope_class 集合零扩张（structure 始终不在）
- 无 Building class
- 无 value 字段
- Store 检查零提前实现
- Git diff 精确落在 4 个文件
- 准备纳入的文件集与设计意图完全一致

唯一的"小遗憾"是用户问的 None 行为测试——让我快速验证一下：

实际上测试里没有显式构造 `Structure(structure_type=None)` 的 case，但 validator 代码逻辑 `if not struct.structure_type` 对 None 触发 True（`not None == True`），所以 None 也会被判为 non-empty 错误。这条路径被 S-5 的空字符串测试覆盖了（因为两个值走的是同一个 validator 分支）。如果需要更严格可以加一条，但 validator 行为是对的。

好，输出最终报告。
# 一、实现验收

逐行通读了 4 个文件（~1130 行），对照 Step 7-C 定版方案的 20+ 条要求逐项核对：

| 验收项 | 状态 | 代码位置 |
|--------|------|---------|
| 无 Building class | ✅ | domain.py 全文件不存在 `class Building` |
| 无新增 scope_class | ✅ | test_domain.py:70 断言 `"structure" not in SCOPE_CLASSES`；registry.py 零改动 |
| Step 7-B 核心代码零改动 | ✅ | git status 确认 types.py / registry.py / id.py / validate.py / test_types.py 均无任何标记 |
| Domain Object 无 value 字段 | ✅ | 5 个 dataclass 全部只有 id / fact_ids / 关系字段；值从 L2 Fact 层读取 |
| Structure scope 规则正确 | ✅ | domain_validate.py:47-54 STRUCTURE_ALLOWED_SCOPES — building→{"building"}，其余→必须空 |
| Component.defect_ids 允许空 | ✅ | domain_validate.py:237 只检查元素非空，不检查 list 非空；test_c1 显式验证 |
| Defect.host_component_id 必填 | ✅ | domain.py:189 `host_component_id: str` 无 Optional/默认；validator:269 空值报错 |
| Evidence.source_refs 非空 | ✅ | validator:314 `if len(ev.source_refs) == 0` 报错 |
| Evidence.related_fact_ids 跨 scope | ✅ | validator:325 `allowed_scopes=None` 跳过 scope 检查；test_e2 显式验证三 scope 混用 |
| Store 存在性检查零提前实现 | ✅ | domain_validate.py:14-21 列出 7 类推迟检查，全文无 Store 调用 |

---

# 二、None / Type Hint 检查

**事实陈述（不虚构 constructor 自动拒绝）**：

Python dataclass 的 type hint `structure_type: str` **没有运行时强制**。以下代码在 Python 中完全合法：
```python
Structure(id="X", structure_type=None, project_id="PROJ01")  # 构造成功
```

**Validator 行为（真正的防线）**：
```python
# domain_validate.py line 168
if not struct.structure_type or not struct.structure_type.strip():
    errs.append("Structure.structure_type must be non-empty ...")
```
`not None` 为 `True`，因此 `validate_struct(Structure(structure_type=None))` 会返回错误。

**测试覆盖现状**：S-5 用空字符串 `structure_type=""` 覆盖了同一 validator 分支。None 构造会走完全相同的错误路径。不需要额外修复代码，但后续可追加显式 None 测试用例增强健壮性。

**结论**：Validator 行为正确。constructor 不拒绝 None 是 Python dataclass 的固有特性，不是 bug。

---

# 三、测试结果

## 全量测试
```
python -m pytest tests/ -v
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
128 passed in 0.25s
```
- Step 7-B 回归：**74 / 74 通过**（test_types.py 全文件）
- Step 7-C 新增：**54 / 54 通过**（test_domain.py 全文件）

## Step 7-C 单独运行
```
python -m pytest tests/cdm/test_domain.py -v
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
54 passed in 0.12s
```
按类分布：
| 类 | 用例 | 状态 |
|----|------|------|
| TestStructureTypesEnum | 2 | ✅ |
| TestProject | 6 | ✅ |
| TestStructure | 13 | ✅ |
| TestComponent | 10 | ✅ |
| TestDefect | 7 | ✅ |
| TestEvidence | 9 | ✅ |
| TestJSONRoundtrip | 7 | ✅ |

---

# 四、Git 状态

```
 M src/cdm/__init__.py
?? reviews/
?? src/cdm/domain.py
?? src/cdm/domain_validate.py
?? tests/cdm/test_domain.py
?? 开发流水线.md
```

**`__init__.py` diff — 精确对应 Step 7-C 导出增量**：
```
+37 / -5 行
# Registry (Step 7-B)     — 保留
# ID (Step 7-B)           — 保留
# Types (Step 7-B)        — 保留
# Domain Objects (Step 7-C)   +5 个新导出
# Domain validators (Step 7-C) +5 个新导出
```

**零越界修改确认**：
- docs/*.md — 零改动
- src/cdm/types.py — 零改动
- src/cdm/registry.py — 零改动
- src/cdm/id.py — 零改动
- src/cdm/validate.py — 零改动
- tests/cdm/test_types.py — 零改动

---

# 五、准备 commit 的文件

**纳入 Step 7-C commit（4 个文件）**：

| 操作 | 文件 | 说明 |
|------|------|------|
| 新建 | `src/cdm/domain.py` | 5 个 Domain Object + STRUCTURE_TYPES |
| 新建 | `src/cdm/domain_validate.py` | scope 一致性表 + 5 个 validate_* + 3 个 helper |
| 新建 | `tests/cdm/test_domain.py` | 54 用例 |
| 修改 | `src/cdm/__init__.py` | +37/-5 导出增量 |

**明确排除（pre-existing 过程文件，与 Step 7-C 无关）**：
- `reviews/` 目录
- `开发流水线.md`

---

# 六、是否可以正式 commit

**可以。** 理由清单：

| 维度 | 判定 |
|------|------|
| 测试正确性 | ✅ 128/128（74 + 54）全绿 |
| Step 7-B 回归 | ✅ 零回归 |
| 设计文档合规 | ✅ 零改动 |
| Step 7-B 核心代码合规 | ✅ 零改动 |
| scope_class 冻结集 | ✅ 零扩张 |
| Domain Model 设计 | ✅ Structure 统一替代 Building；无 value 字段；弱实体 Defect；Evidence 跨 scope；Component.defect_ids 允许空 |
| Validator 边界 | ✅ 零提前 Store 检查；scope 一致性本地可判定 |
| Git diff 精确性 | ✅ 落在 4 个计划内文件 |
| 准备纳入文件集 | ✅ 与设计意图完全一致；pre-existing 文件已排除 |

**Commit 建议**（用户下一条指令可决定是否执行）：
```
git add src/cdm/domain.py src/cdm/domain_validate.py src/cdm/__init__.py tests/cdm/test_domain.py
git commit -m "Step 7-C: L3 Domain Objects (Project/Structure/Component/Defect/Evidence)
- 5 dataclasses with STRUCTURE_TYPES enum
- Scope-consistency validator (local, no Store dependency)
- 54 new tests; 74 Step 7-B tests untouched → 128 total pass
- No design doc changes, no Step 7-B core code changes"
git tag step-7c-complete
```

完成，等待下一条指令。
