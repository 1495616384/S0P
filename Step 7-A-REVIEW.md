我需要先阅读完整的项目文档和检查现有代码结构。让我并行执行这些操作。
用户要求我进入工程实现阶段前，先完成分析工作。需要阅读所有核心文档，检查现有代码结构，然后给出分析报告。

让我先读取所有必要的文档和代码结构。文档中有一些已经在系统提示中了（00_PROJECT_CONTEXT.md, 05_EVALUATION.md, 07_DECISIONS.md），但我还需要读取 02_ARCHITECTURE.md, 03_CANONICAL_DATA_MODEL.md, 04_REPORT_IR.md，以及检查当前项目的实际代码结构。

让我先并行读取这些文档和检查目录结构。

让我再检查一下是否有任何隐藏的配置文件或已存在的代码。
好的，我已经读取了所有核心文档。让我再检查一下是否有隐藏的代码目录（比如 `.gitignore` 可能排除的），以及是否有任何配置文件（`requirements.txt`、`pyproject.toml`、`package.json` 等）。

当前情况：
- 工作目录 `g:\workspace\zixun4` 下**完全没有代码**
- 只有文档和规则文件
- 没有 `requirements.txt`、`pyproject.toml`、`setup.py` 或任何依赖文件
- 没有 `.git`、没有 `.gitignore`（可能还没初始化 git）
- 没有 `tests/` 目录、没有 `src/` 或 `app/` 目录

让我再确认一下是否有其他隐藏文件。

现在我已经完整阅读了所有文档并确认了项目现状。让我调用 AdvisorTool 进行策略性指导。

*正在请求专家建议，优化解决方案，该过程耗时可能较长，请耐心等待...*

好的，让我补充读取项目规则文件，确保没有遗漏任何约束。
好的，确认了——整个项目**完全没有代码**。只有设计文档。这意味着：

1. 这是一个纯 Greenfield 项目
2. 没有任何可复用的现有代码
3. 没有任何与架构冲突的现有代码（因为没有现有代码）
4. 所有代码都需要从零开始创建

现在让我按照用户的要求，进行完整的分析。我需要调用 AdvisorTool 来进行策略性指导，因为这是一个重要的架构决策节点。
正在请求专家建议....

已获取专家建议

AdvisorTool 提醒我还需要读取一些文件。但实际上系统提示中已经把 00_PROJECT_CONTEXT.md、05_EVALUATION.md 和 07_DECISIONS.md 的内容都注入了（这三个文件是通过 system-reminder 方式注入的 Read 结果）。让我再确认一下是否需要读取 project-architecture.md 规则文件。

实际上，这三个文件的完整内容我已经在系统上下文中了。让我读取一下 project-architecture.md 规则文件。

好的，现在我已经完整阅读了所有必要的文档。让我总结一下关键发现：

**项目现状**：完全 Greenfield 项目，零代码，只有设计文档。

**M0 冻结任务（来自 02_ARCHITECTURE.md §17.2）**：
1. Fact Schema + fact_type 注册表 + supersedes
2. Criterion Schema + source_refs / review_status
3. Report IR Schema + anchors 绑定表 + covers[]
4. Evaluation Case Schema + 白名单登记表（含确认方字段）
5. 校验器骨架：Ref 可解析 / 锚定绑定 / 白名单命中

**技术栈未明确**——所有文档都没有指定使用 Python、Go、TypeScript 还是其他语言。文档中提到了 python-docx（可能是对渲染器的提示），但没有硬性规定。

让我现在进行完整的分析输出。需要注意以下几点：

1. 当前无代码，所以"可复用"和"冲突"两部分基本为空
2. 用户要求"不重新设计"、"不写业务代码"、"不创建文件"
3. 重点是确定 M0 / Canonical Data Model 的最小实现范围
4. 技术栈选择是一个必须指出的风险

让我组织输出格式。
好，所有文档都已完整阅读。现在进行分析输出。

---

