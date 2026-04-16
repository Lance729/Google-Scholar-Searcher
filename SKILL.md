---
name: scholar-labs-search
description: Search Google Scholar Labs through BrowserOS MCP to find papers for a topic, with Easy Scholar badge extraction, CCF/SCI ranking, Markdown output, time filtering, and pagination. Use when Codex needs an on-demand literature search against Scholar Labs rather than a full literature review.
---

# Scholar Labs Search

Use BrowserOS MCP to inspect Google Scholar Labs and return best-fit papers for the user's topic. Treat Scholar Labs as the primary search surface. Use Exa only after selection to verify or improve outbound links.

## Setup

Require BrowserOS running locally with MCP enabled before using this skill.

If the `browseros` MCP server is not configured in Codex, stop and tell the user to add it from BrowserOS Settings -> BrowserOS as MCP. Do not guess the MCP URL.

Use this config shape:

```toml
[mcp_servers.browseros]
url = "http://127.0.0.1:9001/mcp"
```

## Workflow

### Step 0 — Paper Count

Ask the user: "请指定需要抓取多少篇论文？（默认10篇，最多60篇，即6页）"

- If user gives N: collect up to N papers
- If user says default / skip: collect 10 papers
- If N > 10: paginate up to ceil(N / 10) pages, max 6 pages
- **Do not expand level filter beyond level 1/2. Do not paginate beyond page 6.**

### Step 1 — Prepare Query

- If user gives keywords: use as-is
- If user gives a problem statement: rewrite once into a compact Scholar query (main topic + method terms only)
- If first query returns zero usable results: retry once after dropping weakest qualifier
- If retry also returns zero: return `0 papers found` and stop

### Step 2 — Open Scholar Labs

Navigate to `https://scholar.google.com/scholar?q=<query>&hl=en`.

Prefer `new_page` over `new_hidden_page` for active searching. After opening, use `take_snapshot` to confirm page loaded.

### Step 3 — Apply Time Filter

Use `take_snapshot` to locate `"Since 2022"` link, click it, then sleep 5s for results to refresh.

Time filter options: `"Any time"`, `"Since 2026"`, `"Since 2025"`, `"Since 2022"`, `"Custom range..."`.

Click `"Since 2022"` (4-year window = current year − 3). Time filter persists across pagination; no need to re-apply.

### Step 4 — Extract All Pages (Single evaluate_script per page)

**Single DOM extraction per page — do NOT use get_page_content separately.**

Use this one `evaluate_script` call per page:

```javascript
(() => {
  const cards = document.querySelectorAll('.gs_r');
  const results = [];
  cards.forEach((card, idx) => {
    // Title + URL from <a href>
    const titleEl = card.querySelector('h3') || card.querySelector('.gs_rt');
    const titleLink = titleEl?.querySelector('a');
    const title = titleEl?.innerText?.replace(/^\d+\.\s*/, '').trim() || '';
    const url = titleLink?.href || '';

    // Authors from .gs_a
    const gsA = card.querySelector('.gs_a');
    const authorText = gsA?.innerText || '';
    const authors = authorText.split('-')[0]?.replace(/\d{4}/, '').trim() || '';

    // Year
    const yearMatch = authorText.match(/\b(19|20)\d{2}\b/);
    const year = yearMatch ? yearMatch[0] : '';

    // Venue (after '-' in .gs_a)
    const venueParts = authorText.split('-');
    const venue = venueParts.slice(1).join('-').replace(/\d{4}/, '').trim() || '';

    // Cited-by count
    const citedByEl = card.querySelector('a[href*="cites"]');
    const citedByText = citedByEl?.innerText || '';
    const citedBy = citedByText.replace(/Cited by /, '').trim() || '0';

    // Snippet / abstract
    const snippetEl = card.querySelector('.gs_rs') || card.querySelector('.gs_sm');
    const snippet = snippetEl?.innerText?.trim() || '';

    // Badges (Easy Scholar)
    const badgeEls = card.querySelectorAll('.easyscholar-ranking');
    const badges = [];
    badgeEls.forEach(badge => {
      const titleAttr = badge.getAttribute('title') || '';
      const sourceMatch = titleAttr.match(/数据来源为：(.+?)(]|$)/);
      const source = sourceMatch ? sourceMatch[1].trim() : 'unknown';
      badges.push({
        label: badge.innerText.trim(),
        level: parseInt(badge.className.match(/easyscholar-(\d)/)?.[1]) || 0,
        source: source
      });
    });

    if (title) {
      results.push({
        idx: idx + 1,
        title,
        url,
        authors,
        year,
        venue,
        citedBy,
        snippet,
        badges,
        page: 1
      });
    }
  });
  return JSON.stringify(results, null, 2);
})()
```

