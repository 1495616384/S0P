# Step 7-E B5 Minimal Repair — Design Check

> 版本：v2（DESIGN RECONCILIATION 修订：legacy 兼容规则收紧 + Table title/note 路径明确）
> 日期：2026-09-28
> 状态：DESIGN READY FOR IMPLEMENTATION
>
> 本文件为 B5（TABLE-NUMERIC-BINDING）最小修复的正式设计裁决文档，仅含最终结论。
> 冻结依据：Step_7_E_Coding_Contract_v10.md 结尾「B5 修复边界」、04_REPORT_IR §41.2.1 / §41.2.2 / §41.2.3 / §41.2.5、D-018 / D-019 / D-025、05_EVALUATION §6.4。
> 本轮不写代码、不进入 Coding Round、不修改 Step 7-D、不修改 04_REPORT_IR、不处理其他候选问题。

---

## 1. Frozen Baseline

以下内容冻结、不得改变：

| # | 冻结项 | 依据 |
| ---- | ---- | ---- |
| 1 | 7 个 P0 Rule 不增删；B5 仍属 P0-4 | INV-1 / §13 |
| 2 | AnchorDeclaration 结构原样：`token / fact_id / char_start / char_end` | §4（与 04_REPORT_IR §13.1 anchors 对齐） |
| 3 | BIND-8 rendered_segments 侧语义：唯一绑定 + span 覆盖（token span ⊆ 声明 span 或文本相等；部分重叠不算覆盖） | §8.1 / INV-24 / T-18 |
| 4 | §8.2 判定链：token → 唯一 fact_id → G3 → fact_type → snapshot.decimals → frozen_round → P0-1 | §8.2 / §13 / INV-19 |
| 5 | BIND-7 比较规则：`normalized_identifier == parse_fact_id(fact_id).instance_key`，禁止 scope_class 推断 | §6.3 / INV-22 |
| 6 | §8.3 区间逐 token 独立锚定；§8.4 whitelist span containment（部分重叠不豁免） | §8.3 / §8.4 |
| 7 | ANCHOR.* 现有错误码体系，不新增错误码 | §8.1 / §8.2 |
| 8 | Fix 3：Validator 自己 tokenize，不信任系统自报 TokenOccurrence | §5 / §4 |
| 9 | 精度唯一来源 = FactDisplaySpecSnapshot.decimals | D-025 / INV-14 / INV-19 |
| 10 | Tokenizer（§5 正则 / overlap 消解 / unit-start/cont 文法）零改动 | §5（B1 闭合） |
| 11 | ActualIssue 冻结六元 identity + build-only | INV-17 / §11 |
| 12 | Step 7-D 只读；04_REPORT_IR / 05_EVAL / CDM 只读 | INV-2 / 本轮禁止事项 |
| 13 | 冻结要求本身：Table 单元格文本、表标题、表注纳入逐 token 锚定；实例标识 token 绑定 `.id` 类 fact | 04_REPORT_IR §41.2.1 / §41.2.2 / §41.2.3 / §41.2.5、D-018、05_EVAL §6.4 规则 3 |

---

## 2. Root Cause

当前 `TableCell = { raw_text, fact_id }` 的 `fact_id` 是 **cell→fact 的单值映射**，而冻结要求（§41.2.2 / R2）是 **token→fact 的逐 token 映射**。结构上丢失两个维度：

1. **粒度丢失**：一个 cell 有 N 个需锚定 token 时，N:1 坍缩为 1:1。`"2.3m / 2.5m"`（两 token 各属不同 fact）、`"2.0～2.5m"`（min/max 独立绑定）、`"K001：32.4MPa"`（需 `fact:component.K001.id` 与 `fact:component.K001.concrete_strength` 两个不同 fact_type 并存）均不可表达——单个 fact_id 在结构上不可能同时是两种 fact_type。
2. **定位丢失**：cell 级 fact_id 无 char 坐标，无法执行冻结 span 规则（token span ⊆ 声明 span）、无法区分 cell 内哪个 token 是绑定目标、无法支撑 whitelist span containment 与 BIND-6 查重。

