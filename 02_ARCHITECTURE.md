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
```

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

- 字体
- 字号
- 行距
- 表格
- 编号
- 页码
- 页眉页脚
- 章节结构

必须保持稳定。

因此：

> 报告语义表示与 Word 模板表示必须解耦。

---

### 2.3 生成与验证

系统不仅需要能够生成报告，还必须能够证明：

- 数值是否正确
- 表格是否正确
- 结论是否与数据一致
- 引用是否存在
- 结构是否符合要求
- 样式是否满足模板要求

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

- OA / 审批流程集成
- 多人协同编辑
- 现场采集 APP
- 检测设备直接对接
- 报告归档与检索平台
- 完整知识库
- 全量规范向量数据库
- 全自动无人值守发布
- 模型训练 / 微调
- 所有报告类型的一次性覆盖
- 图形化报告编辑器

---

## 4. 核心架构原则

### 4.1 系统不产生事实

系统只允许：

- 搬运事实
- 提取事实
- 计算事实
- 组织事实
- 引用事实

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

- 规则
- 公式
- 表格
- 确定性算法

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

- 理解任务
- 判断当前状态
- 选择 Skill
- 调度 Skill
- 处理缺失
- 协调流程
- 触发验证
- 必要时进行有限自修复

Agent 不直接控制 Word 排版，也不直接决定最终数值。

---

### 4.5 Skill 必须是可测试能力

Skill 必须具有：

- 明确输入
- 明确输出
- Schema
- 版本号
- 测试用例
- 可替换实现

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

- 文件名
- 文件 ID
- 页码
- 表格位置
- Excel Sheet
- Excel 单元格
- 段落位置
- 图片 ID
- 其他可定位信息

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

- 可人工检查
- 可版本控制
- 可作为测试 Fixture
- 可回归
- 可定位错误来源

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

- `RawSource`：原始来源
- `Facts`：事实
- `Domain Objects`：工程领域对象
- `Report IR`：报告结构表示

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

- 有目标
- 有状态
- 能选择 Skill
- 能根据结果决定下一步
- 可以进行有限循环

Agent 不直接代替 Skill 完成具体能力。

---

### 8.2 第一阶段 Agent

#### IntakeAgent

职责：

- 判断输入类型
- 建立输入任务
- 调度解析 Skill
- 标记缺失
- 建立 SourceRef

---

#### ExtractionAgent

职责：

- 从文字中抽取 Facts
- 从图片中提取 Evidence
- 对非结构化资料进行结构化

---

#### ReasoningAgent

职责：

- 基于 Facts 与规范进行工程推理
- 执行或调用判定能力
- 形成结论事实
- 形成结论要点

限制：

> 不直接输出最终报告版式文本。

---

#### ReportComposerAgent

职责：

- 根据报告 IR 骨架组织内容
- 将 Facts 引入 Report IR
- 产生结构化 Report IR

---

#### ValidatorAgent

职责：

- 调用 Accuracy Validator
- 调用 Style Validator
- 形成 IssueList
- 根据问题进行有限次数的修复循环

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

- 避免误差累积
- 避免幻觉传播
- 保证接口稳定
- 便于测试

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

- input schema
- output schema
- few-shot
- 低温策略
- 重试策略
- Schema 校验

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

- 统一模型调用方式
- 结构化输出适配
- JSON 提取
- 输出修复
- Retry
- Schema 校验
- Trace 记录

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

- 调试
- 回归
- A/B
- 问题定位

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

- 数值运算
- 单位换算
- 统计计算
- 限值判断
- 规则判断
- 表格生成
- 编号
- 目录
- 页码
- Word 渲染
- 样式检查
- 交叉引用检查

---

### 11.2 允许 AI 处理

AI 主要处理：

- 非结构化文字理解
- 图片理解
- 字段抽取
- 文档分类
- 证据总结
- 结论要点组织
- 受控语言组织

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

- AI 抽取结果确认
- 关键事实确认
- 最终结论签发

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

- Schema / 完整性
- 数据一致性
- 计算一致性
- 结论方向一致性
- Ground Truth 对比

### Style

- 字体
- 字号
- 行距
- 缩进
- 页边距
- 页眉页脚
- 页码
- 表格
- 编号
- 章节结构

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

- 缺失数据
- 数据冲突
- 单位异常
- 超限值
- 无法判断的问题

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

- 调试
- 回归
- 人工检查
- 版本控制

---

### D11. IR 与模板解耦

语义层与版式层分离。

---

### D12. 第一阶段优先 CLI + JSON

第一阶段不优先建设图形化前端。

---

## 15. 第一阶段明确不做

暂不建设：

- 微服务架构
- Kubernetes
- Redis
- MQ
- 全量向量数据库
- 全量 RAG
- Multi-Agent 自治协商
- 知识图谱
- 模型训练 / 微调
- 自研 Word 排版引擎
- 图形化编辑器
- 全自动无人值守生成
- 一次性支持所有报告类型

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
