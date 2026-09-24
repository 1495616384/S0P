好的，我先并行执行所有读取和检查操作，建立完整的当前基线视图。
好的，用户明确要求我先做准备工作——读取文档、检查 git 状态、确认 Step 7-B 基线存在，然后输出实施方案（一~四），**不修改任何文件**。这是一个典型的"先调研、后规划"的任务。

让我先规划一下需要并行执行的操作：

1. 读取 8 个文档文件
2. 执行 git status, git log, git tag 命令
3. 列出 src/cdm 和 tests 目录

但我需要先找到这些文件的正确路径。让我先从 Glob 和 LS 开始。

等等，用户列出的文件路径我需要确认：
- docs/00_PROJECT_CONTEXT.md — 需要 Glob 确认 docs 目录下有哪些文件
- 开发流水线.md 和 编程要求.md — 这些可能在根目录或其他位置

让我先并行执行多个操作来获取工作区全貌。

好，文件结构清晰。现在并行读取所有 8 个文档和执行 git 检查。
好的，文件结构确认了：
- docs 下有 7 个 md 文件（00/02/03/04/05/07 + 02_ARCHITECTURE_REVIEW）
- src/cdm 有 5 个 py 文件（__init__, types, id, registry, validate）
- tests/cdm 有 test_types.py
- 根目录有 开发流水线.md, 编程要求.md

现在并行读取所有文档和执行 git 命令。

好，基线已确认。现在读取 Step 7-B 的源码实现，理解现有数据结构，为 Domain Objects 设计做准备。
好，所有文档和 git 状态已确认。现在我需要读取 Step 7-B 实现的 5 个源码文件，理解现有类型结构，以便设计 Domain Object 的实现方案。

先运行现有测试，确认 Step 7-B 的 74 个测试完好通过：
好，所有 Step 7-B 源码已读完。现在运行现有测试确认基线完好，然后输出完整的实施方案。

PowerShell 没有 tail 命令，直接运行 pytest。

完美，74 个测试全部通过，Step 7-B 基线完好。现在我有了完整的工作区理解，可以输出实施方案了。

让我按照用户要求的四个部分来组织：

---

# 一、当前实现基线

## Git 状态
- **当前分支**: master
- **HEAD commit**: 471d193 — `feat(cdm): Step 7-B — Canonical Data Model v0.3.1 data foundation`
- **tag**: step-7b-complete (description: "Step 7-B Canonical Data Model data foundation completed; 74 tests passed.")
- **基线 tag**: design-baseline-m0
- **工作区**: 干净（untracked 有 reviews/ 和 流水线机械重复示意.md，均非源码）

## 文档基线（已读取并确认）
| 文档 | 版本 | 状态 |
|------|------|------|
| 00_PROJECT_CONTEXT.md | — | 项目背景（AI 负责不确定 + 程序负责确定 + 四层模型） |
| 02_ARCHITECTURE.md | v0.3 | Pipeline + Step 架构，四层数据流 |
| 03_CANONICAL_DATA_MODEL.md | v0.3.1 | Fact 三段式 ID + revision + scope_class 含 defect |
| 04_REPORT_IR.md | v0.3.1.1 | 双模态段落 + 纯绑定锚定 |
| 05_EVALUATION.md | v0.3 | 分级 + 纯绑定 + 经济门禁 |
| 07_DECISIONS.md | v0.1 | D-001 ~ D-027 |

## 代码基线
```
src/cdm/
├── __init__.py       # 导出 Registry + ID + Types
├── types.py          # SourceRef / RawSource / Conflict / ConflictCandidate / Fact
├── id.py             # FactID + parse/build/validate (三段式严格)
├── registry.py       # 固定枚举 + FactTypeRegistry + 默认注册表
└── validate.py       # 5 个 validate_* 函数 + 内部 helpers
```

**已实现的 L1/L2 数据结构**：
| 对象 | 说明 |
|------|------|
| `SourceRef` | 文件 + 定位，fact → 原始资料追溯链 |
| `RawSource` | L1 原始输入记录 |
| `ConflictCandidate` | 冲突候选值 |
| `Conflict` | 多来源不一致记录，resolution_policy 必须 manual |
| `Fact` | L2 核心：三段式 ID + fact_type 不变式 + revision + supersedes + status 五态 + review_status + value_domain |