因此 Table 侧 BIND-7 / BIND-8 / §8.2 当前全部不可执行——即 v10 裁决的 CONFIRMED BLOCKER（§8.1 作用域注、结尾扫描第 6 项）。

---

## 3. Design Options

| 维度 | A. per-cell binding declarations | B. table-level binding declarations |
| ---- | ---- | ---- |
| 表达能力 | 每个 cell 自带声明集，四类场景（单 token / 多 token / range / identifier+numeric）全部可表达；不同 cell 中相同 token 文本（如两行都是 "32.4MPa" 属不同构件）天然无歧义 | 声明集中于表级袋，必须额外携带 cell 定位信息才能区分同文本 token：① 改 AnchorDeclaration 加字段 → 违反硬约束 5；② 引入表文本规范化展平约定 → 新增影子文本坐标系，与各 cell raw_text 可能漂移；③ 仅按 token 文本匹配 → 同文本不同 fact 不可区分，判 DUPLICATE_BINDING 或错绑，均不健全 |
| 定位复杂度 | O(1)：声明与被 tokenize 的文本同对象共置，tokenize 后直接在本 cell 声明集内做 span 覆盖查找——与 rendered_segments 侧同一算法 | O(tokens × declarations)：每个 token 需先做坐标换算（cell 坐标 → 展平坐标）或字段过滤，再查表级袋；引入当前不存在的坐标变换机制 |
| 改动面 | TableCell 增加一个可选字段；StructuredTable / AnchorDeclaration / 渲染侧全部不动 | StructuredTable 增字段 + AnchorDeclaration 变结构或新增展平语义 + Validator 查找机制，三处改动起步 |
| 兼容性 | 纯增量：默认空列表，旧载荷合法；cell.fact_id 以收紧后的单 token legacy 路径保留（§4），不强制迁移 | 表级集中声明迫使单 token cell 的绑定也迁入表级袋，或保留 cell 级旧路径 → 双拓扑并存，机制翻倍 |
| 拓扑一致性 | 与 B4 冻结拓扑同构：声明挂在文本载体自身（RenderedSegment 挂 prose，TableCell 挂 raw_text），仅载体粒度更细 | 引入与 B4 不同的第二种锚定拓扑（声明挂上层容器），偏离已冻结的声明—文本共置惯例 |

---

## 4. Decision

**选择 A：per-cell binding declarations。**

1. **最小修改面**：唯一结构变化 = TableCell 增加可选 `anchor_declarations: list[AnchorDeclaration]`（默认空）。AnchorDeclaration 原样复用，StructuredTable、渲染侧、tokenizer、错误码、判定链零改动。
2. **不破坏 B4**：声明挂在哪个载体是本轮唯一设计自由度；本方案只是把 B4 已冻结的「声明与文本共置」惯例应用到 TableCell 这个新载体。`char_start / char_end` 语义 = 相对所挂载文本载体的偏移（渲染侧即 RenderedSegment.raw_text 的偏移，Table 侧即 cell.raw_text 的偏移）——同一惯例的载体推广，属表达能力补齐，非语义变化。RenderedSegment 侧任何字段、任何检查、任何语义均不触碰。
3. **表达能力**：
   - 单 token cell `"2.3m"`：显式声明 `{token:"2.3m", fact_id, char_start:0, char_end:4}`；或走 legacy 路径 cell.fact_id（见下）。
   - 多 token cell `"2.3m / 2.5m"`：两条显式声明，各自 char span、各自 fact_id。
   - range cell `"2.0～2.5m"`：两条显式声明独立绑定（"～" 非 unit-start，两 token 独立提取），满足 §8.3。
   - identifier+numeric cell `"K001：32.4MPa"`：两条显式声明 `{K001 → fact:component.K001.id}`（BIND-7 判 instance_key）、`{32.4MPa → fact:component.K001.concrete_strength}`（BIND-8 → §8.2）。
