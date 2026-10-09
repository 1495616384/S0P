# Step 8 Fixture Design Sprint — Brief

> **本 Brief 是什么**：Step 8 Coding Round 前的 Fixture Design Session 的具体执行 Brief
> **为什么需要**：Step 8 Decision Closure 仍有 OPEN 决策（D-STEP8-02/03/06/07/09/13）**及 1 个 Coding-Blocking Design Conflict（D-CONFLICT-001）**，全部依赖一份 synthetic_business_fixture。本 Brief 把"设计 fixture"从模糊任务变成可执行清单
> **前置**：Step 8 Decision Closure Report ✅（COMPLETE，状态 NEEDS DECISION）；本 Brief 于 2026-10-09 做跨文档一致性修复
> **目标**：设计并创建一份 **synthetic_business_fixture**（标记 synthetic_business_fixture = true），关闭 OPEN 决策并解决 D-CONFLICT-001，**之后**重新跑 READY FOR CODING 门禁
>
> **当前状态**：本 Brief 为 **PROPOSED**，**尚未执行**。fixture 未创建，registry 未扩展。D-CONFLICT-001 未解决前 **不得进入 Fixture Design Sprint**。

---

## 1. Domain Direction 预候选（保持 OPEN，但有起点）

### 建议 Domain：混凝土抗压强度检测

**理由**：
- `component.concrete_strength` 已在 FactTypeRegistry 注册（**注意：这不等于整条管线无需扩展 registry**——见 §2 与 §5，试块级 `sample.concrete_strength` 未注册）
- 混凝土试块检测是典型的"多测点→统计值→合格判定"管线，能完整覆盖 Step 8 的所有模块（Parser/Mapper/Store/Compute/Criterion/Conclusion/IR）
- 单位 MPa（压力）在 CDM QUANTITY_KINDS 范围内

### 业务背景

| 项目 | 值 | 说明 |
|---|---|---|
| 标准 | GB/T 50081-2019 | 混凝土物理力学性能试验方法 |
| 设计强度等级 | C30 | fcu,k = 30 MPa（标准值） |
| 单个构件试块数 | 3 个 | 平行试验 |
| 试块尺寸 | 100×100×100 mm | 非标准试件（**完整验收体系**需尺寸换算系数 0.95，Step 8 第一版不实现，见 §4） |
| 验收规则（简化） | 见 §4 预候选 ConclusionRule | Step 8 先简化规则 |

> 本表为**业务背景**，用于理解领域，**不构成 Step 8 冻结规则**。Step 8 第一版实际判定规则见 §4。

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

### 关于"一个 component 多个试块值"的 Design Conflict（保持 OPEN — Coding-Blocking）

**D-CONFLICT-001（登记于本次跨文档修复，未解决）：**

CDM 已注册的 `component.concrete_strength` 假设一个 component 有**一个**强度值。但 GB/T 50081-2019 要求每个构件做 **3 个平行试块**，产生 3 个强度值。

**Schema 可表达性结论（证据见 Decision Closure §10.2）：**
- `src/cdm/domain.py` 仅实现 Project / Structure / Component / Defect / Evidence，**无 Sample 类**；
- Component 字段仅 `id / parent_structure_id / defect_ids / fact_ids`，**无 sample 关联字段**；
- Component ↔ Sample 的归属关系在冻结 CDM 中**无法用普通 Fact 或冻结 L3 字段合法表达**；
- registry 仅注册 `sample.id`，未注册任何 sample→component 关联类型。

**因此 D-CONFLICT-001 未解决，属 Coding-Blocking。** 三路径仅为候选，均未批准：

| 路径 | 做法 | registry 变更 | 风险 |
|---|---|---|---|
| **P1（候选）** | 每个试块值 = 独立 `sample.concrete_strength` Fact | +1 类型（Proposed），且**仍需**表达 sample→component 归属（缺 `sample.host_component_id` 等关联能力） | 丢失 component→sample 层级关联，Compute 无法确定性分组 |
| **P2（候选）** | 每个 component 只存 computed 平均值 | 无需扩展 | 违反 CDM「原始值 + 原始单位」原则，丢失 raw 证据链 |
| **P3（候选）** | Excel 每行直接映射到 `component.concrete_strength`（同 fact_id 多值）→ Store 识别为 conflict | 无需扩展 | 同一 fact_id 多值会被 Store 标记为 conflict，语义混乱 |

**当前状态：三路径均保持 OPEN，暂不选型。** 上一版本曾"暂定选择 P1"，本次修复撤销该预设——在 Schema 归属关系未合法确定前，**不得**预先选定路径，更不得把 P1 描述为已确定契约。

> **不得静默采用 P1/P2/P3**；不得自行扩展 registry 或修改 CDM 让问题看似消失。若确认只能通过改 Frozen CDM Schema 解决 → 登记 Design Conflict、停止推进、走 D-045 决策，不得静默修改架构。

