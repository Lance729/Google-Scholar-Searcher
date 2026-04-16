<div align="center">
  <img src="./LOGO.png" alt="Scholar Labs Search" width="360"/>
</div>

# Google Scholar Searcher



[English](./README.md) | [中文](./README_zh_CN.md)

---

A Codex/Claude Code skill for searching Google Scholar via BrowserOS MCP. Searches Google Scholar, extracts paper metadata with Easy Scholar badge rankings, filters by CCF/SCI tier and recency, and saves results as individual Markdown files.

This is a derived work from [Scholar Labs Search](https://github.com/atsyplenkov/scholar-labs-search.git) by [@atsyplenkov](https://github.com/atsyplenkov), heavily extended with Easy Scholar integration, CCF/SCI filtering, time-based filtering, pagination support, and batch Markdown export.



## Key Improvements Over Original

| Feature | Original | This Fork |
|---------|----------|-----------|
| Badge extraction | None | Easy Scholar Chrome extension (level 1–5) |
| CCF/SCI filtering | None | Level 1/2 badge filter + recency |
| Time filtering | None | Since 2022 (4-year window) |
| Pagination | First page only | Up to 6 pages (60 papers) |
| URL extraction | Scholar redirect links | Direct publisher links (IEEE Xplore, ScienceDirect, etc.) |
| Output format | Single combined output | Per-paper `.md` files |
| Single-pass extraction | No | Yes — one `evaluate_script` per page |
| IEEE full-text fetch | None | `/ieee-paper-fetch` integration |

---

## Requirements

- **BrowserOS** installed and running locally with MCP enabled
- **Easy Scholar** Chrome extension (required for badge rankings)
- **Codex** or **Claude Code** with local skills support
- Network access to `https://scholar.google.com`

### BrowserOS MCP Configuration

```toml
[mcp_servers.browseros]
url = "http://127.0.0.1:9001/mcp"
```

Enable in BrowserOS: `Settings` → `BrowserOS as MCP`. See [BrowserOS docs](https://docs.browseros.com) for details.

---

## Easy Scholar Chrome Extension

**This skill requires the [Easy Scholar](https://chrome.google.com/webstore/detail/easy-scholar/alhpnkhbmobokehfejbhjnbkjpjklepp) Chrome extension.**

Easy Scholar overlays Chinese academic tier badges on each Google Scholar result card:

| Badge | Level | Example |
|-------|-------|---------|
| CCF A | 1 | Top-tier Chinese CS conferences/journals |
| SCI Q1 | 1 | Web of Science Q1 journals |
| CCF B | 2 | Second-tier Chinese CS |
| SCI Q2 | 2 | Web of Science Q2 journals |
| EI检索 | 2 | Engineering Index |
| 计算机科学TOP | 1 | CAS Top journal in CS |
| SCI升级版 计算机科学1区 | 1 | CAS升级版 1区 |

**Badge data is NOT visible in the accessibility tree** — it must be extracted via `evaluate_script` on the page DOM. This is handled automatically by the skill.

### Known Badge Patterns

```
Level 1: CCF A, SCI Q1, 计算机科学TOP, SCI基础版 工程技术1区, XJU 一区, SWJTU A++
Level 2: CCF B, SCI Q2, EI检索, SCI升级版 计算机科学2区, XJU 二区
Level 3: CCF C, SCI Q3
Level 4: SCI Q4
Level 5: 简介, 期刊全名 (not useful for filtering)
```

Badge source attribution is extracted from the `title` attribute: `数据来源为：(.+?)(]|$)`

---

## Installation

```text
~/.codex/skills/scholar-labs-search/
```

Or for Claude Code:

```text
~/.claude/skills/scholar-labs-search-master/
```

**Required files:**
```
scholar-labs-search/
├── SKILL.md              # Main skill file
└── references/
    ├── workflow.md       # BrowserOS execution recipe
    └── ranking.md       # Paper ranking rules
```

**Optional:**
```
agents/
└── openai.yaml          # Codex agent definition (optional)
```

---

## Usage

### Basic Search

```
/scholar-labs-search
```

Then specify your topic when prompted.

### Custom Search

```
Search Google Scholar for papers on "graph attention network scheduling edge computing"
```

### Workflow Example

1. Invoke skill → enter topic → specify paper count (default 10, max 60)
2. Skill applies "Since 2022" time filter automatically
3. Skill extracts metadata + badges via single `evaluate_script` per page
4. Papers filtered by Level 1/2 badge + year ≥ 2022
5. Papers ranked by badge priority + citation count
6. Each paper saved as `{badge-tag}-{year}-{slugified-title}.md`
7. Prompt: ask whether to fetch IEEE full-text for IEEE papers

---

## Output Format

### Markdown File Naming

```
{badge-tag}-{year}-{slugified-title}.md
```

**Badge tag priority:** CCF A > SCI Q1 > CCF B > SCI Q2 > CCF C > SCI Q3 > SCI Q4

If a paper has no CCF/SCI badge, the filename starts with year only.

**Example filenames:**
```
CCF-A-2022-weighted-feature-fusion-cnn-gat-hyperspectral.md
SCI-Q1-2024-graph-attention-networks-comprehensive-review.md
2023-novel-task-offloading-method.md
```

### Markdown Content

```markdown
# Paper Title

- **Authors:** Author1, Author2, Author3
- **Year:** 2024
- **Venue:** IEEE Transactions on Image Processing
- **Citations:** 594
- **Link:** [Title](url) | [PDF](pdf_url) | [IEEE Xplore](ieee_url)
- **Badges:** `CCF A` `SCI Q1` `IF 13.7`
- **Reason:** One-line reason for selection

> Abstract / snippet from Scholar (1-3 sentences).
```

---

## Filtering Rules

### Paper Selection (Regular Scholar Mode)

| Condition | Rule |
|-----------|------|
| Year ≥ 2022 | Keep if has ≥1 Level 1 or Level 2 badge |
| Year < 2022 | Keep only if Level 1 badge AND citations ≥ 2000 |
| No badge data | Exclude |

### Badge Priority for Filename

1. **CCF A** — highest priority
2. **SCI Q1**
3. **CCF B**
4. **SCI Q2**
5. **CCF C**
6. **SCI Q3**
7. **SCI Q4**
8. **Others** — not used in filename

Within the same tier, CCF wins over SCI (e.g., paper with both CCF A and SCI Q1 → filename starts with `CCF-A`).

---

## Technical Details

### Single-Pass DOM Extraction

One `evaluate_script` call per page extracts **all** metadata:

```javascript
document.querySelectorAll('.gs_r').forEach(card => {
  // title + direct URL from <a href>
  const titleEl = card.querySelector('h3') || card.querySelector('.gs_rt');
  const url = titleEl?.querySelector('a')?.href;

  // authors + year + venue from .gs_a text
  const authorText = card.querySelector('.gs_a')?.innerText;
  // split by '-' → authors | year | venue

  // citations from a[href*="cites"]
  // snippet from .gs_rs or .gs_sm
  // badges from .easyscholar-ranking
});
```

**URL extraction:** `card.querySelector('h3')?.querySelector('a')?.href` returns the **direct publisher link** (IEEE Xplore, ScienceDirect, MDPI, etc.) — not a Scholar redirect. This eliminates the need for Exa link verification in most cases.

### Pagination

- Element IDs change after each page reload — always `take_snapshot` before clicking
- "Since 2022" time filter persists across pages — no need to re-apply
- Max 6 pages (60 papers) per search

### Handoff States

The skill handles Scholar blocks (CAPTCHA, sign-in) by pausing and asking you to resolve in BrowserOS, then resume.

---

## File Structure

```
scholar-labs-search/
├── LOGO.png                        # Skill logo
├── SKILL.md                       # Main skill entry point
├── README.md                      # English documentation
├── README_zh_CN.md                # 中文文档
├── LICENSE                       # MIT license (from original repo)
└── references/
    ├── workflow.md              # BrowserOS execution recipe
    └── ranking.md               # Paper ranking rules
```

---

## Original Project

This skill is derived from [Scholar Labs Search](https://github.com/atsyplenkov/scholar-labs-search.git) by [@atsyplenkov](https://github.com/atsyplenkov).

The original was designed for Scholar Labs with annotation/key points. This fork targets **Regular Google Scholar** (more reliable, less likely to be blocked) with **Easy Scholar badge integration** for paper quality filtering.

---

## License

MIT — see [LICENSE](./LICENSE). Original work by [@atsyplenkov](https://github.com/atsyplenkov).
