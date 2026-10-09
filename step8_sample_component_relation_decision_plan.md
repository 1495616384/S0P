# Step 8 Sample/Component 关联设计决策 — 执行计划

> 类型：纯设计决策 Session（非 Fixture、非 Coding）
> 输入：用户 Step 8 决策任务书（2026-10-09）
> 输出：决策文档 + （有证据时）同步更新三份 Step 8 文档
> 硬约束：不改 Frozen Docs（02/03/04/05/07）、不改 registry.py、不写生产代码/测试/Fixture、不做破坏性 Git 操作

---

## 1. Summary

通过证据化核查判定 D-CONFLICT-001（Component–Sample 关联）与 D-STEP8-13（Conflict 裁决后转换）的真实状态：

- 核实 MeasurementSet / Measurement 在冻结 CDM 中的真实契约强度（正式 Schema vs 概念示例）
- 比较候选方案 A（复用 MeasurementSet）/ B（新增 Sample 对象）/ C（关系型 Fact）/ D（修改最小业务范围）
- 给出结论 A / B / C 之一；D-CONFLICT-001 与 D-STEP8-13 分别独立跟踪

**预期结论（待文档证据化确认）**：**结论 B** — 存在合规最小替代 Vertical Slice（缺陷/裂缝宽度切片），使用冻结的 `Defect.host_component_id` 弱实体关联表达"多个测量值属于同一 Component"，混凝土试块场景延期。D-CONFLICT-001 保留 OPEN 但重新界定为"非当前路径（Deferred）"；D-STEP8-13 保持 OPEN；D-045 不需要。

---

## 2. Current State Analysis（本次实际读取的核验结果）

### 2.1 MeasurementSet/Measurement 契约强度（关键发现）

| 来源 | 实际内容 | 判定 |
|---|---|---|
| `03` §23 (L1137-1157) | "表示某一个检测项目的一组测量数据"；示例 `K001=32.4 / K002=31.8 / K003=33.1`（**按构件键**的每构件单值） | 仅概念，无 ID/字段/关系定义 |
| `03` §24 (L1161-1177) | "一个具体测量值"；典型信息含 构件/测点/测值/单位/方法/时间/仪器；**"具体字段根据实际报告类型继续细化"** | 字段明确未冻结 |
| `03` §29 (L1428-1439) | Fact 组织为 Component / MeasurementSet / Measurement 的示意 | 组织示意，无 Schema |
| `03` §39 (L1744-1750) | `MeasurementSet → 程序计算 → average` | 概念流程 |
| `03` §8.1 (L345-355) | scope_class 含 `sample`；v0.3.1 明确 `measurement` **不是** scope_class（V3.1-5） | measurement_set 无主体类型 |
| `02` §6.1 (L283) / §17.2 (L753) | XLSX 标准化目标名义写 Measurement/MeasurementSet；M1 实际是 parsers/xlsx + facts/store | 方向性描述 |
| `04` §24 (L938-954) | `Facts → MeasurementSet → 确定性程序 → TableSpec` 表格生成推荐流程 | 流程概念，非 Step 8 对象契约 |
| `src/cdm/types.py` / `domain.py` | **无 MeasurementSet / Measurement 类** | 未实现 |
| `registry.py` | 无 measurement 相关类型 | 未注册 |

结论：MeasurementSet + Measurement **无稳定 ID、无字段类型、无必填字段、无关联约束** → 不是可用契约；使用它们必须先冻结新 Schema（D-045 级），不能作为关闭 D-CONFLICT-001 的现成证据。

### 2.2 Component / Sample / 关系（关键发现）

- `domain.py`：仅 Project / Structure / Component / Defect / Evidence 5 类；`Component` 字段 `id / parent_structure_id / defect_ids / fact_ids`（**无 sample 关联**）；**Defect 是弱实体，`host_component_id` 必填**（L178-210）；模块头 "Relations are explicit L3 fields (B-class), NOT Facts"
- `domain_validate.py`：Component.fact_ids 仅允许 component scope（L241-245）；Defect.fact_ids 仅允许 defect scope（L275-279）
- `registry.py`：`sample` 是合法 scope_class；仅注册 `sample.id`；**无 `sample.concrete_strength`、无任何 sample↔component 关联类型**；`defect.width`(length)、`defect.id`、`component.id`、`component.concrete_strength` 已注册
- `03` §17 (L988-1029)：骨架不含 Sample；crack 是 Defect 实例类型（`fact:defect.C001.width` 等，L1009-1014）；§16.4 裂缝示例为冻结文档自身范例
- `07_DECISIONS.md`：决策止于 D-027（D-041~D-044 未登记 → D-045 为可用下一编号）

