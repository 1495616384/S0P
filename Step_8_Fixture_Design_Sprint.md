# Step 8 Fixture Design Sprint — Brief

> **本 Brief 是什么**：Step 8 Coding Round 前的 Fixture Design Session 的执行清单（Brief）。
> **版本**：**v2**（缺陷判定切片方向修订版，2026-10-09）——替代 v1 混凝土方向版本（v1 关键内容压缩保留于附录 A，Deferred 参考）。
> **执行状态**：**COMPLETE**（2026-10-09 执行完毕；§8 关闭条件 8/8 ✅、§9 工作流 8/8 ✅、回归 412 passed、9/9 门禁 PASS；见尾部状态块）。
> **修订依据**：`Step_8_Sample_Component_Relation_Design_Decision.md` 结论 B（2026-10-09 裁定：合规最小替代 = 缺陷判定切片；D-CONFLICT-001 保持 OPEN / Deferred / 非当前路径，未解决）；S0P 收口任务阶段 2（关闭 Step 8 未决设计并筹备 Fixture Design）。
> **目标**：设计并创建一份 **synthetic_business_fixture**（标记 `synthetic_business_fixture = true`）→ 逐项证据化关闭 D-STEP8-02 / 03 / 06 / 07 / 09；形成 D-STEP8-13 正式处置；随后重跑 9 条 READY FOR CODING 门禁。
> **不做**：不创建 ground_truth / expected_issues / known_issues（OOS-15）；不修改 Frozen Docs（02/03/04/05/07）/ `registry.py` / `src/eval/**`；不进入 Coding；不创建生产代码。

---

## 0. v1 → v2 变更差异

| # | v1（混凝土方向，Deferred） | v2（缺陷判定切片，当前） | 依据 |
|---|---|---|---|
| 1 | domain=concrete；4 构件 × 3 试块 = 12 行 | domain=defect_inspection；3 构件 × 2 缺陷 = 6 行 | 决策报告 §4.2 |
| 2 | 需要 `sample.concrete_strength`（未注册，Proposed）+ 归属表达（BLOCKING） | 全部列命中已注册类型；**零 Registry 变更** | `registry.py` L272-294 |
| 3 | Compute = average（分组键依赖 D-CONFLICT-001） | 第一版统计 Compute **明确排除（∅）**；unit registry 支撑 mm/m | 决策报告 §4.4-①；本 Brief §5 |
| 4 | 判定链 = 试块均值 vs 设计值（含统计 Compute） | 判定链 = 逐缺陷宽度 vs 合成限值；构件级聚合（分组键 = 冻结 L3 字段） | 决策报告 §4.2 |
| 5 | 部件↔试块归属无合法载体 | 构件↔缺陷归属 = `Defect.host_component_id`（冻结必填 L3 字段） | `domain.py` L178-210 |
| 6 | case_id 未定（建议名 step8_concrete_strength_v1 未采用） | case_id = `case_step8_defect_v1` | 本 Brief §2 |

---

## 1. Domain Direction（当前：缺陷判定切片）

**业务链条**：构件缺陷检查记录（Excel）→ 逐缺陷宽度判定（Criterion）→ 构件级方向聚合（按 `host_component_id` 分组）→ 全局方向（Conclusion）→ Report IR → SystemOutput。

**能力覆盖对照**（如实标注）：

| 能力 | 是否覆盖 | 说明 |
|---|---|---|
| Parser / Mapper / Store / Criterion / Conclusion / IR Builder / IR Adapter | ✅ 全链 | 与混凝土方向相同的 Proposal 模块集 |
| 统计 Compute（avg / max / min / count） | ❌ **显式排除** | 需未注册的聚合输出 fact_type；延后（见 §5） |
| 单位换算支撑（mm / m） | ✅ | `unit_registry`（同量纲不同单位，验证单位语义） |
| L3 归属关系 | ✅ | `Defect.host_component_id`（冻结必填字段），由同行 row context 确定性导出 |
| P0-2（表内自洽 TABLE.INTERNAL） | ❌ NOT_EVALUABLE | 明细表无合计/统计语义（v11 语义）；其余 6 条 P0 正向验证 |

