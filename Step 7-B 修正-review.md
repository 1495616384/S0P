好的，这是一个设计澄清后的定点修正。按规则先调用经验召回和战略指导。

*正在请求专家建议，优化解决方案，该过程耗时可能较长，请耐心等待...*

三个高风险点已锁定。按执行顺序推进，先建 Todo，然后从设计文档开始。
用户提供了非常详细的修正方案。我先调用必要的前置流程。经验召回：定点改动必须按点位推进，不要大范围重写。用户已经明确列出了每一个文档和代码文件的修改点，我要逐个点位精准修改。好的，这个经验和我们现在做的完全吻合——我们要在 Fact dataclass 上新增 revision 字段，必须同步更新 to_dict/from_dict/所有构造路径/validate 函数/所有测试。不能漏掉任何一个。
正在请求专家建议.....

已获取专家建议

好，三个关键风险已明确。现在严格按执行顺序：先改设计文档 → 再改代码 → 再改测试 → 运行。

先创建 TodoWrite，然后开始第一步：设计文档。

先读设计澄清会话文件和需要精确改动的 03 文档关键段落：
先读取设计澄清会话文件，确认完整的修正方案细节。同时读取当前 03_CANONICAL_DATA_MODEL.md 的关键章节。

好，先精确定位 03 文档所有要改的段落：
好，现在开始精准修改 03_CANONICAL_DATA_MODEL.md。我需要先精确定位每个要改的段落，然后做定点编辑。让我先把相关段落读出来，然后逐个 Edit。

好，所有目标段落已锁定。开始定点编辑 03_CANONICAL_DATA_MODEL.md。

**批次 1：修订说明 + §8 强制字段表 + §8.1 scope_class grammar**
好，所有需要修改的段落我都定位到了。现在开始执行精准的定点编辑。按顺序：

### 03_CANONICAL_DATA_MODEL.md 修改点：

1. **v0.3 修订说明**（第20行附近）→ 加 v0.3.1 记录
2. **§8 强制字段表**（第292-306行）→ 加 revision 字段
3. **§8.1 scope_class**（第312-333行）→ 定义改为"断言主体的实体类型"，加 defect，加 grammar 规则，加 fact_type ≡ scope_class.attribute
4. **§9 示例**（第389-394行）→ crack→defect
5. **§15.2 building.year**（第665行）→ 补 instance_key
6. **§15.5 更正语义**（第720行起）→ 删除 .v2 示例，改 revision 示例
7. **§16.2 photo**（第792行）→ photo 不作为主体，改 defect + source_refs
8. **§16.3 measurement**（第814行）→ 改合法具名聚合
9. **§16.4 crack**（第863行起）→ crack→defect
10. **§17 Domain Objects**（第907行起）→ Component 下加 Defect

我需要一批一批执行 Edit。先从修订说明和 §8.1 开始。

好，修订说明已更新。下一批：**§8 强制字段表 + §8.1 scope_class 定义与 grammar**
好，修订说明已更新。现在继续 §8 强制字段表和 §8.1。

现在改 **§8.1 scope_class 定义 + 枚举 + grammar 冻结规则**：
好，现在改 §8.1 的核心段落——scope_class 定义、scope_class 枚举、grammar 冻结规则。

继续：**§9 示例（crack→defect）+ §15.2（补 instance_key）+ §15.5（revision 链）**
好，§8.1 改完了。现在继续 §9 示例、§15.2 building.year 补 instance_key、§15.5 更正语义改 revision、§16 示例、§17 加 Defect。

现在改 **§15.2 building.year 补 instance_key + §15.5 更正语义（revision 链替代 .v2）**：
好，继续 §15.2 的 building.year 补 instance_key。让我先定位一下：

现在改 **§15.5 更正语义（revision 链替代 .v2 后缀）**：
好，§15.2 已修正。现在改 §15.5 更正语义——这是最关键的改动，要把 .v2 ID 改成 revision 链。先精确定位当前 §15.5 的完整内容。

现在重写 **§15.5**（删除 `.v2` ID，用 `revision` 字段表达更正链）：
好，§15.5 完整定位到了。现在把这一段从 .v2 ID 方案改成 revision 链方案。同时后面 §16 示例也要改（photo / measurement / crack）。我先把 §15.5 改掉，然后再处理 §16。

