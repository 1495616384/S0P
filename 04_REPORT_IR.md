# Report IR

> 状态：Draft
>
> 版本：v0.1
>
> Report IR（Report Intermediate Representation）是系统内部对“报告结构”的统一表示。
>
> 本文档定义报告在进入 Word / PDF 渲染之前的中间表示，不直接绑定具体 Word 模板。
>
> 本文档重点解决：
>
> - 报告结构如何统一表示
> - 报告正文如何引用事实
> - 表格如何结构化生成
> - 报告语义如何与 Word 模板解耦
> - 报告如何在渲染前进行结构校验
> - 报告如何进行结构化 diff 与验证
>
> 工程事实本身由：
>
> - `03_CANONICAL_DATA_MODEL.md`
>
> 定义。

---

## 1. 设计目标

Report IR 的设计目标：

1. 与 Word 模板解耦。
2. 可以被程序验证。
3. 可以进行结构化 Diff。
4. 可以被程序稳定渲染。
5. 报告中的数字可以追溯到 Canonical Data Model。
6. AI 可以参与报告内容组织，但不能绕过事实引用机制。
7. 同一份 Report IR 可以映射到不同报告模板。
8. 模板变化不应导致业务事实模型变化。

核心关系：

```text
Canonical Data Model
        ↓
     Report IR
        ↓
 Template Mapping
        ↓
     Renderer
        ↓
      DOCX
        ↓
       PDF
```

---

## 2. Report IR 的定位

Report IR 是：

> **工程事实与最终报告之间的中间表示。**

完整数据流：

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
DOCX / PDF
```

其中：

```text
Canonical Data Model
=
描述工程数据和事实
```

而：

```text
Report IR
=
描述这些事实如何组成一份报告
```

---

## 3. 核心原则

### 3.1 Report IR 不等于 DOCX

DOCX 是最终渲染结果。

Report IR 是结构化的中间表示。

因此：

```text
Report IR
    ≠
Word Document
```

---

### 3.2 Report IR 不保存完整工程事实

Report IR 不应该复制一整份 Canonical Data Model。

而应该通过引用：

```text
Ref
```

使用已有 Fact。

例如：

```text
fact:project.name
fact:building.area
fact:component.K001.concrete_strength
```

---

### 3.3 报告正文不能直接使用完全自由文本

这是 Report IR 的核心约束。

普通 Paragraph 不直接保存一个不可追溯的完整自由文本字符串。

而应该使用：

```text
TextTemplate
    =
静态文本片段
+
变量 / Fact 引用
```

即：

```text
Lit
+
Ref
```

这样 AI 可以负责组织表达，但最终事实值仍由程序负责提供。

---

## 4. Report IR 的结构层次

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

即：

```text
Document
├── Section
│   ├── Block
│   │   ├── Inline
│   │   └── Inline
│   └── Block
└── Section
```

---

## 5. Document

`Document` 表示完整报告。

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

### 5.1 Document 的基本属性

第一阶段建议包含：

```text
type
version
metadata
sections
```

其中：

- `type`：对象类型
- `version`：IR Schema 版本
- `metadata`：报告元数据
- `sections`：章节树

---

## 6. Section

`Section` 表示报告章节。

报告章节采用树状结构。

例如：

```text
Document
│
├── Section：工程概况
│
├── Section：检测依据
│
├── Section：现场检查
│   ├── Section：建筑检查
│   └── Section：结构构件检查
│
├── Section：检测结果
│
└── Section：鉴定结论
```

---

### 6.1 Section 的基本属性

建议包括：

```text
type
id
semantic
title
blocks
children
```

示意：

```json
{
  "type": "Section",
  "id": "chapter-02",
  "semantic": "inspection_result",
  "title": "检测结果",
  "blocks": []
}
```

---

## 7. Semantic Layer

Report IR 应具有语义层。

语义标签用于表示：

> **这个内容“是什么”。**

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

语义标签不能直接等同于 Word 样式名称。

例如：

```text
semantic:
conclusion
```

表示：

> 这是报告中的“结论内容”。

而：

```text
Word Style:
ConclusionText
```

表示：

> 在具体模板中，这个内容应该怎样显示。

---

## 8. Block

Block 是报告中的块级内容。

第一阶段采用受控枚举：

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

## 9. Heading

`Heading` 表示章节标题或小节标题。

示意：

```json
{
  "type": "Heading",
  "level": 2,
  "text": "检测结果"
}
```

实际渲染时由 Template Mapping 决定最终的：

- 字体
- 字号
- 段落间距
- 编号
- 样式

---

## 10. Paragraph

`Paragraph` 表示普通正文段落。

### 核心约束

> Paragraph 不允许使用完全自由文本作为唯一表示方式。

应该表示成：

```text
Paragraph
    ↓
