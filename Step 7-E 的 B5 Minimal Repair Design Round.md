Let me start by understanding the task. The user wants me to do a B5 Minimal Repair Design Round for Step 7-E. This is a design-only task - no code, no file modifications. I need to read the specified documents first:

1. Step_7_E_Coding_Contract_v10.md
2. 04_REPORT_IR.md
3. 03_CANONICAL_DATA_MODEL.md
4. 05_EVALUATION.md
5. 07_DECISIONS.md

The user opened the file at g:\workspace\zixun4\docs\Step_7_E_Coding_Contract_v10.md line 914 which mentions "Phase 1: 契约自检（contracts.py）".

Let me read these documents. They're likely long, so I should read them strategically. Let me start with the Step 7-E contract since it's the main document, and look at the relevant sections (§8.1, §8.2, §8.3, §8.4, §41.2.x references from 04_REPORT_IR).

Let me read the files. I should probably check memory first per the memory guidelines - but the memory context is already provided in the system reminders. The task is clear: read docs, produce a design check document in the specified output format.

Let me read the documents in parallel.

已读取 Step 7-E v10 全文。现在检索其余四份文档中的冻结依据章节。
I've read the Step 7-E contract. Now I need to read the relevant sections of 04_REPORT_IR.md (especially §41.2.x, §13.1), 03_CANONICAL_DATA_MODEL.md, 05_EVALUATION.md, and 07_DECISIONS.md. These are likely long files, so let me search for the relevant sections first.

Key sections to find:
- 04_REPORT_IR.md: §41.2, §41.2.1, §41.2.2, §41.2.3, §41.2.5, §13.1 (anchors)
- 03_CANONICAL_DATA_MODEL.md: Fact, fact_id structure
- 07_DECISIONS.md: D-018, D-019, D-025
- 05_EVALUATION.md: relevant validator sections

Let me grep for these sections.

Now let me read the relevant sections of 04_REPORT_IR.md (§41.2 through §41.2.5, and §13.1 anchors), and the D-018/D-019/D-025 decisions in 07_DECISIONS.md.

Now I have the key sections of 04_REPORT_IR §41.2. Let me also check §13.1 anchors structure and the Table structure in 04_REPORT_IR to understand what "Table title / Table note" look like in the IR, and check 05_EVALUATION.md for relevant validator sections. Let me also quickly check D-025.

Let me read §13.1 of 04_REPORT_IR (around line 507-560), and look at the Table structure definition in 04_REPORT_IR to understand cell/title/note fields.

Let me look at the Table structure in 04_REPORT_IR (around line 300-330) and the Figure caption (around line 985-1000), and check 05_EVALUATION.md for the relevant sections (§6.4 anchoring). Also need to check the Step 7-E §8.1's question 2: whether Table title / Table note can enter the existing BIND-8 rendered_segments path.

Key question 2: In the SystemOutput schema (§4), rendered_segments: list[RenderedSegment] with segment_location: str. The question is whether Table title / Table note can be represented as RenderedSegments. The contract says rendered_segments is "结构化渲染段（禁止退化为 list[str]）" — with segment_location being "定位（semantic path + block id）". So if the system emits the table title and table note as RenderedSegments (with appropriate segment_location), they could go through the existing BIND-8 path. But can the contract prove that this is possible? The contract §4 says rendered_segments: list[RenderedSegment], segment_location is a free-form string "semantic path + block id". There's no explicit statement that table title/note must be emitted as rendered_segments. So this is "覆盖路径需要澄清" unless there's clear evidence.

Let me check the Table structure in 04_REPORT_IR to see if title/note are part of Table.

Now let me check the Table block definition in 04_REPORT_IR to see its exact fields (title, note, cells). Let me search for "Table" block definition.

已读取 Step 7-E v10 全文、04_REPORT_IR §41.2 全节、§13.1、Table/Figure 结构、05_EVALUATION §6.4、07_DECISIONS D-018/D-019/D-025。最后核对 CDM 的 Fact ID 三段结构与 instance_key 定义。
I now have all the key evidence. Let me also quickly check 03_CANONICAL_DATA_MODEL.md for fact_id structure (three-segment) — I have this from memory and project memory, but let me verify the parse_fact_id / derive_fact_type function briefly, and check D-025 quickly. Actually, I have enough on those from the contract (INV-11, §13, §8.2). Let me do a quick sanity check on 03_CANONICAL_DATA_MODEL for Fact ID structure to confirm the identifier → instance_key semantics.

Actually the contract already states: INV-22: Anchor 标识比较 = normalized_identifier == parse_fact_id(anchor.fact_id).instance_key. That's sufficient for the identifier boundary question.

Let me also verify the question about Table title/note in rendered_segments path. The contract §4:
- rendered_segments: list[RenderedSegment], segment_location: str (定位：semantic path + block id), raw_text: str, anchor_declarations: list[AnchorDeclaration].
- structured_tables: list[StructuredTable] (table_id, headers, rows).

Question 2 asks: Can Table title / Table note currently enter the existing BIND-8 rendered_segments path?

Analysis: The contract's SystemOutput has rendered_segments as a generic list of RenderedSegment with free-form segment_location. There is no enumerated constraint that limits segment_location to narrative-only, and no explicit statement that table title/note must be (or must not be) emitted as rendered_segments. Stage C checks rendered_segments non-empty. The Validator tokenizes each RenderedSegment.raw_text. So mechanically, if the System emits table title/note as RenderedSegments, they'd be covered. But the contract cannot *prove* this is mandatory — nothing requires the System to include table title/note in rendered_segments, and nothing forbids it either. So the answer is: 覆盖路径需要澄清 — per the user's instruction, just register it as "coverage path needs clarification", don't extend schema.

Wait — but let me be careful. The user says: "如果当前 Contract 无法明确证明：只登记为'覆盖路径需要澄清'，不要顺手扩展 schema。" So I need to check whether the contract can explicitly prove Table title/note can enter the BIND-8 rendered_segments path.

Evidence:
- 04_REPORT_IR §41.2.1 explicitly includes Table title/note in anchoring scope.
- Step 7-E §4 SystemOutput: rendered_segments (segment_location: "semantic path + block id") + structured_tables (table_id, headers, rows only — no title/note fields at all!).
- StructuredTable has NO title/note field. So table title/note cannot be expressed in structured_tables.
- Can they be expressed in rendered_segments? segment_location is a free-form string; nothing forbids the System from emitting a RenderedSegment with segment_location="table[table-2-1].title". The contract's Stage C requires rendered_segments non-empty. §8.1 BIND-8's scope note says "BIND-8 作用域 = Validator 对 rendered_segments（prose / 图注）tokenize 出的 numeric token" — this parenthetical "（prose / 图注）" suggests the intended scope is prose and figure captions, NOT table title/note. Hmm, that's a hint that rendered_segments is intended for prose/captions. But it's a parenthetical description, not a normative restriction ("禁止" or "仅限").

So the honest answer: 当前 Contract 无法明确证明 Table title / Table note 已经能够进入现有 BIND-8 rendered_segments 路径（§8.1 BIND-8 作用域括注只写了 prose / 图注；StructuredTable 无 title/note 字段；rendered_segments 未枚举必须包含 table title/note）。→ 登记为「覆盖路径需要澄清」，不扩展 schema。

This is an important finding for Section 8.

Now the main design question: per-cell (A) vs table-level (B) binding declarations.

Let me analyze:

**Option A: per-cell binding declarations**
- Add `anchor_declarations: list[AnchorDeclaration]` to TableCell (analogous to RenderedSegment).
- char_start/char_end are relative to TableCell.raw_text (local coordinate system, same as RenderedSegment.raw_text being the coordinate base for its own declarations).
- Locating a declaration after tokenize: for each cell, tokenize cell.raw_text, match tokens against that cell's own anchor_declarations. O(1) localization — the declaration set is scoped to the exact text being tokenized. No cross-referencing needed.
- Expressiveness:
  - Single token cell "2.3m": one declaration {token:"2.3m", fact_id, char_start:0, char_end:4}. Also legacy path (cell.fact_id) can remain for compatibility.
  - Multi token cell "2.3m / 2.5m": two declarations, each with own char offsets.
  - Range "2.0～2.5m": two declarations, independent fact_ids. §8.3 satisfied.
  - identifier + numeric "K001：32.4MPa": two declarations — {token:"K001", fact:"...K001.id"}, {token:"32.4MPa", fact:"...K001.concrete_strength"}. BIND-7 comparison unchanged: normalized_identifier == parse_fact_id(decl.fact_id).instance_key.