### 预候选单位

| 单位 | quantity_kind | 对应事实 |
|---|---|---|
| MPa | pressure | concrete_strength |

---

## 3. D-STEP8-07 Preliminary Direction（保持 OPEN）

### 预候选 Compute 需求

**状态：OPEN（D-STEP8-07），且分组依赖 D-CONFLICT-001（当前 BLOCKED）。**

| 计算 | 场景 | 输入 | 输出 |
|---|---|---|---|
| **average** | 同一构件 3 个试块强度 → 平均值代表值 | 按 component 分组的 3 个试块强度 Fact（**分组键来源未定，依赖 D-CONFLICT-001**） | computed Fact（method="computed"，scope_class=component；**输出 Fact ID / fact_type 未定**） |
| min / max（可选） | 辅助判断异常值 | 同上 | computed Fact |
| count（可选） | 统计有效试块数 | 同上 | 整数 |

**预期 Step 8 只实现 `average`（核心）+ `max/min`（可选辅助）**。

> **前置约束**：D-CONFLICT-001 未解决前，无法确定 Component ↔ Sample 归属关系，因而**无法定义分组键**。计算输出落在 Component 时，其 Fact ID / fact_type 必须符合 Frozen CDM；无法在不引入冲突的前提下确定时，D-STEP8-06/07 保持 OPEN。
> 不得仅凭 `sample.id` 的命名规则（如 S001~S003 属于 K001）推断 Component 归属。

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
  - design_strength（Criterion.value = 30 MPa，**本次合成用例的测试参数**，不代表完整混凝土工程验收标准）

规则（Step 8 第一版）：
  concrete_strength >= design_strength → Evaluation.status = "qualified"
  concrete_strength <  design_strength → Evaluation.status = "unqualified"
  Criterion.review_status != confirmed → Evaluation.status = "not_evaluable"
