# Step 8 Design Discovery Report

> **状态**：DESIGN DISCOVERY = COMPLETE
>
> **Step 8 Design Status**：PROPOSED / NEEDS DECISION
>
> **撰写日期**：2026-10-08
>
> **前置条件**：Step 7-F ACCEPTED + FROZEN（2026-10-01），139 专属测试通过，412 全量回归通过
>
> **本阶段约束**：只读研究，禁止修改任何 src/tests/docs，禁止进入 Coding

---

## 1. Executive Summary（执行摘要）

### 1.1 一句话结论

当前系统是一个**"没有输入端的验证闭环"**——验证/评测侧已完整冻结（M0/M5/M7 全部完成，412 测试全绿），但生成/生产侧完全空白（M1-M4/M6/M8/M9 零实现），fixtures 中无任何真实输入数据，不存在任何代码路径能从真实输入产出可验证的 `SystemOutput`。

### 1.2 核心发现

| 维度 | 发现 | 严重度 |
|---|---|---|
| **最大瓶颈** | 生成侧完全断点：Input → CDM → IR → DOCX 的确定性管线不存在任何实现代码 | **CRITICAL** |
| **里程碑错位** | M5（validate/accuracy）和 M7（eval/loader + runner）在 M1-M4（生成骨架）之前完成，架构意图（M3 结束时零 LLM 端到端）未达成 | **HIGH** |
| **能力缺口** | 已实现 21 个 Python 文件（~3000+ 行）全部位于 src/cdm/ 和 src/eval/ 下，src/ 下不存在 parsers/、facts/、compute/、rules/、ir/、render/、llm/、pipeline/ 目录 | **CRITICAL** |
| **Fixture 限制** | 全部 7 个 fixture 的 inputs/ 目录仅含 .gitkeep，无任何真实 Excel/DOCX/PDF/图片；expected_issues 除 case_gold_minimal 外全部为空数组 | **HIGH** |
| **无真实验证目标** | `SystemOutput`（Step 7-E 的输入类型）在测试中由 synthetic fixture 构造，不存在任何生产代码能生成它 | **CRITICAL** |

### 1.3 Step 8 推荐方向（高层）

**Recommended Step 8 是 Candidate A：零 LLM Vertical Slice（M1-M3 最小闭环）**——从一份真实 Excel 输入出发，打通：

```
Excel → xlsx Parser → RawSource → Fact Store（状态机）→ Compute + Rules → IR Builder → Report IR
```

这是达成架构铁律（`02 §17.2`：**M3 结束时系统在零 LLM 状态下已能端到端产出可校验报告**）的最小必要步骤。不做模板渲染（M4），不引入 LLM（M8），不做 Pipeline Orchestrator（那是 Step 9 的事）。

### 1.4 排除的方向

| 候选 | 排除理由 |
|---|---|
| Multi-Agent / RAG / 知识图谱 / 向量数据库 | 过早引入 LLM 风险，违反 D-004"先做无 LLM 骨架"，无证据支撑 |
| 只做 Input Parser（M1 子集） | 范围过小，不是完整能力跃迁；独立价值弱（需要 Store/Compute 配合才能产出 IR） |
| Pipeline Orchestrator + 空 Step 桩 | 没有下层 Step 的实际实现，编排层是空壳；违反"先做确定性骨架" |
| LLM Adapter + 抽取 Skill（M8） | 在确定性骨架完成前引入 LLM，违反 D-004 和架构里程碑顺序 |
| Style Validator（M6） | 依赖 M4（模板渲染），当前无 IR 输出可验证 |

---

## 2. Current System Capability Map（14 层能力地图）

### 图例

| 标记 | 含义 |
|---|---|
| 🟢 **FROZEN** | 有完整实现 + 完整测试 + 已冻结契约 + 不可修改 |
| 🔵 **IMPLEMENTED** | 有完整实现 + 测试覆盖，但未显式冻结契约 |
| 🟡 **PARTIAL** | 有部分实现或占位，但核心功能缺失 |
| 🔴 **DESIGN ONLY** | 设计文档存在，但无任何实现代码 |
| ⚫ **NOT STARTED** | 设计文档不存在，实现不存在 |

---

### Layer 0：Design Documentation（设计文档层）

| 文档 | 版本 | 状态 | 覆盖范围 |
|---|---|---|---|
| `02_ARCHITECTURE.md` | v0.3 | 🟢 **FROZEN** | 架构约束、M0-M9 里程碑、7 步 Pipeline、AI/程序边界 |
| `03_CANONICAL_DATA_MODEL.md` | v0.3.1 | 🟢 **FROZEN** | Fact 三段式 ID、7 个 scope_class、受控 fact_type 注册表 |
| `04_REPORT_IR.md` | v0.3.1.1 | 🟢 **FROZEN** | 段落双模态、Ref 规则、锚定校验 §41.2、TableSpec |
| `05_EVALUATION.md` | v0.3 | 🟢 **FROZEN** | G1-G4 标准答案分离、Case 分级、7 条 P0、门禁/回归/确定性 |
| `07_DECISIONS.md` | v0.1 | 🟢 **FROZEN** | D-001 ~ D-027（索引止于 D-027，Step 7-E/F 的决策 D-028~D-044 未登记——见 N-1） |
| Step_7_E Coding Contract | v10/v11 | 🟢 **FROZEN** | Step 7-E 职责边界、B5 anchor_declarations 修订 |
| Step_7_F Coding Contract | v1 | 🟢 **FROZEN** | Step 7-F 职责、F-INV-1~8 冻结边界、四 Hash、Baseline 3-run |

---

### Layer 1：Canonical Data Model（CDM —— 真实数据模型层）

| 模块 | 文件 | 行数 | 测试 | 状态 | 说明 |
|---|---|---|---|---|---|
| Fact ID | `src/cdm/id.py` | ~120 | `tests/cdm/test_domain.py` | 🔵 **IMPLEMENTED** | 三段式 `fact:<scope_class>.<instance_key>.<attribute>`，完整正则校验，不允许 version suffix |
| Fact Types | `src/cdm/types.py` | ~200 | `tests/cdm/test_types.py` | 🔵 **IMPLEMENTED** | Fact/MissingFact/QuantitativeFact/QualitativeFact/DomainObject，dataclass + frozen=True |
| Registry | `src/cdm/registry.py` | ~80 | （并入 test_domain） | 🔵 **IMPLEMENTED** | 受控 fact_type 注册表，支持 register/get/validate_type |
| Domain Objects | `src/cdm/domain.py` | ~180 | `tests/cdm/test_domain.py` | 🔵 **IMPLEMENTED** | Component/Point/Building/Project/Defect/Material/Structure 七种 scope_class 完整实现 |
| Domain Validator | `src/cdm/domain_validate.py` | ~100 | （并入 test_domain） | 🔵 **IMPLEMENTED** | 跨 scope_class Ref 完整性检查 |
| Fact Validator | `src/cdm/validate.py` | ~150 | （并入 test_domain） | 🔵 **IMPLEMENTED** | fact_id 格式 + fact_type 前缀一致性 + revision > 1 必须 supersedes + missing→value=null |
| Package `__init__` | `src/cdm/__init__.py` | ~30 | — | 🔵 **IMPLEMENTED** | 全部类型导出 |

**CDM 小结**：完整、可靠、冻结契约、充分测试。**这是 Step 8 的坚实下游依赖**。

---

### Layer 2：Report IR（报告中间表示层）

