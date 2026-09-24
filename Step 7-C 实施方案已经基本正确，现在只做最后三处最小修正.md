用户给出了三个最小修正，直接输出最终方案，不需要再读文件或调用 Advisor。规则明确要求本轮只输出方案、不写代码。
好，三处修正已吸收。以下是最终版本。

---

# 一、最终修订后的 Step 7-C 实施范围

## 本轮实现的 5 个 Domain Object（L3 层）

| # | 对象 | 设计文档来源 | 核心角色 |
|---|------|-------------|---------|
| 1 | **Project** | §18 | 顶层容器，承载全部 Structure |
| 2 | **Structure** | §20 | 统一结构层级（Building / Floor / Axis / Zone / StructureUnit 都是 Structure 实例，用 `structure_type` 区分） |
| 3 | **Component** | §21 | 工程构件，1:N Defect |
| 4 | **Defect** | §17 (v0.3.1) | 弱实体，必须携带 `host_component_id`；crack / corrosion / spall 等都是 Defect 实例，不是新 class |
| 5 | **Evidence** | §28 | 证据载体，追溯链 Evidence → RawSource |

## scope_class 合法性（维持 Step 7-B 冻结集合，不扩大）

```
project | building | component | defect | point | sample | material
```

- `fact:building.*` 合法 → 挂在 Structure(type="building")
- `fact:structure.*` **非法** → scope_class 里没有 structure，**永远不能作为合法 Fact ID**
- Structure(type="floor"/"axis"/"zone"/"structure_unit") 本轮 `fact_ids=[]`，未来如需描述这些层级自身事实再单独申请 scope_class

## 本轮严格禁止
- ❌ 独立 Building class
- ❌ 扩大 scope_class（不新增 structure / crack 等）
- ❌ 修改 registry.py / id.py / types.py / validate.py / 任何 docs
- ❌ InspectionItem / Measurement / Criterion / Evaluation / Conclusion / ConclusionRule / Report IR / Parser / DOCX / PDF / Validator 全部逻辑 / LLM / Skill / Agent

---

# 二、最终字段定义

## 字段分类
- **A 类**：03_CANONICAL_DATA_MODEL.md 已明确要求
- **B 类**：L3 实现层为表达关系而引入，未被 03 冻结

---

### 1. Project

| 字段 | 类型 | 分类 | 必填 | 说明 |
|------|------|------|------|------|
| `id` | str | A | ✅ | 稳定领域标识（如 `"PROJ01"`） |
| `fact_ids` | list[str] | A | ✅ | 只允许 `fact:project.*`（scope 一致性检查） |
| `metadata` | dict | B | ❌ | 最小元数据（预留，可空） |

---

### 2. Structure（统一 Building / Floor / Axis / Zone / StructureUnit）

| 字段 | 类型 | 分类 | 必填 | 说明 |
|------|------|------|------|------|
| `id` | str | A | ✅ | 稳定领域标识（如 `"B01"`） |
| `fact_ids` | list[str] | A | ✅ | type="building" 时允许 `fact:building.*`；其他 type 本轮 `fact_ids=[]`（scope 一致性检查） |
| `structure_type` | Optional[str] | B | ❌ | `"building"` / `"floor"` / `"axis"` / `"zone"` / `"structure_unit"` |
| `parent_id` | Optional[str] | B | ❌ | 父 Structure 的 `id`；顶层（如 Building）可为 None |

---

### 3. Component

| 字段 | 类型 | 分类 | 必填 | 说明 |
|------|------|------|------|------|
| `id` | str | A | ✅ | 稳定领域标识（如 `"K001"`） |
| `fact_ids` | list[str] | A | ✅ | 只允许 `fact:component.*`（scope 一致性检查） |
| `parent_structure_id` | str | B | ✅ | 所属 Structure 的 `id` |
| `defect_ids` | list[str] | B | ✅（**允许空**） | 关联 Defect 的 `id`；**无缺陷时必须允许 `[]`** |

---

### 4. Defect（弱实体）

