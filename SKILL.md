---
name: scipilot-cite
description: >-
 SciPilot 家族的文献检索与引用插入技能。读取用户的 Word(.docx) 或 LaTeX(.tex) 论文，
 根据论文主题和用户需求，通过 Semantic Scholar、OpenAlex、Crossref 等学术API
 检索近5年高水平真实论文，验证每篇文献的真实性（DOI可解析、元数据一致），
 然后按指定格式（IEEE/APA/Nature/Vancouver/GB-T-7714等）将引用插入论文正文
 对应位置，并在 References 章节按顺序列出所有参考文献。
 当用户提到以下任何情况时触发此技能：添加参考文献、插入引用、找文献、
 citation、reference、加引用、补充参考文献、文献检索、add references、
 find papers、insert citations，或者用户上传了论文文件并要求增加文献引用。
 即使用户只是说"帮我的论文加几篇参考文献"或"scipilot帮我找文献"，也应该触发此技能。
 属于 SciPilot 科研技能家族。
license: MIT
---

# SciPilot-cite — 学术文献自动检索与引用插入

> SciPilot 家族成员 | 你的文献引用副驾驶

## 概述

SciPilot-cite 将真实、可验证的学术文献自动检索并插入用户论文。
所有文献必须来自学术数据库 API 的真实检索结果，严禁编造或虚构任何文献信息。

## IRON RULES（不可违反的铁律）

1. **真实性铁律**：每一篇引用的文献必须通过 DOI 验证或至少两个独立学术数据库交叉确认。绝对不允许编造任何论文标题、作者、期刊、年份、DOI。如果无法验证某篇文献的真实性，必须丢弃它，宁缺毋滥。
2. **顺序铁律**：引用编号必须严格按照在正文中首次出现的顺序，从 [1] 开始递增。References 列表的排列顺序必须与正文引用编号完全一致。任何编号跳跃、重复或乱序都是严重错误。
3. **时效铁律**：默认只检索近 5 年的文献（用户可自定义）。每篇文献的发表年份必须在允许范围内，过期文献自动排除。
4. **质量铁律**：优先选择高引用量、发表在知名期刊/会议的文献。预印本（arXiv 等）仅在用户明确允许时才包含。
5. **格式一致铁律**：同一篇论文中所有引用必须使用完全相同的格式规范，不允许混用不同引用风格。
6. **不破坏铁律**：插入引用时不得修改论文的任何原有内容（文字、图表、公式等），只允许在合适位置添加引用标记和 References 章节。
7. **证据账本铁律**：最终 bibliography 中的**每一篇**文献都必须在工作目录的 `verification_log.jsonl` 中有对应 `paper_id` 条目，且 `verdict ∈ {VERIFIED, LIKELY_REAL}`。Stage 7 必须运行 `scripts/audit_no_hallucination.py` 做末端 100% 重验证 + 与日志对账，**只有 audit 返回 PASS 才能交付**；任何 FAIL 必须中断、向用户报告疑似幻觉条目，不得"凭印象"添加任何未经脚本验证的论文。

## 触发条件

当用户意图涉及以下任一场景时激活：
- 为论文添加/补充参考文献
- 检索特定主题的学术论文
- 在论文中插入引用标记
- 格式化已有的参考文献列表
- 用户上传了 .docx 或 .tex 文件并提到"引用""参考文献""citation""reference"
- 用户提到 "scipilot" 并涉及文献/引用相关内容

## 工作流程

下面 8 个 Stage 必须严格按顺序执行。每个 Stage 完成前，不得跳跃到后续 Stage。

### Stage 0：信息收集（必须完成）

在执行任何操作之前，必须先向用户确认以下信息。对于有默认值的项，告知用户默认值并询问是否需要修改。

**必须询问的问题：**

1. **论文文件**：请用户提供 .docx 或 .tex 文件（如果用户还没有上传）。
2. **需要添加的文献数量**：用户希望添加多少篇？
 - 建议范围：5-30 篇
 - "适量" 时的建议：短论文 (4-6 页) → 10-15 篇；中等 (8-12 页) → 15-25 篇；长论文 (>12 页) → 25-40 篇