4. **Validator tokenize 后如何定位声明**：对每个 TableCell.raw_text 独立执行既有 tokenizer（§5 正则 + §5.2 overlap 消解，Fix 3 不变），提取 token 及其 char span（坐标基 = 该 cell.raw_text）；§8.4 whitelist span containment 在同一 cell 文本上先行判定；未豁免 token 在**本 cell** 的 anchor_declarations 中按冻结 span 规则找覆盖声明：0 条 → UNANCHORED；≥2 条不同 fact_id → DUPLICATE_BINDING；唯一声明 → BIND-5 语法校验 → §8.2 链。headers 与 rows 均为 TableCell，机制一视同仁，无特例。
5. **span / duplicate / invalid 语义保持**：span 规则、BIND-6/7/8、§8.4 豁免全部原样适用——声明集只是换了挂载载体，查找谓词与错误路径与渲染侧逐字节相同。

### 4.1 Legacy 兼容规则（收紧版，替代「整 cell 隐式 AnchorDeclaration」）

`TableCell.fact_id` 作为旧版兼容表达，**生效条件**（两条同时满足）：

- (a) 该 cell 的 `anchor_declarations` 为空；
- (b) 该 cell 经 §8.4 whitelist 豁免后，**恰好含一个需要锚定的 token**。

「需要锚定的 token」计数口径：未被 whitelist 豁免的 numeric token（BIND-8 目标）+ 未被 whitelist 豁免的 identifier token（BIND-7 目标）；whitelist 豁免先行判定，豁免 token 不计入。

生效时：`cell.fact_id` 等价于对该**唯一需要锚定 token** 的一条绑定（span 覆盖该 token），进入既有 BIND-5/6/7/8 与 §8.2 链。
不满足任一条件时：`cell.fact_id` **不提供任何绑定**（inert）；未被显式声明覆盖的 token → 既有 `ANCHOR.UNANCHORED`。

推论（与 04_REPORT_IR §41.2.3 逐 token 冻结语义严格对齐）：

| cell 内容 | 仅 cell.fact_id | 结论 |
| ---- | ---- | ---- |
| `"2.3m"`（1 个需锚定 token） | 生效 | legacy 路径 PASS |
| `"K001"`（1 个 identifier token，fact_id=…K001.id） | 生效 | BIND-7 instance_key 判定 |
| `"2.3m / 2.5m"`（2 个 numeric） | inert | 两 token UNANCHORED → 必须用 anchor_declarations |
| `"2.0～2.5m"`（2 个 numeric） | inert | 同上 |
| `"K001：32.4MPa"`（identifier + numeric = 2 个） | inert | K001、32.4MPa 均 UNANCHORED → 必须双绑定（.id + 量值），§41.2.3 不被 cell 级单值冒充 |

约束确认：

- 不允许用单个 cell.fact_id 覆盖多个不同语义 token——多 token cell 中 fact_id 完全 inert，由既有 UNANCHORED 强制「必须使用 anchor_declarations」。
- 不新增错误码、不新增 Validator stage。
- 不修改 BIND-7 比较规则（`normalized_identifier == parse_fact_id(fact_id).instance_key`）。
- 不修改 BIND-8 核心语义、不修改 §8.2。
- cell.fact_id 与显式声明并存时：(a) 不满足 → fact_id inert，显式声明为唯一绑定路径（显式声明优先，无重复计数问题）。

本次仅为 legacy compatibility 收紧，不是重写 BIND-7 / BIND-8。

---

## 5. Minimal Change Surface

### MUST CHANGE（共 4 项，全部为表达能力补齐 / 路径声明，无语义变化）

