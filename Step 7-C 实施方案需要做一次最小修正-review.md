好的，先找到 Step 7-C-review.md 再并行执行读取。
好的，用户要求修订 Step 7-C 实施方案，并且在开始前要求先读一份 Step 7-C-review.md。让我先找到这份 review 文件。

用户提到的关键修正点我都看到了：
1. Component.defect_ids[] 可以为空
2. 不创建独立 Building class，只用 Structure
3. 区分 A 类（设计明确）和 B 类（实现层）字段
4. Project 最小化
5. Structure 最小化 + parent_id
6. Component 最小化 + parent_structure_id
7. Defect 保持弱实体
8. Evidence 保持最小
9. 验证边界收紧
10. 测试重点补充

先找一下 Step 7-C-review.md 文件。

Glob 没找到 Step 7-C-review.md。让我再扩大范围搜一下。

用户提到 Step 7-C-review.md 但工作区没找到。可能用户的意思是让我直接根据他刚才的 12 条修订要求 + 已有的设计文档来修订方案。先确认一下 reviews/ 里那份文件的内容，然后直接基于用户列出的 12 条修订输出新方案。

不过用户明确说"请根据当前：... Step 7-C-review.md 重新修订"，但文件不存在。我应该基于用户刚才贴出的 12 条修订点（用户消息里的全部内容）+ 现有设计文档来输出修订方案。用户刚才的消息本身就是 review。

好，直接开始输出修订方案。
好，Step 7-C-review.md 在工作区不存在（只发现 `reviews/设计澄清会话-review.md`）。但您消息中的 12 条修订要求本身就是完整的 review 指令。我直接基于 **您列出的 12 条修订点** + 已冻结的 3 份设计文档（02/03/04）+ Step 7-B 代码基线，输出修订后的方案。

---

# 一、修订后的 Step 7-C 最小范围

## 本轮实现（5 个 Domain Object，全部位于 L3 层）

| # | 对象 | 设计文档来源 | 角色 |
|---|------|-------------|------|
| 1 | **Project** | §18 | 顶层容器，承载全部 Structure |
| 2 | **Structure** | §20 | 统一结构层级（Building / Floor / Axis / Zone / Structure Unit 都是 Structure 的实例） |
| 3 | **Component** | §21 | 工程构件，1:N Defect；不区分 Building 与 Structure |
| 4 | **Defect** | §17 (v0.3.1) | 弱实体，**必须**携带 `host_component_id`；crack 是 Defect 的实例，不是新 class |
| 5 | **Evidence** | §28 | 证据载体，追溯链 Evidence → RawSource |

## 本轮明确不做
- ❌ 独立 Building class — 全部合并到 Structure
- ❌ InspectionItem / Measurement / Criterion / Evaluation / Conclusion / ConclusionRule
- ❌ Report IR / Parser / DOCX / PDF / Validator / LLM / Skill / Agent
- ❌ 修改设计文档、registry.py、id.py、types.py

## 在四层模型中的位置
```
L1 RawSource        ← Step 7-B
L2 Facts            ← Step 7-B（已封）
L3 Domain Objects   ← 本轮 Step 7-C（5 个对象）
L4 Report IR        ← 后续
```

---

# 二、修订后的 Domain Object 字段

### 字段分类说明
- **A 类**：03_CANONICAL_DATA_MODEL.md 已明确要求的字段
- **B 类**：L3 实现层为表达关系而引入的字段，未被 03 冻结

---

## 1. Project

| 字段 | 类型 | 分类 | 必填 | 说明 |
|------|------|------|------|------|
| `id` | str | **A** | ✅ | 稳定领域标识（如 `"PROJ01"`） |
| `fact_ids` | list[str] | **A** | ✅ | 本 Project 相关的三段式 Fact ID 列表（可空） |
| `metadata` | dict | **B** | ❌ | 最小元数据（预留，可空） |

---

## 2. Structure（代替 Building）

Building / Floor / Axis / Zone / StructureUnit **全部是 Structure 的实例**，用 `structure_type` 区分类型。

| 字段 | 类型 | 分类 | 必填 | 说明 |
|------|------|------|------|------|
| `id` | str | **A** | ✅ | 稳定领域标识（如 `"B01"`） |
| `fact_ids` | list[str] | **A** | ✅ | 本 Structure 相关的三段式 Fact ID 列表（可空） |
| `parent_id` | Optional[str] | **B** | ❌ | 父 Structure 的 `id`；顶层 Structure（如 Building）可以为 None |
| `structure_type` | Optional[str] | **B** | ❌ | 区分 `"building"` / `"floor"` / `"axis"` / `"zone"` / `"structure_unit"` |