## 测试基线
**tests/cdm/test_types.py** — 74 个测试，100% 通过：
- Registry: 7
- Fact ID: 14
- SourceRef: 5
- RawSource: 4
- Fact filled quantitative: 3
- Fact filled qualitative: 2
- Fact missing: 3
- Fact supersedes: 4
- Fact status invalid: 3
- Fact method/provenance/review_status: 3
- Fact confidence: 4
- Fact type failures: 4
- Fact JSON roundtrip: 3
- Conflict valid: 2
- Conflict invalid: 6
- Conflict JSON: 1
- Defect Facts: 6

## Step 7-B 核心约束（必须保留）
1. Fact ID 三段式严格冻结（`fact:<scope_class>.<instance_key>.<attribute>`）
2. fact_type 不变式（第一段 = scope_class）
3. scope_class 枚举含 defect
4. revision > 1 必须 supersedes
5. missing → value=null（禁止占位）
6. Conflict.resolution_policy 必须 manual
7. 验证层只做结构正确，跨 revision 链完整性延后到 Store 层
8. 不修改 02/03/04/05/07 设计文档

---

# 二、Step 7-C 最小范围

## 本轮实现（5 个 Domain Object）

| # | 对象 | 来源文档章节 | 核心字段 |
|---|------|-------------|----------|
| 1 | **Project** | §18 | id, fact_ids[], metadata |
| 2 | **Structure / Building** | §20 | id, fact_ids[], parent_id? (Building 可含多个 Structure) |
| 3 | **Component** | §21 | id, fact_ids[], parent_structure_id, defect_ids[] |
| 4 | **Defect** | §17 (v0.3.1) | id, fact_ids[], host_component_id (1:N 弱实体) |
| 5 | **Evidence** | §28 | id, raw_source_id, source_refs[], related_fact_ids[], related_object_ids[] |

## 明确不实现（用户已列出）
- InspectionItem、Measurement、MeasurementSet、Criterion、Evaluation、Conclusion、ConclusionRule
- Report IR、Parser、DOCX、PDF、Validator、LLM Adapter、Skill、Agent
- Redis、MQ、Microservice、Kubernetes

## 设计文档中 Domain Object 的定位（来自 §29）

> Domain Objects 不应该成为另一套独立的数据真相。
> 推荐：RawSource → Facts → Domain Objects
> Fact = 最小事实
> Domain Object = 事实的工程语义组织

所以 Domain Object 的核心是 **组织关系**，不是复制数据。

## 数据模型约束（来自文档）

1. **关系建立**：通过稳定 ID / Fact 引用，不复制 Fact 值
2. **Component 1:N Defect**：`Defect.host_component_id` 指向 Component.id
3. **Defect 是 scope_class 不是 scope_class**：crack 是 Defect 的一种实例，Defect 是 scope_class
4. **追溯链**：Evidence → RawSource / SourceRef

---

# 三、准备创建/修改的文件

## 新建文件

| 文件 | 内容 |
|------|------|
| `src/cdm/domain.py` | 5 个 Domain Object dataclass + 序列化 |
| `tests/cdm/test_domain.py` | Domain Object 单元测试 |

## 修改文件

| 文件 | 改动 |
|------|------|
| `src/cdm/__init__.py` | 新增 domain 对象导出 |
| `src/cdm/validate.py` | 新增 5 个 validate_* 函数 |

**不修改**：registry.py、id.py、types.py（保持 Step 7-B 纯净）
**不修改**：任何设计文档

---

# 四、Domain Object 与 Fact 的关系

## 关系图

```
Project.id = "PROJ01"
├── fact_ids[] = [
│     "fact:project.PROJ01.name",
│     "fact:project.PROJ01.id"
│ ]
│
├── Building.id = "B01"
│   ├── fact_ids[] = [
│   │     "fact:building.B01.id",
│   │     "fact:building.B01.area",
│   │     "fact:building.B01.floor_count"
│   │ ]
│   │
│   ├── Component.id = "K001"
│   │   ├── fact_ids[] = [
│   │   │     "fact:component.K001.id",
│   │   │     "fact:component.K001.concrete_strength"
│   │   │ ]
│   │   ├── defect_ids[] = ["C001"]
│   │   │
│   │   └── Defect.id = "C001"
│   │       ├── fact_ids[] = [
│   │       │     "fact:defect.C001.id",
│   │       │     "fact:defect.C001.width",
│   │       │     "fact:defect.C001.length",
│   │       │     "fact:defect.C001.pattern",
│   │       │     "fact:defect.C001.judgement"
│   │       │ ]
│   │       └── host_component_id = "K001"
│   │
│   Evidence.id = "E001"
│   ├── raw_source_id = "photo-034.jpg"
│   ├── source_refs = [
│   │     SourceRef("photo-034.jpg", "full-image")
│   │ ]
│   └── related_fact_ids = [
│         "fact:defect.C001.width",
│         "fact:defect.C001.pattern"
│       ]
```