---

## 2. Fixture 规格（synthetic_business_fixture）

| 项 | 值 |
|---|---|
| case_id | `case_step8_defect_v1` |
| 目录 | `tests/eval/fixtures/case_step8_defect_v1/` |
| 交付物（仅此三项） | `inputs/inspection.xlsx`、`meta.json`、`notes.md` |
| Sheet | `构件缺陷检查记录`（单 sheet） |
| Header 行 | 第 1 行 |
| 数据行数 | 6（数据行 2-7） |
| 列（6） | 楼栋编号 \| 构件编号 \| 缺陷编号 \| 最大宽度(mm) \| 长度(m) \| 形态描述 |
| 实例 key 规则 | 构件 K001-K003；缺陷 D001-D006（全局唯一）；楼栋 B01 |
| 单位 | mm（宽度）/ m（长度）→ 两者 quantity_kind = `length` |

**数据（6 行）**：

| 楼栋编号 | 构件编号 | 缺陷编号 | 最大宽度(mm) | 长度(m) | 形态描述 | 判定（宽度 ≤ 0.30 → qualified） |
|---|---|---|---|---|---|---|
| B01 | K001 | D001 | 0.12 | 2.30 | 斜向 | qualified |
| B01 | K001 | D002 | 0.30 | 1.80 | 横向 | qualified（**边界：= 限值**） |
| B01 | K002 | D003 | 0.35 | 3.20 | 斜向 | **unqualified（超限）** |
| B01 | K002 | D004 | 0.18 | 2.60 | 网状 | qualified |
| B01 | K003 | D005 | 0.10 | 1.20 | 纵向 | qualified |
| B01 | K003 | D006 | 0.22 | 0.90 | 斜向 | qualified |

**实例 key 的冻结约束（tokenizer 兼容性）**：冻结 tokenizer 的 `IDENTIFIER_TOKEN_RE` 仅识别前缀 `ST|[KCMPSBD]` + 数字 + 可选 `-数字` 段（`src/eval/accuracy/tokenizer.py` L31-39）；带连字符的 id（如 `K001-D001`）会被拆为两个 token 且无法与 instance_key 匹配（BIND-7 约束）→ **fixture 禁止连字符 id**。B/K/D 前缀 + 纯数字全部命中。

**显示策略（fixture-specific display policy，临时、非 TemplateSpec）**：
- `defect.width` → 2 位小数 + `" mm"`；`defect.length` → 2 位小数 + `" m"`；单位与 `fact.unit` 一致；
- 标识（楼栋/构件/缺陷）→ 原文显示（B01 / K001 / D001）；`defect.pattern` → 受控词表原文；
- 必须与 trusted `FactDisplaySpecSnapshot.decimals` 一致（INV-19）；待 coding 阶段按冻结 tokenizer 实测校准。

**形态描述受控词表（fixture 级声明）**：`斜向 / 横向 / 纵向 / 网状`（对应 registry `defect.pattern` description 的 diagonal / transverse / longitudinal / map-crack 工程术语；本 fixture 显式声明为该列的合法值域）。

**IR 文本 token 约束（设计声明）**：段落/单元格文本不得出现未锚定的数值 token；标识 token 必须绑定对应 `fact_id`（如 `K002` → `fact:component.K002.id`）。结论段文本仅用受控短句（避免数值 token）。

---

## 3. fact_type 映射 → D-STEP8-06 关闭依据（全部已注册，零 Registry 变更）