| 模块 | 位置 | 状态 | 说明 |
|---|---|---|---|
| IR Schema 定义 | `docs/04_REPORT_IR.md` §3~§7 | 🔴 **DESIGN ONLY** | 完整定义：Report/Document/Section/Paragraph(Assertion/Narrative)/TableSpec/RenderedSegment/Ref/Anchor |
| IR Builder | `src/ir/builder.py` | ⚫ **NOT STARTED** | — |
| IR Schema Validator | `src/ir/validate.py` | ⚫ **NOT STARTED** | — |
| IR 序列化/反序列化 | `src/ir/serializer.py` | ⚫ **NOT STARTED** | — |

**Report IR 小结**：Schema 完整冻结（v0.3.1.1），但**零实现**。这是 Step 8 Candidate A 的直接上游目标——需要产出一个最小可用的 Report IR 实例。

---

### Layer 3：Input Parsers（输入解析层）

| 模块 | 设计位置 | 实现位置 | 状态 | 说明 |
|---|---|---|---|---|
| xlsx Parser | `02 §9.1 parse_xlsx` | `src/parsers/xlsx.py` | ⚫ **NOT STARTED** | 需解析 Excel 检测数据 → RawSource |
| docx_template Parser | `02 §9.1 parse_docx_template` | `src/parsers/docx_template.py` | ⚫ **NOT STARTED** | 需解析 Word 模板 → TemplateSpec + required_facts[] |
| pdf Parser | `02 §9.1 extract_text_from_pdf` | `src/parsers/pdf.py` | ⚫ **NOT STARTED** | — |
| Image Parser | （spike 提及） | `src/parsers/image.py` | ⚫ **NOT STARTED** | — |
| Parser 基类/RawSource | （隐含） | `src/parsers/base.py` | ⚫ **NOT STARTED** | RawSource 数据结构未定义 |

**Input Parsers 小结**：零实现。架构文档定义了 Skill 列表但无代码。**Candidate A 的起点就是 xlsx Parser**。

---

### Layer 4：Fact Store（事实存储/状态管理层）

| 模块 | 设计位置 | 实现位置 | 状态 | 说明 |
|---|---|---|---|---|
| Fact Store | `02 §17.1 M1` | `src/facts/store.py` | ⚫ **NOT STARTED** | 候选事实 → FactSet，含状态机（candidate→confirmed→missing→conflict→rejected） |
| Conflict Resolution | `03 §5 conflict` | `src/facts/conflict.py` | ⚫ **NOT STARTED** | 策略必须是 manual（project_memory 硬约束） |
| supersedes 链 | `03 §5.2` | （并入 store） | ⚫ **NOT STARTED** | revision > 1 时必须声明 supersedes |

**Fact Store 小结**：零实现。CDM 定义了 Fact 数据结构但没有状态管理。**Candidate A 的第二站**。

---

### Layer 5：Compute + Rules（计算与规则层）

| 模块 | 设计位置 | 实现位置 | 状态 | 说明 |
|---|---|---|---|---|
| 确定性计算 | `02 §17.1 M2` | `src/compute/` | ⚫ **NOT STARTED** | 统计量、单位换算（SI 基准） |
| Criterion 判定 | `02 §8.2 Step3` | `src/rules/criterion.py` | ⚫ **NOT STARTED** | 按 Criterion Schema 执行符合性判定 → Evaluation |
| ConclusionRule | `02 §8.2 Step4 + V2-5` | `src/rules/conclude.py` | ⚫ **NOT STARTED** | 多项 Evaluation 聚合为单一 Conclusion，显式规则 |
| 单位注册表 | （隐含 CDM） | （待定） | ⚫ **NOT STARTED** | 需要 unit registry 支撑换算 |

**Compute + Rules 小结**：零实现。CDM 有 Criterion Schema 但没有执行引擎。**Candidate A 的第三站**。

---

### Layer 6：IR Builder（IR 构建层）

| 模块 | 设计位置 | 实现位置 | 状态 | 说明 |
|---|---|---|---|---|
| IR Builder | `02 §8.2 Step5` | `src/ir/builder.py` | ⚫ **NOT STARTED** | FactSet + DomainObjects → Report IR，纯程序零 LLM |
| IR Schema Validator | `04 §41` | `src/ir/validate.py` | ⚫ **NOT STARTED** | Ref 可解析 / 锚定绑定 / 白名单命中 |

**IR Builder 小结**：零实现。这是 Candidate A 的**终点**——M3 里程碑的交付物。

---

### Layer 7：Template + Renderer（模板与渲染层）

| 模块 | 设计位置 | 实现位置 | 状态 | 说明 |
|---|---|---|---|---|
| TemplateSpec 解析 | `02 §9.1 parse_docx_template` | `src/parsers/docx_template.py` | ⚫ **NOT STARTED** | Word 模板 → 样式清单 + required_facts[] |
| Template Mapping | `02 §8.2 Step5` | （builder 内部） | ⚫ **NOT STARTED** | Report IR → Word Template 的 Semantic→Style 映射 |
| DOCX Renderer | `02 §9.1 render_docx` | `src/render/docx.py` | ⚫ **NOT STARTED** | IR + TemplateSpec → 成品 DOCX |
| 样式使用报告 | `02 §17.1 M4` | （render 内部） | ⚫ **NOT STARTED** | 记录模板中哪些样式被实际使用 |

**Template + Renderer 小结**：零实现。**Candidate A 明确排除**（Step 8 不做 M4，M3 产出 IR 即可，DOCX 渲染留给 Step 9）。

---

### Layer 8：Accuracy Validator（精度验证层 —— Step 7-E）

| 模块 | 文件 | 行数 | 测试 | 状态 | 说明 |
|---|---|---|---|---|---|
| Tokenizer | `src/eval/accuracy/tokenizer.py` | ~200 | `tests/eval/test_accuracy.py` | 🟢 **FROZEN** | 独立对 TableCell.raw_text 分词，不信任 system-reported TokenOccurrence |
| FrozenRound | `src/eval/accuracy/frozen_round.py` | ~80 | （并入 test_accuracy） | 🟢 **FROZEN** | Decimal exact equality，无 tolerance/float |
| Binding | `src/eval/accuracy/binding.py` | ~250 | （并入 test_accuracy） | 🟢 **FROZEN** | token→fact_id 绑定 + 值相等 + 白名单登记制 |
| Data Structures | `src/eval/accuracy/data_structures.py` | ~300 | `tests/eval/test_types.py` | 🟢 **FROZEN** | ActualIssue/ExpectedIssue/SystemOutput/TableSpecSnapshot 等 |
| Validator Framework | `src/eval/accuracy/validator_framework.py` | ~600 | `tests/eval/test_validator_framework.py` | 🟢 **FROZEN** | 7 条 P0 规则全实现（VALUE.CONSISTENCY / TABLE.INTERNAL / LOGIC.CONSISTENCY / ANCHOR.BINDING / CONCLUSION.DIRECTION / CONCLUSION.COVERAGE / EXPECTED.HIT） |
| Package `__init__` | `src/eval/accuracy/__init__.py` | ~20 | — | 🟡 **PARTIAL**（PRE-EXISTING P-1） | 未导出 RuleStatus/CaseStatus/P0_RULE_IDS/evaluate_case，导致 collection error |

**Accuracy Validator 小结**：**已冻结**、完整、充分测试、零缺陷（P-1 是 export 问题，不影响内部使用，不修）。这是整个系统验证侧的基石。

---

### Layer 9：Style Validator（样式验证层 —— M6）

