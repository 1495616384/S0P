可以。下面这三份我已经按你 GitHub `README.md` 当前内容重新整理成**可以直接复制覆盖**的版本：不重复 README、保持三个文件职责边界，并把目前已经形成的架构结论完整沉淀下来。README 中明确了系统边界、四层数据模型、Report IR、Agent/Skill、AI/程序职责以及 LLM Adapter 等内容。([GitHub][1])

> 建议现在把这三个文件都标成 **Draft / v0.1**，因为这是第一次完整设计后的持久化版本，后面还会继续迭代。

---

# `docs/02_ARCHITECTURE.md`

````md
# 系统架构设计

> 状态：Draft
>
> 版本：v0.1
>
> 本文档定义系统级架构、模块边界、数据流、Agent / Skill 边界，以及 AI 与程序的职责边界。
>
> 本文档不定义 Canonical Data Model 的完整字段结构，也不定义 Report IR 的完整 Schema。对应内容分别见：
>
> - `03_CANONICAL_DATA_MODEL.md`
> - `04_REPORT_IR.md`

---

## 1. 系统目标

本项目不是简单的“AI 自动写报告”，而是构建一个可重复、可验证、可追溯的工程报告生产系统。

系统接收：

- 工程文字资料
- 标准规范
- 数字检测 / 监测数据
- 图片
- 表格
- 固定样式的 Word 报告模板
- 历史真实报告（用于构造评测用例）

系统最终输出：

- 成品 DOCX
- 成品 PDF
- 中间结构化数据
- 验证结果
- 问题清单
- 回归测试结果

系统核心目标：

```text
Inputs
  ↓
Facts / Domain Objects
  ↓
Reasoning
  ↓
Report IR
  ↓
Template Rendering
  ↓
DOCX / PDF
  ↓
Validation
````

---

## 2. 核心问题

系统需要同时解决以下几个核心矛盾。

### 2.1 不确定性与确定性

LLM 属于概率生成系统，而工程报告中的数字、计算结果、限值判断和引用必须可验证、可追溯。

因此：

> AI 负责理解、抽取、组织；程序负责确定性计算、校验和渲染。

---

### 2.2 语义灵活与版式刚性

报告文本需要一定的自然语言组织能力，但模板中的：

* 字体
* 字号
* 行距
* 表格
* 编号
* 页码
* 页眉页脚
* 章节结构

必须保持稳定。

因此：

> 报告语义表示与 Word 模板表示必须解耦。

---

### 2.3 生成与验证

系统不仅需要能够生成报告，还必须能够证明：

* 数值是否正确
* 表格是否正确
* 结论是否与数据一致
* 引用是否存在
* 结构是否符合要求
* 样式是否满足模板要求

因此：

> Validation 是核心模块，而不是生成完成之后可有可无的附加功能。

---

### 2.4 快速迭代与回归成本

模型、Prompt、Skill、程序和模板都会不断变化。

因此：

> 每次变化都必须能够通过固定 Evaluation Case 重新运行并与基线比较。

---

## 3. 系统边界

### 3.1 第一阶段范围内

第一阶段系统覆盖：

1. 单份报告从输入资料到 DOCX / PDF 的完整生成流程。
2. 输入资料解析与标准化。
3. Facts / Domain Objects 构建。
4. 确定性计算。
5. Agent / Skill 协同。
6. Report IR 构建。
7. 模板渲染。
8. Accuracy Validation。
9. Style Validation。
10. 历史报告反向构造 Evaluation Case。
11. Regression Test。

---

### 3.2 第一阶段范围外

第一阶段暂不实现：

* OA / 审批流程集成
* 多人协同编辑
* 现场采集 APP
* 检测设备直接对接
* 报告归档与检索平台
* 完整知识库
* 全量规范向量数据库
* 全自动无人值守发布
* 模型训练 / 微调
* 所有报告类型的一次性覆盖
* 图形化报告编辑器

---

## 4. 核心架构原则

### 4.1 系统不产生事实

系统只允许：

* 搬运事实
* 提取事实
* 计算事实
* 组织事实
* 引用事实

任何不能从输入资料或确定性计算中得到的数值，不允许由 LLM 自行补全。

缺失时必须显式表示为：

```text
missing
```

而不能编造。

---

### 4.2 AI 与程序职责分离

核心原则：

> 程序决定“是什么、对不对”；AI 决定“怎么理解、怎么组织”。

凡是可以用：

* 规则
* 公式
* 表格
* 确定性算法

表达的处理，都优先程序化。

---

### 4.3 中间结果全部结构化

各阶段之间不通过自然语言长文本传递业务状态。

核心中间产物使用 JSON 等结构化格式表示。

主要中间对象：

```text
RawSource
Fact
Domain Object
Report IR
IssueList
Evaluation Case
Run Snapshot
```

---

### 4.4 Agent 不直接写最终报告

Agent 负责：

* 理解任务
* 判断当前状态
* 选择 Skill
* 调度 Skill
* 处理缺失
* 协调流程
* 触发验证
* 必要时进行有限自修复

Agent 不直接控制 Word 排版，也不直接决定最终数值。

---

### 4.5 Skill 必须是可测试能力

Skill 必须具有：

* 明确输入
* 明确输出
* Schema
* 版本号
* 测试用例
* 可替换实现

Skill 应能够单独测试。

---

### 4.6 所有报告数字必须可追溯

报告中出现的任何数字，都必须能够沿以下链路追溯：

```text
Report
  ↓
Report IR Ref
  ↓
Domain Object / Fact
  ↓
SourceRef
  ↓