| 字段 | 类型 | 分类 | 必填 | 说明 |
|------|------|------|------|------|
| `id` | str | A | ✅ | 稳定领域标识（如 `"C001"`） |
| `fact_ids` | list[str] | A | ✅ | 只允许 `fact:defect.*`（scope 一致性检查） |
| `host_component_id` | str | A | ✅ | 弱实体锚点，必须指向宿主 Component 的 `id` |

> crack / corrosion / spall 等类型通过 Defect 自身的 `fact_ids`（如 `fact:defect.C001.pattern`）区分，不新增 class。

---

### 5. Evidence

| 字段 | 类型 | 分类 | 必填 | 说明 |
|------|------|------|------|------|
| `id` | str | A | ✅ | 稳定领域标识（如 `"E001"`） |
| `raw_source_id` | str | A | ✅ | 关联的 L1 `RawSource.source_id` |
| `source_refs` | list[SourceRef] | A | ✅ | 定位（图片区域 / 文档页码 / 表格单元格） |
| `related_fact_ids` | list[str] | A | ✅（允许空） | 支撑的 Fact 列表；本轮不限制 scope（跨类关联是 Evidence 的职责） |
| `related_object_ids` | list[str] | B | ❌（可选） | 直接关联的 Domain Object id（预留给 Defect / Component 直接关联） |

---

# 三、最终 Domain Object → Fact 关系

## 关系拓扑（单一 Structure 统一 Building / Floor）

```
Project("PROJ01")
  │
  ├── Structure("B01", structure_type="building")        ← 顶层，parent_id=None
  │     │     fact_ids → ["fact:building.B01.id", "fact:building.B01.area", ...]
  │     │
  │     ├── Structure("F1", structure_type="floor")      ← parent_id="B01"
  │     │     │     fact_ids → []  (本轮暂无非 building 的 scope)
  │     │     │
  │     │     ├── Component("K001")
  │     │     │     parent_structure_id → "F1"
  │     │     │     defect_ids → ["C001"]
  │     │     │     fact_ids → ["fact:component.K001.id", "fact:component.K001.concrete_strength"]
  │     │     │           │
  │     │     │           └── Defect("C001")
  │     │     │                 host_component_id → "K001"  ← 弱实体必填
  │     │     │                 fact_ids → ["fact:defect.C001.width",
  │     │     │                            "fact:defect.C001.pattern",
  │     │     │                            "fact:defect.C001.judgement", ...]
  │     │     │
  │     │     └── Component("K002")
  │     │           defect_ids → []  ← 无缺陷，合法！
  │     │
  │     └── Evidence("E001")
  │           raw_source_id → "photo-034.jpg"
  │           source_refs → [SourceRef("photo-034.jpg", "full-image")]
  │           related_fact_ids → ["fact:defect.C001.width", "fact:defect.C001.pattern"]
  │           related_object_ids → ["C001"]  (可选)
  │
  └── Structure("B02", structure_type="building")
```

## Fact.id 的 scope_class 天然决定归属

```
scope_class          → 允许挂在              → 禁止挂在
───────────────────────────────────────────────────────
project              → Project               → Component / Defect / Structure
building             → Structure(type="building") → Project / Component / Defect
component            → Component             → Project / Defect / Structure
defect               → Defect                → Project / Component / Structure
point / sample / material → (后续)             → 本轮暂不绑定
structure            → ❌ 不存在于 scope_class → 任何地方都是非法 ID
```

## Domain Object 不复制 Fact 真相

```
Fact:                       Domain Object:
─────────────────────────────────────
有 value                    ❌ 没有 value
有 status / review_status   ❌ 没有 status
有 source_refs              只有 fact_ids[] + 关系字段
可被 Report IR Ref 引用     可被 Pipeline 编排
```

---

# 四、最终 Validator 边界

## 本轮 Validator 职责（纯结构检查，不触碰 Fact Store）

### 4.1 通用检查（所有 Domain Object）
| # | 检查 | 性质 | 工具 |
|---|------|------|------|
| G-1 | `id` 非空非空白 | 结构 | 字符串检查 |
| G-2 | `fact_ids[]` 每个元素是合法三段式 Fact ID grammar | 结构 | `is_valid_fact_id(raw)` 不传 registry/fact_type |
| G-3 | 不自持 value | 设计约束 | dataclass 字段设计已保证 |

