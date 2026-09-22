好的，我先并行读取全部六个文档，然后进行系统性的基线核对。
用户要求我重新进行设计基线核对。让我先读取所有六个文档，然后系统地分析每个对象的定义状态和跨文档引用一致性。

我需要：
1. 确认 03_CANONICAL_DATA_MODEL.md 是否为 v0.3
2. 建立设计基线
3. 检查所有对象的定义状态
4. 检查跨文档引用一致性
5. 最后给出结论

让我先并行读取所有六个文档。

好，所有文档都已完整读取。现在让我系统性地进行基线核对。

## 版本确认
- 00_PROJECT_CONTEXT.md：无版本标记
- 02_ARCHITECTURE.md：v0.3
- 03_CANONICAL_DATA_MODEL.md：v0.3 ✓
- 04_REPORT_IR.md：v0.3
- 05_EVALUATION.md：v0.2（注意：文档头部版本号为v0.2）
- 07_DECISIONS.md：v0.1

现在让我逐一核对每个对象的定义状态和跨文档引用一致性。

## 对象逐一核对

### 1. Fact
- **定义位置**：03_CANONICAL_DATA_MODEL.md §6–§16
- **强制字段**：id, fact_type, value/unit/quantity_kind, source_refs[], method, provenance, confidence, review_status, status
- **其他文档引用**：
  - 02 §7/§9.1/§17.1：Facts / 确定性Skill / facts/store
  - 04 §3.2/§15.1/§41.1/§42：Ref引用Fact / Ref可解析 / Fact→Ref溯源
  - 05 §6.1/§6.4/§2：G3事实值Ground Truth / 数值锚定
  - 07 D-001/D-003/D-007/D-008/D-015/D-020：Fact相关决策
- **状态**：已正式定义 ✓

### 2. fact_type
- **定义位置**：03 §8.1 / 02 §6.4
- **内容**：受控注册表；三段式id中attribute_path来自注册表；quantity_kind映射
- **其他文档引用**：
  - 02 §6.4：required_facts[]类型层
  - 04 §17：TemplateSpec.fact_display_spec[fact_type]
  - 07 D-020：引入Fact类型层
- **状态**：已正式定义 ✓

### 3. quantity_kind
- **定义位置**：03 §10
- **枚举值**：文档提到了 length / force / pressure / ratio / count / area / dimensionless / qualitative
- **其他文档引用**：
  - 03 §8强制字段表
  - 05 §6.1：量纲必须一致
  - 07 D-008：取消SI基准，补quantity_kind
- **状态**：已正式定义 ✓（但完整枚举列表需要从注册表来）

### 4. review_status
- **定义位置**：03 §8强制字段 / §14 / §25.1
- **枚举值**：pending / confirmed / rejected
- **其他文档引用**：
  - 02 §11.4：Criterion判定闸门
  - 03 §25.1：Criterion同样具备review_status
  - 07 D-015 / D-021
- **状态**：已正式定义 ✓

### 5. supersedes / superseded
- **定义位置**：03 §15.5
- **内容**：更正语义，与conflict区分
- **其他文档引用**：
  - 07：无直接引用（是v0.3新增）
  - 02 §17.2 M0清单：提到supersedes
- **状态**：已正式定义 ✓

### 6. value_domain
- **定义位置**：03 §16.4 定性事实
- **内容**：定性事实取值域定义
- **其他文档引用**：
  - 04：无直接引用
  - 05：无直接引用
- **状态**：已正式定义（定性Fact专用）✓

### 7. SourceRef
- **定义位置**：03 §5 / 02 §6.2
- **字段**：source_id, location
- **其他文档引用**：
  - 03 §11：Fact.source_refs[]
  - 03 §25.1：Criterion.source_refs[]
  - 04 §42：Ref→Fact→SourceRef
  - 07 D-001：溯源链
- **状态**：已正式定义 ✓

### 8. Criterion
- **定义位置**：03 §25.1
- **强制字段**：criterion_id, value/unit/quantity_kind, condition, source_refs[], review_status, rule
- **判定闸门**：review_status != confirmed → not_evaluable
- **其他文档引用**：
  - 02 §7/§11.4/§13.1：判定闸门 / V3判定逻辑
  - 03 §27.1：ConclusionRule的inputs用criterion筛选
  - 05 §6.3：判定逻辑一致性
  - 07 D-021：同构判定闸门