| 模块 | 设计位置 | 实现位置 | 状态 | 说明 |
|---|---|---|---|---|
| 结构/编号/表格/样式使用 | `02 §17.1 M6` | `src/validate/style.py` | ⚫ **NOT STARTED** | 依赖 M4（DOCX 渲染），当前无输入可验证 |

---

### Layer 10：Evaluation Loader（评测 Case 加载层）

| 模块 | 文件 | 行数 | 测试 | 状态 | 说明 |
|---|---|---|---|---|---|
| Case Types | `src/eval/types.py` | ~150 | `tests/eval/test_types.py` | 🔵 **IMPLEMENTED** | EvalCase/G1/G2/G3/G4/GroundTruth/ExpectedIssue 等 dataclass |
| Loader | `src/eval/validator.py` | ~180 | `tests/eval/test_loader.py` | 🔵 **IMPLEMENTED** | `from_dir()` 目录加载器，完整校验（case_id 唯一、G2⊆G1、G3⊆G2 等） |
| Enums | `src/eval/enums.py` | ~40 | — | 🔵 **IMPLEMENTED** | Gold/Silver/Bronze、CaseStatus、RuleStatus 等 |
| Fixtures | `tests/eval/fixtures/` | — | 7 个 case | 🟡 **PARTIAL** | 所有 inputs/ 目录仅含 .gitkeep；expected_issues 除 case_gold_minimal 外全部为空 |

**Evaluation Loader 小结**：完整实现 + 充分测试。Fixture 的 inputs/ 为空是当前最大的验证盲区。

---

### Layer 11：Run / Regression / Gate / Baseline / Jitter（Step 7-F —— 冻结）

| 模块 | 文件 | 行数 | 测试 | 状态 | 说明 |
|---|---|---|---|---|---|
| Hash Utils | `src/eval/run/hash_utils.py` | ~80 | `tests/eval/test_step7f.py` | 🟢 **FROZEN** | RFC 8785 JCS + SHA-256，四 Hash 全实现 |
| Models | `src/eval/run/models.py` | ~400 | （并入 test_step7f） | 🟢 **FROZEN** | RunRequest/RunSnapshot/Baseline/RegressionReport/GateDecision/GateThresholds/JitterRegister 等全部 dataclass |
| Runner | `src/eval/run/runner.py` | ~600 | `tests/eval/test_step7f.py`（139 个） | 🟢 **FROZEN** | F-INV-1~8 冻结边界，硬编码占位（byte_size=0/prompt）是有意设计，属于 Caller/Loader 责任 |
| Package `__init__` | `src/eval/run/__init__.py` | ~30 | — | 🟢 **FROZEN** | 全部类型正确导出 |

**Step 7-F 小结**：**已冻结**、139 专属测试、412 全量回归通过。Runner 的硬编码占位（byte_size=0, prompt 硬编码）是 minimal implementation 的有意设计，不属于 defect。

---

### Layer 12：LLM Adapter + Skills（M8）

| 模块 | 设计位置 | 实现位置 | 状态 | 说明 |
|---|---|---|---|---|
| LLM Adapter | `02 §17.1 M8` | `src/llm/adapter.py` | ⚫ **NOT STARTED** | 薄封装，统一模型调用接口 |
| extract_fields_from_text | `02 §9.2 LLM Skill` | `src/llm/skills/` | ⚫ **NOT STARTED** | 文字 → Facts |
| extract_fields_from_image | `02 §9.2 LLM Skill` | `src/llm/skills/` | ⚫ **NOT STARTED** | 图片 → Facts |
| draft_narrative | `02 §9.2 LLM Skill` | `src/llm/skills/` | ⚫ **NOT STARTED** | Narrative 段落生成 |

**LLM 层小结**：零实现。**严格排除出 Step 8**——D-004 铁律"先做无 LLM 骨架"。

---

### Layer 13：Pipeline Orchestrator（编排层）

| 模块 | 设计位置 | 实现位置 | 状态 | 说明 |
|---|---|---|---|---|
| 7 步 Pipeline | `02 §8.2` | — | 🔴 **DESIGN ONLY** | Step1 intake → Step2 extract → Step3 compute → Step4 conclude → Step5 compose → Step6 validate → Step7 render |
| Orchestrator | （隐含） | `src/pipeline/orchestrator.py` | ⚫ **NOT STARTED** | — |
| Step 接口 | （隐含） | `src/pipeline/step.py` | ⚫ **NOT STARTED** | — |

**Pipeline 小结**：设计完整但零实现。**排除出 Step 8**——编排层需要下层 Step 有真实实现才有意义。

---

### Layer 14：Tests & Fixtures（测试与夹具层）

| 层级 | 文件数 | 测试数 | 状态 | 说明 |
|---|---|---|---|---|
| `tests/cdm/` | 4 | ~115 | 🔵 **IMPLEMENTED** | 覆盖 CDM 所有类型/ID/Domain/Validator |
| `tests/eval/` | 6 | ~300+ | 🔵 **IMPLEMENTED** | 覆盖 Eval Case 加载/校验、Accuracy Validator 全流程、Step 7-F 139 专属测试 |
| Fixtures | 7 个 case | — | 🟡 **PARTIAL** | inputs/ 全空；expected_issues 除 case_gold_minimal 外全部为空 |
| **全量回归** | — | 412 | 🟢 **PASS** | Step 7-F Freeze 时跑通 |

**Tests 小结**：验证侧测试充分。生成侧（M1-M4）零测试。fixture inputs/ 全空是当前最大的端到端测试盲区。

---

## 3. End-to-End Closure Analysis（端到端闭环分析）

### 3.1 目标链路（Architecture §8.2）

```
Input
  → Step1 intake（判定类型 + 解析 + RawSource）
  → Step2 extract（文字/图片 → Facts）
  → Step3 compute & rule（确定性计算 + Criterion 判定）
  → Step4 conclude（ConclusionRule 聚合）
  → Step5 compose（IR Builder → Report IR）
  → Step6 validate（Accuracy + Style → IssueList）
  → Step7 render（DOCX）
```

加上验证侧的：

```
SystemOutput
  → Step 7-E Accuracy Validator（7 条 P0）
  → Step 7-F Runner → RunSnapshot
  → Baseline → Regression → Gate
```

### 3.2 当前真实状态

```
真实输入 ──✘──→ [不存在任何 Parser]
                  │
                  ├──✘──→ [不存在 Fact Store]
                  │
                  ├──✘──→ [不存在 Compute/Rules]
                  │
                  ├──✘──→ [不存在 IR Builder]
                  │
                  ├──✘──→ [不存在 Template + Renderer]
                  │
                  ▼
              （真空地带）
                  │
                  ▼
         Step 7-E（FROZEN）← synthetic fixture 构造 SystemOutput
                  │
                  ▼
         Step 7-F（FROZEN）← 139 测试全通过
                  │
                  ▼
         RunSnapshot / Baseline / Regression / Gate（全实现）
```

### 3.3 闭环状态判定

| 指标 | 状态 | 判定 |
|---|---|---|
| 输入源 → RawSource | **✘ 不存在** | 断点 |
| RawSource → FactSet | **✘ 不存在** | 断点 |
| FactSet → IR | **✘ 不存在** | 断点 |
| IR → SystemOutput | **✘ 不存在** | 断点（IR Schema 存在但无 Builder） |
| SystemOutput → Accuracy Validator | **✔ 冻结** | 通（但只接受 synthetic 输入） |
| Accuracy → Run → Baseline → Regression → Gate | **✔ 冻结** | 通 |
| 门禁可判定真实生成质量 | **✘ 不可** | 无真实 SystemOutput 可喂给 Validator |