---

## 3. Component

| 字段 | 类型 | 分类 | 必填 | 说明 |
|------|------|------|------|------|
| `id` | str | **A** | ✅ | 稳定领域标识（如 `"K001"`） |
| `fact_ids` | list[str] | **A** | ✅ | 本 Component 相关的三段式 Fact ID 列表（可空） |
| `parent_structure_id` | str | **B** | ✅ | 所属 Structure 的 `id` |
| `defect_ids` | list[str] | **B** | ✅（允许空） | 关联 Defect 的 `id` 列表；**允许为空列表** |

> **关键约束**：Component 没有缺陷时必须允许 `defect_ids = []`（已修原方案错误）。

---

## 4. Defect（弱实体）

| 字段 | 类型 | 分类 | 必填 | 说明 |
|------|------|------|------|------|
| `id` | str | **A** | ✅ | 稳定领域标识（如 `"C001"`） |
| `fact_ids` | list[str] | **A** | ✅ | 本 Defect 相关的三段式 Fact ID 列表（可空） |
| `host_component_id` | str | **A** | ✅ | 弱实体锚点，必须指向宿主 Component 的 `id` |

> **弱实体设计不变**：Defect 不能独立存在，缺失 `host_component_id` 即为非法。
> **crack 不进 class**：`fact:defect.C001.width` / `fact:defect.C001.pattern` 等 Fact 承载裂纹实例类型（由 Fact.value 区分 crack / corrosion / spall / ...）。

---

## 5. Evidence

| 字段 | 类型 | 分类 | 必填 | 说明 |
|------|------|------|------|------|
| `id` | str | **A** | ✅ | 稳定领域标识（如 `"E001"`） |
| `raw_source_id` | str | **A** | ✅ | 关联的 L1 RawSource.source_id |
| `source_refs` | list[SourceRef] | **A** | ✅ | 证据在原始资料中的定位（图片区域 / 文档页码 / 表格单元格） |
| `related_fact_ids` | list[str] | **A** | ✅（允许空） | 本 Evidence 支撑的三段式 Fact ID 列表 |
| `related_object_ids` | list[str] | **B** | ❌（可选） | 本 Evidence 关联的 Domain Object id 列表（预留给 Defect / Component 直接关联） |

---

# 三、Domain Object 与 Fact 的关系

## 核心原则（来自 §29 + 冻结规则）

```
Fact = 最小断言单元        Domain Object = 工程语义组织
├── 有 value（数值/文本）   ├── 没有 value —— 值一律从 Fact 层读取
├── 有 status / review_status ├── 不存状态 —— 状态一律在 Fact 层
├── 有 source_refs         ├── 只有 fact_ids[] 引用 + 关系字段
└── 可被 Report IR Ref 引用 └── 可被 Pipeline 编排 / Concluder 聚合
```

**Domain Object 不复制一套 Fact 真相。** 所有实际值、状态、溯源都通过 `fact_ids[]` 回到 L2 Fact 层获取。

---

## 关系拓扑

```
Project("PROJ01") ──fact_ids──→ ["fact:project.PROJ01.name", ...]
  │
  │ (B类实现层关系)
  ├── Structure("B01") ──fact_ids──→ ["fact:building.B01.id", "fact:building.B01.area", ...]
  │     │                            Structure.parent_id → None  (顶层 Building)
  │     │ (B类实现层关系)
  │     ├── Structure("F1") ──fact_ids──→ [...]
  │     │     │                       parent_id → "B01"
  │     │     │ (B类实现层关系)
  │     │     └── Component("K001") ──fact_ids──→ ["fact:component.K001.id", "fact:component.K001.concrete_strength"]
  │     │                               parent_structure_id → "F1"
  │     │                               defect_ids → ["C001"] 或 []  (允许空)
  │     │                                 │
  │     │                                 └── Defect("C001") ──fact_ids──→ ["fact:defect.C001.width", "fact:defect.C001.pattern", ...]
  │     │                                                      host_component_id → "K001"  ← 必填弱实体锚点
  │     │
  │     └── Component("K002") ──defect_ids──→ []  (无缺陷，合法!)
  │
  └── Evidence("E001")
        raw_source_id → "photo-034.jpg"
        source_refs → [SourceRef("photo-034.jpg", "full-image")]
        related_fact_ids → ["fact:defect.C001.width", "fact:defect.C001.pattern"]
        related_object_ids → ["C001"]  (可选)
```