- **状态**：已正式定义 ✓

### 9. Evaluation
- **定义位置**：03 §26
- **内容**：概念完整，但**强制字段表缺失**（03 §26只有概念描述和示例，没有像Fact/Criterion那样的强制字段表）
- **其他文档引用**：
  - 02 §8.2 Step3：产出Evaluation
  - 03 §27.1 ConclusionRule inputs: evaluation_selector
  - 04 §16：Ref可引用evaluation:xxx
  - 04 §42.1 covers[]: Evaluation id列表
  - 05 §2 G4结论方向 / §6.3判定逻辑 / §6.5结论方向枚举
  - 07：无直接决策（M2阶段定义）
- **状态**：**概念已提出但冻结Schema中强制字段未完整定义** ✗

### 10. Conclusion / ConclusionRule
- **定义位置**：03 §27 / §27.1
- **ConclusionRule强制字段**（概念级）：id, applies_to, inputs, logic, output, source_ref
- **其他文档引用**：
  - 02 §8.2 Step4 / §17.1 rules/conclude
  - 07 D-009：禁止LLM自由综合
- **状态**：ConclusionRule概念完整但**强制字段未列成表**（与Evaluation同问题）

### 11. Evidence
- **定义位置**：03 §28
- **内容**：概念级描述，无强制字段表
- **其他文档引用**：
  - 02 §6.1：图片→Evidence
  - 04 §13.1：Narrative段evidence_refs[]
- **状态**：概念已提出，Schema冻结阶段可以不实现第一阶段最小集

### 12. Report IR
- **定义位置**：04完整文档
- **文档版本**：v0.3
- **其他文档引用**：
  - 02 §5/§7/§8.2 Step5/§17.1 ir/schema
  - 03 §40：边界声明
  - 05 §6.4：锚定与IR anchors[]一致
  - 07 D-002/D-005等多决策
- **状态**：已正式定义 ✓

### 13. anchors
- **定义位置**：04 §13.1 / §41.2
- **内容**：Narrative段的锚定声明；锚定校验规则（三类载体、两类token）
- **其他文档引用**：
  - 02 §11.3 / §13.1 V4
  - 03：无直接引用（Anchor是IR层概念）
  - 05 §6.4：锚定评测规则
  - 07 D-017/D-018/D-019
- **状态**：已正式定义 ✓

### 14. covers
- **定义位置**：04 §42.1
- **内容**：结论段Assertion携带的Evaluation id列表
- **其他文档引用**：
  - 02 §13.1 V4
  - 05 §6.6：结论覆盖度判定
  - 07 D-026
- **状态**：已正式定义 ✓

### 15. whitelist_registry
- **定义位置**：04 §41.2.4 / 05 §3.3
- **内容**：模式级登记制；字段pattern/reason/scope/confirmed_by
- **其他文档引用**：
  - 02 §11.3
  - 07 D-019
- **状态**：已正式定义 ✓

### 16. Evaluation Case
- **定义位置**：05 §5
- **内容**：目录结构、meta.json、known_issues、expected_issues
- **其他文档引用**：
  - 02 §12/§17.1 eval/loader
  - 07 D-012
- **状态**：已正式定义 ✓

## 跨文档引用一致性检查

### 02 → 03 检查
- 02 §7 四层模型 → 03 §2 一致 ✓
- 02 §11.4 判定闸门 → 03 §25.1 一致 ✓
- 02 §6.4 fact_type注册表 → 03 §8.1 一致 ✓
- 02 §17.2 M0 Fact Schema → 03 §8 一致 ✓

### 02 → 04 检查
- 02 §11.3 数字锚定 → 04 §41.2 一致 ✓
- 02 §14 D2双模态 → 04 §3.3 一致 ✓
- 02 §17.2 M0 Report IR Schema → 04 完整 一致 ✓
- 02 §17.2 M0 anchors → 04 §41.2 一致 ✓
- 02 §17.2 M0 covers → 04 §42.1 一致 ✓

### 02 → 05 检查
- 02 §12 评测架构三原则 → 05 §2/§3 一致 ✓
- 02 §17.3 任务0 → 05 §9 一致 ✓