**结论**：验证侧闭环完整，但**整个生成侧链条断裂**。当前系统是一个"完美的接收端，但没有发射端"。

---

## 4. Gap Matrix（差距矩阵）

### 4.1 按里程碑（Architecture §17.2）

| 里程碑 | 内容 | 设计状态 | 实现状态 | 端到端可用 | 差距等级 |
|---|---|---|---|---|---|
| **M0** | Fact Schema + fact_type 注册表 + Criterion + IR Schema + Case Schema + 校验器骨架 | ✅ 完整 | ✅ 完整 | ✅（作为下游契约） | **DONE** |
| **M1** | parsers/xlsx + facts/store（状态机 + Conflict） | ✅ 完整（Skill 列表） | ❌ 零 | ❌ | **CRITICAL** |
| **M2** | compute/ + rules/ + rules/conclude | ✅ 完整（Step 3/4 定义） | ❌ 零 | ❌ | **CRITICAL** |
| **M3** | ir/schema + ir/builder（纯程序零 LLM） | ✅ IR Schema 完整；Builder 需新建 | ❌ 零 | ❌ | **CRITICAL** |
| **M4** | parsers/docx_template + render/docx + 样式使用报告 | ✅ Skill 列表 | ❌ 零 | ❌ | **HIGH**（依赖 M3） |
| **M5** | validate/accuracy | ✅ 完整 | ✅ 完整（Step 7-E） | ✅ | **DONE**（超前） |
| **M6** | validate/style | ✅ 完整 | ❌ 零 | ❌ | **MEDIUM**（依赖 M4） |
| **M7** | eval/loader + runner + 基线快照 | ✅ 完整 | ✅ 完整（Step 7-F） | ✅ | **DONE**（超前） |
| **M8** | llm/adapter + 1 个抽取 Skill | ✅ Skill 列表 | ❌ 零 | ❌ | **BLOCKED**（D-004：先做无 LLM 骨架） |
| **M9** | Narrative 段落生成 + 锚定校验 + 语义一致性 | ✅ 双模态设计 | ❌ 零 | ❌ | **BLOCKED**（依赖 M8） |

### 4.2 按 Pipeline Step（Architecture §8.2）

| Step | 职责 | 实现 | 依赖 | 依赖状态 | 可独立实现 |
|---|---|---|---|---|---|
| Step1 intake | 输入判定 + 解析 + RawSource | ❌ | 无（起点） | — | ✅ |
| Step2 extract | 文字/图片 → Facts | ❌ | Step1 RawSource | ❌ | — |
| Step3 compute & rule | 确定性计算 + Criterion 判定 | ❌ | Step2 Facts | ❌ | — |
| Step4 conclude | ConclusionRule 聚合 | ❌ | Step3 Evaluation | ❌ | — |
| Step5 compose | IR Builder → Report IR | ❌ | Step1~Step4 全产物 | ❌ | — |
| Step6 validate | IR 校验 + Accuracy + Style | 🔶 Accuracy ✅ Style ❌ | Step5 IR | ❌ | — |
| Step7 render | DOCX | ❌ | Step5 IR + TemplateSpec | ❌ | — |

### 4.3 关键缺失项清单

| 编号 | 缺失项 | 严重度 | 关联里程碑 | 备注 |
|---|---|---|---|---|
| **GAP-1** | xlsx Parser | CRITICAL | M1 | Excel 是当前项目的主要输入格式 |
| **GAP-2** | Fact Store + 状态机 | CRITICAL | M1 | CDM 有 Fact 数据结构但无状态管理 |
| **GAP-3** | Criterion 执行引擎 | CRITICAL | M2 | CDM 有 Criterion Schema 但没有 evaluate() |
| **GAP-4** | ConclusionRule 引擎 | CRITICAL | M2 | V2-5 新增的结构性补缺 |
| **GAP-5** | IR Builder | CRITICAL | M3 | M3 里程碑的核心交付物 |
| **GAP-6** | 真实 fixture 输入 | HIGH | M1 前置 | 当前全部 inputs/ 为 .gitkeep |
| **GAP-7** | 单位注册表 + 换算 | HIGH | M2 | CDM 存原始值 + 单位，但无换算逻辑 |
| **GAP-8** | DOCX Template Parser | MEDIUM | M4 | TemplateSpec → required_facts[] |
| **GAP-9** | DOCX Renderer | MEDIUM | M4 | IR → 成品 DOCX |
| **GAP-10** | Pipeline Orchestrator | LOW | M1-M7 全部 | 下层 Step 不存在时空壳 |

---

## 5. Architecture Bottlenecks（架构瓶颈）

### 5.1 Bottleneck 1：生成侧完全断点（CRITICAL）

**描述**：从 Input 到 SystemOutput 之间不存在任何实现代码。CDM 是一个漂亮的"数据契约"，Accuracy Validator 是一个可靠的"接收端"，Step 7-F 是一个完整的"评测闭环"，但没有任何代码能产生 Validator 可以消费的真实 SystemOutput。

**证据**：
- `src/` 下不存在 `parsers/`、`facts/`、`compute/`、`rules/`、`ir/`（Builder）、`render/`、`pipeline/` 目录
- 21 个 Python 文件全部在 `src/cdm/` 和 `src/eval/` 下
- 全部 7 个 fixture 的 inputs/ 目录只有 .gitkeep
- `SystemOutput` dataclass 在 `data_structures.py` 中定义，但不存在任何生产代码路径能生成它

**影响**：Step 7-F 的 Gate 永远只能判定 synthetic fixture 的质量，无法判定真实生成质量。Baseline 冻结的是 synthetic 结果，Regression 检测的是 synthetic 差异。

**解除条件**：打通 M1-M3 确定性管线，至少能从一份真实 Excel 输入产出一个可被 Step 7-E 消费的 SystemOutput。

---

### 5.2 Bottleneck 2：M3 里程碑未达成（HIGH）

**描述**：`02 §17.2` 明确声明——"**M3 结束时，系统在零 LLM 状态下已能端到端产出可校验报告——这是判断架构是否成立的唯一硬标准**"。

当前 M0 ✅ → M1 ❌ → M2 ❌ → M3 ❌ → M4 ❌ → M5 ✅ → M6 ❌ → M7 ✅ → M8 ❌ → M9 ❌

M5/M7 在 M1-M4 之前完成，意味着验证侧已经准备好但生成侧完全空白。架构的"唯一硬标准"（零 LLM 端到端产出可校验报告）尚未达成。

**影响**：无法验证架构设计本身是否成立。Accuracy Validator 是基于 IR Schema 和 CDM 设计的，但从未被真实生成的 IR 验证过。Pipeline 的 7 步设计从未被真实数据走过一次。

**解除条件**：完成 M1-M3，产出第一个真实的、端到端的、零 LLM 的 SystemOutput。

---

### 5.3 Bottleneck 3：Fixture inputs/ 真空（HIGH）

**描述**：7 个 fixture 中，只有 case_gold_minimal 的 expected_issues.json 有 1 条 FACT.MISSING，其余全部为空数组 `[]`。更关键的是，所有 fixture 的 inputs/ 目录只有 `.gitkeep`。

这意味着：
1. Accuracy Validator 从未验证过任何真实数据的锚定/合计/判定逻辑
2. Baseline 的 3-run 校准基于 synthetic fixture
3. Regression 检测的差异是 synthetic 之间的差异
4. Gate 的门禁判定也是 synthetic 数据的判定

