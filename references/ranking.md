# Ranking

## Objective Modes

Use `best-fit` mode by default.

Switch to `importance-weighted` mode only if the user explicitly asks for one of these intents:

- Seminal
- Classic
- Survey
- Review
- Most cited

## Best-Fit Mode

Sort candidates using these precedence rules, in order:

1. Topic match strength
   - Exact phrase or all core query terms in the title
   - All core query terms across title plus annotation
   - Partial but still relevant match
2. Paper type
   - Review, survey, systematic review, meta-analysis, or benchmark
   - Ordinary empirical or method paper
3. Influence proxy
   - **Badge signal** (Easy Scholar badges, checked before citations):
     - Level 1 badges: `CCF A`, `计算机科学TOP`, `SCI基础版 工程技术1区`, `SWJTU A++`, `XJU 一区`
     - Level 2 badges: `CCF B`, `EI检索`, `SCI升级版 计算机科学2区`, `SCI Q2`, `SWJTU A+`
     - Rank by: most level-1 badges → most level-2 badges → highest combined badge level
   - Visible citation count (used when badge signal is equal)
   - Strong venue or source signal when citation count is absent
4. Recency
   - Newer year first
5. Stable tie-breaker
   - Earlier Scholar page position

Discard obviously off-topic candidates before sorting.

## Importance-Weighted Mode

Sort candidates using these precedence rules, in order:

1. Review, survey, systematic review, meta-analysis, or clearly foundational paper
2. **Badge signal** (CCF A/B > SCI Q1/Q2 > others, then citation count)
3. Topic match strength
3. Topic match strength
4. Recency
5. Earlier Scholar page position

## Link Verification

Use Exa only after the top 3 are selected.

Prefer links in this order:

1. DOI landing page
2. Publisher or repository landing page
3. arXiv
4. Scholar link

Replace the Scholar link only if identity is verified by one of these checks:

- Exact DOI match
- Normalized title match plus same year plus matching first author

If identity is ambiguous, keep the Scholar link.

## Quick Acceptance Checks

Use these checks when validating the skill behavior:

- Query `attention is all you need`
  - The candidate set includes `Attention Is All You Need`.
  - The top 3 includes it.
- Query `u-net biomedical image segmentation`
  - The candidate set includes `U-Net: Convolutional Networks for Biomedical Image Segmentation`.
  - The top 3 includes it.
- Query `transformer language models survey`
  - The top 3 includes at least one paper with `survey` or `review` in the title.