**流程 per page:**
1. `evaluate_script` → 单次提取 title + url + authors + year + venue + citedBy + snippet + badges
2. Parse JSON response
3. Append to accumulated papers array
4. `take_snapshot` → find "Next" link ID
5. `click` Next → sleep 5s
6. Repeat until last page

**返回结构示例:**
```json
{
  "title": "Weighted feature fusion of CNN and GAT for hyperspectral",
  "url": "https://ieeexplore.ieee.org/abstract/document/9693311/",
  "authors": "Y Dong, Q Liu, B Du, L Zhang",
  "year": "2022",
  "venue": "IEEE Transactions on Image Processing",
  "citedBy": "594",
  "snippet": "...",
  "badges": [
    { "label": "CCF A", "level": 1, "source": "CCF" },
    { "label": "SCI Q1", "level": 1, "source": "JCR" },
    { "label": "IF 13.7", "level": 1, "source": "JCR影响因子" }
  ],
  "page": 1
}
```

### Step 5 — Filter

Apply to all accumulated candidates:

**Rule A — Year ≥ 2022:**
- Keep if has **at least one level-1 or level-2 badge**
- Pre-2022 only if Rule B applies

**Rule B — Pre-2022 papers (year < 2022):**
- Keep only if: **badge level = 1 AND citations ≥ 2000**

**Rule C — No badge data:**
- Exclude if no Easy Scholar badges at all

**Rule D — Deduplication:**
- Normalize title (lowercase, remove special chars) + year
- Keep higher-positioned card or richer metadata

After filtering, if fewer than 3 remain, return the reduced set and explain the filter was applied.

### Step 6 — Rank

Sort by:
1. Badge level (most level-1 badges first, then level-2)
2. Citation count (descending)
3. Year (newer first)
4. Position on page (lower position first)

Badge priority for filename: CCF A > SCI Q1 > CCF B > SCI Q2 > CCF C > SCI Q3 > SCI Q4

### Step 7 — Ask Save Location

Prompt: "查询完成，共 N 篇候选论文。请选择保存位置：① 仅屏幕展示（不保存）② 保存到文件（请提供路径）③ 保存到项目目录（请指定项目路径）"

### Step 8 — Save as Individual .md Files

Each paper → one `.md` file using naming convention:

```
{badge-tag}-{year}-{slugified-title}.md
```

**Badge tag** (highest priority):
| Priority | Badge | Tag |
|----------|-------|-----|
| 1 | CCF A | CCF-A |
| 2 | SCI Q1 | SCI-Q1 |
| 3 | CCF B | CCF-B |
| 4 | SCI Q2 | SCI-Q2 |
| 5 | CCF C | CCF-C |
| 6 | SCI Q3 | SCI-Q3 |
| 7 | SCI Q4 | SCI-Q4 |
| 8+ | 其他 | skip |

Within same tier, CCF wins. If no CCF/SCI badge, filename starts with year only.

`slugified-title`: lowercase, spaces → hyphens, remove special chars, truncate to 60 chars.

**File content format:**
```markdown
# <Paper Title>

- **Authors:** Author1, Author2
- **Year:** 2024
- **Venue:** Journal / Conference Name
- **Citations:** 39
- **Link:** [Title](url) | [PDF](pdf_url) | [IEEE Xplore](ieee_url)
- **Badges:** `CCF A` `SCI Q1` `IF 8.9`
- **Reason:** One-line reason for selection

> Abstract / snippet (1-3 sentences from snippet).
```

### Step 9 — Return Result Summary

- Output up to user-specified number of papers (default 10, max 60)
- Each paper: title, authors, year, venue, citations, link, reason, all badges with levels
- If fewer than 3 usable results: return smaller set and explain why

### Step 10 — Prompt IEEE Full-Text Fetch