原始输入
```

---

## 5. 总体架构

```text
┌──────────────────────────────────────────────┐
│                  Input Layer                 │
│                                              │
│ 文本 / Excel / CSV / 图片 / PDF / 规范 / 模板 │
└──────────────────────┬───────────────────────┘
                       ↓
┌──────────────────────────────────────────────┐
│              Input Normalization             │
│                                              │
│ 文档解析 / Excel解析 / 图片理解 / 来源定位    │
└──────────────────────┬───────────────────────┘
                       ↓
┌──────────────────────────────────────────────┐
│           Canonical Data Model              │
│                                              │
│ RawSource → Facts → Domain Objects           │
└──────────────────────┬───────────────────────┘
                       ↓
┌──────────────────────────────────────────────┐
│              Agent Runtime                  │
│                                              │
│ Intake / Extraction / Reasoning / Composer   │
└──────────────────────┬───────────────────────┘
                       ↓
┌──────────────────────────────────────────────┐
│                  Skills                      │
│                                              │
│ Deterministic Skills + LLM Skills            │
└──────────────────────┬───────────────────────┘
                       ↓
┌──────────────────────────────────────────────┐
│                 Report IR                   │
│                                              │
│ Document → Section → Block → Inline          │
└──────────────────────┬───────────────────────┘
                       ↓
┌──────────────────────────────────────────────┐
│             Template / Renderer             │
│                                              │
│ Semantic Mapping → Word Template → DOCX      │
└──────────────────────┬───────────────────────┘
                       ↓
                 DOCX / PDF
                       │
                       ↓
┌──────────────────────────────────────────────┐
│                 Validation                   │
│                                              │
│ Accuracy Validator + Style Validator         │
└──────────────────────┬───────────────────────┘
                       ↓
┌──────────────────────────────────────────────┐
│          Evaluation / Regression            │
│                                              │
│ Metrics / IssueList / Baseline / Diff        │
└──────────────────────────────────────────────┘
```

---

## 6. 输入层

### 6.1 输入类型

| 输入            | 常见形态         | 主要处理者           | 标准化目标                    |
| --------------- | ---------------- | -------------------- | ----------------------------- |
| 工程文字资料    | DOCX / PDF / TXT | AI + 程序 + 人工确认 | ProjectMeta / Narrative Facts |
| 检测 / 监测数据 | XLSX / CSV       | 程序                 | Measurement / MeasurementSet  |
| 现场图片        | JPG / PNG        | 多模态 AI + 程序     | Evidence / Image Facts        |
| 标准规范        | PDF / DOCX       | 程序分块 + AI理解    | RuleRef / Clause              |
| Word模板        | DOCX             | 程序                 | TemplateSpec                  |
| 历史真实报告    | DOCX / PDF       | 逆向抽取 + 人工校验  | Evaluation Case               |

---

### 6.2 SourceRef

所有解析结果必须尽可能关联来源定位信息。

来源定位至少应支持：

* 文件名
* 文件 ID
* 页码
* 表格位置
* Excel Sheet
* Excel 单元格
* 段落位置
* 图片 ID
* 其他可定位信息

示意：

```json
{
  "source_id": "inspection-001.xlsx",
  "location": "Sheet1!F23"
}
```

---

### 6.3 输入仪表化

所有重要解析结果都应该产生 JSON 中间文件。

目的：

* 可人工检查
* 可版本控制
* 可作为测试 Fixture
* 可回归
* 可定位错误来源

---

### 6.4 人工确认闸门

第一阶段不追求完全无人值守。

尤其是 AI 抽取结果，需要存在人工确认节点。

第一阶段可以不做 UI，直接通过人工编辑 / 审核 JSON 完成。

---

## 7. Canonical Data Model 与 Report IR 的边界

系统使用四层模型：

```text
L1 RawSource
    ↓
L2 Facts
    ↓
L3 Domain Objects
    ↓
