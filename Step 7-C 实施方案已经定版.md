*正在请求专家建议，优化解决方案，该过程耗时可能较长，请耐心等待...*

好，Advisor 已确认方案可行。开始按顺序实施。先写 `domain.py`。
好，实施方案已经定版，现在开始严格执行。先调用 Advisor 做一次策略确认，因为这是从设计到编码的关键转换点。
正在请求专家建议...

已获取专家建议

Advisor 确认方案可行，提醒我在写代码前确认 id.py 和 types.py 的精确签名——这是个好建议。我之前已经读过这些文件了，让我快速核对关键 API：

1. `is_valid_fact_id(raw)` — 可以不传 registry/fact_type，此时只做 grammar + scope_class 枚举检查。Validator 里只需要 grammar 检查即可（scope 一致性由 domain_validate 自己单独做）。

2. `parse_fact_id(raw)` — 返回 `Optional[FactID]`，FactID 有 `.scope_class` 属性（string）。这个正是我 scope 一致性检查需要的。

3. `SourceRef` — 已有 `to_dict()` / `from_dict()` / `to_json()`。Evidence 嵌套 SourceRef 序列化时直接调用。

4. `validate_source_ref(sr)` — 已有，domain_validate 里 Evidence 直接复用。

好，开始编码。按照 Advisor 建议的顺序：domain.py → domain_validate.py → test_domain.py → __init__.py。

先写 domain.py。

好，开始写 domain.py。让我再快速确认一下 id.py 和 types.py 里我要用到的 API：

1. `parse_fact_id(raw)` → 返回 `Optional[FactID]`，FactID 有 `.scope_class`, `.instance_key`, `.attribute` 属性
2. `is_valid_fact_id(raw)` → 可以只传 raw，不传 registry/fact_type，此时只做 grammar + scope_class 检查
3. `SourceRef` 有 `.to_dict()`, `.from_dict()`, `.to_json()`

现在开始写 domain.py。

好，domain.py 创建成功。现在写 domain_validate.py。这个文件的核心是：
1. scope 一致性检查表（Domain Object 类型 → 允许的 scope_class 集合）
2. 用 parse_fact_id 取出 scope_class 段
3. 复用 Step 7-B 的 is_valid_fact_id（grammar 检查）和 validate_source_ref
4. Structure 的特殊规则：building → 允许 building scope；其他 type → fact_ids 必须空

让我仔细写这个 validator。

现在写 domain_validate.py。核心逻辑：

1. 用 parse_fact_id 取出 scope_class 段，做 scope 一致性检查
2. 用 is_valid_fact_id 做 grammar 检查（不传 registry/fact_type）
3. Structure 的特殊规则：building → 允许 building scope；其他 type → fact_ids 必须空
4. Evidence 不做 scope 一致性检查（跨 scope 是 Evidence 的职责）
5. 复用 validate_source_ref

scope 一致性表：
- Project → {"project"}
- Structure.building → {"building"}
- Structure(floor/axis/zone/structure_unit) → {}  (必须空)
- Component → {"component"}
- Defect → {"defect"}
- Evidence → None  (不检查)

好，domain_validate.py 创建成功。现在写 test_domain.py。这个文件要覆盖实施方案里的全部 ~49 个用例。

让我仔细对照实施方案里的测试表来写：

### Project (6 用例)
P-1: id="PROJ01", fact_ids=[] → ✅
P-2: id="" → ❌
P-3: fact_ids=["fact:project.PROJ01.name", "fact:project.PROJ01.id"] → ✅
P-4: fact_ids=["fact:component.K001.concrete_strength"] → ❌ scope 不一致
P-5: fact_ids=["fact:defect.C001.width"] → ❌ scope 不一致
P-6: fact_ids=["not_a_fact_id"] → ❌ 非法格式