```

**预候选 rule_id = "step8_minimal_ge"**。

> **阈值声明**：`Criterion.value = 30 MPa` 只是本合成 Fixture 的测试参数，**不得**被表述为完整混凝土工程验收标准（如 GB 50107 的合格判定体系）。
>
> **状态枚举区分**：上面 `qualified / unqualified / not_evaluable` 是 **Evaluation.status**（05/Step 7-E 上下文）；`Conclusion.status` 是另一套（`qualified / unqualified / insufficient_evidence / missing`，见 Coding Contract §11）。两套枚举不得混用。
>
> **缺失 / 冲突输入的判定**：必须**优先服从 Frozen CDM 的 status 语义与 05_EVALUATION 契约**（如 missing/conflict Fact 是否可参与判定、P0 规则如何处置）。**缺少设计依据的分支不得随意决定**，应登记为 OPEN。

### 未来扩展（不在 Step 8 实现）

以下规则属于 GB/T 50081-2019 完整验收体系，**暂不进入 Step 8**：
- 所有试块单独值 ≥ 0.88 × 设计值（最低限值）
- 异常值判断 / 中值法（去掉最大值最小值）
- 试件尺寸换算系数（100mm 非标准件 × 0.95）
- 验收批规则（同批次 ≥ 10 组用统计方法，< 10 组用非统计方法）

这些规则将在后续 Step（如 Step 8.5 或 Step 9）中，在**有真实工程需求验证**后逐步引入。Step 8 的目标是打通端到端管线，不是完整规范算法实现。

---

## 5. Excel 结构设计（**候选 — 未批准，不得据此创建文件**）

### 候选结构（依赖 D-CONFLICT-001 关闭后方可实施）

**方案 B：试块为行**（细粒度，**若**采用 P1 路径则适用；P1 未批准）

> **重要**：以下 12 行结构与本 Brief 的 fact_type 映射均为**候选规模**。采用 12 行**不代表** sample→component 关联模型已合法，**不关闭** D-STEP8-02，也不关闭 D-CONFLICT-001。fixture 尚未创建，**不得声称文件已存在**。

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

**行数 = 12**（4 components × 3 measurements）——候选数据规模，统一解决原 D-STEP8-02 "至少 10 行" 与原 "3×3=9 行" 的矛盾。**12 行 ≠ 关联模型合法，不自动关闭 D-STEP8-02。**

### 4 个 Component 的测试目的

| Component | 3 个试块值 | 平均值 | 与 Criterion(30 MPa) 比较 | 测试目的 |
|---|---|---|---|---|
| **K001** | 32.4, 31.8, 33.1 | **32.43** | 32.43 ≥ 30 → **qualified** | 典型合格案例（高于设计值 8%） |
| **K002** | 28.7, 29.1, 30.5 | **29.43** | 29.43 < 30 → **unqualified** | 典型不合格案例（低于设计值 2%） |
| **K003** | 35.2, 34.8, 36.1 | **35.37** | 35.37 ≥ 30 → **qualified** | 高强度合格案例（高于设计值 18%） |
| **K004** | 29.8, 30.1, 30.6 | **30.17** | 30.17 ≥ 30 → **qualified（边界）** | 刚好合格的边界案例（高于设计值 0.6%） |

**覆盖：qualified × 3（含边界）+ unqualified × 1**，足够验证 ConclusionRule 的主要分支。

### Column → Fact Type 映射（**候选**；如实标注 registry 状态）

| Excel 列头 | 目标 fact_type | scope_class | registry 状态 | 备注 |
|---|---|---|---|---|
| 构件编号 | `component.id` | component | ✅ 已注册 | K001, K002, K003, K004 |
| 试块编号 | `sample.id` | sample | ✅ 已注册 | S001~S012 |
| 抗压强度(MPa) | `sample.concrete_strength` | sample | ❌ **未注册（Proposed）** | 需 registry 扩展 + 归属关系，均未批准 |

> **registry 状态如实说明**：`sample.concrete_strength` **未注册**，标记为 **Proposed**；`component.component_id` 也未注册（禁用，不得与 `component.id` 混用）。本映射**尚不可实施**。
>
> **registry 扩展的性质**：扩展 `sample.concrete_strength` 属于"在受控 Registry 补充允许类型"，本身不修改 03_CANONICAL_DATA_MODEL.md；但**仍须设计授权**（D-STEP8-06），**不得自行修改 registry.py**。若所需能力（如 sample→component 归属）无法通过补充受控类型获得，则必须改 Frozen CDM Schema → 登记 D-CONFLICT-001 / D-045，停止推进。
>
> **原始事实 vs 计算事实**：试块强度为**原始测量事实**（`sample.*`，method=measured/quoted）；Component 代表值若由 Compute 产生则为**计算事实**（`component.concrete_strength`，method=computed）。二者不得强行复用同一 fact_type 混同。

### Criterion 设计强度值

设计强度等级 C30 = 30 MPa 作为 fixture 级常量，编码为 Criterion.value = 30，**不进 CDM Fact Store**。

### 需要的设计决策（fixture 创建时）

| # | 决策 | 选项 | 影响 |
|---|---|---|---|
| 1 | Sheet 数量 | 1（单 sheet） | 简化 Mapper |
| 2 | Header 行位置 | Row 1 | 简化 Parser |
| 3 | 数据行数 | **候选固定 12**（4 构件 × 3 试块） | 候选规模，不关闭 D-STEP8-02 |
| 4 | registry 扩展 | **Proposed：+1 类型 `sample.concrete_strength`（需授权，D-STEP8-06）** | 依赖 D-CONFLICT-001；未批准前不得实施 |
| 5 | 是否包含设计强度等级列 | **不包含**（作为 Criterion 常量） | 简化 Mapper |
| 6 | 是否包含检验日期列 | **暂不包含**（registry 缺 sample.date） | 第一版聚焦核心管线 |
| 7 | 是否包含缺失值行 | **暂不包含**（所有 12 个试块有值） | 第一版先打通主干 |
| 8 | 记录 Design Conflict D-CONFLICT-001 | 已登记且**未解决（Coding-Blocking）** | 透明化层级表达缺口；未解决前不得进入 Sprint |

---

## 6. D-STEP8-02 关闭条件

fixture 创建完毕后，必须满足：

| 条件 | 如何验证 |
|---|---|
| **D-CONFLICT-001 已解决**（前置） | Sample→Component 归属关系在冻结 CDM 中有合法表达，或找到无需该关联的合规最小替代方案 |
| Excel 文件存在 | `glob tests/eval/fixtures/<case>/inputs/inspection.xlsx` → 1 个文件 |
| 数据行恰好 **12** | openpyxl 读行数 = 13（1 header + 12 data）；4 components × 3 measurements |
| 列头清晰、语义明确 | 人工检查（"构件编号" → component.id，"抗压强度(MPa)" → sample.concrete_strength） |
| 覆盖 qualified + unqualified + boundary 三种结果 | K001=合格、K002=不合格、K004=边界合格 |
| registry 扩展**已获授权并完成** | `sample.concrete_strength` 经授权后注册；**不得自行修改 registry.py** |
| fixture 有 meta.json | `synthetic_business_fixture: true`, `domain: "concrete"`, `registry_extensions: ["sample.concrete_strength"]` |

### Fixture 文件清单（**候选；Step 8 fixture 交付物仅限 inputs/ + meta.json + notes.md**）

```
tests/eval/fixtures/<case_id>/
├── inputs/
│   └── inspection.xlsx          ← synthetic_business_fixture（D-STEP8-02）
├── meta.json                   ← synthetic_business_fixture: true
└── notes.md                    ← fixture 设计说明（列头→fact_type 映射表）
```

> **OOS-15 边界**：`ground_truth/facts.json` 与 `expected_issues/issues.json` 属于 **Evaluation Case 设计**（Step 6 已冻结，OOS-15），**不是 Step 8 fixture 交付物**。Step 8 不创建、不计算 ground truth。不得以"人工计算期望事实"作为关闭决策的证据。

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

## 8. 关闭 OPEN 决策后的动作（**门禁 9 条**）

```
前提：D-CONFLICT-001 已解决（或找到合规最小替代方案）
      + Fixture 创建完成 + D-STEP8-02/03/06/07/09/13 全部 CLOSED
 ↓
