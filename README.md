# SciPilot-cite

> SciPilot family. Citation copilot for academic writing.
> SciPilot 家族成员 — 学术写作的引用副驾驶。

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Python: 3.9+](https://img.shields.io/badge/Python-3.9%2B-3776AB.svg)](#dependencies--依赖)
[![Status: v1.0.0](https://img.shields.io/badge/Status-v1.0.0-success.svg)](#)
[![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-Skill-orange.svg)](https://claude.com/claude-code)

A [Claude Code](https://claude.com/claude-code) / [Codex](https://github.com/openai/codex) / Cursor Skill that **discovers, verifies, and inserts academic references** into your LaTeX or Word manuscript. Backed by three independent scholarly APIs with an authenticity-first verification pipeline that **never fabricates a citation**.

> [中文文档](#中文文档) | [English](#english)

---

## 中文文档

### 概览

`scipilot-cite` 解决学术写作中最容易出错也最伤诚信的环节 —— **引用**。它读取你的 `.tex` 或 `.docx` 论文，通过 **Semantic Scholar + OpenAlex + Crossref** 三源并行检索近年高引文献，对每篇候选执行 **DOI 解析 + 多源交叉验证**，把无法验证的全部丢弃，按你指定的格式（IEEE / APA 7 / Nature / Vancouver / GB/T 7714）将引用插入正文对应位置并生成 References 章节。

**核心承诺：**
- 一篇都不编造 — 验证失败的文献直接丢弃，宁缺毋滥
- 引用编号严格按正文首次出现顺序，零跳跃零孤儿
- 原文一字不动 — 只追加引用标记和 References，不修改任何已有内容

### 核心特性

| 维度 | 实现 |
|---|---|
| 数据源 | Semantic Scholar（200M+ 论文）、OpenAlex（250M+）、Crossref（150M+ DOI 权威） |
| 检索方式 | 三源 ThreadPool 并行，按引用量降序合并，DOI 主键去重 + 标题模糊去重 |
| 真实性验证 | 三档分级：`VERIFIED`（DOI 命中）/ `LIKELY_REAL`（双源交叉）/ `UNVERIFIED`（丢弃） |
| 引用格式 | IEEE / APA 7 / Nature / Vancouver / GB/T 7714-2015 + BibTeX 导出 |
| 文档格式 | LaTeX `.tex`（`\cite{}` + `thebibliography` 或 BibTeX）、Word `.docx`（保持原格式） |
| 工作流 | 8 个 Stage 严格顺序，每个 Stage 内置质量门 |
| 依赖 | `requests` + `python-docx` + `python-Levenshtein`，**全部 API 免费**，无需 key |

### 安装

#### 方式 A：让 Claude Code / Codex 自己装（推荐）

在终端启动 Claude Code 或 Codex，直接说：

```
请帮我安装这个 Skill：https://github.com/haojaeyang-stack/scipilot-cite.git
```

AI 会自动 `git clone` 到正确目录（`~/.claude/skills/` 或 `~/.codex/skills/`），并提示你装 Python 依赖。

#### 方式 B：手动 clone

```bash
git clone https://github.com/haojaeyang-stack/scipilot-cite.git ~/.claude/skills/scipilot-cite
pip install -r ~/.claude/skills/scipilot-cite/requirements.txt
```

Codex 用户把目标目录换成 `~/.codex/skills/`，Cursor 用户换成 `.cursor/skills/`。

#### 方式 C：下载 ZIP

1. 在 GitHub 仓库页面点 `Code` → `Download ZIP`
2. 解压到 `~/.claude/skills/scipilot-cite/`
3. `pip install -r requirements.txt`

### 使用

#### 1. 自然语言触发

启动 Claude Code 后随便用一句中文或英文：

```
帮我的 paper.tex 加 15 篇参考文献，用 IEEE 格式。
```

```
搜索 large language model alignment 相关近 3 年高引论文，
排除预印本，用 Nature 格式插入到 manuscript.docx。
```

```
读取 thesis.tex，在 Related Work 部分补充 10 篇引用，
按 GB/T 7714 格式。
```

Skill 会在 Stage 0 逐项确认参数（论文文件 / 篇数 / 格式 / 年限 / 预印本 / 特殊要求），全部确认后再开始检索 → 验证 → 插入。**关键决策点会等你拍板**。

#### 2. 命令行直接调脚本

不通过 Skill 也能独立使用每个脚本：

```bash
# 三源并行检索
python scripts/search_papers.py "diffusion model" --limit 10 --json > papers.json

# 验证一个 DOI
python scripts/verify_paper.py doi 10.1038/s41586-024-07421-0 \
       --title "Detecting hallucinations..." --year 2024

# 单条引用格式化
echo '{"title":"...","authors":["..."],"year":2024,"venue":"...","doi":"..."}' | \
  python scripts/format_citation.py --style ieee --number 1

# 端到端插入 LaTeX
python scripts/insert_citations_latex.py paper.tex papers.json --style ieee

# 端到端插入 Word
python scripts/insert_citations_docx.py paper.docx papers.json --style nature
```

### 工作流（8 个 Stage）

```
Stage 0  参数收集  (篇数 / 格式 / 年限 / 预印本 / 特殊要求)
   |
Stage 1  论文分析  (章节抽取 / 已有引用 / 关键词)
   |
Stage 2  文献检索  (S2 / OpenAlex / Crossref 并行 + DOI 去重)
   |
Stage 3  真实性验证 (DOI / 跨源 → VERIFIED / LIKELY_REAL / UNVERIFIED)
   |
Stage 4  筛选排序  (相关性40% + 质量30% + 时效20% + 多样性10%)
   |
   *** 用户确认候选清单后才能继续 ***
   |
Stage 5  引用插入  (LaTeX \cite{} / docx [N] 标记)
   |
Stage 6  格式化输出 (IEEE / APA / Nature / Vancouver / GB/T 7714)
   |
Stage 7  最终自检  (编号连续性 / 引用孤儿 / 真实性复核 / 原文 diff)
```

### 真实性验证分级

| 等级 | 触发条件 | 处置 |
|---|---|---|
| `VERIFIED` | DOI 在 Crossref 命中，且标题相似度 ≥ 0.85、年份精确、首作者姓匹配 | 直接采用 |
| `LIKELY_REAL` | 无 DOI 或 DOI 不匹配，但 Semantic Scholar 与 OpenAlex 都搜到该标题（相似度 ≥ 0.85） | 采用，在最终报告中**显式标注** |
| `UNVERIFIED` | 上述都不通过 | **直接丢弃，不允许引用** |

Stage 7 会随机抽取 20% 已采用文献重新调 Crossref 复核 — 这一层是为了防止单次 API 状态异常导致的假阳性 VERIFIED。

### 支持的引用格式

| 格式 | 主用领域 | 正文标记 | 排列方式 |
|---|---|---|---|
| **IEEE** | 工程、计算机、通信 | `[1]` `[1, 2]` `[1-3]` | 首次出现顺序 |
| **APA 7th** | 心理学、教育学、社科 | `(Author, Year)` | 第一作者字母序 |
| **Nature** | 自然科学顶刊 | 上标 `¹` `¹⁻³` | 首次出现顺序 |
| **Vancouver** | 医学、生物医学 | `(1)` `(1-3)` | 首次出现顺序 |
| **GB/T 7714-2015** | 中文期刊、硕博论文 | `[1]` | 引用顺序或著者-出版年 |

完整格式规范、模板、真实示例见 [`references/citation-formats.md`](references/citation-formats.md)。

### 限制（请如实了解）

- **检索召回不是 100%**：API 索引覆盖不全，冷门方向或极新文献可能漏掉。需要你提供必引论文时，Skill 会强制要求 DOI 或完整元数据，不会自己编。
- **相关性评分基于关键词匹配**：摘要与你的章节有词面重叠才会被认为相关；对纯方法学论述（无具体术语）可能选不准最贴切的文献。
- **Word 格式适配是基础级**：保留段落和 Heading 样式，但复杂的 Field Code、自动编号交叉引用不支持。投顶刊建议导出 LaTeX。
- **不能自动判断需要"哪类"引用**：是否需要综述性引用、对比性引用、方法学引用，由 Stage 4 评分给出建议，最终需要你确认。
- **API 速率限制存在**：S2 默认匿名 5000 次 / 5 分钟，大批量场景会被延迟。

### SciPilot 家族

| Skill | 状态 | 功能 |
|---|---|---|
| **scipilot-cite** | v1.0.0 (本仓库) | 文献检索与引用插入 |
| scipilot-polish | 规划中 | 学术论文润色 |
| scipilot-review | 规划中 | AI 模拟审稿 |
| scipilot-figure | 规划中 | 科研图表生成 |
| scipilot-submit | 规划中 | 投稿格式适配 |
| scipilot-read | 规划中 | 论文阅读与翻译 |

家族成员共享四条设计原则：
1. **AI 是副驾驶**：关键决策点等用户拍板
2. **真实性第一**：不编造任何学术信息
3. **一手文献驱动**：规则来自真实期刊规范，不靠"一般感觉"
4. **格式即法律**：同一论文格式必须绝对一致

### 路线图

- **v1.1**: 增加 ACS / ACM Reference Format / Chicago / MLA 支持
- **v1.2**: 接入 PubMed 与 ADS（天文）作为可选数据源
- **v1.3**: 支持已有 `.bib` 文件合并 / 去重 / 升级
- **v2.0**: 与 `scipilot-polish` 联动，引用插入后自动润色对应句子

### 贡献

欢迎 Issue 报告 bug、Feature Request、新引用格式贡献。提 PR 前请：
1. 确保 `scripts/utils.py`、`scripts/format_citation.py` 的内嵌测试能跑通
2. 新增格式需同时更新 `assets/format_templates/<style>.json` 和 `references/citation-formats.md`
3. SKILL.md 改动控制在 500 行内，详细内容放 `references/`

### 许可证

[MIT](LICENSE) © 2026 Haojae

---

## English

### Overview

`scipilot-cite` solves one of the most error-prone (and integrity-sensitive) parts of academic writing — **citations**. Hand it a `.tex` or `.docx` manuscript and it will search recent high-citation papers across **Semantic Scholar + OpenAlex + Crossref** in parallel, run each candidate through **DOI resolution + multi-source cross-check**, drop everything that fails verification, then insert the survivors into your prose at the right spots and build the References section in the style you choose (**IEEE / APA 7 / Nature / Vancouver / GB/T 7714**).

**Core guarantees:**
- Never fabricates a reference — failed candidates are discarded, not invented
- Reference numbers strictly follow first-occurrence order, no gaps, no orphans
- Your manuscript prose stays byte-identical except for the added markers and References

### Key Features

| Dimension | Implementation |
|---|---|
| Data sources | Semantic Scholar (200M+ papers), OpenAlex (250M+), Crossref (150M+ authoritative DOIs) |
| Retrieval | ThreadPool parallel across 3 APIs, sort by citation count, DOI-primary dedup + fuzzy title dedup |
| Authenticity | Three tiers: `VERIFIED` (DOI hit) / `LIKELY_REAL` (cross-source) / `UNVERIFIED` (dropped) |
| Citation styles | IEEE / APA 7 / Nature / Vancouver / GB/T 7714-2015 + BibTeX export |
| Document formats | LaTeX `.tex` (`\cite{}` + `thebibliography` or BibTeX), Word `.docx` (format-preserving) |
| Workflow | 8 sequential stages, each gated by quality checks |
| Dependencies | `requests` + `python-docx` + `python-Levenshtein`, **all APIs are free**, no key required |

### Installation

#### Option A: Let Claude Code / Codex install it (recommended)

Open Claude Code or Codex in your terminal and just type:

```
Please install this Skill for me: https://github.com/haojaeyang-stack/scipilot-cite.git
```

The agent will `git clone` into the right directory (`~/.claude/skills/` or `~/.codex/skills/`) and prompt you for the Python deps.

#### Option B: Manual clone

```bash
git clone https://github.com/haojaeyang-stack/scipilot-cite.git ~/.claude/skills/scipilot-cite
pip install -r ~/.claude/skills/scipilot-cite/requirements.txt
```

Codex users: target `~/.codex/skills/`. Cursor users: target `.cursor/skills/`.

#### Option C: Download ZIP

1. Click `Code` → `Download ZIP` on the GitHub page
2. Extract into `~/.claude/skills/scipilot-cite/`
3. `pip install -r requirements.txt`

### Usage

#### 1. Natural language trigger

Inside Claude Code, plain prose in English or Chinese:

```
Add 15 references to my paper.tex in IEEE format.
```

```
Find recent (last 3 years) high-citation papers on large language model
alignment, exclude preprints, insert them into manuscript.docx in Nature style.
```

```
Read thesis.tex, add 10 citations to the Related Work section in
GB/T 7714 format.
```

The Skill confirms all parameters in Stage 0 (paper file / count / style / year range / preprint policy / special requirements) before running the search → verify → insert pipeline. **It waits for your sign-off at every decision point.**

#### 2. Standalone CLI

Each script also runs on its own:

```bash
# Parallel multi-source search
python scripts/search_papers.py "diffusion model" --limit 10 --json > papers.json

# Verify a real DOI
python scripts/verify_paper.py doi 10.1038/s41586-024-07421-0 \
       --title "Detecting hallucinations..." --year 2024

# Format one entry
echo '{"title":"...","authors":["..."],"year":2024,"venue":"...","doi":"..."}' | \
  python scripts/format_citation.py --style ieee --number 1

# End-to-end LaTeX insertion
python scripts/insert_citations_latex.py paper.tex papers.json --style ieee

# End-to-end Word insertion
python scripts/insert_citations_docx.py paper.docx papers.json --style nature
```

### Workflow (8 stages)

```
Stage 0  Parameter collection (count / style / years / preprint policy / extras)
   |
Stage 1  Manuscript analysis (sections / existing cites / keywords)
   |
Stage 2  Search (S2 / OpenAlex / Crossref in parallel, dedup by DOI)
   |
Stage 3  Verification (DOI / cross-check -> VERIFIED / LIKELY_REAL / UNVERIFIED)
   |
Stage 4  Ranking (relevance 40 + quality 30 + recency 20 + diversity 10)
   |
   *** Wait for user confirmation of candidate list ***
   |
Stage 5  Insertion (LaTeX \cite{} / docx [N] markers)
   |
Stage 6  Format output (IEEE / APA / Nature / Vancouver / GB/T 7714)
   |
Stage 7  Final audit (numbering continuity / citation orphans / re-verify / diff)
```

### Verification tiers

| Tier | Trigger | Handling |
|---|---|---|
| `VERIFIED` | DOI hits Crossref; title similarity ≥ 0.85, exact year, first-author surname match | Accepted silently |
| `LIKELY_REAL` | No DOI or DOI mismatch, but both Semantic Scholar and OpenAlex return the same title (similarity ≥ 0.85) | Accepted, **explicitly flagged** in the final report |
| `UNVERIFIED` | None of the above | **Discarded, not citable** |

Stage 7 randomly resamples 20% of accepted references and re-queries Crossref — a second layer of defense against single-shot false positives.

### Supported citation styles

| Style | Discipline | In-text marker | Ordering |
|---|---|---|---|
| **IEEE** | Engineering, CS, communications | `[1]` `[1, 2]` `[1-3]` | First occurrence |
| **APA 7th** | Psychology, education, social sciences | `(Author, Year)` | Alphabetical by first author |
| **Nature** | Top natural-science journals | Superscript `¹` `¹⁻³` | First occurrence |
| **Vancouver** | Medicine, biomedicine | `(1)` `(1-3)` | First occurrence |
| **GB/T 7714-2015** | Chinese journals, theses | `[1]` | First occurrence or author-year |

Full specs, templates and real examples in [`references/citation-formats.md`](references/citation-formats.md).

### Limitations (read honestly)

- **Recall is not 100%**: API index coverage varies; very niche or very recent work may be missed. When you supply mandatory references, the Skill requires a DOI or full metadata — it will never invent one.
- **Relevance scoring is keyword-based**: candidates score higher when their abstracts share vocabulary with your sections. For pure-methodology prose without distinctive terminology, top choices may not be optimal.
- **Word format support is basic**: paragraphs and Heading styles are preserved; complex Field Codes and auto-numbered cross-references aren't. For high-impact submissions, prefer LaTeX.
- **The Skill does not auto-decide citation function**: whether a slot needs a survey, a method comparison, or a benchmark citation is suggested in Stage 4 but ultimately needs your confirmation.
- **API rate limits apply**: Semantic Scholar defaults to 5000 calls / 5 min for anonymous use; very large batches will be paced.

### SciPilot family

| Skill | Status | Purpose |
|---|---|---|
| **scipilot-cite** | v1.0.0 (this repo) | Reference discovery and insertion |
| scipilot-polish | Planned | Academic prose polishing |
| scipilot-review | Planned | AI peer-review simulation |
| scipilot-figure | Planned | Scientific figures |
| scipilot-submit | Planned | Submission formatting |
| scipilot-read | Planned | Paper reading and translation |

All members share four design principles:
1. **AI is a copilot** — wait for user sign-off at every decision point
2. **Authenticity first** — never fabricate academic information
3. **Primary-source driven** — rules come from real journal style guides
4. **Format is law** — one paper, one consistent style

### Roadmap

- **v1.1**: Add ACS, ACM Reference Format, Chicago, MLA
- **v1.2**: Optional PubMed and ADS (astronomy) data sources
- **v1.3**: Merge / dedupe / upgrade existing `.bib` files
- **v2.0**: Tight integration with `scipilot-polish` so newly inserted sentences are auto-polished

### Contributing

Bug reports, feature requests, and new-style contributions are welcome. Before opening a PR:
1. Ensure the embedded smoke tests in `scripts/utils.py` and `scripts/format_citation.py` still pass
2. New styles must update both `assets/format_templates/<style>.json` and `references/citation-formats.md`
3. Keep SKILL.md under 500 lines; long-form content belongs in `references/`

### License

[MIT](LICENSE) © 2026 Haojae

### Dependencies

```
requests>=2.31.0
python-docx>=1.1.0
python-Levenshtein>=0.23.0
```

Python 3.9+ recommended.
