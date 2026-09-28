收到。本轮是**边界裁决轮**，不是修复轮：必须依据在盘冻结文档（04_REPORT_IR v0.3.1.1）正式裁决 A/B，禁止设计 schema。先做裁决分析，再同步文档状态。

关键证据我已经掌握（04_REPORT_IR §41.2.1/§41.2.2/R2/R3/§41.2.5 此前已全文读取）。核心判定链：冻结判定粒度是 **token 级**，而 cell 级单 fact_id 只能覆盖单 token cell。现在把裁决结果落盘（撤销 CANDIDATE、升级 CONFIRMED BLOCKER、契约回退 NOT READY）。
The user is asking for a **TABLE NUMERIC BINDING DECISION CHECK** — a boundary adjudication, not a design task. Must answer:

A. Is cell-level TableCell.fact_id sufficient to satisfy 04_REPORT_IR §41.2.1?
or
B. Must support per-numeric-token independent fact_id binding within a cell?

And output:
1. 04_REPORT_IR 当前冻结要求
2. 当前 Step 7-E SystemOutput 能表达什么
3. 当前 Step 7-E 不能表达什么
4. 是否构成当前 Contract 的确定性可执行性缺口
5. 是否属于既有冻结回归
6. 下一步最小修复边界

Key rules:
- Don't design new TableCell schema
- Don't add P0 rules
- Don't modify BIND-8 numeric core semantics
- Don't modify 04_REPORT_IR
- Don't modify Step 7-D
- Don't create v11
- "不是确定性回归" ≠ "可以 READY" — if the contract cannot execute the frozen requirement → NOT READY
- Only if Table frozen requirements are covered, OR formally prove cell-level expression covers all frozen scenarios → continue READY check

So this is an ANALYSIS + DECISION task. I need to:

1. Re-read 04_REPORT_IR §41.2.1 and §41.2.5 precisely to determine what the frozen layer actually requires for tables.
2. Analyze the current SystemOutput/TableCell expression power.
3. Answer A or B with evidence.
4. Make the READY/NOT READY determination.

Let me think through the frozen requirements carefully.

**04_REPORT_IR §41.2.1 (锚定范围):**
"AI 可写数字的载体不止 Narrative 段。锚定校验必须覆盖三类载体：
1. Narrative 段的 prose；
2. Table 的单元格文本与表标题 / 表注；
3. Figure 的图片说明（caption）。
其中：表格数值若来自 Ref，走渲染前校验（§41.1）；若为自由文本，走本节锚定校验。两者都不允许出现无来源数字。"

Also §41.2.2 (判定规则): "渲染后的文本（Narrative prose / Table 单元格 / Figure caption）→ 提取全部数值 token（含单位：2.3m / 0.15mm / 1250.30m²）→ 逐个 token 判定 ├── 显式绑定 fact_id 且 value(token) == fact.value → 通过 └── 命中登记制白名单（§41.2.4）→ 通过 → 任一未命中 → P0 error，构建失败"

R2: "数值 token 必须显式绑定 fact_id，判定取「值相等」而非「值存在」"
R3: "实例标识 token（构件 / 测点 / 样品 / 文件编号）必须与数值 token 同表绑定"
R5: "校验对象是渲染产物，不是 IR 中的声明"

§41.2.5 note: "近似与区间表达（「约 2.3m」「2.0～2.5m」）按 token 逐个锚定：区间内每个数字都需独立绑定并值相等，不允许用区间整体作为一次锚定。"

Also §22 Table example: cells are Ref or Lit. §21 TableSpec: Cell ∈ {Ref | Lit | Computed}. So in the frozen IR design, table cells can be Refs (structured path → §41.1 render-time validation) OR free text (anchoring path).

Key insight from §41.2.1: "表格数值若来自 Ref，走渲染前校验（§41.1）；若为自由文本，走本节锚定校验。"

So for TABLES specifically, the frozen layer has TWO paths:
- Ref path: cell = Ref → render-time Ref validation (§41.1): Fact exists, not missing, not conflict. The value comes from the Fact itself — no token-value comparison needed (the renderer formats it from the Fact). Value correctness is inherited from the Fact.
- Free-text path: cell = free text containing numbers → anchoring validation (§41.2): every numeric token must be explicitly bound to a fact_id with value equality.

Now — the current Step 7-E SystemOutput:
- `structured_tables: list[StructuredTable]`, `TableCell = {raw_text, fact_id: Optional[str]}`
- `rendered_segments: list[RenderedSegment]` with AnchorDeclaration bindings — BIND-8 covers these.

The question: does cell-level fact_id suffice?

Analysis by scenario:

**Scenario 1: Cell with exactly one numeric token.** E.g. "2.3m" with cell fact_id = fact:defect.C001.length. The single numeric token binds to the cell's fact_id → chain works: token → cell fact_id → G3 → snapshot.decimals → value comparison. ✓ Expressible.

But wait — can a cell ALSO be a Ref? In the current Step 7-E SystemOutput, TableCell has raw_text + fact_id. If the cell was rendered from a Ref, the raw_text is the rendered value and fact_id is the referenced fact. That's actually the Ref path expressed in Step 7-E terms: cell fact_id = the Ref's fact. Value correctness: the renderer would have formatted the Fact value with the display spec — so token value should equal frozen_round(G3.value, decimals)... actually the rendered text IS the formatted value, so comparison frozen_round(token_value, snapshot.decimals) == G3.value works the same. Hmm, interesting — so for single-numeric cells, both Ref-path and free-text single-number cells collapse to the same checkable form: one cell, one numeric token, one fact_id.