| Excel 列 | fact_type | scope_class | quantity_kind | registry 证据 | fact_id 示例 |
|---|---|---|---|---|---|
| 楼栋编号 | `building.id` | building | qualitative | `registry.py` L272-273 | `fact:building.B01.id` |
| 构件编号 | `component.id` | component | qualitative | `registry.py` L276-277 | `fact:component.K001.id` |
| 缺陷编号 | `defect.id` | defect | qualitative | `registry.py` L283-284 | `fact:defect.D001.id` |
| 最大宽度(mm) | `defect.width` | defect | length | `registry.py` L285-286 | `fact:defect.D001.width` |
| 长度(m) | `defect.length` | defect | length | `registry.py` L287-288 | `fact:defect.D001.length` |
| 形态描述 | `defect.pattern` | defect | qualitative | `registry.py` L289-290 | `fact:defect.D001.pattern` |

**不变式核对**：fact_id 三段式 `fact:<scope_class>.<instance_key>.<attribute>`；fact_type `<scope_class>.<attribute>` 与 fact_id 首段/末段一致（`registry.validate_id_against_fact_type`）；6/6 已注册；无任何 Proposed / 未注册类型。

**L3 组织（确定性导出，不得由命名推断）**：
- `Structure(id="B01", structure_type="building", project_id=<切片声明>)`；
- `Component(id="K00x", parent_structure_id="B01")`；
- `Defect(id="D00x", host_component_id="K00x")` —— 归属由**同行 row context**（该缺陷的 fact `source_refs` 与该构件 id 单元格同 sheet 同行）确定性导出；**禁止**由 `D00x` 命名或排序推断归属。

---

## 4. 判定 / 结论设计 → D-STEP8-09 关闭依据

**Criterion（1 条；合成测试参数，明确标注）**：

| 字段 | 值 |
|---|---|
| criterion_id | `criterion-step8-defect-width-01` |
| rule | `defect.width <= 0.30` → qualified；`> 0.30` → unqualified |
| value / unit / quantity_kind | 0.30 / mm / length |
| review_status | confirmed |
| source_refs | 标注为「本切片合成测试参数」（**非**规范条文） |

> ⚠️ **0.30 mm 仅为本 Fixture 的合成测试参数**，**不得**表述为工程验收规范限值（不引用任何规范条款号）。

**Evaluation**：逐缺陷（6 条）→ `qualified / unqualified / not_evaluable`（冻结枚举）。预期：D001/D002/D004/D005/D006 = qualified（D002 为边界等值合格）、D003 = unqualified。

**ConclusionRule `step8_defect_slice_v1`（最小聚合）**：

1. **逐构件聚合**（分组键 = `host_component_id`，冻结 L3 字段）：任一缺陷 unqualified → 构件 unqualified；否则任一 not_evaluable → insufficient_evidence；否则全部 qualified → qualified。
2. **全局方向**：任一构件 unqualified → unqualified；否则任一 not_evaluable → insufficient_evidence；否则全部 qualified → qualified；输入为空 → missing。
3. 记录 `rule_id` + 全部 `input_evaluation_ids`；`covers[]` = 全部 6 条 Evaluation id（04 §42.1 / 05 §6.6）。
4. **优先级声明**：unqualified 优先于 not_evaluable（单项否决的保守语义）。

**预期结果**：K001 qualified；K002 unqualified；K003 qualified；全局 `direction = "unqualified"`。

**direction 枚举值域（本切片声明）**：`qualified / unqualified / insufficient_evidence / missing` —— Step 7-D 不限定全局结论枚举（`src/eval/types.py` L197-199 仅查非空）；本切片声明此 4 值为本切片合法值域，供 trusted 与 `report_ir.conclusion.direction` 一致比对（P0-5）。

**not_evaluable 分支的 fixture 覆盖说明**：本 fixture 只有 1 条 confirmed Criterion → 数据不含 not_evaluable；该分支由 coding 阶段单元测试覆盖（不以 fixture 数据冒充）。

---

## 5. Compute 裁定 → D-STEP8-07 关闭依据

