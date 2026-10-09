# Step 8 Fixture Design Sprint — Brief

> **本 Brief 是什么**：Step 8 Coding Round 前的 Fixture Design Session 的具体执行 Brief
> **为什么需要**：Step 8 Decision Closure 有 5 个 OPEN 决策（D-STEP8-02/03/06/07/09），全部依赖一份 synthetic_business_fixture。本 Brief 把"设计 fixture"从模糊任务变成可执行清单
> **前置**：Step 8 Decision Closure Report ✅（COMPLETE，状态 NEEDS DECISION）
> **目标**：设计并创建一份 **synthetic_business_fixture**（标记 synthetic_business_fixture = true），关闭 5 个 OPEN 决策，重新跑 READY FOR CODING 门禁

---

## 1. Domain Direction 预候选（保持 OPEN，但有起点）

### 建议 Domain：混凝土抗压强度检测

**理由**：
- `component.concrete_strength` 已在 FactTypeRegistry 注册（无需扩展 registry）
- 混凝土试块检测是典型的"多测点→统计值→合格判定"管线，能完整覆盖 Step 8 的所有模块（Parser/Mapper/Store/Compute/Criterion/Conclusion/IR）
- 单位 MPa（压力）在 CDM QUANTITY_KINDS 范围内

### 业务背景

| 项目 | 值 | 说明 |
|---|---|---|
| 标准 | GB/T 50081-2019 | 混凝土物理力学性能试验方法 |
| 设计强度等级 | C30 | fcu,k = 30 MPa（标准值） |
| 单个构件试块数 | 3 个 | 平行试验 |
| 试块尺寸 | 100×100×100 mm | 非标准试件（需尺寸换算系数 0.95） |
| 验收规则（简化） | 见 §7 预候选 ConclusionRule | Step 8 先简化规则 |

---

## 2. D-STEP8-06 Preliminary Direction（保持 OPEN）

### 已注册 Fact Types（registry.py L255-304 确认）

**必须严格对齐 registry，不得引用未注册类型：**

| fact_type | scope_class | registry 状态 | 用途 | Step 8 是否需要？ |
|---|---|---|---|---|
| `component.id` | component | ✅ 已注册（L276） | 构件标识（scope_key） | **必选** |
| `component.concrete_strength` | component | ✅ 已注册（L278） | 抗压强度测试值 | **必选** |
| `component.component_id` | component | ❌ **未注册** | — | **禁用**（见 Issue #5 修正） |
| `component.test_block_id` | component | ❌ **未注册** | — | **第一版不需要**（见下） |
| `component.test_date` | component | ❌ **未注册** | — | 可选（需扩展 registry 或暂不进 CDM） |
| `component.design_strength` | component | ❌ **未注册** | 设计强度等级值 | 可选（可作为 Criterion 常量不进 CDM） |

### 设计强度等级的处理（简化方案）

设计强度等级（C30 → 30 MPa）是 fixture 级常量，**第一版不作为 CDM Fact**，直接编码为 Criterion.value = 30。理由：
- 设计强度来自设计文件，不是现场检测产物，业务上就不属于"检测事实"
- 简化 Fact Store 结构，避免为 fixture 单独扩展 registry

### 关于"一个 component 多个试块值"的 Design Conflict（新登记）

**D-CONFLICT-001（本次 Session 发现）：**

CDM 已注册的 `component.concrete_strength` 假设一个 component 有**一个**强度值。但 GB/T 50081-2019 要求每个构件做 **3 个平行试块**，产生 3 个强度值。当前 CDM 没有显式表达"一个 component 对多个 sample 测量值"的层级关系。

三种处理路径：

| 路径 | 做法 | registry 变更 | 风险 |
|---|---|---|---|
| **P1：每个试块值 = 独立 sample Fact** | 扩展 `sample.concrete_strength`（registry 当前只有 `sample.id`） | +1 类型 | 简单，但丢失 component→sample 层级关联（registry 缺 `sample.host_component_id`） |
| **P2：每个 component 只存 computed 平均值** | 原始试块值只在 Parser/Compute 层存在，Mapper→Store 只映射平均值 | 无需扩展 | 违反 CDM "原始值 + 原始单位" 原则，丢失 raw 证据链 |
| **P3：保持方案 A 展开列，单 component 多值在 Store 层聚合** | Excel 每行列头直接映射到 `component.concrete_strength`（同 fact_id 多值）→ Store 层识别为 conflict 或多个事实 | 无需扩展 | 语义混乱，同一 fact_id 多值会被 Store 标记为 conflict |