### 4.2 scope 一致性检查（新增，本轮本地即可做）

```
Domain Object             → 允许的 scope_class in fact_ids[]
─────────────────────────────────────────────────────────
Project                   → 仅 "project"
Structure(type="building")→ 仅 "building"
Structure(type=floor/axis/zone/structure_unit) → 仅允许空 fact_ids（本轮无对应 scope）
Component                 → 仅 "component"
Defect                    → 仅 "defect"
Evidence                  → 不限（跨类关联是 Evidence 职责）
```

**实现方式**：`parse_fact_id(fid).scope_class` 取出 scope_class 段，与 Domain Object 允许集合比对。无需 Store 存在。

### 4.3 关系字段检查

| 对象 | 检查 |
|------|------|
| Component | `parent_structure_id` 必填、非空；`defect_ids` **允许空 list**；list 元素非空（若存在） |
| Defect | `host_component_id` 必填、非空（弱实体锚点） |
| Evidence | `raw_source_id` 必填、非空；`source_refs` 非空（定位必须有），每个元素用 `validate_source_ref`；`related_fact_ids` 允许空 |

### 4.4 本轮**不做**的检查

| 跳过的检查 | 原因 |
|-----------|------|
| Fact 是否真的存在于 Fact Store | 留给 Store / Pipeline 层 |
| `defect_ids` 里的 id 是否真的对应 Defect | 留给 Pipeline 组装 |
| `parent_structure_id` 是否真的对应存在的 Structure | 留给 Pipeline 组装 |
| Evidence 的 `raw_source_id` 是否真的对应 RawSource | 留给 Store 层 |
| Fact.status 是否 filled | 留给 Fact Store / Pipeline 层 |
| Evidence 的 `related_object_ids` 是否真的对应存在 | 留给 Pipeline 组装 |

---

# 五、最终测试计划

## 测试文件：`tests/cdm/test_domain.py`

### Project
| # | 用例 | 预期 |
|---|------|------|
| P-1 | `id="PROJ01"`, `fact_ids=[]` | ✅ 合法 |
| P-2 | `id=""` | ❌ 非法 |
| P-3 | `fact_ids=["fact:project.PROJ01.name", "fact:project.PROJ01.id"]` | ✅ 合法 |
| P-4 | `fact_ids=["fact:component.K001.concrete_strength"]` | ❌ **scope 不一致**（project 对象挂了 component fact） |
| P-5 | `fact_ids=["fact:defect.C001.width"]` | ❌ **scope 不一致** |
| P-6 | `fact_ids=["not_a_fact_id"]` | ❌ 非法格式 |

### Structure（统一 Building / Floor）
| # | 用例 | 预期 |
|---|------|------|
| S-1 | `id="B01"`, `parent_id=None`, `structure_type="building"`, `fact_ids=[]` | ✅ 顶层 Building 合法 |
| S-2 | `id="B01"`, `structure_type="building"`, `fact_ids=["fact:building.B01.area"]` | ✅ building scope 合法 |
| S-3 | `id="F1"`, `parent_id="B01"`, `structure_type="floor"`, `fact_ids=[]` | ✅ 顶层无缺陷 Floor 合法 |
| S-4 | `id="F1"`, `structure_type="floor"`, `fact_ids=["fact:building.B01.area"]` | ✅ 本轮暂不报错（floor 暂无对应 scope） |
| S-5 | `id=""` | ❌ 非法 |
| S-6 | `fact_ids=["fact:structure.F1.area"]` | ❌ **非法 ID（structure 不是 scope_class）** |
| S-7 | `structure_type="building"`, `fact_ids=["fact:component.K001.id"]` | ❌ **scope 不一致** |

### Component
| # | 用例 | 预期 |
|---|------|------|
| C-1 | `id="K001"`, `parent_structure_id="B01"`, `defect_ids=[]` | ✅ **无缺陷合法** |
| C-2 | `defect_ids=["C001"]` | ✅ 有缺陷合法 |
| C-3 | `defect_ids=["C001", "C002"]` | ✅ 多个合法 |
| C-4 | `defect_ids=[""]` | ❌ **非法（list 元素空）** |
| C-5 | `parent_structure_id=""` | ❌ 非法 |
| C-6 | 不传 `parent_structure_id`（默认 None） | ❌ 非法 |
| C-7 | `fact_ids=["fact:component.K001.concrete_strength"]` | ✅ 合法 |
| C-8 | `fact_ids=["fact:defect.C001.width"]` | ❌ **scope 不一致** |
| C-9 | `fact_ids=["fact:project.PROJ01.name"]` | ❌ **scope 不一致** |
| C-10 | `id=""` | ❌ 非法 |

