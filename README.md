# SciPilot-cite

> SciPilot family. Your citation copilot for academic writing.
> SciPilot 家族成员。学术写作的引用副驾驶。

---

## 中文版

学术参考文献**自动检索 + 真实性验证 + 引用插入**的 Claude Code Skill。
读取你的 Word(.docx) 或 LaTeX(.tex) 论文，通过 **Semantic Scholar + OpenAlex + Crossref** 三源并行检索近 5 年高引文献，**DOI + 多源交叉**验证真实性（验证失败的直接丢弃），按 **IEEE / APA 7 / Nature / Vancouver / GB-T-7714** 任一格式插入正文并生成 References 章节。

**绝不编造任何文献。**

### 安装

**方式 A：让 Claude Code / Codex 自己装（推荐，最简）**

把本仓库 GitHub URL 直接告诉 Claude Code 或 Codex 即可：

```
请帮我安装这个 Skill：https://github.com/haojaeyang-stack/scipilot-cite
```

它会自动 clone 到正确目录（`~/.claude/skills/` 或 `~/.codex/skills/`），并提示你装 Python 依赖。

**方式 B：手动 clone**

```bash
git clone https://github.com/haojaeyang-stack/scipilot-cite.git ~/.claude/skills/scipilot-cite
pip install -r ~/.claude/skills/scipilot-cite/requirements.txt
```

**方式 C：下载 ZIP**

1. 在 GitHub 页面点 Code → Download ZIP
2. 解压到 `~/.claude/skills/scipilot-cite/`（或 Codex / Cursor 对应目录）
3. `pip install -r requirements.txt`

三个学术 API 全部免费访问，**无需任何 API key**。

### 使用示例

在 Claude Code 中直接用自然语言触发：

```
帮我的 paper.tex 加 15 篇参考文献，用 IEEE 格式。
```

```
搜索 large language model alignment 相关近 3 年的高引论文，
排除预印本，用 Nature 格式插入到 manuscript.docx。
```

```
读取 thesis.tex，在 Related Work 部分补充 10 篇引用，
按 GB/T 7714 格式。
```

Skill 会先逐项确认参数（论文文件 / 篇数 / 格式 / 年限 / 预印本 / 特殊要求），再开始检索→验证→插入。

### 工作流（8 个 Stage）

```
Stage 0  信息收集（数量/格式/年限/预印本/特殊要求）
   |
Stage 1  论文分析（章节抽取/已有引用/关键词）
   |
Stage 2  文献检索（S2 / OpenAlex / Crossref 并行 + 去重）
   |
Stage 3  文献验证（DOI / 跨源 → VERIFIED / LIKELY_REAL / UNVERIFIED）
   |
Stage 4  筛选排序（相关性40% + 质量30% + 时效20% + 多样性10%）
   |  （此处等待用户确认）
Stage 5  引用插入（LaTeX \cite / docx [N] 标记）
   |
Stage 6  格式化输出（IEEE / APA / Nature / Vancouver / GB-T-7714）
   |
Stage 7  最终检查（连续性 / 一致性 / 真实性复核 / 原文 diff）
```

### 命令行直接调脚本

不通过 Skill 也可单独使用：

```bash
# 检索
python scripts/search_papers.py "diffusion model" --limit 10 --json > papers.json

# 验证 DOI
python scripts/verify_paper.py doi 10.1038/s41586-024-07421-0 \
       --title "Detecting hallucinations..." --year 2024

# 格式化单条
echo '{"title":"...","authors":["..."],"year":2024,"venue":"...","doi":"..."}' | \
  python scripts/format_citation.py --style ieee --number 1

# 插入到 LaTeX
python scripts/insert_citations_latex.py paper.tex papers.json --style ieee

# 插入到 Word
python scripts/insert_citations_docx.py paper.docx papers.json --style nature
```

### SciPilot 家族

| Skill | 状态 | 功能 |
|---|---|---|
| **scipilot-cite** | v1.0 (本仓库) | 文献检索与引用插入 |
| scipilot-polish | 规划中 | 学术论文润色 |
| scipilot-review | 规划中 | AI 模拟审稿 |
| scipilot-figure | 规划中 | 科研图表 |
| scipilot-submit | 规划中 | 投稿格式适配 |
| scipilot-read | 规划中 | 论文阅读翻译 |