- **第一版统计 Compute set = ∅（明确排除）**。理由：最小判定链只需逐缺陷比较（Criterion）+ Evaluation 聚合（ConclusionRule，非统计 Compute）；任何统计/聚合输出 fact_type 均未注册，引入即需 Registry 授权 → 不在第一版。
- `src/compute/unit_registry.py`（第一版唯一 compute 模块）：Unit → quantity_kind 映射（mm→length、m→length）+ 同量纲换算支撑（供 Criterion 比较与显示一致性校验）；确定性纯函数。
- `src/compute/calculate.py` / `compute_statistic()`：**第一版不创建 / 不实现**（避免空壳模块）；统计能力（avg/max/min/count，分组键必须为冻结 L3 字段）延后——出现真实统计需求时按 D-STEP8-07 重新开启。
- **缺失 / 冲突语义（compute 域）= N/A**（无统计输入聚合）；Fact 级缺失/冲突的判定处置归 Store 闸门与 Criterion 冻结语义，本切片不臆断分支。
- **S8-S-06 范围重化**：从「Compute 统计结果 vs expected truth」→「**单位换算 / 量纲一致性** vs 独立 expected truth（统计计算延后，如实标注）」。

---

## 6. D-STEP8-03 Q1-Q6 回答目标（正式回答记录于 fixture `notes.md`）

| # | 问题 | 本 fixture 的答案 | 判定 |
|---|---|---|---|
| Q1 | 列名是否足够稳定、可直接匹配 fact_type？ | 是——6 个固定列头，逐一精确匹配 §3 映射表；无变体列名 | 确定性 |
| Q2 | 是否需要 sheet/row/col context？ | 需要**行级 context**：同行 6 列联合关系用于 L3 归属（host_component_id）与实例分组；单 sheet 无枚举歧义。行级 context 来自 `source_refs` 位置（确定性结构信息，非语义推断） | 确定性 |
| Q3 | 是否存在同义字段？ | 本 fixture 无（每列唯一含义）；受控同义词表**不建立**（延后到出现真实变体时） | 确定性 |
| Q4 | 是否存在无法无歧义映射的字段？ | 无——6 列全部一行一义 | 确定性 |
| Q5 | 确定性规则集？ | (a) 列名精确匹配；(b) 行级 context 导出 L3 归属与分组；(c) 原始单位取自列头声明；(d) 实例 key = 单元格文本 | 冻结为 deterministic |
| Q6 | 不确定性？ | 本 fixture 内无。边界如实声明：真实工程表格的合并单元格、多级表头、跨行拼接不在本 fixture 内（后续工作） | 无 |

**预期结果：结果 A（本 fixture 内全部确定性）** → D-STEP8-03 = CLOSED（结构 + 确定性程度 = 上述规则集内确定性）。

---

## 7. D-STEP8-13 正式处置（本 Sprint 形成的范围决议）

```text
处置决议：范围决议（最小路径不激活）+ D-STEP8-13 独立保持 OPEN
```

1. 最小路径（本切片单源 fixture）**不激活** Conflict 分支与 `resolve_conflict`；第一版 FactStore **不提供** `resolve_conflict` 入口（避免发明「裁决后转换」）。
2. Conflict **记录**能力保留：`mark_conflict` 仅记录 `candidates` + `resolution_policy="manual"`（`resolved_by` / `resolution` 保持 `None`）；该记录行为不构成「裁决后转换」，不违反 §15.5。
3. **不假定、不实现、不伪造**任何 conflict → superseded / revision / supersedes 转换（CDM §15.5：二者不得混用）。
4. S8-S-05 范围相应裁剪：状态机验证覆盖 candidate → filled/pending → confirmed / missing / rejected + conflict 记录；**裁决转换验证延后**。
5. D-STEP8-13 保持 **OPEN**；因最小路径不激活，不构成当前路径 Coding-Blocking（门禁 8 的组成部分）。

---

## 8. D-STEP8-02 关闭条件（Sprint 执行清单）