---

## L2 → L3 数据流向

```
Pipeline Step2 Extract
  ↓ 产出 Fact 集合 + 基础溯源
L3 Domain Object Builder
  ↓ 读取 Fact.id 三段式（scope_class 决定归属）
  ↓ 按 instance_key 聚合
  ↓ 建立关系（parent_id / defect_ids / host_component_id）
产出 Domain Object 集合
```

**Fact.id 的 `scope_class` 段天然决定它属于哪个 Domain Object 类型**：
- `fact:project.*` → Project
- `fact:building.*` / `fact:structure.*` → Structure （本轮 registry 里只有 building）
- `fact:component.*` → Component
- `fact:defect.*` → Defect

---

# 四、验证边界

## Domain Object Validator 职责（本轮只做结构正确）

| # | 检查内容 | 性质 |
|---|---------|------|
| 1 | `id` 非空非空白 | 结构 |
| 2 | `fact_ids[]` 每个元素是**合法三段式 Fact ID 格式**（用已有的 `is_valid_fact_id`，不传 registry/fact_type，只查 grammar + scope_class 枚举） | 结构 |
| 3 | 必填关系字段存在：Component 必须有 `parent_structure_id`；Defect 必须有 `host_component_id`；Evidence 必须有 `raw_source_id` | 结构 |
| 4 | B 类可选 list 元素合法：`defect_ids[]` 元素非空（**允许 list 为空**）；`parent_id` 若存在则非空 | 结构 |
| 5 | 不自持 value：不复制 Fact 值 | 设计约束（编译时已保证，无需运行时检查） |
| 6 | SourceRef 结构合法（复用已有 `validate_source_ref`） | 结构 |

## 本轮**不做**的验证

| 跳过的检查 | 原因 |
|-----------|------|
| Fact 是否真的存在于 Fact Store | 留给后续 Store / Pipeline 层 |
| `defect_ids` 里的 id 是否真的对应 Defect 实例 | 留给 Pipeline 组装阶段 |
| `parent_structure_id` 是否真的对应存在的 Structure | 留给 Pipeline 组装阶段 |
| `host_component_id` 是否真的对应存在的 Component | 留给 Pipeline 组装阶段 |
| Evidence 的 `raw_source_id` 是否真的对应 RawSource | 留给后续 Store 层 |
| Fact 的 status 是否 filled（不允许引用 missing/conflict） | 留给 Fact Store 层或 Pipeline 层 |

## 复用 Step 7-B 已有工具
- `cdm.id.is_valid_fact_id(raw)` — 只查 grammar + scope_class 枚举（不传 registry/fact_type）
- `cdm.validate.validate_source_ref(sr)` — SourceRef 结构校验
- `cdm.registry.SCOPE_CLASSES` — scope_class 枚举

---

# 五、测试计划

## 测试文件：`tests/cdm/test_domain.py`

### Project（基本构造）
| # | 用例 | 预期 |
|---|------|------|
| P-1 | `id="PROJ01"`, `fact_ids=[]` | ✅ 合法 |
| P-2 | `id=""`, `fact_ids=[]` | ❌ 非法（id 空） |
| P-3 | `id="PROJ01"`, `fact_ids=["fact:project.PROJ01.name", "fact:project.PROJ01.id"]` | ✅ 合法 |
| P-4 | `fact_ids=["not_a_valid_id"]` | ❌ 非法（fact_id 格式错误） |

### Structure（Building/Floor 合并实现）
| # | 用例 | 预期 |
|---|------|------|
| S-1 | `id="B01"`, `parent_id=None`, `structure_type="building"`, `fact_ids=[]` | ✅ 顶层 Structure 合法 |
| S-2 | `id="F1"`, `parent_id="B01"`, `structure_type="floor"`, `fact_ids=[]` | ✅ 子层级合法 |
| S-3 | `id=""` | ❌ 非法（id 空） |
| S-4 | `fact_ids=["fact:building.B01.area"]` | ✅ 合法 |
| S-5 | `fact_ids=["invalid"]` | ❌ 非法 |