继续：**§16 三处示例修正 + §17 加 Defect**
好，§15.5 改完了。现在继续 §16 示例——三个要改的地方：
- §16.2 photo → defect + source_refs
- §16.3 measurement.K001.average → 合法具名聚合（fact_type 前缀必须是 scope_class）
- §16.4 crack → defect

还有 §17 Domain Objects 加 Defect。

好，现在改 **§16.1 补 revision + §16.2 photo→defect**：
好，开始改 §16.2、§16.3、§16.4。

我的 Edit 截断出了问题。让我看看残留情况并清理：
啊，我犯了个错误——Edit 的 old_string 只匹配到了 `{ "id": "fact:photo.034.crack.present", "value": true,`，导致后面的内容（包括原来完整的 §16.2 后半部分和整个 §16.3 开头）现在变成了残留文本。让我先读取当前文件状态，看看完整的残留情况，然后一次性清理。

一次性清理残留 + 修正 §16.3 + §16.4 的 crack→defect：
好，我看到了问题。Edit 只替换了开头几行，原来 §16.2 的后半部分（从 `"unit": null,` 开始到结尾）残留了下来。我需要清理这个残留段，然后同时把 §16.3 和 §16.4 也修正。让我一次性把从残留开始到 crack 示例结束的这几部分全部修正掉。