### 03 → 04 检查
- 03 §40 边界声明 → 04 §3.2 一致 ✓
- 03 §8 三段式ID → 04 §16 Ref格式中fact:xxx匹配 一致 ✓
- 03 §25.1 Criterion → 04 §16 Ref支持criterion:一致 ✓
- 03 §26 Evaluation → 04 §16 Ref支持evaluation: 一致 ✓
- 03 §27 Conclusion → 04 §16（Ref格式中未显式写conclusion:）⚠️

这里发现一个小问题：04 §16列出了 `fact:`、`evaluation:`、`criterion:` 三种Ref前缀，但03中Conclusion也是可被引用的结构化对象，Ref格式中没有明确写 `conclusion:` 前缀。

### 03 → 05 检查
- 03 §10 quantity_kind → 05 §6.1 量纲必须一致 一致 ✓
- 03 §15 missing/conflict → 05 §6.4 正确失败评测 一致 ✓
- 03 §25.1 Criterion判定闸门 → 05 §6.3 判定逻辑一致性 一致 ✓
- 03 §25.1 Criterion → 05 §2 G4来源限制 一致 ✓

### 04 → 05 检查
- 04 §41.2 锚定规则 → 05 §6.4 一致 ✓
- 04 §41.2.4 白名单 → 05 §3.3 一致 ✓
- 04 §42.1 covers[] → 05 §6.6 一致 ✓
- 04 §17 Ref无format → 05 §6.1 按显示精度比对 一致 ✓

## 发现的具体问题

### 问题1：Evaluation缺少强制字段表（中等严重）
- 位置：03 §26
- 现象：Evaluation只有概念描述和简单示例，没有Fact/Criterion那样的"强制字段"表
- 影响：M0冻结的Fact Schema只包含Fact/Criterion/Report IR，Evaluation Schema在M2才需要，所以**对Canonical Data Model第一阶段实现影响有限**，但需要在M2前补齐

### 问题2：Conclusion缺少强制字段表（轻微）
- 位置：03 §27
- 同上，M2才需要

### 问题3：Evidence缺少强制字段表（轻微）
- 位置：03 §28
- 第一阶段CDML实现不需要Evidence的完整Schema骨架

### 问题4：Ref中Conclusion前缀未显式列出（轻微）
- 位置：04 §16
- 04 §16只列出了fact:、evaluation:、criterion:，但结论可能也需要被引用
- 可以延期到IR实现时补齐

### 问题5：Report IR version字段仍写"0.1"但文档版本已v0.3（轻微）
- 位置：04 §5 Document示例中 `"version": "0.1"`，§38写 `"version = 0.1"`
- 这是文档更新不彻底，实际Schema版本号应由M0冻结时统一决定

### 问题6：无技术栈决策（不是冲突，是缺失）
- 整个文档体系都没有指定语言/框架

现在让我整理最终输出。
正在请求专家建议......

已获取专家建议

好，Advisor给出了关键结论：**设计基线没有结构性冲突，可以开始Step 7-B**。现在让我整理完整的基线核对输出。
好的，我已经完整读取了全部六个文档。现在进行系统性基线核对。

*正在请求专家建议，优化解决方案，该过程耗时可能较长，请耐心等待...*

---

# 当前唯一设计基线

| 文档 | 当前版本 | 实际状态 |
|------|---------|---------|
| 00_PROJECT_CONTEXT.md | 无版本标记 | 项目背景，不动 |
| 02_ARCHITECTURE.md | v0.3 | 架构骨架，引用下游三份契约 |
| 03_CANONICAL_DATA_MODEL.md | v0.3 ✓ | Fact/Criterion/Conflict 强制字段已冻结 |
| 04_REPORT_IR.md | v0.3 | IR Schema 完整冻结 |
| 05_EVALUATION.md | **v0.2（内容已是V2.1加固）** | 标签滞后，内容完整 |
| 07_DECISIONS.md | v0.1 | 26条有效决策记录 |

---

# 对象定义状态清单

## 已正式定义（有强制字段表）