| # | 对象 | 变化 | 性质 |
| ---- | ---- | ---- | ---- |
| 1 | `TableCell` | 增加可选字段 `anchor_declarations: list[AnchorDeclaration] = []` | 表达能力补齐（纯增量，默认空，旧载荷合法） |
| 2 | `TableCell.raw_text` | 明确为 declarations 的 char 坐标基准（char_start / char_end 相对 cell.raw_text） | 坐标基准登记（B4 同一偏移惯例的载体推广） |
| 3 | `TableCell.fact_id` | 明确仅作为「单需锚定 token cell 的 legacy compatibility path」（§4.1 收紧规则） | 兼容性收紧登记，非新机制 |
| 4 | Table title / note | 明确必须进入 RenderedSegment anchoring path（见 §8） | 路径归属声明，零 schema 变化 |

### MUST NOT CHANGE（14 项）

| # | 对象 |
| ---- | ---- |
| 1 | AnchorDeclaration 结构（token / fact_id / char_start / char_end） |
| 2 | RenderedSegment 结构 |
| 3 | BIND-7 比较规则（normalized_identifier == parse_fact_id(fact_id).instance_key） |
| 4 | BIND-8 核心判定规则（span 覆盖 + 唯一 fact_id + 错误码） |
| 5 | §8.2 numeric comparison chain（token → fact_id → G3 → snapshot.decimals → P0-1） |
| 6 | §8.3 range semantics（区间逐 token 独立锚定） |
| 7 | §8.4 whitelist semantics（span containment，部分重叠不豁免） |
| 8 | Tokenizer（§5 正则 / overlap 消解 / unit 文法） |
| 9 | TrustedInput Schema |
| 10 | ActualIssue identity（冻结六元组 + build-only） |
| 11 | ExpectedIssue matching（(rule_id, fact_id) + 双向 one-to-one） |
| 12 | 7 P0 Rules（不新增、不删除） |
| 13 | Step 7-D（只读） |
| 14 | 04_REPORT_IR（只读） |

---

## 6. Binding Execution Flow

Table 侧与 rendered_segments 侧为同一 BIND-8 管道的两个载体入口：

```
【载体 1】TableCell.raw_text（headers / rows 一视同仁）
    ↓ Validator 自行 tokenize（Fix 3：§5 同一正则 + §5.2 同一 overlap 消解）
numeric token / identifier token（char span，坐标基 = cell.raw_text）
    ↓ whitelist span containment（§8.4 同一规则：完全包含 → 豁免，仍出诊断）
    ↓ 未豁免
绑定来源二选一：
    ├─ anchor_declarations 非空 → 在本 cell 显式声明集中按冻结 span 规则找覆盖声明
    │      （token span ⊆ 声明 span 或文本相等；部分重叠不算覆盖）
    └─ anchor_declarations 为空 且 cell 内恰有 1 个需锚定 token 且 cell.fact_id ≠ None
           → legacy 路径：cell.fact_id 作为该唯一 token 的绑定（§4.1）
    ├─ 0 条覆盖            → ANCHOR.UNANCHORED
    ├─ ≥2 条不同 fact_id   → ANCHOR.DUPLICATE_BINDING
    └─ 唯一绑定 → fact_id
            ├─ BIND-5：is_valid_fact_id() 失败 → ANCHOR.INVALID_FACT_ID（Stage D）
            ├─ identifier token → BIND-7：normalized_identifier == instance_key
            │        不等 → ANCHOR.IDENTIFIER_BINDING_MISMATCH
            └─ numeric token → §8.2 链（与渲染侧逐字节同一）：
                G3 查找 → 缺 → ANCHOR.INVALID_FACT_ID（Stage E）
                parse_fact_id → fact_type → fact_display_spec[fact_type]
                    缺条目 → precondition_error
                → FactDisplaySpecSnapshot.decimals
                → frozen_round(token_value, decimals) == G3.value   （P0-1）
                （并行：unit 级联，UNKNOWN_UNIT / QUANTITY_KIND_MISMATCH / UNIT_MISMATCH）

【载体 2】Table title / Table note（作为 RenderedSegment 提交，见 §8）
    ↓ 与 prose / 图注完全同一管道：
RenderedSegment.raw_text
    → tokenize → whitelist → anchor_declarations 覆盖查找
    → BIND-7 / BIND-8 → §8.2
```