# 一、当前项目实际结构

```
g:\workspace\zixun4\
├── .trae\rules\project-architecture.md
└── docs\
    ├── 00_PROJECT_CONTEXT.md
    ├── 02_ARCHITECTURE.md
    ├── 02_ARCHITECTURE_REVIEW.md
    ├── 03_CANONICAL_DATA_MODEL.md
    ├── 04_REPORT_IR.md
    ├── 05_EVALUATION.md
    └── 07_DECISIONS.md
```

**结论：完全 Greenfield。** 零代码、零依赖配置、零测试目录、零 `.git` 初始化。没有任何 Python/Go/TypeScript 源文件，没有 `pyproject.toml`、`requirements.txt`、`package.json`、`go.mod` 或任何构建文件。

---

# 二、可复用代码

无。项目从零开始。

---

# 三、与架构冲突的现有代码

无。不存在冲突前提。

---

# 四、Canonical Data Model 最小实现范围

按 `02_ARCHITECTURE.md §17.2` 的 M0 冻结里程碑 + `03_CANONICAL_DATA_MODEL.md` v0.3 定义，**第一阶段必须实现**的数据对象：

## 4.1 L1 层：RawSource + SourceRef

| 对象 | 来源章节 | 强制字段 |
|------|---------|---------|
| `SourceRef` | §5 | `source_id`, `location` |
| `RawSource` | §3–4 | 文档描述为"保存原始内容与坐标"，无强制 Schema 字段列表，但必须能与 SourceRef 关联 |

## 4.2 L2 层：Fact（核心）

| 字段 | 来源章节 | 备注 |
|------|---------|------|
| `id` | §8 / §8.1 | 三段式：`fact:<scope_class>.<instance_key>.<attribute_path>` |
| `fact_type` | §8 / §8.1 | 受控注册表，如 `component.concrete_strength` |
| `value` / `unit` / `quantity_kind` | §8 / §10 | 原始值 + 原始单位 + 量纲种类 |
| `source_refs[]` | §11 | 至少一个 |
| `method` | §12 | 枚举：`measured` / `observed` / `computed` / `quoted` / `inferred` |
| `provenance` | §13 | 枚举：`program` / `ai` / `human` |
| `status` | §15 | 枚举：`filled` / `missing` / `conflict` / `rejected` / `superseded` |
| `review_status` | §14 | 枚举：`pending` / `confirmed` / `rejected` |
| `confidence` | §14 | 0–1，诊断字段，不驱动决策 |
| `supersedes` | §15.5 | 可选，更正链指向 |
| `value_domain` | §16.4 | 定性 Fact 专用，受控值域 |

## 4.3 Conflict

| 字段 | 来源章节 |
|------|---------|
| `fact_id` | §34 |
| `candidates[]` | §34 / §35 |
| `resolution_policy` | §36（第一阶段固定 `manual`） |
| `resolved_by` / `resolution` | §34 |

## 4.4 Criterion（v0.3 新增，与 Fact 同构）

| 字段 | 来源章节 | 备注 |
|------|---------|------|
| `criterion_id` | §25.1 | 如 `crit:GB50292-2015.clause-5.2.3` |
| `value` / `unit` / `quantity_kind` | §25.1 | 限值本身是数值 |
| `condition` | §25.1 | 适用条件 |
| `source_refs[]` | §25.1 | 规范文件定位 |
| `review_status` | §25.1 | **判定闸门**：`!= confirmed` 不得参与判定 |
| `rule` | §25.1 | 判定形式，如 `value <= limit` |

## 4.5 Evaluation

文档中 Evaluation 作为 Domain Object 存在（§26），但 v0.3 没有详细列出强制字段。**M0 阶段只需要 Schema 骨架**，具体字段在 M2 compute/rules 阶段再细化。

## 4.6 ConclusionRule

文档 §27.1 定义了聚合规则结构。M0 阶段只需要 Schema 骨架。

## 4.7 fact_type 注册表（受控）