L4 Report IR
```

其中：

* `RawSource`：原始来源
* `Facts`：事实
* `Domain Objects`：工程领域对象
* `Report IR`：报告结构表示

详细定义见：

```text
docs/03_CANONICAL_DATA_MODEL.md
docs/04_REPORT_IR.md
```

关键边界：

> Facts 只表示“是什么”，不表示“报告应该怎么写”。

---

## 8. Agent / Skill 架构

### 8.1 Agent 定义

Agent 是流程编排者。

Agent：

* 有目标
* 有状态
* 能选择 Skill
* 能根据结果决定下一步
* 可以进行有限循环

Agent 不直接代替 Skill 完成具体能力。

---

### 8.2 第一阶段 Agent

#### IntakeAgent

职责：

* 判断输入类型
* 建立输入任务
* 调度解析 Skill
* 标记缺失
* 建立 SourceRef

---

#### ExtractionAgent

职责：

* 从文字中抽取 Facts
* 从图片中提取 Evidence
* 对非结构化资料进行结构化

---

#### ReasoningAgent

职责：

* 基于 Facts 与规范进行工程推理
* 执行或调用判定能力
* 形成结论事实
* 形成结论要点

限制：

> 不直接输出最终报告版式文本。

---

#### ReportComposerAgent

职责：

* 根据报告 IR 骨架组织内容
* 将 Facts 引入 Report IR
* 产生结构化 Report IR

---

#### ValidatorAgent

职责：

* 调用 Accuracy Validator
* 调用 Style Validator
* 形成 IssueList
* 根据问题进行有限次数的修复循环

---

### 8.3 Agent 间通信

Agent 之间禁止依赖自然语言作为业务数据接口。

应传递结构化对象：

```text
Facts
Domain Objects
Report IR
IssueList
```

目的：

* 避免误差累积
* 避免幻觉传播
* 保证接口稳定
* 便于测试

---

## 9. Skill 架构

### 9.1 确定性 Skill

由程序实现，不依赖 LLM。

示例：

```text
parse_xlsx
unit_convert
calc_statistics
apply_rule
render_docx
extract_docx_template
validate_ir_schema
diff_report
extract_text_from_pdf
```

---

### 9.2 LLM Skill

处理不确定性映射。

示例：

```text
extract_fields_from_text
extract_fields_from_image
classify_document
draft_conclusion_points
summarize_evidence
```

所有 LLM Skill 必须具有：

* input schema
* output schema
* few-shot
* 低温策略
* 重试策略
* Schema 校验

---

### 9.3 Skill 元数据

Skill 至少定义：

```text
id
version
kind
input_schema
output_schema
cost
latency
test_cases
```

其中：

```text
kind ∈ { deterministic, llm }
```

---

## 10. LLM Adapter

上层业务不直接绑定具体模型。

统一接口思想：

```text
LLMClient.generate(
    messages,
    schema,
    temperature,
    max_tokens
) -> StructuredResult
```

---

### 10.1 Adapter 职责

Adapter 负责：

* 统一模型调用方式
* 结构化输出适配
* JSON 提取
* 输出修复
* Retry
* Schema 校验
* Trace 记录

---

### 10.2 Model Profile

模型配置外置：

```text
provider
model
base_url
params
context_limit
cost
```

---

### 10.3 Capability Routing

上层不应该依赖具体模型名称，而依赖能力：

```text
text_extraction
vision
long_context
json_mode
```

由 Adapter / Routing 层决定具体模型。

---

### 10.4 Trace

每次调用应尽量记录：

```text
prompt_hash
token_usage
latency
model
model_profile
raw_response
```

用于：

* 调试
* 回归
* A/B
* 问题定位

---

### 10.5 第一阶段不做的事情

不要一开始构建复杂的万能 Provider Framework。

第一阶段只需要：

```text
一个薄 Adapter
+
一个模型配置文件
+
一个 Trace 记录机制
```

---

## 11. AI 与程序的责任边界

### 11.1 必须程序化

以下任务应优先程序实现：

* 数值运算
* 单位换算
* 统计计算
* 限值判断
* 规则判断
* 表格生成
* 编号
* 目录
* 页码
* Word 渲染
* 样式检查
* 交叉引用检查

---

### 11.2 允许 AI 处理

AI 主要处理：

* 非结构化文字理解
* 图片理解
* 字段抽取
* 文档分类
* 证据总结
* 结论要点组织
* 受控语言组织

---

### 11.3 数字红线

任何出现在最终报告中的数字：

```text
必须来自 Facts
    ↓
必须存在 SourceRef 或确定性计算依据
```

禁止：

```text
LLM 自行生成数字
```

---

### 11.4 人工红线

第一阶段保留人工参与：

* AI 抽取结果确认
* 关键事实确认
* 最终结论签发

---

## 12. 历史报告与 Evaluation Architecture

历史报告不是普通训练资料。

它们首先用于构造：

```text
Evaluation Case
```

整体流程：

```text
历史真实报告
    ↓
逆向抽取
    ↓
Facts
    ↓
人工校验
    ↓
Evaluation Case
    ↓
系统重新生成
    ↓
Validation
    ↓
Regression
```

需要特别区分：

> 历史报告中“声称的事实”不等于现实世界中的客观事实。

详细 Evaluation Case 和验证体系后续单独设计。

---

## 13. Validation 在架构中的位置

验证不是报告生成后的附属功能，而是主流程的一部分：

```text
Generate
   ↓
Validate
   ↓
IssueList
   ↓
Fix / Reject
   ↓
Final Output
```

第一阶段重点关注：

### Accuracy

* Schema / 完整性
* 数据一致性
* 计算一致性
* 结论方向一致性
* Ground Truth 对比

### Style

* 字体
* 字号
* 行距
* 缩进
* 页边距
* 页眉页脚
* 页码
* 表格
* 编号
* 章节结构

详细规则后续单独沉淀到 Evaluation 文档。

---

## 14. 关键架构决策

### D1. Report IR 先于模板渲染

先建立稳定的 Report IR，再实现 Word Renderer。

---

### D2. 报告正文不能是自由字符串

Paragraph 使用结构化 Segment 表示：

```text
Lit + Ref
```

---

### D3. 报告数字必须可追溯

所有数字都必须能够从 Report IR 追溯到 Fact 和 SourceRef。

---

### D4. Agent 之间使用结构化数据

禁止将自然语言作为 Agent 间核心业务接口。

---

### D5. 历史报告作为 Ground Truth / Evaluation Case

历史报告首先用于评测与回归，而不是直接作为“真理”。

---

### D6. 正确地失败也是系统能力

遇到：

* 缺失数据
* 数据冲突
* 单位异常
* 超限值
* 无法判断的问题

系统应该：

```text
显式报告问题
```

而不是偷偷补全。

---

### D7. 先做无 LLM 骨架

系统应首先做到：

```text
输入
→ Facts
→ IR
→ 模板渲染
→ 输出
```

即使没有 LLM，也能够形成最小可运行闭环。

---

### D8. 采用分层验证

优先验证：

```text
P0 数值 / 结论
P1 结构
P2 文本
P3 版式
```

具体阈值由 Evaluation 文档定义。

---

### D9. LLM Adapter 与业务解耦

业务逻辑只依赖能力，不直接绑定模型名称。

---

### D10. 中间产物结构化

中间阶段优先使用 JSON，便于：

* 调试
* 回归
* 人工检查
* 版本控制

---

### D11. IR 与模板解耦

语义层与版式层分离。

---

### D12. 第一阶段优先 CLI + JSON

第一阶段不优先建设图形化前端。

---

## 15. 第一阶段明确不做

暂不建设：

* 微服务架构
* Kubernetes
* Redis
* MQ
* 全量向量数据库
* 全量 RAG
* Multi-Agent 自治协商
* 知识图谱
* 模型训练 / 微调
* 自研 Word 排版引擎
* 图形化编辑器
* 全自动无人值守生成
* 一次性支持所有报告类型

---

## 16. 架构演进原则

架构设计允许后续演进，但以下核心接口应保持稳定：

```text
RawSource
Facts
Domain Objects
Report IR
Skill Schema
IssueList
Evaluation Case
```

模型可以替换。

Skill 可以替换。

模板可以替换。

Renderer 可以替换。

但不应因为替换模型而破坏整个系统的数据与验证体系。

---

## 17. 与其他设计文档的关系

```text
02_ARCHITECTURE.md
    │
    ├── 定义系统如何运行
    │
    ├───────────────┐
    ↓               ↓