**Step 8 第一版暂定选择 P1**（最简单明确，registry 扩展 1 个类型 + 可选扩展 host 关联），但需要在 fixture 创建前确认并登记。**不得静默采用 P2 或 P3**。

### 预候选单位

| 单位 | quantity_kind | 对应事实 |
|---|---|---|
| MPa | pressure | concrete_strength |

---

## 3. D-STEP8-07 Preliminary Direction（保持 OPEN）

### 预候选 Compute 需求

| 计算 | 场景 | 输入 | 输出 |
|---|---|---|---|
| **average** | 同一构件 3 个试块强度 → 平均值代表值 | 按 component 分组的 3 个 `sample.concrete_strength` Fact（P1 路径） | computed Fact（method="computed"，scope_class=component） |
| min / max（可选） | 辅助判断异常值 | 同上 | computed Fact |
| count（可选） | 统计有效试块数 | 同上 | 整数 |

**预期 Step 8 只实现 `average`（核心）+ `max/min`（可选辅助）**。

---

## 4. D-STEP8-09 Preliminary Direction（保持 OPEN）

### 业务背景（仅供理解，不是 Step 8 冻结规则）

| 项目 | 值 | 说明 |
|---|---|---|
| 完整标准 | GB/T 50081-2019 + GB 50107 | 混凝土强度检验涉及异常值判断、中值法、0.88× 最小限值、试件尺寸换算系数（100mm 非标准件 0.95 等）、验收批规则等多条复杂逻辑 |
| 完整规则复杂度 | 高 | 不适合 Step 8 第一版冻结 |

### Step 8 第一版实际判定规则（最小、明确、可独立验证）

**业务背景 ≠ Step 8 冻结规则。** 本 Session 显式把两者分开：

```
输入：
  - concrete_strength（Component 的代表值 — 来自 Compute.average 或直接值）
  - design_strength（Criterion.value = 30 MPa，fixture 常量）

规则（Step 8 第一版）：
  concrete_strength >= design_strength → Evaluation.status = "qualified"
  concrete_strength <  design_strength → Evaluation.status = "unqualified"
  Criterion.review_status != confirmed → Evaluation.status = "not_evaluable"
```

**预候选 rule_id = "step8_minimal_ge"**。

### 未来扩展（不在 Step 8 实现）

以下规则属于 GB/T 50081-2019 完整验收体系，**暂不进入 Step 8**：
- 所有试块单独值 ≥ 0.88 × 设计值（最低限值）
- 异常值判断 / 中值法（去掉最大值最小值）
- 试件尺寸换算系数（100mm 非标准件 × 0.95）
- 验收批规则（同批次 ≥ 10 组用统计方法，< 10 组用非统计方法）

这些规则将在后续 Step（如 Step 8.5 或 Step 9）中，在**有真实工程需求验证**后逐步引入。Step 8 的目标是打通端到端管线，不是完整规范算法实现。

---

## 5. Excel 结构设计（核心任务）

### 目标：方案 B 结构 + P1 路径（每个试块 = 独立 sample Fact）

**方案 B：试块为行**（保留原始设计，更细粒度，适合 P1 路径）

```
Sheet1: 抗压强度检测记录

Row 1 (Header): 构件编号 | 试块编号 | 抗压强度(MPa)
Row 2 (Data):   K001     | S001     | 32.4
Row 3 (Data):   K001     | S002     | 31.8
Row 4 (Data):   K001     | S003     | 33.1
Row 5 (Data):   K002     | S004     | 28.7
Row 6 (Data):   K002     | S005     | 29.1
Row 7 (Data):   K002     | S006     | 30.5
Row 8 (Data):   K003     | S007     | 35.2
Row 9 (Data):   K003     | S008     | 34.8
Row 10 (Data):  K003     | S009     | 36.1
Row 11 (Data):  K004     | S010     | 29.8
Row 12 (Data):  K004     | S011     | 30.1
Row 13 (Data):  K004     | S012     | 30.6
```