### 2.3 D-STEP8-13 相关（Conflict 裁决链）

- `03` §34 (L1552-1569)：Conflict 身份键 = `fact_id`；字段 `candidates / resolved_by / resolution`
- `03` §35/§36：`resolution_policy="manual"` 为首阶段唯一值；无裁决后 Fact 转换定义
- `03` §15.3 / §15.5 (L729-799)：`conflict` 为横向来源分歧；`superseded` 仅属 revision 更正链，**"二者不得混用"**
- `types.py` L117-155：`Conflict.resolved_by / resolution` 均为 `Optional[str]`；resolution 内容语义未定义
- 判定：无冻结依据支撑"冲突候选→最终 Fact"的 status/revision/supersedes 转换 → **D-STEP8-13 保持 OPEN**

### 2.4 验证链接口（E2E 可行性）

- `05` §6.1-6.6：P0 数值一致性（显示精度 / 派生量 1e-9）、表内自洽、判定逻辑、数字锚定（含实例标识 token 绑定 `fact:component.K002.id`）、结论方向枚举、结论覆盖度 `covers[]`
- `src/eval/accuracy/validator_framework.py` L78-86：7 条 P0 = VALUE.CONSISTENCY / TABLE.INTERNAL / LOGIC.CONSISTENCY / ANCHOR.BINDING / CONCLUSION.DIRECTION / CONCLUSION.COVERAGE / EXPECTED.HIT
- `tests/eval/test_validator_framework.py`：`evaluate_case(trusted, system)`；trusted 由 EvaluationCase（GroundTruthBundle）构造
- 7 个 fixture 已存在，`inputs/` 全为空（仅 .gitkeep）；workspace 无任何 .xlsx/.csv

### 2.5 冻结 CDM 中"多个子测量值属于同一 Component"的既有合法路径

**唯一**：`Defect.host_component_id`（弱实体锚定）—— 多个缺陷 → 同一构件，可由 Compute 确定性分组。该路径为冻结设计（03 §17 + domain.py + domain_validate.py 三方一致）。

---

## 3. Proposed Changes（文件级）

### 3.1 新建 `docs/Step_8_Sample_Component_Relation_Design_Decision.md`

决策报告，结构：

1. §0 问题界定（D-CONFLICT-001 重述；与 D-STEP8-13 边界分离）
2. §1 证据核查：MeasurementSet / Measurement 逐条回答（真实职责、是否关联 Component、能否表达多测量值、与 Fact/SourceRef 关系、ID/字段/约束）
3. §2 差距矩阵（五分类：冻结明确规定 / 仅概念示例 / 已实现 / 未实现 / 需新增或修改）
4. §3 候选方案 A/B/C/D 比较表（合规性 / Schema 影响 / 实现复杂度 / 来源追溯 / Compute 确定性 / M3 价值）
5. §4 最终建议 = 结论 B：缺陷（裂缝宽度）切片详细设计（输入列 / 事实类型 / L3 关联 / Compute 分组与输出 / Criterion / Conclusion / IR / SystemOutput / 7-E 接口），并逐条论证未牺牲 Fact ID / 追溯 / 数值语义 / P0；混凝土试块场景 DEFERRED 及触发条件
6. §5 D-CONFLICT-001 重新界定（OPEN / 适用范围 / 非当前路径 / 未来关闭条件 → D-045 载体）
7. §6 D-STEP8-13 独立核查结论（保持 OPEN）
8. §7 D-045 状态（当前路径不需要；不创建提案）
9. §8 自检 12 项 + 最终状态块 + 文件修改清单

标注框：`STATUS = ADOPTED（本次决策）`仅适用于切片选型；实现细节仍标 OPEN/Proposed。

### 3.2 更新 `docs/Step_8_Design_Decision_Closure.md`

