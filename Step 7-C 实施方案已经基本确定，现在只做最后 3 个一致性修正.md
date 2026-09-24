用户要求最后3个一致性修正，我先检查一下 registry.py 里 Structure 的实现类型枚举，以及是否还需要确认 domain.py / validate.py / test_domain.py 三个文件。不需要 ExperienceRecall，因为这是纯方案修订。现在直接输出最终方案。
好，三处修正已全部吸收。以下是最终定版。

---

# 一、最终 Step 7-C 范围

## 本轮实现的 5 个 Domain Object（L3 层）

| # | 对象 | 设计文档来源 | 核心角色 |
|---|------|-------------|---------|
| 1 | **Project** | §18 | 顶层容器 |
| 2 | **Structure** | §20 | 统一 Building / Floor / Axis / Zone / StructureUnit 为同一个 class，用 `structure_type` 必填枚举区分 |
| 3 | **Component** | §21 | 工程构件，1:N Defect |
| 4 | **Defect** | §17 (v0.3.1) | 弱实体，必须 `host_component_id`；crack / corrosion / spall 都是 Defect 实例 |
| 5 | **Evidence** | §28 | 证据载体，追溯链 Evidence → RawSource |

## scope_class 合法性（维持 Step 7-B 冻结集合，**不扩大**）

```
project | building | component | defect | point | sample | material
```

- `fact:building.*` 合法 → 挂在 Structure(type="building")
- `fact:structure.*` **永远非法**（不在 scope_class 里）
- Structure(type="floor"/"axis"/"zone"/"structure_unit") 本轮 `fact_ids` **必须为空**

## 本轮严格禁止
- ❌ 独立 Building class
- ❌ 扩大 scope_class
- ❌ 修改 registry.py / id.py / types.py / validate.py / 任何 docs
- ❌ InspectionItem / Measurement / Criterion / Evaluation / Conclusion / Report IR / Parser / DOCX / PDF / LLM / Skill / Agent

---

# 二、最终字段定义

## 字段分类
- **A 类**：03_CANONICAL_DATA_MODEL.md 已明确要求
- **B 类**：L3 实现层关系字段，未被 03 冻结

## STRUCTURE_TYPES 枚举（本轮新增到 domain.py / domain_validate.py）

```python
STRUCTURE_TYPES: frozenset[str] = frozenset({
    "building",
    "floor",
    "axis",
    "zone",
    "structure_unit",
})
```

---

### 1. Project

| 字段 | 类型 | 分类 | 必填 | 允许的 fact_ids scope |
|------|------|------|------|----------------------|
| `id` | str | A | ✅ | — |
| `fact_ids` | list[str] | A | ✅ | 仅 `project` |
| `metadata` | dict | B | ❌ | — |

---

### 2. Structure（统一实现）

| 字段 | 类型 | 分类 | 必填 | 说明 |
|------|------|------|------|------|
| `id` | str | A | ✅ | 稳定领域标识 |
| `fact_ids` | list[str] | A | ✅ | 见下表 scope 规则 |
| `structure_type` | **str**（不再 Optional） | B | ✅ | `building` / `floor` / `axis` / `zone` / `structure_unit`；**禁止 None / 空 / 非法枚举** |
| `project_id` | str | B | ✅ | 顶层 Structure 必须具有；Project 不存反向 structure_ids[] |
| `parent_id` | Optional[str] | B | ❌ | Structure → Structure 层级；顶层可为 None |

**Structure 的 fact_ids scope 规则**（structure_type 必填后才能判定）：

| structure_type | 允许的 scope_class | 说明 |
|----------------|-------------------|------|
| `building` | 仅 `building` | 真实事实可以挂 |
| `floor` / `axis` / `zone` / `structure_unit` | **必须为空列表** | 本轮无对应 scope，暂不描述 |

---

### 3. Component