### Defect（弱实体必须 host）
| # | 用例 | 预期 |
|---|------|------|
| D-1 | `id="C001"`, `host_component_id="K001"`, `fact_ids=[]` | ✅ 合法 |
| D-2 | `host_component_id="K001"`, `fact_ids=["fact:defect.C001.width", "fact:defect.C001.pattern"]` | ✅ 合法 |
| D-3 | `host_component_id=""` | ❌ **非法（弱实体锚点空）** |
| D-4 | 不传 `host_component_id`（默认 None） | ❌ **非法（弱实体锚点缺失）** |
| D-5 | `id=""` | ❌ 非法 |
| D-6 | `fact_ids=["fact:component.K001.id"]` | ❌ **scope 不一致** |
| D-7 | `fact_ids=["fact:project.PROJ01.name"]` | ❌ **scope 不一致** |

### Evidence
| # | 用例 | 预期 |
|---|------|------|
| E-1 | `id="E001"`, `raw_source_id="photo-034.jpg"`, `source_refs=[SourceRef(...)]`, `related_fact_ids=[]` | ✅ 合法 |
| E-2 | `related_fact_ids=["fact:defect.C001.width", "fact:defect.C001.pattern"]` | ✅ 合法（Evidence 不限 scope） |
| E-3 | 不传 `raw_source_id`（默认 None） | ❌ **非法（追溯链断裂）** |
| E-4 | `raw_source_id=""` | ❌ 非法 |
| E-5 | `related_fact_ids=["not_a_fact_id"]` | ❌ 非法 |
| E-6 | 不传 `related_object_ids`（可选默认 []） | ✅ 合法 |
| E-7 | `source_refs=[]` | ❌ **非法（定位必须有）** |
| E-8 | `id=""` | ❌ 非法 |

### JSON Roundtrip
| # | 对象 | 预期 |
|---|------|------|
| J-1 | Project 完整序列化 / 反序列化 | id / fact_ids 一致 |
| J-2 | Structure(type="building") 含 parent_id | 一致 |
| J-3 | Structure(type="floor") 含 parent_id、fact_ids=[] | 一致 |
| J-4 | Component 含 defect_ids=[] 和 defect_ids=["C001"] 两种 | 一致 |
| J-5 | Defect 含 host_component_id | 一致 |
| J-6 | Evidence 含嵌套 SourceRef | 一致 |
| J-7 | Evidence 不传 related_object_ids | 一致（反序列化后仍可缺省） |

---

# 六、准备修改的文件

| 操作 | 文件 | 内容 |
|------|------|------|
| **新建** | `src/cdm/domain.py` | 5 个 Domain Object dataclass（Project / Structure / Component / Defect / Evidence），每个带 `to_dict()` / `from_dict()` / `to_json()` |
| **新建** | `src/cdm/domain_validate.py` | 5 个 `validate_*` 函数。scope 一致性表：`{"Project": {"project"}, "Structure.building": {"building"}, "Component": {"component"}, "Defect": {"defect"}, "Evidence": None}` |
| **新建** | `tests/cdm/test_domain.py` | 上述 P/S/C/D/E/J 共约 40+ 用例 |
| **修改** | `src/cdm/__init__.py` | 新增 domain.py 导出 |

**严格不修改**：
- `src/cdm/types.py` — Step 7-B 纯净
- `src/cdm/registry.py` — 固定枚举纯净（scope_class 集合不变）
- `src/cdm/id.py` — 三段式 grammar 纯净
- `src/cdm/validate.py` — Step 7-B 的 74 个测试纯净
- `tests/cdm/test_types.py` — Step 7-B 测试纯净
- 任何 `docs/*.md` — 不碰设计文档

---

以上是最终方案。确认后执行。