3. **引用格式**：
 - **IEEE**（数字编号 [1][2][3]，工程/计算机类常用）← 默认
 - **APA 7th**（作者-年份，心理学/社会科学）
 - **Nature**（上标数字，自然科学顶刊）
 - **Vancouver**（数字编号，医学/生物）
 - **GB/T 7714-2015**（中文论文国标）
 - **自定义**（用户提供格式说明）
4. **文献年限范围**：默认近 5 年（依当前年份动态计算），用户可自定义。
5. **是否包含预印本**：默认 **不包含** arXiv 等预印本。
6. **特殊要求**（可选）：必须引用的论文 / 偏好的期刊会议 / 研究领域关键词。

收集完信息后，必须向用户**口头复述全部参数**并请求确认，然后才进入 Stage 1。

### Stage 1：论文分析

读取用户论文文件并提取结构。

- **.tex**：直接读源码，识别 `\section{}`、`\cite{}`，记录已有引用与 bibliography 类型。
- **.docx**：用 `python-docx` 读取段落、Heading 样式、已有 References。

输出：论文标题、各章节列表、已有引用列表、核心关键词（自动 + 用户补充）。

### Stage 2：文献检索

调用 `scripts/search_papers.py`。

- 并行查询 Semantic Scholar / OpenAlex / Crossref
- 每个 API 抓取 `ceil(用户需求数量 × 3)` 篇候选
- 按引用量降序
- 过滤年份范围
- 若 `include_preprint=False`，剔除 venue 含 arxiv/biorxiv/medrxiv 的条目
- DOI 主键去重；无 DOI 时按 (标题相似度 >0.9 + 同年份 + 同首作者姓) 合并

### Stage 3：文献验证（关键步骤）

调用 `scripts/verify_paper.py::batch_verify`。

验证分级：
- [OK] **VERIFIED**：DOI 可解析、Crossref 元数据匹配（标题相似度 ≥0.85，年份精确，首作者姓匹配）
- [WARN] **LIKELY_REAL**：DOI 缺失或验证失败，但 Semantic Scholar 和 OpenAlex 都能搜到同一标题（≥0.85 相似）
- [FAIL] **UNVERIFIED**：上述都不通过 → **直接丢弃，不得引用**

只有 VERIFIED 和 LIKELY_REAL 的文献进入下一步。

### Stage 4：文献筛选与排序

从候选中按以下加权挑选用户需要的数量：
- 相关性 40%（文献摘要与论文章节的语义相关）
- 质量 30%（引用量 + 期刊级别）
- 时效 20%（越新越好）
- 多样性 10%（避免同一作者团队集中）

**向用户展示候选清单**：每条含标题、作者、年份、venue、引用量、验证状态、建议章节。
**等待用户确认或要求替换**，若替换则回到 Stage 2。

### Stage 5：引用插入

- **.tex** → `scripts/insert_citations_latex.py`：用 `\cite{key}` 插入，并维护 `thebibliography` 或 BibTeX `.bib` 文件
- **.docx** → `scripts/insert_citations_docx.py`：插入 `[N]` 文本标记，在文末加 References 章节

编号规则：从正文第一段开始按首次出现顺序分配 `[1] [2] [3]...`；已有引用从最大编号 +1 开始（或用户选择重新编号）。

插入位置规则：
- 在相关论述句子末尾、句号之前
- 一个位置可有多个引用：`[3,5,7]`
- 禁止插入到图表标题、公式、章节标题中

### Stage 6：格式化输出

调用 `scripts/format_citation.py::format_citation(paper, style)` 按用户指定格式生成每条参考文献。

格式样例（详见 `references/citation-formats.md`）：

