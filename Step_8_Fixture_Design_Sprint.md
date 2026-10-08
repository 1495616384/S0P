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

### 预候选 Fact Types

Registry 已注册的、fixture 可能用到的类型：

| fact_type | scope_class | 用途 | 是否需要？ |
|---|---|---|---|
| `component.concrete_strength` | component | 抗压强度测试值 | **必选** |
| `component.component_id` | component | 构件编号（scope_key） | **必选** |
| `component.test_block_id` | component | 试块编号（可选区分） | 可选 |
| `component.test_date` | component | 检测日期 | 可选 |

**不需要扩展 registry** — 全部已注册。

### 预候选单位

| 单位 | quantity_kind | 对应事实 |
|---|---|---|
| MPa | pressure | concrete_strength |

---

## 3. D-STEP8-07 Preliminary Direction（保持 OPEN）

### 预候选 Compute 需求

| 计算 | 场景 | 输入 | 输出 |
|---|---|---|---|
| **average** | 同一构件 3 个试块强度 → 平均值代表值 | 3 个 `component.concrete_strength` Fact | computed Fact（method="computed"） |
| min / max（可选） | 辅助判断异常值 | 同上 | computed Fact |
| count（可选） | 统计有效试块数 | 同上 | 整数 |

**预期 Step 8 只实现 `average`（核心）+ `max/min`（可选辅助）**。

---

## 4. D-STEP8-09 Preliminary Direction（保持 OPEN）

### 预候选 ConclusionRule（简化版）

GB/T 50081-2019 完整验收规则很复杂（有异常值判断、中值法等），Step 8 第一版简化为：

```
输入：
  - 构件强度平均值（from Compute.average）
  - 设计强度等级值（Criterion.value = 30 MPa）
  - 所有试块单独值 ≥ 0.88 × 设计值（附加条件）

规则（简化）：
  IF 平均值 ≥ 30 MPa AND 所有单个值 ≥ 0.88 × 30 (= 26.4 MPa)
  → Conclusion.status = "qualified"
  ELSE
  → Conclusion.status = "unqualified"
```

预候选 `rule_id = "concrete_strength_v1_simplified"`。

---

## 5. Excel 结构设计（核心任务）

### 目标：一份 sheet、构件一行、试块一列展开

**方案 A：构件为行、试块编号为列**（推荐）

```
Sheet1: 抗压强度检测记录

Row 1 (Header): 构件编号 | 设计强度等级(MPa) | 试块1强度(MPa) | 试块2强度(MPa) | 试块3强度(MPa) | 检验日期
Row 2 (Data):   K001     | 30               | 32.4            | 31.8            | 33.1            | 2026-09-15
Row 3 (Data):   K002     | 30               | 28.7            | 29.1            | 30.5            | 2026-09-15
...
```

**方案 B：试块为行**（备选，更细粒度）

```
Sheet1: 抗压强度检测记录

Row 1 (Header): 构件编号 | 试块编号 | 设计强度等级(MPa) | 抗压强度(MPa) | 检验日期
Row 2 (Data):   K001     | 试块1    | 30               | 32.4            | 2026-09-15
Row 3 (Data):   K001     | 试块2    | 30               | 31.8            | 2026-09-15
...
Row 4 (Data):   K002     | 试块1    | 30               | 28.7            | 2026-09-15
...
```

**方案 B 更好** — 更细粒度，Mapper 更清晰，Compute 需要按构件分组聚合。

### 需要的设计决策（fixture 创建时）

| # | 决策 | 选项 | 影响 |
|---|---|---|---|
| 1 | Sheet 数量 | 1（单 sheet） | 简化 Mapper |
| 2 | Header 行位置 | Row 1 | 简化 Parser |
| 3 | 数据行数 | 3-5 个构件（10-15 个试块） | 足够 Compute.average 验证 |
| 4 | 是否包含设计强度等级列 | 是（值为 30 MPa） | 可以用来生成 Criterion |
| 5 | 是否包含检验日期列 | 是 | source_refs / fact.date |
| 6 | 是否包含异常数据行 | 是（K002 平均 ~29.4 → unqualified） | 测试 ConclusionRule unqualified 路径 |
| 7 | 是否包含边界数据 | 可选（刚好 ≥ 0.88×30 = 26.4） | 测试 Criterion ≥ 边界 |
| 8 | 是否包含缺失值 | 可选（某试块值为空） | 测试 Fact Store missing 路径 |

---

## 6. D-STEP8-02 关闭条件

fixture 创建完毕后，必须满足：

| 条件 | 如何验证 |
|---|---|
| Excel 文件存在 | `glob tests/eval/fixtures/<case>/inputs/inspection.xlsx` → 1 个文件 |
| 至少 10 个数据行 | openpyxl 读行数 |
| 列头清晰、语义明确 | 人工检查（如"抗压强度" → concrete_strength） |
| 覆盖 qualified + unqualified 两种结果 | K001 合格，K002 不合格 |
| fixture 有 meta.json | `synthetic_business_fixture: true`, `domain: "concrete"` |

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

## 9. Fixture Design Sprint 工作流（预计 30 分钟内）

| # | 动作 | 产出 | 预计耗时 |
|---|---|---|---|
| 1 | 确定 Excel 结构（方案 B：试块为行） | 列头表（构件编号 / 试块编号 / 设计强度 / 抗压强度 / 日期） | 5 min |
| 2 | 创建 fixture 目录 | `tests/eval/fixtures/step8_concrete_strength_v1/` | 2 min |
| 3 | 写 inspection.xlsx | 3 个构件 × 3 个试块 = 9 行数据（7 合格 + 2 不合格） | 5 min |
| 4 | 写 meta.json | synthetic_business_fixture + domain + fact_types | 2 min |
| 5 | 写列头→fact_type 映射表（notes.md） | 明确每个列头对应 registry 中的哪个 fact_type | 5 min |
| 6 | 人工计算 ground_truth/facts.json | 每个试块 Fact + 每个构件 computed Fact（平均值） | 5 min |
| 7 | 回答 D-STEP8-03 Q1-Q6 | 记录在 fixture notes.md | 3 min |
| 8 | 关闭 5 个 OPEN 决策 | Decision Register 正式标记 CLOSED | 3 min |

**总计**：~30 分钟

---

*本 Brief 为 PRELIMINARY 方向，不构成决策。fixture 设计 Session 根据实际 Excel 结构重新验证。*