| 字段 | 类型 | 分类 | 必填 | 允许的 fact_ids scope |
|------|------|------|------|----------------------|
| `id` | str | A | ✅ | — |
| `fact_ids` | list[str] | A | ✅ | 仅 `component` |
| `parent_structure_id` | str | B | ✅ | 所属 Structure 的 id |
| `defect_ids` | list[str] | B | ✅（**允许空**） | 无缺陷时 `[]` 合法 |

---

### 4. Defect（弱实体）

| 字段 | 类型 | 分类 | 必填 | 允许的 fact_ids scope |
|------|------|------|------|----------------------|
| `id` | str | A | ✅ | — |
| `fact_ids` | list[str] | A | ✅ | 仅 `defect` |
| `host_component_id` | str | A | ✅ | 弱实体锚点 |

crack / corrosion / spall 通过 Defect 的 `fact_ids`（如 `fact:defect.C001.pattern`）区分实例类型，不新增 class。

---

### 5. Evidence

| 字段 | 类型 | 分类 | 必填 | 说明 |
|------|------|------|------|------|
| `id` | str | A | ✅ | — |
| `raw_source_id` | str | A | ✅ | 追溯到 L1 RawSource |
| `source_refs` | list[SourceRef] | A | ✅ | 定位（图片区域 / 文档页码 / 表格单元格） |
| `related_fact_ids` | list[str] | A | ✅（允许空） | 跨 scope 关联，本轮**不限制 scope** |
| `related_object_ids` | list[str] | B | ❌（可选） | 直接关联 Domain Object（Defect / Component） |

---

# 三、最终关系

```
Project("PROJ01")
  │
  ├── Structure("B01", structure_type="building", project_id="PROJ01", parent_id=None)
  │     │     fact_ids → ["fact:building.B01.id", "fact:building.B01.area", ...]
  │     │
  │     ├── Structure("F1", structure_type="floor", project_id="PROJ01", parent_id="B01")
  │     │     │     fact_ids → []   ← floor 本轮必须为空
  │     │     │
  │     │     ├── Component("K001")
  │     │     │     parent_structure_id → "F1"
  │     │     │     defect_ids → ["C001"] 或 []  (允许空)
  │     │     │     fact_ids → ["fact:component.K001.id", "fact:component.K001.concrete_strength"]
  │     │     │           │
  │     │     │           └── Defect("C001")
  │     │     │                 host_component_id → "K001"   ← 必填弱实体锚点
  │     │     │                 fact_ids → ["fact:defect.C001.width",
  │     │     │                            "fact:defect.C001.pattern", ...]
  │     │     │
  │     │     └── Component("K002")
  │     │           defect_ids → []  ← 无缺陷，合法
  │     │
  │     └── Evidence("E001")
  │           raw_source_id → "photo-034.jpg"
  │           source_refs → [SourceRef("photo-034.jpg", "full-image")]
  │           related_fact_ids → ["fact:defect.C001.width", "fact:defect.C001.pattern"]  (Evidence 不限 scope)
  │           related_object_ids → ["C001"]  (可选)
  │
  └── Structure("B02", structure_type="building", project_id="PROJ01", parent_id=None)
```

## scope 一致性表（Validator 本地即可判定）

```
Domain Object                → 允许的 scope_class in fact_ids[]
─────────────────────────────────────────────────────────
Project                      → {"project"}
Structure(type="building")   → {"building"}
Structure(other types)       → {}  (fact_ids 必须为空)
Component                    → {"component"}
Defect                       → {"defect"}
Evidence                     → None  (不检查 scope，跨类关联是 Evidence 职责)
任何 Domain Object           → fact:structure.* 永远非法（不在 scope_class）
```

---

# 四、最终 Validator 边界

## 4.1 通用结构检查（复用 Step 7-B 工具）
| # | 检查 | 工具 |
|---|------|------|
| G-1 | `id` 非空非空白 | 字符串检查 |
| G-2 | `fact_ids[]` 每个元素是合法三段式 grammar | `is_valid_fact_id(raw)` 不传 registry/fact_type |
| G-3 | 不自持 value | dataclass 设计已保证 |

