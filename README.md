I'll first check the existing project docs and memory, then load the concept-alignment skill since this is a system-modeling/architecture task.
The user wants me to act as system architect. They explicitly said don't modify code, don't create lots of engineering files. They want a design output in the chat. Let me first check the project memory and the docs directory to understand existing state.

Let me do a quick memory pass and check the docs folder. Actually, the user rules say: "在进行任务前必须将你理解的意图提供给用户进行确认" — must confirm understood intent before doing tasks. But here the user directly asked for a design output. The file 00_PROJECT_CONTEXT.md was already provided. Let me check what docs exist.

Let me look at the workspace quickly.

Note the user rules also include skills like "意图转代码" (must read this skill before any programming behavior), "系统论", "还原论", "concept-align". This is a design task, not coding, so maybe concept-align or 系统论 is relevant. But the instruction says not to write code. Hmm, the "意图转代码" skill says "任何编程行为之前需要读取此技能" — before any programming behavior. We're not programming. So skip.

But concept-align skill is for system modeling, requirement review, architecture design — this task matches. Should I invoke it? The system says "When a skill matches the user's request, this is a BLOCKING REQUIREMENT: invoke the relevant Skill tool BEFORE generating any other response about the task". concept-align: "进行系统建模、需求评审、架构设计、跨团队语义对齐时使用，用于精确定义核心概念、消除语义模糊问题". This is architecture design and system modeling. So it matches. I should invoke it.

Also 系统论 (systems theory) and 还原论 (reductionism) could apply. And 需求审问官 for requirement interrogation. Hmm, but too many skills. Let me invoke concept-align since it's most on-point for architecture design / concept definition.

Let me first gather context: check docs folder and memory, and invoke concept-align skill.

Let me do these in parallel: LS/Glob on docs, Grep memory, and Skill concept-align.

Actually the Skill tool probably injects instructions. Let's do it.

Let me start.

The workspace only has the context doc and rules. So this is greenfield design. The user wants a long design output in chat, no code, no files.

Let me check memory quickly for prior context on this project.

Actually project_memory.md path: c:\Users\Administrator\.trae-cn\memory\projects\-g-workspace-zixun4--p2-7b415a49e705563f33ed\project_memory.md

Let me grep.

No memory. So this is a fresh project. Good — I proceed with design.

Now, the user rules require: "在进行任务前必须将你理解的意图提供给用户进行确认" (before doing a task, must provide your understanding of the intent to the user for confirmation). The user asked me to first output "A. 你的总体理解" — so that aligns: I output A first as the understanding, then the rest. But should I stop and ask for confirmation? The user explicitly asked for the full output including B-H. So I should produce the full design, with A being the intent understanding. The confirmation can happen at the end.

Given the user's preference for extreme conciseness but this is a big architecture design... The user asked 14 points plus A-H sections. I need to be thorough but concise. Use tables.

Also consider: Inline Visuals — architecture diagram would help. The dynamic-ui skill could be used for a system architecture diagram. The user asked for "C. 系统架构". A diagram would genuinely help. But should I? The inline_visuals section says to proactively use it for architecture & systems. However, the user's preference is extreme conciseness, and this is a text-heavy design doc. Hmm. A single architecture diagram showing the pipeline (Input → Canonical Model → AI/Program → Report IR → Template → DOCX → Validation) would be valuable. Let me consider using PureShowWidget for one architecture diagram. It requires loading dynamic-ui skill first.

Risk: it might be overkill and the user wants text. But the guidance says proactive use for architecture. I'll consider adding one diagram for the pipeline. Actually, given the length of the response already, one diagram may be nice but could add noise. The guidelines strongly push it. I'll do one compact pipeline diagram after section C.

Hmm, but I need to be careful about token budget. Generating SVG widget is a big deal. Let me weigh: The user is a Flutter/Go dev with a preference for concise, structured tables. They want A-H output. A diagram of the architecture would help them. I'll do it — one diagram only, after C.

Actually, let me reconsider. The instruction says "You should load the relevant dynamic-ui skill before generating output: diagram...". So I'd need to call Skill dynamic-ui first. That adds a round trip. I'll do it in parallel with composing? No, I need the skill content before generating. Let me just do it.

Let me decide: I'll produce the main text answer first (A, B, then C), then load dynamic-ui and produce a diagram. Actually the flow: I can output text and call the tool in the same message? Tool calls appear inline. Let me do: output A and B text, then call Skill dynamic-ui, then output the diagram + C.

Hmm, that splits output awkwardly. Simpler: just call dynamic-ui skill now in parallel with writing. But I can't write while tool calling in the same block; well I can — text before tool calls is shown.

Plan:
Message 1: brief intro + call Skill dynamic-ui (already did concept-align). Then after getting it, produce the whole answer with one diagram in C.

That's fine. But is the diagram worth a round trip? I think yes for the architecture section. Let me do it.

Actually wait — there's a risk that dynamic-ui skill returns a large module and I burn context. It's designed for this. OK.

Alternatively, use PureShowWidget which is deferred — need ToolSearch first. Let me just load the skill.

Let me now think hard about the actual design content. This is the core deliverable. Let me think deeply.

# Understanding of the project

真实问题：不是"AI写报告"，而是**把"非结构化的工程资料 + 专业人员的判断"变成"一份可复现、可验证、可审计的标准报告"的工业生产流水线**。

核心矛盾识别（用系统论/矛盾驱动）：
1. 不确定性 vs 确定性：LLM 是概率生成器，报告要求数值与结论确定、可复现、可追责。
2. 语义灵活 vs 版式刚性：报告语言需要灵活组织，但模板版式、表格、编号必须严格。
3. 生成能力 vs 验证能力：如果只有生成没有验证，系统可以用"看起来对"的文本骗过人工抽检。
4. 迭代速度 vs 回归成本：换模型/改Prompt/换模板，若无回归测试就不知道是否退化。

所以真正要解决的问题可以表述为：
**构建一个"报告生产函数" f(Inputs) = Report，并且这个函数是 (a) 分解为可独立测试的确定性/不确定性组件，(b) 输出可被机器验证，(c) 有 Ground Truth 数据集支撑回归。**

关键洞察：这个项目的产物不止是"报告生成器"，更是"报告质量的可验证标准"。Evaluation Dataset 的资产价值可能高于生成器本身。

# 系统边界

In scope:
- 单份报告的生成（从输入资料到 DOCX/PDF）
- 验证 + 回归

Out of scope (第一阶段):
- 报告审核审批流程/OA 集成
- 多人协同编辑
- 现场采集 APP
- 检测设备对接
- 报告归档/检索/知识库
- 规范全文检索的向量库
- 训练/微调模型

边界的一条重要原则：**系统不产生"事实"，只搬运、计算、组织事实。** 任何系统无法从输入中推导的数值，必须显式标记为 missing，不能由 LLM 编造。

# 输入层

输入类型与归属:
| 输入 | 形态 | 处理者 | 归一化目标 |
|---|---|---|---|
| 工程文字资料（委托单、任务书、工程概况） | docx/pdf/txt | AI抽取 + 人工校 | ProjectMeta / Narrative facts |
| 检测/监测数字数据 | excel/csv | 程序 | MeasurementRecord[] |
| 现场图片 | jpg/png | AI（多模态） | ImageEvidence[] |
| 标准规范 | pdf/docx | 程序分片 + AI理解 | RuleRef / Clause（按需检索） |
| Word报告模板 | docx | 程序解析 | TemplateSpec（占位符/区块/表格样式） |
| 历史真实报告 | docx/pdf | 半自动(逆抽取) | EvaluationCase(Ground Truth) |

输入层设计原则：
1. **每个输入都必须有 SourceRef**（文件 + 定位：页码/表格坐标/单元格/图片ID）。这是可追溯与可验证的基础。
2. **输入仪表化**：所有解析结果落成 JSON（中间文件），可人工检查，可进版本库，可当测试固件。
3. **人类校验点**：AI 抽取的结果必须有一个"人工确认"闸门（第一阶段可以是人工编辑 JSON，不是 UI）。

# Canonical Data Model

分层，避免大统一模型：
- L1 源数据层（RawSource）：文件、单元格、段落 —— 只读，保留原貌。
- L2 事实层（Facts）：从源抽取/计算得到的最小断言单元。带 provenance 与 confidence。
- L3 领域对象层（Domain Objects）：工程语义实体（项目、构件、检测项、测点、结论）。
- L4 报告层（Report IR）：面向"报告结构"的表示。

关键：**Facts 只存"事实"，不存"怎么表述"**。表述属于 Report IR。

领域对象（针对检测鉴定报告的通用骨架，先给一个可交换的骨架，具体按公司报告类型细化）：
- Project 工程项目
- Client 委托方
- Structure/Component 结构/构件（楼、层、轴、构件编号）
- InspectionItem 检测项（混凝土强度、钢筋保护层、裂缝、沉降、倾斜…）
- Measurement / MeasurementSet 测点与测值（含单位、方法、仪器）
- Criterion 判定依据（规范条文 + 限值）
- Evaluation 判定结果（合格/不合格/等级）
- Conclusion 结论（工程结论/子结论）
- Evidence 证据（图片、原始记录）

每个实体带：id, source_refs[], method(实测/计算/引用/推断), provenance(AI/human/program), confidence, status(filled/missing/conflict)。

冲突处理：同一事实多来源不一致 → 生成 Conflict 记录，不允许静默覆盖。

# Report IR

设计目标：与 Word 解耦、可验证、可 diff、可渲染。

结构分层：
- Document → Section（章节树） → Block → Inline
- Block 类型运动一个受控枚举：Heading, Paragraph, List, Table, Figure, Formula, PageBreak, Toc, Signature
- 关键：**Paragraph 不允许自由文本字符串**，而是 TextTemplate = 静态文本片段 + 变量引用（{fact:...}）。这是让 AI 不能乱写的关键机制。

即：
```
Paragraph(segments=[Lit("经检测，"), Ref(fact:concrete_strength.design_grade), Lit("，"), ...])
```
好处：
1. 渲染前可以校验所有 Ref 都能解析；
2. 数值由程序格式化（单位、小数位、有效数字）；
3. 可做"文本与数据的交叉验证"；
4. diff 更稳定（结构相同则 diff 只看变量值）。

Table 也类似：TableSpec { headers, rows: Cell(Ref|Lit|Computed) }，由程序从 MeasurementSet 生成，不由 LLM 生成。

IR 与模板解耦：
- IR 有 semantic 层（章节语义标签如 "conclusion", "test_method"）
- 模板有 mapping 层（semantic label → Word 样式/位置）
- 这样换模板不改 IR；换内容不改模板。

IR 需要一个 Schema（JSON Schema）+ 版本号 + 校验器。

# Agent / Skill 边界

原则：
- Agent = 编排者，有目标、有状态、有循环，但**不写报告正文**。
- Skill = 单一能力，有明确输入输出 schema，可单测，可替换。

Agent 列表（少而清晰）：
1. IntakeAgent（资料接入与归一化编排）—— 判断输入类型、调度解析 Skill、标记缺失
2. ExtractionAgent（语义抽取）—— 文字→Facts、图片→Evidence
3. ReasoningAgent（工程推理）—— 依据规范做判定、生成结论草稿（只产出"结论事实"与"结论要点"，不产出版式文本）
4. ReportComposerAgent（报告组装）—— 依据 IR 骨架，把 Facts 填进 IR，产出 Report IR
5. ValidatorAgent（验证编排）—— 调 Accuracy/Style Validator，生成问题清单与修复建议（可选：有限次自修复循环）

