好的，这是一个严肃的工程实现会话。我先完整读取所有指定文档并检查项目状态，然后向你汇报确认。让我并行读取所有文档和检查 git 状态。
用户要求我开始 Step 7-B 工程实现，但首先要完整阅读所有设计文档并确认当前工作区状态。这是一个严肃的工程任务，我需要：

1. 先读取所有指定的设计文档
2. 检查 git 状态
3. 确认设计基线一致
4. 整理出实现范围和计划
5. 让用户确认后再动手

让我先并行读取所有需要的文档和检查项目状态。

现在读取开发约束文档并检查项目是否已有代码文件。
好的，我已经读完了所有核心设计文档，并确认了 git 状态。现在让我读取另外两个约束文档，然后检查当前有没有代码目录。

好的，现在我有了完整的项目上下文。让我整理一下：

**当前状态确认：**
- Git tag: `design-baseline-m0` ✓
- Commit: `f483072eb6253b9621e355c2649c6ea19f941684` ✓
- Branch: master
- Working tree: clean
- 项目目录: `g:\workspace\zixun4`
- 代码目录: **不存在** — 没有任何 `.py` / `.ts` / `go.mod` / `Cargo.toml` / `package.json` / `pyproject.toml`，这是纯设计基线
- 测试目录: 不存在

**设计文档版本确认：**
- 02_ARCHITECTURE.md v0.3
- 03_CANONICAL_DATA_MODEL.md v0.3
- 04_REPORT_IR.md v0.3
- 05_EVALUATION.md v0.3
- 07_DECISIONS.md v0.1

**Step 7-B 目标：**
只实现 Canonical Data Model 第一批数据底座：
- RawSource
- SourceRef
- Fact
- Missing
- Conflict

用户明确说"只实现这五个，不要实现其他"。让我仔细看一下 03_CANONICAL_DATA_MODEL.md 中这五个对象的具体字段定义。

现在我需要决定用什么语言。用户没有指定语言，但项目规则中有"程序负责确定性"的描述。让我看看用户的技术栈背景——从 user_profile.md 看，用户熟悉 Flutter/Dart、Kotlin、Go。

考虑到：
1. 这是数据模型层，强类型很重要
2. 需要 Schema/类型校验
3. 未来要跑测试
4. Go 语言在 Windows 上也有支持

但用户没有明确指定语言，我在汇报中应该建议并让用户确认。

让我整理汇报内容，然后向用户确认。

现在我需要调用 AdvisorTool 来确认技术方案，然后向用户汇报。

等等，用户要求我先告诉他：
# 一、当前设计基线
# 二、当前代码状态
# 三、Step 7-B 最小实现范围
# 四、准备创建的文件

然后等确认再实施。

让我整理好这些信息。

首先，让我再仔细过一遍 03_CANONICAL_DATA_MODEL.md 中这五个对象的定义，确保字段准确。

**RawSource（§3）**
- 表示系统接收到的原始输入资料
- 核心原则：保留原始内容、保留原始位置、不修改业务含义、Fact 可追溯到 RawSource
- 可以来自：工程文字资料、Excel/CSV、图片、PDF、Word、规范、人工输入、历史报告
- 有定位能力：文件、页码、段落、表格、Excel Sheet/单元格、图片 ID/区域

文档没有给 RawSource 的具体 JSON Schema，只有概念描述。需要我设计合理的字段结构。

**SourceRef（§5）**
- 用于建立事实与原始资料之间的可追溯关系
- 文档示例：
```json
{
  "source_id": "inspection-001.xlsx",
  "location": "Sheet1!F23"
}
```

字段：`source_id`（string，必选）、`location`（string，必选）