**行数 = 12**（4 components × 3 measurements）——本次 Session 修正，统一解决原 D-STEP8-02 "至少 10 行" 与原 "3×3=9 行" 的矛盾。

### 4 个 Component 的测试目的

| Component | 3 个试块值 | 平均值 | 与 Criterion(30 MPa) 比较 | 测试目的 |
|---|---|---|---|---|
| **K001** | 32.4, 31.8, 33.1 | **32.43** | 32.43 ≥ 30 → **qualified** | 典型合格案例（高于设计值 8%） |
| **K002** | 28.7, 29.1, 30.5 | **29.43** | 29.43 < 30 → **unqualified** | 典型不合格案例（低于设计值 2%） |
| **K003** | 35.2, 34.8, 36.1 | **35.37** | 35.37 ≥ 30 → **qualified** | 高强度合格案例（高于设计值 18%） |
| **K004** | 29.8, 30.1, 30.6 | **30.17** | 30.17 ≥ 30 → **qualified（边界）** | 刚好合格的边界案例（高于设计值 0.6%） |

**覆盖：qualified × 3（含边界）+ unqualified × 1**，足够验证 ConclusionRule 的主要分支。

### Column → Fact Type 映射（严格对齐 registry）

| Excel 列头 | 目标 fact_type | scope_class | registry 状态 | 备注 |
|---|---|---|---|---|
| 构件编号 | `component.id` | component | ✅ 已注册 | K001, K002, K003, K004 |
| 试块编号 | `sample.id` | sample | ✅ 已注册 | S001~S012 |
| 抗压强度(MPa) | `sample.concrete_strength` | sample | ❌ **需扩展** | P1 路径需要 +1 registry 条目 |

> **registry 扩展说明**：扩展 `sample.concrete_strength` 不修改 03_CANONICAL_DATA_MODEL.md（registry 是独立的受控词表，硬编码在 registry.py 中），属于允许的 Step 8 工作范围。

### Criterion 设计强度值

设计强度等级 C30 = 30 MPa 作为 fixture 级常量，编码为 Criterion.value = 30，**不进 CDM Fact Store**。

### 需要的设计决策（fixture 创建时）

| # | 决策 | 选项 | 影响 |
|---|---|---|---|
| 1 | Sheet 数量 | 1（单 sheet） | 简化 Mapper |
| 2 | Header 行位置 | Row 1 | 简化 Parser |
| 3 | 数据行数 | **固定 12**（4 构件 × 3 试块） | 本次 Session 统一 |
| 4 | registry 扩展 | +1 类型 `sample.concrete_strength` | P1 路径必需 |
| 5 | 是否包含设计强度等级列 | **不包含**（作为 Criterion 常量） | 简化 Mapper |
| 6 | 是否包含检验日期列 | **暂不包含**（registry 缺 sample.date） | 第一版聚焦核心管线 |
| 7 | 是否包含缺失值行 | **暂不包含**（所有 12 个试块有值） | 第一版先打通主干 |
| 8 | 记录 Design Conflict D-CONFLICT-001 | 本次 Session 已登记 | 透明化层级表达缺口 |

---

## 6. D-STEP8-02 关闭条件

fixture 创建完毕后，必须满足：

| 条件 | 如何验证 |
|---|---|
| Excel 文件存在 | `glob tests/eval/fixtures/<case>/inputs/inspection.xlsx` → 1 个文件 |
| 数据行恰好 **12** | openpyxl 读行数 = 13（1 header + 12 data）；4 components × 3 measurements |
| 列头清晰、语义明确 | 人工检查（"构件编号" → component.id，"抗压强度(MPa)" → sample.concrete_strength） |
| 覆盖 qualified + unqualified + boundary 三种结果 | K001=合格、K002=不合格、K004=边界合格 |
| registry 扩展完成 | `sample.concrete_strength` 已注册到 registry.py |
| fixture 有 meta.json | `synthetic_business_fixture: true`, `domain: "concrete"`, `registry_extensions: ["sample.concrete_strength"]` |