| # | 条件 | 验证方式 | 执行后状态 |
|---|---|---|---|
| 1 | Excel 文件存在 | `case_step8_defect_v1/inputs/inspection.xlsx` 存在 | ✅ **已执行**（2026-10-09）：文件存在（Glob 确认） |
| 2 | Sheet / header / 行数正确 | openpyxl 读取：1 sheet、header 行 1、6 数据行、6 列 | ✅ **已执行**：回读 `sheets=['构件缺陷检查记录']`、`7 × 6`（1 header + 6 数据），逐行数据与 §2 表一致（含 D002=0.30 边界、D003=0.35 超限） |
| 3 | 列头语义明确 | 本 Brief §3 映射表 + `notes.md` | ✅ **已执行**：6/6 列头精确映射（§3 映射表）；`notes.md` §2 含 registry 行号证据 |
| 4 | fact_type 全部已注册 | `registry.has()` 逐条 6/6（零 Registry 变更） | ✅ **已执行**：`get_default_registry().has()` → 6/6 True；`registry.py` 在 git 状态中无任何变更 |
| 5 | 覆盖 qualified / unqualified / 边界 | §2 数据表（D002 边界、D003 超限） | ✅ **已执行**：数据行含 D002（= 0.30 边界等值）与 D003（0.35 超限） |
| 6 | registry.py 未修改 | git 状态核对（无 diff） | ✅ **已执行**：`git status` 未列出 `src/cdm/registry.py`（无修改、无新增） |
| 7 | fixture meta.json 标记 | `synthetic_business_fixture=true`、`domain`、`registry_extensions=[]`、`case_level=bronze` | ✅ **已执行**：meta.json 已创建，字段全部符合（见下） |
| 8 | OOS-15 边界 | fixture 目录**无** ground_truth / expected_issues / known_issues；现有 7 个 case 目录零触碰 | ✅ **已执行**：目录内容仅 `inputs/inspection.xlsx` + `meta.json` + `notes.md`；git 状态确认既有 case 目录无任何变更 |

**meta.json 字段（已创建，实际内容）**：

```json
{
  "case_id": "case_step8_defect_v1",
  "case_level": "bronze",
  "report_type": "structure_safety_appraisal",
  "source_project": "synthetic-step8-defect-001",
  "created_at": "2026-10-09T00:00:00",
  "synthetic_business_fixture": true,
  "domain": "defect_inspection",
  "registry_extensions": [],
  "fixture_direction": "defect_judgement_slice",
  "input_file": "inputs/inspection.xlsx",
  "fixture_purpose": "step8_input_only",
  "notes": "Step 8 vertical slice input fixture; NOT an Evaluation Case (no ground_truth / expected_issues by OOS-15); synthetic, not real project data"
}
```

> 沿用冻结 Meta 字段名（`src/eval/types.py` §1）以保证后续可加载性；`case_level=bronze` 如实表示开发用合成用例、无 ground truth；gold / silver 不被冒用。`fixture_purpose` 明确声明本目录不是 Evaluation Case（防止被误当作 evaluable case 加载）。

---

## 9. Sprint 工作流（执行步骤）

| # | 动作 | 产出 | 约束 |
|---|---|---|---|
| 1 | 本 Brief v2 修订 | 本文件 | ✅（本节为修订完成标记） |
| 2 | 创建 fixture 目录 + 生成 `inspection.xlsx` | `inputs/inspection.xlsx` | ✅ 已执行（openpyxl 脚本生成后已删除；回读验证 7×6） |
| 3 | 写 `meta.json` | meta.json | ✅ 已执行（§8 字段） |
| 4 | 写 `notes.md` | notes.md | ✅ 已执行（映射表 + Q1-Q6 回答 + 显示策略 + 合成参数声明 + 归属推导声明 + token 约束） |
| 5 | 证据化关闭 D-STEP8-02/03/06/07/09 | Closure / Contract Decision Register 更新 | ✅ 已执行（2026-10-09）：Closure §2 五项 → CLOSED（每项引用本 Brief §3-§6 + fixture 文件证据）；Contract §19「未决策」表移出 → 「已决策」表 5 行；**无虚假关闭**（全部证据可复核） |
| 6 | D-STEP8-13 处置登记 | Closure / Contract 更新 | ✅ 已执行：Closure §2/§4 注记 + Contract §2/§7/§8/§19 + 本 Brief §7；登记为「OPEN（处置决议：最小路径不激活；非 Coding-Blocking）」 |
| 7 | 重跑 9 条门禁 | Closure §14 + Contract §0 | ✅ 已执行：门禁 1/2/3/5/6 由 FAIL → PASS；4（替代路径）/7/8/9 保持 PASS → **9/9 PASS**，READY FOR CODING |
| 8 | 回归验证 | `python -m pytest tests/ --ignore=tests/eval/test_validator_framework.py -q`（+ Step 7-F 专项） | ✅ 已执行（实际结果）：`tests/eval/test_step7f.py -q` → **139 passed**；全量（排除 PRE-EXISTING 的 `tests/eval/test_validator_framework.py` P-1 缺陷套件）→ **412 passed**；`git status` 复核 `src/cdm/registry.py` 与 Frozen Docs（02/03/04/05/07）零修改 |