必须实现一个**确定性的注册表**，定义：
- `scope_class` 枚举值（§8.1）：`project` / `building` / `component` / `point` / `sample` / `material`
- `quantity_kind` 枚举值（§10）：`length` / `force` / `pressure` / `ratio` / `count` / `area` / `dimensionless` / `qualitative`
- fact_type → quantity_kind 的映射表

---

# 五、建议目录结构

基于 `02_ARCHITECTURE.md §17.1` 的模块清单，**M0 最小目录结构**：

```
zixun4/
├── contracts/                  # 三份 JSON Schema（M0 核心）
│   ├── fact.schema.json
│   ├── criterion.schema.json
│   ├── report_ir.schema.json
│   ├── evaluation_case.schema.json
│   └── whitelist_registry.schema.json
├── src/
│   └── canonical/               # Canonical Data Model 实现
│       ├── __init__.py
│       ├── models.py            # Fact / SourceRef / Conflict / Criterion / Evaluation / ConclusionRule 数据类
│       ├── fact_type_registry.py  # 受控注册表（scope_class / quantity_kind / fact_type 映射）
│       ├── store.py             # FactSet 容器 + 状态机（filled→conflict→resolved）
│       ├── conflict.py          # Conflict 构造与检测逻辑
│       └── exceptions.py        # 自定义异常（SchemaViolation / ConflictError / MissingFactError）
├── tests/
│   └── canonical/
│       ├── test_fact_id_format.py
│       ├── test_fact_status.py
│       ├── test_conflict.py
│       ├── test_criterion_gate.py
│       └── test_fact_type_registry.py
├── docs/                        # 已有，不动
└── .trae/                        # 已有，不动
```

---

# 六、第一批需要创建的文件

按优先级排序：

| 优先级 | 文件 | 理由 |
|--------|------|------|
| P0 | `contracts/fact.schema.json` | M0 核心，所有模块依赖 |
| P0 | `contracts/criterion.schema.json` | 判定闸门的 Schema 定义 |
| P0 | `contracts/report_ir.schema.json` | IR 骨架（Document / Section / Block / Paragraph 双模态 / Ref / TableSpec） |
| P0 | `contracts/evaluation_case.schema.json` | 评测用例结构（meta / inputs / ground_truth / known_issues / expected_issues） |
| P0 | `contracts/whitelist_registry.schema.json` | 白名单登记制 Schema（pattern / reason / scope / confirmed_by） |
| P0 | `src/canonical/models.py` | 数据类定义（Fact / SourceRef / Conflict / Criterion / Evaluation / ConclusionRule） |
| P0 | `src/canonical/fact_type_registry.py` | 受控注册表 |
| P0 | `src/canonical/store.py` | FactSet 容器 + 状态机 |
| P0 | `src/canonical/conflict.py` | Conflict 检测逻辑 |
| P0 | `src/canonical/exceptions.py` | 异常定义 |
| P0 | `pyproject.toml` 或 `requirements.txt` | 项目依赖声明 |
| P0 | `.gitignore` | 排除产物目录 |
| P1 | `src/canonical/__init__.py` | 包初始化 |

注意：**contracts/ 下的 5 份 JSON Schema 是 M0 的灵魂**。`02_ARCHITECTURE.md §17.2` 明确写："契约先行且必须单一一致，示例必须过校验器，否则后续模块会在不同版本的契约上分叉。"

---

# 七、第一批测试文件

| 文件 | 测试内容 |
|------|---------|
| `tests/canonical/test_fact_id_format.py` | 三段式 ID 格式校验、非法格式拒绝 |
| `tests/canonical/test_fact_status.py` | filled / missing / conflict / rejected / superseded 五态转换、禁止静默覆盖 |
| `tests/canonical/test_conflict.py` | 多来源一致 / 多来源不一致、Conflict 构造、manual resolution_policy |
| `tests/canonical/test_criterion_gate.py` | `review_status != confirmed` 时 Evaluation 必须为 `not_evaluable`、禁止输出方向性判定 |
| `tests/canonical/test_fact_type_registry.py` | scope_class / quantity_kind 枚举正确性、fact_type 不允许自由命名 |
| `tests/canonical/test_supersedes.py` | 更正链：旧 Fact → superseded，新 Fact 带 supersedes 指向 |