**Scenario 2: Cell with multiple numeric tokens belonging to the same fact.** E.g. "2.3m（最小 2.1m）" — hmm, actually if both belong to different facts... let me do same-fact first: e.g. "2.3m × 2.3m" (width × length of same defect? no — width and length are different facts). Same fact example: hmm, "0.15mm（0.15）"? Contrived. Actually same-fact multi-number cells are rare. The real case: **different facts per token**.

**Scenario 3: Cell with multiple numeric tokens from different facts.** E.g. crack width table row: "0.12 / 0.15 / 0.18" (three measurement points' widths in one cell) — each belongs to a different fact (defect.C001.width, defect.C002.width, defect.C003.width)? Hmm, or more realistically a summary cell "最大 0.18，最小 0.12" — two facts (max, min) or same fact computed... In engineering tables: "2.3m / 2.5m" (user's example) — two crack lengths of two defects, or min/max of one measurement set. Cell-level single fact_id CANNOT express two different facts.

Also user's second example: "K001：32.4MPa" — identifier token + numeric token in one cell. R3 requires identifier token binding (K001 → fact:component.K001.id) AND numeric token binding (32.4 → fact:component.K001.concrete_strength). Cell-level single fact_id can't express both bindings — unless... hmm, the cell fact_id = concrete_strength fact, and the identifier K001's binding could be DERIVED: parse_fact_id(cell.fact_id).instance_key == "K001" — the identifier binding to fact:component.K001.id could be inferred from the numeric binding's fact_id instance_key! Hmm — interesting. Is that legitimate? R3 says identifier must be bound to fact:<scope>.<instance>.id. If the cell declares fact_id = fact:component.K001.concrete_strength, then the identifier K001's required binding fact:component.K001.id is *derivable* (same instance_key). But is that "explicit binding"? R3/R2 require explicit binding declarations. Deriving K001 → K001.id from the numeric's fact_id is an inference — the identifier IS present in the same cell and the fact_id IS present... Hmm. But that's designing. Let me not go down this path too far — the decision check needs to determine A vs B, and the identifier-in-cell case is part of the evidence.

**Scenario 4: Table 标题 / 表注 (table titles and notes).** §41.2.1 includes 表标题 / 表注 as anchoring carriers. Where do titles/notes live in current SystemOutput? StructuredTable = {table_id, headers, rows}. Headers = list[TableCell]. But 表标题 (table caption/title) and 表注 (table notes) — NOT represented in StructuredTable at all! Hmm wait — are they in rendered_segments? The R10 contract: rendered_segments = "渲染后按段落/表格/图注切分" — hmm, the original v10 wording said rendered_segments covers 段落/表格/图注. After R10, RenderedSegment has segment_location (semantic path + block id). Table captions/notes in 04_REPORT_IR are part of the Table block (表标题/表注 per §41.2.1). 

Hmm — actually where would "表3-2 各层混凝土强度" (table title) appear? If SystemOutput's rendered_segments includes table title/note text as segments (with segment_location pointing at the table block), then BIND-8 covers them (they're rendered_segments tokens). If they're not in rendered_segments and not in StructuredTable → gap. Current contract: StructuredTable has no caption/notes fields; rendered_segments scope per BIND-8 note says "prose / 图注" — hmm, I wrote "rendered_segments（prose / 图注）" in the BIND-8 scope note. That wording might exclude table titles/notes! Hmm. Actually the R10 §4 comment said "rendered_segments: list[RenderedSegment] # 结构化渲染段（禁止退化为 list[str]）" and the original v10 line said "渲染后按段落/表格/图注切分" — ambiguous about table titles.

OK so now the decision: A vs B.

**Arguments for A (cell-level sufficient):**
- In the frozen IR design (§21/§22), table cells SHOULD be Refs — the recommended path is "表格不应该由 LLM 直接生成最终自由文本" (§20), TableSpec with Cell ∈ {Ref | Lit | Computed}. For Ref cells, value correctness is guaranteed by §41.1 render-time validation + the display spec; the anchoring value-equality check is redundant-ish (the rendered text is formatted FROM the fact).
- For Lit (free text) cells — §20 says lists/tables items "仍然应采用结构化 Inline 表示，而不是绕过 Report IR 的事实引用机制" — free-text cells with multiple numbers from different facts would violate the spirit of §20.