After all papers saved: scan for IEEE-affiliated papers.
Prompt: "已保存 N 篇论文。其中有 M 篇来自 IEEE Xplore。是否需要抓取 IEEE 论文全文？"
If user says yes: invoke `/ieee-paper-fetch` skill for each IEEE paper.

## Scope

- Use up to 6 pages max (60 papers). Do not paginate beyond page 6.
- **Mode**: try Scholar Labs first → fallback to Regular Scholar if blocked after 2 retries
- **Regular Scholar filtering**: only level-1 or level-2 badge papers (or level-1 + citations ≥ 2000 for pre-2022)
- **Paper count**: user-specified (default 10, max 60). Do not expand level filter beyond 1/2.
- Do not silently fall back to OpenAlex or Semantic Scholar

## BrowserOS Notes

### Scholar Labs
- Reliable tool sequence:
  1. `new_page`
  2. `take_snapshot`
  3. `evaluate_script` to set `#gs_as_i_t`, dispatch `input`, click `#gs_as_i_s`
  4. `sleep 6`
  5. `take_snapshot` + `evaluate_script` for results
- `fill` + `press_key Enter` may not submit — use evaluate_script fallback

### Regular Scholar (fallback or user-specified)
- Direct URL: `https://scholar.google.com/scholar?q=<query>&hl=en`
- Wait `sleep 8` after navigation
- Time filter: click "Since 2022" link via `take_snapshot` then `click`
- Badge extraction: same `evaluate_script` with `.easyscholar-ranking`
- Pagination: `take_snapshot` → "Next" link ID → `click` → sleep 5s

## Failure Rules

Stop and report:
- BrowserOS MCP unavailable
- **Both** Scholar Labs (after 2 retries) **and** regular Scholar blocked
- Regular Scholar returns zero results after one retry
- Filtering reduced all candidates to zero

Returning fewer than 3 results due to filtering is **not a failure** — expected behavior for Regular Scholar mode.

## Easy Scholar Badge Extraction

Easy Scholar (Chrome extension) overlays Chinese-tier badges. These are **not** in the accessibility tree — must use `evaluate_script`.

### DOM Query

```js
(() => {
  const results = [];
  document.querySelectorAll('.easyscholar-ranking').forEach(badge => {
    const card = badge.closest('.gs_or_cite') ||
                 badge.closest('[data-paper-id]') ||
                 badge.closest('.gs_r') ||
                 badge.parentElement;
    const titleEl = card?.querySelector('h3') || card?.querySelector('.gs_rt');
    const title = titleEl?.innerText?.replace(/^\d+\.\s*/, '').trim() || 'unknown';
    const titleAttr = badge.getAttribute('title') || '';
    const sourceMatch = titleAttr.match(/数据来源为：(.+?)(]|$)/);
    const source = sourceMatch ? sourceMatch[1].trim() : 'unknown';
    results.push({
      title: title.substring(0, 80),
      label: badge.innerText.trim(),
      level: parseInt(badge.className.match(/easyscholar-(\d)/)?.[1]) || 0,
      source: source
    });
  });
  return JSON.stringify(results, null, 2);
})()
```

### Badge Level Scale

| Level | Meaning | Example Labels |
|-------|---------|----------------|
| 1 | Highest | 计算机科学TOP, SCI基础版 工程技术1区, SWJTU A++, **SCI Q1**, **CCF A** |
| 2 | High | EI检索, SCI升级版 计算机科学2区, SWJTU A+, **SCI Q2**, **CCF B** |
| 3 | Medium | SCI升级版 计算机科学3区, IF 2.9, **CCF C**, IF(5) |
| 4 | Lower | SCI升级版 计算机科学4区 |
| 5 | Lowest | SWUFE B, 简介, 期刊全名 |

### Badge Label Patterns

**CCF:** `CCF A` (level 1), `CCF B` (level 2), `CCF C` (level 3)
**JCR:** `SCI Q1` (level 1), `SCI Q2` (level 2), `SCI Q3` (level 3), `SCI Q4` (level 4)
**中科院:** `计算机科学TOP`; `SCI升级版 计算机科学1/2/3/4区`; `SCI基础版 工程技术1/2/3/4区`
**EI:** `EI检索` (level 2)