segments[]
    ↓
Lit / Ref
```

---

## 11. Lit

`Lit` 表示静态文本片段。

例如：

```json
{
  "type": "Lit",
  "text": "经检测，建筑面积为"
}
```

`Lit` 中不应该放需要从 Facts 获取的动态事实值。

---

## 12. Ref

`Ref` 表示对 Canonical Data Model 中事实的引用。

例如：

```json
{
  "type": "Ref",
  "ref": "fact:building.area"
}
```

Ref 是 Report IR 与 Canonical Data Model 之间的关键连接。

---

## 13. Paragraph = Lit + Ref

一个 Paragraph 可以由多个 `Lit` 和 `Ref` 组合。

例如：

```text
Paragraph
├── Lit
├── Ref
├── Lit
└── Ref
```

对应：

```text
Paragraph(
    segments = [
        Lit("经检测，"),
        Ref("fact:concrete_strength"),
        Lit("，"),
        Ref("fact:component.status")
    ]
)
```

---

## 14. Paragraph 示例

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
      "ref": "fact:building.area"
    },
    {
      "type": "Lit",
      "text": "。"
    }
  ]
}
```

最终渲染结果可能是：

```text
经检测，建筑面积为1250.30m²。
```

其中：

```text
“经检测，建筑面积为”
```

来自 `Lit`，

而：

```text
1250.30m²
```

来自：

```text
Ref("fact:building.area")
```

---

## 15. 为什么使用 Lit + Ref

### 15.1 渲染前校验

程序可以在渲染前检查：

```text
Ref 是否存在
↓
Fact 是否存在
↓
Fact 是否 missing
↓
Fact 是否 conflict
↓
Fact 是否允许被引用
```

---

### 15.2 数值格式化由程序控制

例如：

```text
单位
小数位
有效数字
百分号
日期
千分位
```

不应该由 LLM 自由决定。

例如：

```text
Ref:
    fact:building.area

Format:
    decimals = 2
    unit = "m²"
```

最终由程序渲染。

---

### 15.3 文本与数据可以交叉验证

因为文本中的动态数据不是自由字符串，而是：

```text
Ref
```

因此可以建立：

```text
报告文字
    ↓
Ref
    ↓
Fact
```

的验证链。

---

### 15.4 Diff 更稳定

当：

```text
Paragraph 结构
```

没有变化，而：

```text
Fact value
```

发生变化时，可以识别为：

```text
同一报告结构，不同数据
```

而不是把整个段落当作全新文本。

---

## 16. Ref 的基本要求

每个 Ref 应能够唯一定位一个：

```text
Fact
```

或者后续定义的其他结构化数据对象。

示意：

```text
fact:project.name
fact:building.area
fact:component.K001.concrete_strength
fact:crack.C001.width
evaluation:component.K001.status
criterion:GBxxxx.clause-5.2
```

---

## 17. Ref 的格式化

Ref 可以附带格式说明。

示意：

```json
{
  "type": "Ref",
  "ref": "fact:building.area",
  "format": {
    "decimals": 2,
    "unit": "m²"
  }
}
```

注意：

> Format 描述“如何显示”，不修改 Fact 本身。

---

## 18. Ref 的语义边界

`Ref` 只负责引用。

不能通过 Ref 修改 Canonical Data Model 中的事实。

即：

```text
Report IR
    ↓
Ref
    ↓
读取 Fact
```

而不是：

```text
Report IR
    ↓
修改 Fact
```

---

## 19. List

`List` 表示列表。

第一阶段支持：

```text
ordered
unordered
```

例如：

```text
1. 工程概况
2. 检测依据
3. 检测结果
```

列表项的正文仍然应采用结构化 Inline 表示，而不是绕过 Report IR 的事实引用机制。

---

## 20. Table

`Table` 表示报告中的结构化表格。

表格不应该由 LLM 直接生成最终自由文本或最终数字。

建议采用：

```text
TableSpec
```

表示。

---

## 21. TableSpec

基本结构：

```text
TableSpec {
    headers,
    rows
}
```