| 对象 | 定义位置 | 强制字段表 | 跨文档引用 |
|------|---------|-----------|-----------|
| **Fact** | 03 §8 | id / fact_type / value·unit·quantity_kind / source_refs[] / method / provenance / confidence / review_status / status | 02/03/04/05 全部一致 |
| **SourceRef** | 03 §5 | source_id / location | 02/03/04 一致 |
| **Conflict** | 03 §34 | fact_id / candidates[] / resolution_policy / resolved_by / resolution | 02/03 一致 |
| **Criterion** | 03 §25.1 | criterion_id / value·unit·quantity_kind / condition / source_refs[] / review_status / rule | 02/03/04/05 一致 |
| **Review Status** | 03 §8/§14/§25.1 | pending / confirmed / rejected | 02/03 一致 |
| **Method** | 03 §12 | measured / observed / computed / quoted / inferred | 03 自洽 |
| **Provenance** | 03 §13 | program / ai / human | 03 自洽 |
| **Status** | 03 §15 | filled / missing / conflict / rejected / superseded | 02/03 一致 |
| **fact_type 注册表** | 03 §8.1 | scope_class 枚举 + quantity_kind 映射 | 02/03 一致 |
| **三段式ID** | 03 §8.1 | fact:\<scope_class\>.\<instance_key\>.\<attribute_path\> | 03/04 一致 |
| **supersedes** | 03 §15.5 | 更正链字段 | 03 完整 |
| **quantity_kind** | 03 §10 | 取值范围由注册表维护 | 03/05 一致 |
| **Report IR** | 04 完整文档 | Document/Section/Block/Paragraph双模态/Ref/Lit/Table 等 | 02/03/04/05 一致 |
| **Ref** | 04 §12/§16 | ref 字符串，无前缀格式限定（fact:/evaluation:/criterion:） | 03/04 一致 |
| **anchors** | 04 §13.1/§41.2 | token→fact_id绑定 | 02/04/05 一致 |
| **covers** | 04 §42.1 | Evaluation id列表 | 02/04/05 一致 |
| **whitelist_registry** | 04 §41.2.4/05 §3.3 | pattern/reason/scope/confirmed_by | 02/04/05 一致 |
| **Evaluation Case** | 05 §5 | meta.json / ground_truth / known_issues / expected_issues | 02/05 一致 |

## 已提出但未冻结（有概念、无强制字段表）

| 对象 | 定义位置 | 问题 | 影响 |
|------|---------|------|------|
| **Evaluation** | 03 §26 | 只有概念描述和示例，无 Fact/Criterion 那样的强制字段表 | 对 M0 实现无阻塞（CDML第一阶段只到Criterion），M2前必须补齐 |
| **Conclusion** | 03 §27 | 只有概念描述，无强制字段表 | 同上，M2前补齐 |
| **ConclusionRule** | 03 §27.1 | 有概念级字段列表（id/applies_to/inputs/logic/output/source_ref），但非正式强制字段表 | M2前补齐 |
| **Evidence** | 03 §28 | 概念级，无强制字段表 | CDML第一阶段不实现Evidence，可延期 |
| **RawSource** | 03 §3–§4 | 概念级定义，无强制字段表 | 第一阶段RawSource是RawSource→SourceRef映射，不影响实现 |
| **value_domain** | 03 §16.4 | 定性Fact专用，概念级（枚举列表或词表引用） | 第一阶段实现时按示例实现即可，无结构风险 |

---

# 跨文档引用一致性检查

| 检查方向 | 结果 | 备注 |
|---------|------|------|
| 02→03 | ✓ 一致 | 四层模型、Fact类型层、Criterion闸门、supersedes 全部对齐 |
| 02→04 | ✓ 一致 | 双模态、锚定规则（纯绑定）、Ref无format、covers[] 全部对齐 |
| 02→05 | ✓ 一致 | 评测三原则、分级、任务0 全部对齐 |
| 03→04 | ✓ 一致 | Ref可引用fact:/evaluation:/criterion:；三段式ID与Ref匹配；边界声明对齐 |
| 03→05 | ✓ 一致 | quantity_kind量纲检查、missing/conflict评测、Criterion闸门 全部对齐 |
| 04→05 | ✓ 一致 | 锚定规则R1-R6与§6.4完全对齐；白名单登记制与§3.3完全对齐；covers[]与§6.6完全对齐 |

