# Canonical Data Model

> 状态：Draft
>
> 版本：v0.1
>
> 本文档定义系统内部对工程数据、事实、领域对象及其来源关系的统一表示。
>
> 本文档重点解决：
>
> - 不同输入来源如何统一表示
> - 工程事实如何保存
> - 事实如何追溯来源
> - 不同来源发生冲突时如何处理
> - 工程事实如何组织成领域对象
>
> 本文档不定义报告正文、章节、段落、表格和模板渲染方式。报告表示由：
>
> - `04_REPORT_IR.md`
>
> 定义。

---

## 1. 设计目标

Canonical Data Model 的目标是建立系统内部统一的数据语言，使：

```text
文字
Excel
CSV
图片
PDF
标准规范
人工输入
历史报告
```

经过解析、抽取和计算后，可以进入同一套数据模型。

系统后续的：

```text
AI 推理
程序计算
规则判定
报告生成
准确性验证
回归测试
```

都应基于该统一模型进行。

---

## 2. 四层结构

系统采用四层结构，禁止直接建立一个同时包含原始数据、事实、工程对象和报告内容的“大统一模型”。

```text
L1 RawSource
    ↓
L2 Fact Layer
    ↓
L3 Domain Objects
    ↓
L4 Report IR
```

| 层 | 名称 | 内容 | 可变性 |
|---|---|---|---|
| L1 | RawSource | 文件、页、段落、单元格、图片的原始内容与坐标 | 只读，永不修改 |
| L2 | Fact Layer | 最小断言单元，带来源与置信度 | 只增不改，冲突独立表达 |
| L3 | Domain Objects | 工程语义实体（项目/构件/检测项/测点等） | 由 Facts 组织而来 |
| L4 | Report IR | 面向报告结构的表示 | 每次生成重建 |

---

## 3. L1：RawSource

`RawSource` 表示系统接收到的原始输入资料。

其核心原则：

- 保留原始内容
- 保留原始位置
- 不在 RawSource 层直接修改业务含义
- 后续产生的 Fact 必须能够追溯到 RawSource

RawSource 可以来自：

```text
工程文字资料
Excel / CSV
图片
PDF
Word 文档
标准规范
人工输入
历史真实报告
```

---

## 4. RawSource 的定位能力

RawSource 不只是保存“哪个文件”，还应尽可能保存其在原始资料中的位置。

支持的定位类型包括：

```text
文件
页码
段落
表格
Excel Sheet
Excel 单元格
图片 ID
图片区域
其他可定位位置
```

例如：

```text
inspection.xlsx
    ↓
Sheet1!F23
```

或者：

```text
project-info.pdf
    ↓
page 12
```

或者：

```text
photo-034.jpg
    ↓
full-image
```

---

## 5. SourceRef

`SourceRef` 用于建立事实与原始资料之间的可追溯关系。

一个 Fact 可以拥有一个或多个 SourceRef。

示意：

```json
{
  "source_id": "inspection-001.xlsx",
  "location": "Sheet1!F23"
}
```

另一个示例：

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

## 6. L2：Fact Layer

Fact 是系统中的最小断言单元。

核心定义：

> Fact 只描述“是什么”，不描述“怎么写”。

例如：

```text
建筑面积 = 1250.30 m²
```

属于 Fact。

而：

```text
本工程建筑面积较大，为1250.30m²。
```

属于报告语言，不属于 Fact。

因此：

```text
Fact
    ↓
表示事实
```

而：

```text
Report IR
    ↓
表示事实如何被表达
```

这两个层次必须保持分离。

---

## 7. Fact Layer 的基本原则

### 7.1 Fact 是最小断言单元

Fact 尽量表达一个可以独立验证的断言。

例如：

```text
fact:building.area = 1250.30 m²
```

而不是：

```text
fact:building_summary = "该建筑面积约1250平方米且为5层框架结构"
```

---

### 7.2 Fact 必须可追溯

每个 Fact 都应该能够追溯到：

```text
Fact
  ↓
SourceRef
  ↓
RawSource
```

---

### 7.3 Fact 与表达方式解耦

同一个 Fact 可以被不同报告模板、不同章节、不同语言表达方式引用。

因此：

> Fact 中不保存报告措辞。

---

## 8. Fact 的强制字段

Fact 至少需要以下字段：