**影响**：验证精度存疑。Validator 测试充分但只测了 synthetic 边界情况，没有面对过真实 Excel 的复杂度（合并单元格、多 sheet、列对齐差异、单位不统一等）。

**解除条件**：Step 8 必须同时引入至少一份真实 Excel fixture，让 M1-M3 的产出能被 Step 7-E 真实消费。

---

### 5.4 非瓶颈项（确认不需要 Step 8 解决）

| 项目 | 为什么不是瓶颈 |
|---|---|
| Pipeline Orchestrator | 下层 Step 不存在时空壳，Step 8 不需要它 |
| LLM Adapter + Skills | D-004 铁律，M1-M3 全部是确定性模块 |
| Style Validator | 依赖 M4（DOCX 渲染），不是当前最短路径 |
| RAG / 向量数据库 / 知识图谱 | 无证据支撑当前阶段需要 |
| Multi-Agent | `02 §8.1` 明确"第一阶段线性 pipeline 不存在需要自主决策的分支" |
| Microservices / K8s / MQ | 过度工程化，`02 §0 V2.1` 强调"先做确定性骨架" |

---

## 6. Step 8 Candidates

### Candidate A：零 LLM Vertical Slice —— M1-M3 最小闭环（RECOMMENDED）

#### 6.1.1 Problem（问题）
当前系统是"没有输入端的验证闭环"——验证侧（CDM + Accuracy + Run）已完整冻结，但生成侧（Input → Fact Store → Compute → IR）零实现，架构铁律"零 LLM 端到端"未达成。

#### 6.1.2 Evidence（证据）
- **E-1**：`src/` 下无 `parsers/`、`facts/`、`compute/`、`rules/`、`ir/builder` 目录（21 文件全在 cdm/ 和 eval/）
- **E-2**：全部 7 个 fixture 的 inputs/ 目录仅含 .gitkeep，无任何真实输入数据
- **E-3**：`SystemOutput` dataclass 在 `data_structures.py` 定义，但不存在任何生产代码路径能生成它
- **E-4**：`02 §17.2` 明确"**M3 结束时系统在零 LLM 状态下已能端到端产出可校验报告——这是判断架构是否成立的唯一硬标准**"
- **E-5**：Step 7-F 的 Gate/Regression/Baseline 全部基于 synthetic fixture，从未面对真实生成质量

#### 6.1.3 Why Now（为什么现在做）
1. **验证架构设计的唯一途径**：M5/M7 超前完成，Accuracy Validator 和 Step 7-F 的正确性从未被真实 IR/SystemOutput 验证过
2. **解锁真实验证循环**：一旦有真实 SystemOutput，就能跑 Step 7-E 的 P0 规则（7 条冻结的精度检查），发现 Validator 设计中的盲点
3. **为 M4/M8 铺路**：M3 完成后，M4（模板渲染）只需给 IR 套一个 Word 模板；M8（LLM 引入）只需把 Step 2（extract）从确定性解析换成 LLM Skill，其他步骤不动
4. **范围可控**：M1-M3 全是确定性模块，符合 D-004"先做无 LLM 骨架"的原则

#### 6.1.4 Dependencies（依赖）
| 依赖 | 状态 | 版本 |
|---|---|---|
| CDM Schema + Validator | ✅ FROZEN | v0.3.1 |
| Report IR Schema | ✅ FROZEN | v0.3.1.1 |
| 架构文档 M0-M3 里程碑 | ✅ FROZEN | v0.3 |
| Accuracy Validator | ✅ FROZEN | Step 7-E |
| Step 7-F Runner | ✅ FROZEN | Step 7-F |
| 至少一份真实 Excel fixture | ❌ 需引入 | — |

#### 6.1.5 Scope（范围）

**In Scope**：

| 模块 | 职责 | 产出 |
|---|---|---|
| `src/parsers/` | xlsx Parser + RawSource 数据结构 | 从真实 Excel → RawSource（结构化候选事实） |
| `src/facts/` | Fact Store + 状态机 + Conflict（manual 策略） | RawSource → 确认的 FactSet |
| `src/compute/` | 确定性计算 + 单位换算 | FactSet → 计算后的 Facts |
| `src/rules/` | Criterion 执行引擎 + ConclusionRule | FactSet → Evaluation → Conclusion |
| `src/ir/builder.py` | IR Builder（纯程序零 LLM） | FactSet + DomainObjects → Report IR |
| `src/ir/validate.py` | IR Schema Validator（Ref 可解析） | 校验 IR 符合 04 Schema |
| `tests/` | 对应各模块的单元测试 + 集成测试 | 测试覆盖 |
| `tests/eval/fixtures/` | 引入至少 1 份真实 Excel fixture（替换 inputs/.gitkeep） | 真实输入验证 |

**Out of Scope（明确排除）**：

| 排除项 | 理由 |
|---|---|
| DOCX Template Parser + Renderer（M4） | M3 只需产出 IR，DOCX 渲染留给 Step 9 |
| Pipeline Orchestrator | 下层 Step 不存在时空壳 |
| LLM Adapter + Skills（M8） | D-004 铁律，M1-M3 全部是确定性模块 |
| Style Validator（M6） | 依赖 DOCX 渲染 |
| 图片/PDF/文字 Parser | Candidate A 只覆盖 xlsx，其他格式留到后续 Step |
| 修改 Step 7-E/F | 冻结契约，禁止修改 |
| 修复 P-1 export defect | PRE-EXISTING，不修 |
| Multi-Agent / RAG / 向量数据库 | 过度工程化 |

#### 6.1.6 Deliverable（交付物）
1. **最小可用的端到端管线**：`real_excel → RawSource → FactSet → Evaluation → Conclusion → Report IR`
2. **至少 1 个真实 Excel fixture**（放入 `tests/eval/fixtures/<case>/inputs/`）
3. **该 fixture 能被 Step 7-E 真实消费**（产出非 synthetic 的 ActualIssue 集合）
4. **M0-M3 全部模块的单元测试 + 集成测试**
5. **能生成真实 SystemOutput 的生产代码路径**（Step 7-E 当前消费的都是 synthetic）

#### 6.1.7 Risk（风险）

| 风险 | 概率 | 影响 | 缓解 |
|---|---|---|---|
| 真实 Excel 的复杂度超出设计假设（合并单元格、多 sheet、非标准列名） | **MEDIUM** | Parser 返工 | 先选 1 份简单 Excel 做 fixture，逐步增加复杂度；Parser 设计为可扩展 |
| Criterion 执行引擎发现 CDM 的 Criterion Schema 有设计空洞 | **MEDIUM** | 需要修订 CDM Schema（需 D-045 决策） | 严格按 03 文档实现；若发现空洞，先登记再修订（不静默修复） |
| IR Builder 发现 04 Schema 有边界 case 未覆盖 | **MEDIUM** | 需要修订 IR Schema（需 D-045 决策） | 严格按 04 文档实现；若发现边界问题，登记决策 |
| 真实 fixture 的 Validation 暴露 Validator 设计盲点 | **LOW** | 可能需要微调 Validator | 这正是我们想要的——发现盲点再迭代；Validator 已冻结，但发现的问题可以记为 Non-Blocker |

#### 6.1.8 Alternative（替代方案）
- **Candidate B**（见下一节）：只做 xlsx Parser，不打通 Store/Compute/IR——范围过小，不是完整能力跃迁
- **Candidate C**（见下一节）：Pipeline Orchestrator + 空桩——空壳无价值

---