关键约束：**Agent 之间不通过自然语言传数据，只通过结构化数据（Facts / IR / IssueList）传数据。** 这是防止误差累积和幻觉传播的核心。

Skill 分类：
- 确定性 Skill（程序实现，无 LLM）：parse_xlsx, unit_convert, calc_statistics, apply_rule, render_docx, extract_docx_template, validate_ir_schema, diff_report, extract_text_from_pdf
- 不确定性 Skill（LLM 实现，有 schema 约束输出）：extract_fields_from_text, extract_fields_from_image, classify_document, draft_conclusion_points, summarize_evidence
- 每个 LLM Skill 必须：输入 schema、输出 schema（结构化）、few-shot、以及"低温+重试+校验"策略。

Skill 的必要属性：id, version, kind(deterministic|llm), input_schema, output_schema, cost, latency, test_cases。→ Skill 本身也要有回归测试。

# AI vs 程序 责任边界

一句话原则：
- 程序决定**"是什么、对不对"**；AI 决定**"像不像、怎么读得通"**。
- 凡是能用规则/公式/表格表达的处理，一律程序化；AI 只处理"从非结构化到结构化"的映射，以及"组织语言"。

红线（必须程序化）：
- 数值运算、单位换算、统计、限值判定
- 表格生成、编号、目录、页码
- 渲染、格式检查
- 任何出现在报告里的数字 → 必须来自 Facts，且 Traceable 到 SourceRef

红线（必须留给人）：
- 第一阶段所有 AI 抽取结果的人工确认（采样或全量）
- 最终结论的签发

LLM 允许做的事：把 Facts 组织成可读段落（在受控 TextTemplate 下）、图片描述、分类、字段抽取。

# 历史报告 → Evaluation Case

流程：
1. 选样：挑 10–20 份代表性历史报告（不同工程类型、不同结构、不同复杂度）
2. 逆抽取：从报告反向抽出 Facts（用 LLM Skill + 人工校验）。注意：**逆抽取出来的 Facts 是"报告声称的事实"，不是"真实世界的事实"** —— 这是重要的语义边界，需在概念表里说清。
3. 固化：生成 case 目录：inputs/ ground_truth/ meta.json（含来源、报告类型、关键字段、陷阱标注）
4. 建立断言：不是"整份文档 diff"，而是**分层断言**：
   - 数值断言：所有表格数值必须逐一相等（容差可配）
   - 结构断言：章节集合、表格数量、图片数量
   - 语义断言：关键结论的判定方向（合格/等级）一致（用 LLM 或规则判定等价）
   - 版式断言：样式名、字体、行距、页码、目录（用样式提取后比对）
5. 分级评测：P0 数值/结论（必须 100%），P1 结构，P2 文本相似度，P3 版式
6. 陷阱案例：故意构造缺失数据、冲突数据、超限值、单位异常，验证系统是否"正确地失败"（fail loudly, not silently）

关键：**"正确地失败"也是评测项。** 系统在不该编造的时候编造 = 最严重的错误。

# Accuracy Validator

分四层：
1. Schema/完整性验证：IR 是否合法、必填是否缺失、Ref 是否可解析
2. 一致性验证（程序）：
   - 数值与源数据一致（数值回查 Facts → SourceRef）
   - 表内合计 = 分项之和
   - 单位/量纲一致
   - 判定与限值逻辑一致（例如强度推定值 < 限值却写"合格" → error）
   - 交叉引用一致（正文引用的表号/图号/章节号存在）
3. 语义验证（AI 辅助，有明确判定标准）：
   - 结论与数据方向一致
   - 结论覆盖了所有检测项
   - 无"凭空出现的数字/规范号"
4. Ground Truth 对比（评测场景）：数值精确比对 + 结论方向比对

输出：IssueList，每条含 severity(error/warn/info)、location(IR path)、evidence、rule_id、suggested_fix。→ 可被 Regression Test 断言。

# Style Validator

分三层，越往后越"软"：
1. 硬版式（程序，可精确）：字体、字号、行距、缩进、对齐、页边距、页眉页脚、页码、标题层级样式名、表格边框、"表x-x"编号连续性、图注位置
2. 结构合规：章节顺序、必需要素（委托方、日期、检测依据、检测结论、签字栏）、编号规则
3. 语言风格（AI/统计）：术语一致性、人称、时态、模板句覆盖率、禁止口语化

用"样式指纹"方法：从模板/历史报告提取 style fingerprint（样式名→属性），生成报告提取同样指纹，做集合比对。这样可差分、可回归。

# Regression Test

三层：
1. 单元层：每个 Skill 有 fixture 输入/期望输出（因果可定位）
2. 用例层：Evaluation Case 全流程跑 → 输出 metrics（数值准确率、结论一致率、结构一致率、版式一致率）+ IssueList diff
3. 变化层：模型/Prompt/模板/程序版本变化时，比较"基线快照 vs 现在的快照"，输出回归报告（新增错误、修复、未变）

工程要求：
- 每次运行产出 run 快照（输入 hash、Skill 版本、模型名+版本、参数、输出 hash、metrics）
- 基线固定 → 任何变化可见
- 阈值 + 门禁：P0 不达 100% 视为失败

# LLM Adapter

接口：
```
LLMClient.generate(messages, schema=None, temperature, max_tokens) -> StructuredResult
```
- 统一"结构化输出"能力：不同模型对 JSON mode / function calling 支持不同 → Adapter 内部做转换与修复（抽取JSON、重试、schema 校验）
- 模型配置外置：model_profile（provider, model, base_url, params, context_limit, cost）
- 上层只依赖 "capability"（如 text_extraction / vision / long_context / json_mode），不依赖具体模型名 → 可按能力路由
- 记录 trace：prompt hash、token、耗时、原始响应 → 用于调试与回归
- 多模型 A/B：同一 Skill 用不同 model_profile 跑同一 case，比较 metrics

不要做的：不要一开始就做抽象"万能 provider 框架"，只需一个薄 Adapter + 一个配置文件 + 一个 trace 记录。

# MVP（第一阶段）

目标：**在 1 类报告上，打穿"输入→生成→验证→回归"的完整闭环，哪怕只覆盖 1 个章节。**

MVP 范围建议：
- 报告类型：选 1 类结构最简单、数据最表格化的（例如某类专项检测报告）
- 章节范围：选 1–2 个"数据驱动"章节（如"检测结果汇总表" + "检测结论"）
- 输入：Excel 数据 + 一份工程概况文本 + 一份模板
- 输出：DOCX
- 验证：Accuracy（数值+结论）+ Style（硬版式）
- 数据集：3–5 份历史报告转 case
- 不接 UI：CLI + JSON 中间产物 + 人工可编辑

实施顺序：
0. 概念表 + Report IR schema（先定义，后实现）→ 就是本文档
1. 输入解析最小集（xlsx → MeasurementSet；概况文本 → ProjectMeta）
2. Fact Layer + 确定性计算 Skill（几个）
3. IR Builder（纯程序，从 Facts 生成 IR，先不做 LLM）
4. Template Engine + DOCX 渲染
5. LLM Adapter + 1 个抽取 Skill（结构化输出校验）
6. Accuracy Validator（数值一致性）
7. 历史报告 → Evaluation Case（3–5 份）
8. 回归测试脚本 + 基线快照
9. Style Validator（硬版式）
10. 接入 LLM 做结论草稿 + 结论一致性校验

关键顺序心法：**先做"无 LLM 也能跑通"的骨架**（步骤 1-4）。这样一旦骨架通，LLM 只是替换某几个节点的实现，且立刻可被验证。这是本项目最重要的实施策略。

# 第一阶段绝对不要做

- 微服务/K8s/Redis/MQ
- 向量数据库 / RAG 全量规范检索（先做"人工指定条文 + 规则表"）
- Multi-Agent 自治协商
- 知识图谱
- 训练/微调模型
- 自研 Word 排版引擎（用 python-docx / docxtpl 等成熟库）
- 图形化编辑器/前端 UI（用 CLI + JSON）
- 全自动无人值守生成（必须有 DP 人工确认闸门）
- 一次性覆盖所有报告类型/所有章节
- 追求"端到端相似度 95%"这种单一指标（要分层指标）

Now also 关键设计决策 section B: list decisions with rationale:
D1 IR 优先于生成器
D2 Paragraph 不用自由文本，用 TextTemplate(Ref/Lit)
D3 数值只能来自 Facts，Facts 必须带 SourceRef
D4 Agent 间传结构化数据不传自然语言
D5 历史报告是 Ground Truth 与测试用例，不是训练数据
D6 逆抽取得到的是"报告声称的事实"，须区分于"客观事实"
D7 先骨架后 LLM（无LLM可跑通）
D8 分层评测 + P0 门禁
D9 "正确地失败"是一等公民（missing/conflict 显式化，禁止静默补全）
D10 LLM Adapter 按能力而非模型名编程
D11 一切中间产物 JSON 化落盘（可检查、可 diff、可当固件）
D12 概念先行：先术语表再建模型

Also 概念表 (concept-align output) — I should include a small table of critical concepts with 内涵/外延/边界. Good idea, integrates the skill. Key ambiguous concepts: 报告/Report vs Document, 事实 Fact, 准确 Accuracy vs 正确 Correctness, 样式 Style vs 版式 Layout, 验证 Validation vs 校验 Verification, 模板 Template, 案例 Case, 生成 vs 渲染.

I'll include a compact concept table (maybe 5-6 concepts, most ambiguity-prone): 事实(Fact), 准确性(Accuracy), 一致性(Consistency), 样式(Style), 报告模板(Template), 评估用例(Evaluation Case), 验证(Validation).

Let me be careful about length. The user asked 14 items + 8 sections. It'll be long but must be dense. Use tables heavily.

Also need "A. 你的总体理解" first including the intent confirmation per user rules.

Let me also include one diagram. I'll load dynamic-ui.

Actually — thinking about cost/benefit again: the response is already going to be very long. Adding a diagram is good but requires the skill load. I'll do it — it's explicitly encouraged for architecture. Let me load the skill now.

Wait, but there's a concern: skill output may be large. Let's just do it once.

Let me call Skill dynamic-ui.

I need to read the scene file for architecture-and-flow and the manifest, plus tokens. Let me read the architecture scene and a relevant template.

Actually to keep it efficient: read scenes/architecture-and-flow.md and templates/manifest.json, and tokens/visual-tokens.md.

Let me read those.

以系统架构设计师身份，先给出总体理解与关键决策。

---

## A. 总体理解

### 你真正要解决的问题（我的重述）

不是"AI 写报告"，而是建立一个**报告生产函数** `f(输入资料) → 成品报告`，并且这个函数满足四个性质：

| 性质 | 含义 | 你原文里的对应诉求 |
|---|---|---|
| 可分解 | 函数由可独立测试的确定性组件与不确定性组件拼装 | 问题 1、3、5 |
| 可验证 | 输出可被机器判定对错，而非靠人眼抽检 | 问题 6、7 |
| 可回归 | 换模型/换模板/改实现后能证明"没变差" | 问题 8、9 |
| 可替换 | LLM 只是某个节点的一种实现 | 结尾"换模型不重构" |

### 三个我认为被隐含、但必须显式化的判断