**Fact（§6-16）**
- 最小断言单元，只描述"是什么"
- 强制字段（§8）：
  - `id`：三段式 `fact:<scope_class>.<instance_key>.<attribute_path>`（v0.3）
  - `fact_type`：受控注册表中的类型路径（v0.3 新增）
  - `value` / `unit` / `quantity_kind`
  - `source_refs[]`
  - `method`：`measured` / `observed` / `computed` / `quoted` / `inferred`
  - `provenance`：`program` / `ai` / `human`
  - `confidence`：0-1，诊断字段
  - `review_status`：`pending` / `confirmed` / `rejected`
  - `status`：`filled` / `missing` / `conflict` / `rejected` / `superseded`（v0.3 新增 superseded）
  - 可选：`supersedes[]`（更正语义 v0.3）、`value_domain`（定性 Fact）

**Missing（§15.2 + §37）**
- missing 不是一个独立数据结构，而是 Fact 的一种 status
- `status = missing` 的 Fact，value 通常为 null
- 文档示例：
```json
{
  "id": "fact:building.year",
  "value": null,
  "unit": null,
  "status": "missing"
}
```

**Conflict（§33-36）**
- 同一事实存在多来源不一致时，生成 Conflict 记录
- 数据结构（§34）：
```
Conflict {
    fact_id,
    candidates: [
        {
            value,
            source_ref,
            provenance
        }
    ],
    resolved_by,
    resolution
}
```
- v0.2 新增：`resolution_policy` 字段（第一阶段固定为 `manual`）
- 文档示例（§35）：
```json
{
  "fact_id": "fact:building.B01.area",
  "candidates": [
    {
      "value": 1250.30,
      "unit": "m²",
      "quantity_kind": "area",
      "source_ref": "design.pdf#page=8",
      "method": "quoted",
      "provenance": "program"
    },
    {
      "value": 1280.00,
      "unit": "m²",
      "quantity_kind": "area",
      "source_ref": "inspection.xlsx#Sheet1!F23",
      "method": "quoted",
      "provenance": "program"
    }
  ],
  "resolution_policy": "manual",
  "resolved_by": null,
  "resolution": null
}
```

注意 Conflict 的 candidates 中每个候选的字段与 Fact 不完全相同——它是候选 Fact 的快照，包含 value、unit、quantity_kind、source_ref、method、provenance。

好，现在让我整理汇报。
好，所有上下文已完整收集。现在整理汇报。

---

# 一、当前设计基线

| 项 | 状态 |
| ---- | ---- |
| Git tag | `design-baseline-m0` ✓ |
| Commit | `f483072eb6253b9621e355c2649c6ea19f941684` ✓ |
| Branch | `master` |
| Working tree | clean |
| 设计文档版本 | 02/03/04/05 均为 **v0.3**，07 为 v0.1 |
| 文档与 tag 一致性 | ✓ 已核对，无漂移 |

---

# 二、当前代码状态

| 项 | 状态 |
| ---- | ---- |
| `.py` 文件 | 0 |
| `.ts` / `package.json` | 0 |
| `go.mod` | 0 |
| `Cargo.toml` | 0 |
| `pyproject.toml` | 0 |
| 测试目录 | 不存在 |
| 任何代码目录 | 不存在 |

**结论：当前是纯设计基线，没有任何一行工程代码。**

---

# 三、Step 7-B 最小实现范围

严格按照 `03_CANONICAL_DATA_MODEL.md v0.3`，只实现以下 5 个对象：

| 对象 | 文档章节 | 核心字段 |
| ---- | ---- | ---- |
| **RawSource** | §3-4 | 原始文件记录（source_id / source_type / location 定位） |
| **SourceRef** | §5 | source_id + location（Fact 溯源链的锚点） |
| **Fact** | §6-16 | 三段式 id / fact_type / value·unit·quantity_kind / source_refs[] / method / provenance / confidence / review_status / status / 可选 supersedes / value_domain |
| **Conflict** | §33-36 | fact_id + candidates[]（每个候选带 value/unit/quantity_kind/source_ref/method/provenance）+ resolution_policy + resolved_by + resolution |
| **Missing** | §15.2 / §37 | 不是独立结构体，是 `Fact.status = "missing"` 的状态。不单独建类，而是 Fact 类本身支持 missing 状态 |