03_CANONICAL...   04_REPORT_IR.md
    │               │
    │ 定义事实      │ 定义报告表示
    │               │
    └───────┬───────┘
            ↓
       Template Renderer
            ↓
          DOCX/PDF
```

````

---

# `docs/03_CANONICAL_DATA_MODEL.md`

```md
# Canonical Data Model

> 状态：Draft
>
> 版本：v0.1
>
> 本文档定义系统内部对“工程事实”的统一表示方式。
>
> 本文档回答：
>
> > 输入资料进入系统之后，系统如何用统一的数据模型表示事实、工程对象、证据、判定和来源？
>
> 本文档不负责定义报告的章节、段落、表格和 Word 样式。报告表示由：
>
> - `04_REPORT_IR.md`
>
> 定义。

---

## 1. 设计目标

Canonical Data Model 的目标：

1. 统一不同输入来源。
2. 保留原始来源。
3. 保证事实可追溯。
4. 区分事实与语言表达。
5. 支持确定性计算。
6. 支持 AI 抽取。
7. 支持人工校验。
8. 支持冲突处理。
9. 支持 Report IR 引用。
10. 支持 Evaluation / Regression。

---

## 2. 四层模型

系统不采用一个巨大的“大统一对象”。

采用四层模型：

```text
L1 RawSource
    ↓
L2 Facts
    ↓
L3 Domain Objects
    ↓
L4 Report IR
````

---

## 3. L1：RawSource

RawSource 表示系统所收到的原始资料。

特点：

* 原始
* 不修改
* 可定位
* 可追溯
* 原则上只读

---

### 3.1 RawSource 类型

可能包括：

```text
Document
Spreadsheet
Image
PDF
Template
Standard
ManualInput
HistoricalReport
```

---

### 3.2 RawSource 示例

```json
{
  "id": "source-001",
  "type": "spreadsheet",
  "name": "inspection.xlsx",
  "hash": "sha256:...",
  "location": {
    "sheet": "Sheet1",
    "cell": "F23"
  }
}
```

---

## 4. SourceRef

SourceRef 用于描述事实来自哪里。

---

### 4.1 设计要求

SourceRef 应尽可能提供精确定位。

可以定位到：

* 文件
* 页码
* 段落
* 表格
* Sheet
* 单元格
* 图片
* 图片区域
* 原始记录编号

---

### 4.2 SourceRef 示例

```json
{
  "source_id": "inspection-001.xlsx",
  "location": "Sheet1!F23"
}
```

PDF 示例：

```json
{
  "source_id": "project-info.pdf",
  "location": "page:12"
}
```

图片示例：

```json
{
  "source_id": "photo-034.jpg",
  "location": "full-image"
}
```

---

## 5. L2：Facts

Fact 是系统中的最小断言单元。

核心定义：

> Fact 描述“是什么”，而不是“应该怎样表达”。

例如：

```text
工程面积 = 1250.30 m²
```

是 Fact。

下面这种不是 Fact：

```text
本工程建筑面积较大，约为1250平方米。
```

后者属于报告语言，应进入 Report IR。

---

## 6. Fact 的核心属性

每个 Fact 应至少具备以下信息：

```text
id
value
unit
source_refs[]
method
provenance
confidence
status
```

---

### 6.1 id

Fact 的唯一标识。

示例：

```text
fact:building.area
fact:component.K001.concrete_strength
fact:crack.C001.width
```

---

### 6.2 value

事实值。

可以是：

* 数值
* 字符串
* 布尔值
* 枚举
* 日期
* 对象
* 数组

---

### 6.3 unit

用于描述物理量的单位。

例如：

```text
m
m²
mm
MPa
kN
%
```

没有单位的离散事实可以为：

```text
null
```

---

### 6.4 source_refs

事实来源。

示例：

```json
[
  {
    "source_id": "inspection.xlsx",
    "location": "Sheet1!F23"
  }
]
```

---

### 6.5 method

描述事实如何获得。

建议使用：

```text
observed
measured
calculated
quoted
inferred
```

含义：

| method     | 含义             |
| ---------- | ---------------- |
| observed   | 观察得到         |
| measured   | 实测得到         |
| calculated | 程序计算得到     |
| quoted     | 从资料直接引用   |
| inferred   | 基于其他信息推导 |

---

### 6.6 provenance

记录事实的产生来源。

例如：

```text
human
program
ai
```

---

### 6.7 confidence

用于描述抽取或推断结果的可信度。

尤其适用于 AI 产生的事实。

示例：

```json
{
  "confidence": 0.92
}
```

确定性程序计算结果可根据系统约定使用高置信度表示。

---

### 6.8 status

事实状态：

```text
filled
missing
conflict
```

含义：

* `filled`：已获得有效值
* `missing`：无法得到
* `conflict`：不同来源存在冲突

---

## 7. Fact 示例

### 7.1 Excel 实测事实

```json
{
  "id": "fact:component.K001.concrete_strength",
  "value": 32.4,
  "unit": "MPa",
  "source_refs": [
    {
      "source_id": "inspection.xlsx",
      "location": "Sheet1!F23"
    }
  ],
  "method": "measured",
  "provenance": "program",
  "confidence": 1.0,
  "status": "filled"
}
```

---

### 7.2 AI 图片识别事实

```json
{
  "id": "fact:photo.034.crack.present",
  "value": true,
  "unit": null,
  "source_refs": [
    {
      "source_id": "photo-034.jpg",
      "location": "full-image"
    }
  ],
  "method": "observed",
  "provenance": "ai",
  "confidence": 0.91,
  "status": "filled"
}
```

---

### 7.3 计算事实

```json
{
  "id": "fact:building.area.total",
  "value": 1250.30,
  "unit": "m²",
  "source_refs": [
    {
      "source_id": "area-calculation"
    }
  ],
  "method": "calculated",
  "provenance": "program",
  "confidence": 1.0,
  "status": "filled"
}
```

---

## 8. L3：Domain Objects

Domain Object 是对工程领域实体的组织。

其目的：

> 将大量离散 Facts 组织成工程上可以理解和操作的对象。

---

## 9. Domain Object 通用骨架

第一阶段采用以下通用骨架：

```text
Project
Client
Structure
Component
InspectionItem
Measurement
MeasurementSet
Criterion
Evaluation
Conclusion
Evidence
```

---

## 10. Project

表示工程项目。

示例属性：

```text
id
project_name
project_no
location
client_ref
structure_ref
```

注意：

Project 本身不是事实数据库，而是事实的组织容器。

---

## 11. Client

表示委托方或相关主体。

可能包括：

```text
id
name
role
source_refs[]
```

---

## 12. Structure

表示建筑物、结构体系或结构层级。

可表达：

```text
建筑
楼层
轴线
区域
结构单元
```

---

## 13. Component

表示具体结构 / 构件。

例如：

```text
柱
梁
板
墙
基础
```

通常具有：

```text
id
component_no
type
location
source_refs[]
```

---

## 14. InspectionItem

表示检测项目。

例如：

```text
混凝土强度
钢筋保护层
裂缝
沉降
倾斜
```

---

## 15. Measurement / MeasurementSet

### Measurement

表示单个测点 / 单次测值。

例如：

```text
构件：K001
测值：32.4
单位：MPa
方法：回弹
```

---

### MeasurementSet

表示一组同类测量结果。

例如：

```text
某楼层全部柱子的混凝土强度检测结果
```

MeasurementSet 可以支持：

* 最大值
* 最小值
* 平均值
* 统计值
* 样本数量
* 异常值

这些统计结果应优先通过确定性程序计算。

---

## 16. Criterion

Criterion 表示判定依据。

例如：

```text
规范条文
限值
适用条件
```

其内容应该能够追溯至相应标准来源。

---

## 17. Evaluation

Evaluation 表示基于事实和 Criterion 得到的判定结果。

例如：

```text
合格
不合格
等级
满足条件
不满足条件
```

Evaluation 应尽量保留：

```text
输入 Fact
引用 Criterion
判定结果
判定方法
```

从而支持反向解释。

---

## 18. Conclusion

Conclusion 表示工程结论。

第一阶段应区分：

```text
结论事实
```

与：

```text
最终报告语言
```

例如：

```text
结论事实：
assessment = qualified
```

而：

```text
经综合评定，该项目满足……
```

属于 Report IR 中的语言组织。

---

## 19. Evidence

Evidence 表示支撑工程判断的证据。

可能包括：

```text
图片
原始检测记录
测量记录
报告附件
规范条文
```

Evidence 也必须可追溯。

---

## 20. Domain Object 的公共元数据

实体原则上应支持：

```text
id
source_refs[]
method
provenance
confidence
status
```

从而保持整个数据模型的一致性。

---

## 21. Missing

系统不能获取某个事实时：

```text
status = missing
```

而不是：

```text
value = AI猜测值
```

---

### 21.1 Missing 的意义

Missing 是合法状态，不是系统异常。

例如：

```json
{
  "id": "fact:building.year",
  "value": null,
  "status": "missing"
}
```

报告生成阶段应根据业务规则决定：

* 是否继续生成
* 是否提示人工
* 是否阻止最终输出

---

## 22. Conflict

当同一个事实存在多个来源且不一致时：

```text
Fact
  ↓