**同一 BIND-8 核心语义的保证**：两个载体共享同一 tokenizer、同一 span 覆盖谓词、同一 whitelist 豁免、同一错误码集、同一 §8.2 判定链与精度来源。唯一差异是声明所挂载的文本载体（RenderedSegment.raw_text vs TableCell.raw_text）。B4 渲染侧管道一行不改。

---

## 7. Identifier Boundary

**B5 不新增任何 identifier 语义。**

- §41.2.3 示例（`K002 → fact:component.K002.id`、`32.4MPa → fact:component.K002.concrete_strength`）要求的只是同一 cell 内两条独立 AnchorDeclaration 的表达能力——per-cell declarations 直接提供。
- identifier token 的全部判定仍由冻结 BIND-7 完成：被至少一条声明覆盖（span 规则同 BIND-8）+ `normalized_identifier == parse_fact_id(fact_id).instance_key`（INV-22）。该规则在 Table 侧的执行结果与 rendered_segments 侧完全一致，零改动。
- 收紧后的 legacy 规则反而强化 §41.2.3：多 token cell 中 cell.fact_id 不再能经 instance_key 巧合替 identifier token 提供绑定，`"K001：32.4MPa"` 必须显式声明 `{K001 → …K001.id}`。

边界声明（仅划定界线，不裁决、不登记为阻塞）：BIND-7 冻结文本只校验 identifier 自身与其绑定 fact 的 instance_key 一致性；「同一 cell 内 numeric token 所绑 fact 的 instance_key 是否必须与 co-located identifier 一致」这一更强交叉校验不属于冻结 BIND-7 语义，B5 不新增、不修改。

---

## 8. Table Title / Note Boundary

**结论：Table title / Table note 属于 rendered_segments anchoring 输入，路径在本轮明确声明。**

- 冻结依据：04_REPORT_IR §41.2.1 明确将「Table 的单元格文本与表标题 / 表注」列入锚定范围；05_EVALUATION §6.4 规则 3 同。
- 路径声明（零 schema 变化）：

```
Table title  → RenderedSegment（segment_location 标注表定位，如 table-2-1.title）
Table note   → RenderedSegment（如 table-2-1.note）
             → 现有 BIND-7 / BIND-8 → 现有 §8.2
```

- 系统必须将 Table title / note 作为 RenderedSegment（携带自身 anchor_declarations）提交；Validator 对其执行与 prose / 图注完全同一的既有管道，无特例、无新代码路径。
- **不新增 Table schema**：StructuredTable 保持 `table_id / headers / rows`；不添加 title / note 字段；不修改 04_REPORT_IR。`segment_location` 为既有自由字符串，足以承载表定位标注，不新增类型。
- 边界注记（不构成本轮设计缺口）：title / note 的**提交完整性**（系统是否漏交 title/note）依赖 trusted 侧表格规格（trusted_table_specs / TableSpecSnapshot，属旧契约扫描候选第 3 项），不在 B5 范围内新增检查，避免扩大 Validator 职责。

---

## 9. Acceptance Criteria

B5 修复完成后，以下案例必须可表达且可判定（错误码均为既有 ANCHOR.* / P0-1）：