残留从第 876 行 `"unit": null,` 到第 888 行 ``` 是旧 §16.2 的残留。
然后 §16.3 measurement.K001.average 要改。
然后 §16.4 的 crack.C001.pattern/judgement 要改成 defect.C001.pattern/judgement。

让我从第 876 行开始，把残留 + §16.3 + §16.4 的示例全部替换掉。

现在改 **§17 Domain Objects 加 Defect**：
好，§16 的残留清干净了。现在改 **§17 Domain Objects 加 Defect**。先定位。

03 文档完成。继续 **02_ARCHITECTURE.md + 04_REPORT_IR.md（只修两段式 Fact Ref 为三段式）** + **07_DECISIONS.md（加 D-027）**：
好，03_CANONICAL_DATA_MODEL.md 的所有定点修改完成了！现在更新 Todo，然后继续：02_ARCHITECTURE.md + 04_REPORT_IR.md（修两段式 Fact Ref）→ 07_DECISIONS.md（加 D-027）。

并行读取 02 和 04，找出两段式 Fact Ref：
现在读取 02_ARCHITECTURE.md 和 04_REPORT_IR.md，找出两段式 Fact Ref 的示例。

02 不需要改，04 有三处：两段式 Ref（缺 instance_key）+ crack→defect。并行处理：
好！02_ARCHITECTURE.md 没有 fact: 引用，不需要改。04_REPORT_IR.md 有多处需要修改：
1. 两段式 → 三段式：`fact:project.name`, `fact:building.area` 这些都缺少 instance_key
2. `crack.*` → `defect.*`（因为 crack 不再是 scope_class）
3. `fact:component.status` → 这个 fact_type 可能没注册，但先至少改成三段式

让我读取这几个区域的上下文，然后逐个定点修改。先看第一段（行 156-158）和 crack 段（行 514-519）。

现在精确修正 04_REPORT_IR.md 的三处（两段式 + crack→defect）：
好，我锁定了 04_REPORT_IR.md 的三处修改点。现在逐个 Edit：

1. 行 155-156：两段式 → 三段式
2. 行 498-500：Ref("fact:concrete_strength") 这种格式 → 三段式
3. 行 514-519 和 539-546 和 724：crack.C001.* → defect.C001.*
4. 行 568 和 601：`fact:building.area` → `fact:building.B01.area`

还有 470, 568, 601, 758, 1367, 1766 的 `fact:building.area` 缺 instance_key。

还有 498: `Ref("fact:concrete_strength")` 和 500: `Ref("fact:component.status")` —— 这些也需要改成三段式。

让我先集中处理最关键的：两段式 Ref 和 crack→defect。

好，继续：13.1 Narrative 段的 crack→defect，以及其他区域的 `fact:building.area` → `fact:building.B01.area`。

好，继续 JSON 形式的 crack→defect 替换（行 539-546 区域），以及行 724 的 crack.width。

处理 Ref 示例和其他 `fact:building.area` / `fact:project.name` 缺 instance_key 的段落：
好，继续 Ref 示例段落（行 498-500）和行 721-724 的 crack.width + 行 721-722 的两段式。还有 `fact:building.area` 出现多次缺 instance_key。让我统一处理。先看 Ref 示例那段和行 720-726 的 Ref 示意段。

好，这个 Ref 示例也要改成三段式：
- `Ref("fact:concrete_strength")` → `Ref("fact:component.K001.concrete_strength")`
- `Ref("fact:component.status")` → `Ref("fact:component.K001.status")`（不过 status 这个 fact_type 未必注册，但至少格式合法）

让我也改行 720-726 的 Ref 示意段。

好，现在改行 720-726 的 Ref 示意段（两段式 + crack.width）。

用 `replace_all` 批量修掉剩余的 `fact:building.area` 和 `fact:project.name`：
好，现在把 04_REPORT_IR.md 里剩下的所有 `fact:building.area`（两段式，缺 instance_key）和 `fact:project.name` 也改掉。用 Grep 找出剩余所有需要改的。

Windows PowerShell 下单引号有问题。改用 Python 脚本文件方式。或者我直接写个临时脚本执行。

好，运行脚本。

好，脚本执行成功。现在验证剩余是否有两段式 Fact Ref 还没改掉，然后删掉临时脚本，继续 07_DECISIONS.md。

04 文档清理完毕，所有 fact: 引用都是三段式 + defect 命名。现在改 **07_DECISIONS.md 加 D-027**：
所有 fact: 引用现在都是合法三段式了，crack 也全部替换为 defect。好，继续 07_DECISIONS.md 加 D-027。先看它的索引区域。

先加索引 D-027，然后在文档末尾加详细条目：
好，我看到索引的最后一条是 D-026。我需要先在索引里加 D-027，然后在文档末尾加 D-027 的详细条目。让我先找到 D-026 详细条目的结尾位置。

找到 D-026 结尾后追加 D-027 详细条目：
好，索引加好了。现在找到 D-026 详细条目的结尾，然后追加 D-027。

好，D-026 从行 374 开始。读取到文件末尾，然后追加 D-027。

好，D-026 结尾在行 385 的 `---`。现在把 D-027 插到附录之前。

所有设计文档修改完成！更新 Todo 进入代码阶段：
所有设计文档修改完毕！更新 Todo 并进入代码修改阶段。

先读 registry.py 的关键段落：
现在进入代码修改。先改 registry.py——需要：
1. SCOPE_CLASSES 加 `defect`
2. 删除默认注册中的非法类型：component.crack.width, component.crack.length, point.average
3. 新增 defect 相关 fact_type：defect.id, defect.width, defect.length, defect.pattern, defect.judgement, defect.present

让我先读取 registry.py 当前内容，精确定位要改的段落。

好，继续读 `_build_default_registry` 部分：

好，现在清楚了：
1. SCOPE_CLASSES（行 16-23）需要加 `"defect"`
2. component.crack.width / component.crack.length（行 280-283）要删
3. point.average（我记得之前改过名，应该在 `point.average`，让我继续读确认）
4. 新增 defect fact_type

让我先读完剩余的默认注册部分。

好，三处修改定点执行：
好，完整定位了。现在做三处定点修改：
1. SCOPE_CLASSES 加 defect
2. 删除 component.crack.width / component.crack.length 和 point.average
3. 新增 defect fact_type 条目

好，继续删旧 crack 条目和 point.average，加 defect 条目。

registry 完成。现在改 **id.py（严格三段 + 删除 version suffix）**：
好！registry.py 完成。现在改 id.py——三项修改：
1. parse_fact_id：严格 3 段，任一段不含 '.'
2. 删除 _strip_version_suffix
3. 所有调用点不再需要 version suffix 处理

让我先读当前 id.py 完整内容。

好，整个文件要重构。现在 rewrite 这个文件：
1. parse_fact_id：用 `split(".")` 严格要求恰好 3 段，任一段含 `.` 就会导致超过 3 段从而返回 None
2. FactID 的 `attribute_path` 改名成 `attribute`（与 v0.3.1 一致）
3. 删除 `_strip_version_suffix` 函数
4. is_valid_fact_id 中去掉 version suffix 相关代码
5. 新增检查 fact_type 第一段必须等于 scope_class（新不变式）

让我整个文件改写。

id.py 完成。现在改 **types.py（Fact 加 revision 字段）**：
好，id.py 完整改完。现在改 types.py——Fact 加 revision:int=1 + to_dict/from_dict。让我先读当前 types.py 的 Fact 部分。

三处定点修改 types.py：
好，清晰了。三处定点修改：
1. `_FactFields` 加 `REVISION = "revision"`
2. Fact dataclass 加 `revision: int = 1` 字段
3. `to_dict` 和 `from_dict` 加 revision

让我依次执行。

好，现在加 revision 字段到 Fact dataclass 定义。

types.py 完成。现在改 **validate.py（加 revision 规则 + fact_type 前缀不变式）**：
types.py 完成。现在改 validate.py——需要加 revision 相关的校验逻辑。让我先读当前 validate.py 的后半部分（特别是 _validate_supersedes 和相关代码）。

好，继续读后面的部分，特别是 _validate_supersedes 和 revision 相关的。

改写 validate.py——加 revision 规则 + 更新 supersedes 校验：
好，现在我清楚 validate.py 需要加的 revision 校验逻辑。我需要：
1. 在 validate_fact 里加一个新的校验步 "revision"（大约插入在 step 3 或 4 之前或之后）
2. 新增一个 `_validate_revision_and_supersedes` 辅助函数，检查：
   - revision 必须是正整数
   - revision > 1 → supersedes 必须非空
   - supersedes 里的每个 id 必须和 fact.id 相同（不能跨身份引用）

注意 Advisor 提醒："如果当前 validator 层拿不到完整 store，可以明确把跨对象链完整性留给后续 store 层，但不得实现错误逻辑"。

让我改写 validate.py。

现在更新 `_validate_supersedes`——加 supersedes 必须指向同一身份的规则：
好，现在更新 `_validate_supersedes` 函数——需要加一个规则：supersedes 里的 id 必须和 fact.id 相同（不能跨身份引用）。这是 v0.3.1 的新要求。

源码改完。先跑 pytest 看哪些旧测试挂了，**定点修复**：
所有 4 个源码文件修改完毕！现在先跑一次 pytest 看看哪些旧测试需要改。这是关键的一步——我不应该盲目改测试文件，而是先运行看具体哪些挂掉，再定点修复。

dataclass 字段顺序错了——带默认值的 revision 不能插在无默认的 fact_type 之前。修复：
啊哈！dataclass 初始化顺序问题：我把 `revision: int = 1`（有默认值）插在了 `id: str` 和 `fact_type: str`（都无默认值）之间。Python dataclass 规则是：带默认值的字段必须放在不带默认值的字段之后。

解决方案：把 `revision` 移到 `fact_type` 之后，或者更好的做法——让所有无默认值的字段都在前面，有默认值的在后面。

让我重新排列 Fact 的字段顺序：
1. id (无默认)
2. fact_type (无默认)
3. value (无默认)
4. unit (无默认)
5. quantity_kind (无默认)
6. revision (有默认 1) ← 现在移到这里来
7. source_refs (有默认)
8. method (有默认)
9. provenance (有默认)
10. confidence (有默认 None)
11. review_status (有默认)
12. status (有默认)
13. supersedes (有默认 None)
14. value_domain (有默认 None)

改 types.py 的 Fact 定义。

再跑看具体哪些旧测试挂：
好，重新跑测试看哪些挂了。

5 个旧测试定点修复：

1. `attribute_path` → `attribute`
2. `.v2` ID 测试 → 改成同一 id + revision=2
3. `point.average` → 合法 fact_type
4. `test_parse_attribute_path_with_dots` → 反转成期望失败

同时新增 defect / revision 测试。让我先定位所有要改的行：
好，5 个失败，每一个都很清晰：

1. `test_parse_standard` / `test_build_roundtrip` — 用了 `fid.attribute_path`（已改名成 `attribute`）
2. `test_parse_attribute_path_with_dots` — 之前期望 `fact:component.K001.crack.width` 解析成功，现在要反转成期望失败
3. `test_computed_average` — 之前用 `point.average`（已从 registry 删除）
4. `test_new_fact_corrects_old` — 用了 `.v2` ID（现在要改成同一 id + revision=2）

让我定点修复这些旧测试，然后再加上 defect 和 revision 的新测试。先读一下测试文件里需要改的部分的精确内容。

先修 TestFactID（attribute_path → attribute + 反转 dots 测试 + revision 相关新测试）：
好，非常清晰。现在批量定点修改测试文件。让我按区域依次修改。

先改 TestFactID 类里的 attribute_path → attribute 以及反转 attribute_path_with_dots 测试。

好，批量修 TestFactID 类。

继续修 supersedes 测试和删除已不存在的 point.average 测试：
好，TestFactID 修完。现在修：
1. test_computed_average（删除，因为 point.average 已从 registry 删除）
2. test_new_fact_corrects_old（改成同一 id + revision=2）
3. test_superseded_with_supersedes_fails（检查 old_id != fact.id 的情况）
4. test_illegal_fact_type_not_registered 和 test_id_attr_path_mismatches_fact_type 里的 attribute_path 文字

让我定位并修复。

现在修 **supersedes 测试（.v2 → revision 链）** 和加 **defect / revision 新测试**：
好，现在修 supersedes 测试。先读当前的 supersedes 测试区域。

好，现在我要：
1. 把 test_new_fact_corrects_old 改成 revision=2 + 同一 id
2. 把 test_superseded_with_supersedes_fails 里的 `.v2` 去掉（那个 id 现在也得是同一身份，但 superseded 状态不能带 supersedes）
3. 加 revision 和 defect 新测试

先看完整的 supersedes 测试区域起始行：

完整替换 supersedes 测试块 + 追加 defect / revision 新测试：
好，现在完整改 supersedes 测试块。

让我重新读当前文件的 supersedes 区域的精确内容：
好，之前的 edit 可能已经改掉了那个 test_new_fact_corrects_old 里的 id，但 grep 输出显示它之前是 `fact:component.K001.concrete_strength`（无 .v2）。让我重新读当前实际内容，精确定位。

好，现在精准替换这个 supersedes 测试块：
好，现在看到了精确内容。原来 test_new_fact_corrects_old 的 id 是 `.v2`（之前第一次提交就用了 v0.3 旧方案），而 test_superseded_with_supersedes_fails 的 supersedes 里有 `.v2`。现在精准替换。

找最后的 `.v2` 和 attribute_path 引用：
好，现在还需要：
1. 把最后那个 test_types.py 里还有的 `.v2` ID（第 696 行附近）修掉
2. 在文件末尾加 defect 和 revision 的新测试
3. 加 JSON round-trip 测试里的 revision

先找最后那个 .v2：

好，还有两处 `.v2`（165 和 738）和两处 attribute_path 文字（315 和 677/687）。让我先看 738 附近的内容——应该是 supersedes 的 JSON roundtrip 测试。

修 roundtrip 里的 `.v2` + 加 revision 到所有 roundtrip 测试 + 在文件末尾加 defect/revision 新测试：
好，test_supersedes_roundtrip 里也有 `.v2`。让我修掉它，同时也加 revision 到这个测试里，并修掉 test_measured_roundtrip。然后加完整的 defect + revision 测试块。

先修 test_supersedes_roundtrip：

在文件末尾追加 defect + revision 测试块：
好，现在在文件末尾加上 defect / revision 测试类。同时修掉那个 attribute_path 文字引用（改成 attribute）。让我先定位文件末尾：

好，让我找到文件的最末尾。

好，TestConflictJSON 是最后一个测试类。让我读到文件末尾。

好，文件结束在 902 行。现在在末尾追加 defect + revision 测试块。

好，最后跑 pytest 看还有多少挂：
好，现在还要修最后两处：
1. test_attr_path_mismatches_fact_type 里的 "attribute_path mismatch" → "attribute"
2. 还有那个 TestFactTypeFailures 里的 test_id_attr_path_mismatches_fact_type

让我先跑一次 pytest 看看还有多少挂，然后定点修。

只差 3 个——defect 定性 Fact 缺 `unit=None`：
好，71 passed，只剩 3 个——都是 defect 测试里缺了 `unit=None`。快修！

最终 pytest：
好，最后跑一次完整 pytest！

最后一个——dimensionless 要求 `unit` 非 None（可用空字符串）：
只剩一个！dimensionless 的 unit 规则——validator 里 dimensionless 要求 unit 非 None（可以是空字符串）。改测试用 `unit=""` 即可。

好，最终跑全量！

**74 passed, 0 failed, 0 error**。现在最终报告：
**74 passed, 0 failed, 0 error！** 全量绿了。现在输出最终报告。先更新 Todo，然后 git status + diff。

好，现在 git status + diff stat。

好，code/tests 都是 untracked（新文件），所以 diff --stat 没输出。让我用 git diff --stat HEAD 或者直接 ls src/ tests/ 来看。

好，全部完成。现在输出最终报告。
---

# Step 7-B 修正版 完成报告

## 一、修改的设计文档

| 文件 | 变更 | 行数 |
|---|---|---|
| `docs/03_CANONICAL_DATA_MODEL.md` | +v0.3.1 修订说明；§8 加 revision 字段表；§8.1 scope_class 加 defect + grammar 冻结 + fact_type 不变式写入；§9 crack→defect；§15.2 building.year 补 instance_key；§15.5 改 revision 链替代 .v2；§16 改 photo→defect + measurement→合法具名聚合 + crack→defect；§17 Component 下加 Defect 弱实体 | +127 / -48 |
| `docs/04_REPORT_IR.md` | 34 处 Fact Ref 全部改成合法三段式（`fact:project.PROJ01.name`, `fact:building.B01.area` 等）；7 处 `crack.*` → `defect.*`；1 处旧 Ref("fact:concrete_strength") → Ref("fact:component.K001.concrete_strength") | +50 / -? |
| `docs/07_DECISIONS.md` | 索引加 D-027；末尾加 D-027 完整条目（scope_class 澄清 + defect 新增 + Fact ID 冻结 + revision 从 ID 移出） | +16 |
| `docs/02_ARCHITECTURE.md` | **无改动**（grep 确认无 fact: 引用） | 0 |

## 二、修改的代码文件

| 文件 | 变更 |
|---|---|
| `src/cdm/registry.py` | SCOPE_CLASSES 加 `defect`；删除默认注册中 `component.crack.width` / `component.crack.length` / `point.average`；新增 6 个 defect fact_type（id / width / length / pattern / judgement / present） |
| `src/cdm/id.py` | parse_fact_id：`split(".")` 严格要求恰好 3 段（不再用 `split(".", 2)`），任一段含 `.` 自动失败；FactID.attribute_path → attribute；删除 `_strip_version_suffix`；is_valid_fact_id 新增 fact_type 前缀不变式校验（fact_type 第一段必须等于 ID 的 scope_class） |
| `src/cdm/types.py` | Fact dataclass 加 `revision: int = 1`（正确放在所有无默认字段之后）；_FactFields 加 REVISION 常量；to_dict 输出 revision；from_dict 读取 revision（默认 1 向后兼容） |
| `src/cdm/validate.py` | 新增 step 2 revision 校验（正整数 + revision>1 必须 supersedes）；_validate_supersedes 加新规则：supersedes 条目必须指向同一 fact identity（`old_id != fact.id` 报错，跨身份引用是 Conflict 不是 correction）；明确注释跨 revision 链完整性留给 Store 层 |

## 三、修改的测试

`tests/cdm/test_types.py`（从 63 → 74 个用例）

| 改动 | 内容 |
|---|---|
| TestFactID | `attribute_path` → `attribute`（2 处）；反转 `test_parse_attribute_path_with_dots` → `test_parse_attribute_with_dots_REJECTED`（确认 4 段失败）；新增 4 个测试：`test_parse_four_segments_REJECTED` / `test_defect_scope_class_accepted` / `test_v2_suffix_REJECTED` / `test_fact_type_prefix_mismatch_REJECTED` |
| TestFactFilledQuantitative | 删除已不存在的 `point.average` 测试 |
| TestFactSupersedes | 全部重写：旧 `.v2` ID → 同 id + `revision=2`；旧 superseded+supersedes 反向测试改 revision+supersedes+superseded 三重矛盾；新增 `test_supersedes_different_identity_fails` |
| **TestDefectFacts（新）** | 6 个用例：defect.width / defect.length（定量）、defect.pattern / defect.judgement（定性）、defect.present（布尔 dimensionless）、defect.unknown_attr 非法属性失败 |
| **TestFactRevision（新）** | 6 个用例：revision 默认 1 / revision=2 / revision=0 失败 / revision<0 失败 / revision=2 无 supersedes 失败 / revision=2 + 同 id supersedes 合法 / revision=3 跳过链（Validator 允许，Store 层检查） |
| TestFactJSON | supersedes roundtrip 加 revision=2；输出校验加 `d["revision"] == 2` |

## 四、测试结果

```
python -m pytest tests/
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
collected 74 items
74 passed in 0.17s
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**0 failed, 0 error, 0 skipped**。