Conflict
```

系统不得静默覆盖。

---

### 22.1 Conflict 示例

```json
{
  "fact_id": "fact:building.area",
  "candidates": [
    {
      "value": 1250.30,
      "source_ref": "source-A"
    },
    {
      "value": 1280.00,
      "source_ref": "source-B"
    }
  ],
  "resolved_by": null,
  "resolution": null
}
```

---

### 22.2 Conflict 处理原则

必须明确：

```text
哪个来源可信
为什么
谁解决
解决结果是什么
```

可以由：

* 人
* 程序规则
* 明确业务规则

解决。

不允许：

```text
LLM 自行选择一个然后隐藏冲突
```

---

## 23. Fact 与语言表达的边界

必须保持以下边界：

```text
Fact
=
工程世界中的断言

Report IR
=
如何把断言表达给读者
```

例如：

```text
Fact：
concrete_strength = 32.4 MPa
```

可以表达成：

```text
实测混凝土强度为32.4MPa。
```

或者：

```text
经检测，该构件混凝土强度实测值为32.4MPa。
```

两者都是不同的语言表达，但引用的是同一个 Fact。

---

## 24. Fact 与报告事实的语义边界

需要特别区分：

```text
“历史报告中声称的事实”
```

与：

```text
“现实世界的客观事实”
```

历史报告反向抽取时得到的是前者。

因此：

```text
Historical Report
    ↓