| 字段 | 说明 | 缺失后果 |
|---|---|---|
| `id` | 稳定标识，IR 通过它引用 | 无法定位 |
| `value` / `unit` | 值 + 单位，数值一律存 SI 基准 + 原始单位 | 单位错误不可查 |
| `source_refs[]` | 文件 + 定位（页/表/单元格/图片 ID） | 不可追溯 |
| `method` | `measured` / `computed` / `quoted` / `inferred` | 无法判断可信度 |
| `provenance` | `program` / `ai` / `human` | 无法定位错误来源 |
| `confidence` | 0–1，仅 AI 来源需要 | 无法设人工闸门 |
| `status` | `filled` / `missing` / `conflict` / `rejected` | 会静默补全 |

---

## 9. Fact.id

`id` 是 Fact 的稳定标识。

它应满足：

- 唯一
- 稳定
- 可被其他对象引用
- 能够参与回归测试
- 能够帮助定位错误

示意：

```text
fact:building.area
fact:building.floor_count
fact:component.K001.concrete_strength
fact:crack.C001.width
```

Report IR 后续通过 Fact ID 引用数据，而不是复制一份数据。

---

## 10. Fact.value / unit

`value` 表示事实值。

`unit` 表示单位。

数值数据应统一处理单位。

设计原则：

> 数值一律存 SI 基准 + 原始单位。

这样能够避免不同输入资料使用不同单位导致系统内部数据不一致。

例如：

```text
原始输入：
32.4 MPa

内部：
value = 32400000
unit = Pa

原始单位：
MPa
```

具体单位转换规则后续由确定性 Skill 定义。

---

## 11. Fact.source_refs[]

`source_refs[]` 是事实的来源定位集合。

它至少应该能够指出：

```text
来自哪个文件
位于哪里
```

例如：

```json
[
  {
    "source_id": "inspection.xlsx",
    "location": "Sheet1!F23"
  }
]
```

一个 Fact 可以拥有多个来源。

---

## 12. Fact.method

`method` 用于描述这个事实是如何得到的。

允许的基础类型：

```text
measured
computed
quoted
inferred
```

### measured

来自检测或测量。

例如：

```text
混凝土实测强度 = 32.4 MPa
```

---

### computed

由程序基于已有数据计算得到。

例如：

```text
平均值
最大值
最小值
总和
统计值
```

---

### quoted

直接从资料中引用。

例如：

```text
报告委托方名称
设计文件中的建筑面积
```

---

### inferred

基于其他信息推导得到。

这类 Fact 必须特别注意：

- 来源
- 推导依据
- confidence
- 人工确认

不能把 inferred 当成原始观测事实。

---

## 13. Fact.provenance

`provenance` 表示这个 Fact 的产生来源。

基础类型：

```text
program
ai
human
```

### program

由确定性程序产生。

例如：

```text
程序根据检测数据计算平均值
```

---

### ai

由 AI 从非结构化资料中抽取、理解或推导得到。

例如：

```text
AI 从图片识别出疑似裂缝
```

---

### human

由人工录入或人工确认得到。

例如：

```text
专业人员确认构件编号
```

---

## 14. Fact.confidence

`confidence` 表示事实的置信度。

建议取值范围：

```text
0 ～ 1
```

例如：

```json
{
  "confidence": 0.92
}
```

主要用于 AI 来源的 Fact。

它的作用包括：

- 判断是否需要人工确认
- 过滤低置信度结果
- 支持后续验证
- 定位 AI 抽取风险

---

## 15. Fact.status

Fact 状态：

```text
filled
missing
conflict
rejected
```

---

### 15.1 filled

表示 Fact 已获得并通过当前阶段的有效性检查。

---

### 15.2 missing

表示系统需要该事实，但目前无法获得。

例如：

```json
{
  "id": "fact:building.year",
  "value": null,
  "unit": null,
  "status": "missing"
}
```

`missing` 是合法状态。

它不允许被 AI 自动填成一个猜测值。

---

### 15.3 conflict

表示存在多个互相不一致的候选事实。

例如：

```text
文件 A：
建筑面积 = 1250.30 m²

文件 B：
建筑面积 = 1280.00 m²
```

此时：

```text
status = conflict
```

而不是静默选择其中一个。

---

### 15.4 rejected

表示该事实候选已经被否决，不允许继续作为有效事实使用。

例如：

```text
AI 从图片错误识别得到：
component_id = K999

人工确认：
错误

status = rejected
```

---

## 16. Fact 示例

### 16.1 测量事实

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

### 16.2 AI 抽取事实

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

### 16.3 程序计算事实

```json
{
  "id": "fact:measurement.K001.average",
  "value": 32.4,
  "unit": "MPa",
  "source_refs": [
    {
      "source_id": "measurement-calculation"
    }
  ],
  "method": "computed",
  "provenance": "program",
  "confidence": 1.0,
  "status": "filled"
}
```