1. **本系统的第一等公民是"中间表示 + 验证体系"，不是 LLM。** LLM 在架构里是一个可替换的叶子节点；Report IR 与 Validator 才是骨架。因此设计顺序必须是先定 IR 和验证，再选模型。
2. **历史报告的价值不是训练数据，而是"规格 + 标准答案"。** 它同时定义了两端：逆抽出"输入应该长什么样"（输入规格），以及"输出应该长什么样"（结论口径、章节结构、版式基准）。所以在概念上必须区分**"报告声称的事实"**与**"客观事实"**——逆抽取只能得到前者。
3. **最大风险不是"AI 写不好"，而是"AI 写得太顺，错误不可见"。** 一份数值错误但行文流畅的报告，比一份明显残缺的报告危险得多。所以架构上必须有一条铁律：**系统不得静默补全**，缺数据就显式缺项。**"正确地失败"是验收指标。**

### 系统边界

| 在范围内 | 在范围外（当前阶段） |
|---|---|
| 单份报告的端到端生成（资料 → DOCX/PDF） | 送审/签章/归档/OA 流程集成 |
| 数据归一化、计算、判定 | 现场采集 APP、检测仪器对接 |
| 报告的准确性与版式校验 | 多人协同编辑、在线编辑器 |
| 历史报告 → 评测用例 → 回归测试 | 报告全文检索、企业知识库 |
| 单一报告类型的最小闭环 | 模型训练/微调 |

边界内的一条硬约束：**系统只搬运、计算、组织事实，不产生事实。** 任何无法从输入推出的数字，必须落成 `missing`，而不是由 LLM 生成。

### 需要你确认的三个前提（会影响后续设计严格度）

| # | 问题 | 影响 |
|---|---|---|
| 1 | 报告类型是否可收敛为**少数几类结构稳定的报表**？ | 决定 Canonical Model 能否设计成有限领域模型，还是退化成万能文档树 |
| 2 | 是否存在**必须人判断**的定性环节（现场描述、责任认定）？ | 决定自动化天花板与"人工闸门"的位置 |
| 3 | 版式验证要求是**逐样式对齐历史报告**，还是**符合模板规范即可**？ | 决定 Style Validator 是做"样式指纹比对"还是"规则检查" |

我的默认假设：1 = 是（先收敛 1–2 类）；2 = 有，且必须保留人工闸门；3 = 符合模板规范 + 关键项对齐。

---

## B. 关键设计决策

### B0. 先精确化 4 个最容易导致返工的概念

| 概念 | 内涵 | 边界（不是什么） |
|---|---|---|
| **事实 Fact** | 带来源引用、可追溯到原始资料定位的最小断言单元 | 不是"章节文本"，不是"结论段落" |
| **准确性 Accuracy** | 报告中的数据/结论与输入数据/判定规则**逻辑一致** | 不是"文风像不像真报告"（那是风格） |
| **一致性 Consistency** | 同一事实在报告内部各处表述不矛盾 | 不是"与历史报告逐字相同" |
| **验证 Validation** | 用可执行规则判定输出是否合法、可签发 | 不是"人工看一眼" |

两个必须钉死的区分：**"报告声称的事实" ≠ "客观事实"**；**"生成能力" ≠ "验证能力"**（二者必须由不同实现提供，否则等于自己判自己的卷子）。

### B1. 设计决策清单

| # | 决策 | 理由 |
|---|---|---|
| D1 | **Report IR 先行**：先定义 IR Schema，再写生成器 | IR 是解耦点，后定义会导致渲染器反向污染业务逻辑 |
| D2 | **IR 中段落不允许自由文本**，只能是 `Lit(静态字面量) + Ref(事实引用)` 的组合 | 这是让 LLM 无法"随口编数字"的结构性机制，比 Prompt 约束可靠 |
| D3 | **报告里出现的每个数字必须来自 Fact，且 Fact 必须带 SourceRef** | 可追溯是可验证的前提 |
| D4 | **Agent 之间只传结构化数据**（Facts / IR / IssueList），不传自然语言 | 阻断幻觉在 Agent 链路上滚雪球 |
| D5 | **历史报告定位为 Ground Truth + 用例**，不是训练数据 | 训练会掩盖错误，评测能暴露错误 |
| D6 | **"正确地失败"一等公民**：缺数据/冲突必须显式化为 `missing` / `conflict` | 静默补全是本系统最严重的缺陷类型 |
| D7 | **先做"无 LLM 也能端到端跑通"的骨架**，再逐个节点替换为 LLM | 让 LLM 的引入变成可测量的增量，而不是不可归因的跳变 |
| D8 | **分层指标 + P0 门禁**，不追求单一"相似度 95%" | 数值错误不能被文本相似度掩盖 |
| D9 | **LLM Adapter 按"能力"编程**（结构化输出/视觉/长上下文），不按模型名 | 换模型不改上层 |
| D10 | **一切中间产物落盘为 JSON** | 可人工检查、可 diff、可直接当测试固件 |
| D11 | **模板与 IR 用语义标签映射**，IR 不感知 Word 样式 | 换模板不改 IR |
| D12 | **第一阶段用退出码 + CLI + JSON**，不做 UI | 验证体系没通之前，UI 是纯负债 |

---

## C. 系统架构

分层与数据流（每层之间只传结构化数据，跨层边界是唯一的解耦点）：

| 层 | 职责 | 实现属性 |
|---|---|---|
| Input Layer | 接收 6 类输入，产出带定位信息的 RawSource | 程序 |
| Normalization | 解析 Excel/CSV/文本/图片，产出候选事实 | 程序为主，AI 为辅 |
| Canonical Data Model | 事实层：唯一可信数据源，带 provenance / status | 纯数据 + Schema |
| Reasoning Layer | 计算、单位换算、规则判定 → Evaluation | 程序 |
| AI Layer | 语义抽取、图片理解、语言组织（只产出结构化结果） | LLM（可替换） |
| Report IR | 与模板解耦的报告表示 | 纯数据 + Schema |
| Template Engine / Renderer | IR → DOCX → PDF | 程序 |
| Validation | Accuracy / Style / Regression | 程序为主，AI 辅助语义判定 |

下面是这条流水线的结构图。核心视觉是 **Report IR 作为唯一的收敛点**，以及**验证结果回流驱动迭代**的闭环。
Now let me check the manifest for architecture-elements template. Actually, I'll just build the node-flow fallback myself with proper tokens. But the skill says prefer ready material. Let me check manifest quickly to see if `architecture-elements` is ready and read its template.md.

To save time, I'll read manifest.json and templates/architecture-elements/template.md.

Actually the diagram I want: a horizontal pipeline with 5 stages: 输入层 → 规范化/Canonical Model → AI+程序双通道 → Report IR → 模板引擎/渲染 → 验证(回流). That's 6 nodes + a feedback loop. Plus a focal point (Report IR as the spine).

Maybe simpler: two lanes showing 程序确定性通道 and AI不确定性通道 converging on Report IR. Hmm, complexity budget: <=6 ungrouped nodes, <=8 connectors.

Design: 
Row 1 (top): 输入层 (6 source chips grouped in one container)
Middle: Canonical Data Model (node) with two lanes: 确定性程序 / AI Skill
Then: Report IR (focal, brand)
Then: Template Engine → DOCX
Bottom: 验证体系 with feedback arrow back to IR/generation.

That's a lot. Let me simplify to a vertical/horizontal flow of 5 primary steps + a feedback loop:

[输入层 Inputs] → [Canonical Data Model] → {程序 Skills | AI Skills} → [Report IR] → [Template+Renderer] → [DOCX/PDF]
and [Validator] with feedback arrow from DOCX back to Report IR, plus [Evaluation Case] feeding Validator.

Hmm the "双通道" is important. Let me do:

Panel 1: horizontal flow, 6 columns? "Horizontal tier: max 4 boxes". So I need to fit 5 steps → group.

Better plan: 3 rows vertical skeleton (grouped architecture).
Row A: 输入层 container with 4 chips (文档 / 表格 / 图片 / 模板+规范)
Row B: 两个 lane 节点：确定性程序通道、AI Skill 通道 — both converge into Canonical Data Model... actually order: inputs → normalization → CDF → then two channels → IR.

Let me lay out:

viewBox 0 0 720 H.

Row 1 (y=40..96): 4 input chips: 文字资料, Excel/CSV, 现场图片, Word模板 — actually 4 chips each width ~150, gap 24: 4*150+3*24 = 600+72=672, safe area 40 each side → 40..680 = 640 width. Too tight. Use 4 chips of width 145 gap 20: 580+60=640. OK width 145 each: x=40, 205, 370, 535 → ends 680. gap = 20. Rule says gap>=32 for unrelated nodes. Hmm.

Alternative: 3 chips: 非结构化资料(文字/图片), 结构化数据(Excel/CSV), 模板与规范. Width 200, gap 24: 3*200+2*24=648 → x=40,264,488 ends 688. gap 24 <32. Use width 192, gap 36: 3*192+72=648, x=40,268,496, ends 688. gap 36 ok. Wait 40+192=232, +36=268, +192=460... let me recompute: x1=40..232; x2=268..460; x3=496..688. gaps=36. Good. Right safe edge 688, viewBox 720, so right margin 32. Slightly less than 40. Adjust: width 190, gap 34: total = 570+68=638; x=40,264,488 ends 678; margins 40/42. 

Hmm, maybe two rows for inputs is fine:
Row1: 输入层 boundary container holding 3 chips. That adds nesting.

Let me simplify the whole diagram: single vertical flow with 5 stages, focal = Report IR, plus a right-side feedback loop.