## 核心设计原则

| 原则 | 实现方式 | 违反了会怎样 |
|------|----------|-------------|
| **不复制 Fact 值** | 只存 fact_id 字符串引用 | Domain Object 变成第二套真相源 |
| **关系显式化** | Component.defect_ids / Defect.host_component_id | 调用方必须反向遍历才能找到关联 |
| **Defect 是弱实体** | host_component_id 必填 | Defect 成为独立 scope_class（违反 D-027） |
| **追溯链完整** | Evidence 携带 raw_source_id + related_fact_ids | Evidence 与 L1/L2 脱节 |

## Domain Object vs Fact 的分工

```
Fact：                   Domain Object：
──────────────────────────────────────────
id（三段式事实标识）      id（领域对象标识，如 "K001"）
fact_type                fact_ids[]（引用哪些 Fact）
value（实际数值）        ❌ 不存值
status（filled/missing） ❌ 不存状态
source_refs              related_fact_ids[] / raw_source_id

→ 最小断言单元            → 工程语义组织
→ 可被 Report IR Ref 引用  → 可被 Pipeline 编排
→ 可被 Evaluator 判定      → 可被 Concluder 聚合
```

## 序列化策略

保持与 Step 7-B 一致：
- 每个 dataclass 实现 `to_dict()` / `from_dict()` / `to_json()`
- 内嵌的 SourceRef 复用已有序列化
- fact_ids / defect_ids 保持字符串列表

---

这个方案确认后我开始实现。
好，所有准备工作完成。以下是 Step 7-C 的完整实施方案。

---

# 一、当前实现基线

## Git 状态
- **分支**: master
- **HEAD**: `471d193` — Step 7-B commit
- **tag**: `step-7b-complete` + `design-baseline-m0`
- **测试**: 74 个测试 100% 通过

## 代码基线（Step 7-B 已封）
```
src/cdm/
├── types.py      # SourceRef / RawSource / Fact / Conflict / ConflictCandidate
├── id.py         # 三段式 FactID 严格 grammar
├── registry.py   # 固定枚举 + FactTypeRegistry + 默认注册表
├── validate.py   # 5 个 validate_* 函数
└── __init__.py   # 统一导出
```

## Step 7-B 核心约束（必须保留）
1. Fact ID 三段式严格冻结（`fact:<scope_class>.<instance_key>.<attribute>`）
2. fact_type 不变式（第一段 = scope_class）
3. scope_class 含 `defect`（v0.3.1）
4. revision > 1 必须 supersedes
5. missing → value = null（禁止占位填充）
6. Conflict.resolution_policy 必须 `manual`
7. 验证层只做结构正确，跨 revision 链完整性延后到 Store 层

---

# 二、Step 7-C 最小范围

## 本轮实现（5 个 Domain Object）

| # | 对象 | 文档来源 | 核心角色 |
|---|------|---------|---------|
| 1 | **Project** | §18 | 顶层容器，承载所有 Building |
| 2 | **Structure / Building** | §20 | 空间层级：Project → Building → Structure |
| 3 | **Component** | §21 | 工程构件（梁、柱、板），1:N Defect |
| 4 | **Defect** | §17 (v0.3.1) | 弱实体，依附 Component 存在；crack 是 Defect 实例 |
| 5 | **Evidence** | §28 | 证据：图片 / 原始记录 / 规范条文，追溯链 Evidence → RawSource |

## 明确不实现
所有 InspectionItem / Measurement / Criterion / Evaluation / Conclusion / ConclusionRule / Report IR / Parser / Validator / Skill / Agent 等 **本轮均不触碰**。