---

## 17. L3：Domain Objects

Domain Objects 用于把大量 Facts 组织成工程领域中的实体。

其作用不是复制 Fact，而是建立工程语义关系。

针对检测鉴定报告，第一阶段采用以下通用骨架：

```text
Project
  ↓
Client / Structure
  ↓
Component
  ↓
InspectionItem
  ↓
MeasurementSet
  ↓
Criterion
  ↓
Evaluation
  ↓
Conclusion
  ↓
Evidence
```

更加完整地表示：

```text
Project
→ Client
→ Structure
→ Component
→ InspectionItem
→ MeasurementSet
→ Criterion
→ Evaluation
→ Conclusion
→ Evidence
```

---

## 18. Project

`Project` 表示工程项目。

例如：

```text
项目名称
项目编号
项目地点
工程类型
```

Project 是工程对象的顶层组织单元。

---

## 19. Client

`Client` 表示委托方或相关主体。

它可以与 Project 建立关系。

例如：

```text
Project
  ↓
Client
```

---

## 20. Structure

`Structure` 表示建筑或结构层级。

可描述：

```text
建筑
楼
层
轴
区域
结构单元
```

其核心作用是为构件和检测结果建立空间层级。

---

## 21. Component

`Component` 表示工程中的结构构件。

例如：

```text
梁
柱
板
墙
基础
```

通常需要具有：

```text
构件编号
构件类型
空间位置
```

例如：

```text
Component:
    id = K001
    type = Column
```

---

## 22. InspectionItem

`InspectionItem` 表示检测项目。

例如：

```text
混凝土强度
钢筋保护层
裂缝
沉降
倾斜
```

它表示：

> 正在检测什么。

---

## 23. MeasurementSet

`MeasurementSet` 表示某一个检测项目的一组测量数据。

例如：

```text
某楼层全部柱子的混凝土强度检测结果
```

MeasurementSet 可以包含多个测点 / 多个 Measurement。

例如：

```text
MeasurementSet
├── K001 = 32.4 MPa
├── K002 = 31.8 MPa
├── K003 = 33.1 MPa
└── ...
```

---

## 24. Measurement

Measurement 表示一个具体测量值。

典型信息包括：

```text
构件
测点
测值
单位
检测方法
检测时间
仪器
```

具体字段根据实际报告类型继续细化。

---

## 25. Criterion

`Criterion` 表示判定依据。

例如：

```text
规范条文
规范限值
适用条件
判定规则
```

Criterion 的作用是说明：

> “按照什么规则进行判断”。

---

## 26. Evaluation

`Evaluation` 表示基于 Facts 与 Criterion 得到的判定结果。

例如：

```text
合格
不合格
某等级
满足要求
不满足要求
```

设计原则：

> `Criterion` 与 `Evaluation` 是领域模型中允许产生新判定信息的核心位置。

其判定应由明确规则和确定性程序执行，而不是由 LLM 自由生成。

---

## 27. Conclusion

`Conclusion` 表示工程层面的结论。

需要区分：

```text
结论事实
```

与：

```text
报告文字
```

例如：

```text
Evaluation:
    status = qualified
```

是结构化结论事实。

而：

```text
经综合评定，该构件满足相关要求。
```

属于报告语言，应由 Report IR 表示。

---

## 28. Evidence

`Evidence` 表示用于支持工程事实或判定的证据。

例如：

```text
现场图片
原始检测记录
测量记录
规范条文
其他原始资料
```

Evidence 应保留：

```text
来源
定位
关联对象
```

从而能够建立：

```text
Conclusion
    ↓
Evaluation
    ↓
Evidence
    ↓
RawSource
```

的追溯链。

---

## 29. Domain Objects 与 Facts 的关系

Domain Objects 不应该成为另一套独立的数据真相。

推荐：

```text
RawSource
    ↓
Facts
    ↓
Domain Objects
```

即：

```text
Fact = 最小事实
Domain Object = 事实的工程语义组织
```

例如：

```text
Fact:
    fact:component.K001.concrete_strength
    value = 32.4 MPa
```

组织后：

```text
Component:
    id = K001

MeasurementSet:
    concrete_strength

Measurement:
    value = 32.4 MPa
```

---

## 30. “不产生事实”原则

系统整体必须坚持：

> 系统不凭空产生事实，只允许搬运、计算、组织和引用事实。

可以产生的新数据主要来自：

```text
原始资料抽取
确定性计算
明确规则判定
人工确认
```