重新跑 Step 8 Decision Closure Report §14 Final Status
 ↓
  READY FOR CODING 门禁检查（9 条，见 Coding Contract §0）：
    a. D-STEP8-02 ✓  Fixture 设计与来源性质明确
    b. D-STEP8-03 ✓  Mapper 确定性边界有证据支持
    c. D-STEP8-06 ✓  第一版 FactType 与 Registry 要求明确
    d. D-CONFLICT-001 ✓  Sample→Component 关联已合法确定，或找到合规最小替代
    e. D-STEP8-07 ✓  Compute 输入/分组/输出与缺失冲突语义明确
    f. D-STEP8-09 ✓  Criterion / ConclusionRule / 输出枚举明确
    g. D-STEP8-11 ✓  SystemOutput projection 无歧义可实现
    h. 无未解决的 Coding-Blocking Design Conflict
    i. 三份文档决策矩阵 / 接口 / 流程 / 风险 / 状态一致
  → 全部满足：STEP 8 DESIGN STATUS = READY FOR CODING
  → 任一不满足：STEP 8 DESIGN STATUS = NEEDS DECISION（当前状态）
 ↓
Step 8 Coding Contract v1 → 状态升级为 READY FOR CODING
 ↓
Step 8 Coding Round（新 Session）
```

> **D-STEP8-11 说明**：其"结构"部分早已 CLOSED（Coding Contract §19），本门禁不要求重复关闭，但要求 projection 无歧义。
> **当前**：D-CONFLICT-001 未解决 → 门禁 h 不通过 → **不得**进入 Fixture Design Sprint。

---

## 9. Fixture Design Sprint 工作流

| # | 动作 | 产出 | 前置/约束 |
|---|---|---|---|
| 0 | **解决 D-CONFLICT-001**（或确定合规最小替代方案） | 归属关系设计结论 | **硬前置；未完成不得开始后续步骤** |
| 1 | **申请 registry 扩展授权**：注册 `sample.concrete_strength`（必要时含归属类型） | 授权记录 + registry.py 变更 | 需设计授权；**不得自行修改 registry.py**（D-STEP8-06） |
| 2 | 确定 Excel 结构（候选方案 B：试块为行） | 列头表（构件编号 / 试块编号 / 抗压强度） | 依赖步骤 0 |
| 3 | 创建 fixture 目录 | `tests/eval/fixtures/step8_concrete_strength_v1/` | — |
| 4 | 写 inspection.xlsx | **候选 4 构件 × 3 试块 = 12 行**数据 | 12 行为候选规模 |
| 5 | 写 meta.json | synthetic_business_fixture + domain + registry_extensions | — |
| 6 | 写列头→fact_type 映射表（notes.md） | Component/sample 双 scope_class 结构说明 | 如实标注 registry 状态 |
| 7 | 定义 fixture 专用 display policy | 每 fact_type 的小数位 / 单位后缀显示（临时、仅本 Slice） | 见 Coding Contract §13；不得用 float 内部表示 |
| 8 | 回答 D-STEP8-03 Q1-Q6 | 记录在 fixture notes.md | — |
| 9 | 关闭 OPEN 决策（含 D-STEP8-13）并解决 D-CONFLICT-001 | Decision Register 正式标记 | 需证据，不得虚假关闭 |

> **不在 Step 8 Sprint 内**：`ground_truth/facts.json`、`expected_issues/issues.json` 的创建（OOS-15）。不得以人工计算 ground truth 充当关闭决策的证据。

**说明**：上述耗时（若有）为工作量估计，不作为承诺；实际以证据完备为准。

---

*本 Brief 为 PROPOSED 方向，**尚未执行**，不构成决策。fixture 设计 Session 需先解决 D-CONFLICT-001，再根据实际 Excel 结构重新验证。*

```text
STEP 8 DESIGN STATUS = NEEDS DECISION
STEP 8 FIXTURE DESIGN = NOT STARTED
STEP 8 CODING = BLOCKED
```

**D-CONFLICT-001 未解决前，不得进入 Fixture Design Sprint。**