Stages (each a full-width-ish node? no, that's a list, not helpful).

OK, think about what's most informative and compact: The "双通道收敛到 Report IR" is the core architectural insight. Let me draw that:

Layout (viewBox 0 0 720 H):
- Top: 输入层 as ONE node (capsule) centered, label "输入层 · 多源资料" 
- Arrow down to two parallel lane nodes side by side:
  - Left: "程序通道 · 确定性" subtitle "解析/计算/规则"
  - Right: "AI 通道 · 不确定性" subtitle "抽取/理解/组织"
- Both converge into "Canonical Data Model" (single node center) — wait, actually canonical model comes BEFORE the two channels (channels consume facts). Hmm: Input → normalize → Canonical Facts → (程序计算 / AI 抽取+组织) → Report IR.

Actually cleaner: 
Row1: 输入层 (one node)
Row2: 归一化 → Canonical Data Model (one node, neutral, wide) — with a boundary showing 4 sub-layers? too much.
Row3: 双通道 side by side
Row4: Report IR (brand focal)
Row5: 模板引擎 + 渲染 → DOCX/PDF
Right/bottom: 验证体系 (Accuracy + Style) + 评测用例集, feedback arrow back to Row4.

Nodes: 输入层, Canonical Data Model, 程序Skill, AI Skill, Report IR, 模板渲染, DOCX, 验证器, Evaluation Case = 9. Over budget (<=6 ungrouped). Need grouping into containers:
- Container "生成流水线" holding: 输入层, Canonical Model, two channels, Report IR, 渲染 → 6 inner
- Outside: 验证体系 (1) with feedback
That's 7. Still slightly over but containers reduce it: container counts as one level. Let me count as: inside one boundary there are 6 nodes; plus Validator container with 2 chips. Total visual nodes ~8. The hard gate says custom diagrams <=6 ungrouped modules and <=8 connectors. With one grouping boundary, 6 inner nodes + 1 external = 7. Borderline.

Let me cut: merge Renderer+DOCX into one node ("模板引擎 → DOCX"). Then inner nodes: 输入层, Canonical Model, 程序Skill, AI Skill, Report IR, 渲染输出 = 6. Plus external 验证 = 7 with one boundary. Acceptable-ish. Actually I could drop the 输入层 node and use the boundary title.

Alternative simpler approach: make the pipeline horizontal 4 boxes and a second panel below for the verification loop. Two panels = more complexity.

Decision: I'll do a vertical grouped architecture with one boundary "报告生成流水线" containing 6 rows-ish, but arranged as: input node, single node, twin lanes, focal node, output node. Then below the boundary, a "验证体系" row with 3 chips (Accuracy/Style/回归) and a feedback dashed connector going back up. Total connectors: 输入→Model (1), Model→two lanes (2), two lanes→IR (2), IR→渲染 (1), 渲染→验证 (1), 验证→IR feedback (1) = 8. Exactly at limit. Hmm, and feedback connector routing will be long. 

Maybe drop the feedback connector and instead put a small dashed connector from 验证 back up along the right side. That's the focal "闭环" — important! The user's core is the closed loop. I could keep the feedback arrow and drop... the 输入层 node's separate existence. Let me: boundary title "生成流水线" and inner nodes: Canonical Model, 程序通道, AI通道, Report IR, 渲染. = 5. Inputs go outside as 3 chips above (external inputs/outputs placed outside the container, per rules). So:

- Top row: 3 input chips (external, above boundary)
- Boundary container row 2: Canonical Data Model (full-width-ish center), then twin lanes, then Report IR (focal), then 渲染
Wait but inputs → Canonical Model: 3 arrows or 1 merged arrow with label? Use 1 merged trunk: chips → a horizontal bus line → one arrow down into Canonical Model. That's cleaner: 3 short lines to bus + 1 arrow = could count as 4 connectors. Hmm.

Simplify inputs to a single pill: "多源输入 · 文档/表格/图片/模板". Then it's just 1 node + 1 arrow. Loses detail but I explain in text. Good — that keeps budget.

Final layout plan (viewBox 0 0 720 H):

Safe area x: 40..680.

Row A (y=32): pill node centered, width 260, height 40: x=230..490, y=32..72. Label "多源输入（文档·表格·图片·模板）" — 中文 15 chars, at 14px 中文 ~14px/char = 210px + padding. Let me widen: width 300 → x=210..510, y=32..72. Text centered 14px. Chinese char width ~14px at 14px font. "多源输入 · 文档 表格 图片 模板" ≈ 16 chars ≈ 224px. fits in 300 with padding.

Hmm, actually simpler label: "多源原始资料" (6 chars ≈ 84px) with subtitle "文档 / 表格 / 图片 / 模板" at 12px (~ 13 chars * 7 = 91px... Chinese at 12px ≈ 12px/char → 13 chars ≈ 156px). Node width = max(6*14, 15*12)+24 = max(84,180)+24 = 204. Let's use width 240: x=240..480, height 56 (2-line node), y=32..88.

Arrow A→B: from (360, 88) to (360, 128). gap 40.

Row B (y=128..184, h=56): node "Canonical Data Model" + subtitle "归一化事实层". Width: title 20 latin chars *8=160; subtitle Chinese "归一化事实层" 6*12=72 → max 160+24=184. Use width 280 centered: x=220..500. Label centered. Actually include both 中文 and English: title "Canonical Data Model" (subtitle 归一化事实层 / Facts+溯源). 

Row C: two lanes at y=224..296 (h=72, 2-line + subtitle = need >=56; use 72):
 - Left lane: x=120..340 (w=220): title "程序通道 · 确定性", subtitle "解析/计算/规则/渲染"
 - Right lane: x=380..600 (w=220): title "AI 通道 · 不确定性", subtitle "抽取/理解/组织语言"
 gap = 40. Good.
 Arrows from B (360,184) to left lane top (230,224) and right lane top (490,224): diagonal. Or a bus: down from B to y=204, then horizontal to x=230 and x=490, then down. Diagonal is allowed and simpler: two diagonal arrows. Both from (360,184) → (230,224) and (360,184) → (490,224). With markers orienting auto. Fine.

Row D (y=336..392): Report IR — focal, brand class. Width: title "Report IR" (9*8=72) subtitle "报告中间表示 · 与模板解耦" (15*12=180) → width 204+24=228 → use 300: x=210..510, h=56. y=336..392.
 Arrows: left lane (230,296) → (300,336)? and right lane (490,296) → (420,336). Converging. Good.

Row E (y=432..488): "模板引擎 + 渲染" subtitle "DOCX / PDF" width 240: x=240..480. 
 Arrow D→E: (360,392)→(360,432).

Then boundary container around B,C,D,E: x=60..660, y=108..508. Title "生成流水线" placed at top-left inside. But then Row A arrow crosses the boundary — that's fine (connector into container).

Wait, boundary x=60..660 but lanes at x=120..340 and 380..600 — inside with padding 60 left, 60 right. Good. Report IR x=210..510 inside. Good.

Row F (below boundary): 验证体系 chips at y=548..604 (h=56):
 - "Accuracy Validator" (数值/结论一致性) 
 - "Style Validator" (版式/结构)
 - "Regression Test" (基线快照)
 3 chips: width 190 each, gap 34: x=40..230, 264..454, 488..678. Hmm that's the full width, no room for container. Wrap them in container x=40..680, y=524..628 with title "验证体系". Inner chips y=548..604 h=56, x=64.., need padding 24 from container edge: inner area 64..656 = 592. 3 chips of 176 + 2 gaps of 32 = 528+64=592. So x=64..240, 272..448, 480..656. Chips width 176: title 18 chars latin*8=144 +24=168 ok; subtitle Chinese needs shorter: "数值与结论" (5*12=60). OK.

Feedback connector: from 验证体系 container left edge (40, 576) → route left? There's no space left of x=40. Alternative: from container right edge (680,576) go right → no space (720). 

Better: put feedback arrow on the LEFT side going up from the 验证 container top-left up to Report IR left edge... but the boundary occupies 60..660. Hmm, conflicting.

Alternative layout: make the generation boundary narrower (x=140..580) so there's a left corridor at x~90 for the feedback line. Let's redo:

Boundary "生成流水线": x=140..580, y=108..508. Inner padding 20 → inner 160..560 = 400 width.
 - B: Canonical Data Model x=200..520 (w=320), y=128..184
 - C lanes: left x=170..350 (w=180), right x=370..550 (w=180), gap 20. Hmm gap should be ≥32 for unrelated, ≥60 for flow steps. These are peer lanes, ≥32. 180+180+32=392 ≤ 400. x=164..344, 376..556. Then padding from boundary: 164-140=24 ok. 
 - D: Report IR x=190..530 (w=340), y=336..392
 - E: 渲染 x=230..490 (w=260), y=432..488
 Wait boundary bottom 508, E bottom 488 → 20 padding. ok.

Now the feedback corridor: from 验证体系 container (x=140..580 if aligned, at y=548..628) — left edge x=140, y=588. Arrow goes left to x=90, then up to y=364 (Report IR center y=364), then right to x=190 (Report IR left edge). This corridor x=90 passes left of boundary x=140. Vertical line at x=90 from 588 to 364. Does it cross anything? No, everything is x>=140 except nothing. Bottom part: horizontal from (140,588) to (90,588) — at y=588, the 验证 container spans 140..580, so going left from its edge is fine. Good.
 And a label on the feedback line: "问题回流" — put capsule at (100, 470)? That'd overlap the line. Put capsule centered at x=... the vertical line is at x=90. A capsule to its right at x=95..? that would be inside boundary region x>=140? no, 95..135 is free. Actually a capsule there might collide with boundary x=140. Capsule width for "验证回流" 4 Chinese chars *12 = 48 + padding 16 = 64 → x from 58 to 122 — that would extend left to 58, fine (safe area x>=40). y=476..500. Hmm the vertical line is at x=90 which passes through the capsule. Bad — rule: capsule must not sit on top of a line. So offset capsule to the left of the line: capsule x=20..84 → too close to edge (safe area 40). 

Alternative: put the label vertically? Or simply omit the label — "obvious relationships may be omitted"? The feedback arrow is not obvious. Hmm.

Alternative: route the feedback on the left but put the label as a rotated text? Avoid.

Simpler: move feedback corridor to x=95, and put the capsule ABOVE the horizontal segment near the 验证 container: e.g., capsule at (95..175, y=528..552)? That would collide with boundary bottom (508) and 验证 container top (548). y=528..552 overlaps 548. Use y=520..544: gap to boundary bottom 508 is 12, gap to 验证 top 548 is 4. Tight. 

Alternative cleaner: put the feedback label as part of the 验证 container title: "验证体系 · 问题回流". That explains it without a floating capsule. 

So: 验证体系 container title → "验证体系（问题回流・驱动迭代）". Then the dashed feedback arrow needs no label. 

Now heights: 
- Row A pill: y=32..88 (h=56, 2 lines) centered x=240..480.
- Boundary y=108..508.
- 验证 container y=548..628.
- Total H = 628 + 32 padding = 660.

Let me double check arrows:
 A→B: (360,88) → (360,128). ✓ (boundary top 108 crossed, fine)
 B→C-left: (280,184) → (254,224)? Let me place lanes y=224..296. Left lane x=164..344 center 254. Right lane x=376..556 center 466. Arrows: (340,184)→(254,224) and (380,184)→(466,224). B's box is x=200..520, bottom y=184. Start points on its bottom edge. Good.
 C→D: left lane bottom (254,296) → (300,336); right lane bottom (466,296) → (420,336). D box x=190..530, top y=336. ✓
 D→E: (360,392)→(360,432). ✓
 E→验证: (360,488)→(360,548). Crosses boundary bottom 508. ✓
 Feedback: (140,588) → (95,588) → (95,364) → (190,364). Dashed, brand color? Use brand for the loop (focal). Per rules: use --brand for the main path/focal relationship. The feedback loop is the focal. Actually the focal point should be ONE. I'd say focal = Report IR (brand node). Then feedback should be neutral dashed or chart-series. Hmm, but the closed loop is the key message. Rules: "Max 1-2 meaning colors per compact diagram". I can use c-brand for Report IR node and brand dashed for the feedback loop — both brand = one meaning color. Good, consistent: "brand = 核心闭环".

 Connector count: 1(A→B) + 2(B→lanes) + 2(lanes→D) + 1(D→E) + 1(E→verif) + 3 segments of feedback (count as 1 logical connector but 3 paths; I'll draw it as one polyline path with marker at end). Total ~8. OK.

Text width checks:
- Node A: title "多源原始资料" 6 chars *14 = 84; subtitle "文档 / 表格 / 图片 / 模板" — mixed: Chinese 8 chars*12=96 + latin/slashes ~12 chars*7=84 → 180. width 240 with 24 padding → available 216. ✓
- Node B: title "Canonical Data Model" 20 chars *8 = 160; subtitle "归一化事实层 · 全量溯源 SourceRef" — Chinese 12*12=144 + latin "SourceRef" 9*7=63 → 207. Node w=320, available 296. ✓
- Lane left: title "程序通道 · 确定性" 9 chars*14=126; subtitle "解析 / 计算 / 规则" ≈ Chinese 6*12=72 + ~8*7=56 → 128. Node w=180, available 156. 126 ≤156 ✓, 128 ≤156 ✓ (tight but ok).
- Lane right: title "AI 通道 · 不确定性" — Chinese 7*14 + "AI" 2*8 = 98+16=114 ✓; subtitle "抽取 / 理解 / 组织语言" Chinese 8*12=96 + 6*7=42 = 138 ≤156 ✓.
- Node D: title "Report IR" 9*8=72; subtitle "报告中间表示 · 与 Word 模板解耦" Chinese 11*12=132 + "Word" 4*7=28 → 160. w=340 → available 316 ✓.
- Node E: title "模板引擎 · 渲染" 7*14=98; subtitle "DOCX / PDF" 11*7=77. w=260 ✓.
- 验证 chips w=176, available 152: title "Accuracy Validator" 18*8=144 ✓; subtitle "数值与结论一致性" 8*12=96 ✓. "Style Validator" 15*8=120; subtitle "版式与结构" 60 ✓. "Regression Test" 15*8=120; subtitle "基线与回归" 60 ✓.
 chips: 3 * 176 + 2*32 = 592 within container inner 140+24=164 .. 580-24=556 = 392! Wait container x=140..580 → inner width 440-48=392. I computed 592 earlier for a wider container. Need to redo.

 验证 container: let me make it span the full width x=40..680 → inner = 640 - 48 = 592. ✓ 3 chips of 176 with gap 32 = 592. x: 64..240, 272..448, 480..656. ✓
 But then the feedback corridor at x=95 would start from container left edge x=40, and the horizontal segment from (40,588) to (95,588) — going right, that's fine, no overlap since container starts at 40 and corridor is at x=95 (inside container's x range but above/below? no, y=588 is inside container y=548..628!). Conflict: the corridor at x=95 vertical from 588 up to 364 would pass through the 验证 container's top edge at y=548, x=95 — which is inside the container's x range (40..680). That means the vertical line crosses the container boundary. Bad.

 Fix: start the feedback from the container's TOP edge at x=95: (95,548) → (95,364) → (190,364). Then the line starts at the container top border and goes up. No crossing. But x=95 at y=548 is within the container's top edge — visually it emanates from the container. Acceptable. But is it inside the boundary container (x=140..580)? No, 95 < 140. ✓. And the horizontal segment at y=364 from x=95 to x=190: does it cross the generation boundary x=140 at y=364? YES — the boundary spans x=140..580, y=108..508, and y=364 is within. So the horizontal line enters the boundary and hits Report IR at x=190. Crossing the boundary rectangle edge is normal for an input arrow, but here it's a feedback entering — that's fine and expected (it's a connector entering the container to reach Report IR). Acceptable and common. But rule says "No connector passes through unrelated nodes" — it doesn't. OK.

Hmm, but the arrow tip at x=190 is Report IR's left edge, and Report IR box is x=190..530, center y=364. ✓ Good.

Actually, cleaner: make the corridor at x=100 and have it enter from the left. Fine.

Alternatively I can avoid the boundary-crossing by ending the feedback at the container edge... no, keep it.

Let me reconsider whether the boundary container is worth it. It adds visual grouping ("生成流水线" vs "验证体系"). Yes, keep.

Now H: 验证 container bottom 628 + 32 = 660.

Colors: 
- boundary container: dashed border, neutral.
- 验证 container: dashed border neutral.
- Node A: c-neutral
- Node B: c-neutral
- lanes: c-neutral
- Node D: c-brand (focal)
- Node E: c-neutral
- chips: c-neutral
- arrows: neutral --text-muted; feedback: --brand dashed.

Legend: >2 connector styles? We have 1 neutral solid + 1 brand dashed = 2 styles, no legend needed. Block styles: neutral + brand = 2. No legend needed. 

Now the card wrapper: a surface card with the title? "Chart-only" rule applies to charts. For diagrams, a card title is fine. Keep a short title "报告生成与验证闭环" and the SVG. The content rules say one focal point. Good.

Now the HTML: root div with data-dynamic-ui-widget and data-template="node-flow".

Let me write the SVG carefully.

Node A (pill): rect x=240 y=32 w=240 h=56 rx=28 (pill). Text title at y=54 centered x=360 anchor middle; subtitle at y=74.

Wait, boundaries: y=32..88 → title baseline y=56, subtitle y=74. ok.

Boundary: rect x=140 y=108 w=440 h=400 rx=12 dashed, fill none, stroke border.
Boundary label: text at x=160 y=132 "生成流水线" 14px medium. But node B top is y=128 — conflict! Move B down: B y=148..204. Then arrows adjust.

Redo rows:
- Boundary y=108..528 (h=420). Bottom padding: E bottom 488 → 40. Fine, let's recompute all.

New plan:
- A pill: y=32..88
- Boundary: y=108..528
- B: y=144..200 (h=56) → arrow A→B from (360,88) to (360,144); boundary label at y=132 sits above B (B top 144), label baseline 132, so label occupies ~124..136. Gap to B: 8. A bit tight; move boundary label into top-left at y=136 and B top 152.
  Let me set boundary y=112..528. Label baseline y=140 ("生成流水线" at x=160). B: y=156..212.
  A→B arrow: (360,88) → (360,156). ✓ gap 68.
- Lanes: y=252..324 (h=72). B bottom 212 → gap 40. Arrow start (300,212) → (254,252); (420,212)→(466,252).
  Lanes x: left 164..344 (w=180), right 376..556 (w=180).
- Report IR (D): y=368..424 (h=56). x=190..530. Arrows from lanes bottom (254,324)→(300,368) and (466,324)→(420,368).
- E: y=464..520 (h=56). x=230..490. Arrow (360,424)→(360,464).
  Boundary bottom 528 → E bottom 520, padding 8. Too tight. Make boundary y=112..544, E y=464..520 → padding 24. ✓
- 验证 container: y=584..664 (h=80). E→验证 arrow (360,520)→(360,584). ✓
  chips inside: y=608..664? Need padding: container y=584..664, inner top after title... The container title "验证体系 · 问题回流驱动迭代" — where? Put it as a title row inside the top: container y=584..672, title baseline y=608 at x=64, chips y=620..676? Let me restructure: container y=584..680 (h=96); title at x=64, baseline y=612 (12px caption? no, use 14px medium); chips y=624..680 → bottom padding 0. Bad.

Simplify: drop the container for 验证 and just place 3 chips directly, with a small caption label above them? Then the feedback arrow starts from... chip middle. Hmm, but then "验证体系" grouping is lost.

Alternative: put the 验证 container title to the LEFT outside? No.

Let me make container y=584..688 (h=104): title baseline y=612 (14px, x=64), chips y=628..684 (h=56) → bottom padding 4. Still tight. 

Use chips h=56, container h=112: y=584..696; title baseline y=610; chips y=624..680; bottom padding 16. Top: 584→ title text top ~598, 14 gap. OK-ish.

Hmm, this is getting fussy. Simpler approach: skip the container; put a single caption line "验证体系" above the three chips, and start the feedback dashed line from above the left chip. Actually I'll keep it clean:

- Caption text "验证体系" at x=40, baseline y=596, 14px medium (acts as group label).
- 3 chips: y=612..668 (h=56).
- Feedback: from (40, 640) — the left chip's left edge? Better from the left chip's left edge x=40 → but that's the safe area edge. Go from the caption row instead: start at (95, 612)? That's the top edge of chip1 (chip1 x=64..240). Emanating from its top edge upward is fine: (95,612) → (95,364) → (190,364). Wait but then the arrow visually comes out of chip 1 (Accuracy Validator) which is semantically OK ("验证发现问题回流").

Hmm, but a line from chip1's top edge at x=95 — chip1 spans 64..240, so x=95 is on its top edge. Fine.

Actually cleaner: start the feedback from the caption "验证体系" region... no. Let me just do: feedback path from (95, 612) up to (95, 364) then right to (190, 364). And label the loop with a small caption text "问题回流" placed at x=104, baseline y=470 (to the right of the vertical line at x=95, which is left of boundary x=140 → text 104..152 would cross x=140 boundary edge!). Text width for 4 Chinese chars at 12px = 48 → 104..152 crosses the boundary line at 140. Bad.

Put the loop label rotated? No. Put it below at y=560 near the horizontal? there is no horizontal.

Option: omit the label and rely on color (brand) + the caption. Since brand = focal/loop meaning, and I explain in text. But rule: "Do not rely on color alone". Hmm. The feedback direction is shown by the arrowhead. I think it's acceptable: the arrow terminates at Report IR with an arrowhead, direction is clear. I'll add the label into the response text instead.

Alternatively widen the left corridor: shift boundary to x=170..580 (w=410) and corridor at x=90 with label to the left of the line at x=20..84 — but safe area is ≥40. Label at x=44..92 would overlap the line at 90. 

Final: make corridor at x=110, boundary x=170..590. Label capsule to the left of the line: capsule x=44..104 (w=60), y=... the label needs to not touch the line at 110 — gap 6, ok but capsule must not sit ON the line: it ends at 104, line at 110. Gap 6 units. Rules say label capsule must be ≥8 units from the stroke. Make capsule x=40..100, line at 110, gap 10. ✓ Capsule w=60 for "问题回流" (4*12=48 + padding 12 = 60). ✓ Height 24: y=458..482, text baseline y=475, text centered x=70.

Hmm, but a capsule between the boundary (x=170) and the safe edge (40) — plenty of room actually. Let me recompute inner nodes for a boundary of x=170..590:
 inner padding 24 → 194..566 = 372.
 B (Canonical): w=320 → x=220..540 ✓
 lanes: w=180 each, gap 32 → 392 > 372. Reduce w to 168, gap 32 → 368 ✓. x=200..368, 400..568. But 568 > 566 (inner limit). Shift: x=198..366, 398..566. ✓ Left padding from 170 = 28 ✓.
 Hmm, lane w=168 → available 144 for text; left lane title 126 ✓ subtitle 128 ✓. Right lane subtitle 138 ≤144 ✓ (tight).
 D (Report IR): w=320 → x=220..540 ✓
 E: w=260 → x=250..510 ✓

 viewBox width 720; boundary 170..590; right side has 590..720 empty (130 units). That's a lot of blank space on the right. Should I use it? I could shift everything right or widen. Blank right space looks unbalanced.

Better: keep boundary centered-ish: x=150..630 (w=480), inner 174..606 (432).
 B w=340 → x=220..560
 lanes w=190 gap 34 → 414 ≤432 ✓; x=190..380, 414..604. padding left 40, right 26. ok. Lane w=190 → text avail 166; left subtitle 128 ✓, right 138 ✓.
 D w=360 → x=210..570
 E w=280 → x=250..530
 Corridor at x=95: does it clash? boundary starts at 150, corridor at 95, capsule at 40..100 — gap to line... line at 95 would be inside capsule 40..100. Conflict.

Set corridor x=110: capsule x=40..100 (gap 10). Boundary left 150. Gap corridor→boundary = 40. ✓
 Feedback path: (110, 612) → (110, 364) → (210, 364). Enters boundary at 150 and hits Report IR left edge 210. Hmm, arrow length inside boundary = 60. Fine.

 Right side blank: 630..720 = 90 units. Not terrible but noticeable. Shift everything left by ~20 and widen? Let me instead make boundary x=130..650 (w=520), inner 154..626 (472).
 B w=360 → x=190..550
 lanes w=200 gap 40 → 440 ≤472; x=160..360, 400..600. left padding 160-130=30 ✓ right 650-600=50. Hmm asymmetric. Center the lanes: inner center = (154+626)/2 = 390. lanes total 440 → x=170..610. So left lane 170..370, gap 40, right 410..610. Padding 40 both sides ✓.
 D w=360 → x=210..570 ✓ (center 390)
 E w=280 → x=250..530 ✓
 Corridor at 110, capsule 40..100. Boundary left 130 → gap 20 from corridor. OK.
 Right blank: 650..720 = 70. Acceptable since safe area is 40.

Hmm, actually node A pill centered at 360, but everything else centered at 390. Let me just center A at 390 too: A x=270..510 (w=240). And the A→B arrow at x=360 → should be 390. Let me use center X = C = 390 for all centered elements: A pill x=270..510, B center = (190+550)/2=370. Hmm B is 190..550 → center 370. Not 390. 

I'm overcomplicating. Let me fix consistent numbers:

C = 390 (main axis).
- A: w=240 → x=270..510, y=32..88. center 390 ✓
- Boundary: x=130..650, y=112..544.
- B: w=360 → x=210..570, y=156..212. center 390 ✓
- Lanes: each w=200, gap 40, total 440 → x=170..370 and 410..610. Global center = 390 ✓
  - left lane center = 270; right lane center = 510.
- D: w=360 → x=210..570, y=368..424. center 390 ✓
- E: w=280 → x=250..530, y=464..520. center 390 ✓
- Arrows:
  - A→B: (390,88) → (390,156)
  - B→left lane: from B bottom (300,212) → (270,252)
  - B→right lane: from (480,212) → (510,252)
  - left lane→D: (270,324) → (330,368)
  - right lane→D: (510,324) → (450,368)
  - D→E: (390,424) → (390,464)
  - E→verif: (390,520) → (390,612)  [chip row top]
  - feedback: (110,612) → (110,364) → (210,364) [D left edge, center y=396? D spans y=368..424 → center 396]. Use y=396: path (110,612) → (110,396) → (210,396). But wait, does the vertical at x=110 from 612 up to 396 cross the lane boxes? Lanes start at y=252..324, x from 170. x=110 < 170 ✓ no.
  - Also the vertical x=110 crosses nothing else.
  - capsule at x=40..100, y=384..408, text "问题回流" centered at (70, 400). Check: does the capsule collide with the horizontal segment? The horizontal is at y=396 from x=110 to 210. Capsule spans x=40..100 → no overlap with x>=110 ✓. Gap 10 ✓.

- Chips row: y=612..668 (h=56). Chip width: inner span from 40 to 680 → 640; 3 chips, gap 32 → w = (640-64)/3 = 192. x: 40..232, 264..456, 488..680. ✓ Gap 32 ✓.
  Caption "验证体系" above? The chips row starts at 612, boundary bottom is 544 → 68 gap. Put caption text at x=40, baseline y=600 (12px? use 14px medium). Then E→verif arrow at x=390 from 520 to 612 passes through y=600 at x=390 — collides with caption? Caption at x=40..96 (4 chars*14=56) → ends at 96, arrow at 390 ✓ no collision.
  Hmm, but is the caption needed? It labels the group. Yes, helpful. Keep "验证体系" at x=40 baseline 600.
  Wait, then the feedback starts at (110,612) which is chip1's top edge (chip1 x=40..232, so 110 is on its top edge) ✓.

- H = 668 + 32 = 700.

Check text fits in chips (w=192, avail 168):
 - "Accuracy Validator" 18*8=144 ✓ ; subtitle "数值与结论" 5*12=60 ✓
 - "Style Validator" 15*8=120 ✓ ; "版式与结构" 60 ✓
 - "Regression Test" 15*8=120 ✓ ; "基线与快照" 60 ✓

Node B text: title "Canonical Data Model" 20*8=160 ✓ (avail 336); subtitle "归一化事实层 · 全量溯源" → Chinese 11*12=132 ✓.

Node D: title "Report IR" 72 ✓; subtitle "报告中间表示 · 与 Word 模板解耦" → Chinese 11*12=132 + latin 6*7=42 = 174 ✓ (avail 336).

Node E: title "模板引擎 · 渲染" 7*14=98 ✓ (avail 256); subtitle "DOCX / PDF" 77 ✓.

Lane left (w=200, avail 176): title "程序通道 · 确定性" 9*14=126 ✓; subtitle "解析 / 计算 / 规则" ~128 ✓.
Lane right: title "AI 通道 · 不确定性" 114 ✓; subtitle "抽取 / 理解 / 组织语言" 138 ✓.

Node A (w=240, avail 216): title "多源原始资料" 84 ✓; subtitle "文档 / 表格 / 图片 / 模板" ~180 ✓.

Text vertical centering: for a 56-height node at y=156..212: title baseline y=182, subtitle baseline y=200. Let me compute: two lines, title 14px (line 20), subtitle 12px (line 18) → total 38, centered in 56 → start at y+9=165, title baseline ≈ 165+15=180, subtitle baseline ≈ 180+18 = 198. Use title y=180, subtitle y=198.

For lanes h=72 (y=252..324): total 38 → start 252+17=269, title baseline 284, subtitle 302.

Node A: y=32..88 (h=56): title y=56, subtitle y=74.

Pill for A: rx=28.

Now for the SVG defs marker.

Let me write the HTML.

Structure:
```html
<style>
:root { ...tokens... }
:root[data-widget-theme="dark"] { ...overrides... }
.wrap {...}
...
</style>
<div class="wrap" data-dynamic-ui-widget data-template="node-flow">
  <div class="ttl">报告生成与验证闭环</div>
  <svg viewBox="0 0 720 700" width="100%" height="auto" role="img" aria-label="...">...</svg>
</div>
```

Hmm — "Compact — show the essential inline" and card rules. A card title is fine. Should the card have surface background? Rules: cards use --surface. For diagrams, the fallback primitive `node-flow` — I don't know its exact spec but I'll follow tokens: card with surface, border, radius-card, padding spacer-20.

Wait, the rule "Top-level widget cards should fill the available chat width" — so width:100%.

I need to check data-dynamic-ui-widget root + data-template attribute. I'll set data-template="node-flow".

Let me also double check: node fill uses var(--surface) but the card is also var(--surface) — invisible. Node borders give definition. Better: card background var(--surface), nodes var(--surface-muted)? The token rules say nested regions use --surface-muted. Nodes on a card = nested → use --surface-muted for node fills, border for stroke. I'll do nodes fill var(--surface-muted). Brand node: fill var(--brand-soft), stroke var(--brand), text var(--brand-text). Boundary container: fill none, dashed stroke var(--border).

Hmm, --brand-soft on light = #E5FFF2, on dark = #004D26 — and --brand-text on light = #004D26, dark = #E5FFF2. The .c-brand class CSS handles it. I'll just include the class CSS from the token file (the documented template) — good, it's authoritative.

I'll include the class CSS block for c-neutral and c-brand plus the text classes and arrow classes. That satisfies "Use only the documented runtime roles".

Let me write final SVG.

```svg
<svg viewBox="0 0 720 700" width="100%" height="auto" role="img" aria-label="报告生成与验证闭环架构图">
  <defs>
    <marker id="arw" viewBox="0 0 8 8" refX="7" refY="4" markerWidth="8" markerHeight="8" markerUnits="userSpaceOnUse" orient="auto">
      <path d="M1 1 L7 4 L1 7 Z" fill="var(--text-muted)"/>
    </marker>
    <marker id="arwb" viewBox="0 0 8 8" refX="7" refY="4" markerWidth="8" markerHeight="8" markerUnits="userSpaceOnUse" orient="auto">
      <path d="M1 1 L7 4 L1 7 Z" fill="var(--brand)"/>
    </marker>
  </defs>
  ...
```

Marker ids must be unique-ish; fine.

Note: `fill="var(--text-muted)"` inside marker — SVG attribute with var() works in modern browsers. The skill says SVG fill must reference var(...). OK.

Node A (pill):
```
<g class="c-neutral">
  <rect x="270" y="32" width="240" height="56" rx="28"/>
</g>
<text class="th" x="390" y="56" text-anchor="middle">多源原始资料</text>
<text class="ts" x="390" y="74" text-anchor="middle">文档 / 表格 / 图片 / 模板</text>
```
Careful: .c-brand > .th selector requires text inside the g. For c-neutral, .th uses var(--text) which is fine globally. To be safe, put texts inside the g. For c-brand I need the text inside the g to get brand-text color. Let me put texts inside each g.

Boundary:
```
<rect x="130" y="112" width="520" height="432" rx="12" fill="none" stroke="var(--border)" stroke-dasharray="6 6"/>
<text class="th" x="154" y="140">生成流水线</text>
```

Node B:
```
<g class="c-neutral">
  <rect x="210" y="156" width="360" height="56" rx="8"/>
  <text class="th" x="390" y="180" text-anchor="middle">Canonical Data Model</text>
  <text class="ts" x="390" y="198" text-anchor="middle">归一化事实层 · 全量溯源</text>
</g>
```

Lanes:
```
<g class="c-neutral">
  <rect x="170" y="252" width="200" height="72" rx="8"/>
  <text class="th" x="270" y="284" text-anchor="middle">程序通道 · 确定性</text>
  <text class="ts" x="270" y="302" text-anchor="middle">解析 / 计算 / 规则</text>
</g>
<g class="c-neutral">
  <rect x="410" y="252" width="200" height="72" rx="8"/>
  <text class="th" x="510" y="284" text-anchor="middle">AI 通道 · 不确定性</text>
  <text class="ts" x="510" y="302" text-anchor="middle">抽取 / 理解 / 组织语言</text>
</g>
```

Node D (brand):
```
<g class="c-brand">
  <rect x="210" y="368" width="360" height="56" rx="8"/>
  <text class="th" x="390" y="392" text-anchor="middle">Report IR</text>
  <text class="ts" x="390" y="410" text-anchor="middle">报告中间表示 · 与 Word 模板解耦</text>
</g>
```

Node E:
```
<g class="c-neutral">
  <rect x="250" y="464" width="280" height="56" rx="8"/>
  <text class="th" x="390" y="488" text-anchor="middle">模板引擎 · 渲染</text>
  <text class="ts" x="390" y="506" text-anchor="middle">DOCX / PDF</text>
</g>
```

Chips (y=612..668):
```
<g class="c-neutral"><rect x="40" y="612" width="192" height="56" rx="8"/>
 <text class="th" x="136" y="636" text-anchor="middle">Accuracy Validator</text>
 <text class="ts" x="136" y="654" text-anchor="middle">数值与结论</text></g>
... 
```
Wait: chip1 x=40..232, and the feedback line starts at (110,612) which is on chip1's top edge — fine.

Chip2 x=264..456, center 360. Chip3 x=488..680, center 584.

Caption "验证体系" at x=40, y=600.

Hmm, chip1 top edge y=612 and caption baseline 600 → caption occupies ~588..603, gap to chip 9. ✓

Arrows:
```
<path class="arr" d="M390 88 L390 150" marker-end="url(#arw)"/>
```
Note: the arrow tip should be at the node top edge y=156 — marker refX=7 with markerUnits userSpaceOnUse places the tip at the path end. So path should end at y=156. Let me end exactly at the target edge: y=156. Similarly others.

- A→B: M390 88 L390 156
- B→left: M300 212 L270 252  (diagonal)
- B→right: M480 212 L510 252
- left→D: M270 324 L330 368
- right→D: M510 324 L450 368
- D→E: M390 424 L390 464
- E→验: M390 520 L390 612
- feedback (brand dashed): M110 612 L110 396 L210 396, marker-end arwb, stroke var(--brand), dasharray.

Wait, the feedback horizontal ends at Report IR's left edge x=210, y=396 (D spans 368..424, center 396 ✓).

Capsule for "问题回流":
```
<rect x="40" y="384" width="60" height="24" rx="12" fill="var(--surface)" stroke="var(--border)"/>
<text class="ts" x="70" y="400" text-anchor="middle">问题回流</text>
```
Text baseline: rect y=384..408, center 396; for 12px text baseline = 396 + 4 = 400. ✓
Hmm, capsule fill var(--surface) = same as card background → it's a "background rectangle hiding collisions" use? No, it's a legitimate label capsule per the rules ("Put connector labels in small rounded capsules with surface fill"). ✓ But since nothing is behind it (it's in blank space), I could simplify to plain text. Actually the rule warns against using background rects to hide collisions. Here there's no collision; a capsule is fine and matches the spec. Keep it but maybe fill --surface-muted for visibility. Rules say capsules with surface fill. Use --surface-muted for slight distinction. Fine.

Wait — the capsule is 60 wide, and 4 Chinese chars at 12px = 48 → fits with 6 padding each side. Rule: "Every bordered or filled container must have internal padding" — 6 each side, ok. Make it 64 wide (x=38..102)? Then gap to line at 110 = 8. Exactly the minimum. Let me keep w=60 at x=40..100, gap 10, text centered at 70 with 6px padding. Acceptable. Actually let me widen slightly: x=36, w=64 → 36..100, gap 10, text padding 8. x=36 is within safe area (40)? Slightly outside. Keep x=40, w=60.

Hmm, "问题回流" at 12px: Chinese chars are typically 12px wide at font-size 12 → 48px. ✓

Now the overall SVG height 700 with last element bottom 668 → 32 padding ✓.

Also need `stroke-width` for arrows: 1.5.

One more check: the "生成流水线" boundary label at x=154, y=140 (baseline), text height ~14 → occupies 128..144. Node B top is 156 → gap 12. ✓ Boundary top 112 → label top 128, 16 padding ✓.

Now, is the boundary containing all inner nodes with padding? Boundary 130..650 x, nodes leftmost = lanes at x=170 (padding 40) ✓, rightmost = lanes at 610 (padding 40) ✓. Top: B at 156, boundary 112 → 44 but the label sits in there. ✓ Bottom: E bottom 520, boundary bottom 544 → 24 ✓.

The A→B arrow crosses the boundary top edge at x=390 — that's expected (entering the container).

Also the feedback arrow enters the boundary from the left at y=396. Fine.

Everything checks. One concern: the E→验证 arrow (390,520)→(390,612) crosses the boundary bottom edge at 544 — expected.

Let me now also double check that no text exceeds node widths. Lane subtitle "解析 / 计算 / 规则": chars 解析计算规则 = 6 Chinese * 12 = 72, plus " / " x2 = 6 latin chars * 7 = 42 → 114 ≤ 176 ✓.

"抽取 / 理解 / 组织语言": 8 Chinese*12=96 + 6*7=42 = 138 ≤ 176 ✓.

A subtitle "文档 / 表格 / 图片 / 模板": 8 Chinese*12=96 + 9*7=63 = 159 ≤ 216 ✓.

Good.

Now write the final response. It's going to be long. Let me structure it well and keep it dense.

I'll place the diagram after section C (系统架构).

Actually, the response order: I should give A first with intent confirmation. Then B..H. The widget can go in C.

Let me draft the content now, being careful and precise but concise. I'll use tables.

Let me write section by section.

---

A. 总体理解

I'll state: 这个项目真正要解决的不是"生成报告"，而是三件事... and confirm intent + point out where I might disagree/or flag assumptions. Per user rule 4 and 1 ("通过自行检查去确认用户所说的信息是不是这样").

Key reframing (value-add):
1. 你真正要造的是"报告生产函数 f(输入)=报告"的**可分解、可验证**实现，而不是一个更聪明的 Prompt。报告只是这个函数的一个投影。
2. 因此本系统真正的第一等公民是**中间表示与验证体系**，不是 LLM。
3. 历史报告的价值不是训练数据，而是**规格来源 + Ground Truth**：它同时定义了"输入长什么样"（逆抽取）和"输出应该长什么样"（模板+版式+结论口径）。
4. 我判断最大的风险不是"AI 写不好"，而是**"AI 写得太好以致于错误不可见"** —— 生成流畅但数值/结论错误，人工抽检抓不住。所以"可验证性"必须先于"生成质量"。
5. 一个隐含但关键的第 15 项需求：**"正确地失败"**。系统在数据不足时必须显式缺项，而不是补全。

需要你确认的判断（3条）：
- 报告类型是否可归类（少数几类结构稳定的）？还是每份都不同？→ 决定 Canonical Model 是否能收敛。
- 是否存在"必须人工判断"的环节（如现场情况定性描述）？→ 决定自动化天花板。
- 是否要求报告与历史报告"版式完全一致"（像素级）还是"符合模板规范"？→ 决定 Style Validator 的严格度。

These questions matter. I'll list them as 待确认 assumptions rather than asking via tool (they asked for output). Actually I could use AskUserQuestion at the end. But the user asked for a deliverable. I'll list them in A as "需要你确认的3个前提" and offer to adjust. Maybe better: use AskUserQuestion at the end? The instructions say reserve for decisions where the answer changes what I do next. Here the answer does change the design... but the user asked for the full design output. I'll deliver the design and note the assumptions. I think delivering is better than blocking. Hmm, but user rule says "在进行任务前必须将你理解的意图提供给用户进行确认". A is that. Good enough.

B. 关键设计决策 (D1-D12 table)

C. 系统架构 — layers + diagram

D. 数据模型 — layers L1-L4 + core entities table + provenance/conflict

E. Report IR — structure, key mechanism (无自由文本), example JSON sketch, versioning

F. Agent/Skill

G. 验证体系 — Accuracy / Style / Regression + metrics + P0-P3

H. MVP — scope + 10 steps order + 不做清单

Also need section for LLM Adapter (item 12) — put into C or its own subsection under F or G. I'll add it under C or as part of B/F. Let me put "LLM Adapter" as a subsection of F (Agent/Skill 与模型无关层) or under C. I'll add it in C as a component and detail in F. Actually item 12 is explicit — I'll make it a labeled subsection under F.

And item 14 (不做清单) under H.

Also "系统边界" (item 2) — put in A or B. I'll add a subsection in A or B: "系统边界". Let me include as part of B (关键设计决策) or a separate short section. I'll put 边界 in A after the reframing, or make B include it. Let me structure:

A. 总体理解（含系统边界）
B. 关键设计决策
C. 系统架构（含分层 + 图 + LLM Adapter 位置）
D. 数据模型
E. Report IR
F. Agent / Skill 设计（含 LLM Adapter 接口）
G. 验证体系
H. MVP 与不做清单

Plus a 概念定义表 (concept-align) — I'll fold into D or B. Maybe put a compact 术语表 at the start of B or as part of D. I'll add it as a short block in B (关键概念精确化), since ambiguous concepts are the root of design rework.

Let me now write it. Keep tables tight.

Regarding numbers: avoid time estimates.

Let me write the final answer.

One more thing about the diagram: I said I'd place it in C. Let me write C text, then call PureShowWidget, then continue with D. That means the tool call comes in the middle of my response — that's allowed (streaming). But since I must output the widget code in the tool call, I need to do it as a tool call in the middle. Actually I can't "pause" text and call a tool and continue... actually I can: my response is a sequence of content blocks; a tool call in the middle is fine, and then I continue after the result. But the tool result returns to me and then I write the rest. That works.

Hmm, but the dynamic-ui skill says: "Prefer calling the tool as the next user-visible action. Do not stream planning notes..." — so I should call it and not narrate. To keep flow simple, I'll put the tool call near the end (after C text), then finish with D-H after the result. That creates a two-turn response. That's acceptable.

Alternative: put the diagram at the top of C and then write D-H after the tool result. Let me do: write A, B, C text → call PureShowWidget → then D-H.

Let me make sure PureShowWidget is a deferred tool — yes, needs ToolSearch first. Let me search for it.

I'll do ToolSearch + write A/B/C text in the same message? Tool search is a tool call; text before it is shown. Yes, do that.

Actually let me just do ToolSearch now and the text.

Given the length, let me write A and B and C fully in this message, then ToolSearch, then in the next message call PureShowWidget, then D-H. Hmm, that's 3 messages. Fine.

Actually simpler: do ToolSearch in this message together with A/B/C text. Then next message: PureShowWidget call. Then next: D-H. OK.

Let me write.Now write A, B, C text, then call PureShowWidget.

Let me write the content.

The widget code got truncated? I passed a truncated JSON (I see my widget_code ends mid-path). The tool returned no output, which is odd. Let me re-issue the full widget code properly.

Actually looking at my call, the widget_code string was cut off at `<path class=\"arr\" d=\"M390 88 L390 156\" marker-end=\"url(#nf-arrow)\"}` — I accidentally truncated it. Let me redo the full call.

Let me write the complete widget code carefully.

The tool returns no output but presumably rendered. Fine. Continue with D-H.

Now write the rest: D 数据模型, E Report IR, F Agent/Skill + LLM Adapter, G 验证体系, H MVP + 不做清单.

Keep it dense with tables.
图中左侧的"问题回流"虚线是整套设计的关键：验证结果必须能定位回 IR 的具体节点，否则回归测试无法归因。

---

## D. 数据模型（Canonical Data Model）

### D1. 四层结构，禁止"一个大统一模型"

| 层 | 名称 | 内容 | 可变性 |
|---|---|---|---|
| L1 | RawSource | 文件、页、段落、单元格、图片的原始内容与坐标 | 只读，永不修改 |
| L2 | **Fact Layer** | 最小断言单元，带来源与置信度 | 只增不改，冲突独立表达 |
| L3 | Domain Objects | 工程语义实体（项目/构件/检测项/测点…） | 由 Facts 组织而来 |
| L4 | Report IR | 面向报告结构 | 每次生成重建 |

关键：**L2 只存"是什么"，不存"怎么写"。** 所有措辞属于 L4。这条线一旦破，模板与内容就会互相污染。

### D2. Fact 的强制字段（这是整个系统可验证性的地基）

| 字段 | 说明 | 缺失后果 |
|---|---|---|
| `id` | 稳定标识，IR 通过它引用 | 无法定位 |
| `value` / `unit` | 值 + 单位，数值一律存 SI 基准 + 原始单位 | 单位错误不可查 |
| `source_refs[]` | 文件 + 定位（页/表/单元格/图片ID） | 不可追溯 |
| `method` | `measured` / `computed` / `quoted` / `inferred` | 无法判断可信度 |
| `provenance` | `program` / `ai` / `human` | 无法定位错误来源 |
| `confidence` | 0–1，仅 AI 来源需要 | 无法设人工闸门 |
| `status` | `filled` / `missing` / `conflict` / `rejected` | 会静默补全 |

### D3. 领域对象骨架（以检测鉴定报告为例）

`Project` → `Client` / `Structure(楼-层-轴)` → `Component(构件)` → `InspectionItem(检测项)` → `MeasurementSet(测点集)` → `Criterion(规范限值)` → `Evaluation(判定)` → `Conclusion(结论)` → `Evidence(图片/原始记录)`

这套骨架里，`Criterion` 与 `Evaluation` 是**唯一允许"产生新信息"的地方**，而且必须是纯程序按规则执行——因为"合格/不合格"是法律意义上的断言。

### D4. 冲突处理

同一事实多来源不一致时，生成 `Conflict` 记录，**不允许静默覆盖**：

```
Conflict{ fact_id, candidates:[{value, source_ref, provenance}], resolved_by, resolution }
```

第一阶段 `resolved_by` 只能是 `human`。冲突被显式暴露，而不是被"最后写入者胜出"抹掉，是本项目区别于 Demo 的关键之一。

---

## E. Report IR

### E1. 结构

```
Document → Section(语义标签, 章节树) → Block → Inline
Block ∈ { Heading, Paragraph, List, Table, Figure, Formula, PageBreak, Toc, Signature }
```

### E2. 核心机制：段落不允许自由文本

这是全系统最重要的一个设计。段落由**片段序列**组成，不是字符串：

```json
{
  "type": "Paragraph",
  "segments": [
    { "lit": "经检测，" },
    { "ref": "fact:concrete.grade" },
    { "lit": "，实测强度推定值为 " },
    { "ref": "fact:concrete.fcu_e", "format": { "decimals": 1, "unit": "MPa" } },
    { "lit": "，满足设计要求。" }
  ]
}
```

由此获得四个能力，缺一不可：

| 能力 | 说明 |
|---|---|
| 数值不可编造 | LLM 只能选择引用哪个 Fact，不能书写数字本身 |
| 格式集中控制 | 小数位、单位、有效数字由程序格式化，不由模型决定 |
| 渲染前可校验 | 所有 `ref` 必须可解析，否则直接失败 |
| diff 稳定 | 结构相同时，diff 只显示变量值变化，回归报告可读 |

### E3. 表格由程序生成，不由 LLM 生成

`TableSpec { headers, rows: Cell[] }`，`Cell ∈ { Ref | Computed | Lit }`，从 `MeasurementSet` 确定性生成。LLM 可以决定"哪张表放在哪一节"，但不能决定"表格里的数字"。

### E4. 与模板解耦

| 层 | 内容 | 谁负责 |
|---|---|---|
| IR 语义层 | 章节标签如 `conclusion`、`test_method`、`result_table` | 业务逻辑 |
| 模板映射层 | 标签 → Word 样式名/位置/编号规则 | 模板配置 |
| 渲染层 | 填充、编号、目录、页码、分页 | 程序 |

换模板只改映射层；换章节内容只改语义层。这是"D11 决策"的落地方式。

### E5. 治理

IR 必须有 **JSON Schema + 版本号 + 校验器**。任何 IR 在被渲染前必须先过 Schema 校验；Schema 变更必须记录到 [07_DECISIONS.md](file:///g:/workspace/zixun4/docs/07_DECISIONS.md)。

---

## F. Agent / Skill 设计

### F1. 分工原则

| | Agent | Skill |
|---|---|---|
| 职责 | 任务拆解、编排、状态管理 | 单一能力 |
| 有无目标/循环 | 有 | 无 |
| 输入输出 | 结构化状态 | Schema 化的输入输出 |
| 是否可单测 | 靠集成测试 | 必须可单测 |

**Agent 不写报告正文。** 这是硬约束，否则又退化成"一个万能 Agent 调 Prompt"。

### F2. Agent 清单（少而清晰，5 个）

| Agent | 职责 | 禁止 |
|---|---|---|
| IntakeAgent | 判定输入类型、调度解析 Skill、登记缺失 | 不得推断缺失值 |
| ExtractionAgent | 文本/图片 → Facts | 不得输出自由文本结论 |
| ReasoningAgent | 按规范做判定、生成结论**要点** | 不得生成段落文本 |
| ComposerAgent | 按 IR 骨架组织 Facts → Report IR | 不得写入任何未注册 Fact |
| ValidationAgent | 调用验证器、汇总 IssueList、有限次自修复 | 不得自行修改 Ground Truth |

**Agent 之间只传结构化数据**（Facts / IR / IssueList），不传自然语言。这是阻断幻觉沿链路放大的关键设计。

### F3. Skill 分类

| 类别 | 实现 | 示例 | 特征 |
|---|---|---|---|
| 确定性 Skill | 程序 | `parse_xlsx`、`unit_convert`、`apply_rule`、`render_docx`、`extract_template_spec`、`validate_ir_schema` | 无 LLM，必然可单测 |
| 不确定性 Skill | LLM | `extract_fields_from_text`、`extract_fields_from_image`、`classify_document`、`draft_conclusion_points` | 有输出 Schema 约束 |

每个 Skill 必须有：`id`、`version`、`kind`、`input_schema`、`output_schema`、`test_cases`。**Skill 自身也要有回归测试**——这是"换模型不重构"的前提。

### F4. LLM Adapter（模型无关层）

只做四件事，不做"万能 Provider 框架"：

| 接口 | 说明 |
|---|---|
| `generate(messages, schema, params) → StructuredResult` | 统一结构化输出；不同模型对 JSON mode / function calling 支持不同，Adapter 内部做转换 + 校验 + 有限次重试 + 修复 |
| `capabilities()` | 声明 `text_extraction` / `vision` / `long_context` / `json_mode` |
| 配置外置 | `model_profile`（provider、model、base_url、params、context_limit、cost），不进代码 |
| Trace 记录 | prompt hash、token、耗时、原始响应 → 供调试与回归比对 |

上层代码只依赖**能力**，不依赖模型名。DeepSeek → GLM → Qwen → Claude → 本地模型，只换 `model_profile` 与一次回归跑分。

不要做：模型抽象"总线"、多 Provider 插件体系、统一 token 计费中心——第一阶段全是负债。

---

## G. 验证体系

### G1. Accuracy Validator（四层，越往后越"软"）

| 层 | 检查内容 | 实现 |
|---|---|---|
| V1 合法性与完整性 | IR Schema、必填项、`ref` 可解析、`missing/conflict` 是否被处理 | 程序 |
| V2 数值与源数据一致 | 报告数值回查 Fact → SourceRef；表内合计 = 分项之和；单位/量纲一致 | 程序 |
| V3 逻辑一致性 | 判定与限值逻辑一致（如推定值 < 限值却判"合格"）；正文引用的表号/图号/章节号存在 | 程序 |
| V4 语义一致性 | 结论方向与数据一致；结论覆盖全部检测项；无凭空出现的数字/规范号 | AI 辅助 + 规则 |

输出统一为 `IssueList`，每条含：`severity`（error/warn/info）、`location`（IR 路径）、`evidence`、`rule_id`、`suggested_fix`。**可被回归测试直接断言。**

### G2. Style Validator（用"样式指纹"，不用截图比对）

| 层 | 检查内容 |
|---|---|
| S1 硬版式（可精确） | 字体/字号/行距/缩进/对齐/页边距/页眉页脚/页码/标题样式名/表格边框/"表x-x"编号连续性/图注位置 |
| S2 结构合规 | 章节顺序、必需要素（委托方、日期、检测依据、结论、签字栏）、编号规则 |
| S3 语言风格 | 术语一致性、人称时态、模板句覆盖率、口语化检测 |

方法：从模板/历史报告提取 `style fingerprint`（样式名 → 属性集合），从生成报告提取同样指纹，做集合比对。可差分、可回归、可定位到具体段落。

### G3. Regression Test（三层）

| 层 | 范围 | 断言对象 |
|---|---|---|
| R1 单元层 | 每个 Skill 的 fixture | 输入 → 期望输出，因果可定位 |
| R2 用例层 | 每个 Evaluation Case 全流程 | 分层 metrics + IssueList 差异 |
| R3 变化层 | 模型/Prompt/模板/程序版本变更 | 基线快照 vs 当前快照 → 回归报告（新增错误/已修复/未变） |

每次运行落盘 `run snapshot`：输入 hash、Skill 版本、模型名+版本、参数、输出 hash、metrics。基线固定，任何变化可见。

### G4. 指标分级与门禁

| 级别 | 指标 | 门禁 |
|---|---|---|
| P0 | 数值准确率、结论判定一致率、**"该失败时是否失败"** | 必须 100%，否则构建失败 |
| P1 | 结构一致率（章节/表格/图片数量） | ≥ 阈值 |
| P2 | 文本语义一致率（结论段落等价判定） | 趋势监控 |
| P3 | 版式一致率 | 趋势监控 |

**不设"端到端相似度"这种单一指标**——它会把 P0 的数值错误掩盖在 95% 的文本相似度里。

### G5. 历史报告 → Evaluation Case 的方法

| 步 | 动作 | 产物 |
|---|---|---|
| 1 | 选样 10–20 份代表性报告（类型/复杂度/边界情况） | 样本清单 |
| 2 | 逆抽取：报告 → Facts（LLM Skill + 人工校验） | **"报告声称的事实"** |
| 3 | 固化用例目录：`inputs/`、`ground_truth/`、`meta.json` | 可版本管理的用例 |
| 4 | 分层断言：数值逐一比对（容差可配）+ 结构集合 + 结论方向 + 样式指纹 | 断言集 |
| 5 | 构造陷阱用例：缺失数据、冲突数据、超限值、单位异常 | "正确地失败"测试集 |

第 2 步的语义边界必须写进概念表：逆抽取得到的是**报告声称的事实**，不等于**客观事实**。这一点决定了评测的合法上限——我们是在验证"系统能否复现这份报告的推理链"，不是"验证这份报告当年是否正确"。

---

## H. MVP 实施顺序

### H1. MVP 目标（一句话）

**在 1 类报告、1–2 个章节上，打穿"输入 → 生成 → 验证 → 回归"的完整闭环。**

范围锁定：Excel 数据 + 工程概况文本 + 一份 Word 模板 → DOCX；验证数值一致性 + 硬版式；3–5 份历史报告转为用例；CLI + JSON 中间产物，无 UI。

### H2. 实施顺序（顺序本身就是设计）

| 步 | 内容 | 关键点 |
|---|---|---|
| 0 | 概念表 + IR Schema + Fact Schema | 先定义，后实现（D1） |
| 1 | 输入解析最小集：xlsx → MeasurementSet；概况文本 → ProjectMeta | 全部落盘 JSON |
| 2 | Fact Layer + 3–5 个确定性计算 Skill | 每个 Skill 配 fixture |
| 3 | **IR Builder（纯程序，不含 LLM）** | 从 Facts 生成 IR |
| 4 | 模板映射 + DOCX 渲染 | 用成熟库，不自研排版 |
| 5 | LLM Adapter + 1 个抽取 Skill | 带 Schema 校验与重试 |
| 6 | Accuracy Validator V1–V2 | 数值回查与合计校验 |
| 7 | 历史报告 → 3–5 个 Evaluation Case | 含 1 个陷阱用例 |
| 8 | 回归脚本 + 基线快照 + P0 门禁 | 先立基线，再谈优化 |
| 9 | Style Validator S1 | 样式指纹比对 |
| 10 | 接入 LLM 生成结论要点 + V4 语义校验 | 最后才让 LLM 上场 |

**第 3 步是分水岭**：此时系统在"零 LLM"状态下已能端到端产出可校验的报告。此后每一步引入 LLM，都是一次可测量的增量，而不是不可归因的跳变。

### H3. 第一阶段绝对不要做

| 类别 | 具体 |
|---|---|
| 基础设施 | 微服务、Kubernetes、Redis、MQ |
| 架构花样 | Multi-Agent 自治协商、知识图谱、事件总线 |
| AI 花样 | 向量数据库 / 全量规范 RAG（先做人工指定条文 + 规则表）、模型微调、Agent 自反思循环 |
| 自研 | Word 排版引擎、DOCX 解析器（用 python-docx / docxtpl 等成熟库） |
| 产品化 | 前端 UI、在线编辑器、审核流、权限 |
| 范围 | 一次覆盖所有报告类型/所有章节；追求"无人值守全自动生成" |
| 指标 | 端到端文本相似度作为主指标；用"人工抽检通过"代替自动门禁 |

---

## 下一步建议

设计层面的三个待你确认项已在 A 节列出（报告类型可否收敛、人工闸门位置、版式严格度）。确认后，我建议的下一步是**先产出 `docs/01_GLOSSARY.md`（概念表）与 `docs/02_REPORT_IR_SPEC.md`（IR Schema 草案）**，这两份文档定稿之前不写任何业务代码——因为 IR 一旦定错，后面所有渲染、验证、回归都要返工。