其中：

```text
Cell ∈ { Ref | Lit | Computed }
```

即单元格可以由：

```text
Ref
```

```text
Lit
```

```text
Computed
```

组成。

---

## 22. Table 示例

```json
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
        "ref": "fact:component.K001.id"
      },
      {
        "type": "Ref",
        "ref": "fact:component.K001.concrete_strength"
      }
    ]
  ]
}
```

---

## 23. Computed

`Computed` 表示由程序根据表达式计算得到的结果。

例如：

```text
平均值
最大值
最小值
合计
统计结果
```

示意：

```json
{
  "type": "Computed",
  "expression": "avg(measurements)"
}
```

具体计算表达式语言后续定义。

---

## 24. 表格生成原则

推荐流程：

```text
Facts / Domain Objects
        ↓
MeasurementSet
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
生成完整 Markdown / HTML 表格
 ↓
转换 DOCX
```

原因：

> 表格中的工程数字必须可计算、可追溯、可验证。

---

## 25. Figure

`Figure` 表示报告中的图片或图件。

可能来源于：

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
  "source_ref": "photo-034.jpg",
  "caption": {
    "type": "Lit",
    "text": "现场检查照片"
  }
}
```

---

## 26. Figure 与 Evidence

推荐关系：

```text
Evidence
    ↓
Ref
    ↓
Figure
```

这样图片同样具有：

- 来源
- 关联对象
- 可追溯性

---

## 27. Formula

`Formula` 表示报告中的公式或计算表达式。

核心原则：

> 公式的计算结果由程序负责，Report IR 负责结构化表示。

因此：

```text
输入数据
    ↓
程序计算
    ↓
计算结果 Fact
    ↓
Report IR Ref
```

最终 Renderer 再决定公式的具体版式。

---

## 28. PageBreak

`PageBreak` 表示明确分页。

可用于：

```text
章节分页
附录分页
封面 / 正文分离
签字页
```

最终具体分页效果仍由 Renderer 与 Word 模板共同决定。

---

## 29. Toc

`Toc` 表示目录。

目录不应该由 LLM 手工生成完整文本。

推荐：

```text
Section Tree
    ↓
Renderer
    ↓
Toc
```

这样目录可以自动跟随章节变化。

---

## 30. Signature

`Signature` 表示签字 / 签章区域。

可能包含：

```text
签字人
岗位
日期
签章位置
```

实际签章、图片、位置等属于 Template / Renderer 层负责的内容。

---

## 31. Report IR 与模板解耦

Report IR 与 Word 模板之间采用：

```text
Semantic Layer
        ↓
Template Mapping Layer
        ↓
Rendering Layer
```

---

## 32. Semantic Layer

Semantic Layer 描述：

> **这个内容是什么。**

例如：

```text
project_overview
test_method
inspection_result
evaluation
conclusion
```

---

## 33. Template Mapping Layer

Template Mapping Layer 描述：

> **这个语义内容在指定 Word 模板中如何表现。**

例如：

```text
semantic:
conclusion

        ↓

Word Style:
ConclusionText
```

或者：

```text
semantic:
heading_2

        ↓

Word Style:
Heading 2
```

---

## 34. Rendering Layer

Rendering Layer 将 Report IR 转换成最终报告：

```text
Report IR
    ↓
Template Mapping
    ↓
Word Template
    ↓
DOCX
    ↓
PDF
```

Renderer 负责：

- 字体
- 字号
- 行距
- 段落格式
- 表格
- 图片
- 编号
- 页码
- 页眉页脚
- 模板布局

Report IR 不直接承担这些具体 Word 排版职责。

---

## 35. 换模板不修改 IR

目标结构：

```text
              Report IR
             /        \
            ↓          ↓
      Template A    Template B
            ↓          ↓
          DOCX        DOCX
```

换模板时主要修改：

```text
Template Mapping
```

而不应修改：

```text
Facts
Domain Objects
```

也不应因为换模板而重新定义工程事实。

---

## 36. 换内容不修改模板

不同工程：

```text
Project A
Project B
Project C
```

可以使用同一个模板：

```text
Facts
    ↓
Report IR
    ↓
同一个 Template Mapping
    ↓