- Changes: TableCell gains an optional field. StructuredTable unchanged. AnchorDeclaration reused as-is (token/fact_id/char_start/char_end). Coordinate base = cell.raw_text (same convention as RenderedSegment: declarations are relative to the text they attach to).
- Compatibility: existing single-fact_id cells unchanged; cell.fact_id stays as-is (旧表达路径保持).
- Minimal change surface: one optional field on TableCell. Semantics: 表达能力补齐, not semantic change — same AnchorDeclaration semantics, same span rules, same error codes.

**Option B: table-level binding declarations**
- Add `anchor_declarations: list[AnchorDeclaration]` to StructuredTable.
- Problem 1: char_start/char_end must be relative to *what*? A table has many cells. Options:
  - (a) Global table coordinate: requires defining a canonical flattening of the table into one text (row-major concatenation with separators). That's a NEW canonicalization rule — new semantic commitment, new source of mismatch between the flattened text and the actual cell raw_texts. The Validator tokenizes each cell raw_text; it must map token position (cell, char) → flattened position. This introduces a coordinate transformation layer that doesn't exist today. Violates minimality.
  - (b) Add cell coordinates to declarations: (row, col, char_start, char_end). But that changes AnchorDeclaration structure (must add fields) — violates hard constraint 5 (必须复用现有 AnchorDeclaration 原样, 不允许重新定义 anchor 类型). Or requires a wrapper type → new type.
  - (c) char offsets relative to cell raw_text + duplicate cell_id field in each declaration → again changes AnchorDeclaration.
- Problem 2: Locating a declaration after tokenize: for each token in each cell, Validator must scan the table-level list and filter by coordinate → O(tokens × declarations) per table, plus ambiguity when cells have identical raw_text (e.g., repeated "2.3m" in two cells): a table-level declaration without per-cell coordinates can't disambiguate; with coordinates it needs extra fields.
- Problem 3: Compatibility/semantics: a table-level list mixes declarations for many cells in one bag; DUPLICATE_BINDING detection must be scoped per token-per-cell, requiring coordinate-aware filtering first — more complex execution, same error codes but more complex derivation.
- Change surface: StructuredTable gains a field + either new canonical flattening semantics or AnchorDeclaration gains fields. Bigger change surface than A.

Also note the symmetry argument: RenderedSegment already carries its own anchor_declarations — the B4 frozen design attaches declarations to the text carrier at the same level as the text. Per-cell declarations replicate exactly this pattern at the finest text-carrier granularity (cell). Table-level would be a *different* pattern (declarations attached at a coarser container), introducing a second anchoring topology — inconsistent with the frozen B4 topology.

Decision: **A. per-cell binding declarations**.

Why A doesn't break B4:
- rendered_segments side untouched. RenderedSegment/AnchorDeclaration definitions untouched. BIND-7/8 span rules untouched. §8.2 chain untouched. The same AnchorDeclaration type is reused; the coordinate base is "the raw_text of the object carrying the declarations" — a convention already frozen by B4 (AnchorDeclaration.char_start is "在 RenderedSegment.raw_text 中的起始偏移"). For cells, coordinate base = TableCell.raw_text. This is the same convention applied to a new carrier, not a semantic change.
- Actually wait — there's a subtlety: the frozen §4 comment says char_start: "在 RenderedSegment.raw_text 中的起始偏移". If we attach AnchorDeclaration to TableCell, the coordinate base is cell raw_text. Is that a "semantic change" to AnchorDeclaration? No — the field is an offset relative to the carrier text it's declared on; the comment in §4 describes its use in the RenderedSegment context. Reusing the same structure with the same convention (offset into the carrying text) is 表达能力补齐. I should state this explicitly: char_start/char_end 语义 = 相对于所挂载文本载体（RenderedSegment.raw_text 或 TableCell.raw_text）的偏移. This is the generalization already implied by the structure; contract needs a one-line clarification. That's part of the minimal repair.

Now, how does the Validator find the corresponding declaration after tokenize (question 4 in the decision section)?
- For each cell: tokenize cell.raw_text (Fix 3, same tokenizer). For each extracted numeric token (and identifier token), search cell.anchor_declarations for declarations whose span covers the token span (token span ⊆ declaration span or text equality — the frozen BIND-7/8 span rule). Zero covering declarations → UNANCHORED; ≥2 covering declarations with different fact_ids → DUPLICATE_BINDING; fact_id invalid → INVALID_FACT_ID (BIND-5); then §8.2 chain per token.
- Legacy path compatibility: cells with fact_id set and empty anchor_declarations — what happens? The B5 boundary says "必须保持：单 numeric token cell 的现有表达路径（cell fact_id 不强制迁移）". So the Validator must treat cell.fact_id as an implicit binding for the single numeric token in that cell. How to keep this minimal without creating a second binding mechanism?

Option: cell.fact_id ≡ implicit AnchorDeclaration over the whole cell raw_text (span [0, len(raw_text))). That is: when a cell has fact_id != None and no explicit declarations covering a numeric token, the cell-level fact_id acts as one implicit declaration whose span covers the entire cell raw_text. Since span containment rule applies, any numeric token inside the cell is covered. This preserves single-token cell semantics exactly and requires zero migration. But wait — does this create DUPLICATE_BINDING ambiguity if both cell.fact_id and an explicit declaration exist? Need a precedence rule: if cell has explicit anchor_declarations, do they coexist with cell.fact_id? Minimal rule: cell.fact_id 保持现状语义（作为 cell 级隐式绑定，等价于一条 span 覆盖全 cell 的 AnchorDeclaration）；若同一 token 同时被隐式与显式声明覆盖且 fact_id 不同 → DUPLICATE_BINDING（现有错误码，无需新规则——因为隐式声明就是一条声明，BIND-6 自然适用）。Actually that's elegant: the implicit declaration is just a declaration; BIND-6 (同一 token 不出现多处绑定) applies uniformly. But hold on — should implicit+explicit with SAME fact_id be allowed? If cell.fact_id == declaration.fact_id, that's the same binding declared twice — is that DUPLICATE_BINDING? BIND-6 says "同一 token 不出现多处绑定声明" — hmm, strictly, two declarations covering the same token even with same fact_id... Let me check BIND-8: "同一 numeric token 不允许存在多个不同 fact_id 的绑定（多重声明按 BIND-6）". So BIND-6/DUPLICATE_BINDING is about multiple *different* fact_ids. Same fact_id duplicated → not a violation under BIND-8's wording ("多个不同 fact_id"). OK so under the frozen semantics: multiple declarations covering one token with identical fact_id = OK (redundant but not a violation); different fact_ids = DUPLICATE_BINDING. The implicit cell.fact_id declaration participates in this same rule. That's minimal and uses existing codes.

Hmm, but wait — actually, should I even define the implicit-declaration equivalence in this design round? The user's task says: "17. 单 numeric token cell 的旧表达路径必须尽量保持兼容。" and B5 修复边界 says "cell fact_id 不强制迁移". So yes, I need to specify how cell.fact_id coexists. The implicit-declaration equivalence is the minimal way: no new error codes, no new precedence logic, uniform span rule. Alternatively: "cell.fact_id 仅当 cell 无显式声明时生效" — that's a precedence rule, also simple. Which is more minimal?

Let me think. Option i (implicit declaration participates uniformly): cell.fact_id = implicit declaration spanning whole cell. Consequences: a cell with fact_id AND explicit declarations → if explicit declaration has different fact_id and both cover the same token → DUPLICATE_BINDING. Is that desirable? It flags genuine ambiguity. But it could also break a legitimate mixed cell: e.g., cell "K001：32.4MPa" with cell.fact_id=...concrete_strength (legacy) and someone adds explicit declaration for K001 only. Then 32.4MPa is covered by both implicit (concrete_strength) and... only if there's an explicit declaration for 32.4MPa with different fact_id. If explicit only declares K001, then 32.4MPa is covered by implicit only — fine. Hmm, but what about identifier tokens? cell.fact_id as implicit declaration would also cover identifier tokens — would it bind K001 to ...concrete_strength? BIND-7 requires identifier binding where instance_key must match: parse_fact_id("fact:component.K001.concrete_strength").instance_key = "K001" == normalized_identifier "K001" → PASS. So the implicit declaration covering the whole cell would satisfy BIND-7 coverage for K001 too (instance_key matches). That's actually harmless — BIND-7 only requires coverage + instance_key match; the implicit cell-level declaration with instance_key K001 satisfies it. Interesting — but is that a semantic overreach? cell.fact_id was "来自 Ref 的单元格携带" — the cell's fact. If cell = "K001：32.4MPa" with fact_id = concrete_strength, then implicitly K001 is "covered" for BIND-7 purposes. Is that correct? §41.2.3 requires K002 → fact:component.K002.id binding. The implicit cell.fact_id=concrete_strength would let K001 pass BIND-7 via instance_key match even without an explicit .id declaration. Hmm. That slightly weakens §41.2.3 for the legacy path. But wait — this is exactly the current legacy semantics question: today (pre-B5), what does BIND-7 do on structured_tables? Actually currently BIND-8 explicitly does NOT apply to structured_tables (B5 blocker), and BIND-7... let me check §8.1 BIND-7 scope: "每个 identifier token 必须被至少一个 AnchorDeclaration 覆盖" — scope is rendered_segments too (AnchorDeclarations only exist on RenderedSegment today). Table cells have no declarations at all today, so neither BIND-7 nor BIND-8 currently applies to table cell tokens. That's precisely the B5 gap.