**本轮明确不实现：**
- fact_type 受控注册表（§8.1 的注册表概念本轮只做类型声明，不做注册表管理）
- quantity_kind 注册表（同上）
- Domain Objects（Project/Component/Criterion/Evaluation/Conclusion 等，§17-28 全部跳过）
- ConclusionRule
- Report IR
- DOCX / PDF
- Validator
- LLM Adapter / Skill / Agent
- 任何解析器（xlsx/docx/pdf/text）
- 任何计算模块

**必须实现的能力：**
1. 强类型数据结构（class / struct / dataclass）
2. JSON 序列化 / 反序列化
3. Schema / 字段校验（必填检查、枚举检查、格式检查、ID 格式三段式检查）
4. 正常路径测试 + 失败路径测试
5. 可运行的测试框架

---

# 四、准备创建的文件（建议 Go 语言，原因见下方）

### 技术选型建议：Go

| 考虑因素 | 选择 |
| ---- | ---- |
| 强类型数据模型 | Go struct + 显式类型 > Python dataclass |
| Windows 环境 | Go 原生支持良好 |
| JSON 编解码 | 标准库 `encoding/json`，零依赖 |
| 测试框架 | 标准库 `testing` |
| 无依赖、最小闭环 | 不需要任何第三方库 |
| 用户已有 Go 经验 | user_profile 标注 Go 熟练 |
| 未来扩展性 | 纯程序模块，不绑定任何 Web 框架 |

### 目录结构

```
g:\workspace\zixun4\
├── go.mod                          # go mod init zixun4
├── internal/
│   └── cdm/                        # Canonical Data Model 包
│       ├── types.go                # SourceRef, RawSource, Fact, Conflict 结构体定义 + 枚举常量
│       ├── id.go                   # 三段式 Fact ID 解析 / 生成 / 校验
│       ├── validate.go             # 各对象的 Validate() 方法
│       └── registry.go             # fact_type / quantity_kind / scope_class / method / provenance / status 等受控注册表（纯常量列表）
└── internal/
    └── cdm/
        └── types_test.go           # 所有测试（正常 + 失败路径）
```

### 每个文件的职责

| 文件 | 内容 |
| ---- | ---- |
| `go.mod` | module zixun4，go 1.22+ |
| `types.go` | 所有结构体 + 所有枚举常量定义 |
| `id.go` | `ParseFactID()` / `BuildFactID()` / `IsValidFactID()` |
| `validate.go` | 每个结构体的 `Validate() error` |
| `registry.go` | 受控词表：scope_class / method / provenance / fact_status / review_status / quantity_kind（当前先放枚举定义 + 校验函数，fact_type 注册表本轮只放框架） |
| `types_test.go` | 覆盖每个对象的正常构造 + 校验失败场景 + JSON round-trip |

### 测试覆盖计划

| 对象 | 正常测试 | 失败路径测试 |
| ---- | ---- | ---- |
| SourceRef | 构造 / JSON round-trip | source_id 为空 / location 为空 |
| RawSource | 构造 / JSON round-trip | source_id 为空 / source_type 非法 |
| Fact | filled / missing / conflict / rejected / superseded 各状态构造 / 定性 Fact / computed Fact / JSON round-trip | id 非三段式 / fact_type 为空 / status 非法 / method 非法 / review_status 非法 / missing 状态却有 value / superseded 缺 supersedes |
| Conflict | 双来源冲突 / 单候选 / resolution_policy | fact_id 为空 / candidates 为空 / resolution_policy 非 manual |

---

# 需要你确认

1. **语言选择**：用 Go 是否可以？还是你倾向其他？
2. **目录结构**：`internal/cdm/` 这个路径层级是否 OK？
3. **Registry 范围**：fact_type 注册表本轮只放框架（空 map + 校验入口），不填具体类型，是否 OK？
4. **Go 版本**：go.mod 声明哪个版本？

确认后我再开始创建文件和写测试。