## 4.2 scope 一致性检查（本轮本地新增）
| 对象 | 规则 |
|------|------|
| Project | fact_ids 每个 scope_class 必须 ∈ {"project"} |
| Structure(type="building") | 必须 ∈ {"building"} |
| Structure(type=floor/axis/zone/structure_unit) | fact_ids **必须空** |
| Component | 必须 ∈ {"component"} |
| Defect | 必须 ∈ {"defect"} |
| Evidence | 不检查（跨 scope 关联是 Evidence 的职责） |

**实现**：用已有的 `parse_fact_id(fid).scope_class` 取出 scope 段，比对允许集合。无需 Store。

## 4.3 关系字段检查
| 对象 | 检查 |
|------|------|
| Structure | `structure_type` 必填非空 ∈ STRUCTURE_TYPES；`project_id` 必填非空；`parent_id` 若非空则非空字符串；顶层（parent_id=None）必须是 building？——**本轮不强制**（Store/组装层再判定） |
| Component | `parent_structure_id` 必填非空；`defect_ids` 允许空 list；list 元素若非空则非空字符串 |
| Defect | `host_component_id` 必填非空（弱实体锚点） |
| Evidence | `raw_source_id` 必填非空；`source_refs` 非空且每个用 `validate_source_ref`；`related_fact_ids` 允许空；`related_object_ids` 允许空（可选） |

## 4.4 本轮**不做**的检查（留给后续层）
- Fact 是否真的存在于 Fact Store
- `defect_ids` 里的 id 是否对应真实 Defect
- `parent_structure_id` / `project_id` 是否对应真实 Structure / Project
- Evidence 的 `raw_source_id` 是否对应真实 RawSource
- `host_component_id` 是否对应真实 Component
- Fact.status 是否 filled

---

# 五、最终测试计划

## 测试文件：`tests/cdm/test_domain.py`

### Project（约 6 用例）
| # | 用例 | 预期 |
|---|------|------|
| P-1 | `id="PROJ01"`, `fact_ids=[]` | ✅ |
| P-2 | `id=""` | ❌ |
| P-3 | `fact_ids=["fact:project.PROJ01.name", "fact:project.PROJ01.id"]` | ✅ |
| P-4 | `fact_ids=["fact:component.K001.concrete_strength"]` | ❌ scope 不一致 |
| P-5 | `fact_ids=["fact:defect.C001.width"]` | ❌ scope 不一致 |
| P-6 | `fact_ids=["not_a_fact_id"]` | ❌ 非法格式 |

### Structure（约 11 用例）
| # | 用例 | 预期 |
|---|------|------|
| S-1 | `structure_type="building"`, `fact_ids=[]`, `project_id="PROJ01"` | ✅ |
| S-2 | `structure_type="building"`, `fact_ids=["fact:building.B01.area"]`, `project_id="PROJ01"` | ✅ |
| S-3 | `structure_type="floor"`, `fact_ids=[]`, `project_id="PROJ01"`, `parent_id="B01"` | ✅ |
| S-4 | `structure_type="floor"`, `fact_ids=["fact:building.B01.area"]` | ❌ **floor 本轮必须空** |
| S-5 | `structure_type=None`（默认未传） | ❌ |
| S-6 | `structure_type="illegal_type"` | ❌ |
| S-7 | `fact_ids=["fact:structure.F1.area"]` | ❌ scope_class 根本不存在 |
| S-8 | `structure_type="building"`, `fact_ids=["fact:component.K001.id"]` | ❌ scope 不一致 |
| S-9 | `structure_type="building"`, `project_id="PROJ01"` | ✅ |
| S-10 | `structure_type="building"`, 不传 `project_id` | ❌ |
| S-11 | `structure_type="building"`, `project_id=""` | ❌ |