Reverse Extraction
    ↓
Claimed Facts
```

这些数据可以作为：

```text
Evaluation Ground Truth
```

但不能自动被视为现实世界的绝对真值。

---

## 25. LLM 产生 Fact 的约束

LLM 可以负责：

```text
非结构化
    ↓
结构化 Fact
```

但输出必须：

1. 满足 Schema。
2. 带 SourceRef。
3. 带 provenance。
4. 带 confidence。
5. 可人工确认。
6. 可进入测试用例。

---

## 26. Program 产生 Fact 的约束

程序可以产生：

```text
计算事实
统计事实
规则判定事实
```

程序产生的 Fact 同样需要：

```text
method
provenance
source_refs
```

确保可以追踪计算来源。

---

## 27. 与 Report IR 的关系

完整关系：

```text
RawSource
    ↓
Fact
    ↓
Domain Object
    ↓
Report IR
```

Report IR 不应该重新复制一整份工程事实。

而应该：

```text
Ref → 引用 Canonical Data Model
```

例如：

```text
fact:building.area
fact:component.K001.concrete_strength
```

---

## 28. 设计目标总结

Canonical Data Model 最终要实现：

```text
统一表示
+
来源追踪
+
冲突显式化
+
缺失显式化
+
AI / 程序可共同使用
+
可被 Report IR 引用
+
可被 Evaluation 验证
```

````

---

# `docs/04_REPORT_IR.md`

```md
# Report IR

> 状态：Draft
>
> 版本：v0.1
>
> Report IR（Report Intermediate Representation）是系统内部对“最终报告”的统一结构化表示。
>
> 本文档回答：
>
> > 系统如何在不直接依赖 Word 模板的情况下，表示一个完整报告？
>
> 本文档不定义工程事实本身。事实由：
>
> - `03_CANONICAL_DATA_MODEL.md`
>
> 定义。

---

## 1. 设计目标

Report IR 的目标：

1. 与 Word 模板解耦。
2. 能被机器验证。
3. 能进行结构化 diff。
4. 能被程序渲染。
5. 能保证报告数字来源可追踪。
6. 能支持 AI 参与内容组织。
7. 能支持多个模板。
8. 能支持回归测试。

---

## 2. 核心原则

Report IR 不等于最终 DOCX。

正确关系：

```text
Canonical Data Model
        ↓
     Report IR
        ↓
 Template Mapping
        ↓
    Renderer
        ↓
     DOCX/PDF
````

---

## 3. 文档结构

Report IR 采用：

```text
Document
    ↓
Section
    ↓
Block
    ↓
Inline
```

---

## 4. Document

Document 表示整个报告。

示意：

```json
{
  "type": "Document",
  "version": "0.1",
  "metadata": {},
  "sections": []
}
```

---

### 4.1 Document Metadata

可以包括：

```text
report_type
project_id
template_id
template_version
created_at
generator_version
```

这些元数据用于：

* 追踪
* 回归
* 调试
* 版本管理

---

## 5. Section

Section 表示报告章节。

章节构成：

```text
Document
 ├── Section
 │    ├── Section
 │    ├── Section
 │    └── Block
```

支持章节树。

---

### 5.1 Section 的核心属性

示意：

```json
{
  "type": "Section",
  "id": "chapter-02",
  "semantic": "inspection_result",
  "title": "...",
  "blocks": []
}
```

---

## 6. Semantic Label

Section 和内容应尽量拥有语义标签。

例如：

```text
project_overview
project_description
test_method
inspection_result
calculation
evaluation
conclusion
appendix
```

语义标签不是 Word 样式名称。

语义标签描述：

> “这里是什么内容”。

Word 样式描述：

> “这里应该怎么显示”。

---

## 7. Block

Block 是报告中的块级内容。

第一阶段建议使用受控枚举：

```text
Heading
Paragraph
List
Table
Figure
Formula
PageBreak
Toc
Signature
```

---

## 8. Heading

Heading 表示标题。

示意：

```json
{
  "type": "Heading",
  "level": 2,
  "text": "检测结果"
}
```

实际实现中应尽量通过结构化字段表达标题，而不是依赖纯字符串判断。

---

## 9. Paragraph

Paragraph 表示普通段落。

### 核心限制

> Paragraph 不允许直接使用一个完全自由的正文字符串作为核心数据结构。

应使用：

```text
segments[]
```

每个 segment 是：

```text
Lit
或
Ref
```

---

## 10. Lit

Lit 表示静态文字。

例如：

```json
{
  "type": "Lit",
  "text": "经检测，"
}
```

---

## 11. Ref

Ref 表示对 Canonical Data Model 中事实或对象的引用。

例如：

```json
{
  "type": "Ref",
  "ref": "fact:building.area"
}
```

---

## 12. Paragraph = Lit + Ref

示意：

```text
Paragraph
 ├── Lit
 ├── Ref
 ├── Lit
 └── Ref