### Component（修 defect_ids 允许为空）
| # | 用例 | 预期 |
|---|------|------|
| C-1 | `id="K001"`, `parent_structure_id="B01"`, `defect_ids=[]` | ✅ **无缺陷 Component 合法** |
| C-2 | `id="K001"`, `parent_structure_id="B01"`, `defect_ids=["C001"]` | ✅ 有缺陷合法 |
| C-3 | `id="K001"`, `parent_structure_id="B01"`, `defect_ids=["C001", "C002"]` | ✅ 多个缺陷合法 |
| C-4 | `id="K001"`, `parent_structure_id="B01"`, `defect_ids=[""]` | ❌ **非法**（list 元素空） |
| C-5 | `parent_structure_id=""` | ❌ 非法（必填字段缺失） |
| C-6 | `parent_structure_id=None`（未传） | ❌ 非法（必填字段缺失） |
| C-7 | `fact_ids=["fact:component.K001.concrete_strength"]` | ✅ 合法 |
| C-8 | `id=""` | ❌ 非法 |

### Defect（弱实体必须 host）
| # | 用例 | 预期 |
|---|------|------|
| D-1 | `id="C001"`, `host_component_id="K001"`, `fact_ids=[]` | ✅ 合法 |
| D-2 | `id="C001"`, `host_component_id="K001"`, `fact_ids=["fact:defect.C001.width", "fact:defect.C001.pattern"]` | ✅ 合法 |
| D-3 | `id="C001"`, `host_component_id=""` | ❌ **非法**（host_component_id 空） |
| D-4 | `id="C001"`, 不传 `host_component_id` | ❌ **非法**（必填弱实体锚点缺失） |
| D-5 | `id=""`, `host_component_id="K001"` | ❌ 非法 |
| D-6 | `fact_ids=["not_a_fact_id"]` | ❌ 非法 |

### Evidence（追溯链）
| # | 用例 | 预期 |
|---|------|------|
| E-1 | `id="E001"`, `raw_source_id="photo-034.jpg"`, `source_refs=[SourceRef(...)]`, `related_fact_ids=[]` | ✅ 合法 |
| E-2 | `related_fact_ids=["fact:defect.C001.width", "fact:defect.C001.pattern"]` | ✅ 合法 |
| E-3 | 不传 `raw_source_id`（默认 None） | ❌ **非法**（追溯链断裂） |
| E-4 | `raw_source_id=""` | ❌ 非法 |
| E-5 | `related_fact_ids=["invalid"]` | ❌ 非法 |
| E-6 | 不传 `related_object_ids`（可选，默认 []） | ✅ 合法 |
| E-7 | `source_refs=[]` | ❌ 非法（Evidence 必须有定位） |
| E-8 | `id=""` | ❌ 非法 |

### JSON Roundtrip
| # | 对象 | 预期 |
|---|------|------|
| J-1 | Project 完整序列化 / 反序列化 | id / fact_ids 一致 |
| J-2 | Structure 含 parent_id + structure_type | 一致 |
| J-3 | Component 含 defect_ids=[] 和 defect_ids=["C001"] 两种情况 | 一致 |
| J-4 | Defect 含 host_component_id | 一致 |
| J-5 | Evidence 含 source_refs（嵌套 SourceRef 序列化） | 一致 |

---

# 六、准备创建/修改的文件

| 操作 | 文件 | 说明 |
|------|------|------|
| **新建** | `src/cdm/domain.py` | 5 个 Domain Object dataclass + to_dict / from_dict / to_json |
| **新建** | `src/cdm/domain_validate.py` | 5 个 `validate_*` 函数（复用 `validate_source_ref` / `is_valid_fact_id`） |
| **新建** | `tests/cdm/test_domain.py` | 上述 P/S/C/D/E/J 共约 35+ 测试用例 |
| **修改** | `src/cdm/__init__.py` | 新增 `domain.py` 导出 |

**严格不修改**：
- `src/cdm/types.py` — 保持 Step 7-B 纯净
- `src/cdm/registry.py` — 保持固定枚举纯净
- `src/cdm/id.py` — 保持三段式 grammar 纯净
- `src/cdm/validate.py` — 保持 Step 7-B 的 74 个测试纯净
- 任何 `docs/*.md` — 不碰设计文档
- 任何 `tests/cdm/test_types.py` — 不碰 Step 7-B 测试

---

以上是修订后的 Step 7-C 实施方案。确认后开始写代码。