**不在 Sprint 内**：ground_truth / expected_issues（OOS-15）；生产代码（Coding 阶段）；Frozen Docs 修改；evaluable case 注册。

---

## 10. 门禁重跑路径（READY FOR CODING 9 条）

| # | 门禁 | 关闭依据 |
|---|---|---|
| a | D-STEP8-02 Fixture 设计与来源性质 | §2/§8（文件存在 + 校验记录） |
| b | D-STEP8-03 Mapper 确定性边界 | §6（Q1-Q6 回答记录于 notes.md） |
| c | D-STEP8-06 第一版 FactType / Registry 要求 | §3（6/6 已注册；零扩展） |
| d | Sample→Component 关联合规最小替代 | 已满足（决策报告，保持不变） |
| e | D-STEP8-07 Compute 输入/分组/输出/缺失冲突语义 | §5（统计 ∅ 裁定 + unit registry + 语义归属） |
| f | D-STEP8-09 Criterion / ConclusionRule / 输出枚举 | §4 |
| g | D-STEP8-11 SystemOutput projection | 已 PASS（保持不变） |
| h | 无未解决的 Coding-Blocking Design Conflict | §7（D-STEP8-13 处置）+ D-CONFLICT-001 非当前路径 |
| i | 三份文档一致性 | Sprint 后核对（Closure §2/§14、Contract §0/§19、本 Brief） |

---

## 附录 A — 原混凝土方向（Deferred 参考，压缩保留）

- **方向**：混凝土抗压强度检测（GB/T 50081-2019 背景：每构件 3 平行试块 → 构件代表值 → 检验）。
- **阻塞**：D-CONFLICT-001（sample→component 归属无合法载体 + `sample.concrete_strength` / 聚合输出类型未注册）。
- **v1 候选结构（未采用、未创建）**：4 构件 × 3 试块 = 12 行；`构件编号 | 试块编号 | 抗压强度(MPa)`；P1/P2/P3 三路径均未批准。
- **恢复条件**：见决策报告 §5.2（独立决策批准 03 层面关联表达，或给出无需该关联的合规最小替代并经验证）；恢复时本冲突自动回到「当前路径阻塞」状态。
- **状态**：D-CONFLICT-001 = OPEN / DEFERRED / 非当前路径（未解决）；本 Brief 不再承载其实施。

---

*本 Brief v2 修订自 v1（混凝土方向）并**执行完毕**（2026-10-09）。所有结论可由证据复核：每项引用具体文件 / 字段 / 条款；§8/§9 全部事项已标注「✅ 已执行」并附实际验证结果，无虚假关闭。*

```text
STEP 8 DESIGN STATUS  = READY FOR CODING（9/9 门禁通过；登记于 Closure §14 / Contract §0）
STEP 8 FIXTURE DESIGN = COMPLETE（fixture 三件交付物已创建并验证：inspection.xlsx + meta.json + notes.md）
STEP 8 CODING         = READY（可启动 Coding Round；尚未开始）
D-CONFLICT-001 = OPEN / DEFERRED / 非当前路径（未解决）
D-STEP8-13 = OPEN（处置决议：最小路径不激活；不构成 Coding-Blocking；见 §7）
```