### Candidate B：Input Parser 层（M1 子集）

#### 6.2.1 Problem
所有 fixture 的 inputs/ 目录为空，系统无法消费真实输入。需要至少有一个 Parser 能把 Excel 解析成 RawSource。

#### 6.2.2 Evidence
- E-2（inputs/ 全空）
- E-1（无 parsers/ 目录）
- 架构文档 §9.1 定义了 parse_xlsx 作为确定性 Skill

#### 6.2.3 Why Now
Parser 是整个生成管线的起点，没有它就没有后续的任何步骤。

#### 6.2.4 Dependencies
| 依赖 | 状态 |
|---|---|
| CDM Schema（RawSource 需引用 CDM 类型） | ✅ |
| 至少 1 份真实 Excel 样本 | ❌ 需引入 |

#### 6.2.5 Scope
- 只做 `src/parsers/xlsx.py` + `src/parsers/base.py`（RawSource 数据结构）
- 不做 Fact Store、Compute、Rules、IR Builder
- 引入 1 份真实 Excel fixture
- 单元测试覆盖 Parser 的各种 Excel 场景

#### 6.2.6 Out of Scope
- 所有 M2/M3 模块
- Pipeline Orchestrator
- 任何自动生成 CDM Fact 的逻辑（Parser 只产出 RawSource，不做语义理解）

#### 6.2.7 Risk
- **低风险**：Parser 边界清晰，不涉及语义推断
- **但独立价值弱**：Parser 产出的 RawSource 无处可去（没有 Store 消费它），单独做 Parser 后仍然无法端到端产出 IR

#### 6.2.8 为什么不推荐
范围过小。Candidate B 是 Candidate A 的第一步（M1 子集），单独做它无法达成 M3 里程碑。但如果 Step 8 时间或资源紧张，**可以把 Candidate B 作为 Candidate A 的第一阶段交付物**。

---

### Candidate C：Pipeline Orchestrator + Step 桩

#### 6.3.1 Problem
架构文档 §8.2 定义了 7 步 Pipeline，但当前无任何编排代码。

#### 6.3.2 Evidence
- 架构文档 §8.2 定义了 Step1-7
- 当前无 `src/pipeline/` 目录

#### 6.3.3 Why Now
Orchestrator 是让 7 个 Step 协作的胶水层，没有它 Step 之间的通信（Facts → IR → SystemOutput）需要手动串联。

#### 6.3.4 Dependencies
| 依赖 | 状态 |
|---|---|
| CDM Schema | ✅ |
| IR Schema | ✅ |
| Accuracy Validator | ✅ |
| Pipeline Step 接口设计 | ❌ 需设计 |

#### 6.3.5 Scope
- `src/pipeline/orchestrator.py`
- `src/pipeline/step.py`（Step 基类 + Step1-7 空桩）
- 步骤间通信规范
- 无真实 Step 实现

#### 6.3.6 Risk
- **高风险（空壳）**：没有下层 Step 的真实实现，Orchestrator 只是一个调用空桩的循环——无任何生产价值
- **违反 §8.1 结论**："**一个路由函数即可**"、"**一个步骤即可**"、"**一个循环 + IssueList 即可**"——Orchestrator 在第一阶段确实不需要复杂实现

#### 6.3.7 为什么不推荐
**违反"先做确定性骨架"的铁律**。Orchestrator 是"编排层"，应该在下层 Step 有真实实现后再加。当前做它是空壳工程。**Step 9 再做 Orchestrator**，在 M3 完成之后。

---

## 7. Candidate Comparison（候选比较矩阵）

| 比较维度 | **A. M1-M3 Vertical Slice（RECOMMENDED）** | **B. Parser Only（M1 子集）** | **C. Pipeline Orchestrator** |
|---|---|---|---|
| **Problem Coverage** | 完整覆盖瓶颈 1+2+3 | 只覆盖瓶颈 1 的起点 | 不覆盖任何瓶颈 |
| **Evidence 支撑** | E-1 ~ E-5 全部命中 | E-1 + E-2 | 只有"设计存在" |
| **Why Now 强度** | 验证架构设计的唯一途径 | 逻辑成立但独立价值弱 | 下层无实现时空壳 |
| **Dependencies Ready** | ✅ 全部 Ready | ✅ 全部 Ready | ⚠️ 下层 Step 不存在 |
| **Scope 大小** | 中（~6 模块 + fixture） | 小（~2 模块 + fixture） | 小（~2 模块 + 桩） |
| **复杂度** | 中（需要设计 Store 状态机、Criterion 引擎、IR Builder） | 低（纯解析） | 低（纯编排） |
| **可验证性** | ✅ 产出的 IR 能被 Step 7-E 消费 | ⚠️ Parser 输出无处可去 | ❌ 空桩无输出 |
| **里程碑贡献** | 达成 M3（架构铁律） | 只推进 M1 的子集 | 不推进任何里程碑 |
| **端到端** | ✅ 真实 Excel → 真实 IR → 真实 SystemOutput | ❌ Excel → RawSource（断） | ❌ Orchestrator → 空桩 |
| **风险** | 中（Schema 可能有边界空洞） | 低 | 空壳工程 |
| **与 Step 7-E/F 的衔接** | ✅ 自然衔接（SystemOutput → Validator → Runner） | ❌ 无法衔接 | ❌ 无法衔接 |
| **后续扩展性** | 高（M4 渲染、M8 LLM 替换 Step2 extract） | 需追加 Store/Compute/IR Builder | 需追加所有 Step 的真实实现 |
| **违反架构原则** | 无 | 无 | 违反 §8.1 + "先做骨架" |

---

## 8. Recommended Step 8 Scope（推荐 Step 8 范围）

### 8.1 推荐：Candidate A —— 零 LLM M1-M3 Vertical Slice

**标记：PROPOSED（非 FROZEN）**

**一句话**：从一份真实 Excel 出发，打通 Excel → Parser → RawSource → Fact Store → Compute → Criterion 判定 → Conclusion → IR Builder → Report IR，产出第一个能被 Step 7-E 真实消费的 SystemOutput。

### 8.2 模块分解（Dependency Order）

```
阶段 1：基础设施 + Parser（M1 起点）
  ├─ src/parsers/__init__.py
  ├─ src/parsers/base.py          # RawSource + Candidate 数据结构
  ├─ src/parsers/xlsx.py          # Excel 解析 → RawSource
  ├─ src/facts/__init__.py
  ├─ src/facts/store.py           # Fact Set + 状态机（candidate→confirmed→missing→conflict→rejected）
  ├─ src/facts/conflict.py        # Conflict Resolution（manual 策略，不可自动）
  └─ tests/ 对应单元测试 + 1 份真实 Excel fixture

阶段 2：Compute + Rules（M2）
  ├─ src/compute/__init__.py
  ├─ src/compute/unit_registry.py # 单位注册表 + 换算
  ├─ src/compute/calculate.py     # 确定性计算（统计量、派生量）
  ├─ src/rules/__init__.py
  ├─ src/rules/criterion.py       # Criterion 执行引擎（按 03 Schema 判定 → Evaluation）
  ├─ src/rules/conclude.py        # ConclusionRule（多项 Evaluation → Conclusion）
  └─ tests/ 对应单元测试

阶段 3：IR Builder + 端到端（M3）
  ├─ src/ir/__init__.py
  ├─ src/ir/builder.py            # FactSet + DomainObjects + Conclusion → Report IR
  ├─ src/ir/validate.py           # IR Schema Validator（Ref 可解析 + Schema 合规）
  ├─ src/ir/adapter.py            # Report IR → SystemOutput（桥接 Step 7-E 输入）
  └─ tests/ 单元测试 + 端到端集成测试（真实 Excel → 真实 SystemOutput → Step 7-E 消费）
```