同一个 Word Template
```

只替换数据和内容，而不是修改模板结构。

---

## 37. Schema

Report IR 必须具有明确的 Schema。

第一阶段建议使用：

```text
JSON Schema
```

Schema 用于保证：

- Document 合法
- Section 合法
- Block 合法
- Inline 合法
- 必填字段存在
- Ref 格式正确
- TableSpec 结构正确

---

## 38. Version

Report IR 必须带版本号。

例如：

```text
version = 0.1
```

Schema 发生不兼容变化时，应升级版本。

例如：

```text
0.1
↓
0.2
```

而不是无记录修改旧结构。

---

## 39. IR Validator

Report IR 在进入 Renderer 之前，必须经过 Schema 校验。

流程：

```text
Report IR
    ↓
IR Validator
    ↓
Pass / Fail
```

如果 Schema 校验失败：

```text
禁止正常渲染
```

应输出明确错误。

---

## 40. IR Validator 检查范围

第一阶段至少检查：

```text
Document 类型是否合法
version 是否兼容
Section 结构是否合法
Block 类型是否合法
Inline 类型是否合法
必填字段是否存在
Ref 格式是否合法
TableSpec 是否完整
```

---

## 41. Ref Validation

除了 JSON Schema，还必须检查 Ref 是否能够被真正解析。

流程：

```text
Ref
 ↓
查找 Fact
 ↓
Fact 是否存在？
 ↓
Fact 是否 missing？
 ↓
Fact 是否 conflict？
 ↓
当前业务是否允许继续？
```

例如：

```text
Ref:
fact:building.area
```

如果：

```text
Fact.status = missing
```

则不能静默生成：

```text
0
```

也不能由 LLM 自动补一个值。

应该进入：

```text
missing
```

或：

```text
error
```

状态，由上层流程决定是否继续。

---

## 42. Report IR 与 Accuracy Validation

Report IR 的结构应该支持后续准确性验证。

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

因此可以验证：

```text
报告数字
    =
Canonical Data Model 中的 Fact
```

并进一步追溯：

```text
Fact
    ↓
SourceRef
    ↓
原始输入
```

---

## 43. Report IR 与 Style Validation

Report IR 提供：

```text
结构
+
语义
```

Template / Renderer 提供：

```text
样式
+
版式
```

因此最终可以分别验证：

```text
IR Structure
+
Template Mapping
+
Rendered DOCX / PDF
```

---

## 44. Report IR 与 Diff

Report IR 应支持结构化 Diff。

主要比较对象：

```text
Document
Section
Block
Paragraph
Table
Figure
Ref
```

这样可以区分：

```text
结构变化
```

与：

```text
事实值变化
```

避免所有变化都只能依赖 PDF 像素差异判断。

---

## 45. AI 在 Report IR 中的允许范围

AI 可以负责：

- 根据报告目标选择需要的内容
- 选择需要引用的 Fact
- 组织 `Lit + Ref`
- 组织结论要点
- 组织受控语言
- 选择合适的 Evidence
- 形成 Report IR 草稿

但 AI 不应该直接：

- 编造数字
- 绕过 Fact
- 修改 Canonical Data Model
- 修改程序计算结果
- 绕过 Schema
- 直接控制最终 Word 排版
- 把未经验证的数据写入最终报告

---

## 46. Report IR 的最小闭环

第一阶段首先打通：

```text
Fact
  ↓
程序生成 Report IR
  ↓
Schema Validation
  ↓
Ref Validation
  ↓
Template Mapping
  ↓
DOCX
```

即使完全没有 LLM，也应该可以形成：

```text
Facts
→
IR
→
DOCX
```

这一最小闭环。

---

## 47. 完整示例

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
                "ref": "fact:component.K001.id"
              },
              {
                "type": "Ref",
                "ref": "fact:component.K001.concrete_strength"
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

## 48. 核心关系总结

整个系统的数据关系：

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
Renderer
    ↓
DOCX / PDF
```

其中：

```text
03_CANONICAL_DATA_MODEL.md
```

负责：

```text
事实是什么
```

而：

```text
04_REPORT_IR.md
```

负责：

```text
事实如何组成报告
```

---

## 49. 核心设计原则

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
事实引用
```

最终目标不是：

```text
LLM
 ↓
直接写 Word
```

而是：

```text
LLM / Program
      ↓
Canonical Data Model
      ↓
Report IR
      ↓
Schema Validation
      ↓
Template Mapping
      ↓
Renderer
      ↓
DOCX / PDF
```

其中：

> **AI 参与构造 Report IR，但最终事实值、结构校验和文档渲染由程序控制。**