### Component（约 10 用例）
| # | 用例 | 预期 |
|---|------|------|
| C-1 | `defect_ids=[]`, `parent_structure_id="B01"` | ✅ 无缺陷合法 |
| C-2 | `defect_ids=["C001"]` | ✅ |
| C-3 | `defect_ids=["C001", "C002"]` | ✅ |
| C-4 | `defect_ids=[""]` | ❌ list 元素空 |
| C-5 | `parent_structure_id=""` | ❌ |
| C-6 | 不传 `parent_structure_id` | ❌ |
| C-7 | `fact_ids=["fact:component.K001.concrete_strength"]` | ✅ |
| C-8 | `fact_ids=["fact:defect.C001.width"]` | ❌ scope 不一致 |
| C-9 | `fact_ids=["fact:project.PROJ01.name"]` | ❌ scope 不一致 |
| C-10 | `id=""` | ❌ |

### Defect（约 7 用例）
| # | 用例 | 预期 |
|---|------|------|
| D-1 | `host_component_id="K001"`, `fact_ids=[]` | ✅ |
| D-2 | `host_component_id="K001"`, `fact_ids=["fact:defect.C001.width", "fact:defect.C001.pattern"]` | ✅ |
| D-3 | `host_component_id=""` | ❌ |
| D-4 | 不传 `host_component_id` | ❌ |
| D-5 | `id=""` | ❌ |
| D-6 | `fact_ids=["fact:component.K001.id"]` | ❌ scope 不一致 |
| D-7 | `fact_ids=["fact:project.PROJ01.name"]` | ❌ scope 不一致 |

### Evidence（约 8 用例）
| # | 用例 | 预期 |
|---|------|------|
| E-1 | `raw_source_id="photo-034.jpg"`, `source_refs=[SourceRef(...)]`, `related_fact_ids=[]` | ✅ |
| E-2 | `related_fact_ids=["fact:defect.C001.width", "fact:component.K001.concrete_strength"]` | ✅ 跨 scope 合法 |
| E-3 | 不传 `raw_source_id` | ❌ |
| E-4 | `raw_source_id=""` | ❌ |
| E-5 | `related_fact_ids=["not_a_fact_id"]` | ❌ 非法格式 |
| E-6 | 不传 `related_object_ids`（可选） | ✅ |
| E-7 | `source_refs=[]` | ❌ |
| E-8 | `id=""` | ❌ |

### JSON Roundtrip（约 7 用例）
| # | 对象 | 验证项 |
|---|------|--------|
| J-1 | Project | id / fact_ids / metadata 一致 |
| J-2 | Structure(building) | structure_type 必填、project_id、parent_id=None |
| J-3 | Structure(floor) | parent_id 非空、fact_ids=[] |
| J-4 | Component | defect_ids=[] 和 defect_ids=["C001"] 两种 |
| J-5 | Defect | host_component_id |
| J-6 | Evidence | 嵌套 SourceRef 完整 roundtrip |
| J-7 | Evidence 不传 related_object_ids | 反序列化后仍可缺省 |

**测试总数**：P(6) + S(11) + C(10) + D(7) + E(8) + J(7) ≈ **49 用例**

---

# 六、准备创建/修改的文件

| 操作 | 文件 | 内容 |
|------|------|------|
| **新建** | `src/cdm/domain.py` | 5 个 dataclass + STRUCTURE_TYPES 枚举 + to_dict/from_dict/to_json |
| **新建** | `src/cdm/domain_validate.py` | 5 个 validate_* 函数 + scope 一致性表 + structure_type 枚举校验 |
| **新建** | `tests/cdm/test_domain.py` | 上述 ~49 用例 |
| **修改** | `src/cdm/__init__.py` | 新增 domain.py / domain_validate.py 导出 |

**严格不修改**：
- `src/cdm/types.py` — Step 7-B 纯净
- `src/cdm/registry.py` — 固定枚举纯净（scope_class 集合不动）
- `src/cdm/id.py` — 三段式 grammar 纯净
- `src/cdm/validate.py` — Step 7-B 的 74 个测试纯净
- `tests/cdm/test_types.py` — Step 7-B 测试纯净
- 任何 `docs/*.md` — 不碰设计文档

---

方案已收敛。等待下一条指令开始写代码。