## 五、v0.3 → v0.3.1 关键变化速查

| 维度 | v0.3 | v0.3.1 |
|---|---|---|
| scope_class 数量 | 6（无 defect） | 7（新增 defect） |
| Fact ID 段数 | 3（attribute 可含 `.`） | 严格 3（任一段不含 `.`） |
| `.v2` 版本后缀 | 允许（attribute 上） | **禁止** |
| revision | 不存在 | `int`，默认 1，>1 必须 supersedes |
| fact_type 不变式 | 未显式写入 | 正式：`fact_type.第一段 == ID.scope_class` |
| supersedes 身份约束 | 无 | 必须指向同一 id（跨身份=Conflict） |
| crack 作为 scope_class | 隐含允许 | **不允许**，crack → defect 实例 |
| measurement 作为 scope_class | 隐含允许 | **不允许**，用具名合法 scope_class |

## 六、与 Architecture / Report IR 的一致性

| 文档 | 一致性 |
|---|---|
| 02_ARCHITECTURE.md | 无 fact: 引用，未改。架构整体 V2 结构未动 ✓ |
| 03_CANONICAL_DATA_MODEL.md | v0.3 → v0.3.1 定点修订 10 处，无结构变更 ✓ |
| 04_REPORT_IR.md | 34 处 Ref 全部三段式合法；7 处 crack→defect；anchors/covers/fact_display_spec 等未动 ✓ |
| 07_DECISIONS.md | D-027 记录取代关系（推翻 D-020 中 attribute_path 可含 `.` 的隐含假设） ✓ |