### Structure (11 用例)
S-1: structure_type="building", fact_ids=[], project_id="PROJ01" → ✅
S-2: structure_type="building", fact_ids=["fact:building.B01.area"], project_id="PROJ01" → ✅
S-3: structure_type="floor", fact_ids=[], project_id="PROJ01", parent_id="B01" → ✅
S-4: structure_type="floor", fact_ids=["fact:building.B01.area"] → ❌
S-5: structure_type=None → ❌ (不传，dataclass 有默认值? 不对 — structure_type 现在是必填的 str 无默认值，所以不传会直接报 dataclass TypeError)
S-6: structure_type="illegal_type" → ❌
S-7: fact_ids=["fact:structure.F1.area"] → ❌
S-8: structure_type="building", fact_ids=["fact:component.K001.id"] → ❌
S-9: structure_type="building", project_id="PROJ01" → ✅
S-10: structure_type="building", 不传 project_id → ❌ (同样是 dataclass 必填字段)
S-11: structure_type="building", project_id="" → ❌

等一下，让我再看 Structure 的 dataclass 定义。我写的是：
```python
@dataclass
class Structure:
    id: str
    structure_type: str
    project_id: str
    parent_id: Optional[str] = None
    fact_ids: list[str] = field(default_factory=list)
```

所以：
- id, structure_type, project_id 都是必填无默认值
- parent_id 可选，默认 None
- fact_ids 有默认值

所以 S-5（structure_type=None）在构造阶段就会报错——dataclass 不允许 None 作为 str 的值除非显式声明 Optional[str]。好，我写的 dataclass structure_type 是 str 不是 Optional[str]，所以传 None 会报 TypeError。那测试里这个用例怎么写？我可以传 structure_type=""（空字符串），validator 会检出。或者我用 pytest.parametrize 把不传的情况包含进去？

实际上我应该把测试分成：
- **构造层面必填检查**：structure_type 和 project_id 不传 → dataclass TypeError（这已经在 Python 层面保证了）
- **validator 层面检查**：structure_type="" → validator 检出；structure_type="illegal" → validator 检出

让我重新调整 S-5：不传 structure_type → 构造失败（TypeError）。这其实是 Python dataclass 的内置类型保障，比 validator 更强。测试可以用 pytest.raises(TypeError) 来覆盖。

同样 S-10：不传 project_id → 构造失败（TypeError）。

好，让我按这个逻辑写测试。

### Component (10 用例)
C-1: defect_ids=[], parent_structure_id="B01" → ✅
C-2: defect_ids=["C001"] → ✅
C-3: defect_ids=["C001", "C002"] → ✅
C-4: defect_ids=[""] → ❌
C-5: parent_structure_id="" → ❌
C-6: 不传 parent_structure_id → TypeError（构造必填）
C-7: fact_ids=["fact:component.K001.concrete_strength"] → ✅
C-8: fact_ids=["fact:defect.C001.width"] → ❌
C-9: fact_ids=["fact:project.PROJ01.name"] → ❌
C-10: id="" → ❌

### Defect (7 用例)
D-1: host_component_id="K001", fact_ids=[] → ✅
D-2: host_component_id="K001", fact_ids=["fact:defect.C001.width", "fact:defect.C001.pattern"] → ✅
D-3: host_component_id="" → ❌
D-4: 不传 host_component_id → TypeError（构造必填）
D-5: id="" → ❌
D-6: fact_ids=["fact:component.K001.id"] → ❌
D-7: fact_ids=["fact:project.PROJ01.name"] → ❌

### Evidence (8 用例)
E-1: raw_source_id="photo-034.jpg", source_refs=[SourceRef(...)], related_fact_ids=[] → ✅
E-2: related_fact_ids=["fact:defect.C001.width", "fact:component.K001.concrete_strength"] → ✅ (跨 scope 合法)
E-3: 不传 raw_source_id → TypeError（构造必填）
E-4: raw_source_id="" → ❌
E-5: related_fact_ids=["not_a_fact_id"] → ❌
E-6: 不传 related_object_ids（可选默认 []） → ✅
E-7: source_refs=[] → ❌
E-8: id="" → ❌

### JSON Roundtrip (7 用例)
J-1: Project
J-2: Structure(building) 含 parent_id=None
J-3: Structure(floor) 含 parent_id、fact_ids=[]
J-4: Component defect_ids=[] 和 defect_ids=["C001"]
J-5: Defect 含 host_component_id
J-6: Evidence 含嵌套 SourceRef
J-7: Evidence 不传 related_object_ids

好，让我按这个写 test_domain.py。

写测试文件。对照实施方案里的测试表严格覆盖。

好，test_domain.py 创建成功。现在需要修改 __init__.py 来导出新模块。让我先读一下当前的 __init__.py 内容。