## Domain Object 在四层模型中的位置
```
L1 RawSource
L2 Facts              ← Step 7-B（已封）
L3 Domain Objects     ← 本轮 Step 7-C
L4 Report IR          ← 后续
```

## 关键设计约束（来自 §29 + D-027）
- **不复制 Fact 值**：Domain Object 只存 `fact_ids[]` 引用，值从 Fact 层读取
- **不存独立状态**：status / review_status 都在 Fact 层
- **关系显式化**：Component → Defect 1:N 用 `defect_ids[]` + `host_component_id` 双向
- **Defect 是弱实体**：必须携带 `host_component_id`
- **Evidence 可追溯**：携带 `raw_source_id` + `related_fact_ids[]`

---

# 三、准备创建/修改的文件

| 操作 | 文件 | 内容 |
|------|------|------|
| **新建** | `src/cdm/domain.py` | 5 个 Domain Object dataclass |
| **新建** | `tests/cdm/test_domain.py` | Domain Object 单元测试 |
| **修改** | `src/cdm/__init__.py` | 新增 domain 对象导出 |
| **修改** | `src/cdm/validate.py` | 新增 5 个 `validate_*` 函数 |

**不修改**：registry.py / id.py / types.py（保持 Step 7-B 纯净）
**不修改**：任何设计文档

---

# 四、Domain Object 与 Fact 的关系

## 关系图

```
Project("PROJ01")
├── fact_ids = [
│     "fact:project.PROJ01.name",
│     "fact:project.PROJ01.id"
│ ]
│
├── Building("B01")
│   ├── fact_ids = [
│   │     "fact:building.B01.id",
│   │     "fact:building.B01.area",
│   │     "fact:building.B01.floor_count"
│   │ ]
│   │
│   ├── Component("K001")
│   │   ├── fact_ids = [
│   │   │     "fact:component.K001.id",
│   │   │     "fact:component.K001.concrete_strength"
│   │   │ ]
│   │   ├── parent_building_id = "B01"
│   │   ├── defect_ids = ["C001"]
│   │   │
│   │   └── Defect("C001")
│   │       ├── fact_ids = [
│   │       │     "fact:defect.C001.id",
│   │       │     "fact:defect.C001.width",
│   │       │     "fact:defect.C001.length",
│   │       │     "fact:defect.C001.pattern",
│   │       │     "fact:defect.C001.judgement",
│   │       │     "fact:defect.C001.present"
│   │       │ ]
│   │       └── host_component_id = "K001"   ← 弱实体，必填
│   │
│   Evidence("E001")
│   ├── raw_source_id = "photo-034.jpg"      ← L1 追溯
│   ├── source_refs = [SourceRef("photo-034.jpg", "full-image")]
│   └── related_fact_ids = [
│         "fact:defect.C001.width",
│         "fact:defect.C001.pattern"
│       ]
```

## Domain Object vs Fact 分工

```
Fact：                       Domain Object：
──────────────────────────────────────────────
id（三段式事实标识）           id（领域对象标识，如 "K001"）
fact_type                    fact_ids[]（引用哪些 Fact）
value（实际数值）             ❌ 不存值 — 值从 Fact 层读取
status（filled/missing）       ❌ 不存状态 — 状态在 Fact 层
source_refs                  related_fact_ids[] / raw_source_id
                             host_component_id / parent_building_id

→ 最小断言单元                 → 工程语义组织
→ 可被 Report IR Ref 引用      → 可被 Pipeline 编排
→ 可被 Evaluator 判定          → 可被 Concluder 聚合
```

## 验证逻辑

每个 Domain Object 的 validate 函数检查：
| 对象 | 检查项 |
|------|--------|
| Project | id 非空；所有 fact_ids[] 是合法三段式 Fact ID |
| Building | id 非空；parent_project_id 非空；fact_ids[] 合法 |
| Component | id 非空；parent_building_id 非空；fact_ids[] 合法；所有 defect_ids 非空 |
| Defect | id 非空；**host_component_id 必填**（弱实体约束）；fact_ids[] 合法 |
| Evidence | id 非空；raw_source_id 非空；source_refs 列表合法；related_fact_ids 合法 |

## 序列化策略

与 Step 7-B 保持一致：`to_dict()` / `from_dict()` / `to_json()`。`fact_ids[]` 保持字符串列表，内嵌的 SourceRef 复用已有序列化。

---

以上是完整方案。确认后开始实现。