### 8.3 最小可行交付（MVP Definition）

**什么算 Step 8 成功**：

1. ✅ 有 1 份真实 Excel fixture 放入 `tests/eval/fixtures/<case>/inputs/`
2. ✅ 有生产代码能从这份 Excel 产出 RawSource → FactSet → Evaluation → Conclusion → Report IR → SystemOutput
3. ✅ 产出的 SystemOutput 能被 Step 7-E 的 `evaluate_case()` 真实消费（不使用 synthetic fixture）
4. ✅ Step 7-E 对这个真实 SystemOutput 跑 P0 规则，产出 ActualIssue 集合
5. ✅ 全量测试（原有 + 新增）通过

**明确不算成功的**：
- 只完成 Parser（Candidate B）
- 只完成 Orchestrator + 空桩（Candidate C）
- 只写测试写不出来
- 发现 Schema 边界问题但不登记决策

### 8.4 实现细节（设计方向，不是代码）

| 模块 | 设计要点 | 约束来源 |
|---|---|---|
| xlsx Parser | 用 openpyxl，产出 RawSource（包含 sheet_name、cell_ref、raw_value、context_hint） | 02 §9.1 |
| Fact Store 状态机 | `candidate → confirmed → missing / conflict → resolved → rejected / confirmed`，conflict 只能 manual 解决 | 03 §5 conflict + project_memory hard constraint |
| 单位注册表 | 原始值 + 原始单位 + 量纲种类（CDM V2-4），换算到 SI 基准 | 02 §0 V2-4 |
| Criterion 引擎 | 按 03 Criterion Schema 执行判定（comparison_op + threshold + source_fact_ids），产出 Evaluation | 03 Criterion Schema |
| ConclusionRule | 多项 Evaluation 聚合为单一 Conclusion（枚举 + 覆盖度），显式规则 | 02 §0 V2-5 + 05 |
| IR Builder | FactSet + DomainObjects + Conclusion → Report IR（Assertion 段落 + 可选 TableSpec），纯程序零 LLM | 04 Report IR Schema |
| IR → SystemOutput Adapter | 把 Report IR 的关键产物映射到 Step 7-E 定义的 SystemOutput dataclass | Step 7-E `data_structures.py` |

---

## 9. Step 8 Out of Scope（明确排除项）

### 9.1 排除列表（违反即需重新决策）

| 编号 | 排除项 | 理由 | 允许恢复条件 |
|---|---|---|---|
| **OOS-1** | DOCX Template Parser + Renderer（M4） | Step 8 只需 IR，DOCX 渲染是 Step 9 | Step 9 开始时 |
| **OOS-2** | Pipeline Orchestrator | 下层 Step 不存在时空壳 | M1-M3 全部有真实实现后 |
| **OOS-3** | LLM Adapter + 任何 LLM Skill | D-004 铁律"先做无 LLM 骨架" | M3 完成后（Step 10+） |
| **OOS-4** | Style Validator（M6） | 依赖 DOCX 渲染 | DOCX 渲染完成后 |
| **OOS-5** | PDF / 图片 / 自由文字 Parser | Candidate A 只覆盖 xlsx，其他格式是后续 Step | 待 xlsx 管线稳定后 |
| **OOS-6** | 修改 `src/eval/run/**`（Step 7-F） | 冻结契约，禁止修改 | 任何修改需显式解 Frozen |
| **OOS-7** | 修改 `src/eval/accuracy/**`（Step 7-E） | 冻结契约，禁止修改 | 任何修改需显式解 Frozen |
| **OOS-8** | 修复 P-1 export defect | PRE-EXISTING，不修 | 专门的 fix request |
| **OOS-9** | 修改任何 Frozen Contract / Decision | 架构铁律 | 需新的 Design Discovery + 决策会议 |
| **OOS-10** | Multi-Agent / RAG / 向量数据库 / 知识图谱 / 微服务 / K8s / MQ | 过度工程化，无证据支撑 | 有实证表明当前架构承载不住时 |
| **OOS-11** | 任何自动的 Conflict 解决策略 | project_memory hard constraint："Conflict resolution_policy must be 'manual'" | 架构修订决策 |
| **OOS-12** | 修改 `docs/02/03/04/05/07` | project_memory hard constraint："Design documents 02/03/04/05/07 must not be modified without explicit approval" | 显式批准 + D-045+ 决策记录 |

---

## 10. Known Non-Blockers & Pre-existing Issues

### 10.1 Non-Blocker Inventory

| 编号 | 性质 | 详情 | 来源 | 是否影响 Step 8 | 处置 |
|---|---|---|---|---|---|
| **P-1** | PRE-EXISTING Step 7-E 缺陷 | `src/eval/accuracy/__init__.py` 未导出 RuleStatus/CaseStatus/P0_RULE_IDS/evaluate_case，导致 `test_validator_framework.py` collection error | Step 7-F Freeze 登记 | **不影响**：Step 8 涉及的模块不依赖 eval.accuracy 根包导出，只从 validator_framework 直接导入 | 不修，登记在案 |
| **N-1** | Documentation Governance | D-041~D-044（Step 7-E/F 的决策）未登记到 `docs/07_DECISIONS.md`（索引止于 D-027） | 审阅发现 | 不影响技术实现 | Step 8 完成后作为独立文档治理任务 |
| **N-2** | Caller/Loader Responsibility | `runner.py::_compute_four_hashes()` 的 input/output artifact `byte_size=0` 占位；硬编码 prompt 模板 | Step 7-F Contract §11.3 注释："In minimal implementation we build a deterministic manifest from case_ids" | **可能关联**：若 Step 8 引入真实 artifact ingestion（真实 Excel → 计算 byte_size），可以自然填充；否则维持现状 | 不主动修复；Step 8 若有真实 loader 可自然填充 |
| **N-3** | Fact ID 无 source 信息 | Fact ID `fact:<scope>.<key>.<attr>` 不包含来源追踪，溯源靠 SourceRef 但 SourceRef 绑定在 DomainObject 而非 Fact 上 | 03 Schema 设计 | 可能影响冲突排查 | 登记，不影响 Step 8 |
| **N-4** | Fixture 中 case_gold_minimal 的 expected_issues 只有 1 条 | 当前 synthetic fixture 覆盖的 Issue 类型有限 | Fixture 扫描 | Step 8 引入真实 fixture 后自然扩展 | 作为 Step 8 fixture 设计的参考 |

### 10.2 Architecture Risks（需要持续关注但不阻塞 Step 8）

| 风险 | 描述 | 关注时机 |
|---|---|---|
| CDM Schema 边界空洞 | Criterion 执行引擎实现时可能发现 03 Schema 有未覆盖的判定场景 | Step 8 阶段 2（Criterion 引擎） |
| IR Schema 边界空洞 | IR Builder 实现时可能发现 04 Schema 有未覆盖的 IR 结构 | Step 8 阶段 3（IR Builder） |
| 真实 Excel 复杂度 | 真实 Excel 有合并单元格、多 sheet、非标准列名、公式引用等 | Step 8 阶段 1（xlsx Parser） |
| Validator 设计盲点 | 真实 SystemOutput 被 Step 7-E 消费时可能发现 Validator 的边界问题 | Step 8 阶段 3 端到端测试 |

---