---

# 发现的结构性问题

## 必须修复（阻塞Step 7-B）

**无。** 基线无结构性冲突。

## 可以延期修复（不阻塞实现）

| # | 问题 | 位置 | 性质 | 修复时机 |
|---|------|------|------|---------|
| 1 | **05_EVALUATION.md版本号滞后** | 文档头部标注v0.2，但内容已包含V2.1加固 | 标签不一致，不是结构冲突 | M0冻结前，文档修复即可 |
| 2 | **03 §2表格描述过期** | L2行写"带来源与置信度"，但§14已将confidence降为诊断字段 | 文字描述滞后 | M0冻结前更新表格描述为"带来源、状态与判定方法" |
| 3 | **02 §17.1说"三份JSON Schema"但§17.2列了5项** | contracts/模块清单写三份，M0清单列5项（Fact/Criterion/IR/Evaluation Case/白名单） | 文字不一致 | M0冻结前修正数字 |
| 4 | **02 §6.4 required_facts示例ID不完整** | 示例用 `"fact_type": "concrete_strength"` 但03 §8.1用 `"component.concrete_strength"` | 示例路径不一致，根因是scope_class注册表未冻结具体列表 | M0冻结fact_type注册表时统一修正 |
| 5 | **04 §5 Document示例中version仍写0.1** | 文档版本已是v0.3，但示例中的 `"version": "0.1"` 未更新 | 示例滞后 | M0冻结IR Schema时一并修正 |
| 6 | **Ref前缀中缺conclusion:** | 04 §16只列fact:/evaluation:/criterion:，但Conclusion也是结构化对象 | 轻微，可以实现时补齐 | M2实现Conclusion前 |
| 7 | **Evaluation强制字段表缺失** | 03 §26无正式强制字段表 | 可以从§26概念+示例+§27.1 ConclusionRule inputs反推 | M2前补齐 |

---

# 不应该修改的问题

| # | 内容 | 理由 |
|---|------|------|
| 1 | confidence保持诊断字段 | D-015已决策，不得改回闸门 |
| 2 | SI基准保持作废 | D-008已推翻，不得恢复 |
| 3 | 锚定保持纯绑定，不回加全文值集合兜底 | D-017已推翻D-006兜底规则，不得回退 |
| 4 | Ref不带format | D-025已推翻，真相源统一到TemplateSpec |
| 5 | 段落保持双模态 | D-005已推翻单模态约束 |
| 6 | Pipeline不回退到五Agent | D-011已推翻 |
| 7 | Criterion判定闸门保持 | D-021，红线不得动 |
| 8 | missing状态不得静默补全 | D-003，红线不得动 |
| 9 | Conflict保持manual resolution第一阶段 | D-003/03 §36 |

---

# 结论：是否可以开始 Step 7-B？

**可以。**

全部阻塞项检查通过：

1. **Fact Schema（含三段式ID + fact_type + supersedes + status五态）** — 已冻结，有完整强制字段表
2. **Criterion Schema（与Fact同构 + review_status判定闸门）** — 已冻结，有完整强制字段表
3. **Conflict数据结构** — 已冻结，有完整字段表
4. **SourceRef** — 已冻结
5. **review_status / method / provenance / status / quantity_kind 枚举** — 全部冻结
6. **fact_type注册表结构** — 已冻结（scope_class枚举 + quantity_kind映射）
7. **跨文档引用** — 02→03→04→05 全部对齐，无断裂
8. **红线规则** — 全部明确（missing不补全、conflict不静默覆盖、Criterion闸门、锚定纯绑定、Ref无format）

**M0清单中需要实现的对象**（02 §17.2 M0）：
1. Fact Schema + fact_type注册表 + supersedes — ✓ 冻结
2. Criterion Schema + review_status — ✓ 冻结
3. Report IR Schema + anchors + covers — ✓ 冻结
4. Evaluation Case Schema + 白名单登记表 — ✓ 冻结
5. 校验器骨架：Ref可解析 / 锚定绑定 / 白名单命中 — ✓ 规则冻结

**建议延期修复的7项问题**（标签/示例文字不一致）属于文档清理，不是架构阻塞，可以在M0冻结Schema JSON时一并修正为最终版本号。