```
IEEE [1] A. Author, B. Author, "Title," *Journal*, vol. X, 2024, doi: 10.xxxx/xxxxx.
APA 7 Author, A., & Author, B. (2024). Title. *Journal*. https://doi.org/10.xxxx/xxxxx
Nature 1. Author, A., Author, B. Title. *Journal* 600, 1-10 (2024).
Vanc. 1. Author A, Author B. Title. Journal. 2024.
GB/T [1] 作者. 题名[J]. 刊名, 年, 卷(期): 起止页码.
```

### Stage 7：最终检查与交付

#### 第 0 项（阻塞性幻觉门控 — Gate 8）

**这是第一项，先于其他自检执行，失败立即中断整个交付流程。**

把最终采用的文献清单写到 `final_papers.json`，然后强制运行：

```bash
python scripts/audit_no_hallucination.py final_papers.json \
       --log verification_log.jsonl \
       --report audit_report.json
```

捕获 exit code：
- `0` → audit PASS，所有引用通过 100% 重验证 + 日志对账，继续后续自检
- `2` → audit FAIL，立刻中断 Stage 7。读 `audit_report.json` 的 `per_paper` 段，把所有 `pass=false` 的条目逐条向用户报告（标题、DOI、失败原因），询问用户是要丢弃这些条目（推荐）、用替代论文重检（回 Stage 2）、还是放弃整次操作
- `3` → 运行性错误（文件缺失/格式错误），修复后重跑

**绝对不允许**：跳过 audit、伪造 audit 报告、把 FAIL 条目当 PASS 处理。这条违反了 IRON RULE 7。

#### 第 1-5 项（仅在第 0 项 PASS 后执行）

1. **编号连续性**：正文 [1] 到 [N] 无跳跃、无重复
2. **引用-列表一致性**：每个正文编号在 References 都有；References 无未引用孤儿
3. **格式一致性**：所有条目同一 style
4. **原文完整性**：diff 比对确认未修改原文
5. **预印本占比**：若 `include_preprint=False`，复查 venue 不含 arxiv

交付时输出一份 **SciPilot 引用报告**：
- 总计添加 N 篇
- audit_report.json 的 PASS/FAIL 摘要
- 各文献的验证状态、来源 API
- 各章节引用分布
- 任何 LIKELY_REAL 文献必须明确标注

## 错误处理

- **API 失败**：自动重试 3 次（间隔 2s 指数退避）；某 API 持续失败则用其他 API 补充
- **数量不足**：告知用户实际找到数量，询问是否放宽搜索或接受较少
- **解析异常**：请用户确认文件完整性

## 与 SciPilot 家族协作

- 引用插入后 → 推荐 **scipilot-polish** 润色新增引用句
- 全面审查 → 推荐 **scipilot-review**
- 图表配合 → 推荐 **scipilot-figure**

## 依赖

```
pip install requests python-docx python-Levenshtein
```
所有 API 都是免费访问层级，无需 API key。

## 工作目录产物（必须保留至交付完成）

| 文件 | 由谁生成 | 用途 |
|---|---|---|
| `evidence_log.jsonl` | `search_papers.py` 每条结果落盘 | 证明每篇候选都来自真实 API 响应 |
| `verification_log.jsonl` | `verify_paper.py::batch_verify` 每条验证落盘 | Stage 7 第 0 项的对账依据 |
| `final_papers.json` | LLM 在 Stage 4 用户确认后写出 | audit 的输入 |
| `audit_report.json` | `audit_no_hallucination.py` 输出 | 给用户的 Gate 8 凭证 |

**这四个文件构成本 Skill 的"证据链"**。LLM 没有写权限直接产生 evidence/verification 日志——必须通过运行对应的 Python 脚本来产生。这是把"不编造"从口号变成机器可检查契约的核心机制。

## 参考文件

- `references/citation-formats.md` — 各引用格式完整规范与示例
- `references/api-reference.md` — Semantic Scholar / OpenAlex / Crossref API 详细文档
- `references/workflow.md` — 完整工作流程与边界情况处理
- `assets/format_templates/*.json` — 各格式的 JSON 模板（可被自定义覆盖）