## 七、剩余问题

| 项 | 状态 | 处理 |
|---|---|---|
| `required_facts[].scope` vs `scope_class` 语义分离 | 03 §8.1 已写明区分 | 不进 CDM 层，留给后续 TemplateSpec 实现 |
| `superseded` Fact 带 `revision` 但不带 `supersedes` | Validator 已允许（revision 语义只要求 NEW→OLD 方向） | Store 层负责维护 revision 链完整性 |
| `fact:component.K001.status` 等 fact_type | 04 出现但未注册 | 第一阶段最小集不含 status 类型，不阻塞 |
| 跨 revision 链完整性（revision=3 必须 supersedes revision=2，不允许跳） | Validator 只做结构正确检查 | Store 层保证，本轮不实现 |

## 八、Git status

```
On branch master
design-baseline-m0
HEAD: f483072 chore: 建立工程设计基线

Modified (tracked):
  M docs/03_CANONICAL_DATA_MODEL.md   (+127/-48)
  M docs/04_REPORT_IR.md             (34 处 Ref 修正 + 7 处 crack→defect)
  M docs/07_DECISIONS.md             (+16)

Untracked (new files, Step 7-B 代码):
  pytest.ini
  src/cdm/__init__.py
  src/cdm/registry.py
  src/cdm/id.py
  src/cdm/types.py
  src/cdm/validate.py
  tests/__init__.py
  tests/cdm/__init__.py
  tests/cdm/test_types.py
  reviews/设计澄清会话-review.md (用户会话文件)
  流水线机械重复示意.md (02_ARCHITECTURE.md 的备份?)
  开发流水线.md
  编程要求.md
```

**未 commit**（按要求）。

## 九、停止

Step 7-B 修正版到此结束。**不进入 Step 7-C**。等待确认后再继续。