- 头部日期/状态行：加注 2026-10-09 决策（结论 B）
- §2 决策矩阵：D-STEP8-06/07/09 证据行更新（缺陷切片候选）；D-CONFLICT-001 行改 "OPEN（Deferred / 非当前路径）"
- §5 数据流：Compute 行去掉 "BLOCKED"，注明分组键 = `Defect.host_component_id`
- §6 Fixture Contract：业务场景候选改为缺陷（裂缝宽度）切片；混凝土试块标记 DEFERRED
- §10.1/§10.2：保留原证据，追加 2026-10-09 复核结论（MeasurementSet 概念级证据 + 替代路径裁定）
- §14：决策分布、门禁 4/8 更新（4 → PASS 替代方案；8 → 条件性 PASS）；OPEN 清单更新；最终状态块更新（Coding 仍 BLOCKED，但不再由 D-CONFLICT-001 阻塞）

### 3.3 更新 `docs/Step_8_Coding_Contract_v1.md`

- §0 状态声明：决策分布与门禁 4/8 更新
- §2 In Scope #11：fixture 候选改为缺陷切片
- §4 Fixture：列结构 / fact_type 映射候选改为 `component.id` / `defect.id` / `defect.width`（全已注册）+ 计算输出类型 Proposed
- §9 Compute：分组键合法性说明（host_component_id）；候选统计量 = max；输出 fact_type 仍标 Proposed（需 D-STEP8-06 授权）
- §10/§11：Criterion 候选（宽度限值，合成测试参数）+ Conclusion 候选更新
- §14 E2E 流程与 §19 Decision Register 同步
- 尾部状态块：D-CONFLICT-001 = OPEN (Deferred)

### 3.4 更新 `docs/Step_8_Fixture_Design_Sprint.md`

- 头部：阻塞状态更新（D-CONFLICT-001 已重新界定；可进入 Fixture Design Sprint）
- §1 Domain Direction：缺陷切片为主候选；混凝土试块列 DEFERRED
- §2/§3/§4：fact_type 映射 / Compute / Criterion 候选更新
- §5：Excel 结构候选改为 构件编号 | 缺陷编号 | 最大宽度(mm)（候选规模 ~10-12 行）
- §6 关闭条件 / §9 工作流步骤 0：更新（替代路径已裁定）

### 3.5 不做的（明确排除）

- 不创建 Fixture / Excel / meta.json
- 不修改 registry.py、src/**、tests/**
- 不修改 02/03/04/05/07
- 不创建 D-045 提案（结论 B 下不需要）
- 不执行 commit / push / reset / checkout

---

## 4. Assumptions & Decisions

| # | 假设/决策 | 依据 |
|---|---|---|
| 1 | 结论 B：缺陷（裂缝宽度）切片为合规最小替代 | Defect.host_component_id 为冻结 L3 关联；defect.width/id、component.id 已注册；03 §16.4/§17 裂缝即 Defect 实例 |
| 2 | 切片选型 = 本次决策；实现细节（列/统计量/输出类型授权）保持 OPEN/Proposed | 用户约束"不能把候选方案改写成已批准方案" |
| 3 | D-CONFLICT-001 保留 OPEN、重新界定为 Deferred（非当前路径、非 Coding-Blocking） | 用户授权"按其真实适用范围保留为非当前路径冲突"；不得仅换名宣称已解决 |
| 4 | D-STEP8-13 保持 OPEN | 冻结 CDM 无裁决后转换定义（§15.5/§34-36 证据） |
| 5 | D-045 不需要（当前路径） | 替代路径不触发 Frozen Schema 变更 |
| 6 | 计算输出需 1 个受控 Registry 扩展（Proposed，如 `component.max_defect_width`） | 与 `sample.concrete_strength` 同类：受控词表变更 + 需授权（D-STEP8-06），非 03 修改 |
| 7 | 三份 Step 8 文档允许更新 | 用户 §8 条件："通过合规最小替代方案解除当前阻塞" → 满足 |

---

## 5. Verification（完成后执行）

1. 重读本次全部修改文件（决策文档 + 3 份 Step 8 文档），逐项核对 12 条验收清单（用户 §9）
2. `git status` + `git diff --stat` 如实报告（注意未跟踪文件情形）
3. 确认：Frozen Docs / registry.py / src / tests / fixtures 均未修改
4. 按用户 §10 格式输出最终报告 A-F（证据核查 / 方案比较 / 最终建议 / 决策状态 / 文件修改 / 最终状态）

---

## 6. Stop Conditions（本次停止）

完成决策文档、文档同步与验证后立即停止：不创建 Fixture、不改 registry、不改 Frozen CDM、不进入 Coding、不自动开启下一阶段。