好，现在修改 __init__.py 来新增 domain.py 的导出。我需要：
1. 新增 from .domain import ...
2. 新增 from .domain_validate import ... （validator 也应该导出方便外部使用）
3. 更新 __all__

注意：domain_validate 里用到了 domain 里的类型（Structure, Project 等），所以 domain 必须在 domain_validate 之前导入。

好，所有文件已创建/修改。现在运行 pytest 来验证：
1. Step 7-B 的 74 个测试全部继续通过
2. Step 7-C 新增的测试全部通过

好，运行 pytest。

128 个测试全部通过！Step 7-B 原有 74 个测试 + Step 7-C 新增 54 个测试 = 128。让我确认一下新增了多少——128 - 74 = 54 个新测试。好。

现在运行 git status 和 git diff --stat 来确认没有越界修改。

好，让我检查一下 git status：
- modified: src/cdm/__init__.py — 正确（我修改了它来新增导出）
- Untracked: src/cdm/domain.py, src/cdm/domain_validate.py, tests/cdm/test_domain.py — 正确（新建文件）
- Untracked: reviews/ — 这是之前就有的
- Untracked: 开发流水线.md — 这是之前就有的
- 没有 docs/*.md 被修改 — 正确
- 没有 types.py / registry.py / id.py / validate.py 被修改 — 正确

完美！没有越界修改。

现在让我再运行一次完整的 pytest 来确认，并且让我把 test_domain.py 的测试数单独跑一下来确认新增了多少。

让我把所有 todo 标记完成。

PowerShell 没有 tail 命令，用 Select-Object -Last 代替。

完美！54 个 Step 7-C 新增测试全部通过。加上 Step 7-B 的 74 个测试，总共 128 个全部通过。

现在让我运行 Advisor 做一次最终审查，因为按照规则，在宣称任务完成前必须调用 Advisor。

现在调用 Advisor 做最终审查，确认完成后输出最终报告。
正在请求专家建议..

已获取专家建议

好，Advisor 确认完成。提到的流水线机械重复示意.md 是之前就有的，不是本次创建的。

现在输出最终 8 节报告。


*正在请求专家建议，优化解决方案，该过程耗时可能较长，请耐心等待...*

Step 7-C 实施完成。以下是最终报告。

---

# 一、实际修改文件

| 操作 | 文件 | 行增量 |
|------|------|--------|
| **新建** | `src/cdm/domain.py` | ~220 行 — 5 个 Domain Object dataclass + STRUCTURE_TYPES 枚举 + to_dict/from_dict/to_json |
| **新建** | `src/cdm/domain_validate.py` | ~250 行 — scope 一致性表 + 5 个 validate_* 函数 + 3 个内部 helper |
| **新建** | `tests/cdm/test_domain.py` | ~560 行 — 54 测试用例（Project 6 + Structure 13 + Component 10 + Defect 7 + Evidence 9 + JSON 7 + Enum 2） |
| **修改** | `src/cdm/__init__.py` | +37 / -5 行 — 新增 domain.py / domain_validate.py 导出 |

**未修改**（正确保持不动）：
- `src/cdm/types.py` — Step 7-B 纯净
- `src/cdm/registry.py` — scope_class 集合不变
- `src/cdm/id.py` — 三段式 grammar 纯净
- `src/cdm/validate.py` — Step 7-B validator 纯净
- `tests/cdm/test_types.py` — Step 7-B 测试纯净
- `docs/*.md` — 设计文档零改动

---

# 二、Domain Object 实现

## STRUCTURE_TYPES 枚举
```python
STRUCTURE_TYPES: frozenset[str] = frozenset({
    "building", "floor", "axis", "zone", "structure_unit",
})
```
**注意**：`structure` **不在** `SCOPE_CLASSES` 里。Scope 枚举（registry.py）和 Structure 实例类型（domain.py）是两个独立概念。

## 5 个 Domain Object 结构

| 对象 | 必填字段 | Optional 字段 | 类型 |
|------|---------|--------------|------|
| Project | `id`, `fact_ids` | `metadata` | dataclass |
| Structure | `id`, `structure_type`, `project_id`, `fact_ids` | `parent_id` | dataclass |
| Component | `id`, `parent_structure_id`, `defect_ids`, `fact_ids` | — | dataclass |
| Defect | `id`, `host_component_id`, `fact_ids` | — | dataclass |
| Evidence | `id`, `raw_source_id`, `source_refs`, `related_fact_ids` | `related_object_ids` | dataclass |

**关键设计点**：
1. **Structure 统一替代 Building**：没有独立 Building class。`structure_type="building"` 是 Structure 的一种实例类型。
2. **structure_type 是 str 不是 Optional[str]**：必填，避免 validator 无法判定 scope 的歧义状态。
3. **Component.defect_ids 默认 []**：允许无缺陷 Component（实现了原方案修正 1）。
4. **Defect 是弱实体**：`host_component_id` 必填，弱实体锚点。
5. **Evidence 可跨 scope**：`related_fact_ids` 不限制 scope 一致性（跨 scope 关联是 Evidence 的职责）。
6. **序列化**：每个类都有 `to_dict()` / `from_dict()` / `to_json()`，SourceRef 复用 Step 7-B 已有实现。

---

# 三、Validator 实现

## scope 一致性表（`STRUCTURE_ALLOWED_SCOPES`）
```python
STRUCTURE_ALLOWED_SCOPES = {
    "building":         frozenset({"building"}),
    "floor":            frozenset(),     # 本轮必须空
    "axis":             frozenset(),
    "zone":             frozenset(),
    "structure_unit":   frozenset(),
}
```

各对象 scope 规则：
| 对象 | 允许 scope |
|------|-----------|
| Project | `{"project"}` |
| Structure(building) | `{"building"}` |
| Structure(其他) | `{}` （必须空） |
| Component | `{"component"}` |
| Defect | `{"defect"}` |
| Evidence | **不检查 scope**（跨 scope OK） |

## Validator 职责边界

| 检查类别 | 本轮做 / 不做 | 说明 |
|---------|-------------|------|
| `id` 非空 | ✅ 做 | 字符串非空非空白 |
| `fact_ids[]` grammar | ✅ 做 | `is_valid_fact_id(raw)` 不传 registry |
| `fact_ids[]` scope 一致性 | ✅ 做 | `parse_fact_id(fid).scope_class` 比对 |
| Structure.structure_type 枚举 | ✅ 做 | 非空 + ∈ STRUCTURE_TYPES |
| Structure.project_id 非空 | ✅ 做 | 必须表达 Project → Structure 关系 |
| Component.parent_structure_id | ✅ 做 | 必填 |
| Component.defect_ids 允许空 | ✅ 做 | list 允许空，但元素（若存在）必须非空 |
| Defect.host_component_id | ✅ 做 | 弱实体锚点 |
| Evidence.raw_source_id 非空 | ✅ 做 | 追溯链 Evidence → RawSource |
| Evidence.source_refs 非空 | ✅ 做 | 至少一个定位；内部复用 `validate_source_ref` |
| Fact Store 存在性 | ❌ 不做 | 留给 Pipeline / Store |
| ID 引用指向真实对象 | ❌ 不做 | 留给 Pipeline 组装阶段 |
| Fact.status 是否 filled | ❌ 不做 | 留给 Fact Store |

---

# 四、测试结果

## Step 7-C 新增测试：54 / 54 通过 ✅

| 测试类 | 测试数 | 覆盖 |
|--------|--------|------|
| TestStructureTypesEnum | 2 | 枚举完整性 + `structure` 不在 SCOPE_CLASSES |
| TestProject | 6 | 最小合法、空 id、scope 正确、scope 不一致 (component/defect)、非法格式 |
| TestStructure | 13 | building 空 facts、building + building facts、floor 空 facts、floor + facts→非法、空 structure_type、非法枚举、fact:structure.*→非法、building+component scope→非法、空 project_id、空 id、parent_id=None、parent_id="  " |
| TestComponent | 10 | defect_ids=[] 合法（关键！）、单缺陷、多缺陷、空元素、空 parent、scope 正确、defect scope→非法、project scope→非法、空 id、非法格式 |
| TestDefect | 7 | 最小合法、scope 正确、空 host、component scope→非法、project scope→非法、空 id、非法格式 |
| TestEvidence | 9 | 最小合法、**跨 scope 合法**、空 raw_source_id、非法格式、related_object_ids 默认空、source_refs=[]→非法、空 source_ref 元素、空 id、空 related_object_ids 元素 |
| TestJSONRoundtrip | 7 | Project、Structure(building)、Structure(floor)、Component 两种 defect_ids 变体、Defect、Evidence 含嵌套 SourceRef、Evidence related_object_ids 缺省 |

---

# 五、Step 7-B 回归结果

## Step 7-B 原有 74 个测试：74 / 74 通过 ✅

```
tests/cdm/test_types.py  — 74 tests
test_conflict_candidate_*
test_conflict_*
test_source_ref_*
test_raw_source_*
test_fact_*
test_fact_json_*
test_defect_facts_*
... all PASSED
```

**没有修改** `types.py` / `registry.py` / `id.py` / `validate.py` / `test_types.py`。Step 7-B 核心代码 100% 纯净保留。

---

# 六、与 Step 7-C 实施方案的一致性

| 定版要求 | 实现状态 |
|---------|---------|
| 5 个 Domain Object：Project / Structure / Component / Defect / Evidence | ✅ |
| 不创建独立 Building class | ✅ 只有 Structure |
| structure_type ∈ {building, floor, axis, zone, structure_unit} | ✅ |
| fact:structure.* 永远非法 | ✅ SCOPE_CLASSES 无 structure；test_s7 覆盖 |
| Project 不增加 structure_ids[] | ✅ Project 只有 id / fact_ids / metadata |
| Structure.project_id 必填，parent_id 可选 | ✅ |
| Component.defect_ids **允许为空** | ✅ test_c1 明确验证 |
| Component.parent_structure_id 必填 | ✅ |
| Defect.host_component_id 必填（弱实体） | ✅ |
| crack / corrosion / spall 是 Defect 实例，不是新 class | ✅ |
| Evidence.raw_source_id 必填 | ✅ |
| Evidence.source_refs 非空 | ✅ |
| Evidence.related_object_ids 可选 | ✅ |
| Evidence 可跨 scope 关联 fact_ids | ✅ test_e2 验证 |
| Domain Object 不存 Fact value | ✅ dataclass 无 value 字段 |
| scope 一致性本地验证（不依赖 Store） | ✅ |
| 不验证 Store 存在性 | ✅ |
| 不修改 02/03/04/05/07 设计文档 | ✅ git diff 确认 |
| 不修改 Step 7-B 核心代码 | ✅ git diff 确认 |
| 不实现 InspectionItem / Measurement / Criterion / Evaluation / Conclusion | ✅ |
| 不实现 Report IR / Parser / DOCX / PDF | ✅ |
| 不实现 LLM / Skill / Agent | ✅ |
| 不自动 commit | ✅ 执行 |

---

# 七、Git 状态

```
On branch master
HEAD: 471d193 (tag: step-7b-complete)

Modified:     src/cdm/__init__.py   (+37 / -5)
Untracked:    src/cdm/domain.py
              src/cdm/domain_validate.py
              tests/cdm/test_domain.py
              reviews/              (pre-existing)
              开发流水线.md           (pre-existing)
```

**检查结论**：
- ✅ 没有 docs/*.md 改动
- ✅ 没有 src/cdm/types.py / registry.py / id.py / validate.py 改动
- ✅ 没有创建未来模块（Parser / Validator / DOCX / LLM ...）
- ✅ 没有创建 Building class
- ✅ 没有新增 scope_class（structure 不在 SCOPE_CLASSES）
- ✅ 没有 commit

---

# 八、发现的问题

| # | 问题 | 严重度 | 说明 | 处理 |
|---|------|--------|------|------|
| 1 | 无阻塞问题 | — | — | — |

**次要观察（不阻塞）**：
- `test_s4_floor_carrying_facts_fails` 的断言用了嵌套 `any() ... or ... any()` 逻辑，可读性略差但语义正确。future cleanup 时可以简化为直接检查具体错误文本模式。
- 工作区有一些 pre-existing untracked 文件（`reviews/`、`开发流水线.md`），与本次 Step 7-C 无关，已确认不是本次创建的。

---

Step 7-C 完成。全量 **128 / 128 测试通过**，零越界修改，零设计文档改动。等待下一条指令。