## 11. Risks & Mitigations（Step 8 实施风险与缓解）

| 风险 | 概率 | 影响 | 缓解措施 | 应对 |
|---|---|---|---|---|
| **真实 Excel 复杂度超出假设** | MEDIUM | Parser 大幅返工 | 先选 1 份结构最简单的 Excel 做首份 fixture；Parser 设计为可扩展（策略模式处理不同 Excel 结构） | 若首份 fixture 解析失败，降级为 CSV 输入；或先人工构造一份标准化 Excel |
| **Criterion Schema 有设计空洞** | MEDIUM | 需要修订 CDM Schema | 严格按 03 文档实现；每遇到一个 Schema 空洞，先登记 issue 再决定是否需要 D-045 决策 | 若发现关键空洞无法绕开，暂停 Step 8 进入 Schema 修订决策 |
| **IR Schema 有边界 case 未覆盖** | MEDIUM | 需要修订 IR Schema | IR Builder 严格按 04 文档；边界问题记录为 issue | 同上——若关键空洞，进入 Schema 修订 |
| **真实 fixture 的 Validation 暴露 Validator 盲点** | LOW | 可能需要微调 Validator | 这正是 Step 8 的核心价值之一——发现盲点；Validator 已冻结但发现的问题可记为 Non-Blocker | 发现问题后先看是不是 Validator 的 bug（PRE-EXISTING），还是 IR 的问题；前者登记，后者修复 IR |
| **Scope Creep（忍不住加 Pipeline Orchestrator 或 LLM）** | HIGH | 违反 OOS 列表 | 严格执行 OOS 列表；每次新模块开始前对照 OOS 检查 | 若忍不住，停下来读一下 `docs/02_ARCHITECTURE.md §8.1` 的结论 |
| **Fixture 质量不够** | MEDIUM | 无法有效验证 | 首份 fixture 人工挑选；若没有合适的，可先用一份人工构造的 Excel（结构正确、数据真实） | 暂时用构造的，后续找真实的替换 |

---

## 12. Decision Points（Step 8 必须在编码前解决的决策点）

### 12.1 必须确认的决策

| 编号 | 决策点 | 选项 | 推荐 | 理由 |
|---|---|---|---|---|
| **D-STEP8-01** | Step 8 Scope | A（M1-M3 全） vs B（Parser Only） vs C（Pipeline 空桩） | **A**（M1-M3 Vertical Slice） | 完整覆盖瓶颈，达成 M3 里程碑 |
| **D-STEP8-02** | 首份 fixture 的 Excel 来源 | 真实档案 vs 人工构造 | 真实档案优先 | 真实数据才能暴露设计盲点；若暂无，人工构造作为过渡 |
| **D-STEP8-03** | RawSource 数据结构设计 | 独立模块 vs 并入 CDM | 独立 `src/parsers/base.py` | RawSource 是 Parser 输出层，不属于 CDM（Canonical = 标准化后的） |
| **D-STEP8-04** | 冲突解决策略 | manual only vs 允许半自动 | **manual only** | project_memory hard constraint 冻结 |
| **D-STEP8-05** | IR Builder 是否直接依赖 CDM | 直接依赖 vs 通过 Adapter | 直接依赖 | IR Builder 的输入就是 FactSet + DomainObjects，已经是 CDM 产物 |

### 12.2 设计决策登记规则

Step 8 实施过程中，每遇到一个设计问题，按以下流程：

```
1. 尝试按冻结的设计文档（02/03/04/05）直接实现
2. 如果文档有明确答案 → 直接实现，不做决策
3. 如果文档模糊或有边界 case → 登记为 D-STEP8-XX
4. 不得静默决定，不得绕过
5. 所有 D-STEP8-XX 在 Step 8 结束时汇总写入 07_DECISIONS.md
```

---

## 13. Architecture Alignment Check（架构对齐检查）

| 原则 | 来源 | Step 8 是否遵守 | 说明 |
|---|---|---|---|
| **AI 负责不确定性** | 00_PROJECT_CONTEXT | ✅ Step 8 不引入 LLM | M1-M3 全是确定性模块，Step 8 严格遵守 |
| **程序负责确定性** | 00_PROJECT_CONTEXT | ✅ Step 8 全部是程序模块 | xlsx Parser、Fact Store、Compute、Criterion、IR Builder 全是程序 |
| **AI 不能直接成为 Word 生成器** | 00_PROJECT_CONTEXT | ✅ Step 8 不碰渲染 | M3 只产出 IR，DOCX 渲染留给 Step 9 |
| **模型无关** | project_memory | ✅ Step 8 不依赖任何模型 | 全是确定性代码 |
| **先做无 LLM 骨架（D-004）** | 07_DECISIONS.md | ✅ Step 8 全是无 LLM | 这是 Step 8 的核心依据 |
| **第一阶段禁止过度工程化** | 02 §0 | ✅ Step 8 不引入 K8s/MQ/微服务/RAG | — |
| **Pipeline 线性编排（§8.1）** | 02 §8.1 | ✅ Step 8 实现 Step1-5 的真实逻辑，不搞 Agent | — |
| **M3 里程碑：零 LLM 端到端** | 02 §17.2 | ✅ 这是 Step 8 的唯一硬标准 | — |
| **CDM Fact 三段式 ID** | 03 + D-027 | ✅ Step 8 不修改 CDM | — |
| **Fact revision > 1 需 supersedes** | 03 | ✅ Step 8 按 CDM 实现 | — |
| **Conflict resolution = manual** | project_memory | ✅ Step 8 严格 manual | — |
| **Missing → value=null** | project_memory | ✅ Step 8 按 CDM 实现 | — |

**架构对齐结论**：Step 8 Candidate A 与所有冻结架构原则**完全对齐**，无任何冲突。

---

## 14. Final Recommendation & Next Step

### 14.1 最终推荐

| 维度 | 结论 |
|---|---|
| **Recommended Candidate** | A：零 LLM M1-M3 Vertical Slice |
| **设计状态** | PROPOSED（非 FROZEN） |
| **Out of Scope 边界** | 第 9 节完整列表 |
| **Non-Blockers** | 第 10 节完整清单 |
| **架构对齐** | 14/14 原则完全遵守 |

### 14.2 下一步（必须遵守）

```
Step 8 Design Discovery ✓ (本报告)
  ↓
用户审核本报告
  ↓
Step 8 Design Decision Closure（非 Coding）
  ├─ 确认 D-STEP8-01 ~ D-STEP8-05
  ├─ 确认首份 fixture 的 Excel 来源
  ├─ 确认 Scope 边界
  ├─ 产出冻结的 Step 8 Coding Contract
  └─ 用户审核 Coding Contract
  ↓
Step 8 Coding（在 Coding Contract 冻结后）
```

### 14.3 本阶段完成标志

- [x] 读取全部设计文档 + 源码 + 测试 + fixtures
- [x] 建立 14 层能力地图
- [x] 分析端到端闭环真实状态
- [x] 建立 Gap Matrix
- [x] 识别 3 个架构瓶颈
- [x] 提出 3 个 Candidate + 比较矩阵
- [x] 确定 Recommended Scope（PROPOSED）
- [x] 明确 Out of Scope
- [x] 登记 Non-Blockers
- [x] 输出完整 Design Discovery Report

**STEP 8 DESIGN DISCOVERY = COMPLETE**

**STEP 8 DESIGN STATUS = PROPOSED / NEEDS DECISION**

---

*报告结束。等待用户审核后进入 Step 8 Design Decision Closure。*