任何无法从已有数据和规则推出的数值，都不能由 AI 自行补全。

不能得到时：

```text
status = missing
```

存在冲突时：

```text
status = conflict
```

被人工否决时：

```text
status = rejected
```

---

## 31. Fact 的来源链

完整的事实追溯链：

```text
Fact
  ↓
source_refs[]
  ↓
RawSource
```

更完整时：

```text
Report
  ↓
Report IR
  ↓
Ref
  ↓
Fact
  ↓
SourceRef
  ↓
RawSource
```

因此，系统最终应能够回答：

> 报告里的这个值来自哪里？

---

## 32. 多来源事实

一个 Fact 可以由多个来源支持。

例如：

```text
Fact:
    building.area = 1250.30 m²

source_refs:
    ├── design.pdf page 8
    └── inspection.xlsx Sheet1!F23
```

多个来源一致时，可以作为同一事实的多个证据来源。

多个来源不一致时，必须进入 Conflict。

---

## 33. Conflict

同一事实存在多来源不一致时，生成 `Conflict` 记录。

原则：

> 不允许静默覆盖。

不允许使用：

```text
最后一次写入的数据覆盖之前的数据
```

作为默认冲突解决策略。

---

## 34. Conflict 数据结构

示意：

```text
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

---

## 35. Conflict 示例

```json
{
  "fact_id": "fact:building.area",
  "candidates": [
    {
      "value": 1250.30,
      "source_ref": "design.pdf#page=8",
      "provenance": "quoted"
    },
    {
      "value": 1280.00,
      "source_ref": "inspection.xlsx#Sheet1!F23",
      "provenance": "quoted"
    }
  ],
  "resolved_by": null,
  "resolution": null
}
```

在冲突没有解决之前，不应把某一个候选值静默认定为最终有效事实。

---

## 36. 第一阶段 Conflict 处理

第一阶段：

```text
resolved_by = human
```

即：

> 冲突必须显式暴露，由人工确认。

系统不自动隐藏冲突。

也不采用：

```text
最后写入者胜出
```

这种隐式策略。

---

## 37. Missing 与 Conflict 的区别

### Missing

系统没有足够信息得到事实。

```text
status = missing
```

---

### Conflict

系统得到了多个互相不一致的候选事实。

```text
status = conflict
```

---

### 正确处理方式

```text
缺失
 ↓
missing
```

```text
冲突
 ↓
conflict
```

而不是：

```text
missing
 ↓
AI 猜一个
```

或者：

```text
conflict
 ↓
AI 随便选一个
```

---

## 38. AI Fact 的使用原则

AI 可以参与：

```text
非结构化文字
    ↓
结构化 Fact

图片
    ↓
Evidence / Fact

PDF
    ↓
结构化 Fact
```

但 AI 产生的 Fact 必须：

```text
有 Schema
+
有 SourceRef
+
有 provenance
+
有 confidence
+
可人工确认
```

---

## 39. Program Fact 的使用原则

程序可以产生：

```text
计算结果
统计结果
规则判断结果
```

例如：

```text
MeasurementSet
    ↓
程序计算
    ↓
average
```

或者：

```text
Fact
    +
Criterion
    ↓
程序判定
    ↓
Evaluation
```

程序产生的结果同样应保留来源和推导关系。

---

## 40. 与 Report IR 的边界

Canonical Data Model 和 Report IR 必须保持明确边界：

```text
Canonical Data Model
    ↓
描述：
工程中“是什么”

Report IR
    ↓
描述：
这些事实“如何组成报告”
```

因此：

```text
Fact
=
事实

Report IR
=
事实的报告表达
```

Fact 中不能保存报告措辞。

---

## 41. 最终数据流

完整的数据关系：

```text
RawSource
    ↓
Fact Layer
    ↓
Domain Objects
    ↓
Report IR
```

其中：

```text
RawSource
=
原始资料
```

```text
Fact Layer
=
最小事实
```

```text
Domain Objects
=
工程语义组织
```

```text
Report IR
=
报告结构表示
```

---

## 42. 设计原则总结

Canonical Data Model 必须满足：

```text
统一
+
可追溯
+
可验证
+
可冲突表达
+
可缺失表达
+
可人工确认
+
AI / 程序均可使用
+
可被 Report IR 引用
```

核心原则归纳为：

> **L1 保存原始资料，L2 保存事实，L3 组织工程语义，L4 负责报告表示。**

> **L2 只存“是什么”，不存“怎么写”。**

> **事实必须能够追溯来源。**

> **缺失必须显式表示，冲突必须显式表示，不允许静默补全。**