**测试原则**：所有测试基于 `pytest`，Mock 隔离外部依赖（当前无外部依赖），覆盖**文档中定义的红线规则**（如"AI 抽取值不在词表内 → pending"、"missing 状态不允许静默填值"）。

---

# 八、发现的风险与架构冲突

## 8.1 技术栈未明确（高风险）

**所有架构文档均未指定编程语言和构建体系。**

已知线索：
- `02_ARCHITECTURE.md §17.1` 提到了 `parsers/xlsx`、`render/docx`
- 渲染 Word 的主流选择是 `python-docx`（Python）或 `docx` npm 包（TypeScript）
- `05_EVALUATION.md` 的 JSON Schema 校验、`04_REPORT_IR.md` 的数字锚定正则提取，都对语言选择有隐性影响

**建议的技术栈决策点**：

| 选项 | 优势 | 劣势 |
|------|------|------|
| **Python + pydantic + pytest** | `python-docx` 成熟、`pydantic` Schema 校验强、数据科学生态适合工程计算、JSON Schema 实现方便 | 异步模型调用不如 Go/TS 优雅 |
| **Go** | 并发强、编译快、单文件部署 | 数据类定义冗余、Word 处理生态弱（docx-go 不成熟） |
| **TypeScript / Node** | JSON 原生友好、`docx` 包可用 | 数值精度处理（BigDecimal）较弱、单测生态不如 Python |

**推荐 Python**。理由：python-docx 是当前 Word 渲染唯一成熟选择，pydantic 能直接对接 JSON Schema 校验，pytest 覆盖测试生态完整。

## 8.2 Fact Schema 缺少显式 Schema 版本号（中风险）

`03_CANONICAL_DATA_MODEL.md` v0.3 没有给 Fact 定义显式的 `schema_version` 字段。而 `04_REPORT_IR.md §38` 明确要求 Report IR 必须带 `version`。这个不对称性可能导致未来 Schema 演进时难以做兼容性判断。

**建议**：在 M0 冻结 Fact Schema 时补上 `schema_version` 字段（如 `"v0.3"`），避免后续重构。

## 8.3 Evaluation / ConclusionRule 的强制字段未完整定义（中风险）

`03_CANONICAL_DATA_MODEL.md §26` 和 `§27.1` 给出了 Evaluation 和 ConclusionRule 的概念和示例，但没有像 Fact 和 Criterion 那样列出"强制字段表"。M0 冻结 Schema 时必须补齐，否则 contracts JSON Schema 无法编写。

**建议**：按 §25.1 Criterion 的格式，给 Evaluation 和 ConclusionRule 也补"强制字段表"。

## 8.4 项目名未定（低风险）

工作目录叫 `zixun4`，但没有正式包名。Python 项目名建议用下划线风格，如 `zixun_report` 或沿用 `zixun4`。

## 8.5 无 `.git` 初始化（低风险）

项目尚未版本控制。M0 创建文件后应立即 `git init`，否则无法利用 `07_DECISIONS.md` 的设计决策记录来追踪变更。

---

# 总结

| 维度 | 结论 |
|------|------|
| **当前状态** | 100% Greenfield，零代码零配置 |
| **第一阶段核心任务** | M0 冻结：5 份 JSON Schema + canonical 包的 5 个模块 |
| **可复用** | 无 |
| **冲突** | 无 |
| **最大风险** | 技术栈未明确 → 立即决策 Python |
| **文档缺口** | Evaluation / ConclusionRule 强制字段表、Fact schema_version |

以上全部为分析结论，**本轮未修改任何文件**。
