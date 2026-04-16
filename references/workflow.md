# Workflow

## Regular Scholar (Default Search Surface)

### Single-Pass DOM Extraction

**One `evaluate_script` per page extracts everything needed — do NOT pair with `get_page_content`.**

For each page:

1. `evaluate_script` → single call extracts title, url, authors, year, venue, citedBy, snippet, badges
2. Parse response, append to accumulated papers array
3. `take_snapshot` → find "Next" link ID
4. `click` Next → `sleep 5s`
5. Repeat

### Concrete Execution Recipe

1. **Open page:**
   `new_page` → `https://scholar.google.com/scholar?q=<query>&hl=en`
2. **Apply time filter:**
   `take_snapshot` → locate "Since 2022" link → `click` → `sleep 5s`
3. **Extract page 1:**
   `evaluate_script` with the single-pass extraction script → append to papers array
4. **Paginate:**
   `take_snapshot` → "Next" link ID → `click` → `sleep 5s`
5. **Extract pages 2–N:**
   Repeat step 3–4 for each remaining page (max 6 pages)
6. **Filter + rank** accumulated papers
7. **Save** as individual `.md` files
8. **Report** summary → prompt IEEE fetch

### Single-Pass Extraction Script

```javascript
(() => {
  const cards = document.querySelectorAll('.gs_r');
  const results = [];
  cards.forEach((card, idx) => {
    const titleEl = card.querySelector('h3') || card.querySelector('.gs_rt');
    const titleLink = titleEl?.querySelector('a');
    const title = titleEl?.innerText?.replace(/^\d+\.\s*/, '').trim() || '';
    const url = titleLink?.href || '';

    const gsA = card.querySelector('.gs_a');
    const authorText = gsA?.innerText || '';
    const authors = authorText.split('-')[0]?.replace(/\d{4}/, '').trim() || '';
    const yearMatch = authorText.match(/\b(19|20)\d{2}\b/);
    const year = yearMatch ? yearMatch[0] : '';
    const venueParts = authorText.split('-');
    const venue = venueParts.slice(1).join('-').replace(/\d{4}/, '').trim() || '';

    const citedByEl = card.querySelector('a[href*="cites"]');
    const citedByText = citedByEl?.innerText || '';
    const citedBy = citedByText.replace(/Cited by /, '').trim() || '0';

    const snippetEl = card.querySelector('.gs_rs') || card.querySelector('.gs_sm');
    const snippet = snippetEl?.innerText?.trim() || '';

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
      results.push({ idx: idx + 1, title, url, authors, year, venue, citedBy, snippet, badges });
    }
  });
  return JSON.stringify(results, null, 2);
})()
```

**关键点：**
- `url` 直接从 `titleEl.querySelector('a')?.href` 提取，是出版社真实链接（IEEE Xplore / ScienceDirect / MDPI 等）
- `authors` / `venue` 从 `.gs_a` 文本解析，分割符为 `-`
- `citedBy` 从 `a[href*="cites"]` 提取
- `snippet` 从 `.gs_rs` 或 `.gs_sm` 提取
- `badges` 从 `.easyscholar-ranking` 提取（与 badge 提取脚本共用）

### Scholar Labs (If Specifically Requested)

1. `new_page` → `https://scholar.google.com/scholar_labs/search?hl=en`
2. `take_snapshot` → confirm "Ask Scholar" textbox visible
3. `evaluate_script` to set `#gs_as_i_t`, dispatch `input`, click `#gs_as_i_s`
4. `sleep 6`
5. `evaluate_script` for results (same single-pass script)
6. For new queries in same tab: click "New session" first

### Blocked State Handling

Blocked when:
- Google sign-in is visible
- CAPTCHA or unusual-traffic page is visible

When blocked after 2 retries → fall back to Regular Scholar.

## Handoff State Machine

`start -> blocked -> await_user -> resume | abort`

When blocked: "Google Scholar is blocked. Complete the page in BrowserOS until results are visible, then reply 'resume'. Reply 'abort' to stop."

After user replies `resume`: inspect page, continue only if results cards visible.

After user replies `abort`: stop and report failure.

## Deduplication

After extracting all pages:
- Normalize: lowercase + remove special chars from title
- Group by normalized title + year
- Keep: higher-positioned card, or richer metadata card

## Pagination Rules

- Element IDs change after each page reload — always `take_snapshot` before clicking
- Time filter "Since 2022" persists across pages — no need to re-apply
- Max 6 pages (60 papers)
- Stop pagination once user-specified count is reached