```

例如：

```json
{
  "type": "Paragraph",
  "segments": [
    {
      "type": "Lit",
      "text": "经检测，建筑面积为"
    },
    {
      "type": "Ref",
      "ref": "fact:building.area",
      "format": {
        "decimals": 2,
        "unit": "m²"
      }
    },
    {
      "type": "Lit",
      "text": "。"
    }
  ]
}
```

最终渲染后：

```text
经检测，建筑面积为1250.30m²。
```

---

## 13. 为什么 Paragraph 不能自由生成

如果模型直接输出：

```text
经检测，建筑面积约为1250平方米。
```

系统无法保证：

* 1250 从哪里来
* 单位是否正确
* 小数位是否正确
* 是否与原始资料一致

而使用 Ref：

```text
Ref("fact:building.area")
```

渲染器可以在生成最终文本前验证：

```text
Ref 是否存在
↓
Fact 是否有效
↓
Fact 是否冲突
↓
Fact 是否 missing
↓
如何格式化
```

---

## 14. Ref 的价值

### 14.1 渲染前校验

所有 Ref 必须可以解析。

---

### 14.2 数值格式统一

由程序控制：

* 单位
* 小数位
* 有效数字
* 千分位
* 百分号
* 日期格式

---

### 14.3 文本与数据交叉验证

报告文字中的值可以与 Canonical Data Model 进行对照。

---

### 14.4 稳定 Diff

当段落结构不变，只是 Fact 值变化时，可以只比较 Ref 对应值。

---

## 15. Ref 不仅用于数字

Ref 可以指向：

```text
数值
字符串
日期
枚举
结论状态
工程名称
构件编号
规范条文
图片
```

例如：

```text
fact:project.name
fact:building.area
fact:component.K001.concrete_strength
evaluation:component.K001.status
criterion:GBxxxx.clause-5.2
```

---

## 16. List

List 表示列表。

可支持：

```text
ordered
unordered
```

列表项仍然应使用结构化 Inline 表达。

---

## 17. Table

Table 表示报告中的表格。

表格不应该由 LLM 直接自由生成最终数字。

建议结构：

```text
TableSpec
{
    headers,
    rows
}
```

---

## 18. Table Cell

Cell 允许：

```text
Ref
Lit
Computed
```

即：

```text
Cell ∈ { Ref | Lit | Computed }
```

---

## 19. Table 示例

```json
{
  "type": "Table",
  "id": "table-2-1",
  "semantic": "concrete_strength_result",
  "headers": [
    {
      "type": "Lit",
      "text": "构件编号"
    },
    {
      "type": "Lit",
      "text": "实测强度"
    }
  ],
  "rows": [
    [
      {
        "type": "Ref",
        "ref": "component.K001.id"
      },
      {
        "type": "Ref",
        "ref": "measurement.K001.concrete_strength"
      }
    ]
  ]
}
```

---

## 20. Computed

Computed 用于需要程序计算的单元格。

例如：

```text
平均值
合计
最大值
最小值
统计结果
```

示意：

```json
{
  "type": "Computed",
  "expression": "avg(measurements)"
}
```

具体表达式语言后续定义。

---

## 21. 表格生成原则

推荐：

```text
Facts / MeasurementSet
        ↓
确定性程序
        ↓
TableSpec
        ↓
Renderer
        ↓
DOCX
```

而不是：

```text
LLM
 ↓
完整表格文本
 ↓
DOCX
```

---

## 22. Figure

Figure 表示图片 / 图件。

可能引用：

```text
Evidence
Image
Chart
Drawing
```

示意：

```json
{
  "type": "Figure",
  "id": "figure-2-3",
  "source_ref": "photo-
  034.jpg",
  "caption": {
    "type": "Lit",
    "text": "现场检查照片"
  }
}
```

---

## 23. Figure 与 Evidence

图片本身应来自 Canonical Data Model 中的 Evidence。

关系：

```text
Evidence
   ↓
Ref
   ↓
Figure
```

这样图片也具有来源追踪能力。

---

## 24. Formula

Formula 用于表示公式或计算表达式。

原则：

* 计算结果由程序确定
* IR 表示公式结构
* Renderer 决定最终排版方式

---

## 25. PageBreak

PageBreak 表示明确分页。

用于：

* 章节分页
* 附录分页
* 签字页
* 封面 / 正文分离

---

## 26. Toc

Toc 表示目录。

目录内容不应由 LLM 手工写死。

建议：

```text
Section Tree
    ↓
Renderer
    ↓
Toc
```

---

## 27. Signature

Signature 表示签字 / 签章区域。

可包括：

```text
签字人
日期
岗位
签章位置
```

具体盖章实现属于 Renderer / Template 层。

---

## 28. Report IR 与模板解耦

核心设计：

```text
Semantic Layer
      ↓
Template Mapping Layer
      ↓
Rendering Layer
```

---

### 28.1 Semantic Layer

描述：

```text
这是什么内容
```

例如：

```text
conclusion
test_method
inspection_result
```

---

### 28.2 Template Mapping Layer

描述：

```text
这个语义内容在具体 Word 模板中怎么显示
```

例如：

```text
semantic: conclusion
        ↓
Word Style: ConclusionText
```

或者：

```text
semantic: heading_2
        ↓
Word Style: Heading 2
```

---

### 28.3 Rendering Layer

负责：

```text
IR
 ↓
Template Mapping
 ↓
DOCX
 ↓
PDF
```

---

## 29. 换模板不应修改 IR

目标：

```text
IR
 │
 ├── Template A → DOCX
 │
 └── Template B → DOCX