**Arguments for B (per-token binding required):**
- §41.2.1 explicitly says: "表格数值若来自 Ref，走渲染前校验（§41.1）；**若为自由文本，走本节锚定校验**。两者都不允许出现无来源数字。" — the frozen layer EXPLICITLY contemplates free-text table cells going through anchoring.
- §41.2.2: "渲染后的文本（… Table 单元格 …）→ 提取全部数值 token → 逐个 token 判定：显式绑定 fact_id 且值相等" — **逐个 token** — per-token binding is the frozen判定粒度. R2: "数值 token 必须显式绑定 fact_id" — token-level.
- §41.2.5: 区间表达 "按 token 逐个锚定：区间内每个数字都需独立绑定" — if a cell contains "2.0～2.5m", the two numbers may belong to two different facts (min/max) → requires two independent bindings.
- Real engineering tables DO have multi-number cells: crack width statistics "0.12～0.18"（min～max of a measurement set → two facts or computed values), "2.3m / 2.5m" (user's example).
- R5: "校验对象是渲染产物，不是 IR 中的声明" — the Validator checks the rendered artifact. A rendered cell "2.3m / 2.5m" has two tokens; per-token binding is required by §41.2.2 regardless of how the IR declared it.

So the frozen layer's判定粒度 is unambiguous: **per numeric token** (逐个 token 判定, R2, §41.2.5). The question is whether cell-level fact_id can satisfy it.

Cell-level fact_id can satisfy per-token binding ONLY IF every cell contains at most one numeric token (or multiple tokens all belonging to that one fact AND... wait, even same-fact multi-token: cell fact_id covers both tokens — both bind to the same fact — but then value equality must hold for EACH token against the SAME G3.value — if cell is "2.3m × 2.3m" both 2.3, fine. But "2.3m 和 2.5m" same fact? Can't both equal G3.value unless equal numbers. So multi-token cells realistically involve either different facts or non-G3 numbers (which must be whitelisted or anchored elsewhere)).

So: cell-level fact_id is sufficient ONLY under the restriction "each cell contains ≤ 1 numeric token (or all tokens same fact & same value)". The frozen layer does NOT impose that restriction — §41.2.2 explicitly contemplates free-text cells with multiple tokens ("提取全部数值 token，逐个 token 判定"), and §41.2.5 explicitly requires per-token anchoring for ranges WITHIN any carrier including table cells.

Therefore: **A is insufficient; B is required by the frozen layer.** The current contract has a deterministic executability gap for the free-text multi-number cell scenario.

Hmm wait — but let me steelman A once more. Could one argue that multi-number free-text cells are prohibited by the IR layer (§20: tables should use Ref/structured Inline)? §20 says "表格不应该由 LLM 直接生成最终自由文本" and "列表项的正文仍然应采用结构化 Inline 表示" — recommended, and §41.2.1 STILL explicitly handles the free-text case ("若为自由文本，走本节锚定校验"). The frozen validation layer must handle what §41.2.1 says it handles. The Validator cannot assume cells are single-number; §41.2.2's "提取全部数值 token" presumes multi-token extraction. And R5 says the check target is the rendered artifact — the Validator doesn't control whether the System emitted a multi-number cell; it must be able to JUDGE it. If the contract can't express judgments for multi-number cells, P0-4 cannot prove coverage of the frozen carrier set.

Also the identifier case: "K001：32.4MPa" — R3 requires identifier token binding same-table. Cell fact_id (concrete_strength) gives instance_key derivable, but the identifier's own binding to fact:component.K001.id is a separate declaration. With cell-level single fact_id, you cannot declare both bindings. Hmm — could the cell fact_id be the .id fact and the numeric derive? No — numeric needs concrete_strength fact for value comparison. Two tokens, two different facts → needs two bindings. Confirmed: cell-level insufficient.

Also table title/notes: hmm — the user's decision question is specifically about TableCell.fact_id ("当前 TableCell.fact_id 的 cell 级绑定是否已经足够"). Titles/notes: where are they? If they flow through rendered_segments (as RenderedSegments with table-block location), BIND-8 covers them — but the contract doesn't explicitly say table titles/notes appear in rendered_segments. Hmm. Should I flag this too? The user's scope: "Table Numeric Binding 边界裁决" focused on cell-level vs token-level. But 表标题/表注 are part of §41.2.1's frozen carrier list. Let me check current contract state: StructuredTable = {table_id, headers, rows} — no caption/notes. rendered_segments scope note (BIND-8): "prose / 图注". So table titles/notes are in neither → a secondary gap. But wait — the user's B4-round instruction and my BIND-8 note said structured_tables 单元格 is the candidate. Titles/notes... Hmm, actually — do I know titles/notes aren't in rendered_segments? The contract says rendered_segments = "渲染后按段落/表格/图注切分" (original v10 line was replaced in R10 — current §4 doesn't enumerate; RenderedSegment.segment_location = "semantic path + block id" — a table's caption block would have a location). Honestly: the contract doesn't exclude table titles/notes from rendered_segments; segment_location can point at a table block's caption. So titles/notes CAN flow through rendered_segments and be covered by BIND-8 — provided the segment list includes them. Is that "covered"? The contract doesn't explicitly enumerate, but it doesn't exclude either. Hmm.

For the decision check, I should be precise:
1. Frozen requirements (04_REPORT_IR): §41.2.1 five carriers; §41.2.2 per-token judgment; R2 token-level explicit binding; R3 identifier same-table binding; §41.2.1's dual path for tables (Ref → §41.1; free text → §41.2); §41.2.5 per-token range anchoring.
2. What current SystemOutput can express: single-numeric-token cells (cell fact_id → per-token binding trivially); all rendered_segments tokens (prose/captions — and table titles/notes IF emitted as segments); whitelist.
3. What it cannot express: cell with ≥2 numeric tokens of different facts (each needs own fact_id); cell with identifier + numeric needing two different fact bindings (K001：32.4MPa → .id fact + concrete_strength fact); range "2.0～2.5m" in a cell with min/max from different facts.
4. Deterministic executability gap? YES — for the free-text multi-token cell scenario the frozen layer explicitly requires per-token anchoring (§41.2.1 "若为自由文本，走本节锚定校验" + §41.2.2 "逐个 token 判定" + §41.2.5), and the current contract cannot express it → P0-4 cannot prove coverage of all frozen carriers. This is a deterministic executability gap (not merely a regression candidate): the frozen layer REQUIRES the capability; the contract lacks it.
5. 既有冻结回归? — Hmm. Is it a "regression" (v9 had it, v10 lost it)? Can't prove v9 had a cell-token binding schema (v9 text not on disk). But the user's framing this round changes the classification: the gap is against 04_REPORT_IR's frozen requirements (a design-doc-level frozen source, which IS on disk and verifiable!). So this is NOT a "v9→v10 regression candidate" — it's a **deterministic executability gap against a verifiable frozen requirement** (04_REPORT_IR §41.2.1/§41.2.2/R2 are on-disk frozen documents). That's why it can't stay a CANDIDATE. The classification: 确定性可执行性缺口（对可验证冻结依据），回归属性无法确证（v9 原文缺失）但不再重要——缺口本身已确定。
6. Minimal repair boundary (next round, NOT designed now): extend the table-side binding to token granularity with minimal surface change — e.g., per-cell anchor declarations reusing the SAME AnchorDeclaration structure (cell text tokens + AnchorDeclaration list per cell or a cell-level anchor_declarations list), reusing BIND-8 span rules verbatim; or alternatively keep cell fact_id for single-token cells and add per-token declarations only for multi-token cells. But DO NOT design now — just state the boundary: (a) reuse AnchorDeclaration as-is (no new structure), (b) span rule identical to BIND-7/8, (c) error codes identical, (d) still P0-4, (e) single-token cell case must remain expressible (cell fact_id or one declaration).

Hmm wait — the user says "不要直接设计新的 TableCell schema" but asks for "下一步最小修复边界" — so I describe the BOUNDARY (what the minimal fix must and must not touch), not the design.

Now — the verdict: NOT READY. The user said: "如果证明当前 Contract 无法执行：最终必须 NOT READY。只有当 Table 的冻结要求被明确覆盖，或者正式证明现有 cell-level 表达已经足够覆盖所有冻结场景，才允许继续 READY 检查。"

My analysis: cell-level is NOT sufficient (proven by frozen text), table frozen requirements NOT covered for multi-token cells → **NOT READY**.

So I must also update the document: change item 6 in the scan table from CANDIDATE to CONFIRMED BLOCKER (B5 / CONFIRMED), and change the final verdict from READY to NOT READY. Wait — should I modify the document? The user's output requirement is the DECISION CHECK (chat output). But the document currently says READY — which is now wrong per this adjudication. The user says "本轮最终状态：… 最终必须 NOT READY"。So the document's ending must be updated to NOT READY, and the scan item 6 must be re-classified from CANDIDATE to CONFIRMED BLOCKER (deterministic executability gap). Yes — update the document accordingly (that's the honest state machine), plus add the decision record to the document (a concise section). But careful: "不要直接设计新的 TableCell schema" — I won't design; I'll only re-classify and record the decision + NOT READY.

Hmm — how much document editing? Minimal:
1. Scan table item 6: change "是否确定回归" → "是——确定性可执行性缺口（对 04_REPORT_IR §41.2.1/§41.2.2/R2 可验证冻结依据）；回归属性（v9→v10）无法确证但不再重要" and 最终结论 → "CONFIRMED BLOCKER（B5）：本轮边界裁决判定 cell 级绑定不足，需逐 token 绑定；待下轮最小修复".
2. Revision record: add R18 row? The decision is an adjudication, not a repair. Add: "> | R18 | B4 轮候选第 6 项（TABLE-NUMERIC-BINDING）边界裁决：cell 级绑定不足 → CONFIRMED BLOCKER（B5），契约回退 NOT READY | 结尾扫描第 6 项 |" — hmm, or just update the scan item + ending. A revision row keeps the record complete. Add R18 row.
3. Ending: B4 section — keep; READY 条件核对 — the "B4 numeric-token binding 完整闭环" row stays ✅ but add row "Table 冻结载体覆盖 | ✗（B5 CONFIRMED：cell 级绑定不足以覆盖 §41.2.1/§41.2.2 逐 token 判定）" and change final line to "**Step 7-E Coding Contract v10 = NOT READY**" with the transition note updated.
4. Header status: "> 状态：READY" → "> 状态：NOT READY（B5：TABLE-NUMERIC-BINDING 确定性缺口）".

Also the intro of scan section says "确认任一成立即回退 NOT READY" — item 6 is now confirmed → the document is consistent with NOT READY.

Wait — one more consideration on the analysis. Let me re-verify the Ref-path: §41.2.1 "表格数值若来自 Ref，走渲染前校验（§41.1）" — for Ref cells, Step 7-E'sTableCell {raw_text, fact_id} DOES express the binding (fact_id = Ref target), and the value check works (rendered value formatted from fact). So Ref-path single-value cells: covered. Multi-Ref cells (a cell containing two Refs)? §21: "Cell ∈ {Ref | Lit | Computed}" — a single Cell is ONE of these. Can a cell be a composite of multiple Refs? §22 example shows each cell = one Ref or one Lit. Hmm — if the IR forbids multi-Ref cells, then multi-number cells can only arise from... Lit cells (free text) or Computed. Computed cells: "avg(measurements)" → rendered value — single computed value → single number → cell fact_id?? A computed value has no fact_id (it's computed, not a Fact)! Hmm — how does a Computed cell's number get anchored? §41.2.2 requires every numeric token bound to a fact_id... a computed average isn't a Fact in G3. Hmm — this is another wrinkle: computed cells' numbers. But wait — G3 might contain computed facts (method: computed — e.g., fact:point.K001.average exists in CDM examples!). So averages CAN be Facts. OK — computed cells whose result is a registered Fact → bindable. Computed values NOT registered as facts → hmm, edge case beyond scope; whitelist/登记 handles non-fact numbers per §41.2.4? No — whitelist is for non-fact numbers like page numbers. A computed statistic not in G3... P0-2 TABLE.INTERNAL handles table internal consistency (sums/averages vs components) — that's the rule for aggregates! So aggregate cells are P0-2's domain (internal consistency), and if the aggregate is also a G3 fact → P0-1 binding works. OK, not needed for the decision — don't over-extend. Stay focused: the decision is about cell-level vs token-level for §41.2.1's free-text anchoring path.

Let me also verify: does 04_REPORT_IR §41.2.2 "渲染后的文本（Narrative prose / Table 单元格 / Figure caption）" — yes Table 单元格 explicitly in the anchoring extraction scope. And "逐个 token 判定 ├── 显式绑定 fact_id 且 value(token) == fact.value" — per token. R2 rule table: "R2 | 数值 token 必须显式绑定 fact_id，判定取「值相等」而非「值存在」". R3: "R3 | 实例标识 token（构件 / 测点 / 样品 / 文件编号）必须与数值 token 同表绑定，防止值正确而归属错配" — "同表绑定" means identifier and numeric bound in the same table (binding table), i.e., both declared.

So the frozen判定粒度 = token-level, unambiguous. Cell-level single fact_id satisfies it only for single-token (or uniform) cells. Multi-token free-text cells are explicitly contemplated by §41.2.1 ("若为自由文本，走本节锚定校验") — the Validator must judge them → per-token binding expression is REQUIRED. → Answer: **B**.

Now, the user's examples:
- Cell "2.3m" + one fact_id → expressible ✓ (single token)
- Cell "2.3m / 2.5m" two tokens possibly two facts → NOT expressible with cell-level (each needs own fact_id) → B confirmed
- Cell "K001：32.4MPa" identifier + numeric → needs two bindings (K001→…K001.id per R3; 32.4→…concrete_strength per R2) → NOT expressible → B confirmed

Unless… could one argue the identifier binding is derivable? R3 says identifier must be 同表绑定 (explicitly bound). Deriving K001→K001.id from the numeric's fact_id instance_key is an inference the Validator could make… but R2/R3 say "显式绑定" (explicit). And D-018's rationale: preventing mis-attribution requires explicit declarations. Deriving would be a Validator-side inference = designing new semantics — and honestly, for the same-cell case it would be safe-ish, but for a cell "K001：32.4MPa" the numeric's fact instance_key is K001, so K001's .id binding is derivable... Still: this is exactly the kind of "inferred binding" the frozen layer rejects (D-017's spirit: declarations must be explicit; the Validator checks the rendered artifact against explicit bindings). And regardless — the multi-numeric case (2.3/2.5) has NO derivation path. So B stands on the multi-numeric case alone; the identifier case reinforces.

So the decision: **B — cell 内逐 numeric token 独立 fact_id binding 是冻结层要求的必要表达力**。Cell-level alone is insufficient. Current contract has a deterministic executability gap → **NOT READY**.

Also — question 5 "是否属于既有冻结回归": The gap is against 04_REPORT_IR (frozen design doc, on disk, v0.3.1.1) — so it's a deterministic conflict with a frozen artifact. Whether v9's contract had cell-token binding — unknowable. Classification: 确定性可执行性缺口（依据在盘冻结文档），v9→v10 回归属性无法确证（候选属性被升级为 CONFIRMED，因为依据可验证且当前契约明确缺表达力）。Actually more precisely: it's a confirmed CONTRACT-vs-FROZEN-LAYER gap; the "regression" label (did v10 lose something v9 had) is moot — the contract as it stands cannot execute the frozen requirement.

Question 6 "下一步最小修复边界" — boundary only:
- Must: extend StructuredTable side to token-granular binding; REUSE AnchorDeclaration structure as-is; span rule identical to BIND-7/8; error codes identical (UNANCHORED / INVALID_FACT_ID / DUPLICATE_BINDING); single-token cell must remain expressible (no forced migration); still P0-4; Fix 3 unchanged (Validator tokenizes cell raw_text itself).
- Must not: new P0 rule; redesign of RenderedSegment-side BIND-8 semantics; touching 04_REPORT_IR / Step 7-D; cell fact_id semantics for the single-token case (keep backward expression); per-cell vs per-table declaration placement choice → that IS the design decision for next round (present options? No — just name the decision point).

Hmm — the user says "6. 下一步最小修复边界" — so I list the boundary constraints. Also I could name the two candidate shapes (per-cell declarations list vs cell-level + optional per-token overrides) as decision points WITHOUT designing. Keep it as boundary description.

Document updates:
1. Revision record R18 row.
2. Scan table item 6: re-classify → CONFIRMED BLOCKER (B5).
3. Scan intro: adjust "第 6 项已由本轮边界裁决升级为 CONFIRMED BLOCKER" 
4. Ending: add "边界裁决（B5，本轮）" subsection: decision A/B → B; NOT READY.
5. READY 条件核对: add row "Table 冻结载体覆盖（§41.2.1） | ✗ CONFIRMED BLOCKER B5" and change the 无未处理确定性回归 row.
6. Header status → NOT READY; final line → NOT READY.

Also — BIND-8's scope note in §8.1 (L537) says "structured_tables 单元格 numeric token 的逐 token 绑定机制为已登记候选（结尾扫描第 6 项），本轮不设计" — update to reflect the adjudication: "…为 CONFIRMED BLOCKER（B5，见结尾扫描第 6 项）——cell 级绑定经裁决不足以覆盖 §41.2.1/§41.2.2 逐 token 判定，待下轮最小修复". Hmm — the user said don't modify BIND-8's numeric core semantics; updating the scope note's classification is not touching semantics. OK.

Let me now also double-check: is there any reading under which cell-level IS enough? Devil's advocate: §41.2.1's Ref-path — "表格数值若来自 Ref，走渲染前校验（§41.1）" — for Ref cells, NO per-token anchoring needed (the §41.1 path). If the frozen layer's intent were "table cells must be Refs; free-text cells are tolerated but must anchor per-token", then a System that only emits single-number Ref/Lit cells is compliant, and multi-number free-text cells… §41.2.1 still requires the Validator to handle them ("两者都不允许出现无来源数字"). The Validator's job is to judge WHATEVER rendered artifact arrives — §41.2.2 R5. A contract that cannot express judgment for a case the frozen layer explicitly contemplates = executability gap. Even if multi-number cells are "discouraged", §41.2.5's range-in-cell case ("2.0～2.5m" in a cell) is ordinary engineering practice (min～max columns!) — and §41.2.5 explicitly requires per-token independent binding with possibly different facts. Range columns in real inspection tables are extremely common ("0.12～0.18" min-max). So B is solid.

Also note: "2.0～2.5m" — the ～ separates; per §41.2.5 each number independently bound — often min and max are DIFFERENT facts (or the same measurement set's min/max — which might be computed facts in G3 or not). Either way: two tokens, potentially two fact_ids → cell-level single fact_id cannot express. Confirmed.

One more check — the user's criterion: "只有当 Table 的冻结要求被明确覆盖，或者正式证明现有 cell-level 表达已经足够覆盖所有冻结场景，才允许继续 READY 检查。" — I've proven cell-level insufficient → NOT READY. 

Now the chat output: TABLE NUMERIC BINDING DECISION CHECK with the 6 required sections + final NOT READY. And document edits.

Let me now do the document edits:

**Edit 1 — revision record**: add after R17:
`> | R18 | B5 边界裁决：TABLE-NUMERIC-BINDING 升级 CONFIRMED BLOCKER（cell 级绑定不足以覆盖 §41.2.1/§41.2.2 逐 token 判定）→ 契约回退 NOT READY | 结尾扫描第 6 项 / 结尾判定 |`

**Edit 2 — header status**: "> 状态：READY" → "> 状态：NOT READY（B5：TABLE-NUMERIC-BINDING 确定性可执行性缺口，待最小修复）"

**Edit 3 — BIND-8 scope note (L537)**: 
old: "- BIND-8 作用域 = Validator 对 rendered_segments（prose / 图注）tokenize 出的 numeric token；structured_tables 单元格 numeric token 的逐 token 绑定机制为已登记候选（结尾扫描第 6 项），本轮不设计"
new: "- BIND-8 作用域 = Validator 对 rendered_segments（prose / 图注）tokenize 出的 numeric token；structured_tables 单元格侧的逐 token 绑定经 B5 边界裁决确认为 **CONFIRMED BLOCKER**（cell 级 fact_id 不足以覆盖 §41.2.1/§41.2.2 逐 token 判定，见结尾扫描第 6 项），待下轮最小修复"

**Edit 4 — scan table item 6**:
old row 6 → new:
`| 6 | TABLE-NUMERIC-BINDING（TableCell 级 numeric token 逐 token 绑定） | **是——冻结依据在盘可验证**：04_REPORT_IR §41.2.1（Table 单元格 / 表标题 / 表注纳入锚定；自由文本 cell 走 §41.2 锚定）+ §41.2.2（逐 token 判定）+ R2（token 级显式绑定）+ §41.2.5（区间逐 token 独立绑定） | TableCell.fact_id 为 cell 级单值：单 numeric token cell 可表达；多 numeric token cell（"2.3m / 2.5m"、区间 "2.0～2.5m"）、identifier+numeric cell（"K001：32.4MPa" 需 .id 与 concrete_strength 两个绑定）**均无法表达** | **是——确定性可执行性缺口（B5 边界裁决，见结尾裁决记录）；v9→v10 回归属性无法确证但不再重要** | 否（本轮为裁决轮，不设计 TableCell schema） | **CONFIRMED BLOCKER B5**：需 cell 内逐 numeric token 独立 fact_id 绑定；最小修复边界见裁决记录；修复前契约 NOT READY |`

**Edit 5 — scan intro**: "旧契约遗漏扫描（5 项）+ B4 轮新增候选（第 6 项）：…" → update to note item 6 upgraded:
"旧契约遗漏扫描（5 项）+ 第 6 项（B4 轮新增、B5 轮裁决升级为 CONFIRMED BLOCKER）：第 1–5 项无法从磁盘证据确证为确定性回归，标记 CANDIDATE、不直接修改；第 6 项冻结依据在盘可验证（04_REPORT_IR §41.2.1/§41.2.2/R2），经 B5 边界裁决确认为确定性可执行性缺口："

**Edit 6 — ending**: after B4 section add B5 裁决 section:
```
**边界裁决（B5，本轮）**：

| 项 | 裁决 | 依据 | 状态 |
| ---- | ---- | ---- | ---- |
| TABLE-NUMERIC-BINDING | 裁决为 **B：必须支持 cell 内逐 numeric token 独立 fact_id 绑定**；cell 级单 fact_id 仅覆盖单 token cell，不满足 §41.2.1 / §41.2.2 / R2 / §41.2.5 | 04_REPORT_IR §41.2.1（自由文本 cell 走锚定）+ §41.2.2（逐 token 判定）+ R2 + §41.2.5（区间逐 token）——冻结依据在盘可验证 | **CONFIRMED BLOCKER** → 契约回退 NOT READY；最小修复边界见结尾「B5 修复边界」 |
```
And add "**B5 修复边界（下一轮执行，本轮不设计）**：" block:
```
- 必须支持：cell 内逐 numeric token 独立 fact_id 绑定（多 token / 区间 / identifier+numeric cell 场景）
- 必须复用：AnchorDeclaration 结构原样（token / fact_id / char_start / char_end）；BIND-7/8 同一 span 规则；现有 ANCHOR.* 错误码；仍属 P0-4；Fix 3（Validator 对 cell raw_text 自行 tokenize）
- 必须保持：单 numeric token cell 的现有表达（cell fact_id 路径不强制迁移）；BIND-8 rendered_segments 侧语义不动
- 禁止：新增 P0 Rule；修改 04_REPORT_IR / Step 7-D；重新设计 RenderedSegment 侧；引入 LLM
- 待决策点（下一轮由用户裁决）：绑定声明挂载位置（per-cell 列表 vs 表级声明集合）——属最小修复的唯一设计自由度
```
Hmm — is naming the decision point "designing"? It's naming what must be decided, not deciding. OK.

**Edit 7 — READY 条件核对**: modify:
- Add row: "| Table 冻结载体覆盖（§41.2.1 / §41.2.2 逐 token 判定） | ✗ **CONFIRMED BLOCKER B5**（cell 级绑定不足，待最小修复） |"
- Update "无未处理的确定性回归" row: "✗（第 6 项经 B5 裁决确认为确定性可执行性缺口；第 1–5 项仍为 CANDIDATE）"
- Transition note: "B5 边界裁决确认 TABLE-NUMERIC-BINDING 为确定性缺口 → 契约回退 NOT READY；待下一轮按「B5 修复边界」完成最小修复后重新判定。"
- Final line: "**Step 7-E Coding Contract v10 = NOT READY**"

Also the 内部交叉检查 table has "P0-4 numeric binding 闭环 | ✅" — should add nuance? The rendered_segments side is closed; table side is B5. The cross-check rows: "P0-4 numeric binding 闭环 | ✅ | §8.1 BIND-8 + §8.2 判定链（B4 本轮补齐）" — add "（rendered_segments 侧；Table 侧见 B5）". Minimal edit.

Also header intro lines (L7-8): "本轮为回归审计（REGRESSION FOUND）后的恢复轮：…" — leave as is; the status line change suffices. Hmm, the header says "> 状态：READY" — change to NOT READY with reason.

Let me execute the edits now. Then the chat output: TABLE NUMERIC BINDING DECISION CHECK with 6 sections + NOT READY verdict.

For the chat output structure:

# TABLE NUMERIC BINDING DECISION CHECK

## 1. 04_REPORT_IR 当前冻结要求
- §41.2.1 五类载体 + 表格双路径（Ref→§41.1；自由文本→§41.2）
- §41.2.2 逐 token 判定（Table 单元格明确在提取范围）
- R2 token 级显式绑定；R3 identifier 同表绑定
- §41.2.5 区间逐 token 独立绑定

## 2. 当前 Step 7-E SystemOutput 能表达什么
- 单 numeric token cell（cell fact_id 即该 token 的绑定）
- rendered_segments 全部 token（BIND-8）——含 prose/图注；表标题/表注若作为 segment 输出亦可覆盖
- whitelist 豁免

## 3. 当前 Step 7-E 不能表达什么
- 多 numeric token cell（2.3m / 2.5m 各属不同 fact）
- 区间 cell（2.0～2.5m，min/max 可能各属不同 fact）
- identifier + numeric cell（K001：32.4MPa 需 K001.id + concrete_strength 两个绑定，R3+R2）

## 4. 是否构成确定性可执行性缺口
**是**。判定依据：冻结判定粒度是 token 级（§41.2.2 "逐个 token 判定" + R2 + §41.2.5），冻结层明确预期自由文本 cell 进入锚定路径（§41.2.1），且多 token cell 是工程常规（min～max 列）；R5 要求校验渲染产物——Validator 必须能判定这类 cell，而当前契约无表达手段 → P0-4 无法证明覆盖全部冻结载体。

## 5. 是否属于既有冻结回归
升级说明：不再以"v9→v10 回归"定性（v9 原文缺失无法确证）；定性为**对在盘冻结文档（04_REPORT_IR v0.3.1.1）的确定性可执行性缺口**——依据可验证，缺口确定成立。CANDIDATE 标记撤销，升级 CONFIRMED BLOCKER（B5）。

## 6. 下一步最小修复边界
- 必须支持 / 必须复用 / 必须保持 / 禁止 / 待决策点（如上）

## 裁决结论
A/B 判定：**B**
最终状态：**NOT READY**（文档已同步回退）

Let me execute. First the document edits (6-7 edits), then verify, then output.

Edit for header status:
old:
```
> 版本：v10（回归修复版）
> 状态：READY
```
new:
```
> 版本：v10（回归修复版）
> 状态：NOT READY（B5：TABLE-NUMERIC-BINDING 确定性可执行性缺口，待最小修复）
```

Hmm wait — the header also has line 7-8 describing the round. Fine.

One more consideration — the user's instruction "不要修改 04_REPORT_IR" ✓, "不要修改 Step 7-D" ✓. My document edits are confined to the contract file. Good.

Also the earlier established pattern: the ending transition note + final line. Update them.

Execute now.

Now update the ending: add B5 裁决 section + B5 修复边界 after the B4 section; update the cross-check table's P0-4 numeric binding row; update READY 条件核对; update transition note + final line to NOT READY.

Document updated. Now output the TABLE NUMERIC BINDING DECISION CHECK with the 6 required sections and the NOT READY verdict.
文档已同步回退。以下是正式裁决。

---

# TABLE NUMERIC BINDING DECISION CHECK

## 1. 04_REPORT_IR 当前冻结要求（在盘可验证：v0.3.1.1）

| 条款 | 冻结内容 |
| ---- | ---- |
| §41.2.1 | 锚定覆盖三类载体：**Narrative prose / Table 单元格文本与表标题表注 / Figure caption**；且表格数值**双路径**——「若来自 Ref，走渲染前校验（§41.1）；**若为自由文本，走本节锚定校验**。两者都不允许出现无来源数字」 |
| §41.2.2 | 渲染后文本（**Table 单元格**在提取范围）→「提取**全部**数值 token → **逐个 token** 判定：显式绑定 fact_id 且 value(token) == fact.value」 |
| R2 | 数值 token 必须显式绑定 fact_id，判定取「值相等」——判定粒度 = **token 级** |
| R3 | 实例标识 token 必须与数值 token 同表绑定（各自独立声明） |
| §41.2.5 | 区间表达「按 token 逐个锚定：区间内每个数字都需**独立绑定**并值相等，不允许用区间整体作为一次锚定」 |
| R5 | 校验对象是**渲染产物**，不是 IR 中的声明——Validator 必须能判定实际到达的任何 cell |

**结论：冻结判定粒度 = cell 内逐 numeric token，且冻结层明确预期自由文本多 token cell 进入锚定路径。**

## 2. 当前 Step 7-E SystemOutput 能表达什么

- **单 numeric token cell**：`TableCell.fact_id` 即该 token 的绑定 → BIND-8 语义可平移，链路完整 ✅
- **rendered_segments 全部 token**（prose / 图注，含表标题表注若作为 segment 输出）：BIND-8 已闭环 ✅
- whitelist 豁免、区间逐 token（rendered_segments 侧）✅

## 3. 当前 Step 7-E 不能表达什么

| 场景 | 所需绑定数 | cell 级单 fact_id |
| ---- | ---- | ---- |
| `"2.3m"`（单 token） | 1 | ✅ 可表达 |
| `"2.3m / 2.5m"`（两个 token 各属不同 fact，如两条缺陷长度） | 2 个不同 fact_id | ❌ 只有一个 fact_id |
| `"2.0～2.5m"`（区间 cell，min/max 可各属不同 fact） | 2 个独立绑定（§41.2.5 强制） | ❌ 同上 |
| `"K001：32.4MPa"`（identifier + numeric） | 2 个绑定（R3：K001→…K001.id；R2：32.4→…concrete_strength） | ❌ 一个 fact_id 无法同时表达两个声明 |

min～max 统计列是工程检测表格的**常规形态**，不是边角案例。

## 4. 是否构成当前 Contract 的确定性可执行性缺口

**是，确定性缺口。** 推理链：
1. 冻结判定粒度是 token 级（§41.2.2「逐个 token 判定」+ R2 + §41.2.5）；
2. 冻结层明确预期自由文本 cell 进入锚定路径（§41.2.1），且 R5 要求校验渲染产物——Validator 无法选择「只接收单 token cell」；
3. 当前契约对多 token cell / identifier+numeric cell **无绑定表达手段** → P0-4 无法证明覆盖 §41.2.1 全部冻结载体；
4. 因此 P0-4 在 Table 载体上不可执行——这不是「待确认的候选」，而是**对在盘冻结文档的直接冲突**。

## 5. 是否属于既有冻结回归

分类升级：撤销「REGRESSION CANDIDATE」，升级为 **CONFIRMED BLOCKER（B5）**。
- v9→v10 回归属性：仍无法确证（v9 原文缺失）——但**不再重要**；
- 决定性依据不在 v9 契约，而在**在盘冻结文档 04_REPORT_IR v0.3.1.1**（§41.2.1 / §41.2.2 / R2 / R3 / §41.2.5 / R5 全部可验证）——契约与之冲突，缺口确定成立。

## 6. 下一步最小修复边界（已登记入契约「B5 修复边界」，本轮不设计）

- **必须支持**：cell 内逐 numeric token 独立 fact_id 绑定（多 token cell / 区间 cell / identifier+numeric cell 三类场景）
- **必须复用**：AnchorDeclaration 结构原样；BIND-7/8 同一 span 规则；现有 ANCHOR.* 错误码；仍属 P0-4；Fix 3（Validator 对 cell raw_text 自行 tokenize）
- **必须保持**：单 token cell 的 cell fact_id 表达路径不强制迁移；BIND-8 rendered_segments 侧与 §8.2 判定链零改动
- **禁止**：新增 P0 Rule；修改 04_REPORT_IR / Step 7-D；重新设计 RenderedSegment 侧；引入 LLM
- **待你决策的唯一设计自由度**：绑定声明在 Table 侧的挂载位置（per-cell 声明列表 vs 表级声明集合）

---

# 裁决结论

**A/B 判定：B** —— cell 级绑定**不足**，必须支持 cell 内逐 numeric token 独立 fact_id 绑定。

**最终状态：NOT READY**

文档已同步：扫描第 6 项升级 CONFIRMED BLOCKER（[扫描表](file:///g:/workspace/zixun4/docs/Step_7_E_Coding_Contract_v10.md#L980)）、B5 裁决与修复边界入档（[结尾](file:///g:/workspace/zixun4/docs/Step_7_E_Coding_Contract_v10.md#L965-L977)）、状态行与结尾判定回退 **Step 7-E Coding Contract v10 = NOT READY**。

下一轮你确认「B5 修复边界」及挂载位置决策点后，即可执行最小修复并重新判定。