| # | 案例 | 表达要求 | 判定 |
| ---- | ---- | ---- | ---- |
| 1 | `"2.3m"` | 单 token：可用 cell.fact_id（legacy，§4.1 生效条件满足）或显式声明 | BIND-8 覆盖 → §8.2 值判定；两条路径均闭合 PASS |
| 2 | `"2.3m / 2.5m"` | **必须**两条 AnchorDeclaration、两个独立 fact_id；仅 cell.fact_id → inert → 两 token UNANCHORED | 两 token 独立通过 BIND-8 → 各自 §8.2 |
| 3 | `"2.0～2.5m"` | **必须**两条独立 AnchorDeclaration（"～" 非 unit-start，两 token 独立提取，第二个带单位 m） | 各自独立进入 §8.2；禁止区间整体锚定（§8.3） |
| 4 | `"K001：32.4MPa"` | **必须**逐 token 双绑定：K001 → …K001.id（identifier binding，BIND-7）；32.4MPa → …K001.concrete_strength（numeric binding，BIND-8 → §8.2）；仅 cell.fact_id → inert → K001、32.4MPa 均 UNANCHORED | 禁止单个 cell.fact_id 覆盖整个 cell；identifier 错绑（instance_key 不等）→ IDENTIFIER_BINDING_MISMATCH |
| 5 | whitelist 豁免 numeric token（如 `"依据 GB 50292-2015 检测"`） | 无需绑定声明 | token span 完全包含于登记 pattern 匹配 span → 豁免 BIND-8（仍出诊断）；部分重叠 → 不豁免 → UNANCHORED |
| 6 | duplicate binding | 同一 token 被 ≥2 条**不同 fact_id** 声明覆盖 | ANCHOR.DUPLICATE_BINDING |
| 7 | partial overlap | 声明 span 与 token span 相交但不包含 | 不算覆盖 → ANCHOR.UNANCHORED（冻结 span 规则） |
| 8 | invalid fact_id | 声明 fact_id 未过 `is_valid_fact_id()`（Stage D / BIND-5）；或格式合法但 ∉ 用例 G3（Stage E） | ANCHOR.INVALID_FACT_ID（两级定义与 §8.2 注一致） |
| 9 | value mismatch | 绑定有效、fact 合法，但 `frozen_round(token_value, snapshot.decimals) != G3.value` | P0-1 VALUE.CONSISTENCY FAIL（精度唯一来源 = snapshot.decimals） |
| 10 | Table title / note 含 numeric token | title/note 作为 RenderedSegment 提交并携带 anchor_declarations | 进入既有 BIND-7/BIND-8 → §8.2，与 prose 同一判定 |

---

## 10. Final Status

一致性检查（八项条件逐项核对）：

| 条件 | 结果 |
| ---- | ---- |
| A. per-cell binding declarations 已确定 | ✅（§4） |
| B. legacy cell.fact_id 兼容规则不绕过 token-level binding | ✅（§4.1 收紧规则：多 token cell 中 fact_id inert，由既有 UNANCHORED 强制逐 token 声明；§41.2.3 双绑定不被 cell 级单值冒充） |
| C. Table title/note 已明确进入现有 RenderedSegment anchoring path | ✅（§8 路径声明，零 schema 变化） |
| D. B4 rendered_segments 零改动 | ✅（§5 MUST NOT CHANGE 2/4/5；§6 同一管道声明） |
| E. 没有修改 04_REPORT_IR | ✅（只读引用） |
| F. 没有修改 Step 7-D | ✅（只读引用） |
| G. 没有新增 P0 | ✅（仍 7 条，B5 属 P0-4） |
| H. 没有引入新的 Validator 职责 | ✅（无新 stage、无新错误码、无新 trusted 字段；仅既有检查作用域按冻结要求覆盖 Table 载体） |

最小性确认：真正改变的对象仅三项——TableCell（+ 可选 anchor_declarations 及坐标基准登记）、BIND-7/BIND-8 对 TableCell 的作用域声明、Table title/note 的 rendered_segments 路径声明。无 StructuredTable redesign、无 Table flattening、无全局表格坐标系、无新 Anchor 类型、无新 Validator Stage。

**DESIGN READY FOR IMPLEMENTATION**