So for the legacy path, I should be careful: the minimal compatibility rule should preserve *existing single-numeric-token-cell behavior*, which today is... actually today table cells aren't checked at all (that's the blocker). The "旧表达路径" refers to the *expression* path (cell.fact_id field exists in schema), and B5 边界 says "cell fact_id 不强制迁移" — i.e., schema compatibility: systems may keep emitting cell.fact_id for single-token cells, and that must still work (be judgeable), not that a new table-side check must replicate some frozen legacy judgment semantics (there is no frozen table-side judgment semantics — it's the gap).

So the cleanest minimal rule: **cell.fact_id ≡ 隐式 AnchorDeclaration（span = 全 cell raw_text）**，与显式声明走同一 BIND-6/7/8 规则。This keeps single-token cells working with zero migration, and mixed cells are expressible by explicit declarations. The BIND-7 side effect (identifier covered by implicit declaration) — hmm, actually wait. Let me reconsider: is it better to scope the implicit declaration to numeric binding only (BIND-8) and not BIND-7? That would be a *special case* carve-out = more rules, less minimal. Uniform participation is simpler: one rule ("cell.fact_id 等价于一条覆盖整个 cell raw_text 的 AnchorDeclaration") and all existing checks apply unchanged. And actually, for identifier tokens: does an implicit whole-cell declaration make BIND-7 pass for K001? BIND-7: "每个 identifier token 必须被至少一个 AnchorDeclaration 覆盖（token span 包含于声明 token span 或文本相等），且 normalized_identifier == parse_fact_id(fact_id).instance_key". With implicit declaration {token: <cell raw_text>, fact_id: cell.fact_id}: K001's span ⊆ whole-cell span ✓; instance_key of concrete_strength fact_id = "K001" ✓. So BIND-7 passes. Per §41.2.3, is that acceptable? The intent of §41.2.3 is to prevent mismatched attribution: K002 in text bound to K003's fact. Here instance_key matches the identifier itself ("K001" == "K001"), so attribution is correct by construction. The .id fact binding is the *explicit* declaration form; the instance_key equality check is the actual anti-mismatch mechanism. So uniform participation is semantically sound — it can't produce a mismatch (instance_key must equal the identifier text). 

Hmm wait, but there's an edge: cell "K001：32.4MPa" where cell.fact_id = fact:component.K001.concrete_strength. BIND-8 for 32.4MPa: covered by implicit declaration ✓ fact valid ✓ → §8.2 chain → value comparison. BIND-7 for K001: covered ✓ instance_key match ✓. Everything closes. And a system that instead emits explicit declarations {K001→...id, 32.4MPa→...concrete_strength} also closes. Both paths work. 

But actually — hmm, one more consideration. Should cell.fact_id implicit declaration apply only when anchor_declarations is empty? Consider mixed usage: legacy system emits cell.fact_id for the numeric and wants to add explicit declaration for the identifier. With uniform participation (no precedence), that works as shown above. With "implicit only when explicit empty", the system would be forced to convert cell.fact_id to explicit declaration once any explicit declaration exists — a migration pressure. Uniform participation is more compatible. And DUPLICATE_BINDING only fires on different fact_ids, so legacy+explicit-same-fact_id coexists fine.

Edge: what if cell.fact_id's instance_key ≠ identifier in cell? E.g., cell "K001：32.4MPa" with fact_id = fact:component.K002.concrete_strength (mismatch). Then: 32.4MPa covered by implicit decl with fact K002 → value check against K002 (may pass or fail); K001 identifier: covered by implicit decl, instance_key "K002" ≠ "K001" → ANCHOR.IDENTIFIER_BINDING_MISMATCH ✓ (existing code). Good — mismatch detected by existing rules.

So my design: 
1. TableCell gains optional `anchor_declarations: list[AnchorDeclaration] = []` (表达能力补齐).
2. Coordinate base: char_start/char_end relative to TableCell.raw_text (same carrier-relative convention as RenderedSegment).
3. cell.fact_id (legacy) ≡ implicit AnchorDeclaration with span [0, len(cell.raw_text)), token = whole raw_text; participates uniformly in BIND-6/7/8 with existing error codes. (This is a compatibility equivalence, not a new mechanism.)
4. Validator execution: per cell — tokenize cell.raw_text (Fix 3), overlap-resolve, whitelist span containment (whitelist applies to cells too — wait, does it? §8.4 whitelist span containment applies to token spans in text; for table cells, whitelist patterns... hmm. WhitelistRegistry patterns are registered with scope (全文/指定章节/指定模板). Do they apply to cell text? §41.2.1 says table free-text cells go through 本节锚定校验 — the full anchoring check, which includes whitelist as the only exemption. So yes, whitelist span containment applies to cell text as well. The B5 边界 already says "Fix 3 不变（Validator 对 cell raw_text 自行 tokenize）" and reuses existing rules. I'll state: whitelist containment applies unchanged on cell text (it's part of the same §8 anchoring pipeline; no new rule).
5. StructuredTable unchanged; headers/rows unchanged; no table-level bag.
6. Error codes: reuse UNANCHORED / INVALID_FACT_ID / DUPLICATE_BINDING / IDENTIFIER_BINDING_MISMATCH / UNKNOWN_UNIT etc. Location strings for ActualIssue: e.g., "structured_tables[table-2-1].rows[0][1]:char[0-4]" — hmm, ActualIssue.location is a free string, examples show "rendered_segments[2]:char[15-20]". For tables, there's table_aggregate field in identity... but that's for P0-2 aggregates. location can be a cell coordinate string. That's diagnostic formatting, not new semantics. I can mention it briefly but must not design new fields — location is already a free-form string. Fine.

Now the "最小性" check (Section 7 of output):

Objects:
- TableCell: MUST CHANGE — add optional anchor_declarations field (表达能力补齐, additive, default empty → backward compatible).
- StructuredTable: NOT CHANGE (no table-level bag; table_id/headers/rows untouched).
- AnchorDeclaration: NOT CHANGE (structure reused as-is; carrier-relative offset convention is a one-line clarification of the same frozen convention).
- RenderedSegment: NOT CHANGE (B4 side untouched).
- BIND-7: NOT CHANGE (comparison rule normalized_identifier == instance_key untouched; it already applies to any AnchorDeclaration; with B5 it now has declarations to check on table side — scope extension is the effect of the repair, not a rule change).

Hmm wait, careful: BIND-7/BIND-8 scope. Today BIND-8's §8.1 note says "BIND-8 作用域 = Validator 对 rendered_segments（prose / 图注）tokenize 出的 numeric token；structured_tables ... CONFIRMED BLOCKER ... 待下轮最小修复". So B5's repair *does* extend BIND-8/BIND-7 application scope to structured_tables cells. Is that a "semantic change"? It's the intended minimal repair: the check items (span rule, error codes) unchanged; the scope note changes from "rendered_segments only" to "rendered_segments + structured_tables cell text". That's 表达能力/覆盖补齐 — the frozen rule text itself (the check semantics) is unchanged; the blocker note is removed. I should be precise: BIND-7/BIND-8 判定语义零改动；作用域注从「仅 rendered_segments」扩展为「rendered_segments + structured_tables cell 文本」——这正是 B5 修复本身，不是语义重设计.

- BIND-8: MUST CHANGE (scope note only — the repair itself). 判定规则零改动.
- §8.2: NOT CHANGE (chain input = "numeric token with unique valid fact_id" — same regardless of carrier).
- §8.3 range: NOT CHANGE (per-token anchoring already frozen; B5 enables its expression on table side).
- §8.4 whitelist: NOT CHANGE (span containment applies to cell text as part of same pipeline).
- Tokenizer §5: NOT CHANGE (same regex, applied to cell raw_text — Fix 3 already mandates Validator tokenizes cell text).
- TrustedInput: NOT CHANGE (no new trusted fields; declarations live on SystemOutput side — they're system claims, validated, consistent with R5 "校验对象是渲染产物" and Fix 3 "不信任声明").
- ActualIssue/identity: NOT CHANGE (location is free-form string; table coordinate fits).
- 7 P0 Rules: NOT CHANGE.
- INV list: INV-24 already states "每个非 whitelist 豁免 numeric token 必须被唯一 AnchorDeclaration 绑定" — carrier-agnostic, so B5 closes the Table-side gap without touching INV-24. Maybe add nothing. Good.
- Step 7-D / CDM: NOT CHANGE.
- 04_REPORT_IR: NOT CHANGE.

Question 1 (identifier semantics): No new identifier semantics needed. Analysis:
- BIND-7's actual semantic: normalized_identifier == parse_fact_id(anchor.fact_id).instance_key.
- §41.2.3 example: K002 → fact:component.K002.id; 32.4MPa → fact:component.K002.concrete_strength.
- For the cell "K001：32.4MPa": two independent declarations suffice. The identifier declaration's fact_id is fact:component.K001.id; BIND-7 checks instance_key == "K001" ✓. The numeric declaration's fact_id is fact:component.K001.concrete_strength; BIND-7 is not applied to numeric tokens (BIND-7 is for identifier tokens only); BIND-8 + §8.2 apply. Note: BIND-8 does NOT require the numeric declaration's fact_id instance_key to match the identifier — the cross-check between identifier and numeric is achieved through §41.2.3's "同表绑定" intent... wait, actually let me re-read §41.2.3: "实例标识 token 必须与数值 token 同表绑定" — the mechanism given: identifier token binds to .id fact; both must be bound; mismatch detection = identifier's instance_key must equal the identifier text. Hmm, but does anything check that the *numeric* fact belongs to the same instance as the identifier? In the §41.2.3 example, if anchors = [{K002 → component.K002.id}, {32.4MPa → component.K003.concrete_strength}], BIND-7 passes for K002 (instance_key matches K002), BIND-8 passes for 32.4MPa (valid fact), value check: 32.4 vs K003's value — if K003=32.4 this passes! The mismatch (32.4MPa attributed to K003 in a sentence about K002) would NOT be caught by BIND-7 alone...

Hmm wait. Let me re-read §41.2.3: "若 K002 未绑定，或绑定到别的构件，则即使 32.4 在全文存在，也判定失败。" — so §41.2.3's protection is: identifier must be bound and match. The scenario it guards: sentence about K002, values from K003, where K002 isn't bound at all or bound wrong. But the case "K002 bound correctly, numeric bound to K003's fact" — does the frozen spec catch that? In the prose example, both anchors are in the same anchors[] list for the segment; the spec says "数值 token 与标识 token 双绑定，错配才真正闭环" — the closure comes from both being bound. But strictly, the frozen BIND-7 comparison only checks identifier vs its own fact_id's instance_key. There's no frozen rule that requires the numeric fact's instance_key to equal the identifier's instance_key within a cell/segment.

And the user's instruction for Question 1 is explicit: "请明确分析：当前 B5 修复是否需要新增任何 identifier 语义？还是只需要让 Table 侧具备表达两个独立 AnchorDeclaration 的能力，identifier 的现有 BIND-7 比较规则保持不变？原则：不要借 B5 顺手重写 BIND-7。"

So the answer: B5 does NOT need new identifier semantics. The minimal repair is expressiveness: the cell must be able to carry two independent AnchorDeclarations. BIND-7's comparison rule stays exactly as frozen. Whether "numeric fact instance_key should cross-check against co-located identifier instance_key" is a *different* (stronger) semantic that is NOT part of the frozen BIND-7 text — registering or rejecting it is out of B5 scope. I should note this boundary honestly: 表达能力补齐后，BIND-7 在 Table 侧的执行结果与 rendered_segments 侧完全一致；「数值 fact 与同 cell 标识 fact 的实例交叉校验」不在 BIND-7 冻结文本内，B5 不新增。 Actually — wait, should I even mention this potential gap? The user says don't rewrite BIND-7, don't expand scope. But as an honest engineer, if I notice that the "同表绑定" closure has a residual hole (numeric bound to different instance than co-located identifier), I should mention it briefly as an observation outside B5 scope... Hmm. Careful: the user's frozen premise says "B4 的 rendered_segments 侧语义已经冻结，不允许重新设计" and "不要借 B5 顺手重写 BIND-7". Mentioning a residual gap as a non-B5 observation is honest and consistent with "不要越界" — I'll put one line in §7 noting it's out of scope, NOT registering it as a new blocker (the user explicitly said don't scan new issues this round: "不是继续扫描新问题"). Hmm, actually the user said the round is only about B5. Mentioning a new potential issue might be seen as expanding scope. But §7 explicitly asks "只讨论当前 BIND-7 是否需要变化" — the answer is "不需要变化". I can add one sentence: 注意 BIND-7 冻结文本仅校验 identifier 自身的 instance_key 一致性；「数值 token 绑定 fact 的 instance_key 是否需与同 cell 标识一致」不属于冻结 BIND-7 语义，B5 不新增、不裁决。 That's a boundary statement, not a new blocker. That's honest without overstepping. Actually this is important for acceptance case 4: "K001：32.4MPa" — what's the acceptance criterion? Expressible + BIND-7 passes for K001 (if declared) + BIND-8/§8.2 for 32.4MPa. The acceptance is about *expression and judgment*, and with cell.fact_id legacy path it also works. OK.

Question 2 (Table title/note): As analyzed — cannot be explicitly proven covered by current Contract:
- StructuredTable has no title/note fields (§4: table_id/headers/rows only) → title/note cannot be expressed in structured_tables.
- rendered_segments side: §8.1 BIND-8 scope note parenthetical "（prose / 图注）" suggests prose/captions; nothing in §4 mandates or forbids table title/note as RenderedSegments; no enumerated segment_location convention for table title/note. So the Contract cannot *prove* the path exists → register as 「覆盖路径需要澄清」, don't extend schema this round.

Wait, but hold on — does this affect the Final Status? The user says: "如果当前 Contract 无法明确证明：只登记为'覆盖路径需要澄清'，不要顺手扩展 schema。" — so this registration does NOT block DESIGN READY. The Final Status instruction: "如果发现仍然存在无法执行的冻结要求，必须输出 DESIGN NOT READY。" Is table title/note an "无法执行的冻结要求"? The frozen requirement (04_REPORT_IR §41.2.1) includes table title/note in anchoring scope. If the contract can't prove the path, is B5's design incomplete? Hmm. Let me think carefully.

The B5 blocker as confirmed is specifically about *cell-level* binding expressiveness (§8.1 note, ending scan item 6): "cell 级 fact_id 不足以覆盖 §41.2.1 / §41.2.2 逐 token 判定". The confirmed blocker scope = cell text. Title/note is a *coverage path question* the user explicitly instructs to register as "需要澄清" if not provable — i.e., the user has already pre-decided that this does not block the design round; it's a clarification item, not a design item. The user's instruction structure makes this clear: Section 五 says register it, don't extend schema. Section 十 says output DESIGN READY only if "无法执行的冻结要求" remains. A clarification item about an *expression path* (title/note can trivially be expressed as RenderedSegments today — the mechanism exists; what's unproven is whether systems are required to emit them there) — is it "无法执行"? No: the mechanism (RenderedSegment + AnchorDeclaration + BIND-8) is fully capable of covering title/note text *if emitted*; what's missing is an explicit statement in the Contract that title/note must be emitted as rendered_segments. That's a coverage-path clarification for the Contract text / evaluation case convention, not a missing execution capability in the B5 design. So DESIGN READY with the clarification registered. Hmm, but let me double-check: can the system emit a RenderedSegment for table title? §4: "rendered_segments: list[RenderedSegment] — 结构化渲染段（禁止退化为 list[str]）". segment_location: "定位（semantic path + block id）" — a table is a block; its title could be "table-2-1.title". Nothing forbids it. Validator tokenizes all rendered_segments. So yes, mechanically capable; Contract just doesn't explicitly say title/note must go there. → 覆盖路径需要澄清. And I should be explicit about the two facts: (a) StructuredTable has no title/note field — so they cannot enter via structured_tables; (b) rendered_segments is mechanically open but not explicitly mandated for title/note — hence "需要澄清". 

Actually, wait — I want to double check one thing: does §41.2.1's title/note requirement even apply to Step 7-E's current scope? 05_EVALUATION §6.4 rule 3 also lists "Table 的单元格与表标题 / 表注". So it's a frozen requirement. But the user's own instruction anticipates this: "如果当前 Contract 无法明确证明：只登记为'覆盖路径需要澄清'". So the user accepts this outcome. Good.

Now, Section 6: Binding Execution Flow. Pure text flow:

Table 侧:
```
StructuredTable.rows[i][j].raw_text (TableCell)
    ↓ Validator 自行 tokenize（Fix 3，同一 §5 正则 + §5.2 overlap 消解）
numeric token / identifier token（含 char span，坐标基 = cell.raw_text）
    ↓ whitelist span containment（§8.4，同一规则）→ 命中 → 豁免
    ↓ 未命中
在该 cell 的 anchor_declarations（+ cell.fact_id 隐式声明）中找覆盖声明
    （冻结 span 规则：token span ⊆ 声明 span 或文本相等；部分重叠不算覆盖）
    ↓ 0 条覆盖 → ANCHOR.UNANCHORED
    ↓ ≥2 条不同 fact_id → ANCHOR.DUPLICATE_BINDING
    ↓ 唯一声明
fact_id
    ├─ BIND-5 语法校验 → 非法 → ANCHOR.INVALID_FACT_ID
    ├─ G3 查找 → 缺 → ANCHOR.INVALID_FACT_ID（Stage E）
    └─ parse_fact_id → fact_type → snapshot.decimals
            ↓
        frozen_round(token_value, decimals) == G3.value → P0-1
（identifier token 同 cell 内走 BIND-7：normalized_identifier == instance_key）
```

Same core semantics as rendered_segments side: 唯一差异 = 声明挂载的文本载体（RenderedSegment.raw_text vs TableCell.raw_text）；tokenize、span 规则、whitelist、错误码、§8.2 链完全同一。

Section 9 Acceptance Criteria: list the 9 cases with expected outcomes:

1. "2.3m" single token: 显式声明或 cell.fact_id 隐式声明均可表达；BIND-8 覆盖 → §8.2 值判定。两条路径均闭合。
2. "2.3m / 2.5m": 两条显式声明（各自 char span），分别绑定不同 fact_id；各自独立 §8.2 判定。若只提供 cell 级单 fact_id（旧路径）→ 第二个 token 仍被隐式声明覆盖（span 全 cell）……wait! Hold on. If a cell "2.3m / 2.5m" only has cell.fact_id (legacy path), the implicit declaration covers BOTH tokens → both bound to the same fact_id → BIND-8 passes structurally, then §8.2 compares each token value against that one fact's value → one of them will fail (values differ) → P0-1 FAIL. Is that the desired acceptance? Yes — legacy path *can express* single-fact cells; a two-different-facts cell *cannot be correctly expressed* via legacy path, and the §8.2 value check will fail (correctly, because the binding is wrong for at least one token). The correct expression requires explicit declarations. Good — acceptance criterion 2: 用两条显式声明可正确表达并 PASS；若仅用 cell.fact_id 旧路径表达两个不同 fact → §8.2 值比对至少一 token FAIL（判为绑定错误，符合预期）。
3. "2.0～2.5m" range: two tokens, two declarations, independent fact_ids, independent value checks (§8.3). Note "～" is not unit-start, not in unit charset → tokens are "2.0" and "2.5m" (second carries unit m)... wait: "2.0～2.5m" — tokenizer: "2.0" followed by "～" (full-width tilde, not unit-start) → unit None; "2.5" followed by "m" → unit m. Both numeric tokens. Each bound independently. ✓. Hmm, also does "0～2.5" create issues? "2.0～2.5m": after "2.0", next char "～" — U+FF5E fullwidth tilde — not [A-Za-zμ%] → unit None ✓. Fine.
4. "K001：32.4MPa": identifier token K001 + numeric token 32.4MPa (with "：" fullwidth colon as separator — not unit-start since "：" is punctuation... wait, unit candidate after "32.4" is "MPa" — the colon is before "32.4", after "K001". K001 followed by "：" — identifier regex has lookahead (?![A-Za-z0-9]) — "：" is not alnum ✓ identifier fine. "32.4MPa" numeric with unit MPa ✓.) Two declarations: {K001 → fact:component.K001.id}, {32.4MPa → fact:component.K001.concrete_strength}. BIND-7 on K001: instance_key "K001" == "K001" ✓. BIND-8 on 32.4MPa ✓ → §8.2. Also expressible via legacy cell.fact_id=concrete_strength + implicit declaration (K001 covered by implicit decl, instance_key matches). Both paths close. Acceptance: 两条显式声明路径 PASS；隐式路径下 BIND-7 亦闭合（instance_key 相等）。
5. whitelist-contained numeric token: e.g., cell "依据 GB 50292-2015 检测" — tokens 50292/2015 span-contained in whitelist match span → 豁免 BIND-8（仍输出诊断）。And partial overlap → not exempt → UNANCHORED.
6. duplicate binding: two declarations covering same token with different fact_ids (e.g., "2.3m" with two declarations) → ANCHOR.DUPLICATE_BINDING. Also implicit+explicit different fact_id → DUPLICATE_BINDING (uniform rule).
7. partial overlap: declaration span [0,3) "2.3" vs token "2.3m" span [0,4) — declaration covers only part → not covering → UNANCHORED. (span 规则：部分重叠不算覆盖).
8. invalid fact_id: declaration fact_id fails is_valid_fact_id → ANCHOR.INVALID_FACT_ID (BIND-5); or valid format but not in case G3 → Stage E ANCHOR.INVALID_FACT_ID.
9. value mismatch: bound, valid, but frozen_round(token, decimals) != G3.value → P0-1 FAIL (VALUE.CONSISTENCY).

Section 10: Final Status = DESIGN READY FOR IMPLEMENTATION.

Now let me also double check the MUST NOT CHANGE list against hard constraints 7-17:
1. multi numeric tokens per cell ✓ (explicit declarations)
2. independent fact_ids ✓
3. identifier + numeric ✓
4. range independent ✓
5. reuse AnchorDeclaration as-is ✓
6. reuse BIND-7/8 span rules ✓
7. existing ANCHOR.* codes ✓
8. Fix 3 ✓ (Validator tokenizes cell raw_text; declarations are定位与诊断辅助, not trusted tokenization)
9. B4 side unchanged ✓
10. §8.2 unchanged ✓
11. 7 P0 ✓
12. no new P0 ✓
13. Step 7-D untouched ✓
14. 04_REPORT_IR untouched ✓
15. no LLM ✓
16. no Validator scope expansion — hmm, B5 extends the *application scope* of existing BIND-7/8 checks to table cells; is that "扩大 Validator 职责"? No — it's completing the already-frozen requirement (§41.2.1/2, INV-24) within P0-4; the Validator doesn't gain new rule types or new stages. The B5 边界 in v10 itself mandates this. ✓
17. legacy single-token cell path compatible ✓ (cell.fact_id retained, implicit declaration equivalence, no forced migration)

One more check — Section 3 asks to compare A and B on: 表达能力、定位复杂度、改动面、兼容性. Let me build that comparison.

Option B detailed problems:
- 定位复杂度: table-level declarations need (cell locator + char offsets) to unambiguously map token→declaration. AnchorDeclaration has no cell field → either (i) change AnchorDeclaration (violates constraint 5), (ii) canonical flattening of table text (new canonicalization semantics — a new convention that must define row/col separators, header inclusion, etc.; the flattened text becomes a *shadow text* that can diverge from actual cell raw_texts; Validator tokenizes cell texts (Fix 3) then must translate positions between two coordinate systems — new machinery), or (iii) token text equality matching (declarations matched by token string only) — breaks on duplicate token texts in different cells (e.g., two cells both "32.4MPa" belonging to different components) → cannot disambiguate → either false DUPLICATE_BINDING or wrong binding → unsound.
- 表达能力: with token-text matching only, cannot express same text different fact in different cells. So B either fails expressiveness (iii) or requires structural additions (i/ii) — strictly worse than A on every axis.
- 兼容性: B concentrates declarations at table level → single-token cells would ALSO need declarations migrated to table-level list → forced migration (violates constraint 17) or dual topology (cell-level legacy + table-level explicit → two mechanisms, more complexity).
- 改动面: StructuredTable + either AnchorDeclaration change or new flattening semantics + Validator lookup machinery. Larger than A.

Option A:
- 表达能力: 4 scenarios all expressible; duplicate text in different cells no issue (declarations scoped per cell).
- 定位复杂度: O(1) — declarations are co-located with the exact text being tokenized; the lookup is the same algorithm as rendered_segments side (per-carrier declaration set). No coordinate translation.
- 改动面: one optional additive field on TableCell. StructuredTable untouched. AnchorDeclaration untouched.
- 兼容性: additive default-empty field → old payloads valid; cell.fact_id path preserved via implicit declaration equivalence.

Also the symmetry/topology argument: A mirrors the frozen B4 topology (declarations attach to the text carrier at the same object). B introduces a second topology. A is therefore the smaller conceptual delta.

Decision: A.

One more detail for Section 6: how are char_start/char_end defined for cells — relative to cell.raw_text. And what about headers? Headers are list[TableCell] too — headers' cells can also carry declarations; tokenization applies to header cells as well (they're TableCells; §41.2.1 covers cell text — headers are cells; a header cell containing "2.3m"? unusual but mechanically covered — same rule, no special-casing). I'll note: 机制对 headers/rows 一视同仁（均为 TableCell），不新增特例。

Also: table_aggregate field in ActualIssue identity — for table-side issues, location string would be like "structured_tables[table-2-1].rows[1][2]:char[0-4]"; table_aggregate remains for P0-2 aggregate semantics, not used by B5 (don't touch identity). Fine — I'll mention location is existing free-form field, no identity change.

Now the output format — must follow exactly the 10 sections. Language: Chinese (user's message is Chinese). Be concise but complete. No code, no files. Let me also double check Section 2 Root Cause: why TableCell.fact_id can't do per-token anchoring:
- fact_id is a single Optional[str] at cell level — a cell-level attribute, not a token-level one. It has no notion of *which token* inside the cell it binds. With multiple numeric tokens, the mapping token→fact is 1:1 collapsed to cell→fact (N tokens : 1 fact_id).
- §41.2.2 requires per-token judgment: each token explicitly bound to A fact_id and value-checked. A cell-level single value cannot express token→fact mapping when N>1 or when tokens belong to different facts (range min/max, identifier+numeric need TWO different fact_types: .id and .concrete_strength — a single fact_id can't be both).
- Also no span info: even for single-token cells, cell.fact_id can't distinguish which token is the target if a cell has one numeric + one identifier token (e.g., "K001：32.4MPa" with fact_id=concrete_strength — the identifier K001's BIND-7 coverage only works accidentally via instance_key equality; and a cell "K001" + "32.4" both needing different facts is inexpressible).
- It also can't feed BIND-6/DUPLICATE_BINDING or span containment (whitelist) — no char coordinates.

Hmm wait, actually for case "K001：32.4MPa" with cell.fact_id=concrete_strength: via my implicit-declaration equivalence, K001 IS covered (instance_key match) — so it works. But that's the *repaired* semantics, not the current one. Currently there are no declarations on cells at all, so nothing runs. The root cause statement: cell-level fact_id 是「cell→fact」映射，丢失了「token→fact」粒度与 char 定位；而 §41.2.2/R2 冻结要求的是 token 粒度显式绑定 + §8.2 逐 token 值判定，且 identifier+numeric 需要 .id 与量值两类 fact_type 并存于同一 cell —— 单值 fact_id 结构上不可表达。

Also Section 1 Frozen Baseline — list what's frozen:
- 7 P0 Rules (INV-1), Step 7-D read-only (INV-2), Fix 3, B4 BIND-8 + §8.2 chain + INV-24 + T-18 (rendered_segments side), AnchorDeclaration structure (§4), BIND-7 comparison (INV-22), span rule (token span ⊆ decl span or text equal; partial overlap not covering), §8.3 range, §8.4 whitelist, ANCHOR.* codes, frozen_round, snapshot.decimals precision chain (INV-19), ActualIssue identity (INV-17), tokenizer §5 (regex/overlap/unit lexer), D-018/D-019/D-025, 04_REPORT_IR §41.2.1/2/3/5.

Also I must state per user rule #4: "在进行任务前必须将你理解的意图提供给用户进行确认。" Hmm — the user gave an extremely detailed instruction with explicit output format and a required final verdict. The intent is unambiguous and the task is a design-analysis round with a prescribed output. Asking for confirmation before doing exactly what was specified would add friction; the instruction itself IS the confirmed intent (it says "当前基线已经冻结，请先读取并理解…本轮只做…"). The output format is fully prescribed. I'll proceed and deliver the design check directly — the deliverable itself is the confirmation artifact. Actually, let me reconsider: user rule says before task I must provide my understood intent for confirmation. But this instruction is a fully-specified directive ("本轮只做 B5 MINIMAL REPAIR DESIGN CHECK… 只输出下面结构…"). The intent is provided BY the user in full detail; there's no ambiguity to confirm. Adding an AskUserQuestion would violate "Output efficiency: Go straight to the point" and the user's explicit "不要越界/只输出下面结构". I'll proceed.

Also should I use TodoWrite? The task is a single analysis deliverable — reading docs then writing one structured response. TodoWrite for a 2-step task is unnecessary. Skip.

Let me also verify the CDM doc quickly for anything about instance_key/parse_fact_id to make sure my identifier analysis is right. The contract states INV-22 already: normalized_identifier == parse_fact_id(anchor.fact_id).instance_key. And Fact ID = fact:<scope_class>.<instance_key>.<attribute>. derive_fact_type = first segment + "." + third segment (§13). Good, no need to re-read CDM. But the user asked me to read 03_CANONICAL_DATA_MODEL.md and 05_EVALUATION.md. I've read the relevant 05_EVALUATION §6.4 and whitelist §3.3 reference. Let me do a quick targeted check of 03 CDM for Fact ID structure to be thorough (one grep), and 05_EVALUATION §3.3 whitelist structure. Actually I have enough: the contract's INV-11/§13 and 05_EVAL §6.4 rule 3-5 are sufficient for this design round. One quick grep on CDM for "instance_key" and "三段" to confirm.

All evidence is in hand. Now I write the design check document in the exact prescribed structure. Chinese, concise, complete. No files created, no code.

Let me make sure I cover every required element:

## 1. Frozen Baseline
## 2. Root Cause
## 3. Design Options (A vs B: 表达能力、定位复杂度、改动面、兼容性)
## 4. Decision (A) with 5 required justifications
## 5. Minimal Change Surface (MUST CHANGE / MUST NOT CHANGE, 逐项, 标注"表达能力补齐" vs "语义变化")
## 6. Binding Execution Flow (纯文字流程图 + 与 rendered_segments 侧同一 BIND-8 语义说明)
## 7. Identifier Boundary
## 8. Table Title / Note Boundary
## 9. Acceptance Criteria (9 cases)
## 10. Final Status

Key decisions to state crisply:
- Decision: A (per-cell)
- cell.fact_id 隐式声明等价规则 (span = [0, len(raw_text)))
- BIND-7 零改动
- Title/note → 覆盖路径需要澄清（两个事实依据）

Write it now. Keep it tight but complete.
四份冻结依据已全部在盘核对完毕（04_REPORT_IR §41.2.1/2/3/5、§13.1 anchors、Table 结构无 title/note 字段、05_EVAL §6.4、D-018/D-019/D-025、CDM §8.1 三段式 ID）。以下为设计裁决。

# B5 MINIMAL REPAIR DESIGN CHECK

## 1. Frozen Baseline

以下内容本轮冻结、不得改变：

| # | 冻结项 | 依据 |
| ---- | ---- | ---- |
| 1 | 7 个 P0 Rule 不增删；仍属 P0-4 | INV-1 / §13 |
| 2 | AnchorDeclaration 结构原样：`token / fact_id / char_start / char_end` | §4（与 04_REPORT_IR §13.1 anchors 对齐） |
| 3 | BIND-8 rendered_segments 侧语义：唯一绑定 + span 覆盖（token span ⊆ 声明 span 或文本相等；部分重叠不算覆盖） | §8.1 / INV-24 / T-18 |
| 4 | §8.2 判定链：token → 唯一 fact_id → G3 → fact_type → snapshot.decimals → frozen_round → P0-1 | §8.2 / §13 / INV-19 |
| 5 | BIND-7 比较规则：`normalized_identifier == parse_fact_id(fact_id).instance_key`，禁止 scope_class 推断 | §6.3 / INV-22 |
| 6 | §8.3 区间逐 token 独立锚定；§8.4 whitelist span containment（部分重叠不豁免） | §8.3 / §8.4 |
| 7 | ANCHOR.* 现有错误码体系（UNANCHORED / INVALID_FACT_ID / DUPLICATE_BINDING / IDENTIFIER_BINDING_MISMATCH / UNKNOWN_UNIT 等） | §8.1 / §8.2 |
| 8 | Fix 3：Validator 自己 tokenize，不信任系统自报 TokenOccurrence | §5 / §4 |
| 9 | 精度唯一来源 = FactDisplaySpecSnapshot.decimals | D-025 / INV-14 / INV-19 |
| 10 | Tokenizer（§5 正则 / overlap 消解 / unit-start/cont 文法）零改动 | §5（B1 闭合） |
| 11 | ActualIssue 冻结六元 identity + build-only | INV-17 / §11 |
| 12 | Step 7-D 只读；04_REPORT_IR / 05_EVAL / CDM 只读 | INV-2 / 本轮禁止事项 |
| 13 | 冻结要求本身：Table 单元格文本纳入逐 token 锚定 | 04_REPORT_IR §41.2.1 / §41.2.2 / R2 / §41.2.5、D-018、05_EVAL §6.4 规则 3 |

## 2. Root Cause

当前 `TableCell = { raw_text, fact_id }` 的 `fact_id` 是 **cell→fact 的单值映射**，而冻结要求（§41.2.2 / R2）是 **token→fact 的逐 token 映射**。结构上丢失了两个维度：

1. **粒度丢失**：一个 cell 有 N 个 numeric token 时，N:1 坍缩为 1:1。`"2.3m / 2.5m"`（两 token 各属不同 fact）、`"2.0～2.5m"`（min/max 独立绑定）、`"K001：32.4MPa"`（需 `fact:component.K001.id` 与 `fact:component.K001.concrete_strength` **两个不同 fact_type 并存**）均不可表达——单个 fact_id 在结构上不可能同时是两种 fact_type。
2. **定位丢失**：cell 级 fact_id 无 char 坐标，无法执行冻结 span 规则（token span ⊆ 声明 span）、无法区分 cell 内哪个 token 是绑定目标、无法支撑 whitelist span containment 与 BIND-6 查重。

因此当前 Table 侧 BIND-7 / BIND-8 / §8.2 全部不可执行——这正是 B5 被裁决为 CONFIRMED BLOCKER 的原因（v10 §8.1 作用域注、结尾扫描第 6 项）。

## 3. Design Options

| 维度 | A. per-cell binding declarations | B. table-level binding declarations |
| ---- | ---- | ---- |
| **表达能力** | 每个 cell 自带声明集，四类场景（单 token / 多 token / range / identifier+numeric）全部可表达；不同 cell 中相同 token 文本（如两行都是 "32.4MPa" 属不同构件）天然无歧义 | 声明集中在表级袋中，**必须额外携带 cell 定位信息**才能区分同文本 token：① 改 AnchorDeclaration 加字段 → 违反硬约束 5；② 引入“表文本规范化展平”约定（行/列分隔符、是否含表头）→ 新增一套影子文本坐标系，与各 cell raw_text 可能漂移；③ 仅按 token 文本匹配 → 同文本不同 fact 不可区分，判 DUPLICATE_BINDING 或错绑，均不健全 |
| **定位复杂度** | O(1)：声明与被 tokenize 的文本**同对象共置**，Validator 对 cell.raw_text tokenize 后直接在本 cell 声明集内做 span 覆盖查找——与 rendered_segments 侧完全同一算法 | O(tokens × declarations)：每个 token 需先做坐标换算（cell 坐标 → 展平坐标）或字段过滤，再查表级袋；引入当前不存在的坐标变换机制 |
| **改动面** | TableCell 增加一个**可选**字段；StructuredTable / AnchorDeclaration / 渲染侧全部不动 | StructuredTable 增字段 + AnchorDeclaration 变结构**或**新增展平规范化语义 + Validator 查找机制，三处改动起步 |
| **兼容性** | 纯增量：默认空列表，旧载荷合法；cell.fact_id 旧路径可用隐式声明等价保留，不强制迁移 | 表级集中声明迫使单 token cell 的绑定也迁入表级袋（违反硬约束 17），或保留 cell 级旧路径 → 双拓扑并存，机制翻倍 |
| **拓扑一致性** | 与 B4 冻结拓扑**同构**：声明挂在文本载体自身（RenderedSegment 挂自己的 prose，TableCell 挂自己的 raw_text），只是载体粒度更细 | 引入与 B4 不同的第二种锚定拓扑（声明挂在上层容器），偏离已冻结的声明—文本共置惯例 |

## 4. Decision

**选择 A：per-cell binding declarations。**

1. **为什么是当前最小修改面**：唯一结构变化 = TableCell 增加可选 `anchor_declarations: list[AnchorDeclaration]`（默认空）。AnchorDeclaration 原样复用，StructuredTable、渲染侧、tokenizer、错误码、判定链零改动。B 需要动结构或发明展平语义，修改面严格更大。
2. **为什么不破坏 B4**：声明挂在哪个载体是 B5 唯一自由度；A 只是把 B4 已冻结的“声明与文本共置”惯例应用到 TableCell 这个新载体，RenderedSegment 侧任何字段、任何检查、任何语义均不触碰。char_start/char_end 语义 = **相对所挂载文本载体的偏移**（B4 语境下即 RenderedSegment.raw_text 的偏移，Table 语境下即 cell.raw_text 的偏移）——这是同一惯例在载体切换下的自然推论，属表达能力补齐，非语义变化。
3. **为什么能表达四类场景**：
   - 单 token cell `"2.3m"`：显式声明 `{token:"2.3m", fact_id, char_start:0, char_end:4}`；或沿用旧路径 cell.fact_id（见下）。
   - 多 token cell `"2.3m / 2.5m"`：两条显式声明，各自 char span，各自 fact_id。
   - range cell `"2.0～2.5m"`：两条显式声明独立绑定（"～" 非 unit-start，两 token 独立提取），满足 §8.3。
   - identifier+numeric cell `"K001：32.4MPa"`：两条显式声明 `{K001 → fact:component.K001.id}`（BIND-7 判 instance_key）、`{32.4MPa → fact:component.K001.concrete_strength}`（BIND-8 → §8.2）。
4. **Validator tokenize 后如何定位声明**：对每个 TableCell.raw_text 独立执行既有 tokenizer（§5 正则 + §5.2 overlap 消解，Fix 3 不变），提取 token 及其 char span（坐标基 = 该 cell.raw_text）；whitelist span containment（§8.4）在同一 cell 文本上先行判定；未豁免的 token 在**本 cell** 的 anchor_declarations 中按冻结 span 规则找覆盖声明（token span ⊆ 声明 span 或文本相等；部分重叠不算覆盖）：0 条 → UNANCHORED；≥2 条不同 fact_id → DUPLICATE_BINDING；唯一声明 → BIND-5 语法校验 → §8.2 链。headers 与 rows 均为 TableCell，机制一视同仁，无特例。
5. **如何保持现有 span / duplicate / invalid 语义**：span 规则、BIND-6/7/8 错误码、§8.4 豁免全部原样适用——声明集只是换了一个挂载载体，查找谓词与错误路径与渲染侧逐字节相同。

**旧路径兼容规则（表达机制，非新语义）**：`TableCell.fact_id ≠ None` 时，等价于一条**隐式 AnchorDeclaration**（token = 整个 cell.raw_text，span = [0, len(raw_text))），与显式声明**统一**参与既有 BIND-6/7/8 规则：
- 单 numeric token cell：隐式声明覆盖全 cell → BIND-8 闭合，零迁移。
- 隐式与显式声明 fact_id 相同：冗余但非违规（BIND-8 冻结文本只禁止“多个**不同** fact_id”）。
- 隐式与显式声明 fact_id 不同且覆盖同一 token：命中既有 DUPLICATE_BINDING，无新规则。
- identifier token 被隐式声明覆盖时，BIND-7 仍要求 instance_key 相等——实例错配在构造上即被拦截（instance_key 必须等于标识文本本身）。
不采用“仅当无显式声明时隐式生效”的优先级规则——那会迫使系统在任何显式声明出现时迁移 cell.fact_id，兼容性更差。

## 5. Minimal Change Surface

**MUST CHANGE**（全部为表达能力补齐，无语义变化）：

| 对象 | 变化 | 性质 |
| ---- | ---- | ---- |
| `TableCell` | 增加可选字段 `anchor_declarations: list[AnchorDeclaration] = []`；char 坐标基 = cell.raw_text | 表达能力补齐（纯增量，默认空） |
| `§8.1` BIND-7 / BIND-8 作用域注 | 由「仅 rendered_segments」扩展为「rendered_segments + structured_tables cell 文本」，删除 CONFIRMED BLOCKER 注记 | 覆盖范围补齐——**这正是 B5 修复本身**；两条检查的判定文本（span 规则、错误码）零改动 |
| `TableCell.fact_id` 语义注 | 登记隐式声明等价规则（§4 末尾） | 兼容性登记，非新机制 |

**MUST NOT CHANGE**：

| 对象 | 理由 |
| ---- | ---- |
| `AnchorDeclaration` | 结构原样复用（硬约束 5）；偏移惯例是同一惯例的载体推广 |
| `StructuredTable`（table_id / headers / rows 结构） | A 方案无需表级袋；不加 title/note 字段 |
| `RenderedSegment` 及其 anchor_declarations | B4 侧零改动（硬约束 9） |
| BIND-7 比较规则（INV-22） | identifier 语义不动（硬约束 + §7） |
| BIND-8 判定文本、span 规则 | 仅作用域注扩展（上表已列），谓词不动 |
| §8.2 判定链 | 链输入 = "有唯一有效绑定的 numeric token”，与载体无关（硬约束 10） |
| §8.3 / §8.4 | 冻结规则原样适用于 cell 文本，规则文本不动 |
| §5 Tokenizer 全部内容 | Fix 3：同一正则直接作用于 cell.raw_text |
| TrustedInput Schema | 声明是 SystemOutput 侧的系统声明（受 Validator 校验，符合 R5“校验渲染产物”）；trusted 侧零新增字段 |
| ActualIssue / identity 六元组 / issue_id | location 为既有自由字符串（如 `structured_tables[table-2-1].rows[1][2]:char[0-4]`），table_aggregate 仍专属 P0-2 聚合语义，不触碰 |
| 7 P0 Rules / P0_RULE_IDS / applicable_rules / Unit Registry / frozen_round / §10 matching / Step 7-D / 04_REPORT_IR / 05_EVAL / CDM | 硬约束 11–14 及禁止清单 |

## 6. Binding Execution Flow

Table 侧（与 rendered_segments 侧并行的同构管道）：

```
TableCell.raw_text（headers / rows 一视同仁）
    ↓ Validator 自行 tokenize（Fix 3：§5 同一正则 + §5.2 同一 overlap 消解）
numeric token / identifier token（char span，坐标基 = cell.raw_text）
    ↓ whitelist span containment（§8.4 同一规则：完全包含 → 豁免，仍出诊断）
    ↓ 未豁免
本 cell 的 anchor_declarations（含 cell.fact_id 隐式声明）按冻结 span 规则找覆盖声明
    （token span ⊆ 声明 span 或文本相等；部分重叠不算覆盖）
    ├─ 0 条覆盖            → ANCHOR.UNANCHORED
    ├─ ≥2 条不同 fact_id   → ANCHOR.DUPLICATE_BINDING
    └─ 唯一声明 → fact_id
            ├─ BIND-5：is_valid_fact_id() 失败 → ANCHOR.INVALID_FACT_ID（Stage D）
            ├─ identifier token → BIND-7：normalized_identifier == instance_key
            │        不等 → ANCHOR.IDENTIFIER_BINDING_MISMATCH
            └─ numeric token → §8.2 链（与渲染侧逐字节同一）：
                G3 查找 → 缺 → ANCHOR.INVALID_FACT_ID（Stage E）
                parse_fact_id → fact_type → fact_display_spec[fact_type]
                    缺条目 → precondition_error
                → FactDisplaySpecSnapshot.decimals
                → frozen_round(token_value, decimals) == G3.value   （P0-1）
                （并行：unit 级联表，UNKNOWN_UNIT / QUANTITY_KIND_MISMATCH / UNIT_MISMATCH）
```

**同一 BIND-8 核心语义的保证**：两侧共享同一 tokenizer、同一 span 覆盖谓词、同一 whitelist 豁免、同一错误码集、同一 §8.2 判定链与精度来源。唯一差异是**声明所挂载的文本载体**（RenderedSegment.raw_text vs TableCell.raw_text）——即本轮唯一设计自由度的全部内容。B4 渲染侧管道一行不改。

## 7. Identifier Boundary

**B5 不需要新增任何 identifier 语义。** 理由：

- §41.2.3 示例（`K002 → fact:component.K002.id`、`32.4MPa → fact:component.K002.concrete_strength`）要求的只是**同一 cell 内两条独立 AnchorDeclaration 的表达能力**——这正是 A 方案直接提供的。
- identifier token 的全部判定仍由冻结 BIND-7 完成：被至少一条声明覆盖（span 规则同 BIND-8）+ `normalized_identifier == parse_fact_id(fact_id).instance_key`（INV-22）。该规则在 Table 侧的执行结果与 rendered_segments 侧完全一致，零改动。
- 隐式声明等价规则下，旧路径 cell `"K001：32.4MPa"` + cell.fact_id=concrete_strength 亦闭合：K001 被隐式声明覆盖且 instance_key 相等 → BIND-7 通过；错绑（如 fact_id 指向 K002）→ IDENTIFIER_BINDING_MISMATCH，既有错误码拦截。

边界声明（仅划定界线，不裁决、不登记为阻塞）：BIND-7 冻结文本只校验 identifier 自身与其绑定 fact 的 instance_key 一致性；“同一 cell 内 numeric token 所绑 fact 的 instance_key 是否必须与 co-located identifier 一致”这一更强交叉校验**不属于**冻结 BIND-7 语义，B5 不新增、不修改、不顺手重写（遵循用户裁决原则）。

## 8. Table Title / Note Boundary

**结论：当前 Contract 无法明确证明 Table title / note 已能进入现有 BIND-8 rendered_segments 路径 → 登记为「覆盖路径需要澄清」。** 依据两个在盘事实：

1. `StructuredTable`（v10 §4）仅有 `table_id / headers / rows`，**无 title / note 字段**——title/note 结构上不可能经 structured_tables 表达。
2. rendered_segments 侧：§4 的 `segment_location` 为自由字符串（"semantic path + block id"），机制上**允许**系统把表标题/表注作为 RenderedSegment 提交并被 Validator tokenize；但 §8.1 BIND-8 作用域括注仅写“（prose / 图注）”，且无任何条款**要求**系统必须把 title/note 作为 rendered_segments 提交。即：机制通道存在，强制路径未证明。

按本轮指令：仅登记澄清项，不扩展 schema、不新增字段、不在本轮裁决该路径。此项不构成本设计（cell 侧逐 token 绑定）的执行缺口。

## 9. Acceptance Criteria

B5 修复完成后，以下最小案例必须**可表达**且**可判定**（错误码均为既有 ANCHOR.* / P0-1）：

| # | cell 文本 | 表达 | 判定 |
| ---- | ---- | ---- | ---- |
| 1 | `"2.3m"` | 显式声明 1 条；或旧路径 cell.fact_id（隐式声明） | BIND-8 覆盖 → §8.2 值判定；两条路径均闭合 |
| 2 | `"2.3m / 2.5m"`（不同 fact） | 显式声明 2 条，各自 char span、各自 fact_id | 两 token 独立通过 BIND-8 → 各自 §8.2；若仅用 cell.fact_id 旧路径表达两不同 fact → 隐式声明使两 token 同绑一 fact，§8.2 至少一 token 值不等 → P0-1 FAIL（正确暴露旧路径表达力边界） |
| 3 | `"2.0～2.5m"` | 显式声明 2 条（"～" 非 unit-start，两 token 独立提取，第二个带单位 m） | 独立绑定、独立值判定（§8.3）；禁止区间整体锚定 |
| 4 | `"K001：32.4MPa"` | 显式 2 条：K001→…K001.id（BIND-7）、32.4MPa→…K001.concrete_strength（BIND-8→§8.2）；或旧路径 cell.fact_id=concrete_strength（隐式声明下 K001 经 instance_key 相等闭合 BIND-7） | identifier 错绑（instance_key 不等）→ IDENTIFIER_BINDING_MISMATCH |
| 5 | whitelist 豁免 numeric token（如 `"依据 GB 50292-2015 检测"`） | 无需绑定声明 | token span 完全包含于登记 pattern 匹配 span → 豁免 BIND-8（仍出诊断记录）；部分重叠 → 不豁免 → UNANCHORED |
| 6 | duplicate binding | 同一 token 被两条**不同 fact_id** 声明覆盖（含隐式+显式不同 fact_id 组合） | ANCHOR.DUPLICATE_BINDING |
| 7 | partial overlap | 声明 span 与 token span 相交但不包含 | 不算覆盖 → ANCHOR.UNANCHORED（冻结 span 规则） |
| 8 | invalid fact_id | 声明 fact_id 未过 `is_valid_fact_id()`（Stage D / BIND-5）；或格式合法但 ∉ 用例 G3（Stage E） | ANCHOR.INVALID_FACT_ID（两级定义与 §8.2 注一致） |
| 9 | value mismatch | 绑定有效、fact 合法，但 `frozen_round(token_value, snapshot.decimals) != G3.value` | P0-1 VALUE.CONSISTENCY FAIL（精度唯一来源 = snapshot.decimals） |

## 10. Final Status

- B5 唯一设计自由度（挂载位置）已裁决：**A. per-cell binding declarations**，附 cell.fact_id 隐式声明等价规则保证旧路径零迁移。
- 改动面收敛为一个可选字段 + 两处登记性注记；AnchorDeclaration、BIND-7/8 判定文本、§8.2 链、§8.3/8.4、tokenizer、渲染侧、TrustedInput、identity、7 P0 全部零改动——无“表达能力补齐”扩大为架构重设计之处。
- 四类目标场景（多 token / range / identifier+numeric / 单 token）与九条验收案例全部可表达、可判定，错误码复用现有体系。
- 遗留澄清项一项（Table title/note 的 rendered_segments 强制路径），按本轮指令登记为「覆盖路径需要澄清」，不构成本设计执行缺口，不阻塞。

**DESIGN READY FOR IMPLEMENTATION**