```

换模板只修改：

```text
Mapping
```

而不修改：

```text
Facts
Domain Objects
Report IR
```

---

## 30. 换内容不应修改模板

不同工程项目的数据变化：

```text
Project A
Project B
Project C
```

只替换：

```text
Facts
Domain Objects
Report IR Data
```

而不是修改 Word 模板。

---

## 31. Report IR Schema

Report IR 必须有 Schema。

建议采用：

```text
JSON Schema
```

Schema 用于验证：

* Document 合法性
* Section 合法性
* Block 类型
* Inline 类型
* 必填字段
* Ref 格式
* TableSpec 结构

---

## 32. 版本号

Report IR 必须具有版本。

例如：

```text
IR version = 0.1
```

Schema 变化后应该明确升级：

```text
0.1 → 0.2
```

而不是无记录地修改旧结构。

---

## 33. IR Validator

在进入 Renderer 前：

```text
Report IR
   ↓
Schema Validator
   ↓
Pass / Fail
```

失败时不能继续正常渲染。

---

### 33.1 Validator 至少检查

```text
结构是否合法
Section 是否合法
Block 是否合法
Inline 是否合法
Ref 是否存在
Ref 是否可解析
TableSpec 是否完整
必要字段是否存在
版本是否兼容
```

---

## 34. Ref Validation

渲染前必须验证：

```text
Ref
 ↓
解析 Fact
 ↓
Fact 存在？
 ↓
Fact missing？
 ↓
Fact conflict？
 ↓
允许继续？
```

例如：

```text
Ref = fact:building.area
```

如果该 Fact 是：

```text
status = missing
```

系统不能默认生成：

```text
0
```

而应该进入：

```text
missing / error
```

由上层流程决定是否继续。

---

## 35. Report IR 与 Accuracy Validation

Report IR 本身应该支持准确性验证。

例如：

```text
Paragraph
  ↓
Ref
  ↓
Fact
  ↓
SourceRef
```

这样可以验证：

```text
报告数字
=
Canonical Data Model 中的数字
```

---

## 36. Report IR 与 Style Validation

Report IR 提供结构语义。

Template Renderer 负责版式。

因此 Style Validation 可以分别检查：

```text
IR Structure
+
Template Mapping
+
Rendered DOCX/PDF
```

---

## 37. Report IR 与 Diff

Report IR 支持结构化 Diff。

可以比较：

```text
Document
Section
Block
Paragraph
Table
Figure
Ref
```

从而避免完全依赖 PDF 像素差异。

---

## 38. IR 的一个完整示例

```json
{
  "type": "Document",
  "version": "0.1",
  "metadata": {
    "report_type": "inspection_report",
    "project_id": "project-001"
  },
  "sections": [
    {
      "type": "Section",
      "id": "chapter-01",
      "semantic": "project_overview",
      "title": "工程概况",
      "blocks": [
        {
          "type": "Paragraph",
          "segments": [
            {
              "type": "Lit",
              "text": "本工程名称为"
            },
            {
              "type": "Ref",
              "ref": "fact:project.name"
            },
            {
              "type": "Lit",
              "text": "。"
            }
          ]
        }
      ]
    },
    {
      "type": "Section",
      "id": "chapter-02",
      "semantic": "inspection_result",
      "title": "检测结果",
      "blocks": [
        {
          "type": "Table",
          "id": "table-2-1",
          "semantic": "measurement_result",
          "headers": [
            {
              "type": "Lit",
              "text": "构件编号"
            },
            {
              "type": "Lit",
              "text": "实测值"
            }
          ],
          "rows": [
            [
              {
                "type": "Ref",
                "ref": "component.K001.id"
              },
              {
                "type": "Ref",
                "ref": "measurement.K001.value"
              }
            ]
          ]
        }
      ]
    }
  ]
}
```

---

## 39. AI 在 Report IR 中的允许范围

AI 可以：

* 选择章节内容
* 选择需要引用的 Fact
* 组织受控语言
* 生成结论要点
* 选择证据
* 判断内容组织方式

AI 不应该直接：

* 自行制造数字
* 自行修改 Fact
* 自行修改程序计算结果
* 自行生成最终版式
* 绕过 Ref
* 绕过 Schema

---

## 40. Report IR 的最终定位

Report IR 是：

```text
事实世界
    ↓
报告世界
```

之间的中间表示。

完整链路：

```text
RawSource
    ↓
Facts
    ↓
Domain Objects
    ↓
Report IR
    ↓
Template Mapping
    ↓
DOCX
    ↓
PDF
```

---

## 41. 设计原则总结

Report IR 必须满足：

```text
结构化
+
可验证
+
可追溯
+
可 Diff
+
可渲染
+
与模板解耦
+
受控文本
+
数字引用化
```

最终目标不是让 AI “直接写 Word”。

而是：

> AI 参与构造一个可验证的 Report IR，再由程序负责将 Report IR 稳定地渲染成最终报告。

````

---

## 这三份现在的关系

你现在直接把它们放进去即可：

```text
docs/
├── 02_ARCHITECTURE.md
├── 03_CANONICAL_DATA_MODEL.md
└── 04_REPORT_IR.md
````

三份文件的边界应该始终保持：

```text
02_ARCHITECTURE.md
    ↓
系统怎么运行

03_CANONICAL_DATA_MODEL.md
    ↓
系统里的“事实”是什么、从哪里来

04_REPORT_IR.md
    ↓
这些事实如何被组织成“报告”
```

而 README 里比较详细的 **Accuracy Validator、Style Validator、Regression Test、MVP 实施顺序**，我没有硬塞进这三个文件；它们下一步单独沉淀成 `05_EVALUATION.md`、`06_IMPLEMENTATION_PLAN.md` 会更干净。README 对这些部分本身已经有比较完整的设计。([GitHub][1])

[1]: https://github.com/1495616384/S0P/blob/master/README.md