家族成员共享四条设计原则：
1. AI 是副驾驶，关键决策点等待用户确认
2. 真实性第一，不编造任何学术信息
3. 一手文献驱动，规则来自真实期刊规范
4. 格式即法律，同一论文格式必须绝对一致

### 许可证

MIT — 见 [LICENSE](LICENSE)。

---

## English

A Claude Code Skill for **automatic citation discovery, authenticity verification, and reference insertion**.
Read your Word (.docx) or LaTeX (.tex) manuscript; search recent high-citation papers via **Semantic Scholar + OpenAlex + Crossref** in parallel; verify every paper through **DOI resolution + multi-source cross-check** (failures are discarded, not fabricated); insert citations in the body and build the References section in **IEEE / APA 7 / Nature / Vancouver / GB/T 7714** style.

**Never fabricates any reference.**

### Installation

**Option A: Let Claude Code / Codex install it for you (recommended)**

Just hand the GitHub URL to Claude Code or Codex:

```
Please install this Skill for me: https://github.com/haojaeyang-stack/scipilot-cite
```

It will clone the repo into the correct skills directory (`~/.claude/skills/` or `~/.codex/skills/`) and prompt you to install the Python deps.

**Option B: Manual clone**

```bash
git clone https://github.com/haojaeyang-stack/scipilot-cite.git ~/.claude/skills/scipilot-cite
pip install -r ~/.claude/skills/scipilot-cite/requirements.txt
```

**Option C: Download ZIP**

1. On the GitHub page click Code → Download ZIP
2. Extract into `~/.claude/skills/scipilot-cite/` (or your Codex / Cursor skills folder)
3. `pip install -r requirements.txt`

All three academic APIs use free-tier endpoints. **No API key required.**

### Usage

Trigger from natural language inside Claude Code:

```
Add 15 references to my paper.tex in IEEE format.
```

```
Find recent (last 3 years) high-citation papers on large language model
alignment, exclude preprints, and insert them into manuscript.docx
in Nature style.
```

```
Read thesis.tex, add 10 citations in the Related Work section using
GB/T 7714 format.
```

The Skill will confirm all parameters (paper file / count / style / year range / preprint policy / special requirements) before running the search → verify → insert pipeline.

### Workflow (8 stages)

```
Stage 0  Parameter collection (count / style / years / preprint / extras)
   |
Stage 1  Manuscript analysis (sections / existing cites / keywords)
   |
Stage 2  Search (S2 / OpenAlex / Crossref in parallel, dedup by DOI)
   |
Stage 3  Verification (DOI / cross-check -> VERIFIED / LIKELY_REAL / UNVERIFIED)
   |
Stage 4  Ranking (relevance 40 + quality 30 + recency 20 + diversity 10)
   |  (wait for user confirmation)
Stage 5  Insertion (LaTeX \cite / docx [N] markers)
   |
Stage 6  Format output (IEEE / APA / Nature / Vancouver / GB-T-7714)
   |
Stage 7  Final audit (numbering / consistency / re-verify / diff)
```

### Command-line usage

Run the scripts directly without the Skill harness:

```bash
# Search
python scripts/search_papers.py "diffusion model" --limit 10 --json > papers.json

# Verify a real DOI
python scripts/verify_paper.py doi 10.1038/s41586-024-07421-0 \
       --title "Detecting hallucinations..." --year 2024

# Format a single entry
echo '{"title":"...","authors":["..."],"year":2024,"venue":"...","doi":"..."}' | \
  python scripts/format_citation.py --style ieee --number 1

# Insert into LaTeX
python scripts/insert_citations_latex.py paper.tex papers.json --style ieee

# Insert into Word
python scripts/insert_citations_docx.py paper.docx papers.json --style nature
```

### SciPilot family

| Skill | Status | Purpose |
|---|---|---|
| **scipilot-cite** | v1.0 (this repo) | Reference discovery & insertion |
| scipilot-polish | Planned | Academic prose polishing |
| scipilot-review | Planned | AI peer-review simulation |
| scipilot-figure | Planned | Scientific figures |
| scipilot-submit | Planned | Submission formatting |
| scipilot-read | Planned | Paper reading & translation |

All members share four design principles:
1. AI is a copilot; wait for user confirmation at every decision point
2. Authenticity first; never fabricate academic information
3. Primary-source driven; rules come from real journal style guides
4. Format is law; one paper, one consistent style

### License

MIT — see [LICENSE](LICENSE).