### Fixture 文件清单

```
tests/eval/fixtures/<case_id>/
├── inputs/
│   └── inspection.xlsx          ← synthetic_business_fixture（D-STEP8-02）
├── ground_truth/
│   └── facts.json              ← 预期 Facts（人工计算）
├── expected_issues/
│   └── issues.json             ← 可选（Step 6 已冻结）
├── meta.json                   ← synthetic_business_fixture: true
└── notes.md                    ← fixture 设计说明（列头→fact_type 映射表）
```

### 推荐 case_id

建议命名为 `step8_concrete_strength_v1`（清晰表达内容和版本）。

---

## 7. D-STEP8-03 Q1-Q6 回答目标

fixture 创建后，Mapper 设计阶段需要正式回答：

| # | 问题 | fixture 给出答案后 |
|---|---|---|
| Q1 | 列名是否足够稳定？ | "是"（如"抗压强度"直接 match 到 registry）→ **确定性** |
| Q2 | 是否需要 sheet/row/col context？ | "不需要"（单 sheet，直接列名匹配）→ **确定性** |
| Q3 | 是否存在同义字段？ | fixture 列头都是标准工程术语 → 可能无同义 → **确定性** |
| Q4 | 是否存在无法无歧义映射？ | fixture 简单明了 → 预计无 → **确定性** |
| Q5 | 确定性规则有哪些？ | 列名精确匹配 + 受控同义词表（如后续有扩展） |
| Q6 | 哪些是不确定性？ | fixture 无复杂语义 → 预计无 |

### 预期结果：**结果 A（全部确定性）**

fixture 设计越简单，Mapper 越能确定性。但必须如实回答——如果 fixture 列名混乱，必须降级到结果 B 或 C。

---

## 8. 关闭 OPEN 决策后的动作

```
Fixture 创建完成 + D-STEP8-02=D-STEP8-09=D-STEP8-11=CLOSED
 ↓
重新跑 Step 8 Decision Closure Report §14 Final Status
 ↓
  READY FOR CODING 门禁检查：
    a. D-STEP8-02 ✓（CLOSED）
    b. D-STEP8-03 ✓（CLOSED 确定性程度）
    c. D-STEP8-06 ✓（CLOSED）
    d. D-STEP8-07 ✓（CLOSED）
    e. D-STEP8-09 ✓（CLOSED）
    f. Design Conflict: 0 ✓
  → STEP 8 DESIGN STATUS = READY FOR CODING
 ↓
Step 8 Coding Contract v1 → 状态升级为 READY FOR CODING
 ↓
Step 8 Coding Round（新 Session）
```

---

## 9. Fixture Design Sprint 工作流

| # | 动作 | 产出 | 预计耗时 |
|---|---|---|---|
| 1 | 确认 registry 扩展：注册 `sample.concrete_strength` | registry.py +1 类型 | 5 min |
| 2 | 确定 Excel 结构（方案 B：试块为行） | 列头表（构件编号 / 试块编号 / 抗压强度） | 3 min |
| 3 | 创建 fixture 目录 | `tests/eval/fixtures/step8_concrete_strength_v1/` | 2 min |
| 4 | 写 inspection.xlsx | **4 构件 × 3 试块 = 12 行**数据（K001 合格、K002 不合格、K003 高强度合格、K004 边界合格） | 5 min |
| 5 | 写 meta.json | synthetic_business_fixture + domain + registry_extensions | 2 min |
| 6 | 写列头→fact_type 映射表（notes.md） | Component/sample 双 scope_class 结构说明 | 5 min |
| 7 | 人工计算 ground_truth/facts.json | 12 个 sample Fact + 4 个 component computed Fact（平均值） | 8 min |
| 8 | 回答 D-STEP8-03 Q1-Q6 | 记录在 fixture notes.md | 3 min |
| 9 | 关闭 5 个 OPEN 决策 | Decision Register 正式标记 CLOSED | 3 min |

**总计**：~36 min（+ registry 扩展 5 min）

---

*本 Brief 为 PRELIMINARY 方向，不构成决策。fixture 设计 Session 根据实际 Excel 结构重新验证。*
